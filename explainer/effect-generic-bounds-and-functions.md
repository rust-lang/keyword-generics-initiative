- Feature Name: `effect-generic-bounds-and-functions`
- Start Date: (2024-01-20)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

[RFC 0000](./effect-generic-trait-declarations.md) introduces traits which are
generic over an effect, but implementers have to pick whether they want to
implement the base version or the effectful version of the trait. This RFC
extends that system further by removing that limitation, and enabling authors to
write functions which themselves are generic over effects. For example, here is
a function `io::copy` which would be able to operate either synchronously or
asynchronously, depending on which types are passed to it.

```rust
/// This defines a trait `Read` which may or may not
/// be async, using the design introduced in RFC 0000.
#[maybe(async)]
pub trait Read {
    #[maybe(async)]
    fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
}

/// This defines a trait `Write` which may or may not
/// be async, using the design introduced in RFC 0000.
#[maybe(async)]
pub trait Write {
    #[maybe(async)]
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    #[maybe(async)]
    fn flush(&mut self) -> Result<()>;
}

/// This defines a function `copy` which copies bytes from a
/// reader into a writer. This RFC enables this function to
/// operate either synchronously or asynchronously. Where if
/// operating synchronously, the `.await` operator becomes a
/// no-op.
#[maybe(async)] 
pub fn copy<R, W>(reader: &mut R, writer: &mut W) -> Result<()>
where
    R: #[maybe(async)] Read + ?Sized,
    W: #[maybe(async)] Write + ?Sized,
{
    let mut buf = vec![0; 1024];
    loop {
        match reader.read(&mut buf).await? {
            0 => return Ok(()),
            n => writer.write(&mut buf[0..n]).await?,
        };
    }
}
```

# Motivation
[motivation]: #motivation

[RFC 0000](./effect-generic-trait-declarations.md) introduced effect-generic
trait definitions: traits which are generic over effects, but implementors of
the trait have to pick which version they implement. This works fine when
authors know which effects they will be working with, like in applications. But
library authors will often want to write code which not only works with one
effect, but any number of effects. And for that effect-generic functions and
bounds would greatly help reduce the amount of code duplication.

The blog post: ["The bane of my
existence: Supporting both async and sync code in
Rust"](https://nullderef.com/blog/rust-async-sync/) documents the negative
experience of one of the [`rspotify`] authors maintaining both sync and async
versions of a crate. With ecosystem crates such as
[`maybe_async`](https://docs.rs/maybe-async/latest/maybe_async/) and
[`async-generic`](https://crates.io/crates/async-generic) attempting to provide
mitigations via the macro system. But these crates are limited in what they can
provide when it comes to integrating with Rust's libraries, diagnostics,
tooling, and inference systems.

`rspotify` is a thoroughly documented example of an author wanting to be
generic over an effect, but there are others. Specifically for the async effect,
the `mongodb` crate has both [sync][mdb] and [async][mdba] variants. So does the
`postgres` crate [[sync][pg], [async][pga]], as well as the `reqwest` crate
[[sync][rqw], [async][rqwa]], and both the [`tokio`] and [`async-std`] crates
duplicate large swaths of the stdlib's functionality. Other effects are also
covered, such as [`fallible-iterator`] for a version of the stdlib's `Iterator`
trait which short-circuits on error.  And [`fallible_vec`] for a `Vec` type with
methods which may returns errors rather than panics if an allocation fails.

[`rspotify`]: https://docs.rs/rspotify/latest/rspotify/index.html
[mdb]: https://docs.rs/mongodb/latest/mongodb/sync/index.html
[mdba]: https://docs.rs/mongodb/latest/mongodb/index.html
[pg]: https://docs.rs/postgres/latest/postgres/
[pga]: https://docs.rs/tokio-postgres/latest/tokio_postgres/index.html
[rqw]: https://docs.rs/reqwest/latest/reqwest/blocking/index.html
[rqwa]: https://docs.rs/reqwest/latest/reqwest/
[`tokio`]: https://docs.rs/tokio
[`async-std`]: https://docs.rs/async-std
[`fallible-iterator`]: https://docs.rs/fallible-iterator/latest/fallible_iterator/index.html
[`fallible_vec`]: https://crates.io/crates/fallible_vec

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## TODO: Effect-generic functions
## TODO: Eliminating effect forwarding notations

- strip the `.await` / `?` etc
- can only call other effect-generic functions
- 

## TODO: Effect-generic trait bounds

- yay, forward it
- consistent per row

## TODO: Inference

- only one of each
- must always be resolved, or must be generic

## Effect generic inherent trait methods

- are now enabled!

## specializing on an effect

- const intrinsic, but everywhere.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Desugaring

- how do function bodies desugar?
- how is the effect generic param passed around?

## TODO: Effect-generic bodies logic

|                                 | caller does not have effect | caller may have an effect | caller always has effect |
| ------------------------------: | ----------------------------------- | --------------------------------- | -------------------------------- |
| **callee does not have effect** | ✅ allowed to evaluate              | ✅ allowed to evaluate            | ✅ allowed to evaluate           |
|      **callee may have effect** | ✅ allowed to evaluate              | ✅ allowed to evaluate            | ✅ allowed to evaluate           |
|    **callee always has effect** | ❌ not allowed to evaluate          | ❌ not allowed to evaluate        | ✅ allowed to evaluate           |

Evaluating an async function in a non-async context is not possible.

```rust
//! Caller context does not have an effect,
//! callee always has an effect

async fn callee() {}
fn caller() {
    callee().await // ❌ cannot call `.await` in non-async context
}
```

The caller's context may be evaluated as synchronous, but the callee is
guaranteed to always be asynchronous. Because as we've seen it's not possible to
evaluate async functions in non-async contexts.

```rust
//! Caller context may have an effect,
//! callee always has an effect

async fn callee() {}
#[maybe(async)]
fn caller() {
    callee().await // ❌ cannot call `.await` in maybe-async context
}
```

# Drawbacks
[drawbacks]: #drawbacks

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Don't require forwarding notation

- important though; as that's where control flow may happen
- the possibility of something happening is the entire point of annotating it

# Prior art
[prior-art]: #prior-art

## const fn

Using the `const` keyword this is already possible in Rust: a single `const fn`
function can both be evaluated during compilation and at runtime. Contrast this
to `const {}` blocks, which can only be evaluated at compile-time. And using the
[`const_eval_select`](https://doc.rust-lang.org/core/intrinsics/fn.const_eval_select.html)
intrinsic it will even be possible to provide different implementations
depending on whether the function is evaluated at runtime or during compilation.
This enables the runtime variant of a function to provide more optimized
implementations, for example by leveraging platform-specific SIMD capabilities.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

# Future possibilities
[future-possibilities]: #future-possibilities
