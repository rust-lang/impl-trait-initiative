# Impl trait in type aliases

**Status:** Under active development, `type_alias_impl_trait` feature gate.

One downside of "output impl Trait" is that the returned type is anonymous. If you have a function `odd_integers`...

```rust
fn odd_integers(start: u32, stop: u32) -> impl Iterator<Item = u32> {
    (start..stop).filter(|i| i % 2 != 0)
}
```

...it can be inconvenient for users to name the returned iterator type explicitly (for example, to define a struct that embeds the returned iterator as a field). Using type alias impl Trait allows you to give a name that others can reference:

```rust
type OddIntegers = impl Iterator<Item = u32>;

fn odd_integers(start: u32, stop: u32) -> OddIntegers {
    (start..stop).filter(|i| i % 2 != 0)
}
```

Here, the type alias `OddIntegers` is defined to alias an output impl Trait. This output impl Trait works exactly like the output impl Traits that we have seen before, except that its **defining scope** is the **enclosing module where it is defined**. Consider the following example:

```rust
mod odd {
    pub type OddIntegers = impl Iterator<Item = u32>;

    pub fn odd_integers(start: u32, stop: u32) -> OddIntegers {
        (start..stop).filter(|i| i % 2 != 0)
    }
}

struct TempStruct {
    next_odd: odd::OddIntegers,
}
```

Here, the `OddIntegers` type alias is defined inside the module `odd`. Within `odd`, functions are able to define and influence the type of `OddIntegers`. Outside of `odd`, references to `OddIntegers` are opaque -- it simply refers to "some type that implements `Iterator`". 

Because the defining scope of type alias impl trait is an entire module, it is possible to have multiple functions that can be used to infer the hidden type. The Rust rule is that each of those functions must fully specify the hidden type, and all the functions must specify the *same* hidden type (for details on how type inference works, see the section below).
