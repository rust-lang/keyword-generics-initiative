# What should we support

There are a lot of things we would ideally support, this file is intended as a collection of all of them. Not all of them will end up being supported as they may conflict with other design and implementation goals.

## purely const

Dealing with `const`-ness does not change type signatures.

```rust
impl<const N: usize, T> [T; N] {
    // `map` is const if `F` is const.
    //
    // This is the case as `map` eagerly uses `f` in
    // its function body.
    const<C> fn map<U, F, const<C>>(self, f: F) -> [U; N]
    where
        F: const<C> FnMut(T) -> U,
    {
        // ...
    }
}

trait Iterator {
    type Item;
    // `map` is always `const`.
    //
    // This is the case as `map` only creates the `Map`
    // struct but does not eagerly use `f`.
    const fn map<B, F>(self, f: F) -> Map<Self, F>
    where
        Self: Sized,
        F: FnMut(Self::Item) -> B,
    {
        Map::new(self, f)
    }
}

pub struct Map<I, F> {
    iter: I,
    f: F,
}

impl<B, I, F, const<C>> const<C> Iterator for Map<I, F>
where
    I: const<C> Iterator,
    F: const<C> FnMut(I::Item) -> B,
{
    type Item = B;
    // ...
}
```

## how does `impl const<C> Trait` work

`const` trait impls are intended to mean that **all** functions of that trait are const. But, how const?

As seen in the `Iterator::map` example up above, simply adding `const<C>` to all trait bounds of trait methods is too restrictive. The same holds for a lot of other iterator adaptors.

A similar issue is dropping generic parameters. Looking at the following example:
```rust
trait Trait {
    fn takes_self(self);

    fn doesnt_take_self();

    fn has_other_param<T>();

    fn uses_other_param<T>(x: T);
}
```

What would a `const` impl of that trait look like? The most permissive option for implementors would be:
```rust
impl<T, const<C>> const<C> Trait for T {
    fn takes_self(self)
    where
        T: const<C> Destruct,
    {}

    fn doesnt_take_self()
    where
        T: const<C> Destruct,
    {}

    fn has_other_param<U>()
    where
        T: const<C> Destruct,
        U: const<C> Destruct,
    {}

    fn uses_other_param<U>(x: U)
    where
        T: const<C> Destruct,
        U: const<C> Destruct,
    {}
}
```
While the impl which can be used in the most contexts would have no `const<C> Destruct` bounds at all. Any automated way of figuring this out would sometimes be incorrect, as functions with equal signatures can still have different requirements.

Considering that traits apparently have to opt-in to being `const`-able, due to backwards compatability of default methods, we could require the trait itself to declare its bounds when const, which may look like the following:
```rust
const<C> trait Trait {
    // Sidenote: the `X: const<C> Destruct` for generic parameters
    // of the trait could instead be moved to the super traits or
    // the impl instead. This is probably the more sensible approach
    // for most traits.
    fn takes_self(self)
    where
        Self: const<C> Destruct;

    // No bounds.
    fn doesnt_take_self();

    // Not visible in the method signature,
    // but maybe its expected that this function creates
    // and drops an instance of the `Self` type, so the
    // trait author assumes that `Self` can be dropped.
    fn has_other_param<T>();
    where
        Self: const<C> Destruct;

    fn uses_other_param<T>(x: T)
    where
        T: const<C> Destruct;
}
```

## multi `const` effect fun

```rust
const<C1 & C2> fn merge<T, U, R, F1, F2, const<C1>, const<C2>>(
    x: Vec<T>,
    y: Vec<U>,
    f1: F1,
    f2: F2
) -> Vec<R>
where
    F1: const<C1> FnMut(T) -> R,
    F2: const<C2> FnMut(U) -> R,
{
    // ...
}
```
Using the same `const` effect parameter (in our impl) doesn't work. Checking `F1: const<?> FnMut(T) -> R` may infer the `const`-ness to `yes` at which point `F2: const<yes> FnMut(U) -> R` would result in an error if `F2` isn't const, even though it should only result in `merge` not being `const`.

Unifying const effect infer variables may require some experimentation but shouldn't be too bad.

## `async` and more involved effects

it's confusing that `const` is actually the reverse of an effect as it reduces what this function may do. `async` functions are actually **more powerful** than ordinary functions.

**TODO: integrate the rest of https://hackmd.io/Kit1pLhmTVKcvjQHYq29Xw**