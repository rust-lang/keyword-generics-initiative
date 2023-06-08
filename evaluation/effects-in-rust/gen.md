# Multiplicity

## Feature Status

The `Iterator` trait has been stable in Rust since 1.0, but the generator syntax
is currently _unstable_. This document will assume that generators are created
with the `gen` keyword, but that's for illustrative purposes only.

## Description

todo

## Technical Overview

| Position      | Syntax          |
| ------------- | --------------- |
| Create        | `gen`           |
| Return item   | `yield`         |
| Forward item  | N/A             |
| Consume scope | `for..in`       |
| Reification   | `impl Iterator` |

## Refinements

| Name                      | Description                                           |
| ------------------------- | ----------------------------------------------------- |
| `Step`                    | Has a notion of successor and predecessor operations. |
| `TrustedLen` † (unstable) | Reports an accurate length using `size_hint`.         |
| `TrustedStep` (unstable)  | Upholds all invariants of `Step`.                     |
| `DoubleEndedIterator`     | Is able to yield elements from both ends.             |
| `ExactSizeIterator` †     | Knows its exact length.                               |
| `FusedIterator`           | Always continues to yield `None` when exhausted.      |

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
