# Generic parameters and type alias impl trait

![nightly][]

{{#include ../badges.md}}

Type alias impl Traits can also be generic:

```rust
type SomeTupleIterator<I, J> = impl Iterator<Item = (I, J)>;
```

The hidden type inferred for an impl trait that appears in a type alias may always reference any of the generic parameters from that type alias.

## Limits on the generic arguments to a type alias impl trait

References to a type aliases that contain an impl trait, however, are subject to some restrictions. In particular, those type aliases can only be used with other generic types that are in scope, and each parameter must be distinct. Example:

```rust
fn foo1<A, B>() -> SomeTupleIterator<A, B> { /* ok */ }
fn foo2<A, B>() -> SomeTupleIterator<B, A> { /* ok */ }
fn foo3<A, B>() -> SomeTupleIterator<A, A> { /* not ok -- same parameter used twice */ }
fn foo4<A, B>() -> SomeTupleIterator<A, u32> { /* not ok -- u32 is not a type parameter */ }
```

These rules ensure that inference is tractable. Consider the case of `-> SomeTupleIterator<A, u32>`. Imagine that `foo4` returned an iterator over `(A, u32)` tuples. How do we translate that value to the hidden type for `SomeTupleIterator`?  There would actually be multiple possibilities. Either it would be an iterator over `(I, J)` (with `I = A` and `J = u32` in this case), or an iterator over `(I, u32)` (with `I = A` and `J` unused).
