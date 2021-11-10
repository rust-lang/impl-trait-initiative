# Draft RFC: Named function types

> This is a **draft RFC** that will be submitted to the rust-lang/rfcs repository when it is ready.
>
> Feedback welcome!

---

- Feature Name: (fill me in with a unique ident, `my_awesome_feature`)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

* Given a function `fn foo`, introduce a type named `fn#foo` to represent the zero-sized type of the function ("function def").
    * When possible, a type namd `foo` is also introduced, unless another type with tha tname already exists.
* The generic parameters of a "function def" type are defined to include only *named* parameters. Anonymous parameters that appear in the argument are excluded:
    * Elided lifetimes in argument position are excluded.
    * `impl Trait` in argument position are excluded.
    * This implies that turbofish with a "function def" type cannot name the `impl Trait` arguments.
    * As a temporary measure, "function def" types that have named, late-bound lifetimes cannot accept lifetime parameters.
        * This rule already exists today.

# Motivation
[motivation]: #motivation

## Problems we are solving

This RFC proposes two related changes that, together, work to close two remaining major "holes" in the `impl Trait` story:

* How to name the return type of a function
* How turbofish interacts with `impl Trait` in argument position ([explainer](https://rust-lang.github.io/impl-trait-initiative/explainer/apit.html))

These two problems at first seem orthogonal, but they turn out to be related. Let's first introduce the two problems.

### Naming the return type of a function

"Return position" impl Trait (RPIT) generally refers to an ["anonymous" opaque type whose value is inferred by the compiler](../explaner/rpit.md):

```rust
fn odd_integers(start: u32, stop: u32) -> impl Iterator<Item = u32> {
    (start..stop).filter(|i| i % 2 == 0)
}

// becomes something like:

type OddIntegers = impl Iterator<Item = u32>;
fn odd_integers(start: u32, stop: u32) -> OddIntegers {
    (start..stop).filter(|i| i % 2 == 0)
}
```

When using [RPIT in traits](./rpit_in_traits.md), this anonymous type is a kind of associated type:

```rust
trait IntoIntIterator {
    fn into_int_iter(self) -> impl Iterator<Item = u32>;
}

// becomes

trait IntoIntIterator { // desugared
    type IntoIntIter: Iterator<Item = u32>;
    fn into_int_iter(self) -> Self::IntoIntIter;
}
```

In both cases, it would sometimes be nice to be able to name that return type! Of course, people can introduce type aliases or associated types (similar to the desugared form), but that is inconvenient, and it requires that the API author has made sure to do so.

### Turbofish and impl Trait in argument position

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

## Background material

This RFC covers some esoteric corners of Rust's type system. This section provides background material that introduces terms that are necessary to understand what follows.

### Function definition types

Currently, when you define a function `my_function`, this declares a value `my_function` that can be accessed by the user. The type of that value, however, is a bit complicated. Consider this program:

```rust
fn my_function() {
    println!("Hello, world");
}

fn main() {
    let f = my_function;
    println!("{}", std::mem::size_of_val(&f));

    let g: fn() = my_function;
    println!("{}", std::mem::size_of_val(&f));
}
```

Here we reference `my_function` twice, once to create a variable named `f` with an inferred type, and once to declare a variable `g` with type `fn()`. You might expect that the type of `f` would be inferred to `fn()` and that these two variables are equivalent, but if you run this program you will find that their sizes are different: 0 and 8 respectively ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=c7904fef1e661bbe82499cfadf7455ae)). What is going on?

The answer is that the type of `my_function` is actually a special zero-sized type called a *function definition type* (hereafter: fndef). There is a unique fndef type for every function in your program. These types implement the `Fn` traits, so they can be called: when you call one, the compiler knows *exactly* what function was called just from the type, since it is tied to a specific function. In contrast, the `fn()` type represents a function *pointer* -- it too can be called, but, without some kind of dataflow analysis, the compiler doesn't know what function is being called. This is analogous to using traits with generic types vs `dyn Trait`.

There is currently no syntax for a fndef type, so there is no way to write the type of `f` above. The closest you can do is an as-yet-unimplemented extension `impl Trait`, [impl trait in let bindings][lbit], which would let you write `let f: impl Fn() = my_function`.

