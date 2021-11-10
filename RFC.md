# ✨ RFCs

The design for impl trait has been defined by a number of RFCs and decisions. This repository represents the "accumulated state" of all these RFCs, and also includes drafts of new RFCs that are in the works. Sometimes, though, it is useful to return to an RFC to see the details. Here is a comprehensive listing of the impl trait RFCs along with some of the core ideas they describe. Please keep in mind that details of the implementation or design may have been amended since the RFC was accepted, however.

## Accepted RFCs

* [RFC 1522]: Proposed `-> impl Trait` notation in return position for free functions and inherent methods, inclued how [auto traits "leak" through impl Trait](./explainer/auto_trait.md).
* [RFC 1951]: Proposed [argument position impl trait](./explainer/apit.md) with a uniform syntax for [input vs output role](./where_ok.md#role-input-vs-output). Defined the [capture rules](./explainer/rpit_capture.md) regarding which generic parameters are in scope for [return position impl trait](./explainer/rpit.md).
* [RFC 2071]: Proposed ["type alias impl trait"](./explainer/tait.md) and defined the core idea of a hidden type that is inferred from multiple functions. Established the principle that each of those functions should independently define the full hidden type. Also introduced the ability to have `impl Trait` in [let, const, and static bindings](./explainer/lbit.md). The syntax at the time was `existential type`, later revised by [RFC 2515].
* [RFC 2515]: Revised the syntax from `existential type Foo: Trait` to `type Foo = impl Trait`.

[RFC 1522]: https://github.com/rust-lang/rfcs/blob/master/text/1522-conservative-impl-trait.md
[RFC 1951]: https://github.com/rust-lang/rfcs/blob/master/text/1951-expand-impl-trait.md
[RFC 2071]: https://github.com/rust-lang/rfcs/blob/master/text/2071-impl-trait-existential-types.md
[RFC 2515]: https://github.com/rust-lang/rfcs/blob/master/text/2515-type_alias_impl_trait.md

## Draft RFCs

* [Return position impl trait in traits](./RFCs/rpit-in-traits.md): Extends return position impl Trait to be usable in traits and trait impls. ([explainer](./explainer/rpit_trait.md))
* [Named function types](./RFCs/named-function-types.md): Introduces a way to name function types, and hence their return types, which in turn permits ["return position impl trait"](./explainer/rpit.md) to be named.