- Name: (fill me in: `name-of-design`)
- Proposed by: [@name](link to github profile)
- Original proposal (optional): (url)

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
// fill me in
```

## maybe async

<!-- A variant where all items are generic over `async` -->

```rust
// fill me in
```

## generic over all modifier keywords

<!-- A variant where all items are generic over all modifier keywords (e.g.
`async`, `const`, `gen`, etc.) -->

```rust
// fill me in
```

# Notes

<!-- Add additional notes, context, and thoughts you want to share about your design
here -->
