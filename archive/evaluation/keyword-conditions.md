# Conditions for keywords

When comparing `const fn` and `async fn` we can observe that keywords may find
themselves in three different states. Let's start with `async fn`, we can
observe that there are two states in which it can find itself using today's
syntax:

```rust
// Always non-async
fn copy<R, W>(reader: &mut R, writer: &mut W) -> io::Result<u64>
where
    R: Read + ?Sized,
    W: Write + ?Sized,
{}

// Always async
async fn copy<R, W>(reader: &mut R, writer: &mut W) -> io::Result<u64>
where
    R: AsyncRead + ?Sized,
    W: AsyncWrite + ?Sized,
{}
```

Assuming we'd want to enable `async` modifiers on traits, we might instead be
able to the async variant as:

```rust
// Always async
async fn copy<R, W>(reader: &mut R, writer: &mut W) -> io::Result<u64>
where
    R: async Read + ?Sized,
    W: async Write + ?Sized,
{}
```

For `const` there are two states as well, but they're different from the states
of `async`:

```rust
// Always non-const
fn add<T: Add>(a: T, b: T>) -> T::Output {
    a + b
}

// This will be `const` if called in const contexts, non-const if
// called in non-const contexts.
const fn add<T: ~const Add>(a: T, b: T>) -> T::Output {
    a + b
}
```

These states are different from async Rust. We can plot them as such:

|                                   | keyword `async`      | keyword `const`       |
| --------------------------------- | -------------------- | --------------------- |
| **keyword never applies**         | `fn foo() {}`        | `fn foo() {}`         |
| **keyword always applies**        | `async fn foo() {}`  | `const FOO: () = {};` |
| **keyword conditionally applies** | `~async fn foo() {}` | `const fn foo() {}`   |

For the `async` keyword, we're exploring whether we can make it apply
conditionally as well. Because it's a superset of Rust, with wide ranging
implications, just making it "async depending on the context you call it in",
similar to `const` could have surprising behaviour.

Going back to the previous example, but this time allowing the same function to
be called asynchronously and synchronously:

```rust
async fn copy<R, W>(reader: &mut R, writer: &mut W) -> io::Result<u64>
where
    R: ~async Read + ?Sized,
    W: ~async Write + ?Sized,
{
}
```

Now, if we just allowed non-async readers and writers, we could run into
trouble. Consider the following function body:

```rust
let x = reader.read_line().await;
let z = writer.write(x);
let y = reader.read_line().await;
z.await
```

if the reader and writer were in fact synchronous and this were implemented
naively, this could already block at the write call instead of the await.
Instead what actually happens is that `z` is a "fake future", so basically just
an argumentless `FnOnce` closure of the actual logic and `z.await` executes that
closure.

The chance of this occurring would would be highest specifically for types which
were implemented _incorrectly_: in order for async polymorphism to work, the
behavior between async and non-async code paths must be identical. For async
this means that side-effects should not start _until `.await` has been called._
This should be not expressable for any code written using async polymorphism.
And should be considered a bug for code written using _async overloading_ (more
on that in a next section) since behavior should match.

Given we're seeking to provide a complete stdlib experience for _all_ variants
of Rust, we have a fair amount of control over this experience through the
stdlib, and in addition to guidance we can provide assurances which will limit
the practicality of this occuring in the wild.
