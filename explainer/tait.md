# Impl trait in type aliases

![nightly][]

{{#include ../badges.md}}

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

| Position                         | Who determines the hidden type   |
| -------------------------------- | -------------------------------- |
| [Argument position][apit]        | The caller                       |
| Type alias                       | Code within the enclosing module |
| ... (full table in [Appendix B]) |                                  |

[apit]: ./apit.md
[Appendix B]: ./where_ok.md

