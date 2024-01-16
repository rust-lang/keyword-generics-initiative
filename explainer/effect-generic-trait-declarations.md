- Feature Name: `effect-generic-trait-decls`
- Start Date: (2024-01-01)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

This RFC introduces Effect-Generic Trait Declarations. These are traits which
are generic over Rust's built-in effect keywords such as `async` and `const`.
Instead of defining two near-identical traits per effect, this RFC allows a
single trait to be declared which is generic over the effect. Here is a variant
of the `Into` trait which can be implemented as either async or not.

```rust
#[maybe(async)]
trait Into<T>: Sized {
    #[maybe(async)]
    fn into(self) -> T;
}
```

Implementers can then choose whether to implement the base version or the
effectful version of the trait. If they want the base version they don't include
the `async` effect. If they want the `async` version they can include the
`async` keyword.

```rust
/// The base implementation
impl Into<Loaf> for Cat {
    fn into(self) -> Loaf {
        self.nap()
    }
}

/// The async implementation
impl async Into<AsyncLoaf> for AsyncCat {
    async fn into(self) -> AsyncLoaf {
        self.async_nap().await
    }
}
```

# Motivation
[motivation]: #motivation

Rust is a single language that's made up of several different sub-languages.
There are the macro languages, as well as the generics language, patterns,
const, unsafe, and async sub-languages. Rust works anywhere from a
micro-controller to Windows, and even browsers. One of the biggest challenges we
have is to not only keep the language as easy to use as we can, it's to ensure
it works relatively consistently on all the different platforms we support.

We're currently in the process of adding support for the `const` and `async`
language features to Rust. But we're looking at various other extensions as
well, such as generator functions, fallible functions, linearity, and more.
These are really big extensions to the language, whose implementation will take
on the order of years. If we want to successfully introduce these features,
they'll need to be integrated with every other part of the language. As well as
having wide support in the stdlib.

Effect Generic Trait Declarations are a minimal language feature which enable
traits to add support for new effects, without needing to duplicate the trait
itself. So rather than having a trait `Into`, `TryInto`, `AsyncInto`, and the
inevitable `TryAsyncInto` - we would declare a single trait `Into` once, which
has support for any combination of `async` and `try` effects. This is
backwards-compatible by design, and should be able to support any number of
effect extensions we come up with in the future. Ensuring the language can keep
evolving to our needs.

## Guaranteeing API consistency

Evolving a programming language and stdlib is pretty difficult. We have to pay
close attention to details. And in Rust specifically: once we make a mistake
it's pretty hard to roll back. And we've made mistakes with effects in the past,
which we now have to work with.

