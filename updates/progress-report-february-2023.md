# Progress Report February 2023

* This post is intended to be published on the Rust internals blog.*

## Introduction

About 9 months ago [we announced][announce] the creation of the Keyword Generics
Initiative; a group working under the lang team with the intent to solve the 
[function coloring problem][color] [^color] through the type system not just for
`async`, but for `const` and all current and future function modifier keywords
as well.

We're happy to share that we've made a lot of progress over these last several
months, and are now working on writing RFCs. This post is an overview of the
designs we'll be covering in those RFCs.

[announce]: https://blog.rust-lang.org/inside-rust/2022/07/27/keyword-generics.html
[color]: https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/

[^color]: To briefly recap this problem: you can't call an `async fn` from a
non-async fn. This makes the "async" notation go viral, as every function that
calls it also needs to be async. But we believe possibly more importantly: it
requires a duplication of most stdlib types and ecosystem libraries. Instead we
suspected we might be able to overcome this issue by introducing a new kind of
generic which would enable functions and types to be "generic" over whether
they're async or not, const or not, etc.

## An async example

In our last post we introduced the placeholder `async<A>` syntax to describe the
concept of a "function which is generic over its asyncness". We always knew we
wanted something lighter weight, and we've chosen to go with the
`?async`-notation for functions and types which are generic over their
"asyncness". We refer to these as "maybe-async" functions.

To show an example. Say we took the [`Read` trait][read] and the
[read_to_string_methods][rts]. In the stdlib their implementations look somewhat
like this today:

```rust
trait Read {
    fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
    fn read_to_string(&mut self, buf: &mut String) -> Result<usize> { ... }
}

/// Read from a reader into a string.
fn read_to_string(reader: &mut impl Read) -> std::io::Result<String> {
    let mut string = String::new();
    reader.read_to_string(&mut string)?;
    Ok(string)
}
```

Now, what if we wanted to make these async in the future? Using `?async`
notation we could change them to look like this:

```rust
trait ?async Read {
    ?async fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
    ?async fn read_to_string(&mut self, buf: &mut String) -> Result<usize> { ... }
}

/// Read from a reader into a string.
?async fn read_to_string(reader: &mut impl ?async Read) -> std::io::Result<String> {
    let mut string = String::new();
    reader.read_to_string(&mut string).await?;
    Ok(string)
}
```

The way this works is that `Read` and `read_to_string` are now generic over
their "asyncness". When compiled for an `async` context, they'll behave
asynchronously. When compiled in a non-async context, they'll behave
synchronously. The `.await` in the `read_to_string` function body is necessary
to mark the cancellation pointin case the function is compiled as async; but it
when not async it will essentially be a no-op [^always-async-maybe]:

[^always-async-maybe]: One important rule for `?async` contexts is that they can
only call other `?async` and non-`async` functions. Because if it would call an
always-`async` function, there would be no clear right thing to do when compiled
as a non-async function. So async-only things like concurrency won't directly
work in always-async contexts. But using the `if is_async() {} else {}`
functionality we show later on in the post you could define `?async` functions
which make use of concurrency operations *only* when compiled in async mode.

```rust
// `read_to_string` is inferred to be `!async` because
// we didn't `.await` it, nor expect a future of any kind.
#[test]
fn sync_call() {
    let _string = read_to_string("file.txt")?;
}

// `read_to_string` is inferred to be `async` because
// we `.await`ed it.
#[async_std::test]
async fn async_call() {
    let _string = read_to_string("file.txt").await?;
}
```

## A const example

For `const` we're planning to introduce syntax which mirrors the syntax for
`async`. We want function modifier keywords in Rust to feel like they're part
of a single language, following consistent rules. So this means using `?const`
to denote "maybe-const" bounds and contexts.

Taking the `Read` example we had earlier, we could imagine a "maybe-const" version
of the `Read` trait to look like this:

```rust
trait ?const Read {
    ?const fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
    ?const fn read_to_string(&mut self, buf: &mut String) -> Result<usize> { ... }
}

/// Read from a reader into a string.
?const fn read_to_string(reader: &mut impl ?const Read) -> std::io::Result<String> {
    let mut string = String::new();
    reader.read_to_string(&mut string).await?;
    Ok(string)
}
```