[lbit]: https://rust-lang.github.io/impl-trait-initiative/explainer/lbit.html

#### Generic function definition types

The fndef type for a generic function is itself generic. You can specify the values for those generics using turbofish:

```rust
fn generic_fn<T>(t: T) { }

fn main() {
    let f = generic_fn::<u32>;
}
```

### Early vs late binding

Rust groups generic parameters on functions into two categories: early vs late bound. *Early bound* parameters are those whose values must be supplied when the function is referenced (either explicitly via turbofish, or implicitly with inference). *Late-bound* parameters are parameters that are bound only at the time when the function is called. At present, generic type and const parameters are always early bound; lifetime parameters can be either early or late depending on whether they are referenced outside of the argument listing (the exact rules are not pertinent to this RFC).

It is perhaps easiest to understand this by considering the impl of the `FnOnce` trait for a fndef type. Consider this example, of a function `create_with_default` that will create an instance of `C`. The input supplied to the creation function is either the string `input` or -- in some cases -- a default string.

```rust
trait Create {
    fn create(input: &str);
}

fn create_with_default<'a, C: Create>(input: &'a str) -> C {
    if input.is_empty() {
        C::create("default")
    } else {
        C::create(input)
    }
}
```

We will refer to the fndef type of `create_with_default` as `create_with_default`[^surprise]. Conceptually, there is an impl of `FnOnce` for `create_with_default` rather like so:

```rust
impl<'a, C> FnOnce<(&'a str,)> for create_with_default<C> {
    //             ^^^^^^^^^^                         ^^^
    //                 |                               |
    //                 |            Early bound type parameters
    //                 |            appear in this list.
    //                 |
    // Late bound parameters appear only in these
    // argument types.
    type Output = C;

    fn call(self, args: (&'a str,)) {
        (self)(args.0)
    }
}
```

Here there are two "input types" supplied to the trait: the argument parameter `A` to `FnOnce<A>`, which defines the types of the arguments being supplied, and the self type (`create_with_default<C>`), which defines the type of the value being called. The set of generic parameters on the impl is the same as the set on the `fn` declaration. We can classify those parameters in two ways:

* **Late-bound** parameters are those that appear **only** in the arguments -- this means that they different values can be supplied for them each time the function is called. In this example, the lifetime `'a` is late-bound.
* **Early-bound** parameters are those that appear in the `Self` type. This means that they are "baked into" the function that is being called (which always has a single type) and cannot change from call to call. In this example, `C` is early-bound.

In general, it is better for parameters to be late-bound, as that allows the user more flexibility. However, because of the rules that every impl must be [constrained][^why-constrianed], parameters that don't appear in the argument types *must* appear in the Self type, and hence be early bound. In practice, the compiler today only allows lifetimes to be late-bound; all type parameters are early bound. This desugaring however shows that it is possible to consider type parameters as late-bound as well.

[constrained]: https://doc.rust-lang.org/nightly/reference/items/implementations.html?highlight=impl#generic-implementations

[^why-constrained]: ...which are there to ensure that the compiler can always figure out what impl to invoke at monomorphization time.

[^surprise]: Surprise! This is precisely the notation that this RFC is going to propose!

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Please refer to the [impl trait explainer], which has a section on [names for return position impl trait][rn] that aims to be a "end-user manual" for this feature.

[impl trait explainer]: https://rust-lang.github.io/impl-trait-initiative/explainer.html
[rn]: https://rust-lang.github.io/impl-trait-initiative/explainer.html

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The proposal has a few parts:

* **Named fndef types:** 
    * For every function `my_function`, define both a value and a type, except when there is already a type with that name.
    * As a disambiguator, also export a type `fn#my_function` that can be used to unambiguously name the fndef type.
    * In traits, for every `fn my_function` item, introduce a `type my_function` associated type (except when there is already an associated type with that name).
* **Late-bound argument position impl Trait:** 
    * Change `impl Trait` in argument position so that it desugars to "late-bound types"
    * In so doing, settle the question of whether `impl Trait` in argument position can be constrained with turbofish (no)
