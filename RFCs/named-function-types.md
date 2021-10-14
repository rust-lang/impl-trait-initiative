# Draft RFC: Named function types

> This is a **draft RFC** that will be submitted to the rust-lang/rfcs repository when it is ready.
>
> Feedback welcome!

---

- Feature Name: (fill me in with a unique ident, `my_awesome_feature`)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

* Introduce a type for every `fn foo` named `foo`, after the `fn` itself.
    * This type corresponds to the zero-sized type of that function.
* If a type already exists with the same name as the function, no type is created.
    * As a result, no types are generated for tuple structs or enum variants (which already have corresponding types).

# Motivation
[motivation]: #motivation

"Return position" impl Trait (RPIT) generally refers to an ["anonymous" opaque type whose value is inferred by the compiler](../explaner/rpit.md):

```rust
fn odd_integers(start: u32, stop: u32) -> impl Iterator<Item = u32> {
    (start..stop).filter(|i| i % 2 == 0)
}

// becomes something like:

type OddIntegers = impl Iterator<Item = u32>;
fn odd_integers(start: u32, stop: u32) -> OddIntegers {
    (start..stop).filter(|i| i % 2 == 0)
}
```

When using [RPIT in traits](./rpit_in_traits.md), this anonymous type is a kind of associated type:

```rust
trait IntoIntIterator {
    fn into_int_iter(self) -> impl Iterator<Item = u32>;
}

// becomes

trait IntoIntIterator { // desugared
    type IntoIntIter: Iterator<Item = u32>;
    fn into_int_iter(self) -> Self::IntoIntIter;
}
```

In both cases, it would sometimes be nice to be able to name that return type! Of course, people can introduce type aliases or associated types (similar to the desugared form), but that is inconvenient, and it requires that the API author has made sure to do so.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

TODO

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## 

## 

## Late-bound and elided lifetimes

# Drawbacks
[drawbacks]: #drawbacks


## Giving values for uncaptured parameters

Impl trait in return position do not capture all 

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why not introduce the `typeof` keyword?

It has been proposed to use the `typeof` keyword to permit users to take the resulting type from arbitrary expressions. This would mean that one could fo

```rust
fn foo<T> { }
```

## Why not introduce a named type 

## Why not introduce a named type into the environment?

It is difficult to decide 

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
