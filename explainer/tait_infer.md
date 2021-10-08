# Inferring the hidden type

![nightly](https://img.shields.io/badge/status-nightly-red)

When you use `impl Trait` in a type alias, the hidden type is inferred based on the code in the enclosing module. We call this enclosing module the **defining scope** for the impl trait. In our example, the defining scope is the module `odd`:

```rust
mod odd {
    pub type OddIntegers = impl Iterator<Item = u32>;
    pub fn odd_integers(mut start: u32, stop: u32) -> OddIntegers { .. }
}
```

As we type-check the functions within the module `odd` (or its submodules), we look at the ways that they use the type `OddIntegers`. From this we can derive a set of constraints on what the hidden type has to be. For example, the compiler computes that `odd_integers` returns a value `StepBy<Range<u32>>`, but the function is declared to return `OddIntegers`: therefore we conclude that `odd_integers` requires the hidden type to be `StepBy<Range<u32>>`.

## There can be multiple functions that constrain the type

Of course, modules can have more than one function! It's fine to have multiple functions that constrain the type `OddIntegers`, so long as they all ultimately constrain it to be the same thing. It's also possible to constrain `OddIntegers` by referencing it in other places. For example, consider the function `another_function` here:

```rust
mod odd {
    pub type OddIntegers = impl Iterator<Item = u32>;

    // Requires `OddIntegers` to be `StepBy<Range<u32>>`
    pub fn odd_integers(mut start: u32, stop: u32) -> OddIntegers {..}

    // Requires `OddIntegers` to be `StepBy<Range<u32>>`
    fn another_function() {
        let x: OddIntegers = (3..5).step_by(2);
    }
}
```

`another_function` references `OddIntegers` as the type of the variable `x` -- but the value assigned to `x` is also of type `StepBy<Range<u32>>`, so everything works.

## Each function within the defining scope must specify the same type

These two functions each independently require that `OddIntegers` has the same underlying type. What is *not* ok is to have two functions that require *different* types:

```rust
type OddIntegers = impl Iterator<Item = u32>;
//                 ^^^^^^^^^^^^^^^^^^^^^^^^^ :tada:!

fn odd_integers(mut start: u32, stop: u32) -> OddIntegers {
    if (start % 2) == 0 {
        start += 1;
    }
    // Requires `OddIntegers` to be `StepBy<Range<u32>>`
    (start..stop).step_by(2)
}

fn another_function() {
    // Requires `OddIntegers` to be `Filter<...>`.
    //
    // ERROR!
    let x: OddIntegers = (3..5).filter(|x| x % 2 != 0);
}
```

## Each function within the defining scope must specify the complete type

Similarly, it is not ok to have functions that only partially constrain and impl Trait. Consider this example:

```rust
type SomethingDebug = impl Debug;

fn something_debug() -> SomethingDebug {
    // Constrains SomethingDebug to `Option<_>`
    // (where `_` represents "some type yet to be inferred")
    None
}
```

Here, the `something_debug` function constrains `SomethingDebug` to be some sort of `Option`, but it doesn't say the full type. We only have `Option<_>`. That's not good enough. You could change this function to specify what kind of option it is, though, and that would be fine:

```rust
type SomethingDebug = impl Debug;

fn something_debug() -> SomethingDebug {
    // Constrains SomethingDebug to `Option<()>`
    None::<()>
}
```
