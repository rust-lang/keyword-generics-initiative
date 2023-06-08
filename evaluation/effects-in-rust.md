# Effects in Rust

Rust has a number of built-in effects already. In this section we cover what
those effects are, how they're in use today, touch on some of the pain-points
experienced by them.

## Effect Kinds

Rust has roughly two kinds of effects:

- Effects which apply to functions and scopes, such as `async fn` which are
reified as traits or types such as `impl Iterator`.
- Effects which apply to data types, encoded as auto-traits. For example: the `Send` trait which is used for the thread-safety effect.

Because in Rust functions can be passed around as types, it means that effects
which apply to types also apply to functions since functions in Rust _are_ types
(e.g.  `impl FnMut`).
