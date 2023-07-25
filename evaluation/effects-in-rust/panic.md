# Unwinding (Scoped Effect)

## Feature Status

todo

## Description

todo

## Refinements

| Modifier | Description |
| -------- | ----------- |

The `panic` effect currently has no refinements.

## Feature categorization

| Position    | Syntax                     |
| ----------- | -------------------------- |
| Effect      | N/A                        |
| Yield       | `panic!`                   |
| Apply       | `foo()` / `resume_unwind`  |
| Consume     | `catch_unwind` / `fn main` |
| Reification | N/A                        |

Panics differ from all other control-flow oriented effects because every
function is assumed to potentially panic. This means that the syntax to forward
a panic from a function is just a regular function call. Panics are not
represented in the type system, instead they exist as a property _outside_ of
it.

## Interactions with other effects

### Asynchrony
### Compile-time Execution
### Fallibility
### Iteration
### Unwinding
### Memory-Safety
### Immovability
### Object-Safety
### Ownership
### Thread-Safety
