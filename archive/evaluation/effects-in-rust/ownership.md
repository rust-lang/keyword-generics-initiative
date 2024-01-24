# Ownership (Data-Type Effect)

## Description

## Feature Status

## Feature categorization

| Position    | Syntax |
| ----------- | ------ |
| Effect      |        |
| Yield       |        |
| Apply       |        |
| Consume     |        |
| Reification |        |

## Positions Available

| Position            | Available | Example                            |
| ------------------- | --------- | ---------------------------------- |
| Manual trait impl   | ✅        | `impl Future for Cat {}`           |
| Free functions      | ✅        | `async fn meow() {}`               |
| Inherent functions  | ✅        | `impl Cat { async fn meow() {} } ` |
| Trait methods       | ⏳         | `trait Cat { async fn meow() {} }` |
| Trait declarations  | ❌        | `async trait Cat {}`               |
| Block scope         | ✅        | `fn meow() { async {} }`           |
| Argument qualifiers | ❌        | `fn meow(cat: impl async Cat) {}`  |
| Data types †        | ❌        | `async struct Cat {}`              |
| Drop †              | ❌        | `impl async Drop for Cat {}`       |
| Closures            | ❌        | `async ǀǀ  {}`                     |
| Iterators           | ❌        | `for await cat in cats {}`         |

## Refinements

| Modifier | Description |
| -------- | ----------- |

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

## References

- [Keywords II: Const Syntax](https://blog.yoshuawuyts.com/const-syntax/)
