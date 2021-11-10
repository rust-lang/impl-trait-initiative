# Turbofish

![needs decision][]

{{#include ../badges.md}}

Using `impl Trait` in argument position conceptually adds a generic parameter to the function. However, this generic parameter is different from explicit, named parameters. Its value cannot be specified explicitly using the turbofish operator, and must be inferred. Therefore, these two functions behave in slightly different ways:

```rust
fn foo1(x: impl Clone) { }
fn foo2<C: Clone>(x: C) { }
```

The difference is that one can write `foo2::<i32>` to manually specify the value of the `C` parameter, but the value of the `impl Clone` parameter on `foo1` must be inferred from the argument types that are supplied (e.g, `foo1(22)`).

It is possible to mix explicit generics with impl Trait:

```rust
fn foo3<A: Debug>(x: A, b: impl Clone) { }
```

In this case, one can use turbofish to specify the value of `A`, but not the `impl Clone`. In other words, one could write `foo3::<i32>` to get a function where `A` is specified as `i32`, but the value of `impl Clone` will still have to be inferred from usage.

**Implication.** The fact that impl trait and explicit generics are not equivalent means that one cannot migrate between them in a semver-compliant way, although breakage is quite unusual in practice.

**Implementation note:** This section describes the "recommended" behavior that we expect to propose to the lang team. The currently implemented semantics are different. They forbid turbofish from being used on any function that employs argument position impl trait.