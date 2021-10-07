# 📚 Explainer

> The "explainer" is "end-user readable" documentation that explains how to use the feature being developed by this initiative.
> If you want to experiment with the feature, you've come to the right place.
> Until the feature enters "feature complete" form, the explainer should be considered a work-in-progress.

## Overview

This explainer describes the "impl Trait" feature as a complete unit. However, not all parts of this story are in the same state. Some parts are available on stable, others are available on nightly, others have accepted RFCs but are not yet implemented, and still others are in the exploration phase. Each section of the explainer has a "status" tag that indicates whether the status of that particular content.

## Impl trait: make the compiler do the hard work of figuring out the exact type

**Status:** Stable

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

**Status:** Stable in free functions and inherent methods.

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

**Status:** Stable.

In cases like this, where the `impl Trait` is mapped to a single type that is determined by the body of a function or some other thing, we say that it is an **output impl Trait**. It occurs when the `impl Trait` appears in **output position**, such as the return type of a function (we'll see some other output positions later on).

Every output impl Trait has a **hidden type** and a **defining scope**. The **hidden type** is the actual type that this `impl Trait` is inferred to represent; in the example of `odd_integers`, this hidden type was `Filter<Range<u32>, _>` (you can see that the hidden type often includes anonymous types, like the types of closures, that don't have a user-writable syntax). We call this the "hidden" type because callers don't know what it's value is, they only know that it is *some type that impements `Iterator`*. 

The **defining scope** of an `impl Trait` is the region of code that is used to determine the hidden type for the `impl Trait`. The hidden type is inferred as part of type-checking the functions and other code blocks within that defining scope. The compiler tries to find some hidden type H such that replacing the `impl Trait` with that hidden type would still make the code type check.

In the case of `odd_integers`, the defining scope for the `-> impl Iterator` is the function `odd_integers` itself. We'll see examples later on where the defining scope of an output `impl Trait` can be larger, such as an impl or even a module, and we can dig into the inference of hidden types more then.

### Generic parameter capture

**Status:** Stable.

When you use an impl trait in return position, there are some limitations on the "hidden type" that may be used. 

### Output impl trait in trait functions

**Status:** Not yet proposed in RFC form.

You can use impl Trait as the return type for a trait method:

```rust
trait IntoIntIterator {
    fn iter(self) -> impl Iterator<Item = u32>;
}
```

When 
Semantically, this is equivalent to 

## Type alias impl Trait (TAIT)

**Status:** Under active development, `type_alias_impl_trait` feature gate.

One downside of "output impl Trait" is that the returned type is anonymous. If you have a function `odd_integers`...

```rust
fn odd_integers(start: u32, stop: u32) -> impl Iterator<Item = u32> {
    (start..stop).filter(|i| i % 2 == 0)
}
```

...it can be inconvenient for users to name the returned iterator type explicitly (for example, to define a struct that embeds the returned iterator as a field). Using type alias impl Trait allows you to give a name that others can reference:

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

Because the defining scope of type alias impl trait is an entire module, it is possible to have multiple functions that can be used to infer the hidden type. The Rust rule is that each of those functions must fully specify the hidden type, and all the functions must specify the *same* hidden type (for details on how type inference works, see the section below).

### Generics and type alias impl trait

**Status:** Implemented on nightly with `type_alias_impl_trait` feature gate.

Type alias impl Traits can also be generic:

```rust
type SomeTupleIterator<I, J> = impl Iterator<Item = (I, J)>;
```

The values that you use for these generic parameters, however, are subject to some restrictions. In particular, type aliases can only be used with other generic types that are in scope, and each parameter must be distinct. Example:

```rust
fn foo1<A, B>() -> SomeTupleIterator<A, B> { /* ok */ }
fn foo2<A, B>() -> SomeTupleIterator<B, A> { /* ok */ }
fn foo3<A, B>() -> SomeTupleIterator<A, A> { /* not ok -- same parameter used twice */ }
fn foo4<A, B>() -> SomeTupleIterator<A, u32> { /* not ok -- u32 is not a type parameter */ }
```

These rules ensure that inference is tractable. Consider the case of `-> SomeTupleIterator<A, u32>`. Imagine that `foo4` returned an iterator over `(A, u32)` tuples. How do we translate that value to the hidden type for `SomeTupleIterator`?  There would actually be multiple possibilities. Either it would be an iterator over `(I, J)` (with `I = A` and `J = u32` in this case), or an iterator over `(I, u32)` (with `I = A` and `J` unused).

## Impl trait in let bindings

**Status:** Accepted RFC, not yet implemented

You can also use `impl Trait` in the type of a local variable:

```rust
let x: impl Clone = 22_i32;
```

This is equivalent to introducing a type alias impl trait with the scope of the enclosing function:

```rust
type X = impl Clone;
let x: X = 22_i32;
```

## Return position impl Trait in trait definitions and impl

**Status:** RFC not yet written.

When you use `impl Trait` as the return type for a function within a trait definition or trait impl, the semantics are somewhat different than in other cases. Consider the following trait:

```rust
trait IntoIntIterator {
    fn into_int_iter(self) -> impl Iterator<Item = u32>;
}
```

The semantics of this are analogous to introducing a new associated type within the surrounding trait;

```rust
trait IntoIntIterator { // desugared
    type IntoIntIter: Iterator<Item = u32>;
    fn into_int_iter(self) -> Self::IntoIntIter;
}
```

(In general, this associated type may be generic; it would contain whatever generic parameters are captured per the generic capture rules given previously.)

This associated type is introduced by the compiler and cannot be named by users.

The impl for a trait like `IntoIntIterator` must also use `impl Trait` in return position:

```rust
impl IntoIntIterator for Vec<u32> {
    fn into_int_iter(self) -> impl Iterator<Item = u32> {
        self.into_iter()
    }
}
```

This is equivalent to specify the value of the associated type as an `impl Trait`:

```rust
impl IntoIntIterator for Vec<u32> {
    type IntoIntIter = impl Iterator<Item = u32>
    fn into_int_iter(self) -> Self::IntoIntIter {
        self.into_iter()
    }
}
```

## Naming return position impl trait

**Status:** RFC not yet written.

Return position impl Trait introduces types that do not have proper names. If you find yourself frequently giving that type a name, your best bet is to introduce a type alias. However, in a pinch, it is possible to access those types by getting the type for the surrounding function and extracting its `FnOnce::Output` associated type. Given a function like...

```rust
fn make_iter() -> impl Iterator<Item = u32> {
    0 .. 100
}
```

...one could use the type `make_iter::Output` to access the inferred `impl Iterator` (which will be a `Range<u32>`, in this case).

Function types can generally be named via their path:

* For function items, simply the name of the function (`fn_name`)
* For inherent methods, the fully qualified syntax of `Type::fn_name`
* For trait methods defined in an impl, use the fully qualified syntax `<Type as Trait>::fn_name` (as if `fn_name` were an associated type)

In each case, the type for the function is a zero-sized type that implements the `Fn`, `FnMut`, and `FnOnce` traits. This type is considered to be defined in its surrounding module and it is also possible to use it in other contexts, such as to implement the `Default` trait:

```rust
fn foo() -> u32 { 22 }

impl Default for foo {
    fn default() -> Self {
        foo
    }
}
```

## Auto trait leakage

**Status:** Stable.

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

## Inference details

**Status:** Under active development; stable features should be compatible with this description. This description varies slightly from the accepted RFCs and will have to be formalized.

The process to infer the hidden type for an impl Trait is as follows.

* When type-checking functions or code within the *defining scope* of an impl Trait `X`:
    * Accumulate subtyping constraints from type check:
        * For example, `let var: X = 22_i32` would create the subtyping constraint that `i32` is a subtype of `X`.
        * Similarly, given `fn foo() -> X`, `let var: i32 = foo()` wold create the subtyping constraint that `X` be a subtype of `i32`.
    * The type `X` is assumed to implement the traits that appear in its bounds
        * Exception: auto traits. Proving an auto trait `X: Send` is done by "revealing" the hidden type and proving the auto trait against the hidden type.

## Appendix A. Where can impl Trait be used, and what does it mean?

What follows is a full set of locations where impl Trait can be used, and what they mean in that position.

| Example                                           | Name                              | Description                                                           |
| ------------------------------------------------- | --------------------------------- | --------------------------------------------------------------------- |
| `type Foo = impl Trait`                           | top-level type alias              | "output" impl trait, inferred from code within the enclosing item     |
| `impl Trait for Type { type Foo = impl Trait }`   | associated type                   | "output" impl trait, inferred from code within the impl               |
| `fn foo(x: impl Trait) {...}`                     | argument position                 | "input" impl trait: equivalent to a type parameter `fn foo<T: Trait>` |
| `trait Trait { fn foo(x: impl Trait) }`           | argument position, trait method   | "input" impl trait: equivalent to a type parameter `fn foo<T: Trait>` |
| `impl Trait for Type  { fn foo() -> impl Trait }` | argument position, trait impl     | "input" impl trait: equivalent to a type parameter `fn foo<T: Trait>` |
| `fn foo() -> impl Trait {...}`                    | return position (free function)   | "output" impl trait with the fn body as its defining scope            |
| `impl Type { fn foo() -> impl Trait }`            | return position (inherent method) | "output" impl trait with the fn body as its defining scope            |
| `trait Trait { fn foo() -> impl Trait }`          | return position, trait method     | "output" impl trait, equivalent to an associated type                 |
| `impl Trait for Type  { fn foo() -> impl Trait }` | return position, trait impl       | "output" impl trait, equivalent to an associated type                 |
| `let x: impl Trait`                               | let binding                       | "output" impl trait, inferred from enclosing function                 |
| `const x: impl Trait`                             | type of const                     | "output" impl trait, inferred from enclosing function                 |
| `static x: impl Trait`                            | type of static                    | "output" impl trait, inferred from enclosing function                 |

More details follow.

### General rules for "input" vs "output"

In general, the role of `impl Trait` and `'_` both follow the same rules in terms of being "input vs output".  When in an argument listing, that is an "input" role and they correspond to a fresh parameter in the innermost binder. Otherwise, they are in "output" role and the corresponding to something which is inferred or selected from context (in the case of `'_` in return position, it is selected based on the rules of lifetime elision; `'_` within a function body corresponds to inference).

### Type alias impl trait

```rust
type Foo = impl Trait;
```

Creates a opaque type whose value will be inferred by the contents of the enclosing module (and its submodules).

### Fn argument position

```rust
fn foo(x: impl Trait)
```

becomes an "anonymous" generic parameter, analogous to

```rust
fn foo<T: Trait>(x: T)
```

However, when `impl Trait` is used on a function, the resulting type parameter cannot be specified using "turbofish" form; its value must be inferred. (**status:** this detail not yet decided).

Places this can be used:

* Top-level functions and inherent methods
* Trait methods
    * Implication: trait is not dyn safe

### Fn return position

```rust
fn foo() -> impl Trait)
```

becomes an "anonymous" generic parameter, analogous to

```rust
fn foo<T: Trait>(x: T)

type Foo = impl Trait; // defining scope: just the fn
fn foo() -> Foo
```

Places this can be used:

* Top-level functions and inherent methods
* Trait methods (pending; no RFC)

### Let bindings

```rust
fn foo() {
    let x: impl Debug = ...;
}
```

becomes a type alias impl trait, analogous to

```rust
type Foo = impl Debug; // scope: fn body
fn foo() {
    let x: Foo = ...;
}
```

## Appendix B. Stability and status

| Location                       | Stability and status |
| ------------------------------ | -------------------- |
| fn arguments: free functions   | stable               |
| fn return: free functions      | stable               |
| fn arguments: inherent methods | stable               |
| fn return: inherent methods    | stable               |
| type alias impl trait          | `feature()`          |
| fn arguments: trait methods    | evaluation           |
| fn return: trait methods       | evaluation           |
| fn arguments: trait methods    | evaluation           |
| fn return: trait methods       | evaluation           |
| let bindings                   | RFC, no impl         |

## Appendix C

Where is impl Trait not (yet?) accepted and why.

### dyn Types, angle brackets

```rust
fn foo(x: &mut dyn Iterator<Item = impl Debug>)
```

### dyn Types, parentheses

```rust
fn foo(x: &mut dyn FnMut(impl Debug))
```

Unclear whether this should (eventually) be short for `dyn for<T: Debug> FnMut(T)` (which would not be legal) to stay analogous to `impl FnOnce(impl Debug)`.

### dyn Types, return types

```rust
fn foo(x: &mut dyn FnMut() -> impl Debug)
```

### Nested impl trait, parentheses

```rust
fn foo(x: impl FnMut(impl Debug))
```

Unclear whether this should (eventually) be short for `impl for<T: Debug> FnMut(T)` or some other notation.

### Where clauses, angle brackets

```rust
fn foo()
where T: PartialEq<impl Clone>
```


### Where clauses, parentheses

```rust
fn foo()
where T: FnOnce(impl Clone)
```

### Where clauses, return position

```rust
fn foo()
where T: FnOnce() -> impl Clone
```

### Struct fields

```rust
struct Foo {
    x: impl Debug
}
```

It would be plausible to accept this as an "output" position, which also seems correct. The main reason that it is not accepted is that it has not been proposed, and there aren't a strong body of use cases. This is also the role that would be predicted from the "general rule for input vs output".

It is not possible to accept this in *input* position, because that would imply that the type `Foo` has more generic parameters than are written on the type itself. Unlike with functions, where the values of those generic parameters can be inferred from the arguments, structs can appear in many contexts where inferring the values of those generic types would not be tractable.