* **Random bug fixes:**
    * Inherent associated types (`impl Foo { type Bar = u32; }` and you can write `Foo::Bar`).
    * Fix the `Foo::Output` code in the compiler (not really RFC worthy).

## Named fndef types

Introduce a set of changes designed to make it so that users can name the fndef type for every function by writing types whose name is the same as the name that refers to the fn by value:

* `foo` to name the type for a top-level fn `foo`.
* `Type::foo` to name the type for an inherent method `foo`.
* `<T as Trait>::foo` to name the type for a trait method `foo` (or `Trait::foo` and so forth).

### Top-level function items

Top-level function declarations like `fn my_function` will also export a value in the type namespace with the same name (`my_function`) that corresponds to the fndef type for that function. Therefore, the following works:

```rust
fn foo() {}

struct SomeType {
    f: foo // refers to the fndef type for `foo`
}

fn main() {
    let x = Sometype { f: foo };
}
```

### Function definitions in traits

Function declarations in inherent and trait impls introduce an associated type into the surrounding impl with the same name as the function whose value is the fndef type for the function:

```rust
trait Clone {
    // Introduces the equivalent of:
    //
    //     type clone;
    fn clone(&self);
}
```

### Function definitions in impls

Function declarations in inherent and trait impls introduce an associated type into the surrounding impl with the same name as the function whose value is the fndef type for the function:

```rust
struct MyType { }

impl MyType {
    // Introduces:
    //
    //     type new = <fndef for new>;
    fn new() -> Self {

    }
}

impl Clone for MyType {
    // Introduces:
    //
    //     type clone = <fndef for clone>;
    fn clone(&self) -> ... {

    }
}
```

### Shadowing rules

Although Rust's naming conventions make this unlikely, it is possible today to define a type and a function with the same name. This obviously presents a conflict in attempting to introduce types with the same names as functions. This section discusses the possible shadowing that can arise in each case and how we manage it. The general strategy is to only generate the fn type if there is not already a type defined with the same name.

#### Edition Rust.next

The plan is that in the edition Rust.next (presumably Rust 2024), all cases of shadowing will become hard errors. In the current edition, however, cases of shadowing yield warnings where needed to maintain backwards compatibility.

#### Traits and trait impls

If a trait defines an explicit associated type with the same name as one of its functions, then the associated type for that function is suppressed. This also applies to associated types defined in *supertraits*. Therefore:

```rust
trait Foo: Bar {

    // A `foo` associated type exists: 
    //
    // nothing is generated for the `fn foo`.
    type foo;
    fn foo(&self);

    // A `bar` associated type exists in the supertrait:
    //
    // nothing is generated for the `fn bar`.
    fn bar(&self);

    // The `Bar` supertrait defines an associated type `baz`, but implicitly:
    //
    // generate an associated type `baz.`
    fn baz(&self);
}

trait Bar {
    type bar;

    fn baz(&self);
}
```

In all of these cases, we issue warnings. In Rust 2024, this will be an error.

**Rationale:** Currently given `T: Foo`, `T::bar` would be legal today, so we don't want to generate a type there. But otherwise we do.

#### Inherent impls

Inherent impls do not support associated types today, so we don't have to worry about conflicts.

If an associated type defined on T has the same name as an inherent function on T, then we report an error.

#### Tuple struct and enum variants

Although not a function declaration, tuple structs and enum variants define constructor functions with the same name as the type itself:

```rust
struct Foo(u32); // defines a fn `Foo` of type `u32 -> Foo`
```

In this case, we simply do not generate a fndef type for the constructor. (This is also true in Rust 2024.)

#### Top-level functions

Top-level functions can conflict with named types in a number of ways (examples follow). In all such cases, the intent is to have the function not generate a type alias.

**Conflict with prelude** ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=6719c374699851268dc8ffc363836ffd))

```rust
fn Vec() {
    
}

fn main() {
    let x = Vec();
}
```

**Conflict with name from use statement** ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=53fc8cd1cea9bd465cbcc4c5dd49f820))

```rust
mod foo {
    pub type bar = u32;
}

use foo::bar;

fn bar() { }

fn main() {
    
}
```

**Conflict with macro-generated items** ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=e3e3cc25cf16bb4cb813854b512b5980))

