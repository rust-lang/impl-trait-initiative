# APIT and Turbofish

## Summary

## Introduction

`impl Trait` in argument position [introduces a new generic parameter to the function][apit]:

[apit]: ../explainer/apit.md

```rust
fn collect_to_vec<T>(iter: impl Iterator<Item = T>) -> Vec<T> {
    iter.collect()
}

// is roughly equivalent to:

fn collect_to_vec_desugared<T, I>(iter: I) -> Vec<T>
where
    I: Iterator<Item = T>,
{
    iter.collect()
}
```

With the desugared version, users can write an expression that manually specifies the values for its type parameters:

```rust
let f = collect_to_vec_desugared::<u32, vec::IntoIter<u32>>;
```

The question addressed here is whether it should be possible to write an equivalent expression for `collect_to_vec`.

## Current behavior

The compiler today prevents the use of turbofish for any function with [APIT]. This is a future compatibility restriction and not a long term plan.

## Observations

### Users should not *have* to acknowledge impl Trait during turbofish, that'd be confusing

Today, when using turbofish form, we require that users write a value for *all* generic type parameters, even if that value is `_` (we allow lifetime parameters to be elided if the user chooses, and in fact *require* them to be elided in some cases). Therefore, it would be an error to write `collect_to_vec_desugared::<u32>(vec.into_iter())`. It is allowed to write `collect_to_vec_desugared::<u32, _>(vec.into_iter())`, however, and thus to lean on type inference for one parameter.

While [APIT] is "roughly equivalent" to a new generic parameter, this parameter is not listed explicitly in the list of generics. Therefore, it would likely be surprising for users to get an error for not giving it a value. This argues that `collect_to_vec::<u32>(vec.into_iter())` should be accepted.

### If we wish to allow turbofish to specify the value for an impl Trait, we would need to linearize the list

Furthermore, if we wished to extend `impl Trait` parameters into the list of type parameters, we would have to come up with an ordering. Presumably it would be "left to right" as they are written within the arguments.

### Some type parameters are given explicit values more often than others and Rust doesn't let one make that distinction

It sometimes happens that functions have some type parameters which are useful (even mandatory) to specify, and others which are not. This usually happens when some type parameters appear in the argument list, and hence can be inferred from the call arguments, but others either do not appear anywhere or appear only in the return type. Example: 

```rust
fn log_the_things<D: DebugLog>(strings: impl IntoIterator<Item = String>) {
    for s in strings {
        D::log_str(s);
    }
}
```

Here, it would be common for users to wish to write something like `log_the_things::<SomeLoggerType>(some_vector)`, where the type `D` is manually specified but the value of the `impl IntoIterator` is inferred from the argument. Without using `impl Trait`, this would have to be written `log_the_things::<SomeLoggerType, _>(...)`.

### You sometimes want to get a function type without calling it

In all the examples so far, we have assumed that it was possible to infer the value of an `impl Trait` argument from the arguments that were supplied. But that might not always be true. Consider this rather artificial example:

```rust
struct Foo<T> {
    t: T
}

fn test_fn(x: impl Iterator) {
    
}

fn make() -> Foo<impl Debug> {
    Foo { x: test_fn }
}
```

Here, we are not calling `test_fn`, just taking its value.



## Alternatives

### Allow users to the values for impl Traits, but do not require it

### 