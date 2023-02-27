- Name: `attribute based effects`
- Proposed by: [@oli-obk](https://github.com/oli-obk)
- Original proposal (optional): (url)

# Design

Use function and trait attributes to make a function/trait have effect-like behaviour
instead of adding new syntax. There's still some new syntax in trait bounds, but these are
removed by the attribute at attribute expansion time.

This is experimentally being built with a proc macro in https://github.com/yoshuawuyts/maybe-async-channel.

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
#[async]
pub trait Iterator {
    type Item;
    #[async]
    fn next(&mut self) -> Option<Self::Item>;
    fn size_hint(&self) -> (usize, Option<usize>);
}

/// An adaptation of `Iterator::find` to a free-function
#[async]
fn find<I, T, P>(iter: &mut I, predicate: P) -> Option<T>
where
    I: async Iterator<Item = T> + Sized,
    P: async FnMut(&T) -> bool;
```

## maybe async

<!-- A variant where all items are generic over `async` -->

```rust
#[maybe_async]
pub trait Iterator {
    type Item;
    #[maybe_async]
    fn next(&mut self) -> Option<Self::Item>;
    fn size_hint(&self) -> (usize, Option<usize>);
}

/// An adaptation of `Iterator::find` to a free-function
#[maybe_async]
fn find<I, T, P>(iter: &mut I, predicate: P) -> Option<T>
where
    I: async Iterator<Item = T> + Sized,
    P: async FnMut(&T) -> bool;
```

## generic over all modifier keywords

<!-- A variant where all items are generic over all modifier keywords (e.g.
`async`, `const`, `gen`, etc.) -->

```rust
#[effect]
pub trait Iterator {
    type Item;
    #[effect]
    fn next(&mut self) -> Option<Self::Item>;
    fn size_hint(&self) -> (usize, Option<usize>);
}

/// An adaptation of `Iterator::find` to a free-function
#[effect]
fn find<I, T, P>(iter: &mut I, predicate: P) -> Option<T>
where
    I: effect Iterator<Item = T> + Sized,
    P: effect FnMut(&T) -> bool;
```

# Notes

<!-- Add additional notes, context, and thoughts you want to share about your design
here -->
