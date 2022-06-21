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

### Backwards Compatibility

One important part of keyword generics is that they will allow us to introduce them to the standard library without
breaking any existing code. Right now we're already adding constness to many functions in the standard library.

Take for example the method `Option::map`. This method takes a closure `f`. If we want to make this method `const` the closure must be `const` as well. This means the constness of the method and the constness of the input types are _linked_.

Before constification, `Option::map` looked like this:

```rust
impl<T> Option<T> {
    fn map<U>(self, f: impl FnOnce(T) -> U) -> Option<U> {
        match self {
            None => None,
            Some(x) => Some(f(x)),
        }
    }
}
```

We might be tempted to make the function "maybe-const" by adding a `const`
keyword-generic to the function:

```rust
impl<T> Option<T> {
    const<C> fn map<U>(self, f: impl FnOnce(T) -> U) -> Option<U> {
        match self {
            None => None,
            Some(x) => Some(f(x)),
        }
    }
}
```

But the compiler won't let us do this, because we're calling the closure `f`
inside the function body, but `f` is not marked "maybe-const":

```
error[E0000]: `f` is not const
 --> lib/option.rs:5:20
  |
5 |             Some(x) => Some(f(x)),
  |                             ^ implements `FnOnce` but is lacking a `const` keyword
  |
hint: consider applying the `C` generic to the `FnOnce` bound:
 --> lib/option.rs:2:48
  |
2 |    const<C> fn map<U>(self, f: impl FnOnce(T) -> U * C) -> Option<U> {
  |                                                   ++++
```

- current use of map will keep working because any type which implements `FnOnce` will just keep working.
- Yosh question: so we base inference on the parameters we pass it?

```rust
fn main() {
    my_opt.map(|x| x * 2);       // non-async
}
async<A> fn foo(i: i32) -> i32 { i * 2 }
async fn main() {
    my_opt.map(async |x| x * 2).await; // async
    //         ^               ^ or is it this?
    //         | is it this?        
    my_opt.map(foo).await;
             //^^^ do we infer this to be `async` because of the `await`
}
```

- Conclusion: we must infer based on the `.await`
    - Because: if we infer based on the argument then in the case of a `maybe async` type `foo` we'd need to 
               start turbofishing arguments. In contrast: `.await` here is unambiguous!
    - But, we want to go further: if the function and the argument are both
      `maybe async`, and you're inside of an `async fn`, then _not_ `.await`ing
      should trigger a warning.
          - And: the default should fall back to the !async variant.
          - TODO: create an example for this!

```rust
// we cannot base inference on the type of the outer function. This code is valid
// today, but if we made it infer based on the enclosing function's type we'd get:
async fn main() {
    my_opt.map(|x| x * 2);
    // compile error!
}
```

The repeated use of `const<C>` enforces that if `map` is called with "not const" (so called
from a base Rust function), then the `FnOnce` bound can be satisfied by any type that
implements `FnOnce`, irrespective of its constness.

Thus backwards compatibility is preserved for `const`.

If `Option::map` is called from another `const fn` or from within a true `const`
context (like a const item's initializer), then the `C` is either "maybe const" or "definitely const",
but either way, only types that implement `const FnOnce` are allowed for `f`.

```rust
impl<T> Option<T> {
    const<C> fn map<U>(self, f: impl FnOnce(T) -> U * C) -> Option<U> {
    //    ^  is  linked   with   this   right   here  ^
        match self {
            None => None,
            Some(x) => Some(f(x)),
        }
    }
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

