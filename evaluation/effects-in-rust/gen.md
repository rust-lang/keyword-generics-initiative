# Multiplicity

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
