# 📚 Explainer

> The "explainer" is "end-user readable" documentation that explains how to use the feature being deveoped by this initiative.
> If you want to experiment with the feature, you've come to the right place.
> Until the feature enters "feature complete" form, the explainer should be considered a work-in-progress.

## Impl trait: make the compiler do the hard work of figuring out the exact type

Oftentimes, you have a situation where you don't care *exactly* what type something has, you simply care that it implements particular traits. For example, you might have a function that accepts any kind of iterator as long as it produces integers. In this case, you could write the following:

```rust
fn sum_integers(integers: impl Iterator<Item = u32>) -> u32 {
    //                    ^^^^^^^^^^^^^
    let mut sum = 0;
    for integer in integers {
        sum += x;
    }
    sum
}
```

Here, the `impl Iterator` type is a special sort of type. It doesn't actually name any particular type. Instead, it is a *shorthand* for a *generic type*. In other words, the function above is roughly equivalent to the following desugared version:

```rust
fn sum_integers<I>(integers: I) -> u32
where
    I: Integers<Item = u32>
{
    let mut sum = 0;
    for integer in integers {
        sum += x;
    }
    sum
}
```

Intuitively, a function that has an argument of type `impl Iterator` is saying "you can give me any sort of iterator that you like".

## Impl trait in return types

You can also use impl trait in the return type of a function:

```rust
fn odd_integers(start: u32, stop: u32) -> impl Iterator<Item = u32> {
    (start..stop).filter(|i| i % 2 == 0)
}
```

When you use `impl Trait` in an output position like this, the meaning is slightly different. Here, the function is saying "I will give you back some kind of integer". The difference is that the *function*