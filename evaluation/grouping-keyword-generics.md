## Grouping Keyword Generics

We expect it to be common that if a function takes generics and has conditional
keywords on those, it will want to be conditional over *all* keywords on those
generics. So in order to not have people repeat params over and over, we should
provide shorthand syntax.

Here is the "base" variant we're changing:
```rust
// without any effects
fn find<I>(
    iter: impl Iterator<Item = I>,
    closure: impl FnMut(&I) -> bool,
) -> Option<I> {
    ...    
}
```

We could imagine wanting a fallible variant of this which can short-circuit
based on whether an `Error` is returned or not. We could imagine the "base"
version using a `TryTrait` notation, and the "effect" version using the `throws`
keyword. Both variants would look something like this:

```rust
// fallible without effect notation
fn try_find<I, E>(
    iter: impl TryIterator<Item = I, E>,
    closure: impl TryFnMut<(&I), E> -> bool,
) -> Result<Option<I>, E> {
    ...
}

// fallible with effect notation
fn try_find<I, E>(
    iter: impl Iterator<Item = I> ~yeets E,
    closure: impl FnMut(&I) ~yeets E -> bool,
) -> Option<I> ~yeets E {
    ...
}
```

For `async` we could do something similar. The "base" version would use
`AsyncTrait` variants. And the "effect" variant would use the `async` keyword:

```rust
// async without effect notation
fn async_find<I>(
    iter: impl AsyncIterator<Item = I>,
    closure: impl AsyncFnMut(&I) -> bool,
) -> impl Future<Output = Option<I>> {
    ...
}

// async with effect notation
~async fn async_find<I>(
    iter: impl ~async Iterator<Item = I>,
    closure: impl ~async FnMut(&I) -> bool,
) -> Option<I> {
    ...
}
```

Both the "fallible" and "async" variants mirror each other closely. And it's
easy to imagine we'd want to be conditional over both. However, if neither the
"base" or the "effect" variants are particularly pleasant.

```rust
// async + fallible without effect notation
fn try_async_find<I, E>(
    iter: impl TryAsyncIterator<Item = Result<I, E>>,
    closure: impl TryAsyncFnMut<(&I), E> -> bool,
) -> impl Future<Output = Option<Result<I, E>>> {
    ...
}

// async + fallible with effect notation
~async fn try async_find<I, E>(
    iter: impl ~async Iterator<Item = I> ~yeets E,
    closure: impl ~async FnMut(&I) ~yeets E -> bool,
) -> Option<I> ~yeets E {
    ...
}
```

The "base" variant is entirely unworkable since it introduces a combinatorial
explosion of effects ("fallible" and "async" are only two examples of effects).
The "effect" variant is a little better because it composes, but even with just
two effects it looks utterly overwhelming. Can you imagine what it would look
like with three or four? Yikes.

So what if we could instead treat effects as an actual generic parameter? As we
discussed earlier, in order to lower effects we already need a new type of
generic at the MIR layer. But what if we exposed that type of generics as user
syntax too? We could imagine it to look something like this:

```rust
// conditional
fn any_find<I, effect F>(
    iter: impl F * Iterator<Item = I>,
    closure: impl F * FnMut(&I) -> bool,
) -> F * Option<I> {
    ...    
}
```

There are legitimate questions here though. Effects which provide a superset of
base Rust may change the way we write Rust. The clearest example of this is
`async`: would having an `effect F` require that we when we invoke our `closure`
that we suffix it with `.await`? What about a `try` effect, would that require
that we suffix it with a `?` operator? The effects passed to the function might
need to change the way we author the function body [^implication].

[^implication]: One interesting thing to keep in mind is that the total set of
effects is strictly _bounded_. None of these mechanisms would be exposed to
end-users to define their own effects, but only used by the Rust language. This
means we can know which effects are part of the set. And any change in calling
signature (e.g. adding `.await` or `?`, etc.) can be part of a Rust edition.

Another question is about _bounds_. Declaring an `effect F` is maximally
inclusive: it would capture all effects. Should we be able to place restrictions
on this effect, and if so what should that look like?
