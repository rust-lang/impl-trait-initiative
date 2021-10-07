# Traits and impls

You can use `impl Trait` in argument position in traits and impls, but you must use it consistently in both. For example, the following is legal:

```rust
trait Operation {
    fn compute(x: impl Iterator<Item = u32>) -> u32;
}

struct Sum;
impl Operation for Sum {
    fn compute(x: impl Iterator<Item = u32>) -> u32 {
        x.sum()
    }
}
```

But the following would be illegal:

```rust
struct Max;
impl Operation for Max {
    fn compute<I: Iterator<Item = u32>>(x: I) -> u32 {
        x.max()
    }
}
```
