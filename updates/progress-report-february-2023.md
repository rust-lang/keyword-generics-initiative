# Progress Report Feburary 2023

* This post is intended to be published on the Rust internals blog.*

## Introduction

About 9 months ago [we announced][announce] the creation of the Keyword Generics
Initiative; a group working under the lang team with the intent to solve the 
[function coloring problem][color] [^color] not just for `async`, but for
`const` and all current and future function keywords as well.

We're happy to share that we've made a lot of progress over these last several
months, and are now working on writing RFCs.  This post is a summary of what
we'll be including in those RFCs.

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

In our last post we introduced the placeholder `async<A>` syntax. We used this
to describe the semantics of keyword generics. We always knew we wanted something
lighter weight, and we've chosen to go with the `?async`-notation for functions and types which are generic over their "asyncness". We refer to this as "maybe-async" functions.

Say we took the [`Read` trait][read] and the [read_to_string_methods][rts]. In the stdlib their implementations look somewhat like this today:

```rust
trait Read {
    fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
    fn read_to_string(&mut self, buf: &mut String) -> Result<usize> { ... }
}

/// Read from a reader into a string.
fn read_to_string(reader: &mut impl Read) -> std::io::Result<String> {
    let mut string = String::new();
    reader.read_to_string(&mut string)?;
    string
}
```

Now, what if we wanted to make these async in the future? Using `?async`
notation we change them to look like this:

```rust
trait ?async Read {
    ?async fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
    ?async fn read_to_string(&mut self, buf: &mut String) -> Result<usize> { ... }
}

/// Read from a reader into a string.
?async fn read_to_string(reader: &mut impl ?async Read) -> std::io::Result<String> {
    let mut string = String::new();
    reader.read_to_string(&mut string).await?;
    string
}
```

The `read_to_string` function is `?async`, which means it can be called from
`async`, `?async` and non-`async` contexts alike:

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
    string
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

## A const and async example

For completion's sake, let's take a look at what the `Read` trait would look like when
it's both maybe-const and maybe-async:

```rust
trait ?const ?async Read {
    ?const ?async fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
    ?const ?async fn read_to_string(&mut self, buf: &mut String) -> Result<usize> { .. }
}

/// Read from a reader into a string.
?const ?async fn read_to_string(reader: &mut impl ?const ?async Read) -> std::io::Result<String> {
    let mut string = String::new();
    reader.read_to_string(&mut string).await?;
    string
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

## The plan

Our initial focus will be on the `async` and `const` keywords specifically.
We're now ready to start writing RFCs, so we have the following planned (in no
particular order):

1. `?async fn` notation without trait bounds, including an `async_eval_select`
    mechanism [^async-eval].
2. `async trait` declarations and bounds.
3. `?async trait` declarations and bounds, `?const trait` declarations and bounds
4. `?const fn` notation, becoming the recommended notation for Rust 2024.

We're going to be working with the Lang Team, and Async WG on these.
Specifically the `async trait` notation RFC will be driven by the Async WG,
though the Keyword Generics Initiative will be advicing.

Only once those RFCs have been completed will we start looking more closely at
the `effect/.do` notation. After all: without it there would be no keywords to
be generic over!

[^async-eval]: Think
[`const_eval_select`](https://doc.rust-lang.org/std/intrinsics/fn.const_eval_select.html)
but for `?async` contexts. This is needed to enable concrete `?async` types to
function such as `?File` - because it relies on a different implementation for
`async` and non-`async` operation.

## Conclusion

And that concludes the summary of what we're looking at. We intentionally didn't
drill too deep into details because that's what the RFCs will be for. Which
speaking of: we'll be publishing those over the coming months, and once they go
through it shouldn't be long before we can bring the features to nightly. Most
of what we've discussed in this post (except `effect/.do` and specific inference
bits) has already been successfully prototyped outside of the stdlib, meaning
we have a really good sense of how it's going to work.
