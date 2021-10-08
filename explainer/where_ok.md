# Appendix B: Where can impl trait be used

## Overview

We can now extend the table of impl trait positions introduced in the [type alias][tait] chapter with an additional entry:

| Position                                | Who determines the hidden type       | Status               |
| --------------------------------------- | ------------------------------------ | -------------------- |
| [Argument position][apit]               | Each caller                          | stable               |
| [Type alias][tait]                      | Code within the enclosing module     | nightly              |
| [Return position, free fns][rpit]       | The function body                    | stable               |
| [Return position, inherent impls][rpit] | The function body                    | stable               |
| [Return position, trait impls][rpit]    | The function body                    | planning rfc         |
| [Return position, traits][rpit_trait]   | The impl                             | planning rfc         |
| [Let binding][lbit]                     | The enclosing function or code block | rfc'd, unimplemented |
| [Const binding][lbit]                   | The const initializer                | rfc'd, unimplemented |
| [Static binding][lbit]                  | The static initializer               | rfc'd, unimplemented |

[apit]: ./apit.md
[tait]: ./tait.md
[rpit]: ./rpit.md
[rpit_trait]: ./rpit_trait.md

## General rules for "input" vs "output"

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
