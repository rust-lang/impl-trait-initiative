# Generic parameter capture

![stable][]

{{#include ../badges.md}}

When you use an impl trait in return position, the hidden type may make use of any of the type parameters, and hence the following function is legal:

```rust
// Hidden type: Option<T>, which references T
fn foo<T: Clone>(t: T) -> impl Clone {
    Some(t)
}
```

However, it may not reference lifetime parameters *unless* those lifetime parameters appear in the impl trait bounds. The following

```rust
// Error: hidden type `Option<&'a u32>` references `'a`
fn foo<'a>(t: &'a u32) -> impl Clone {
    Some(t)
}
```


XXX document:

* you can capture type parameters
* but not lifetimes, unless they appear in the bounds
