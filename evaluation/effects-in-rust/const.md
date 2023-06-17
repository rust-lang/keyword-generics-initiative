# Compile-time Execution (Scoped Effect)
## Description

The `const` keyword marks functions as "is allowed to be evaluated during
compilation". When used in scope position its meaning changes slightly to: "this
_will_ be evaluated during compilation". There is no way to declare "must be
evaluated at compilation" functions, causing the meaning of "const" to be
context-dependent.

|                                   | declaration          | usage                         |
| --------------------------------- | -------------------- | ----------------------------- |
| **keyword never applies**         | `fn meow() {}`       | `fn hello() { meow() }`       |
| **keyword always applies**        | -                    | `const CAT: () = {};`         |
| **keyword conditionally applies** | `const fn meow() {}` | `const fn hello() { meow() }` |

## Feature Status

The `const` feature is integrated in a lot of the stdlib and ecosystem already,
but it's notoriously missing any form of const-traits. Because a lot of Rust's
language features make use of traits, this means const contexts have no access
to iteration, `Drop` handlers, closures, and more.

## Feature categorization

| Position    | Syntax                         |
| ----------- | ------------------------------ |
| Create      | `const fn`                     |
| Yield       | N/A                            |
| Forward     | automatic                      |
| Consume     | `const {}`, `const X: Ty = {}` |
| Reification | N/A                            |

## Positions Available

| Position            | Available | Example                            |
| ------------------- | --------- | ---------------------------------- |
| Manual trait impl   | ❌        | N/A                                |
| Free functions      | ✅        | `const fn meow() {}`               |
| Inherent functions  | ✅        | `impl Cat { const fn meow() {} } ` |
| Trait methods       | ⏳         | `trait Cat { const fn meow() {} }` |
| Trait declarations  | ❌        | `const trait Cat {}`               |
| Block scope         | ✅        | `fn meow() { const {} }`           |
| Argument qualifiers | ❌        | `fn meow(cat: impl const Cat) {}`  |
| Data types          | ❌        | `const struct Cat {}`              |
| Drop                | ❌        | `impl const Drop for Cat {}`       |
| Closures            | ❌        | `const ǀǀ {}`                      |
| Iterators           | ❌        | `for cat in cats {}`               |

## Refinements

There are currently no refiments to the compile-time execution effect.

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
- [Const as an auto-trait](https://without.boats/blog/const-as-an-auto-trait/)
