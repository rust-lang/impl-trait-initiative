# Appendix B: Where can impl trait NOT be used

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

It would be plausible to accept this as an "output" position, which also seems correct. The main reason that it is not accepted is that it has not been proposed, and there aren't a strong body of use cases. This is also the role that would be predicted from the "general rule for input vs output".

It is not possible to accept this in *input* position, because that would imply that the type `Foo` has more generic parameters than are written on the type itself. Unlike with functions, where the values of those generic parameters can be inferred from the arguments, structs can appear in many contexts where inferring the values of those generic types would not be tractable.


