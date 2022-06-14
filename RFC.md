# ✨ RFC

> When you're ready to start drafting, copy in the [template text](https://raw.githubusercontent.com/rust-lang/rfcs/master/0000-template.md) from the [rfcs](https://github.com/rust-lang/rfcs) repository.

## Motivation

TBD

## Design

### Const

```rust
const fn foo() {}
```

desugars to

```rust
const<C> fn foo() {}
```

This means

```rust
const fn foo<T: const SomeTrait>(t: T) {}
```

desugars to this, which is just one character more:

```rust
const<C> fn foo<T: SomeTrait * C>(t: T) {}
```


which allows you to call trait methods within the `const fn`.

For the common case however, effects do not need to be named and can be referenced to by `_`:

```rust
const<_> fn foo<T: SomeTrait * _>(t: T) {}
```

### Async

In contrast to `const` (which is "maybe"-by-default) there is no
sugared version.

```rust
async<A> fn foo() {}
```

You can then pass this effect into other uses of `async` within
the maybe-`async` function:

```rust
async<A> fn foo<T: SomeTrait * A>(t: T) {
    t.some_method()
}
```

which allows you to call trait methods within the `~async fn`.

### Combining effects

If you have both `async` and `const` modifiers, your function may not perform
any non-const operations like accessing the file system or static items, but is
allowed to use `await`.

```rust
// Can be const, async, both or neither
const<X> async<X> fn foo<T: SomeTrait * X>(t: T)
where
    effect X: async ^ const,
{
    t.bar().await
}
```

`async` and `const` can mutually exclusive, because `async` is useful for non-blocking IO,
but `const` doesn't have *any* IO at all. Thus we allow declaring arbitrary *exclusive-or*
bounds for effects.

```rust
// Can be const, async, both or neither
const<X> async<X> fn foo<T: SomeTrait * X>(t: T)
where
    effect X: async ^ const,
{
    t.bar().await
}
```

- Yosh's brain error brain worms: we shouldn't have `try` functions, we should
  have functions which use `throws`, and it specifies which error we throw.
- Oli: the carried error from `try` could just be inferred from the call-site.

```rust
async<E> try<E> fn foo(t: impl SomeTrait * E, p: impl SomeTrait * E) {
    t.some_method().await?;
    p.some_method().await?;
}
```

```rust
async<E1> try<E2> fn foo(t: impl SomeTrait * E1, p: impl SomeTrait * E2, q: impl SomeTrait * E1 * E2) {
    t.some_method().await;
    p.some_method()?;
    q.some_method().await?;
}
```

### Multiple different effects

We don't have a decision yet on how `try` or (`throws`) is supposed to work, which may need to provide a way to define N-copies of the same effect.
This RFC does not address this, however it is forward compatible with adding multiple independent effects of a single keyword:

```rust
try<E1, E2> fn foo(t: impl SomeTrait * E1 * E2) {
    t.some_method()??;
}
```

## Alternatives

## Future Extensions

