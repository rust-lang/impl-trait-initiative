# Impl trait in return types

![stable][]

{{#include ../badges.md}}

In the section on [type aliases][tait], we gave the example of a function `odd_integers` that returned a type alias `OddIntegers`. If you prefer, you can forego defining the type alias and simply return an `impl Trait` directly:

[tait]: ./tait.md

```rust
fn odd_integers(start: u32, stop: u32) -> impl Iterator<Item = u32> {
    (start..stop).filter(|i| i % 2 == 0)
}
```

This is *almost* equivalent to the type alias we saw before, but there are two differences:

* The **defining scope** for the impl trait is just the function `odd_integers`, and not the enclosing module.
    * This means that other functions within the same module cannot observe or constrain the hidden type.
* There is no direct way to name the resulting type (because you didn't define a type alias).
    * But see the section on [naming impl trait in return type](./rpit_names.md) for indirect techniques.

