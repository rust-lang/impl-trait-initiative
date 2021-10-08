# Impl trait in return types

**Status:** Stable in free functions and inherent methods.

You can also use impl trait in the return type of a function:

```rust
fn odd_integers(start: u32, stop: u32) -> impl Iterator<Item = u32> {
    (start..stop).filter(|i| i % 2 != 0)
}
```

When you use `impl Trait` in an output position like this, the meaning is slightly different. Here, the function is saying "I will give you back some kind of iteratoer over integers". The difference is that the function body is the one that selects the kind of iterator in question, rather than the caller. (In this case, that type is going to be `Filter<Range<u32>, _>`, where `_` represents the anonymous type of the closure) The caller, meanwhile, doesn't know the precise type of `Iterator` they were given, only that they got "some iterator":

```rust
let mut i = odd_integers();

// OK -- we know that `i` is an `Iterator`
let v: u32 = i.next();

// ERROR -- This method comes from `DoubleEndedIterator`, and we
// only know that `i` implements `Iterator` (in fact, the hidden
// type *does* implement `DoubleEndedIterator`, but we can't
// rely on that).
let w: u32 = i.next_back(); 
```

