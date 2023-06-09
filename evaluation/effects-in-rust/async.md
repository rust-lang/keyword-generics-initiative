# Asynchrony

## Feature Status

`async/.await` in Rust is considered "MVP stable". This means the reification of
the effect is stable, and both the `async` and `.await` keywords exist in the
language, but not all keyword positions are available yet.

## Description

todo

## Feature categorization

| Position    | Syntax                                  |
| ----------- | --------------------------------------- |
| Create      | `async`                                 |
| Yield       | N/A                                     |
| Forward     | `.await`                                |
| Consume     | `thread::block_on` †, `async fn main` ‡ |
| Reification | `impl Future`                           |

> † `thread::block_on` is not yet part of the stdlib, and only exists as a
> library feature. An example implementation can be found in the
> [`Wake`](https://doc.rust-lang.org/std/task/trait.Wake.html#examples) docs.

> ‡ `async fn main` is not yet part of the language, and only exists as a
> proc-macro extension as part of the ecosystem. It chiefly wraps the existing `fn
> main` logic in a `thread::block_on` call.

## Refinements

| Modifier            | Description                          |
| ------------------- | ------------------------------------ |
| cancellation-safe † | Has no associated future-local state |

> † The name "cancellation-safe" is in quotes because currently it's more like a
> term of art than an agreed-upon term. For example: it is yet to be encoded 
> in the type system anywhere. And when we do, we probably would want to call it
> something else.

A `FusedFuture` super-trait also exists, but it does not meaningfully feel like
a modifier of the "async" effect. It only adds an `is_terminated` method which
returns a bool. It does not inherently change the semantic functioning of the
underlying `Iterator` trait, or enhance it with behavior which is otherwise
absent. This is different from e.g. `FusedIterator` which says something about
the behavior of the `Iterator::next` function.

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

> † In non-async Rust if you place a value which implements `Drop` inside of
> another type, the destructor of that value is run when the enclosing type is
> destructed. This is called _drop-forwarding_. In order for drop-forwarding to
> work with async drop, some form of "async value" notation will be required.

## Interactions with other effects

### Fallibility
### Compile-time Execution
### Multiplicity
### Thread-Safety
### Must-not Move
### Unwinding
### Thread Safety
### Fallibility

