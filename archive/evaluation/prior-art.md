# Prior Art

## C++: `noexcept(noexcept(…))`

C++'s `noexcept(noexcept(…))` pattern is used to declare something as `noexcept`
if the evaluated pattern is also `noexcept`. This makes `noexcept` conditional
on the pattern provided.

This is most commonly used in generic templates to mark the output type as
`noexcept` if all of the input types are `noexcept` as well.

- [Raymond Chen, “Please Repeat Yourself: The Noexcept(Noexcept(…)) Idiom,” The Old New Thing, April 8, 2022](https://devblogs.microsoft.com/oldnewthing/20220408-00/?p=106438)

## C++: implicits and `constexpr`

`constexpr` can be applied based on a condition. The following example works:

#### C++ 11
```cpp
template <
class U = T,
detail::enable_if_t<std::is_convertible<U &&, T>::value> * = nullptr,
detail::enable_forward_value<T, U> * = nullptr>
constexpr optional(U &&u) : base(in_place, std::forward<U>(u)) {}

template <
class U = T,
detail::enable_if_t<!std::is_convertible<U &&, T>::value> * = nullptr,
detail::enable_forward_value<T, U> * = nullptr>
constexpr explicit optional(U &&u) : base(in_place, std::forward<U>(u)) {}
```

#### C++ 20
```cpp
template <
class U = T,
detail::enable_forward_value<T, U> * = nullptr>
explicit(std::is_convertible<U &&, T>::value) constexpr optional(U &&u) : base(in_place, std::forward<U>(u)) {}
```

__todo:__ validate what this does exactly by someone who can actually read C++.

## Rust: `maybe-async` crate

- [fmeow, “maybe-async crate,” January 15, 2020](https://crates.io/crates/maybe-async)

## Rust: `fn main`

Rust provides overloads for `async fn main` through the `Termination` trait. The
`main` function can optionally be made fallible by defining `-> Result<()>` as
the return type. In the ecosystem it's common to extend `fn main` with async
capabilities by annotating it with an attribute. And this mechanism has been
shown to work in the compiler as well by implementing `Termination for F:
Future`.

The mechanism of overloading for `fn main` differs from what we're proposing,
but the outcomes are functionally the same: greater flexibility in which
function modifiers are accepted, and less need to duplicate / wrap code.

## Zig: async functions

> Zig infers whether a function is async, and allows async/await on non-async
> functions, which means that Zig libraries are agnostic of blocking vs async I/O.
> Zig avoids function colors.

— [Zig contributors, “Zig In-Depth Overview: Concurrency via Async Functions,” October 1, 2019](https://ziglang.org/learn/overview/#concurrency-via-async-functions)

## Swift: async overloading

```swift
// Existing synchronous API
func doSomethingElse() { ... }

// New and enhanced asynchronous API
func doSomethingElse() async { ... }
```

- https://github.com/apple/swift-evolution/pull/1392
