# Asynchrony

## Feature Status

`async/.await` in Rust is considered "MVP stable". This means the reification of
the effect is stable, and both the `async` and `.await` keywords exist in the
language, but not all keyword positions are available yet. At the time of writing the most notable omissions are

## Description

todo

## Feature categorization

| Position      | Syntax               |
| ------------- | -------------------- |
| Create scope  | `async`              |
| Return item   | N/A                  |
| Forward item  | `.await`             |
| Consume scope | `thread::block_on` † |
| Reification   | `impl Future`        |

> † `thread::block_on` is not yet part of the stdlib, and only exists as a
> library feature. An example implementation can be found in the
> [`Wake`](https://doc.rust-lang.org/std/task/trait.Wake.html#examples) docs.

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
| Drop                | ❌        | `impl async Drop for Cat {}`       |
| Closures            | ❌        | `async ǀǀ  {}`                     |
| Iterators           | ❌        | `for await cat in cats {}`         |
