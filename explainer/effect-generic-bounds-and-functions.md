- Feature Name: `effect-generic-bounds-and-functions`
- Start Date: (2024-01-20)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

[RFC 0000](./effect-generic-trait-declarations.md) introduces traits which are
generic over an effect, but implementers have to pick whether they want to
implement the base version or the effectful version of the trait. This RFC
extends that system further by removing that limitation, and enabling authors to
write functions which themselves are generic over effects. For example, here is
a function `io::copy` which would be able to operate either synchronously or
asynchronously, depending on which types are passed to it.

```rust
/// This defines a trait `Read` which may or may not
/// be async, using the design introduced in RFC 0000.
#[maybe(async)]
pub trait Read {
    #[maybe(async)]
    fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
}

/// This defines a trait `Write` which may or may not
/// be async, using the design introduced in RFC 0000.
#[maybe(async)]
pub trait Write {
    #[maybe(async)]
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    #[maybe(async)]
    fn flush(&mut self) -> Result<()>;
}

/// This defines a function `copy` which copies bytes from a
/// reader into a writer. This RFC enables this function to
/// operate either synchronously or asynchronously. Where if
/// operating synchronously, the `.await` operator becomes a
/// no-op.
#[maybe(async)] 
pub fn copy<R, W>(reader: &mut R, writer: &mut W) -> Result<()>
where
    R: #[maybe(async)] Read + ?Sized,
    W: #[maybe(async)] Write + ?Sized,
{
    let mut buf = vec![0; 1024];
    loop {
        match reader.read(&mut buf).await? {
            0 => return Ok(()),
            n => writer.write(&mut buf[0..n]).await?,
        };
    }
}
```

# Motivation
[motivation]: #motivation

[RFC 0000](./effect-generic-trait-declarations.md) introduced effect-generic
trait definitions: traits which are generic over effects, but implementors of
the trait have to pick which version they implement. This works fine when
authors know which effects they will be working with, like in applications. But
library authors will often want to write code which not only works with one
effect, but any number of effects. And for that effect-generic functions and
bounds would greatly help reduce the amount of code duplication.

