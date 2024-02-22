<iframe class="youtube-video"
src="https://www.youtube-nocookie.com/embed/MTnIexTt9Dk"
title="YouTube video player" frameborder="0" allow="accelerometer; autoplay;
clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
allowfullscreen></iframe>

_This was originally posted on [Yosh's
blog](https://blog.yoshuawuyts.com/extending-rusts-effect-system/), but included
in this repository to be more easily referenced._

_This is the transcript of the RustConf 2023 talk: "Extending Rust's Effect
System", presented on September 13th 2023 in Albuquerque, New Mexico and
streamed online._

## Introduction

Rust has continuously evolved since version 1.0 was released in 2015. We've
added major features such as the try operator (`?`), const generics, generic
associated types (GATs), and of course: `async/.await`. Out of those four
features, three are what can be considered to be "effects". And though we've
been working on them for a long time, they are all still very much in-progress.

Hello, my name is Yosh and I work as a Developer Advocate for Rust at Microsoft.
I've been working on Rust itself for the past five years, and I'm among other
things a member of the Rust Async WG, and the co-lead of the Rust Effects
Initiative.

The thesis of this talk is that we've unknowingly shipped an effect system as
part of the language in since Rust 1.0. We've since begun adding a number of new
effects, and in order to finish integrating them into the language we need
support for _effect generics_.

In this talk I'll explain what effects are, what makes them challenging to
integrate into the language, and how we can overcome those challenges.

## Rust Without Generics

When I was new to Rust it took me a minute to figure out how to use generics. I
was used to writing JavaScript, and we don’t have generics there. So I found
myself mostly writing functions which operated on *concrete types*. I remember
my code felt pretty clumsy, and it wasn't a great experience. Not compared to,
say, the code the stdlib provides.

An example of a generic stdlib function is the `io::copy` function. It reads
bytes from a reader, and copies them into a writer. We can give it a file, a
socket, or any combination of the two, and it will happily copy bytes from one
into the other.  This all works as long as we give it the right types.

But what if Rust actually didn't have generics? What if the Rust I used to write
at the beginning was actually all we had? How would we write this `io::copy`
function? Well, given we're trying to copy bytes between sockets and file types,
we could probably hand-code individual functions for these. For our two types
here we could write four unique functions.

But unfortunately for us that would only solve the problem right in front of us.
But the stdlib doesn’t just have two types which implement read and write. It
has 18 types which implement read, and 27 types which implement write. So if we
wanted to cover the entire API space of the stdlib, we’d need 486 functions in
total. And if that was the only way we could implement `io::copy`, that would
make for a pretty bad language.

Now luckily Rust does have generics, and all we ever need is the one `copy`
function. This means we're free to keep adding more types into the stdlib
without having to worry about implementing more functions. We just have the one
`copy` function, and the compiler will take care of generating the right code
for any types we give it.

## Why effect generics?

Types are not the only things in Rust we want to be generic over. We also have
"const generics" which allow functions to be generic over constant values. As
well as "value generics" which allow functions to be generic over different
values. This is how we can write functions which can take different values -
which is a feature present in most programming languages.

```rust
fn by_value(cat: Cat) { .. }
fn by_reference(cat: &Cat) { .. }
fn by_mutable_reference(cat: &mut Cat) { .. }
```

But not everything that can lead to API duplication are things we can be generic
over. For example, it's pretty common to create different methods or types
depending on whether we take a value as owned, as a reference, or as a mutable
reference. We also often create duplicate APIs for constant values and for
runtime values. As well as create duplicate structures depending on whether the
API needs to be thread-safe or not.

But out of everything which can lead to API duplication, effects are probably
one of the biggest ones. When I talk about effects in Rust, what I mean is
certain keywords such as `async/.await` and `const`; but also `?`, and types
such as `Result`, and `Option`. All of these have a deep, semantic connection to
the language, and changes the meaning of our code in ways that other keywords
and types don't.

Sometimes we'll write code which doesn't have the right effects, leading to
_effect mismatches_. This is also known as the _function coloring problem_, as
described by Robert Nystrom. Once you become aware of _effect mismatches_ you
start seeing them all over the place, not just in Rust either.

The result of these effect mismatches is that using effects in Rust essentially
drops you into a second-rate experience. Whether you're using const, async,
Result, or Error - almost certainly somewhere along the line you'll run into a
compatibility issue.

```rust
let db: Option<Database> = ..;
let db = db.filter(|db| db.name? == "chashu");
```

Take for example the `Option::filter` API. It takes a type by reference and
returns a bool. If we try and use the `?` operator inside of it we get an error,
because the function doesn't return `Result` or `Option`. Not being able to use
`?` inside of arbitrary closures is an example of an effect mismatch.

But simple functions like that only scratch the surface. Effect mismatches are
present in almost every single trait in the stdlib too. Take for example
something common like the `Debug` trait which is implemented on almost every
type in Rust.

We could implement the `Debug` trait for our made-up type `Cat`. The parameter
`f` here implements `io::Write` and represents a stream of bytes. And using the
`write!` macro we can write bytes into that stream. But if for some reason we
wanted to write bytes asynchronously into, say, an async socket. Well, we can't
do that. `fn fmt` is not an async function, which means we can't await inside of
it.

One way out of this could be to create some kind of intermediate buffer, and
synchronously write data into it. We could then write data out of that buffer
asynchronously. But that would involve extra copies we didn't have before.

If we wanted to make it identical to what we did before, the solution would be
to create a _new_ `AsyncDebug` trait which _can_ write data asynchronously into
the stream. But we now have duplicate traits, and that's exactly the problem
we're trying to prevent.

It's tempting to say that maybe we should just add the `AsyncDebug` trait and
call it a day. We can then also add async versions of `Read`, `Write`, and
`Iterator` too. And perhaps `Hash` as well, since it too writes to an output
stream. And what about `From` and `Into`? Perhaps `Fn`, `FnOnce`, `FnMut`, and
`Drop` too since they're built-ins? And so on. The reality is that effect
mismatches are structural, and duplicating the API surface for every effect
mismatch leads to an exponential explosion of APIs. Which is similar to what
we've seen with data type generics earlier on.

Let me try and illustrate this for a second. Say we took the existing family of
`Fn` traits and introduced effectful versions of them. That is: versions which
work with `unsafe` [^unsafe-note], `async`, `try`, `const`, and generators. With one effect
we're up to six unique traits. With two effects we're up to twelve. With all
five we're suddenly looking at 96 different traits.

[^unsafe-note]: Correction from 2024: after having discussed this with Ralf Jung
we've concluded that semantically `unsafe` in Rust is not an effect. But
syntactically it would be fair to say that `unsafe` is "effect-like". As such
any notion of "maybe-unsafe" would be nonsensical. We don't discuss such a
feature in this talk, but it is worth clearing up ahead of time in case this
leaves people wondering.

The problem space in the stdlib is really broad. From analyzing the Rust 1.70
stdlib, by my estimate about 75% of the stdlib would interact with the const
effect. Around 65% would interact with the async effect. And around 30% would
interact with the try effect. The exact numbers are imprecise because parts of
the various effects are still in-progress. How much this will result in
practice, very much will depend on how we end up designing the language.

If you compare the numbers then it appears that close to 100% of the stdlib
would interact with one or more effect. And about 50% would interact with two or
more effects. If we consider that whenever effects interact with each other they
can lead to exponential blowup, this should warn us that clever one-off
solutions won't cut it. I believe that the best way to deal with this is to
instead allow Rust to enable items to be generic over effects.

## Stage I: Effect-Generic Trait Definitions

Now that we've taken a good look at what happens when we can't be generic over
effects, it's time we start talking about what we can do about it. The answer,
unsurprisingly, is to introduce effect generics into the language. To cover all
uses will take a few steps, so let's start with the first, and arguably most
important one: effect-generic trait definitions.

This is important because it would allow us to introduce effectful traits as
part of the stdlib. Which would among other things would help standardize the
various async ecosystems around the stdlib.

```rust
pub trait Into<T>: Sized {     
    fn into(self) -> T;
}
```

```rust
impl Into<Loaf> for Cat {     
    fn into(self) -> Loaf {
        self.nap()
    }
}
```

Let's use a simple example here: the `Into` trait. The `Into` trait is used to
convert from one type into another. It is generic over a type `T`, and has one
function "into" which consumes `Self` and returns the type `T`. Say we have a
type cat which when it takes a nap turns into a cute little loaf. We can
implement `Into<Loaf>` for `Cat` by calling `self.nap` in the function body.

```rust
pub trait AsyncInto<T>: Sized {     
    async fn into(self) -> T;
}
```

```rust
impl AsyncInto<Loaf> for Cat {     
    async fn into(self) -> Loaf {
        self.nap().await
    }
}
```

But what if the cat doesn't take a nap straight away? Maybe `nap` should
actually be an async function. In order to `await` nap inside the trait impl,
the `into` method would need to be async. If we were writing an async trait from
scratch, we could do this by exposing a new `AsyncInto` trait with an async
`into` method.

But we don't just want to add a new trait to the stdlib, instead we want to
_extend_ the existing `Into` trait to work with the async effect. The way we
could extend the `Into` trait with the `async` effect is by making the async
effect _optional_. Rather than requiring that the trait is always sync or async,
implementors should be able to choose which version of the trait they want to
implement.

```rust
#[maybe(async)]
impl Into<Loaf> for Cat {     
    #[maybe(async)]
    fn into(self) -> Loaf {
        self.nap()
    }
}
```

The way this would work is by adding a new notation on the trait: "maybe async".
We don't yet know what syntax we want to use for "maybe async", so in this talk
we'll be using attributes. The way the "maybe async" notation works is that we
mark all methods which we want to be "maybe async" as such. And then mark our
trait itself as "maybe async" too.

```rust
impl Into<Loaf> for Cat {     
    fn into(self) -> Loaf {
        self.nap()
    }
}
```

```rust
impl async Into<Loaf> for Cat {     
    async fn into(self) -> Loaf {
        self.nap().await
    }
}
```

Implementors then get to choose whether they want to implement the sync or async
versions of the trait. And depending on which version they choose, the methods
then ends up being either sync or async. This system would be entirely
backwards-compatible, because implementing the sync version of `Into` would
remain the same as it is today. But people who want to implement the async
version would be able to, just by adding a few extra `async` keywords to the impl.

```rust
impl async Into<Loaf> for Cat {
    async fn into(self) -> Loaf {
        self.nap().await
    }
}
```
```rust
impl Into<Loaf, true> for Cat {
    type ReturnTy = impl Future<Output = Loaf>;
    fn into(self) -> Self::ReturnTy {
        async move {
            self.nap().await
        }
    }
}
```

Under the hood the implementations desugars to regular Rust code we can already
write today. The sync implementation of the type returns a type `T`. But the
async impl returns an `impl Future` of `T`. Under the hood it is just a single
const bool and some associated types.

- good diagnostics
- gradual stabilization,
- backwards-compatibility
- clear inference rules

It would be reasonable to ask why we're bothering with a language feature, if
the desugaring ends up being so simple. And the reason is: effects are
everywhere, and we want to make sure effect generics feel like part of the
language. That not only means that we want to tightly control the diagnostics.
We also want to enable them to be gradually introduced, have clear language
rules, and be backwards-compatible.

But if you keep all that in mind, it's probably okay to think of effect generics
as mostly syntactic sugar for const bools + associated types.

## Stage II: Effect-Generic Bounds, Impls, and Types

Being able to declare effect-generic traits is only the beginning. The stdlib
not only exposes traits, it also exposes various types and functions. And
effect-generic traits don't directly help with that.

```rust
pub fn copy<R, W>(
    reader: &mut R,
    writer: &mut W
) -> io::Result<()>
where
    R: Read,
    W: Write;
```

Let's take our earlier `io::copy` example again. As we've said `copy` takes a
reader and writer, and then copies bytes from the reader to the writer. We've
seen this.

```rust
pub fn async_copy<R, W>(
    reader: &mut R,
    writer: &mut W
) -> io::Result<()>
where
    R: AsyncRead,
    W: AsyncWrite;
```

Now what would it look like if we tried adding an async version of this to the
stdlib today. Well, we'd need to start by giving it a different name so it
doesn't conflict with the existing `copy` function. The same goes for the trait
bounds as well, so instead of taking `Read` and `Write`, this function would
take `AsyncRead` and `AsyncWrite.

```rust
pub fn async_copy<R, W>(
    reader: &mut R,
    writer: &mut W
) -> io::Result<()>
where
    R: async Read,
    W: async Write;
```

Now things get a little better once we have effect-generic trait definitions.
Rather than needing to take async duplicates of the `Read` and `Write` traits,
the function can instead choose the async versions of the existing `Read` and
`Write` traits. That's already better, but it still means we have two versions
of the `copy` function.

```rust
#[maybe(async)]
pub fn copy<R, W>(
    reader: &mut R,
    writer: &mut W
) -> io::Result<()>
where
    R: #[maybe(async)] Read,
    W: #[maybe(async)] Write;
```

Instead the ideal solution would be to allow `copy` itself to be generic over
the async effect, and make that determine which versions of `Read` and `Write`
we want. These are what we call "effect-generic bounds". The effect of the
function and the effect of the bounds it takes all become the same. In
literature this is also known as "row-polymorphism".

```rust
copy(reader, writer)?;                // infer sync
copy(reader, writer).await?;          // infer async
copy::<async>(reader, writer).await?; // force async
```

Because the function itself is now generic over the async effect, we need to
figure out at the call-site which variant we intended to use. This system will
make use of _inference_ to figure it out. That's a fancy way of saying that
the compiler is going to make an educated guess about which effects the
programmer intended to use. If they used `.await` they probably wanted the async
version. Otherwise they probably wanted the sync version. But as with any guess:
sometimes we guess wrong, so for that reason we want to provide an escape hatch
by enabling program authors to force the variant. We don't know the exact syntax
for this yet, but we assume this would likely be using the turbofish notation.

```rust
struct File { .. }
impl File {
    fn open<P>(p: P) -> Result<Self>
    where
        P: AsRef<Path>;
}
```

But effect-generics aren't just needed for functions. If we want to make the
stdlib work well with effects, then types will need effect-generics too. This
might seem strange at first, since an "async type" might not be very intuitive.
But for example files on Windows need to be initialized as either sync or async.
Which means that whether they're async or not isn't just a property of the
functions, it's a property of the type.

Let's use the stdlib's `File` type as our example here. For simplicity let's
assume it has a single method: `open` which returns either an error or a file.

```rust
struct AsyncFile { .. }
impl AsyncFile {
    async fn open<P>(p: P) -> Result<Self>
    where
        P: AsRef<AsyncPath>;
}
```

If we wanted to provide an async version of `File`, we again would need to
duplicate our interfaces. That means a new type `AsyncFile`, which has a new
async method `open`, which takes an async version of `Path` as an argument. And
`Path` needs to be async because it itself has async filesystem methods on it.
As I've said before: once you start looking you notice effects popping up
everywhere.

```rust
#[maybe(async)]
struct File { .. }

#[maybe(async)] 
impl File {
    #[maybe(async)]
    fn open<P>(p: P) -> Result<Self>
    where
        P: AsRef<#[maybe(async)] Path>;
}
```

Instead of creating a second `AsyncFile` type, with effect generics on types
we'd be able to open `File` as async instead. Allowing us to keep just the one
`File` definition for both sync and async variants.

```rust
#[maybe(async)]
fn copy<R, W>(reader: R, writer: W) -> io::Result<()> {
    let mut buf = vec![4028];
    loop {
        match reader.read(&mut buf).await? {
            0 => return Ok(()),
            n => writer.write_all(&buf[0..n]).await?,
        }
    }
}
```

Now I've sort of hand-waved away the internal implementations of both the `copy`
function and the `File` type. The way they work is a little different for the
two. In the case of the `copy` function, the implementation between the async
and non-async variants would be identical. If the function is compiled as async,
everything works as written. But if the function compiles as sync, then we just
remove the `.await`s and the function should compile as expected.

As a result of this "maybe-async" functions can only call sync or other
"maybe-async" functions. But that should be fine for most cases.

```rust
impl File {
    #[maybe(async)]
    fn open<P>(p: P) -> Result<Self> {
        if IS_ASYNC { .. } else { .. }
    }
}
```

Concrete types like `File` are a little trickier. They often want to run
different code depending on which effects it has. Luckily types like `File`
already conditionally compile different code depending on the platform, so
introducing new types conditions shouldn't be too big of a jump. The key thing
we need is a way to detect in the function body whether code is being compiled
as async or not - basically a fancy bool.

We can already do this for the const effect using the `const_eval_select`
intrinsic. It's currently unstable and a little verbose, but it works reliably.
We should be able to easily adapt it to something similar for async and the rest
of the effects too.

## What are effects?

Systems research on effects has been a topic in computer science for nearly 40
years. That's about as old as C++. It's become a bit of a hot topic recently in
PL spheres with research languages such as Koka, Eff, and Frank showing how
effects can be useful. And languages such as Scala, and to a lesser extent
Swift, adopting effect features.

When people talk about effects they will broadly refer to one of two things:

- __Algebraic Effect Types:__ which are semantic notations on functions and contexts
  that grant a permission to _do_ something.
- __Algebraic Effect Handlers:__ which are a kind of typed control-flow
  primitive which allows people to define their own versions of `async/.await`,
  `try..catch`, and `yield`.

A lot of languages which have effects provide both effect types and effect
handlers. These can be used together, but they are in fact distinct features. In
this talk we'll only be discussing effect types.

```rust
pub async fn meow(self) {}
pub const unsafe fn meow() {}
```

What we've been calling "effects" in this talk so far have in fact been _effect
types_. Rust hasn't historically called them this, and I believe that's probably
why effect generics weren't on our radar until recently. But it turns out that
reinterpreting some of our keywords as effect types actually makes perfect
sense, and provides us with a strong theoretical framework for how to reason
about them.

We also have `unsafe` which allows you to call `unsafe` functions. The unstable
try-block feature which doesn't require you to `Ok`-wrap return types. The
unstable generator closure syntax which gives you access to the `yield` keyword.
And of course the `const` keyword which allows you evaluate code at
compile-time.

```rust
async { async_fn().await }; // async effect
unsafe { unsafe_fn() };     // unsafe effect
const { const_fn() };       // const effect
try { try_fn()? };          // try effect (unstable)
|| { yield my_type };       // generator effect (unstable)
```

In Rust we currently have five different effects: `async`, `unsafe`, `const`,
`try`, and generators. All six of these are in various stages of completion. For
example: async Rust has functions and blocks, but no iterators or drop. Const
doesn't have access to traits yet. Unsafe functions can't be lowered to `Fn`
traits. Try does have the `?` operator, but try blocks are unstable. And
generators are entirely unstable; we only have the `Iterator` trait.

Some of these effects are what folks on the lang team have started calling
"carried". Those are effects which will desugar to an actual type in the type
system. For example when you write `async fn`, the return type will desugar to
an `impl Future`.

Some other effects are what we're calling: _"uncarried"_. These effects don't desugar
to any types in the type system, but serve only as a way to communicate
information back to the compiler. This is for example `const` or `unsafe`. While
we do check that the effects are used correctly, they don't end up being lowered
to actual types.

```rust
let x = try async { .. };
```

```
1. -> impl Future<Output = Result<T, E>>
2. -> Result<impl Future<Output = T>, E>
```

When we talk about carried effects, effect composition becomes important. Take
for example "async" and "try" together. If we have a function which has both?
What should the resulting type be? A future of Result? Or a Result containing a
Future?

Effects on functions are order-independent _sets_. While Rust currently does
require you declare effects in a specific order, carried effects themselves can
only be composed in one way. When we stabilized `async/.await`, we decided that
if an async function returned a Result, that should always return an `impl
Future` of `Result`.  And because effects are _sets_ and not dependent on
ordering, we can define the way carried effects should compose as part of the
language.

People can still opt-out from the built-in composition rules by manually writing
function signatures. But this is rare, and for the overwhelming majority of uses
the built-in composition rules will be the right choice.

```rust
const fn meow() {}  // maybe-const
const {}            // always-const
```

The `const` effect is a bit different from the other effects. `const` blocks are
_always_ evaluated during compilation. While `const` functions merely _can_ be
evaluated during during compilation. It's perfectly fine to call them at runtime
too. This means that when we write `const fn`, we're already writing
effect-generics. This mechanism is the reason why we've gradually been able to
introduce const into the stdlib in a backwards-compatible way.

Const is also a bit strange in that among other things it disallows access to
the host runtime, it can't allocate, and it can't access globals. This feels
different from effects like say, `async`, which only allow you to do more
things.

| effect set     | can access                                                   | cannot access                    |
| -------------- | ------------------------------------------------------------ | -------------------------------- |
| std rust       | non-termination, unwinding, non-determinism, statics, runtime heap, host APIs | N/A                              |
| alloc          | non-termination, unwinding, non-determinism, globals, runtime heap            | host APIs                        |
| core           | non-termination, unwinding, non-determinism, globals                          | runtime heap, host APIs          |
| const          | non-termination, unwinding                                   | non-determinism, globals, runtime heap, host APIs |

What's missing from this picture is that all functions in Rust carry an implicit
set of effects. Including some effects we can't directly name yet. When we write
`const` functions, our functions have a different set of effects, than if we
write `no_std` functions, which again are different from regular "std" rust
functions.

The right way of thinking about const, std, etc. is as adding a different
effects to the empty set of effects. If we start from zero, then all effects are
merely additive. They just add up to different numbers.

Unfortunately in Rust we can't yet name the empty set of effects. In effect
theory this is called the "total effect". And some languages such as Koka do
support the "total" effect. In fact, Koka's lead developer has estimated that
around 70% of a typical Koka program can be total. Which begs the question: if
we could express the total effect in Rust, could we see similar numbers?

## Stage III: More Effects

So far we've only talked about how we could finish the work on existing effects
such as `const` and `async`. But one nice thing of effect generics is that they
would not only allow us to finish our ongoing effects work. It would also lower
the cost of introducing _new_ effects to the language.

Which opens up the question: if we could add more effects, which effects might
make sense to add? The obvious ones would be to actually finish adding `try` and
generator functions. But beyond that, there are some interesting effects we
could explore. For brevity I'll only discuss what these features are, and not
show code examples.

- **no-divergence**: guarantees that a function cannot loop indefinitely,
  opening up the ability to perform static runtime-cost analysis.
- **no-panic**: guarantees a function will never produce a panic, causing
  the function to unwind.
- **parametricity**: guarantees that a function only operates on its arguments. That
  means no implicit access to statics, no global filesystem, no thread-locals.
- **capability-safety**: guarantees that a function is not only parametric, but can't
  downcast abstract types either. Say if you get an `impl Read`, you can't reverse
  it to obtain a `File`.
- **destructor linearity**: guarantees that `Drop` will *always* be called,
  making it a safety guarantee.
- **pattern types**: enables functions to operate directly on variants of enums
  and numbers
- **must-not-move types**: would be a generalization of pinning and the
  pin-project system, making it a first-class language feature

Though there's nothing inherently stopping us from adding any of these features
into Rust today, in order to integrate them into the stdlib without breaking
backwards-compatibility we need effect generics first.

```rust
effect const  = diverge + panic;
effect core   = const + statics + non_determinism;
effect alloc  = core + heap;
effect std    = alloc + host_apis;
```

This brings us to the final part of the design space: effect aliases. If we keep
adding effects it's very easy to eventually reach into a situation where we have
our own version of "public static void main".

In order to mitigate that it would instead be great if we could name specific
sets of effects. In a way we've already done that, where `const` represents "may
loop forever" and "may panic". If we actually had "may loop forever" and "may
panic" as built-in effects, then we could redefine `const` as an alias to those.

Fundamentally this doesn't change anything we've talked about so far. It's just
that this would syntactically be a lot more pleasant to work with. So if we ever
reach a state where we have effect generics and we want notice we maybe have one
too many notation in front of our functions, it may be time for us to start
looking into this more seriously.

## Outro

Rust already includes effect types such as async, const, try, and unsafe.
Because we can't be generic over effect types yet, we usually have to choose
between either duplicating code, or just not addressing the use case. And this
makes for a language which feels incredibly rough once you start using effects.
Effect generics provide us with a way to be generic over effects, and we've
shown they can be implemented today as mostly as syntax sugar over
const-generics.

We're currently in the process of formalizing the effect generic work via the
A-Mir-Formality. MIR Formality is an in-progress formal model of Rust's type
system. Because effect generics are relatively straight forward but have
far-reaching consequences for the type system, it is an ideal candidate to test
as part of the formal model.

In parallel the const WG has also begun refactoring the way const functions are
checked in the compiler. In the past const-checking happened right before borrow
checking at the MIR level. In the new system const-checking will happen much
sooner, at the HIR level. This will not only make the code more maintainable, it
will also be generalizable to more effects if needed.

Once both the formal modeling and compiler refactorings conclude, we'll begin
drafting an RFC for effect-generic trait definitions. We expect this to happen
sometime in 2024.

And that's the end of this talk. Thank you so much for being with me all the way
to the end. None of the work in this talk would have been possible without the
following people:

- Oliver Scherer (AWS)
- Eric Holk (Microsoft)
- Niko Matsakis (AWS)
- Daan Leijen (Microsoft)

Thank you!
