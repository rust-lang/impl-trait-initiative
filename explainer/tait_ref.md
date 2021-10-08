# Referencing the type alias outside of the module

![nightly][]

{{#include ../badges.md}}

Since it is declared to be public, the type alias `OddIntegers` can be referenced from outside the module `odd`. This allows callers to give a name to the return type of `odd_integers`:

```rust
mod odd {
    pub type OddIntegers = impl Iterator<Item = u32>;
    pub fn odd_integers(mut start: u32, stop: u32) -> OddIntegers { /* as above */ }
}

fn main() {
    let odds: odd::OddIntegers = odd::odd_integers(1, 10);
    for i in odds {
        println!("{}", i);
    }
}
```

Because that code is outside of the defining scope, however, it is not allowed to influence or observe the hidden type. It can only rely on the that were declared[^auto] (e.g., `Iterator<Item = u32>`). For example, the following code would not type check:

[^auto]: With the exception of [auto traits](./auto_trait.md).

```rust
mod odd {
    pub type OddIntegers = impl Iterator<Item = u32>;
    pub fn odd_integers(mut start: u32, stop: u32) -> OddIntegers { /* as above */ }
}

fn main() {
    // Error!
    let odds: std::iter::StepBy<std::iter::Range<u32>> =
        odd::odd_integers(1, 10);
    for i in odds {
        println!("{}", i);
    }
}
```

This code fails because `odds` is not known to have that exact type -- `main` can only see that `odds` returns "some kind of iterator". 

