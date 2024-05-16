# Immovable types

In today's Rust we can express referential stability for types via "pinned
references". These types which use the `Pin` type, and whose memory location is
fixed and cannot be moved, enabling things like self-references in structs. This
system comes with a number of rules, such as: once a type has been pinned it
must remain pinned for the remainder of its existence. Unless a type implements
`Unpin`, in which case these rules don't apply.

The most important aspect of Rust's pinning system is what are called: "pin
projections" - or a way to guarantee referential stability for individual fields
of a struct, rather than the entire struct. These rules largely live outside of
Rust's type system, and require `unsafe` to be enforced. Though in practice most
of the ecosystem chooses to use the `pin-project` crate instead, which provides
a language feature-like macro that statically upholds the rules. However, the macro comes with heavy restrictions (cannot use methods), and 
bringing it into the language [may not be
feasible](https://blog.yoshuawuyts.com/safe-pin-projections-through-view-types/).

Finally, while pinning was designed to enable self-referential types, Rust does
not provide any facilities to make this easy. Meaning that there are plenty of
cases where an `async {}` block or function cannot be lowered into the
equivalent `Future` state machine code without making heavy use of `unsafe {}`.
Instead a potentially more fruitful avenue might be to rethink the existing
pinning system entirely. 

Immovable types provide an alternative to pinning: being designed from the
ground up to enable safe, self-referential types. This is possible because
immovable types directly extend the language, so they can integrate with all
language features. And by being polymorphic, we can introduce them in a
backwards-compatible way.

## The base design

Let's say we didn't stabilize the `Future` trait using `Pin`. By itself that
means the trait probably would have looked like this:

```rust
trait Future {
    type Output;
    fn poll(&mut self, cx: &mut Context<'_>) -> Poll<Self::Output>;
    //      ^ instead of `self: Pin<&mut Self>`
}
```

This describes a trait where types can't be self-referential. That means the
following code would not be allowed:

```rust
async {
    let vec = vec![1, 2, 3, 4];
    foo().await;
    dbg!(vec[0]); // ❌ compiler error; can't hold a reference across an .await point
}
```

In order to make this work, we'd need to enable `Future` to be able to work with
self-referential types. Which in order to be self-referential must preserve
referential stability. And the way we're proposing we do that is by using an
`immovable` annotation. Or well, actually, there are three core pieces required
for this to work:

1. The ability to declare "immovable types" using an effect-like notation
2. The ability to construct types in-place, so that we can create immovable
   types in a memory location outside of the function. This enables constructors.
3. An `IntoFuture` trait, so that a type can be movable until the moment it
   needs to be used - at which point it can be converted to be immovable.

Let's walk through how we could leverage that to create "immovable types" which
provide equivalent semantics to today's `Pin` system.

If you haven't yet read the section on [immovable types](./immovable-types.md),
please do so now. Much of the conversation around shapes and tradeoffs around
API design is discussed there.

```rust
//! Create an immovable type.
immovable struct Immovable<immovable T, U> {
    immovable pinned: T
    unpinned: U,
}

impl<immovable T, U> immovable Immovable<immovable T, U> {
    fn new(t: impl Into<T>, u: U) -> inplace Self {
        Self {
            pinned: t.into(),
            unpinned: U,
        }
    }

    fn method(&mut self) {
        let _: &T = this.pinned; // reference to the immovable field
        let _: &U = this.unpinned; // reference to the movable field
    }
}
```
```rust
//! Create a type which can be initially moved, but eventually has a fixed
//! memory location.
struct Movable<IntoT, immovable T, U>
where
    IntoT: Into<Output = immovable T>,
{
    pinned: IntoT,
    unpinned: U,
}

impl<IntoT, immovable T, U> Into<Immovable<T, U>> for Immovable<T, U>
where
    IntoT: Into<Output = immovable T>
{
    fn into(self) -> inplace Immovable<T, U> {
        Immovable::new(self.pinned, self.unpinned)
    }
}
```

TODO: `async fn` doesn't return `Future`, but returns `IntoFuture`.
