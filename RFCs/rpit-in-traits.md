# Return position impl Trait in traits

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

* Permit `impl Trait` in fn return position within traits and trait impls.
* This desugars to an anonymous associated type.

# Motivation
[motivation]: #motivation

The `impl Trait` syntax is currently accepted in a variety of places within the Rust language to mean "some type that implements `Trait`" (for an overview, see the [explainer] from the impl trait initiative). For function arguments, `impl Trait` is [equivalent to a generic parameter][apit] and it is accepted in all kinds of functions (free functions, inherent impls, traits, and trait impls). In return position, `impl Trait` [corresponds to an opaque type whose value is inferred][rpit]. In that role, it is currently accepted only in free functions and inherent impls. This RFC extends the support to cover traits and trait impls, just like argument position.

[explainer]: https://rust-lang.github.io/impl-trait-initiative/explainer.html
[apit]: https://rust-lang.github.io/impl-trait-initiative/explainer/apit.html
[rpit]: https://rust-lang.github.io/impl-trait-initiative/explainer/rpit.html

## Example use case

The use case for `-> impl Trait` in trait fns is similar to its use in other contexts: traits often wish to return "some type" without specifying the exact type. As a simple example that we will use through the RFC, consider the `NewIntoIterator` trait, which is a variant of the existing `IntoIterator` that uses `impl Iterator` as the return type:

```rust
trait NewIntoIterator {
    type Item;
    fn into_iter(self) -> impl Iterator<Item = Self::Item>;
}
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Please refer to the ["return position" section of the explainer][rpit].

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Equivalent desugaring for traits

Each `-> impl Trait` notation appearing in a trait fn return type is desugared to an anonymous associated type; the name of this type is a fresh name that cannot be typed by Rust programmers. In this RFC, we will use the name `$` when illustrating desugarings and the like.

As a simple example, consider the following (more complex examples follow):

```rust
trait NewIntoIterator {
    type Item;
    fn into_iter(self) -> impl Iterator<Item = Self::Item>;
}

// becomes

trait NewIntoIterator {
    type Item;

    type $: Iterator<Item = Self::Item>;

    fn into_iter(self) -> <Self as NewIntoIterator>::$;
}
```

## Equivalent desugaring for trait impls

Each `-> impl Trait` notation appearing in a trait impl fn return type is desugared to the same anonymous associated type `$` defined in the trait along with a function that returns it. The value of this associated type `$` is an `impl Trait`. 

```rust
impl NewIntoIterator for Vec<u32> {
    type Item = u32;

    fn into_iter(self) -> impl Iterator<Item = Self::Item> {
        self.into_iter()
    }
}

// becomes

impl NewIntoIterator for Vec<u32> {
    type Item = u32;
    
    type $ = impl Iterator<Item = Self::Item>;

    fn into_iter(self) -> <Self as NewIntoIterator>::$ {
        self.into_iter()
    }
}
```

## Impl trait must be used in both trait and trait impls

Using `-> impl Trait` notation in a trait requires that all trait impls also use `-> impl Trait` notation in their retrn types. Similarly, using `-> impl Trait` notation in an impl is only legal if the trait also uses that notation:

```rust
trait NewIntoIterator {
    type Item;
    fn into_iter(self) -> impl Iterator<Item = Self::Item>;
}

// OK:
impl NewIntoIterator for Vec<u32> {
    type Item = u32;
    fn into_iter(self) -> impl Iterator<Item = u32> {
        self.into_iter()
    }
}

// Not OK:
impl NewIntoIterator for Vec<u32> {
    type Item = u32;
    fn into_iter(self) -> vec::IntoIter<u32> {
        self.into_iter()
    }
}
```

**Rationale:** Maximizing forwards compatibility. We may wish at some point to permit impls and traits to diverge but there is no reason to do it at this time.

## Generic parameter capture and GATs

As with `-> impl Trait` in other kinds of functions, the hidden type for `-> impl Trait` in a trait may reference any of the type or const parameters declared on the impl or the method; it may also reference any lifetime parameters that explicitly appear in the trait bounds ([details](https://rust-lang.github.io/impl-trait-initiative/explainer/rpit_capture.html)). We say that a generic parameter is *captured* if it may appear in the hidden type.

When desugaring, captured parameters from the method are reflected as generic parameters on the `$` associated type. Furthermore, the `$` associated type has the required brings whatever where clauses are declared on the method into scope (excepting those which reference other parameters that are not captured). This transformation is precisely the same as the one which is applied to other forms of `-> impl Trait`, except that it applies to an associated type and not a top-level type alias.

Example:

```rust
trait RefIterator for Vec<u32> {
    type Item<'me>
    where 
        Self: 'me;

    fn iter<'a>(&'a self) -> impl Iterator<Item = Self:Item<'a>>;
}

// Since 'a is named in the bounds, it is captured.
// `RefIterator` thus becomes:

trait RefIterator for Vec<u32> {
    type Item<'me>
    where 
        Self: 'me;

    type $<'a>: impl Iterator<Item = Self::Item<'a>>
    where 
        Self: 'a; // Implied bound from fn

    fn iter<'a>(&'a self) -> Self::$<'a>;
}
```

## Dyn safety

To start, traits that use `-> impl Trait` will not be considered dyn safe, *even if the method has a `where Self: Sized` bound*. This is because dyn types currently require that all associated types are named, and the `$` type cannot be named. The other reason is that the value of `impl Trait` is often a type that is unique to a specific impl, so even if the `$` type *could* be named, specifying its value would defeat the purpose of the `dyn` type, since it would effectively identify the dynamic type.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Can traits migrate from a named associated type to `impl Trait`?

Not compatibly, no, because they would no longer have a named associated type.

## Can traits migrate from `impl Trait` to a named associated type?

Generally yes, but all impls would have to be rewritten.

## Would there be any way to make it possible to migrate from `impl Trait` to a named associated type compatibly?

Potentially! There have been proposals to allow the values of associated types that appear in function return types to be inferred from the function declaration. So the trait has `fn method(&self) -> Self::Iter` and the impl has `fn method(&self) -> impl Iterator`, then the impl would also be inferred to have `type Iter = impl Iterator` (and the return type rewritten to reference it). This may be a good idea, but it is not proposed as part of this RFC.

## What about using a named associated type?

One alternative under consideration was to use a named associated type instead of the anonymous `$` type. The name could be derived by converting "snake case" methods to "camel case", for example. This has the advantage that users of the trait can refer to the return type by name.

We decided against this proposal:

* Introducing a name by converting to camel-case feels surprising and inelegant.
* Return position impl Trait in other kinds of functions doesn't introduce any sort of name for the return type, so it is not analogous.

There is a need to introduce a mechanism for naming the return type for functions that use `-> impl Trait`; we plan to introduce a second RFC addressing this need uniformly across all kinds of functions.

As a backwards compatibility note, named associated types could likely be introduced later, although there is always the possibility of users having introduced associated types with the same name.

# Prior art
[prior-art]: #prior-art

There are a number of crates that do desugaring like this manually or with procedural macros. One notable example is [real-async-trait](https://crates.io/crates/real-async-trait).

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- None.

# Future possibilities
[future-possibilities]: #future-possibilities

We expect to introduce a mechanism for naming the result of `-> impl Trait` return types in a follow-up RFC.

Similarly, we expect to be introducing language extensions to address the inability to use `-> impl Trait` types with dynamic dispatch. These mechanisms are needed for async fn.
