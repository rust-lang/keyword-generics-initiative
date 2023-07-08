# Overloading Keyword Generics

In the previous section we saw that we cannot use `join` to await two futures
concurrently because in "base Rust" we cannot run two closures concurrently. The
capabilities introduced by the superset (async) have no counterpart in the
subset ("base Rust"), and therefore we cannot write it.

But sometimes we _do_ want to be able to specialize implementations for a
specific context, making use of the capabilities they provide. In order to do
this we need to be able to declare two different code paths, and we propose
_effect overloading_ as the mechanism to do that.

This problem is not limited to async Rust either; const implementations may want
to swap to platform-specific intrinsics at runtime, but keep using portable
instructions during CTFE. This is only a difference in implementation, and
should not require users to switch between APIs.

The way we envision _effect overloading_ to work would be similar to
specialization. A base implementation would be declared, with an overload in the
same scope using the same signature except for the effects. The compiler would
pick up on that, and make it work as if the type was written in a polymorphic
fashion. Taking our earlier example, we could imagine the `sum` function could
then be written like this:

```rust
// Sum the output of two functions:
default fn sum<T>(
    lhs: impl FnMut() -> T,
    rhs: impl FnMut() -> T
) -> T {
   lhs() + rhs()
}

async fn sum<T>(
    lhs: impl async FnMut() -> T,
    rhs: impl async FnMut() -> T
) -> T {
   let (lhs, rhs) = (lhs(), rhs()).join().await;
   lhs + rhs
}
```

We expect _effect overloading_ to not only be useful for performance: we suspect
it may also be required when defining the core (async) IO types in the stdlib
(e.g. `TcpStream`, `File`). These types carry extra fields which their base
counterparts do not. And operations such as reading and writing to them cannot
be written in a polymorphic fashion.

__While we expect a majority of ecosystem and stdlib code to be written using
_effect polymorphism_, there is a point at which implementations do need to be
specialized, and for that we need _effect overloading_.__
