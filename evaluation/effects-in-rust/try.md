# Fallibility (Scoped Effect)

## Feature Status

todo

## Description

todo

## Refinements

| Modifier            | Description                                           |
| ------------------- | ----------------------------------------------------- |
| `Option<T>`         | Used to describe optional values                      |
| `Result<T, E>`      | Used to describe errors or success values             |
| `ControlFlow<B, C>` | Used to represent control-flow loops                  |
| `Poll<T>`           | Used to describe the state of `Future` state machines |

While the reification of the fallibility effect in bounds ought to be `impl
Try`, it more commonly is the case that we see concrete types used.

## Feature categorization

| Position    | Syntax                            |
| ----------- | --------------------------------- |
| Create      | `try`                             |
| Yield       | `throw`                           |
| Forward     | `?`                               |
| Consume     | `match` / `fn main()` † |
| Reification | `impl Try`                        |

> † `fn main` implements effect polymorphism over the fallibility effect
> by making use of the [`Termination` trait]. It stands to reason that _if_ we
> had a `try` notation for functions, that it should be possible to write
> `try fn main` which desugars to a `Result` type being returned.
 
[`Termination` trait]: https://doc.rust-lang.org/std/process/trait.Termination.html


## Interactions with other effects

### Asynchrony
### Compile-time Execution
### Fallibility
### Iterativity
### May Panic
### Memory-Unsafety
### Must-not Move
### Object-Safety
### Ownership
### Thread-Safety
