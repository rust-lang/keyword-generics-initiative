# Unleakable Types

## A trait-based system for unleakable types

In the [Linear Types
One-Pager](https://blog.yoshuawuyts.com/linear-types-one-pager/) post Yosh
presented a system for types which cannot be leaked. This showed how by
introducing a new auto-trait `Leak`, we could construct a system that would
prevent types from being leaked.

By preventing types from being leaked, destructors would be guaranteed to run -
which would give allow types in Rust to uphold linear type invariants. Meaning:
destructors could be relied on for the purposes of safety, because some code
will always be run when a type goes out of scope.

The way to think about this system is as follows:

1. We define a new unsafe auto-trait named `Leak`
2. All bounds take an implicit + Leak bound, like we do for `+ Sized`.
3. Certain functions such as `mem::forget` will always keep taking `+ Leak` bounds.
4. Functions which want to opt-in to linearity can take `+ ?Leak` bounds.
5. Types which want to opt-in to linearity can implement `!Leak` or put a `PhantomLeak` type in a field.

In code we could see this system expressed as follows:

```rust
// Define the trait and create a blanket impl for all types.
// The language would automatically add `+ Leak` bounds to all bounds.
auto trait Leak;
impl<T> Leak for T {}

// Types are by default assumed to be leakable.
struct Leakable;

// Mark a type as unleakable, guaranteeing destructors are run
struct Unleakable;
impl !Leak for Unleakable {}

// A function which requires types implement `Leak`.
// Here `T: Leak` bounds would be assumed by default.
fn will_leak<T>(value: T) {..}

// A function which operates on types which may or may not leak.
// We're using `?Leak` to opt-out of the automatic `+ Leak` bound.
fn may_leak<T: ?Leak>(value: T) -> T {..}
```

## The limitation of auto-traits

In the challenges section of [Linear Types
One-Pager](https://blog.yoshuawuyts.com/linear-types-one-pager/) post, Yosh
remarks the following:

> We should look at an alternate formulation of these bounds by treating them
> as built-in effects. That would allow us to address the issues of versioning,
> visual noise, etc. in a more consistent and ergonomic way. But that's not a
> requirement to start testing this out.

The limitations of auto-traits are well-documented, and nobody would be excited
by the prospect of introducing `+ ?Leak` bounds to virtually every bound. For
that reason there was a recommendation to explore alternate effect-based
formulations instead.

A concrete example of a limitation for `Leak` as an auto-trait is provided by
Saoirse in their posts "Changing the rules of Rust", "Follow up to "Changing
the rules of Rust", and "Generic trait methods and new auto traits". They
provide an example equivalent to the following:

```rust
#[edition = 2027]
crate may_leak {
    #[leak_compatible]  // ← allows bounds to optionally add `+ Leak`
    pub trait MayLeak {
        fn may_leak<T>(input: T);
    }
}

#[edition = 2024]
crate will_leak {
    pub struct WillLeak;
    impl super::may_leak::MayLeak for WillLeak {
        fn may_leak<T>(input: T) { // ← takes an implicit `+ Leak` bound
            core::mem::forget(input);
        }
    }
}

#[edition = 2027]
crate may_not_leak {
    struct Unleakable;
    impl !Leak for Unleakable {}
    pub fn may_not_leak<T: #[no_leak] super::may_leak::MayLeak>() { // ← disables the optional `+ Leak` bound
        T::may_leak(Unleakable);
    }
}

// The edition doesn't matter for this function.
fn main() {
    may_not_leak::may_not_leak::<will_leak::WillLeak>();
}
```

Under the rules we provided earlier, when we pass `WillLeak` to `may_not_leak`
it should yield a compile-error. This ends up trying to pass a type which is
`!Leak` to a function which takes an implicit `+ Leak` bound, which shouldn't
compile.

The limitations of this approach very clearly show up once we consider how this
would be rolled out in practice. Every API which would want to types optionally
leaking would need to add a `+ ?Leak` or `#[maybe(leak)]` annotation to every
parameter which may ever want to leak. This system mixes bespoke attributes
together with auto-traits to create a system very similar to that of `const`.

## Reformulating leaking as an effect

The system of `#[leak_compatible]` and `#[no_leak]` annotations presented is a
bespoke encoding of the general system covered by effect generics. Fundamentally
both effects and trait bounds can be in one of three states:

- **required**: covered earlier by the implicit `+ Leak` bound, and by `effect T` under effect generics.
- **optional**: covered earlier by the `#[leak_compatible]` annotation, and `#[maybe(effect)]` under effect generics.
- **absent**: covered by the earlier `#[no_leak]` annotation, and `#[no(effect)]` under effect-generics.

When discussing effects, they can either be opt-in (e.g. `async`, `try`) where
we assume capabilities are not present unless we state we want them. Or opt-out
(e.g. `const`) where we assume capabilities are present, and we opt-out of them
go gain some other property. Currently Rust code may always leak, and what we're
taking away is the ability to leak. Likely the right approach here would be to
name the effect `leak`, and allow people to write both `leak T` and `#[no(leak)]
T`. Under these rules the system would look like this:

```rust
// Mark a type as unleakable, guaranteeing destructors are run
#[not(leak)]
struct Unleakable;

// A type which may be leaked. `leak struct` is assumed,
// but can be written out for clarity.
struct Leakable;
leak struct Leakable;

// A function which requires all arguments can be leaked.
// `leak fn` is assumed by default, but may be written out for clarity.
fn will_leak<T>(value: T) {..}
leak fn will_leak<T>(value: T) {..}

// A function which operates on types which may or may not leak.
#[maybe(leak)]
fn may_leak<T>(value: T) -> T {..}
```

Applying this sytem to the longer example would look like this:

```rust
#[edition = 2027]
crate may_leak {
    #[maybe(leak)] // ← indicates this trait may of may not leak
    pub trait MayLeak {
        fn may_leak<#[maybe(leak)] T>(input: T); // ← indicates the type in this bound may or may not leak
    }
}

#[edition = 2024]
crate will_leak {
    pub struct WillLeak;
    impl super::may_leak::MayLeak for WillLeak {
        fn may_leak<T>(input: T) { // ← is assumed to be a `leak fn`; assumes `leak T`
            core::mem::forget(input);
        }
    }
}

#[edition = 2027]
crate may_not_leak {
    #[not(leak)] // ← this type may not be leaked
    struct Unleakable;

    #[not(leak)] // ← states all bounds take `#[no(leak)]`
    pub fn may_not_leak<T: super::may_leak::MayLeak>() {
        T::may_leak(Unleakable);
    }
}

// The edition doesn't matter for this function.
fn main() {
    may_not_leak::may_not_leak::<will_leak::WillLeak>();
}
```

While similar to the previous design, this version applies a consistent logic
and naming to the bounds and ascriptions following the system laid out by
effect-generics. This would make it so introducing linearity into the type
system wouldn't be its own design with its own attributes, but part of a
consistent framework by which we can evolve the language.

## Changing defaults across editions

An alternative design for `#[not(leak)]` would be to follow the design of the
const keyword more closely, and introduce a positive effect. Perhaps something
like a `linear T` / `linear fn`. However if we assume we will eventually be
successful in the transition to adoption linearity, this would put us in the
awkward position where the ideal system would end up with more ascriptions.

The beauty of `#[not(leak)]` as the discriminant for linearity is that we could
eventually change the default across editions to not assume leaking is provided, and only if you
want to opt-in to being able to leak you have to add it to your functions. This
is currently already the same for keywords such as `async` and `gen`. Under
these rules, code would be able to change like this:

```rust
// All types are assumed to be unleakable by default
struct Unleakable;

// A type which may be leaked.
leak struct Leakable;

// A function which requires all types can be leaked.
leak fn will_leak<T>(value: T) {..}

// A function which operates on types which may or may not leak.
fn may_leak<T>(value: T) -> T {..}
```

Hypothetically a third kind of function could be described where all arguments
are assumed not to leak. But just like we don't (yet?) have a clear use case for
const-only functions, it's unclear that no-leak-only functions would be
beneficial. That said, effect generic provides a consistent framework which
would enable for these to be introduced.

## On the choice of keywords

None of the keywords or syntax proposed in this post are intended to be final.
The emphasis of this post is on the semantics of the system and showing how we
can express linear types in a consistent way by leveraging the framework of
effect generics.

## References

- [The pain of Real Linear Types in Rust](https://faultlore.com/blah/linear-rust/)
- [Must move types](https://smallcultfollowing.com/babysteps/blog/2023/03/16/must-move-types/)
- [Linearity and Control](https://blog.yoshuawuyts.com/linearity-and-control/)
- [Linear Types One-Pager](https://blog.yoshuawuyts.com/linear-types-one-pager/)
- [Async destructors, async genericity and completion futures](https://sabrinajewson.org/blog/async-drop)
- [Changing the Rules of Rust](https://without.boats/blog/changing-the-rules-of-rust/)
- [Follow up to "Changing the rules of Rust"](https://without.boats/blog/follow-up-to-changing-the-rules-of-rust/)
- [Generic trait methods and new auto traits](https://without.boats/blog/generic-trait-methods-and-new-auto-traits/)
