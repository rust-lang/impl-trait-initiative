# Return types in trait definitions and impls

![evaluation](https://img.shields.io/badge/status-evaluation-red)

When you use `impl Trait` as the return type for a function within a trait definition or trait impl, the semantics are somewhat different than in other cases. Consider the following trait:

```rust
trait IntoIntIterator {
    fn into_int_iter(self) -> impl Iterator<Item = u32>;
}
```

The semantics of this are analogous to introducing a new associated type within the surrounding trait;

```rust
trait IntoIntIterator { // desugared
    type IntoIntIter: Iterator<Item = u32>;
    fn into_int_iter(self) -> Self::IntoIntIter;
}
```

(In general, this associated type may be generic; it would contain whatever generic parameters are captured per the generic capture rules given previously.)

This associated type is introduced by the compiler and cannot be named by users.

The impl for a trait like `IntoIntIterator` must also use `impl Trait` in return position:

```rust
impl IntoIntIterator for Vec<u32> {
    fn into_int_iter(self) -> impl Iterator<Item = u32> {
        self.into_iter()
    }
}
```

This is equivalent to specify the value of the associated type as an `impl Trait`:

```rust
impl IntoIntIterator for Vec<u32> {
    type IntoIntIter = impl Iterator<Item = u32>
    fn into_int_iter(self) -> Self::IntoIntIter {
        self.into_iter()
    }
}
```