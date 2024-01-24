# Effects in Rust

Rust has a number of built-ins which sure look a lot like effects. In this
section we cover what those are, how they're in use today, touch on some of the
pain-points experienced by them.

## What do we mean by "effect" in this section?

For the purpose of this section we're considering effects in the broadest terms:
"Any built-in language mechanism which triggers a bifurcation of the design
space". This means: anything which causes you to create a parallel, alternate
copy of the same things is considered an effect in this space.

This is probably not the definition we'll want to use in other sections, since
effects should probably only ever apply to functions. In this section we're
going to use "effect" as a catch-all term for "things that sure seem effect-y".
When discussing effects we'll differentiate between:

- **Scoped Effects**: which are effects which apply to functions and scopes, such
   as `async fn` which are reified as traits or types such as `impl Iterator`.
- **Data-Type Effects**: which are Effects which apply to data types, encoded as
  auto-traits. For example: the `Send` auto-trait is automatically implemented on structs
  as long as its contained types are `Send`, and marks it as "thread-safe".
