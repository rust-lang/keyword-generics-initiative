# 📜 keyword generics Charter

_Not being able to write code which is generic over keywords limits what we can
express in Rust, and we should seek to improve that._
<!--
 Provide an introduction summarising the goals and motivation behind your
 initiative.
-->

## Proposal

We're in the process of adding new features to Rust. The Const WG is creating an
extension to Rust which enables arbitrary computation at compile time.
While the Async WG is in the process of adding capabilities for asynchronous
computation. We've noticed that both these efforts have a lot in common, and may
in fact require similar solutions. This document describes a framework for
thinking about these language features, describes their individual needs, and
makes the case that we should be considering a generalized language design for
"keywords" (aka "definitely not effects"), so that we can ensure that the Rust
language and standard library remain consistent in the face of extensions.

## A broad perspective on language extensions

`const fn` and `async fn` are similar language extensions, but the way they
extend the language is different:

- `const fn` creates a *subset* of "base Rust", enabling functions to be
executed during compilation. `const` functions can be executed in "base"
contexts, while the other way around isn't possible.
- `async fn` creates a *superset* of "base Rust", enabling functions to be
executed asynchronously. `async` types cannot be executed in "base" contexts
[^bridge], but "base" in `async` contexts _is_ possible.

[^bridge]: In order to bridge async and non-async Rust, functionality such as
`thread::block_on` or `async fn` must be used, which runs a future to completion
from a synchronous context. `const` Rust does not require such a bridge, since
the difference in contexts is "compile time" and "run-time".

```
                      +---------------------------+                               
                      | +-----------------------+ |                               
                      | | +-------------------+ | |     - numbers                 
                      | | |                   | | |     - containers (soon)      
                      | | |    const Rust     |-------{ - functions               
                      | | |                   | | |     - control flow            
 - networking         | | +-------------------+ | |     - traits                  
 - filesystem         | |                       | |     - types                   
 - threads     }--------|      "base" Rust      | |                               
 - system time        | |                       | |                               
 - statics            | +-----------------------+ |                               
                      |                           |     - ad-hoc concurrency      
                      |        async Rust         |---{ - ad-hoc cancellation     
                      |                           |     - ad-hoc parking/unparking
                      +---------------------------+
```

In terms of standard library these relationships also mirror each other. "Base"
Rust will want to do everything during runtime what `const` rust can do, but in
addition to that also things like network and filesystem IO. Async Rust will in
turn want to do everything "base" Rust can do, but in addition to that will also
want to introduce methods for ad-hoc concurrency, cancellation, and execution
control. It will also want to do things which are blocking in "base" Rust as
non-blocking in async Rust.

And it doesn't stop with `const` and `async` Rust; it's not hard to imagine that
other annotations for "can this panic", "can this return an error", "can this
yield values" may want to exist as well. All of which would present extensions
to the "base" Rust language, which would need to be introduced in a way which
keeps it feeling like a single language - instead of several disjoint languages
in a trenchcoat.

## Membership

| Role      | Github                            |
|-----------|-----------------------------------|
| [Owner]   | [Yosh Wuyts](https://github.com/yoshuawuyts) |
| [Owner]   | [Oli Scherer](https://github.com/oli-obk) |
| [Liaison] | [Niko Matsakis?](https://github.com/nikomatsakis) |

[Owner]: https://lang-team.rust-lang.org/initiatives/process/roles/owner.html
[Liaison]: https://lang-team.rust-lang.org/initiatives/process/roles/liaison.html