Just like with `?async` traits, traits will need to be marked as `?const` both
when declared and when used. But additionally we also intend to change the
existing `const fn` notation to become `?const fn`, making it clearer that what
we call a `const fn` today is not in fact guaranteed to be const-evaluated: it
*may* be used in `const` contexts, but doesn't have to. In contrast: using a `const`-declaration or `const` block will be guaranteed to be *always*-const [^const-post].

[^const-post]: You can read more about this in the [Keywords II: Const Syntax](https://blog.yoshuawuyts.com/const-syntax/) blog post.

[read]: https://doc.rust-lang.org/std/io/trait.Read.html
[rts]: https://doc.rust-lang.org/std/io/fn.read_to_string.html

## Combining const and async

For completion's sake, let's take a look at what the `Read` trait would look like when
it's both maybe-const and maybe-async:

```rust
trait ?const ?async Read {
    ?const ?async fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
    ?const ?async fn read_to_string(&mut self, buf: &mut String) -> Result<usize> { .. }
}

/// Read from a reader into a string.
?const ?async fn read_to_string(reader: &mut impl ?const ?async Read) -> io::Result<String> {
    let mut string = String::new();
    reader.read_to_string(&mut string).await?;
    Ok(string)
}
```

While this does in fact accurately describe what's going on, you can see it's
starting to look rather verbose. You can imagine that if we were to throw
fallibility (`throws`) or some other keyword in the mix it would quickly start
resembling a bit of a keyword stew. To fix this we intend to a new `effect/.do`
notation which will enable being generic over *all* modifier keywords. This is
not yet part of what we'll be proposing, as it needs a lot more design work. But
it is an important part of our final vision, so we want to provide a brief
glimpse at what we think it might look like if we rewrote the `Read` trait using that:

```rust
trait ?effect Read {
    ?effect fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
    ?effect fn read_to_string(&mut self, buf: &mut String) -> Result<usize> { .. }
}

/// Read from a reader into a string.
?effect fn read_to_string(reader: &mut impl ?effect Read) -> std::io::Result<String> {
    let mut string = String::new();
    reader.read_to_string(&mut string).do;  // note the singular `.do` here
    string
}
```

`?effect` would be equivalent to `?const` and `?async`, but importantly: would
in the future extend to include any other keywords as well. For example: `?gen`
or `?throws`. And using the `.do` notation we could make sure the right
combination of `.await`, `?`, `yield`, etc. are always inserted in the right
spot.

There are a bunch of things we still need to figure out for this. For example:
we probably would want a modifier keyword for fallibility (e.g. `throws`) to be
stabilized first, so that `.do` can be expanded to include `?` as well. And we
probably want to find a way to enable different modifier keywords to be marked
as mutually exclusive. For example: "can panic" and "will return an error" may
be exclusive to one another. Or perhaps a function wants to be generic over most
modifier keywords, but requires it to always be `async`. It should be possible
for functions to enforce these kinds of restrictions in the type system, and we're
not yet sure how we'd go about this.

## Adding support for types

We don't just want `?const` and `?async` to apply to functions, traits, and
trait bounds. We want people to be able to use it in types as well. This will
enable us to start providing async types such as `TcpStream` and `File` directly
from the stdlib, without needing to duplicate interfaces.

The challenge with concrete async types is that their behavior will be different
when used in async and non-async contexts. Instead of blocking on a call, it
instead asynchronously _waits_ on a call. In the implementation this means
different system calls, and different fields in the struct. Luckily we have
accounted for this in our design.

Say we wanted to implement `?async File` with a single `?async open` method. The
way we expect this to look will be something like this:

```rust
/// A file which may or may not be async
struct ?async File {
    file_descriptor: std::os::RawFd,  // shared field in all contexts
    async waker: Waker,               // field only available in async contexts
    !async meta: Metadata,            // field only available in non-async contexts
}

impl ?async File {
    /// Attempts to open a file in read-only mode.
    ?async fn open(path: Path) -> io::Result<Self> {
        if is_async() {   // compiler built-in function
            // create an async `File` here; can use `.await`
        } else {
            // create a non-async `File` here
        }
    }
}
```

This enables authors to use different fields depending on whether they're compiling
for async or not, while still sharing a common core. And within function bodies
to provide different behaviors depending on the context as well. The function
body notation would work as a generalization of the currently unstable
[`const_eval_select`][eval-select] intrinsic, and at least for the function
bodies we expect a similar `is_const()` compiler built-in to be made available
as well.

[eval-select]: https://doc.rust-lang.org/std/intrinsics/fn.const_eval_select.html
[connect]: https://doc.rust-lang.org/std/net/struct.TcpStream.html#method.connect

## Consistent syntax

One of the biggest challenges in language design is adding features in a way that
makes them feel like they're a core part of the language, rather than a separate
language which just happens to be shipped in the same package. And modifier
keywords such as `async` and `const` definitely risk feeling like their own
niche if we're not careful.

Luckily in Rust we have the ability to make surface-level changes to the
language through the edition system. There are many things this doesn't let us
do, but it does allow us to require syntax changes. We want to leverage this
to help make `const` and `async` feel consistent with one another, and to make
their base versions feel consistent with their maybe-counterparts.

For `const` this means there should be a syntactic distinction between `const`
declarations and `const` uses. Currently when you write `const fn` you get a
function which can be evaluated both at runtime and during compilation. But when
you write `const FOO: () = ..;` the meaning of `const` there guarantees
compile-time evaluation. Same keyword, different meanings. So for that reason we
want to propose we change `const fn` to `?const fn` to more clearly indicate
that this function may be const-evaluated, but can also be used at runtime.

```rust
//! Define a function which may be evaluated both at runtime and during
//! compilation.

// Current
const fn meow() -> String { .. }

// Proposed
?const fn meow() -> String { .. }
```

For `async` we're planning similar surface-level consistency changes. To implement
inherent `?async` methods on types we will required that the type itself is
labeled with `?async`. This is currently not the case when implementing inherent
always-async methods on types, which is an inconsistency we may want to address:

```rust
//! Proposed: define a method on a maybe-async type
struct ?async File { .. }
impl ?async File {
    ?async fn open(path: PathBuf) -> io::Result<Self> { .. }
}

//! Current: define a method on an always-async type
struct File { .. }
impl File {
    async fn open(path: PathBuf) -> io::Result<Self> { .. }
}

//! Proposed: define a method on an always-async type
struct async File { .. }
impl async File {
    async fn open(path: PathBuf) -> io::Result<Self> { .. }
}
```

Similarly, the Async WG is in the process of moving from "async functions in
traits" to a design for "async traits". This would unlock the `async(Send)`
syntax which will enable declaring bounds on the anonymous futures return by the
async methods returned by the trait. The ["Lightweight, Predictable Async Send
Bounds"][bounds-post] by Eric Holk covers this design in more detail, but here's
a brief example of what it would look like when combined with the `struct async`
notation:

```rust
struct async File { .. }
impl async Read for async File {
    async fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> { .. }
}

async fn read_to_string(reader: &mut impl async Read) -> io::Result<String> {
    let mut string = String::new();
    reader.read_to_string(&mut string).await?;
    Ok(string)
}
```

What's nice about the `reader: &mut impl async Read` bound is that it won't
distinguish between "maybe-async" and "always-async" types. If it takes a type
which implements `?async Read`, it will just interpret it as a type implementing
`async Read`. While this is a slight increase in verbosity, it will make usage
of async types unambiguous, and importantly make `async` and `?async` types
behave consistently and interchangeably. This also means that it will be
backwards-compatible to mark an existing (async) type or trait as
`?async`, since they would require bounds on their usage.

## The plan

Our initial focus will be on the `async` and `const` keywords specifically.
We're now ready to start writing RFCs, so we have the following planned (in no
particular order):

1. `?async fn` notation without trait bounds, including an `is_async` mechanism.
2. `trait async`  declarations and bounds.
3. `trait ?async` declarations and bounds, `trait ?const` declarations and bounds.
4. `?const fn` notation without trait bounds.
5. `struct async` notation and `struct ?const` notation.

We're going to be working with the Lang Team, and Async WG on these.
Specifically the `async trait` notation RFC will be driven by the Async WG,
with the Keyword Generics Initiative taking an advicing role. Only once those
RFCs have been completed will we start looking more closely at the `effect/.do`
notation. We're doing that because `effect/.do` needs keywords to be generic
over, and there currently are none yet.

## Conclusion

And that concludes the summary of what we're looking at. We intentionally didn't
drill too deep into details because that's what the RFCs will be for. Which
speaking of: we'll be publishing those over the coming months, and once they go
through it shouldn't be long before we can bring the features to nightly. Most
of what we've discussed in this post (except `effect/.do` and specific inference
bits) has already been successfully prototyped outside of the stdlib, meaning
we have a really good sense of how it's going to work and are fairly confident
in what we're presenting in this post.
