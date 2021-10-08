# Impl trait in type aliases

**Status:** Under active development, `type_alias_impl_trait` feature gate.

Rust has a concept of *type aliases*, which let you declare one type name as an alias for another:

```rust
type Integer = i32;
```

Type aliases can be useful for giving a short alias for some complex type. For example, imagine we had a module `odd` that defined an `odd_integers` function. `odd_integers(x, y)` returns an iterator over all the odd integers between `x` and `y`. Because the full return type is fairly complicated, the module defines a type alias `OddIntegers` that people can use to refer to it.

```rust
mod odd {
    pub type OddIntegers = std::iter::StepBy<std::ops::Range<u32>>;

    pub fn odd_integers(mut start: u32, stop: u32) -> OddIntegers {
        if (start % 2) == 0 {
            start += 1;
        }
        (start..stop).step_by(2)
    }
}
```

Of course, typing that long type is kind of annoying. Moreover, there are some types in Rust that don't have explicit names. For example, another way to write the "odd integers" iterator would be to call `filter` with a closure. But if you try to write out the type for that, you'll find that you need to give a name to the type representing the closure itself, which you can't do!

```rust
mod odd {
    pub type OddIntegers = std::iter::Filter<std::ops::Range<u32>, /* what goes here? */>;

    pub fn odd_integers(mut start: u32, stop: u32) -> OddIntegers {
        (start..stop).filter(|x| x % 2 != 0)
    }
}
```

## Enter impl Trait[^paren]

[^paren]: "He says in parentheses"

The truth is that specifying the exact type for `OddIntegers` is overkill anyway. Chances are that we don't actually *care* what exact type is used there, we only care that `odd_integers` returns *some kind of iterator*. In fact, we might *prefer* not to specify it: that gives us the freedom to change how `odd_integers` is implemented in the future without potentially affecting our callers. If this sounds familiar, that's good -- it's exactly the kind of scenario that `impl Trait` is meant to solve!

Using `impl Trait`, we can still have a type alias `OddIntegers`, but we can avoid specifying exactly what its value is. Instead, we just say that it is "some iterator over u32":

```rust
mod odd {
    pub type OddIntegers = impl Iterator<Item = u32>;
    //                 ^^^^^^^^^^^^^^^^^^^^^^^^^ :tada:!

    pub fn odd_integers(mut start: u32, stop: u32) -> OddIntegers {
        if (start % 2) == 0 {
            start += 1;
        }
        (start..stop).step_by(2)
    }
}
```

Now it's the compiler's job to figure out the value of that type alias. In this case, the type for `OddIntegers` would be inferred to `StepBy<Range<u32>>`, but we could also change the definition to use `filter`, in which case the hidden type would be `Filter<Range<u32>, C>`, where `C` represents the type of the closure. This shows something important: the value of an `impl Trait` can include types that don't otherwise have names (in this case, the closure type `C`).

## Hidden types

You can see that impl trait behaves a bit differently depending on where you use it. What all positions have in common is that they stand in for "some type that implements the given trait". We call that type the *hidden type*, because you generally don't get to rely on exactly what it is, you only get to rely on the bounds that it satisfies. What distinguishes the various positions where you can use impl Trait is which code determines the hidden type:

| Position                               | Who determines the hidden type   |
| -------------------------------------- | -------------------------------- |
| [Argument position][apit]              | The caller                       |
| [Type alias][tait]                     | Code within the enclosing module |
| ... (we'll extend this table as we go) |                                  |

[apit]: ./apit.md

## Inferring the hidden type

As the table says, when you use `impl Trait` in an argument position, the hidden type is determined by the caller of the function (just like other generic parameters). When you use `impl Trait` in a type alias, the hidden type is inferred based on the code in the enclosing module. We call this enclosing module the **defining scope** for the impl trait. In our example, the defining scope is the module `odd`:

```rust
mod odd {
    pub type OddIntegers = impl Iterator<Item = u32>;
    pub fn odd_integers(mut start: u32, stop: u32) -> OddIntegers { .. }
}
```

As we type-check the functions within the module `odd` (or its submodules), we look at the ways that they use the type `OddIntegers`. From this we can derive a set of constraints on what the hidden type has to be. For example, the compiler computes that `odd_integers` returns a value `StepBy<Range<u32>>`, but the function is declared to return `OddIntegers`: therefore we conclude that `odd_integers` requires the hidden type to be `StepBy<Range<u32>>`.

### There can be multiple functions that constrain the type

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

### Each function within the defining scope must specify the same type

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

### Each function within the defining scope must specify the complete type

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

## Referencing the type alias outside of the module

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

