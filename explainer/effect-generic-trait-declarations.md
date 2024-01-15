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
impl async Into<Loaf> for Cat {
    async fn into(self) -> Loaf {
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

## Effects gone wrong

Evolving a programming language and stdlib is pretty difficult. We have to pay
close attention to details. And in Rust specifically: once we make a mistake
it's pretty hard to roll back. And we've made mistakes with effects in the past,
which we now have to work with.

In Rust 1.34 we stabilized a new trait: `TryInto`. This was supposed to be the
fallible version of the `Into` trait, containing a new associated type `Error`.
However since Rust 1.0 we've also hadd the
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

## TODO: trait implementations

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## TODO: Carried vs Uncarried Effects

- const is uncarried
- async, gen, try are carried


## TODO: Lowering of Uncarried Effects

- relies on language built-ins
- but still using const bools

## TODO: Lowering of Carried Effects

```rust
// Trait definition
#[maybe(async)] trait Into<T>: Sized {
    #[maybe(async)] fn into(self) -> T;
}

// Lowered trait definition
trait Into<T, const IS_ASYNC: bool>: Sized {
    type Ret = T;
    fn into(self) -> Self::Ret;
}

// Lowered async trait impl
impl Into<Loaf, true> for Cat { // IS_ASYNC = true
    type Ret = impl Future<Output = T>;
    fn into(self) -> Self::Ret;
}

// Lowered non-async trait impl
impl Into<Loaf, false> for Cat { // IS_ASYNC = false
    type Ret = T;
    fn into(self) -> Self::Ret;
}
```

## TODO: Stability Attributes

- methods such as `map` will not be "maybe-async" since it needs support for types

## TODO: Unique Selection

- for now only one of a kind should be allowed per type

## TODO: Backwards Compatibility

- yep it's backwards compatible

# Drawbacks
[drawbacks]: #drawbacks

# Prior art
[prior-art]: #prior-art

## TODO: Swift

- async polymorphism
- `rethrow`

## TODO: C++

- noexcept
- constexpr

## TODO: Koka

- do research

## TODO: Rust

- const fn

## TODO: Zig

- maybe async functions

# Unresolved questions
[unresolved-questions]: #unresolved-questions

## TODO: Syntax

# Future possibilities
[future-possibilities]: #future-possibilities

## TODO: Effect-Generic Types and Bodies

- types
- functions

## TODO: Effect Sets

- named effect sets
- unify `core` and `std` via sets

## TODO: Effectful Drop

- maybe async drop
- maybe try drop
