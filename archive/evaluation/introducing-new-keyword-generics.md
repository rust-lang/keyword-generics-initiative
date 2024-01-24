# Adding new keyword generics

```rust
const fn foo() { // maybe const context
    let file = fs::open("hello").unwrap();
    // compile error! => `fs::open` is not a maybe const function!
}

~base fn foo() { // assume `const` as the default; invert the relationship
    let file = fs::open("hello").unwrap();
    // compile error! => `fs::open` is
    // a base function which cannot be
    // called from a maybe base context
}

~async fn foo() {
    let file = my_definitely_async_fn().await;
    // compile error!
}
```

```rust
fn foo<effect F: const>(f: impl F * Fn() -> ()) {
    f();
}
fn foo<effect F: const>(f: impl effect<F> Fn() -> ()) {
    f();
}

// compile error!
// effect `F` is maximally inclusive!
// missing `.await`

// maximally inclusive effects are not forward compatible! - once
// we add a new effect existing code will not compile!
// The calling convention may change each time we add a new effect!

fn main() {
    foo(some_fn); // Infer all effects to Not*
}
```

Adding new effects to the language does not break anyone, because effects must
be opted in. Adding a new effect to the opt-in effect generics of a function
will break callers that infer the effect to be required.

Editions can add new effects to the list of defaults. This is not a breaking
change because calling crates can stay on old editions, even if the lib crate
got updated to a newer edition. THe lower edition crates don't see the defaults
and turn them off.
