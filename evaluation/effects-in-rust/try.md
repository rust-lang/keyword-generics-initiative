# Fallibility

## Feature Status

todo

## Description

todo

## Refinements

| Modifier | Description |
| -------- | ----------- |

The `try` effect currently has no refinements.

## Feature categorization

| Position    | Syntax                            |
| ----------- | --------------------------------- |
| Create      | `try`                             |
| Yield       | `throw`                           |
| Forward     | `?`                               |
| Consume     | `match` / `fn main() -> Result` † |
| Reification | `impl Try`                        |

> † Interestingly `fn main` implements effect polymorphism over the "try" effect
> by making use of the [`Termination` trait]. It stands to reason that _if_ we
> had a `try` notation for functions, that it should be possible to just write
> `try fn main` because it matches the try-desugaring.
 
[`Termination` trait]: https://doc.rust-lang.org/std/process/trait.Termination.html


## Interactions with other effects

### Asynchrony
### Fallibility
### Compile-time Execution
### Multiplicity
### Thread-Safety
### Must-not Move
### Unwinding
### Thread Safety
### Fallibility
