# Appendix B: Where can impl trait be used

{{#include ../badges.md}}

## Overview

Impl trait is accepted in the following locations:

| Position                                | Who determines the hidden type       | Role     | Status                                       |
| --------------------------------------- | ------------------------------------ | -------- | -------------------------------------------- |
| [Argument position][apit]               | Each caller                          | [Input]  | ![stable][]                                  |
| [Type alias][tait]                      | Code within the enclosing module     | [Output] | ![nightly][]                                 |
| [Return position, free fns][rpit]       | The function body                    | [Output] | ![stable][]                                  |
| [Return position, inherent impls][rpit] | The function body                    | [Output] | ![stable][]                                  |
| [Return position, trait impls][rpit]    | The function body                    | [Output] | [![planning rfc][]](../RFCs/rpit-in-traits.md) |
| [Return position, traits][rpit_trait]   | The impl                             | [Input]  | [![planning rfc][]](../RFCs/rpit-in-traits.md) |
| [Let binding][lbit]                     | The enclosing function or code block | [Output] | ![accepted rfc][]                            |
| [Const binding][lbit]                   | The const initializer                | [Output] | ![accepted rfc][]                            |
| [Static binding][lbit]                  | The static initializer               | [Output] | ![accepted rfc][]                            |

[apit]: ./apit.md
[tait]: ./tait.md
[rpit]: ./rpit.md
[rpit_trait]: ./rpit_trait.md
[lbit]: ./lbit.md
[input]: #input-impl-trait
[output]: #output-impl-trait

Impl trait is not accepted in other locations; [Appendix C](./where_not_ok.md) covers some of the locations where `impl Trait` is not presently accepted and why.

<a name="role">

## Role: Input vs output

</a>

Impl traits in general play one of two roles.

### Input impl trait

An *input* impl trait corresponds loosely to a generic parameter. The code that references the impl Trait may be instantiated multiple times with different values. For example, a function using impl Trait in [argument position](./apit.md) can be called with many different types for the impl Trait.

### Output impl trait

An *output* impl trait plays a role similar to a type that is given by the user. In this case, the impl trait represents a single type (although that type may be relative to generic types are in scope) which is inferred from the code around the definition. For example, a free function with an impl Trait in [return position](./rpit.md) will have the true return type inferred from the function body. The type represented by an output impl trait is called the *hidden type*. The code that is used to infer the value for an output impl trait is called its *defining scope*.

### General rules for "input" vs "output"

In general, the role of `impl Trait` and `'_` both follow the same rules in terms of being "input vs output".  When in an argument listing, that is an "input" role and they correspond to a fresh parameter in the innermost binder. Otherwise, they are in "output" role and the corresponding to something which is inferred or selected from context (in the case of `'_` in return position, it is selected based on the rules of lifetime elision; `'_` within a function body corresponds to inference).

## Type alias impl trait

```rust
type Foo = impl Trait;
```

Creates a opaque type whose value will be inferred by the contents of the enclosing module (and its submodules).

## Fn argument position

```rust
fn foo(x: impl Trait)
```

becomes an "anonymous" generic parameter, analogous to

```rust
fn foo<T: Trait>(x: T)
```

However, when `impl Trait` is used on a function, the resulting type parameter cannot be specified using "turbofish" form; its value must be inferred. (**status:** this detail not yet decided).

Places this can be used:

* Top-level functions and inherent methods
* Trait methods
    * Implication: trait is not dyn safe

## Fn return position

```rust
fn foo() -> impl Trait)
```

becomes an "anonymous" generic parameter, analogous to

```rust
fn foo<T: Trait>(x: T)

type Foo = impl Trait; // defining scope: just the fn
fn foo() -> Foo
```

Places this can be used:

* Top-level functions and inherent methods
* Trait methods (pending; no RFC)

## Let bindings

```rust
fn foo() {
    let x: impl Debug = ...;
}
```

becomes a type alias impl trait, analogous to

```rust
type Foo = impl Debug; // scope: fn body
fn foo() {
    let x: Foo = ...;
}
```