The blog post: ["The bane of my
existence: Supporting both async and sync code in
Rust"](https://nullderef.com/blog/rust-async-sync/) documents the negative
experience of one of the [`rspotify`] authors maintaining both sync and async
versions of a crate. With ecosystem crates such as
[`maybe_async`](https://docs.rs/maybe-async/latest/maybe_async/) and
[`async-generic`](https://crates.io/crates/async-generic) attempting to provide
mitigations via the macro system. But these crates are limited in what they can
provide when it comes to integrating with Rust's libraries, diagnostics,
tooling, and inference systems.

`rspotify` is a thoroughly documented example of an author wanting to be
generic over an effect, but there are others. Specifically for the async effect,
the `mongodb` crate has both [sync][mdb] and [async][mdba] variants. So does the
`postgres` crate [[sync][pg], [async][pga]], as well as the `reqwest` crate
[[sync][rqw], [async][rqwa]], and both the [`tokio`] and [`async-std`] crates
duplicate large swaths of the stdlib's functionality. Other effects are also
covered, such as [`fallible-iterator`] for a version of the stdlib's `Iterator`
trait which short-circuits on error.  And [`fallible_vec`] for a `Vec` type with
methods which may returns errors rather than panics if an allocation fails.

[`rspotify`]: https://docs.rs/rspotify/latest/rspotify/index.html
[mdb]: https://docs.rs/mongodb/latest/mongodb/sync/index.html
[mdba]: https://docs.rs/mongodb/latest/mongodb/index.html
[pg]: https://docs.rs/postgres/latest/postgres/
[pga]: https://docs.rs/tokio-postgres/latest/tokio_postgres/index.html
[rqw]: https://docs.rs/reqwest/latest/reqwest/blocking/index.html
[rqwa]: https://docs.rs/reqwest/latest/reqwest/
[`tokio`]: https://docs.rs/tokio
[`async-std`]: https://docs.rs/async-std
[`fallible-iterator`]: https://docs.rs/fallible-iterator/latest/fallible_iterator/index.html
[`fallible_vec`]: https://crates.io/crates/fallible_vec

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Effect-generic functions

Effects such as `async`, `try`, or `gen` define a superset of the language. With
some minor exceptions, they provide access to more features than
functions which don't have the effect. However, to ensure the effect forwards
correctly through function bodies, we require some degree of annotations. In the
case of the `async` effect, we require function calls to be suffixed with
`.await`. In the case of `try`, we require function calls to be suffixed with
`?`. This is called "effect forwarding".

Say we wanted to write a function `sum` which takes an `impl Iterator<Item =
u32>` and sums all numbers together. If we included the trait definition, we could write it like so:

```rust
/// The `Iterator` trait
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

/// Iterate over all numbers in the
/// iterator and sum them together
pub fn sum<I: Iterator<Item = u32>>(iter: I) -> u32 {
    let mut total = 0;
    while let Some(n) = iter.next() {
        total += n;
    }
    total
}
```

This works fine for non-async code. In fact: this is almost exactly how the
stdlib versions of
[`Iterator::sum`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.sum)
is defined. But this code has a limitation: the underlying iterator will always
block between calls to `next`. To resolve that, `next` should add support for
`async/.await`, which we can do by adding the `#[maybe(async)]` notation.

```rust
/// The `Iterator` trait with optional
/// support for the `async` effect
#[maybe(async)]
pub trait Iterator {
    type Item;
    #[maybe(async)]
    fn next(&mut self) -> Option<Self::Item>;
}
```

Nothing too exciting is going on yet. This is all uses the mechanisms we defined
in [RFC 0000](), and adding support for `async/.await` just required some extra
`#[maybe(async)]` annotations. This becomes more interesting once we start
looking to not only allow traits to be generic over the `async` effect, but
*functions* and *trait bounds* as well. The way we can do that is fairly
mechanical: all we have to do is add some extra `#[maybe(async)]` notations to
the function signature, and some extra `.await`s inside the function body.

```rust
/// Iterate over all numbers in the
/// iterator and sum them together
#[maybe(async)]                                // 1. async
pub fn sum<I>(iter: I) -> u32
where
    I: #[maybe(async)] Iterator<Item = u32>    // 2. async
{
    let mut total = 0;
    while let Some(n) = iter.next().await {    // 3. .await
        total += n;
    }
    total
}
```

In this example we've added additional `#[maybe(async)]` notations at comments
1, and 2. And in the function body added additional `.await` points at comment 3.
What's key here is that if we remove the `async` and `await` notations, we
end up back with a perfectly valid non-async code. And that's basically the way
the system works under the hood: when compiled as async, the `.await` points
signal suspension points. While when the function is compiled as non-async, you
can think of the `.await` points as immediately returning if they suspend.

In order to make this system work though, we have to apply some rules. The
first rule is that `#[maybe(async)]` functions can't directly call `async`
functions. Remember: our function needs to be able to strip away the `.await`
points and still compile. If a function is always async, then removing the
`.await` points would leave us with an invalid function, so that's not allowed.

The second rules is: maybe-async functions may or may not return futures. So we
can't treat the return type like a concrete future which we can freely pass
around. That means that in `#[maybe(async)]` contexts, the only valid thing to
do with futures is to `.await` immediately them.

## Using effect-specific behavior

Not being able to treat futures as first-class items in `#[maybe(async)]`
functions might seem like a pretty big restriction: half the reason to use
`async/.await` in the first place is to be able to concurrently execute
computations. But there is a direct way out for us here: intrinsic-based
specialization.

```rust
fn runtime() -> i32 { 1 }
const fn compiletime() -> i32 { 1 }
unsafe { const_eval_select((), compiletime, runtime) }
```

const functions can specialize their behavior using the `const_eval_select`
intrinsic. Depending on whether execution is occurring during compilation or at
runtime, different code will be run. For `async` and other effects, we'll be
providing a similar intrinsic. Depending on whether a function is compiled as
async or not, different code will be run.

```rust
fn not_async() { println!("hello sync"); }
async fn yes_async() { println!("hello async"); }
unsafe { async_eval_select((), not_async, yes_async) }
```

Within the `async` function it would be possible to freely operate on futures as
first-class items: freely applying concepts such as concurrency, cancellation,
and combinations of the two. In the future we may choose to expose similar
functionality via a first-class language feature instead. See the future
possibilities section for a discussion of this.

## Effect selection and inference

While functions can be written as generic over effects, when they're finally
compiled the compiler needs to know which variant to use. In most cases the
compiler should be able to infer this unambiguously from the context. When a
`#[maybe(async)]` function is called from an `async` context, and the is then
`.await`ed - the compiler can be pretty certain we're interested in the `async`
version of the function.

```rust
#[maybe(async)]
fn meow() {
    println!("meow");
}

async fn main() {
    meow().await; // `fn meow` can be inferred to be `async`
}
```

If the presence - or absence - of `.await` calls isn't enough to inform the
compiler, it will look at whether the enclosing context is `async` or not. And
if that's not enough to unambiguously figure out which variant to select, it
will always be possible to explicitly tell the compiler which variant we
expected using turbofish notation.

```rust
#[maybe(async)]
fn meow() {
    println!("meow");
}

async fn main() {
    let fut = meow::<async>();   // `fn meow` is async
    meow::<#[not(async)]>();     // `fn meow` is not async
}
```

## Effect generic provided trait methods

This RFC not just enables free functions to be generic over effects: it also
enables provided trait methods to work with effect generics. Take for example
our maybe-async trait `Read` again. By default it provides a number of methods,
including `read_to_end`. With effect generic trait definitions, those provided
functions can be generic over effects, meaning they can be made available to the
effectful and base variants of the trait alike.

```rust
#[maybe(async)]
pub trait Read {
    // Using RFC 0000 required trait methods gained
    // support for `#[maybe(effect)]` annotations.
    #[maybe(async)]
    fn read(&mut self, buf: &mut [u8]) -> Result<usize>;

    // With this RFC, entire functions can be made
    // generic over effects, meaning provided functions
    // in traits now also work with `#[maybe(effect)]`.
    #[maybe(async)]
    fn read_to_end(&mut self, buf: &mut Vec<u8>) -> Result<usize> { .. }
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## TODO: Lowering

_NOTE: this section is incomplete and in-progress. While we know it is possible
because we have implemented a working version of this outside of the compiler,
there are changes in the compiler happening which means this section may change.
Once those changes have landed, this section should be rewritten to match that.
Until that time please consider this section incomplete and subject to change._

Let's continue with our earlier example of the trait `Iterator` and the function
`sum` which both have conditional support for the `async` effect via
`#[maybe(async)]`. In its base form the trait `Iterator` looks like this:

```rust
/// The `Iterator` trait
#[maybe(async)]
pub trait Iterator {
    type Item;
    #[maybe(async)]
    fn next(&mut self) -> Option<Self::Item>;
}
```

Which using the desugaring proposed in RFC 0000, would desugar to a trait with a
const-generic bool which determines whether it is async or not:

```rust
// Lowered trait definition
pub trait Iterator<const IS_ASYNC: bool = false> {
    type Item;
    type Ret = Option<Self::Item>;
    fn next(&mut self) -> Ret;
}
```

We've seen how when maybe-async traits are implemented as sync they return their
base type, and when implemented as async they wrap that up in an `impl Future` -
see RFC 0000's section on "Lowering" for more details. Now for our function
body, the base definition looks like this:

```rust
#[maybe(async)]
pub fn sum<I>(iter: I) -> u32
where
    I: #[maybe(async)] Iterator<Item = u32>
{
    let mut total = 0;
    while let Some(n) = iter.next().await {
        total += n;
    }
    total
}
```

TODO: ask oli for more details about the desugaring. Ref is: https://github.com/yoshuawuyts/maybe-async-channel/blob/2fc6fa012830525482d62a8facfae5e5a5e762fe/maybe-async-std/src/lib.rs#L47-L54

## Carried effects as non-destructive code transformations

The reason why this RFC is able to write function bodies which are generic over
effects is because effects such as `async` are non-destructive. Adding the
`async` notation to a function does not lose any information - meaning you can
arrive at the function signature and body you might had had before simply by
removing the `async` and `.await` notation.

Take this simple async function. It calls the method `meow` on some type
`cat`, returning `String`.

```rust
/// 1. Our base `meow` function
fn meow() -> String {
    cat.meow()
}
```

Say `cat.meow` was `async` instead. We could change our function to support it
simply by adding the necessary `async` and `.await` notations.

```rust
/// 2. The async version of `meow`
async fn meow() -> String {
    cat.meow().await
}
```

Because adding `async` and `.await` does not erase any information from the base
function, it is non-destructive. Meaning we can always reverse it by removing
all the calls to `async` and `.await`, arriving back at the function we
initially had.

```rust
/// 3. Stripping the `async/.await` notations
/// yields our base function again
fn meow() -> String {
    cat.meow()
}
```

Uncarried effects such as `const` don't require any forwarding notations, and so
are by definition non-destructive in their transformation. In addition to
`async`, Rust has two other carried effects: `gen` and `try`. Parts of both of
these effects are unstable or undecided, but there is no reason we should
require their notation to be destructive. Given the unstable nature of these
effects, we'll cover them in more detail in the "future possibilities" section
of this RFC.

| effect name | forwarding notation | desugaring                            | logical return type | carried type |
| ----------- | ------------------- | ------------------------------------- | ----------- | ------------ |
| `async`     | `.await`            | `impl Future<Output = T>`             | `T`         | `!`          |
| `try`†      | `?`                 | `impl Try<Output = T, Residual = R>`† | `T`         | `R`          |
| `gen`†      | `yield from` ‡      | `impl Iterator<Item = U>`             | `()`        | `U`          |



_† These items exist in Rust, but are unstable._

_‡ These items have been discussed for inclusion, but have not yet been included on nightly._

_"logical return type" in this context means: the type the function returns
after the function has been evaluated and any forwarding notation has been
applied. The "carried type" here refers to the additional types end-users need
to be aware of when the effect is introduced. For example, when using the `try`
effect, users will be exposed to additional `Result<_, E>` or `Option` types._

## TODO: Unambiguous variant selection

- `copy::<async>(read, writer).await?;`

## TODO: Effect-generic bodies logic

|                                 | caller does not have effect | caller may have an effect  | caller always has effect |
| ------------------------------: | --------------------------- | -------------------------- | ------------------------ |
| **callee does not have effect** | ✅ allowed to evaluate      | ✅ allowed to evaluate     | ✅ allowed to evaluate   |
|      **callee may have effect** | ✅ allowed to evaluate      | ✅ allowed to evaluate     | ✅ allowed to evaluate   |
|    **callee always has effect** | ❌ not allowed to evaluate  | ❌ not allowed to evaluate | ✅ allowed to evaluate   |

Evaluating an async function in a non-async context is not possible.

```rust
//! Caller context does not have an effect,
//! callee always has an effect

async fn callee() {}
fn caller() {
    callee().await // ❌ cannot call `.await` in non-async context
}
```

The caller's context may be evaluated as synchronous, but the callee is
guaranteed to always be asynchronous. Because as we've seen it's not possible to
evaluate async functions in non-async contexts.

```rust
//! Caller context may have an effect,
//! callee always has an effect

async fn callee() {}
#[maybe(async)]
fn caller() {
    callee().await // ❌ cannot call `.await` in maybe-async context
}
```

## TODO: effect-row polymorphism

- the effect for all members in a function is the same bound
- if you mix async + non-async, both have to be async to work
- but that's generally ok: there's a subtyping relationship possible, so even if
  we don't do it automatically we can just do it ourselves

## TODO: Prerequisites

- ask oli about which compiler features we're missing to implement this

# Drawbacks
[drawbacks]: #drawbacks

## Limits the future effects we can add

- Being able to add N new carried effects for various different purposes is out of the cards
- But we're specifically fine with the carried effects we have, uncarried
  effects are more interesting as they provide more features by constraining the
  design space
- Arbitrary user-defined effects can likely be defined by contexts/capabilities + yield

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## TODO: Don't require forwarding notation

- important though; as that's where control flow may happen
- the possibility of something happening is the entire point of annotating it

## TODO: Flattened compositional hierarchy

- requires a `do` notation / `.await?` / `?.await` become a single operation
- ends up with a single `Coroutine` uber trait from which all other traits are derived
- only covers carried effects, not uncarried ones
- unclear how it would enable effect-generic functions to be authored
- results in a system of trait aliases

## TODO: sans-io

- yeah sans-io is cool
- but it depends on passing things like `impl Read` rather than directly calling `File::open`
- this means taking maybe-async interfaces, and so we need maybe-async logic
- ergo: while a good idea, it's not an alternative

## TODO: TLS-preserving closures

- unclear how we would combine scope escapes with the borrow checker
- threading through effects + forwarding notations through all call sites achieves the same effect
- main challenge is backwards-compat, but effect generics address that

# Prior art
[prior-art]: #prior-art

## const fn

Using the `const` keyword this is already possible in Rust: a single `const fn`
function can both be evaluated during compilation and at runtime. Contrast this
to `const {}` blocks, which can only be evaluated at compile-time. And using the
[`const_eval_select`](https://doc.rust-lang.org/core/intrinsics/fn.const_eval_select.html)
intrinsic it will even be possible to provide different implementations
depending on whether the function is evaluated at runtime or during compilation.
This enables the runtime variant of a function to provide more optimized
implementations, for example by leveraging platform-specific SIMD capabilities.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

# Future possibilities
[future-possibilities]: #future-possibilities


## TODO: `try/?` contexts

- is already non-destructive
- just unstable right now

## TODO: `gen/yield from` contexts

- Recognize that the return type is not the yield type
- An additional `yield from`-alike syntax would be helpful here
    - preferred notation: `for yield..in expr;`

## TODO: Composition of `gen/yield from`, `async/.await` and `try/?`

- 2/3 of these effects have unstable components
- but they would compose Just Fine
- this should be its own RFC though

## TODO: Arbitrary user-defined carried effects

- composition of `yield`, contexts/capabilities, and concrete types
- a handler can be expressed as a context or capability
- we can yield N values to it by passing it a generator function
- See also: "capabilities: effects for free" which applies this idea
- Removes the need for arbitrary built-in control-flow effects
