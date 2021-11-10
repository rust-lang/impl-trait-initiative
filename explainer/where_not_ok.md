# Appendix C: Where can impl trait NOT be used

Where is impl Trait not (yet?) accepted and why.

### dyn Types, angle brackets

```rust
fn foo(x: &mut dyn Iterator<Item = impl Debug>)
```

### dyn Types, parentheses

```rust
fn foo(x: &mut dyn FnMut(impl Debug))
```

Unclear whether this should (eventually) be short for `dyn for<T: Debug> FnMut(T)` (which would not be legal) to stay analogous to `impl FnOnce(impl Debug)`.

### dyn Types, return types

```rust
fn foo(x: &mut dyn FnMut() -> impl Debug)
```

### Nested impl trait, parentheses

```rust
fn foo(x: impl FnMut(impl Debug))
```

Unclear whether this should (eventually) be short for `impl for<T: Debug> FnMut(T)` or some other notation.

### Where clauses, angle brackets

```rust
fn foo()
where T: PartialEq<impl Clone>
```

### Where clauses, parentheses

```rust
fn foo()
where T: FnOnce(impl Clone)
```

### Where clauses, return position

```rust
fn foo()
where T: FnOnce() -> impl Clone
```

### Struct fields

```rust
struct Foo {
    x: impl Debug
}
```

It would be plausible to accept this as equivalent to a type alias at module scope, which seems reasonable, but it is easy to write that explicitly, and there may be future meanings that are more interesting (such as an existential type local to the struct).

It is not possible to accept this in *input* position, because that would imply that the type `Foo` has more generic parameters than are written on the type itself. Unlike with functions, where the values of those generic parameters can be inferred from the arguments, structs can appear in many contexts where inferring the values of those generic types would not be tractable.