```rust
mod foo {
    macro_rules! make_bar {
        () => { pub type bar = u32; }
    }
    
    make_bar!();
}

use foo::bar;

fn bar() { }

fn main() {
    let x: bar = 22_u32;
}
```

## Late-bound argument position impl Trait and turbofish

Currently, impl Trait in argument position is desugared to the equivalent of another explicit parameter:

```rust
fn foo(x: impl Debug) { }

// becomes roughly equivalent to
// fn foo<D: Debug>(x: D) { }
```

This parameter is currently considered "early-bound". As discussed in the motivation, that means that the fndef type `foo<D>` is generic over some `D: Debug`. Conceptually it looks something like this:

```rust
struct foo<D: Debug>;
impl<D: Debug> FnOnce<(D,)> for foo<D> {
    ...
}
```

This means that if you were to try and name the type of `foo`, say in a struct, you would need some to way to specify the value of `D`:

```rust
struct Wrapper {
    f: foo // What is the value of D here?
}
```

We could allow you to write `foo<u32>`, but that would be strange, because if you look at the definition of `foo`, there are no declared type parameters. What's more, if there were multiple `impl Trait`, we'd have to define an arbitrary ordering for them. Overall, we'd prefer for people to be able to think of `impl Trait` more intuitively without having to understand the desugaring.

As noted in that section, however, generic parameters that appear in the argument types can be made late-bound instead of early-bound. In the case of `impl Trait` arguments, they always (by definition) appear in the argument types. Therefore, we could make them "late-bound", so that they are only parameters of the impl and not of the type. Conceptually then the fndef type for `foo` would be like this:

```rust
struct foo;
impl<D: Debug> FnOnce<(D,)> for foo {
    ...
}
```

As a result, the type of `foo` is just written as `foo`, with no type arguments, and hence this struct is perfectly legal:

```rust
struct Wrapper {
    f: foo
}
```

What's more, making `impl Trait` late bound is actually more flexible. For example, this code does not compile today, but it would under this proposal:

```rust
fn foo(d: impl Debug) { /* ... */ }

fn main() {
    let f = foo;
    f(22_u32); // call once with `u32`
    f(22_i32); // call again with `i32`
}
```

### Turbofish interaction

This also settles 'en passante' on of the open questions about impl Trait in argument position: should you be allowed to specify their value in turbofish? Clearly, the answer under this proposal is no, as there are no type parameters whose value needs to be specified.

### Implication: migration to `impl Trait` is not fully backwards compatible

Settling the turbofish question in this matter does mean that one cannot migrate from a generic function to `impl Trait` with perfect fidelity:

```rust
fn foo<D: Debug>(d: D) { }
```

is different than

```rust
fn foo(d: impl Debug) { }
```

because the former permits turbofish and the latter does not.

### Implication: Backwards incompatibility around inference

Unfortunately, there is a *slight* (largely theoretical) backwards incompatibility with making `impl Trait` type parameters late bound. It is possible today to leverage inference *across* calls to the same function. The following code compiles today but would become an error in the future:

```rust
fn foo(d: impl Debug) { /* ... */ }

fn main() {
    let f = foo;
    f(None); // call with `Option<_>`, unknown value type
    f(Some(22_i32)); // call again with `Option<i32>`
}
```

Today, we are able to infer that both calls must be using an `Option<i32>`. This works because all calls to `f` must use the same value for the impl trait (it is early bound). If it becomes late bound, that is no longer true, so we would require the `f(None)` call to use an explicit type annotation (e.g., `f(None::<i32>)`). **This form of breakage is permitted by our semver rules. We judge the likelihood of this impacting many crates to be small, but we will have to test it.**

### Implication: Backwards incompatibility around inference

# Drawbacks
[drawbacks]: #drawbacks

## Giving values for uncaptured parameters

Impl trait in return position do not capture all 

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why not introduce the `typeof` keyword?

It has been proposed to use the `typeof` keyword to permit users to take the resulting type from arbitrary expressions. This would mean that one could 

```rust
fn foo<T> { }
```

## Why not introduce a named type 

## Why not introduce a named type into the environment?

It is difficult to decide 

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
