# Auto Concurrency

Async Rust brings [three unique capabilities to
Rust](https://blog.yoshuawuyts.com/why-async-rust/): the ability to apply ad-hoc
concurrency, the ability to arbitrarily pause, cancel and resume operations, and
finally the ability to combine these capabilities into new ones - such as ad-hoc
timeouts. Async Rust also does one other thing: it decouples "concurrency" from
"parallelism" - while in non-async Rust both are coupled into the "thread"
primitive.

One challenge however is to make use of these capabilities. People notoriously
struggle to use cancellation correctly, and are often caught off guard that
computations after being suspended at an `.await` point may not necessarily be
resumed ("cancelled"). Similarly: users will often struggle to apply
fine-grained concurrency in their applications - because it fundamentally means
exploding sequential control-flow sequences into Directed Acyclic control-flow
Graphs (control-flow DAGs).

## By Example: Swift

Swift has introduced the `async let` keyword to enable linear-looking
control-flow which statically expands to a concurrent DAG backed by tasks. To
see how this works we can reference
[SE-0304](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)'s
example which provides a `makeDinner` routine:

```swift
func makeDinner() async throws -> Meal {
  async let veggies = chopVegetables()                    // 1. concurrent with: 2, 3
  async let meat = marinateMeat()                         // 2. concurrent with: 1, 3
  async let oven = preheatOven(temperature: 350)          // 3. concurrent with: 1, 2, 4

  let dish = Dish(ingredients: await [try veggies, meat]) // 4. depends on: 1, 2, concurrent with: 3
  return await oven.cook(dish, duration: .hours(3))       // 5. depends on: 3, 4, not concurrent
}
```

The following constraints and operations occur here:

- constraint: `dish` depends on `veggies` and `meat`.
- concurrency: `veggies`, `meat`, and `oven` are computed concurrently
- constraint: `Meal` depends on `oven` and `dish`
- concurrency: `oven` and `dish` are computed concurrently

In Swift the `async let` syntax automatically spawns tasks and ensures that they
resolve when they need to. In Swift `await {}` and `try {}` apply not just to
the top-level expressions but also to all sub-expressions, so for example
awaiting the `oven` is handled by `await oven.cook (..)`. We can translate this
to Rust using the [`futures-concurrency`
library](https://docs.rs/futures-concurrency) without having to use parallel
tasks - just concurrent futures. That would look like this:

```rust
use futures_concurrency::prelude::*;

async fn make_dinner() -> SomeResult<Meal> {
    let dish = {
        let veggies = chop_vegetables();
        let meat = marinate_meat();
        let (veggies, meat) = (veggies, meat).try_join().await?;
        Dish::new(&[veggies, meat]).await
    };
    let (dish, oven) = (dish, preheat_oven(350)).try_join().await?;
    oven.cook(dish, Duration::from_mins(3 * 60)).await
}
```

Compared to Swift the control-flow here is much harder to tease apart. We've
accurately described our concurrency DAG; but reversing it to understand
_intent_ has suddenly become a lot harder. Programmers generally have a better
time understanding code when it can be read sequentially; and so it's no
surprise that the Swift version is better at stating intent.

## Auto-concurrency for Rust's Async Effect Contexts

Rust's async system differs a little from Swift's, but only in the details. The
main differences as it comes to what we'd want to do here are three-fold:

1. Swift's async primitive are tasks: which are managed, parallel async
   primitives. In Rust it's `Future`, which is unmanaged and not parallel by
   default - it's only concurrent.
2. In Rust all `.await` points have to be explicit and recursive awaiting of
   expressions is not supported. This is because as mentioned earlier: functions
   may permanently yield control flow at `.await` points, and so they have to be
   called out in the source code.

For these reasons we can't quite do what Swift does - but I believe we could
probably do something similar. From a language perspective, it seems like it
should be possible to do a similar system to `async let`. Any number of `async
let` statements can be joined together by the compiler into a single
control-flow graph, as long as their outputs don't depend on each other. And if
we're calling `.await?` on `async let` statements we can even ensure to insert
calls to `try_join` so concurrently executing functions can early abort on
error.

```rust
async fn make_dinner() -> SomeResult<Meal> {
    async let veggies = chop_vegetables();  // 1. concurrent with: 2, 3
    async let meat = marinate_meat();       // 2. concurrent with: 1, 3
    async let oven = preheat_oven(350);     // 3. concurrent with: 1, 2, 4

    async let dish = Dish(&[veggies.await?, meat.await?]);   // 4. depends on: 1, 2, concurrent with: 3
    oven.cook(dish.await, Duration::from_mins(3 * 60)).await // 5. depends on: 3, 4, not concurrent
}
```

Here, just like in the Swift example, we'd achieve concurrency between all
independent steps. And where steps are dependent on one another, they would be
computed as sequential. Each future still needs to be `.await`ed - but in order
to be evaluated concurrently the program authors no longer have to figure it out
by hand.

If we think about it, this feels like a natural evolution from the principles of
`async/.await`. Just the syntax alone provides us with the ability to convert
complex asynchronous callback graphs into seemingly imperative-looking code. And
by extending that to concurrency too, we're able to reap even more benefits from it.

## What about other concurrency operations?

A brief look at the [`futures-concurrency`
library](https://docs.rs/futures-concurrency/latest/futures_concurrency/) will
reveal a number of concurrency operations. Yet here we're only discussing one:
`Join`. That is because all the other operations do something which is unique to
async code, and so we have to write async code to make full use of it. Whereas
`join` does not semantically change the code: it just takes independent
sequential operations and runs them in concert.

## Maybe-async and auto-concurrency

The main premise of `#[maybe(async)]` notations is that they can take sequential
code and optionally run them without blocking. Under the system described in
this post that code could not only be non-blocking, it could also be concurrent.
Taking the system we're describing in the "Effect Generic Function Bodies and
Bounds" draft, we could write our `async let`-based code example as follows to
make it conditional over the `async` effect:

```rust
#[maybe(async)]  // <- changed `async fn` to `#[maybe(async)] fn`
fn make_dinner() -> SomeResult<Meal> {
    async let veggies = chop_vegetables();
    async let meat = marinate_meat();
    async let oven = preheat_oven(350);

    async let dish = Dish(&[veggies.await?, meat.await?]);
    oven.cook(dish.await, Duration::from_mins(3 * 60)).await
}
```

Which when evaluated synchronously would be lowered to the following code. This
code blocks and runs sequentially, but that is the best we can do without async
Rust's ad-hoc async capabilities.

```rust
fn make_dinner() -> SomeResult<Meal> {
    let veggies = chop_vegetables();
    let meat = marinate_meat();
    let oven = preheat_oven(350);

    let dish = Dish(&[veggies?, meat?]);
    oven.cook(dish, Duration::from_mins(3 * 60))
}
```

This is not the only way that `#[maybe(async)]` code could leverage async
concurrency operations: an async version of
[`const_eval_select`](https://doc.rust-lang.org/std/intrinsics/fn.const_eval_select.html)
would also work. It would, however, be by far the most convenient way of
creating parity between both contexts. As well as make async Rust code that much
easier to read.

## A note on syntax

An earlier version of this document proposed using `.co.await`, `.co_await`,
just `.co` or some other keyword to take the place of `async let` to indicate a
concurrent `.await` can happen. The feasibility of syntax like that is not
clear; though there would likely be distinct benefits to preserving the postfix
nature of existing notations. Any further exploration of this direction should
consider alternate syntaxes to `async let`. In particular as concurrent
execution of `for await` loops is something that's also desirable, and would
likely want syntax parity with concurrent execution of futures.

## Conclusion

In this document we describe a mechanism inspired by Swift's `async let`
primitive to author imperative-looking code which is lowered into concurrent,
unmanaged futures. Rather than needing to manually convert linear code into a
concurrent directed graph, the compiler could do that for us. Here is an example
code as we would write it today using the
[`Join::join`](https://docs.rs/futures-concurrency/latest/futures_concurrency/future/trait.Join.html)
operation, compared to a high-level `async let` based variant which would
desugar into the same code.

```rust
/// A manual concurrent implementation using Rust 1.76 today.
async fn make_dinner() -> SomeResult<Meal> {
    let dish = {
        let veggies = chop_vegetables();
        let meat = marinate_meat();
        let (veggies, meat) = (veggies, meat).try_join().await?;
        Dish::new(&[veggies, meat]).await
    };
    let (dish, oven) = (dish, preheat_oven(350)).try_join().await?;
    oven.cook(dish, Duration::from_mins(3 * 60)).await
}

/// An automatic concurrent implementation using a hypothetical `async let`
/// feature. This would desugar into equivalent code as the manual example.
async fn make_dinner() -> SomeResult<Meal> {
    async let veggies = chop_vegetables();  // 1. concurrent with: 2, 3
    async let meat = marinate_meat();       // 2. concurrent with: 1, 3
    async let oven = preheat_oven(350);     // 3. concurrent with: 1, 2, 4

    async let dish = Dish(&[veggies.await?, meat.await?]);   // 4. depends on: 1, 2, concurrent with: 3
    oven.cook(dish.await, Duration::from_mins(3 * 60)).await // 5. depends on: 3, 4, not concurrent
}
```

This is not the first proposal to suggest an some form of concurrent notation
for async Rust; to our knowledge that would be Conrad Ludgate in their [async
let blog post](https://conradludgate.com/posts/async-let). However just like in
Swift it seems to be based on the idea of managed multi-threaded tasks - not
Rust's unmanaged, lightweight futures primitive.

A version of this is likely possible for multi-threaded code too; ostensibly via
some kind of `par` keyword (`par let` / `par for await..in`). A full design is out
of scope for this post; but it should be possible to improve Rust's parallel
system in both async and non-async Rust alike (using tasks and threads
respectively).

## References

- [Swift SE-0304: Structured Concurrency](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- [Conrad Ludgate: Async Let - A New Concurrency Primitive?](https://conradludgate.com/posts/async-let)
