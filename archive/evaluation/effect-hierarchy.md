# Implications of the effect hierarchy

One implication of the subset-superset relationship is that code which is
generic over effects will not be able to use all functionality of the superset
in the subset case. Though it will need to use the _syntax_ of the superset.

Take for examle the following code. It takes two async closures, awaits them,
and sums them:

```rust
// Sum the output of two async functions:
~async fn sum<T>(
    lhs: impl ~async FnMut() -> T,
    rhs: impl ~async FnMut() -> T
) -> T {
   let lhs = lhs().await; 
   let rhs = rhs().await; 
   lhs + rhs
}
```

One of the benefits of async execution is that we gain _ad-hoc concurrency_, so
we might be tempted to perform the comptutation of `lhs` and `rhs` concurrently,
and summing the output once both have completed. However this should not be
possible solely using effect polymorphism since the generated code needs to work
in both async and non-async contexts.


```rust
// Sum the output of two async functions:
~async fn sum<T>(
    lhs: impl ~async FnMut() -> T,
    rhs: impl ~async FnMut() -> T
) -> T {
   let (lhs, rhs) = (lhs(), rhs()).join().await;
   //                             ^^^^^^^
   // error: cannot call an `async fn` from a `~async` context
   // hint: instead of calling `join` await the items sequentially
   //       or consider writing an overload instead
   lhs + rhs
}
```

And this is not unique to `async`: in maybe-`const` contexts we cannot call
functions from the super-context ("base Rust") since those cannot work during
`const` execution. This leads to the following implication: __Conditional effect
implementations require the syntactic annotations of the super-context, but
cannot call functions which exclusively work in the super-context.__
