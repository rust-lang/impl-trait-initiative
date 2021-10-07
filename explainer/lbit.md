# Impl trait in let bindings

**Status:** Accepted RFC, not yet implemented

You can also use `impl Trait` in the type of a local variable:

```rust
let x: impl Clone = 22_i32;
```

This is equivalent to introducing a type alias impl trait with the scope of the enclosing function:

```rust
type X = impl Clone;
let x: X = 22_i32;
```