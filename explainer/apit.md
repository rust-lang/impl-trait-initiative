# Impl trait in argument types

When you use an `impl Trait` in the type of a function argument, that is generally equivalent to adding a generic parameter to the function.
So this function:

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

is roughly equivalent to the following generic function:

```rust
fn sum_integers<I>(integers: I) -> u32
where
    I: Iterator<Item = u32>
{
    let mut sum = 0;
    for integer in integers {
        sum += x;
    }
    sum
}
```

Intuitively, a function that has an argument of type `impl Iterator` is saying "you can give me any sort of iterator that you like".

## Interaction with turbofish
Using impl trait 
