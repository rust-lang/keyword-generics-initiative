- Name: `progress-report-feb-2023`
- Proposed by: [@yoshuawuyts](https://github.com/yoshuawuyts) and [@oli-obk](https://github.com/oli-obk)
- Original proposal: [Keyword Generics Progress Report: February 2023 | Inside Rust Blog](https://blog.rust-lang.org/inside-rust/2023/02/23/keyword-generics-progress-report-feb-2023.html)

# Design

<!-- Please fill out the snippets labeled with "fill me in". If there are any
other examples you want to show, please feel free to append more.-->

## base (reference)

<!-- This is the snippet which is being translated to various scenarios we're
translating from. Please keep this as-is, so we can reference it later.-->

```rust
/// A trimmed-down version of the `std::Iterator` trait.
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    fn size_hint(&self) -> (usize, Option<usize>);
}

/// An adaptation of `Iterator::find` to a free-function
pub fn find<I, T, P>(iter: &mut I, predicate: P) -> Option<T>
where
    I: Iterator<Item = T> + Sized,
    P: FnMut(&T) -> bool;
```

## always async

<!-- A variant where all items are always `async` -->

```rust
pub trait async Iterator {
    type Item;
    async fn next(&mut self) -> Option<Self::Item>;
    fn size_hint(&self) -> (usize, Option<usize>);
}

pub async fn find<I, T, P>(iter: &mut I, predicate: P) -> Option<T>
where
    I: async Iterator<Item = T> + Sized,
    P: async FnMut(&T) -> bool;
```

## maybe async

<!-- A variant where all items are generic over `async` -->

```rust
pub trait ?async Iterator {
    type Item;
    ?async fn next(&mut self) -> Option<Self::Item>;
    fn size_hint(&self) -> (usize, Option<usize>);
}

pub ?async fn find<I, T, P>(iter: &mut I, predicate: P) -> Option<T>
where
    I: ?async Iterator<Item = T> + Sized,
    P: ?async FnMut(&T) -> bool;
```

## generic over all modifier keywords

<!-- A variant where all items are generic over all modifier keywords (e.g.
`async`, `const`, `gen`, etc.) -->

*A slight modification from the report, using `effect` instead of `?effect`.*

```rust
pub trait effect Iterator {
    type Item;
    effect fn next(&mut self) -> Option<Self::Item>;
    fn size_hint(&self) -> (usize, Option<usize>);
}

pub effect fn find<I, T, P>(iter: &mut I, predicate: P) -> Option<T>
where
    I: effect Iterator<Item = T> + Sized,
    P: effect FnMut(&T) -> bool;
```

# Notes

This is the original design proposed by the Keyword Generics Initiatve in the
Feb 2023 Progress Report.

<!-- Add additional notes, context, and thoughts you want to share about your design
here -->
