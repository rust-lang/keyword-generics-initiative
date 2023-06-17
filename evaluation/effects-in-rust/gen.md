# Iteration (Scoped Effect)

## Feature Status

The `Iterator` trait has been stable in Rust since 1.0, but the generator syntax
is currently _unstable_. This document will assume that generators are created
with the `gen` keyword, but that's for illustrative purposes only.

## Description

todo

## Technical Overview

| Position    | Syntax          |
| ----------- | --------------- |
| Create      | `gen`           |
| Yield       | `yield`         |
| Forward     | N/A             |
| Consume     | `for..in`       |
| Reification | `impl Iterator` |

## Refinements

| Modifier      | Description                                           |
| ------------- | ----------------------------------------------------- |
| step          | Has a notion of successor and predecessor operations. |
| trusted len † | Reports an accurate length using `size_hint`.         |
| trusted step  | Upholds all invariants of `Step`.                     |
| double-ended  | Is able to yield elements from both ends.             |
| exact size †  | Knows its exact length.                               |
| fused         | Always continues to yield `None` when exhausted.      |

> † The difference between `TrustedLen` and `ExactSizeIterator` is that
>  `TrustedLen` is marked as `unsafe` to implement while `ExactSizeIterator` is
>  marked as _safe_ to implement. This means that if `TrustedLen` is implemented,
>  you can rely on it for safety purposes, while with `ExactSizeIterator` you
>  cannot.

## Positions Available

| Position            | Available | Example                          |
| ------------------- | --------- | -------------------------------- |
| Manual trait impl   | ✅        | `impl Iterator for Cat {}`       |
| Free functions      | ❌        | `gen fn meow() {}`               |
| Inherent functions  | ❌        | `impl Cat { gen fn meow() {} } ` |
| Trait methods       | ❌        | `trait Cat { gen fn meow() {} }` |
| Trait declarations  | ❌        | `gen trait Cat {}`               |
| Block scope         | ❌        | N/A                              |
| Argument qualifiers | ❌        | `fn meow(cat: impl gen Cat) {}`  |
| Drop                | ❌        | `impl gen Drop for Cat {}`       |
| Closures            | ❌        | `gen ǀǀ  {}`                     |
| Iterators           | ❌        | `for cat in cats {}`             |

## Interactions with other effects

### Asynchrony



| Overview                       | Description                                                                                                                        |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| Composition                    | iterator of futures                                                                                                                |
| Description                    | Creates an iterator of futures. The future takes the iterator by `&mut self`, so only a single future may be executed concurrently |
| Example                        | [`AsyncIterator`][ai]                                                                                                              |
| Implementable as of Rust 1.70? | No, async functions in traits are unstable                                                                                         |

[ai]: https://docs.rs/async-iterator/latest/async_iterator/

### Fallibility

| Overview                       | Description                                                        |
| ------------------------------ | ------------------------------------------------------------------ |
| Composition                    | iterator of tryables                                               |
| Description                    | Creates an iterator of tryables, typically an iterator of `Result` |
| Example                        | [`FallibleIterator`][fi]                                           |
| Implementable as of Rust 1.70? | No, try in traits is not available                                 |

[fi]: https://docs.rs/fallible-iterator/latest/fallible_iterator/trait.FallibleIterator.html

### Compile-time Execution

| Overview                       | Description                                                    |
| ------------------------------ | -------------------------------------------------------------- |
| Composition                    | const iterator                                                 |
| Description                    | Creates an iterator which can be iterated over at compile-time |
| Example                        | N/A                                                            |
| Implementable as of Rust 1.70? | No, const traits are unstable                                  |

### Thread-Safety

| Overview                       | Description                                                                 |
| ------------------------------ | --------------------------------------------------------------------------- |
| Composition                    | iterator of tryables                                                        |
| Description                    | Creates an iterator whose items which can be sent across threads            |
| Example                        | `where I: Iterator<Item = T>, T: Send`                                      |
| Implementable as of Rust 1.70? | Yes, as a bound on use. And by unit-testing the `Send` auto-trait on decls. |

### Immovability
| Overview                       | Description                                               |
| ------------------------------ | --------------------------------------------------------- |
| Composition                    | an iterator which takes `self: Pin<&mut Self>`            |
| Description                    | An iterator which itself holds onto self-referential data |
| Example                        | N/A                                                       |
| Implementable as of Rust 1.70? | Yes                                                       |

### Unwinding

| Overview                       | Description                                      |
| ------------------------------ | ------------------------------------------------ |
| Composition                    | iterator may panic instead of yield              |
| Description                    | Creates an iterator which may panic              |
| Example                        | `Iterator` (may panic by default)                |
| Implementable as of Rust 1.70? | Yes, but cannot opt-out of "may panic" semantics |