In Rust 1.34 we stabilized a new trait: `TryInto`. This was supposed to be the
fallible version of the `Into` trait, containing a new associated type `Error`.
However since Rust 1.0 we've also had the
[`FromStr`](https://doc.rust-lang.org/std/str/trait.FromStr.html) trait, which
*also* provides a fallible conversion, but has an associated type `Err`. This
means that when writing a fallible conversion trait, it's unclear whether the
associated type should be called `Err` or `Error`.

This might seem minor, but without automation these subtle
similar-but-not-quite-the-same kinds of differences stand out. The only way to
ensure that different APIs in different contexts work consistently is via
automation. And the best automation we have for this is the type system.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Trait definitions

The base of Effect Generic Trait Declarations is the ability to declare traits
as being generic over effects. This RFC currently only considers the `async`
effect, but should be applicable to most other effects (modulo `unsafe` and
`const`, more on that later). The way a trait is defined is by adding a
`#[maybe(effect)]` notation.  This signals that a trait may be implemented as
carrying the effect.  For example, a version of the `Into` trait which may or
may not be `async` would be defined as:

```rust
#[maybe(async)]
pub trait Read {
    #[maybe(async)]
    fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
}
```

## Trait implementations

Traits can be implemented as either async or non-async.  The trait-level
`#[maybe(async)]` can be thought of as a const-generic bool which determines the
value of the method-level `#[maybe(async)]` declarations.  So if a trait is
implemented as `async`, all methods tagged as `#[maybe(async)]` have to be async
too.

```rust
/// The base implementation
impl Read for Reader {
    fn read(&mut self, buf: &mut [u8]) -> Result<usize> {
        // ...
    }
}

/// The async implementation
impl async Read for AsyncReader {
    async fn read(&mut self, buf: &mut [u8]) -> Result<usize> {
        // ...
    }
}
```

## Method markers

This RFC only covers trait methods which carry an effect or not. It does not
cover types which may or may not have effects. The intent is to add this via a
future extension, so for the scope of this RFC we have to be able to declare
certain methods as not being generic over effects. This is the default behavior;
no extra annotations are needed for this.

Taking the `Read` trait example again; the `chain` method returns a type `Chain`
which implements `Iterator`. Accounting for the `chain` method, the declaration
of `Read` would be:

```rust
#[maybe(async)]
pub trait Read {
    ...

    // This method is not available for `impl async Read`
    fn chain<R: Read>(self, next: R) -> Chain<Self, R>
       where Self: Sized { .. }
}
```

Because `chain` is not marked as `maybe(async)`, when implementing `async Read`,
it will not be available. If a synchronous method has to be available in an
async context, it should be possible to mark it as `not(async)`, so that it's
clear it's part of the API contract for the async implementation - and is never
async.

```rust
#[maybe(async)]
pub trait Read {
    ...

    // This method would be available for `impl async Read`
    #[not(async)]
    fn chain<R: Read>(self, next: R) -> Chain<Self, R>
       where Self: Sized { .. }
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Effect lowering

At the MIR level the lowering of `#[maybe(effect)]` is shared with `const`, and
is essentially implemented via const bools. Take the following maybe-async
definition of `Into`:

```rust
// Trait definition
#[maybe(async)]
trait Into<T>: Sized {
    #[maybe(async)]
    fn into(self) -> T;
}
```

At the MIR level the `#[maybe(async)]` system is lowered to a const bool which
determines whether the function should be async. If the trait is implemented as
async, the bool is set to true. If it isn't, it's set to false.

```rust
// Lowered trait definition
trait Into<T, const IS_ASYNC: bool>: Sized {
    type Ret = T;
    fn into(self) -> Self::Ret;
}
```

By default the const bool should be set to false. The return type of the
function here is the base return type of the definition:

```rust
/// The base implementation
impl Into<Loaf> for Cat {
    fn into(self) -> Loaf {
        self.nap()
    }
}

// Lowered base trait impl
impl Into<Loaf, false> for Cat { // IS_ASYNC = false
    type Ret = T;
    fn into(self) -> Self::Ret {
        self.nap()
    }
}
```

However if we implement the async version of the trait things change a little.
In the lowering the const bool is set to `true` to indicate we are in fact
async. And in the lowering we wrap the return type in an `impl Future`, as well
as return an anonymous `async {}` block from the function.

```rust
/// The async implementation
impl async Into<AsyncLoaf> for AsyncCat {
    async fn into(self) -> AsyncLoaf {
        self.async_nap().await
    }
}

// Lowered async trait impl
impl Into<AsyncLoaf, true> for AsyncCat { // IS_ASYNC = true
    type Ret = impl Future<Output = T>;
    fn into(self) -> Self::Ret {
        async move {
            self.async_nap().await
        }
    }
}
```

## Effect lowering with lifetimes

Things become more interesting when lifetimes are involved in the effectful
lowering of a trait. The return type of an `async fn` which takes a reference
has to be a future with a lifetime. Which means it's in our lowering our
associated type can't be a plain future - it has to be a future with a lifetime
attached. And this requires lifetime GATs to work.

Say instead of the async version of `Into`, we tried to write the maybe-async
version of [`AsRef`](https://doc.rust-lang.org/std/convert/trait.AsRef.html)
[^asref]. We could define it as follows:

[^asref]: this is just for the purpose of an example; I don't actually know of
any cases which want an async version of `AsRef`. But never say never.

```rust
/// The trait definition
#[maybe(async)]
pub trait AsRef<T>
where
    T: ?Sized,
{
    #[maybe(async)]
    fn as_ref(&self) -> &T;
}

/// The lowering of the trait definition
pub trait AsRef<T, const IS_ASYNC: bool>
where
    T: ?Sized,
{
    type Ret<'a>
        where Self: 'a;
    fn as_ref(&self) -> Self::Ret<'_>;
}
```

We could then implement it like we did with our `Into` impl. The non-async impl
would look like this:

```rust
/// The base implementation
impl AsRef<Loaf> for Cat {
    fn as_ref(&self) -> &Loaf {
        self.nap_ref()
    }
}

/// Lowering of the base implementation
impl AsRef<Loaf, false> for Cat { // IS_ASYNC = false
    type Ret<'a> = &'a Loaf
        where Self: 'a;
    fn as_ref(&self) -> Self::Ret<'_> {
        self.nap_ref()
    }
}
```

And the async implementation would look like this:

```rust
/// The base implementation
impl async AsRef<Loaf> for AsyncCat {
    async fn as_ref(&self) -> &Loaf {
        self.async_nap_ref().await
    }
}

/// Lowering of the base implementation
impl AsRef<AsyncLoaf, true> for AsyncCat { // IS_ASYNC = true
    type Ret<'a> = impl Future<Output = &'a Loaf> + 'a
        where Self: 'a;
    fn as_ref(&self) -> Self::Ret<'_> {
        async {
            self.async_nap_ref().await
        }
    }
}
```

While effect-generic trait definitions with lifetimes do rely on GATs in their
lowering, crucially they don't rely on any potential notion of lifetime-generics to
function. The right lifetime GATs can be emitted by the compiler during
lowering, and should therefor always be accurate.

## Effect logic

This RFC reasons about effects as being in one of four logical states:

- __Always:__ This is when an effect is always present. For example: if a
    function implements some kind of concurrency operations, it may always want to
    be async. This is signaled by the existing meaning of the `async fn` notation.
- __Maybe__: This is when an effect may sometimes be present. This will
  apply to most traits in the stdlib. For example, if we want to write an async
  version of the `Read` trait its associated methods will also want to be `async`.
- __Not__: This is when an effect is never present. For example:
  `Iterator::size_hint` will likely *never* want to be async, even if the trait
  and most methods are async. In order for methods to be available in the
  effectful implementatin of the trait, they have to be marked as never
  carrying the effect.
- __Unknown:__ Methods which haven't explicitly declared which logical state
  they're in are *unknown*. This is a distinct state from *not*, because a method
  may be converted from *unknown* to *maybe* without breaking backwards
  compatibility.

For the `async` effect methods which are always async are labeled `async fn`.
Methods which may or may not be async are labeled `#[maybe(async)]`. Methods
which are never async are labeled `#[not(async)]`. All other methods are
unlabeled, and are not made available to the async implementation of the trait.

## Concrete impls and coherence

With the eye on forward-compatibility, and a potential future where types can
themselves also be generic over effects, for now types may only implement either
the effectful or the base variant of the trait. This ensures that the door is
kept open for effect generic implementations later on. As well as ensures that
during trait selection the trait variant remains unambiguous. The diagnostics
for this case should clearly communicate that only a single trait variant can be
implemented per type.

```text
error[E0119]: conflicting implementations of trait `Into` for type `Cat`
 --> src/lib.rs:5:1
  |
4 | impl Into for Cat {}
  | ----------------- first implementation here
5 | impl async Into for Cat {}
  | ^^^^^^^^^^^^^^^^^ conflicting implementation for `Cat`
  |
  | help: types can't both implement the sync and async variant of a trait
```

## Trait bounds

Using effect generic trait definitions in trait bounds should be no problem,
assuming the bounds are concrete. Unlike concrete types, generic bounds may
implement both effecful and uneffectful implementations for the same bounds as
long as they target non-overlapping sets of traits. For example, assuming we had
a maybe-async version of `Into`, introducing a maybe-async version of `From`
would allow us to write the following non-overlapping generic bounds.

```rust
/// If we also introduce a maybe-async
/// version of the `From` trait…
#[maybe(async)]
pub trait From<T>: Sized {
    #[maybe(async)]
    fn from(value: T) -> Self;
}

/// …we can implement the synchronous
/// variant for any type `T, U: From<T>`…
impl<T, U> Into<U> for T
where
    U: From<T> {}

/// …as well as the asynchronous variant for
/// any type `T, U: async From<T>`.
impl<T, U> async Into<U> for T
where
    U: async From<T> {}
```

For the purpose of the trait resolver, `From` and `async From` should be
considered non-overlapping bounds. This is a new capability which we'll need to
introduce, and effectively comes down to treating `U: From<T, false>` and `U:
From<T, true>` as non-overlapping bounds. Effect-generic trait bounds
(conditional effects in bounds) are not introduced by this RFC, but may be introduced
by a future extension.

## Super traits

Super-trait hierarchies should be supported, as long as they are appropriately
annotated. Say we wanted to define a maybe-async version of
[`BufRead`](https://doc.rust-lang.org/std/io/trait.BufRead.html) which has
`Read` as a supertrait. For that to work, the `Read` trait would also need to be
marked maybe-async. That way if we implement the async version of `BufRead` we
also require the async version of `Read` - idem for the non-async variants.

```rust
#[maybe(async)]
pub trait BufRead: #[maybe(async)] Read {
    #[maybe(async)]
    fn fill_buf(&mut self) -> Result<&[u8]>;
    #[maybe(async)]
    fn consume(&mut self, amt: usize);
}
```

If a trait wants to have a non-async super-trait, it has to mark the super-trait
as not being async. In the case that the supertrait eventually becomes generic
over an effect, it's clear from the beginning which variant we 've chosen. 

```rust
#[maybe(async)]
pub trait SuperTrait {}

#[maybe(async)]
pub trait SubTrait: #[not(async)] SuperTrait { }
```

Certain traits may want to guarantee ahead of time that they will never support
a certain effect. For these traits it is possible to omit the effect marker, as
the state of the effect is already unambiguous. It is expected most marker
traits will want to be unambiguously never support for example the `async`
effect.

```rust
// The trait `Sized` guarantees it
// will not ever be an `async trait`… 
#[not(async)]
trait Sized {}

// …which means it does not require annotations
// when used as a supertrait.
#[maybe(async)]
trait Into<T>: Sized { .. }
```

## TODO: prerequisites

- ask oli about which compiler features we're missing to implement this

# Drawbacks
[drawbacks]: #drawbacks

## TODO: Additional syntax

- we're adding some new syntax, that's going to be A Thing

## TODO: Direction

- while not inherently closing any doors, we are kind of committing to the idea
  that we want to extend the stdlib to be effectful - that's the point
- this has repercussions for how we structure our base traits and interfaces too

# Prior art
[prior-art]: #prior-art

TODO:
- swift: async polymorphism + rethrow
- c++: noexcept + constexpr
- koka: effect handlers (free monad)
- rust: const fn
- zig: maybe async functions

# Unresolved questions
[unresolved-questions]: #unresolved-questions

## TODO: Syntax

- `#[maybe(async)]` is a placeholder
- `maybe(async)` is clear but is verbose
- `?async` is sigil-heavy, but has precedence in the trait system
- `~async` is sigil-heavy, and also reserves a new sigil
- `if/else` at the trait level does not create bidirectional relationships
- `async<A>` is less clear and verbose

# Alternatives
[alternatives]: #alternatives

## TODO: Do nothing (null hypothesis)

- effect differences are inherent, which means they have to be solved somewhere
- effect composition is where it gets bad; we have an async version of the stdlib, not an async + fallible version
- things like linearity seem quite far out of reach right without this

# Future possibilities
[future-possibilities]: #future-possibilities

## TODO: Integration with other keywords

- fallible functions
- generator functions
- linearity

## TODO: Effect-generic types and bodies

- types
- functions

```rust
/// Before: the base implementation
impl Into<Loaf> for Cat {
    fn into(self) -> Loaf {
        self.nap()
    }
}

/// Before: the async implementation
impl async Into<AsyncLoaf> for AsyncCat {
    async fn into(self) -> AsyncLoaf {
        self.async_nap().await
    }
}
```

```rust
// After: a single implementation
...
```

## TODO: Effect sets

- named effect sets
- unify `core` and `std` via sets

## TODO: Normalize const

- `const fn` is maybe-const
- `const {}` is always const
- this is super annoying lol, and that's why this system doesn't work for `const` right now
