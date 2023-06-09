# Asynchrony (Scoped Effect)
## Description

Asynchrony in Rust enables non-blocking operations to be authored in an
imperative fashion. This can be helpful for performance reasons, but
feature-wise it enables "arbitrary concurrency" and "arbitrary cancellation" of
computations. These can in turn be composed and leveraged by higher-level
control-flow primitives such as "arbitrary timeouts" and "arbitrary
parallel execution".

Asynchrony in Rust is implemented using a pair of keywords. `async` is used to
create an async context which is reified into a state machine backed by the
`Future` trait. And `.await` is used on the call-site to access the values
inside of an async context. Because `.await` can only be called inside of
`async` contexts, it eventually needs to be consumed by a top-level function
which knows how to run a future to completion.

## Feature Status

`async/.await` in Rust is considered "MVP stable". This means the reification of
the effect is stable, and both the `async` and `.await` keywords exist in the
language, but not all keyword positions are available yet.

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

## Refinements

| Modifier          | Description                          |
| ----------------- | ------------------------------------ |
| cancellation-safe | Has no associated future-local state |

### Cancellation-Safe Futures

"cancellation-safety" is currently more like a term of art than an first-class
term. It is a property used and relied upon by ecosystem APIs, but it is not
represented in the type system anywhere. Which means APIs which rely on
"cancellation-safety" do so without compiler-backing, which makes them a
notorious source of bugs. This should probably be fixed, and when we do we
probably will not want to call it "cancellation-safety" since it relates less to
"cancellation" and more to the statefulness of futures, and whether or not they
can be recreated without side-effects or data loss.

### Fused Futures

A `FusedFuture` super-trait also exists, but it does not meaningfully feel like
a modifier of the "async" effect. It only adds an `is_terminated` method which
returns a bool. It does not inherently change the semantic functioning of the
underlying `Iterator` trait, or enhance it with behavior which is otherwise
absent. This is different from e.g. `FusedIterator` which says something about
the behavior of the `Iterator::next` function.


## Interactions with other effects
### Asynchrony
### Compile-time Execution
### Fallibility
### Iterativity
### May Panic
### Memory-Unsafety
### Must-Not Move
### Object-Safety
### Ownership
### Thread-Safety
