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
[SE-304](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)'s
example which provides a `makeDinner` routine:

```swift
func makeDinner() async throws -> Meal {
  async let veggies = chopVegetables()
  async let meat = marinateMeat()
  async let oven = preheatOven(temperature: 350)

  let dish = Dish(ingredients: await [try veggies, meat])
  return await oven.cook(dish, duration: .hours(3))
}
```

The following constraints and operations occur here:

- constraint: `dish` depends on `veggies` and `meat`.
- concurrency: `veggies`, `meat`, and `oven` are computed concurrently
- constraint: `Meal` depends on `oven` and `dish`
- concurrency: `oven` and `dish` are computed concurrently

In Swift the `async let` syntax automatically spawns tasks and ensures that they
resolve when they need to - occasionally it will even implicitly insert `await`
points where needed (e.g. `oven` is never explicitly `await`ed). We can
translate this to Rust using the [`futures-concurrency`
library](https://docs.rs/futures-concurrency) without having to use parallel
tasks - just concurrent futures. That would look like this:

```rust
use futures_concurrency::prelude::*;

async fn make_dinner() -> SomeResult<Meal> {
    let dish_fut = {
        let veggies_fut = chop_vegetables();
        let meat_fut = marinate_meat();
        let (veggies, meat) = (veggies_fut, meat_fut).join().await?;
        Dish::new(&[veggies, meat]).await
    };
    let (dish, oven) = (dish_fut, preheat_oven(350)).join().await;
    oven.cook(dish, Duration::mins(3 * 60)).await
}
```

Compared to Swift the control-flow here is much harder to tease apart. We've
accurately described our concurrency DAG; but reversing it to understand
_intent_ has suddenly become a lot harder. Programmers generally have a better
time understanding code when it can be read sequentially; and so it's no
surprise that the Swift version is better at stating intent.

## Auto-concurrency for Rust's Async Effect Contexts

Rust's async system differs a little from Swift's, but only in the details. The
main differences as it comes to what we'd want to do here are two-fold:

1. Swift's async primitive are tasks: which are managed, parallel async
   primitives. In Rust it's `Future`, which is unmanaged and not parallel by
   default - it's only concurrent.
2. In Rust all `.await` points have to be explicit and should not be omitted.
   This is because as mentioned earlier: functions may permanently yield control
   flow at `.await` points, and so they have to be called out in the source code.

For this reason we can't quite do what Swift does - but I believe we could
probably do something similar. From a language perspective, it seems like it
should be possible to do a similar system using `async let` and `.await` - but
where we statically analyze the control-flow graph to figure out where to
implement the right concurrent `.await` points. An example:

```rust
async fn make_dinner() -> SomeResult<Meal> {
    async let veggies = chop_vegetable();
    async let meat = marinate_meat();
    async let oven = preheat_oven(350);

    async let dish = Dish(ingredients: [veggies.await?, meat.await?]);
    oven.await.cook(dish.await, Duration::mins(3 * 60)).await
}
```

This would achieve something very similar to

## References

- [Swift SE-0304: Structured Concurrency](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- [Conrad Ludgate: Async Let - A New Concurrency Primitive?](https://conradludgate.com/posts/async-let)
