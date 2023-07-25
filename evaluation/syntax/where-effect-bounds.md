- Name: `where effect bounds`
- Proposed by: [@CaioOliveira793](https://github.com/CaioOliveira793)
- Original proposal: None

# Design

This syntax focus on being simple and recognizable rust code, with the possibility to incrementally extend the capabilities that keyewords generic may provide.

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
pub trait Iterator<effect>
where
    effect: async
{
    type Item;

    // opt-in for a always async effect
    fn next(&mut self) -> Option<Self::Item>
    where
        effect: async;

    // the size_hint is left unchanged, since the effect is opt-in
    fn size_hint(&self) -> (usize, Option<usize>);
}

pub fn find<I, T, P, effect>(iter: &mut I, predicate: P) -> Option<T>
where
    effect: async,
    I: Iterator<Item = T> + Sized,
    <I as Iterator>::effect: async,
    P: FnMut(&T) -> bool,
    <P as FnMut>::effect: async;
```

## maybe async

<!-- A variant where all items are generic over `async` -->

```rust
pub trait Iterator<effect>
where
    effect: ?async
{
    type Item;

    fn next(&mut self) -> Option<Self::Item>
    where
        effect: ?async;

    fn size_hint(&self) -> (usize, Option<usize>);
}

pub fn find<I, T, P, effect>(iter: &mut I, predicate: P) -> Option<T>
where
    effect: ?async,
    I: Iterator<Item = T> + Sized,
    <I as Iterator>::effect: ?async,
    P: FnMut(&T) -> bool,
    <P as FnMut>::effect: ?async;
```

## generic over all modifier keywords

<!-- A variant where all items are generic over all modifier keywords (e.g.
`async`, `const`, `gen`, etc.) -->

```rust
pub trait Iterator<effect>
where
    // LIMITATION: in order to be generic over all keywords the effect clause must specify all keywords available
    effect: ?async + ?const
{
    type Item;

    fn next(&mut self) -> Option<Self::Item>
    where
        effect: ?async + ?const;

    fn size_hint(&self) -> (usize, Option<usize>);
}

pub fn find<I, T, P, effect>(iter: &mut I, predicate: P) -> Option<T>
where
    effect: ?async + ?const,
    I: Iterator<Item = T> + Sized,
    <I as Iterator>::effect: ?async + ?const,
    P: FnMut(&T) -> bool,
    <P as FnMut>::effect: ?async + ?const;
```

# Notes

## Trait effect bounds

The syntax for specifying the effect of a trait implemented by some generic argument `<I as Iterator>::effect: const` could be different

```rust
I: Iterator<effect = ?async + ?const>

// or

// Associated type bounds [RFC 2289](https://github.com/rust-lang/rfcs/blob/master/text/2289-associated-type-bounds.md)
I: Iterator<effect: ?async + ?const>
```

The current way mimics how associated types are bound

```rust
fn print_iter<I, effect>(iter: I)
where
    effect: ?async,
    I: Iterator,
    <I as Iterator>::Item: Display,
    <I as Iterator>::effect: ?async;
```

## Explicit generic over all modifier keywords

The syntax does not give shortans for specifying all modifiers at once. Instead, the function, trait or type should **explicit bound** over all keywords it could be generic.

Although being inconvenient to list it manually, this has some advantages over the *generic over all keywords available* syntax.

### Explicit

Readers does not have to remind which keywords are available that may need to be implemented in some specific way.

### Backwards compatible to introduce new keywords in the language

Allowing the *generic over all* means that in case a new keyword lands, all *complete generic* functions and traits may be affected by the keyword, requiring at least some considerations on the side effects.

## Limitations

These are some limitations (hopefully, not yet supported features) noticed in the syntax.

### Effect sets

Function generic over sets of effects, limiting it to be called by **only one** group of effects.

```rust
fn compute<effect<KernelSpace | UserSpace | PreComputed>>() -> Response
where
    effect<KernelSpace>: !alloc + !panic + !async,
    effect<UserSpace>: alloc + ?async,
    effect<PreComputed>: const
{
    if effect<KernelSpace> {
        // ensures that in "KernelSpace" will not alloc, panic or run futures
    }
    if effect<UserSpace> {
        // allow allocations and futures
    }
    if effect<PreComputed> {
        // only compile-time evaluation
    }
}

fn caller1()
where
    effect: ?alloc + !panic + !async
{
    compute<effect<KernelSpace>>(); // allowed
}

fn caller2()
where
    effect: alloc + async
{
    compute<effect<KernelSpace>>(); // allowed
    compute<effect<UserSpace>>(); // allowed
}

fn caller3()
where
    effect: !alloc
{
    compute<effect<UserSpace>>(); // compiler error
}

fn caller4()
where
    effect: const
{
    compute<effect<PreComputed>>(); // allowed
}
```
