- Name: `const bool-like effects`
- Proposed by: [@Lili Zoey](https://github.com/sayaks)
- Original proposal: [comment](https://github.com/rust-lang/keyword-generics-initiative/issues/10#issuecomment-1445263558)

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
In all 
The methods on the trait are assumed async because the trait is `async`.

Variation A:

```rust
pub async trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    !async fn size_hint(&self) -> (usize, Option<usize>);
}

pub async fn find<I, T, P>(iter: &mut I, predicate: P) -> Option<T>
where
    I: Iterator<Item = T> + Sized,
    P: async FnMut(&T) -> bool;
```

Variation B. Using an "`effect`-generics" notation:

```rust
pub trait Iterator<effect async> {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    fn size_hint<effect !async>(&self) -> (usize, Option<usize>);
}

pub fn find<I, T, P, effect async>(iter: &mut I, predicate: P) -> Option<T>
where
    I: Iterator<Item = T> + Sized,
    P: FnMut<effect async>(&T) -> bool;
```

Variation C. Using an `effect`-notation in `where`-bounds:

```rust
pub trait Iterator
where
    effect async
{
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    fn size_hint(&self) -> (usize, Option<usize>)
    where
        effect !async;
}

pub fn find<I, T, P>(iter: &mut I, predicate: P) -> Option<T>
where
    I: Iterator<Item = T> + Sized,
    P: FnMut<effect async>(&T) -> bool;
```

## maybe async

<!-- A variant where all items are generic over `async` -->

For all variations the use of `<effect async = A>` on `fn next` is elided.

Variation A. Using an `effect A: async` + `!async fn` in the trait definition:

```rust
pub trait Iterator<effect A: async> {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    !async fn size_hint(&self) -> (usize, Option<usize>);
}

pub fn find<I, T, P, effect A: async>(iter: &mut I, predicate: P) -> Option<T>
where
    I: Iterator<Item = T, effect async = A> + Sized,
    P: FnMut<effect async = A>(&T) -> bool;
```

Variation B. Using `effect A: async` + `effect! async` in the trait definition:

```rust
pub trait Iterator<effect A: async> {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    fn size_hint<effect !async>(&self) -> (usize, Option<usize>);
}

pub fn find<I, T, P, effect A: async>(iter: &mut I, predicate: P) -> Option<T>
where
    I: Iterator<Item = T, effect async = A> + Sized,
    P: FnMut<effect async = A>(&T) -> bool;
```

Variation C. Using `effect A: async` + `where effect !async` notation.  If we'd
instead written `where A = !async`, the `size_hint` method would only exist if
the context was not async. It instead now exists as not async in all contexts:

```rust
pub trait Iterator<effect A: async> {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    fn size_hint(&self) -> (usize, Option<usize>)
    where
        effect !async;
}

pub fn find<I, T, P, effect A: async>(iter: &mut I, predicate: P) -> Option<T>
where
    I: Iterator<Item = T, effect async = A> + Sized,
    P: FnMut<effect async = A>(&T) -> bool;
```

## generic over all modifier keywords

<!-- A variant where all items are generic over all modifier keywords (e.g.
`async`, `const`, `gen`, etc.) -->

```rust
pub trait Iterator<effect A: for<effect>> {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    !async fn size_hint(&self) -> (usize, Option<usize>);
}

pub fn find<I, T, P, effect A: for<effect>>(iter: &mut I, predicate: P) -> Option<T>
where
    I: Iterator<Item = T, for<effect> = A> + Sized,
    P: FnMut<for<effect> = A>(&T) -> bool;
```
[See also](#foreffect-bounds-and-traits)

# Notes
`!async fn foo` could be `sync fn foo` or omitted entirely in favor of only having `fn foo<effect !async>`. It is also a question if *all* effects should allow for `effect fn foo` syntax.

`for<effect>` should maybe be made more special-looking since it behaves quite differently from other generic effect variables.

The exact syntax of `effect A: E` and `effect E = A` for declaring a generic and specifying a bound for an effect could maybe be made different. 

It might be easier to implement specialization for specifically effect-generics, as they are rather simple, effectively just being bools, and there not being any lifetime parameters on them.

## Some nice things about the syntax

### Specific behavior
To make a function have specific behavior in the case where an effect is or is not true, we could do this:
```rs
fn foo<effect A: async>() {
    if A {
        // do stuff when foo is async
    } else {
        // do stuff when foo is not async
    }
}
```

### Impl blocks
impl blocks could look very similar to any other generics.
```rust
impl<effect A: async> SomeTrait<effect async = A> MyGenericType { ... }
impl SomeTrait<effect async> MyAsyncType { ... }
impl SomeTrait<effect !async> MySyncType { ... }
```

## Description
We can add effects to generics like `<effect A: E>`, and create bounds for the effects of types by doing `effect E = A` in the `<..>` list or the where-clause.

The basic syntax is that `effect async = true` means the type is async, whereas `effect async = false` means it is not.

For convenience we'd let `effect async` be the same as `effect async = true` and `effect !async` be the same as `effect async = false`.

`async fn foo` would be syntactic sugar for `fn foo<effect async = true>`. and similar for other effects.

So as an example, here are some equivalent ways of writing an async function:
```rust
fn foo<T, O, const N: usize, effect async = true>(...) {...}
fn foo<T, O, const N: usize, effect async>(...) {...}
async fn foo<T, O, const N: usize>(...) {...}
fn foo<T, O, const N: usize>(...) where effect async {...}
```

Every effect has a default value, and if there is no bound on the type for that specific effect it is assumed to have its default value. So the function above, having no bound on `const`, would be assumed not-const.

This could be explicitly stated like
```rust
async fn foo<T, O, const N: usize>(...) where effect !const {...}
```
However this would be unneccesary.

If a type has only one generic for an effect, and no other bounds for that effect. It is assumed to have the same bound as that one generic. Meaning the following are equivalent ways of making a function generic over `async`.
```rust
fn foo<T, O, const N: usize, effect A: async>foo(...) where effect async = A {...}
fn foo<T, O, const N: usize, effect A: async>foo(...) {...}
```

However if there are multiple generics, we'd need to explicitly state what the bound should be for the type itself.
```rust
fn foo<T, O, const N: usize, effect A: async, effect B: async>foo(...) where effect async = A | B {...}
```
This would mean that `foo` is async if either `A` is true or `B` is true. We could also use `A + B` if wanted it to be async whenever both are true.

Declaring an type to have/not have an effect different from the default value might change the type. For instance 
`fn foo<effect async>() -> T` would become `foo() -> Future<Output = T>`.

Every generic effect variable (except `for<effect>`) is also like a constant boolean value, which is true whenever the type is in a context where it has that effect, and false otherwise.

In traits, the items are assumed to have the same effect bounds as the trait itself. But this can be overridden using specific bounds for that item.

For instance
```rs
trait Read<effect A: async> {
    // This function is now generic over async
    fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
    // or equivalently
    fn read(&mut self, buf: &mut [u8]) -> Result<usize> where effect async = A;

    // This function is now always async
    async fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
    // or equivalently
    fn read(&mut self, buf: &mut [u8]) -> Result<usize> where effect async;

    // This function now only exists when the trait is async
    fn read(&mut self, buf: &mut [u8]) -> Result<usize> where A;
}
```

This also shows that unlike normal `const _: bool` we can actually use whether the generic effects are `true`/`false` in the where-clause.

### `for<effect>`

`for<effect>` is a universal effect bound that allows you to place bounds on all the effects of a type. Adding a `effect A: for<effect>` makes `A` a generic variable that ranges over every effect. This means its value is no longer a simple `true`/`false` and so can't be used bare in where-clauses.

If another bound is added that is more specific, that bound will limit the possible values of `A` as well. Meaning that if you have `<effect A: for<effect>, effect async>`, we would have the type be generic over every effect except async. And the type would always be async.

For instance, to make a function generic over all effects except const we'd write
```rust
fn foo<effect A: for<effect>>(...) where effect async {...}
```

To place bounds on every effect we write `for<effect> = A` where `A` is some bound. This should probably be limited somewhat to avoid people writing code that can very easily break. Consider for instance `for<effect> = true`, which would declare something as having *every* effect. This could lead to breakage if a new effect is added and the function isn't compatible with this new effect. The main uses of placing bounds on `for<effect>` would to use it with other universal bounds.

Using `A + B` and `A | B` bounds for universal bounds may also be problematic, as it may not always be possible to create any meaningful code that is generic in all those cases. So we may have to either disallow having multiple generic universal bounds, or have the compiler automatically infer the relationship between effects.

For instance
```rust
fn foo<O, F1, F2, effect A: for<effect>, effect B: for<effect>>(closure1: F1, closure2: F2) -> O
where
    F1: FnMut<for<effect> = A>() -> O,
    F2: FnMut<for<effect> = B>() -> O
{ ... }
```
Here it is unclear when `foo` should be async and const. For instance, usually a function is `async` if there is *any* async code in the function. Whereas it is `const` if *all* the code is `const`.

I'm not entirely sure if this is best left up to the compiler to infer, it should be disallowed, or if the user must specify the bounds on every specific effect they may use.

However if the compiler infers it all, we could still specify specific relationships, like:
```rust
fn foo<O, F1, F2, effect A: for<effect>, effect B: for<effect>>(closure1: F1, closure2: F2) -> O
where
    effect async = A + B,
    F1: FnMut<for<effect> = A>() -> O,
    F2: FnMut<for<effect> = B>() -> O
{ ... }
```
To make this function async only if *both* `A` and `B` are async (or rather `async = true` in both sets `A` and `B`).

### semi-formal description
<details>
<summary>Syntax</summary>
There's a new kind of generic called effect-generics. For any given type, that effect may be `true` meaning the type has that effect, or it can be `false` meaning the type does not have that effect. 

We can make a type generic over an effect by adding `effect A: E`, where `A` is a generic variable and `E` is an effect.

An effect bound is one of: `true`, `false`, `default`, `A`, `B1 + B2`, `B1 | B2`, `!B1`. Where `A` is a generic variable, `B1` and `B2` are effect bounds.

An effect is either: the name of an effect, a generic variable, or `for<effect>`

To specify that a type must fit some effect bound we write `effect E = A`, where `E` is an effect and `A` is an effect bound, either in the `<..>` list or in the where-clause.
</details>

<details>
<summary>Semantics</summary>

- `effect E = true` means "has the effect `E`"
- `effect E = false` means "does not have the effect `E`"
- `effect E = default` means "has the effect `E` if the default for the effect is true"
- `effect E = A` where `A` is a generic variable, means "has the effect `E` if `A` is true"
- `effect E = B1 + B2` means "has the effect `E` if the bounds `B1` and `B2` are true"
- `effect E = B1 | B2` means "has the effect `E` if the bounds `B1` or `B2` are true"
- `effect E = !B` means "has the effect `E` if the bound `B` is false"
- `effect for<effect> = B` means "the effect bound `B` applies to every effect"
- `effect A: E` means "`A` is a generic variable corresponding to the effect `E`"

</details>

## `for<effect>` bounds and traits
In the [generic over all keywords](#generic-over-all-modifier-keywords) case we'd have that `size_hint` is generic over all effects except async. So it might be better to make such universal bounds not automatically apply to all items in a trait.

In that case we'd have
```rust
pub trait Iterator<effect A: for<effect>> {
    type Item;
    fn next(&mut self) -> Option<Self::Item> where for<effect> = A;
    fn size_hint(&self) -> (usize, Option<usize>);
}
```
Alternatively we could have an opt-out syntax, which would look something like
```rust
pub trait Iterator<effect A: for<effect>> {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    fn size_hint(&self) -> (usize, Option<usize>) where for<effect> = default;
}
```

