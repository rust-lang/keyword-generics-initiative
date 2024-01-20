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

Why are we doing this? What use cases does it support? What is the expected outcome?

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.
- Discuss how this impacts the ability to read, understand, and maintain Rust code. Code is read and modified far more often than written; will the proposed feature make code easier to maintain?

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
