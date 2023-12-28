- Feature Name: `effect-generic-trait-decls`
- Start Date: (2024-01-01)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

This RFC introduces Effect-Generic Trait Declarations. These are traits which
can be generic over Rust's built-in effect keywords such as `async` and `const`.
Unlike the current status quo where manually duplicting traits to different
effects is the norm, this enables the type system and compiler to directly
reason about the presence of effects.

```rust
/// A basic "into" trait, as defined in the stdlib today.
trait Into<T>: Sized {
    fn into(self) -> T;
}

/// A hypothetical async version of the `Into` trait, copying the interface but
/// adding `async` in front of the methods.
trait AsyncInto<T>: Sized {
    async fn into(self) -> T;
}

/// With this RFC we could define a single implementation, which can be
/// implemented as either the sync or async variant, sharing a single
/// implementation.
#[maybe(async)] trait Into<T>: Sized {
    #[maybe(async)] fn into(self) -> T;
}
```

# Motivation
[motivation]: #motivation

## TODO: Language-Level Coherence

- Duplication across effects has to exist somewhere: right now it's mostly
  user-space.
- This is error-prone: we got the API of `TryInto` wrong lol
- Tokio went off the rails and did its own thing, creating subtly different APIs
  which isn't a great reason
- We got it right with `const` though, gradually constifying the stdlib with
  relatively little additional burden for stdlib maintainers.
- Think of it as portability: shared core, extensions only where needed

## TODO: Exponential API Surface

- Every single generic
- Every single function (const)
- Every single trait (const, try, async, gen)
- All interacting with each other

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## TODO: trait definitions

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
