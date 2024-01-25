# Pattern Types and Backwards Compatibility

## Introduction

[Pattern types](https://gist.github.com/joboet/0cecbce925ee2ad1ee3e5520cec81e30)
are an in-progress proposal for Rust to add a limited form of refinement types /
liquid types to Rust via pattern the pattern notation. Take for example the
existing
[`AtomicBool::load`](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicBool.html#method.load)
operation. Its signature looks like this:

```rust
/// A boolean type which can be safely shared between threads.
struct AtomicBool { .. }

impl AtomicBool {
    /// Loads a value from the bool.
    pub fn load(&self, order: Ordering) -> bool { .. }
}
```

Atomics are part Rust's memory model, and are how we're able to share data
between threads. Depending on what we want to do with an atomic, we'll want to
give it a different
[`Ordering`](https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html)
argument. `Ordering` is just an enum, which has the following variants:

```rust
#[non_exhaustive]
pub enum Ordering {
    Relaxed,
    Release,
    Acquire,
    AcqRel,
    SeqCst,
}
```

For this example it doesn't exactly matter what each of these variants are for.
But what's important is that not all variants are valid arguments for
`AtomicBool::load`. Its documentation says that `SeqCst`, `Acquire`, and
`Relaxed` are valid. But if the `Release` or `AcqRel` variants are used, it will
panic at runtime.

Pattern types would in theory enable us to "shift-left" on this, by encoding the
allowed variants directly into the function's parameters. This would encode this
invariant directly via the type system, meaning we've "shifted left" from a
runtime error (e.g. we need to run tests to find the bug), to a compiler error
(e.g. we need to run `cargo check` to find the bug). Using the pattern types
draft RFC, this would look something like this:

```rust
struct AtomicBool { .. }
impl AtomicBool {
    pub fn load(&self,
        order: Ordering is Ordering::SeqCst | Ordering::Acquire | Ordering::Relaxed
    ) -> bool { .. }
}
```

## Backwards-compatibility issues

Moving checks from runtime to compile-time is generally considered a good thing,
as it shortens the time it takes to discover bugs. But when we use pattern types
as inputs to functions, we're *constraining* the input space from all variants
to just the legal variants. Take for example the following code, which is legal
to write today.

```rust
pub fn load_wrapper(order: Ordering, bool: &AtomicBool) -> bool {
    bool.load(order)
}
```

This code does not know about pattern types, and Rust's backwards-compatibility
guarantees require that it keeps compiling in future releases of the compiler.
That means that changing `AtomicBool::load` to require taking pattern types as
its input would be a backwards-incompatible change. So we cannot just do that.

One alternative would be to create a duplicate version of `AtomicBool` which
does know how to take pattern types. But duplicating code just to improve it
feels pretty bad - instead it would be nice if we could update existing
functions without it leading to breaking changes.

## Resolving the backwards-compatibility issues

On Zulip people have brought up the idea of using editions to resolve these
issues. That might be possible, but it would mean a clean break between code
written on an older edition, and code written on a newer edition. And while we
can leverage editions to change defaults in the language, this kind of break
feels like it would push against the intended goal of maintaining compatibility
between editions.

The idea underlying it seems right though: we do want some way to express
*modality* in our type system. We've already done this before using the `const`
effect. Functions tagged as `const` can be evaluated either during compilation
or at runtime. And it's backwards-compatible to take an existing runtime-only
`fn` and change it to a `const fn`.

```rust
fn meow() -> &'static str { "meow" }         // 1. The base `fn meow`
const fn meow() -> &'static str { "meow" }   // 2. Changing `meow` to a `const fn` is backwards-compatible
```

We could do something very similar with pattern types as well. The base
mechanism for this is to define a function which can be compiled in one of two
modes:

1. **Invariants are evaluated at compile-time**: The pattern types are evaluated by
  the compiler according to the pattern type RFC. A compiler error is raised if
  the pattern's invariants are violated.
2. **Invariants are evaluated at runtime**: The pattern types are converted to a
  sequence of assertions, and evaluated at runtime.

The translation here from pattern types to runtime assertions should be fairly
mechanical. We could imagine some notation which signals that while a function
may declare pattern types, the caller has an option to either evaluate them at
runtime or during compilation. Taking our earlier `AtomicBool::load` example, we
could imagine something like this:

```rust
struct AtomicBool { .. }
impl AtomicBool {
    #[maybe(pattern_types)]
    pub fn load(&self,
        order: Ordering is Ordering::SeqCst | Ordering::Acquire | Ordering::Relaxed
    ) -> bool { .. }
}
```

With this notation, all existing uses of `AtomicBool::load` would continue
working. But optionally it could be called using pattern types, which would be
evaluated at compile-time. Depending on which variant of the function is
selected, the lowering of the function would change. Desugared, this would
roughly look like this:

```rust
/// `AtomicBool::load` using compile-time checks
pub fn load(&self,
    order: Ordering is Ordering::SeqCst | Ordering::Acquire | Ordering::Relaxed
) -> bool { .. }

/// `AtomicBool::load` using runtime assertions
pub fn load(&self, order: Ordering) -> bool {
    assert!(matches!(order, Ordering::SeqCst | Ordering::Acquire | Ordering::Relaxed));
    ..
}
```

## TODO: Effect logic and notation

- in its base there are four states possible: `always | never | maybe | unknown`
- the relation to subtyping and return types will determine which states of this system we'll want to encode
- unclear what the benefits are for a strictly "always subtyping" notation
- in practice we'll want to independently gate the stabilization of pattern
  types for existing stdlib APIs - which means we need a labeling system in the compiler

## How widespread is this?

Maintaining strict backwards-compatibility is primarily a concern for the Rust
stdlib. While it might be difficult to create major versions for certain other
codebases, the Rust stdlib is in the unique position that it is both used by
everyone, and we can never break existing APIs. So when we're looking at using
pattern types in input positions, it's okay to assume the Rust stdlib will be
the main user of it. To date we know of at least the following APIs which would
want to leverage pattern types as inputs:

- __number primitives__: Number types in Rust expose a wide range of operations.
  Take for example a look at the [`u8`
  type](https://doc.rust-lang.org/std/primitive.u8.html). It exposes around 20
  operations per type which will panic if certain number ranges are passed.
- __atomics__: this is the example we've been using in this post. Atomic
  operations take an `Ordering` enum, where each operation can only take certain
  variants of that enum. Being able to check that during compilation would be a
  boon.
- __iterator methods__: For example
[`Iterator::step_by`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.step_by)
currently takes a `usize`, but would want to take a `usize is 0..`. The same is
true for the unstable `Iterator::array_chunks` and `Iterator::map_windows`.
