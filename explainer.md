# 📚 Explainer

> The "explainer" is "end-user readable" documentation that explains how to use the feature being deveoped by this initiative.
> If you want to experiment with the feature, you've come to the right place.
> Until the feature enters "feature complete" form, the explainer should be considered a work-in-progress.

## Overview

The `impl Trait` syntax, a quick guide

| Example                                | Name                              | Description                                                           |
| -------------------------------------- | --------------------------------- | --------------------------------------------------------------------- |
| `fn foo(x: impl Trait) {...}`          | argument position                 | "input" impl trait: equivalent to a type parameter `fn foo<T: Trait>` |
| `fn foo() -> impl Trait {...}`         | return position (free function)   | "output" impl trait with the fn body as its defining scope            |
| `impl Type { fn foo() -> impl Trait }` | return position (inherent method) | "output" impl trait with the fn body as its defining scope            |
| `trait Type { fn foo(x: impl Trait) }` | argument position, trait method   | "input" impl trait: equivalent to a type parameter `fn foo<T: Trait>` |
|                                        |                                   |                                                                       |

```rust
type OddIntegers = impl Iterator<Item = u32>; /* Type alias impl Trait */

fn some_function(
    x: impl Trait,
    y: impl FnOnce(impl Debug) -> impl Debug,
    y: impl FnOnce(impl FnOnce(impl Debug)) -> impl Debug,
) -> impl Trait 
where
    T: PartialEq<impl Trait>,
    T: Iterator<Item = impl Debug>,
    F: Fn(impl Debug) -> impl Debug
{
    let x: impl Trait = ...;
}

const C: impl Trait = ...;

struct S {
    f: impl Trait
}

trait Trait {
    type Bar;
    fn method(x: &impl Trait) -> impl Trait;
}

impl Trait for () {
    type Bar = impl Trait;
    fn method(x: &impl Trait) -> impl Trait;
}
```

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

When you use `impl Trait` in an output position like this, the meaning is slightly different. Here, the function is saying "I will give you back some kind of iteratoer over integers". The difference is that the function body is the one that selects the kind of iterator in question, rather than the caller. (In this case, that type is going to be `Filter<Range<u32>, _>`, where `_` represents the anonymous type of the closure) The caller, meanwhile, doesn't know the precise type of `Iterator` they were given, only that they got "some iterator":

```rust
let i = odd_integers();

// OK -- we know that `i` is an `Iterator`
let v: u32 = i.next();

// ERROR -- This method comes from `DoubleEndedIterator`, and we
// only know that `i` implements `Iterator` (in fact, the hidden
// type *does* implement `DoubleEndedIterator`, but we can't
// rely on that).
let w: u32 = i.next_back(); 
```

### Output impl Traits: hidden types and defining scopes

In cases like this, where the `impl Trait` is mapped to a single type that is determined by the body of a function or some other thing, we say that it is an **output impl Trait**. It occurs when the `impl Trait` appears in **output position**, such as the return type of a function (we'll see some other output positions later on).

Every output impl Trait has a **hidden type** and a **defining scope**. The **hidden type** is the actual type that this `impl Trait` is inferred to represent; in the example of `odd_integers`, this hidden type was `Filter<Range<u32>, _>` (you can see that the hidden type often includes anonymous types, like the types of closures, that don't have a user-writable syntax). We call this the "hidden" type because callers don't know what it's value is, they only know that it is *some type that impements `Iterator`*. 

The **defining scope** of an `impl Trait` is the region of code that is used to determine the hidden type for the `impl Trait`. Code within this defining scope doesn't treats the `impl Trait` as an opaque, hidden type, but rather as a type to be inferred. The compiler will figure out "what value would make this code type check" and treat that as the hidden value for the impl Trait.

In the case of `odd_integers`, the defining scope for the `-> impl Iterator` is the function `odd_integers` itself. We'll see examples later on where the defining scope of an output `impl Trait` can be larger, such as an impl or even a module.

## Auto trait leakage

When talking about output impl Traits in the previous section, we said that callers cannot rely on the precise hidden type, but must instead rely only on the declared bounds from the impl Trait. This was actually a simplification: in reality, callers are able to rely on *some* aspects of the hidden type. Specifically, they are able to deduce whether the hidden type implements the various [auto traits], like `Send` and `Sync`:

[auto traits]: https://doc.rust-lang.org/nightly/reference/special-types-and-traits.html#auto-traits

```rust
fn odd_integers(start: u32, stop: u32) -> impl Iterator<Item = u32> {
    (start..stop).filter(|i| i % 2 == 0)
}

fn other_function() {
    let integers = odd_integers();

    // This requires that `integers` is `Send`.
    // The compiler "peeks" at the hidden type
    // to check that this is true.
    std::thread::spawn(move || {
        for integer in integers {
            println!("{}", integer);
        }
    }).join();
}
```

The motivation behind auto trait leakage is that it makes working with `impl Trait` as convenient as other kinds of "newtype wrappers" when it comes to threading. For example, if you were to replace the `odd_integers` return type with a newtype `OddIntegers` that hides the iterator type (as is common practice)...

```rust
struct OddIntegers {
    iterator: Filter<Range<...>, ...>
}

fn odd_integers(start: u32, stop: u32) -> OddIntegers { ... }
```

...you would find that `OddIntegers` implements `Send` if that hidden type (that appears in its private field) implements `Send`. (In this particular case, it would be challenging to create a struct because it would have to refer to the returned closure type, which is anonymous.)

## Type alias impl Trait (TAIT)

One downside of "output impl Trait" is that the returned type is anonymous. If you have a function `odd_integers`...

```rust
fn odd_integers(start: u32, stop: u32) -> impl Iterator<Item = u32> {
    (start..stop).filter(|i| i % 2 == 0)
}
```

...there is no convenient way for users to name the returned iterator type explicitly (for example, to define a struct that embeds the returned iterator as a field). Using type alias impl impl Trait allows you to give a name that others can reference:

```rust
type OddIntegers = impl Iterator<Item = u32>;

fn odd_integers(start: u32, stop: u32) -> OddIntegers {
    (start..stop).filter(|i| i % 2 == 0)
}
```

Here, the type alias `OddIntegers` is defined to alias an output impl Trait. This output impl Trait works exactly like the output impl Traits that we have seen before, except that its **defining scope** is the **enclosing module where it is defined**. Consider the following example:

```rust
mod odd {
    pub type OddIntegers = impl Iterator<Item = u32>;

    pub fn odd_integers(start: u32, stop: u32) -> OddIntegers {
        (start..stop).filter(|i| i % 2 == 0)
    }
}

struct TempStruct {
    next_odd: odd::OddIntegers,
}
```

Here, the `OddIntegers` type alias is defined inside the module `odd`. Within `odd`, functions are able to define and influence the type of `OddIntegers`. Outside of `odd`, references to `OddIntegers` are opaque -- it simply refers to "some type that implements `Iterator`".

### Impl that that are constrained by multiple functions

### 


## Details

### Output impl Traits and specialization

