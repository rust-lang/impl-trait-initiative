# Appendix B: Where can impl trait be used

What follows is a full set of locations where impl Trait can be used, and what they mean in that position.

| Example                                           | Name                              | Stage      | Description                                                           |
| ------------------------------------------------- | --------------------------------- | ---------- | --------------------------------------------------------------------- |
| `type Foo = impl Trait`                           | top-level type alias              | nightly    | "output" impl trait, inferred from code within the enclosing item     |
| `impl Trait for Type { type Foo = impl Trait }`   | associated type                   | nightly    | "output" impl trait, inferred from code within the impl               |
| `fn foo(x: impl Trait) {...}`                     | argument position                 | stable     | "input" impl trait: equivalent to a type parameter `fn foo<T: Trait>` |
| `trait Trait { fn foo(x: impl Trait) }`           | argument position, trait method   | stable     | "input" impl trait: equivalent to a type parameter `fn foo<T: Trait>` |
| `impl Trait for Type  { fn foo() -> impl Trait }` | argument position, trait impl     | stable     | "input" impl trait: equivalent to a type parameter `fn foo<T: Trait>` |
| `fn foo() -> impl Trait {...}`                    | return position (free function)   | stable     | "output" impl trait with the fn body as its defining scope            |
| `impl Type { fn foo() -> impl Trait }`            | return position (inherent method) | stable     | "output" impl trait with the fn body as its defining scope            |
| `trait Trait { fn foo() -> impl Trait }`          | return position, trait method     | evaluation | "output" impl trait, equivalent to an associated type                 |
| `impl Trait for Type  { fn foo() -> impl Trait }` | return position, trait impl       | evaluation | "output" impl trait, equivalent to an associated type                 |
| `let x: impl Trait`                               | let binding                       | rfc'd      | "output" impl trait, inferred from enclosing function                 |                                                                       
| `const x: impl Trait`                             | type of const                     | rfc'd      | "output" impl trait, inferred from enclosing function                 |                                                                       
| `static x: impl Trait`                            | type of static                    | rfc'd      | "output" impl trait, inferred from enclosing function                 |                                                                       

More details follow.

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
