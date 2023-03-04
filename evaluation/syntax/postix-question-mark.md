- Name: `postix-question-mark`
- Proposed by: [@tvallotton](https://github.com/tvallotton)

# Design

## base (reference)

<!-- This is the snippet which is being translated to various scenarios we're
translating from. Please keep this as-is, so we can reference it later.-->

```rust
/// A trimmed-down version of the `std::Iterator` trait.
pub trait async? Iterator {
    type Item;
    async? fn next(&mut self) -> Option<Self::Item>;
    fn size_hint(&self) -> (usize, Option<usize>);
}

/// An adaptation of `Iterator::find` to a free-function
pub async? fn find<I, T, P>(iter: &mut I, predicate: P) -> Option<T>
where
    I: async? Iterator<Item = T> + Sized,
    P: FnMut(&T) -> bool;
```

## always async

<!-- A variant where all items are always `async` -->

```rust
/// An adaptation of `Iterator::find` to a free-function
pub async fn find<I, T, P>(iter: &mut I, predicate: P) -> Option<T>
where
    I: async Iterator<Item = T> + Sized,
    P: FnMut(&T) -> bool;
```

## maybe async

<!-- A variant where all items are generic over `async` -->

```rust
pub async? fn find<I, T, P>(iter: &mut I, predicate: P) -> Option<T>
where
    I: async? Iterator<Item = T> + Sized,
    P: FnMut(&T) -> bool;
```

## generic over all modifier keywords

<!-- A variant where all items are generic over all modifier keywords (e.g.
`async`, `const`, `gen`, etc.) -->

```rust
/// A trimmed-down version of the `std::Iterator` trait.
pub trait effect Iterator {
    type Item;
    effect fn next(&mut self) -> Option<Self::Item>;
    fn size_hint(&self) -> (usize, Option<usize>);
}
```

# Notes

This is just a postfix version of the originally proposed syntax.
This should appear more familiar, as the question mark is normally used at the end of a 
sentence, not at the beginning, and it looks similar to typescripts nullable types. 
it also makes generic references more legible `&mut? T` vs `&?mut T`. 
