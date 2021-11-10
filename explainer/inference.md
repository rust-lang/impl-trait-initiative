# Appendix A: Inference details

![nightly][]

{{#include ../badges.md}}

## Summary

The process to infer the hidden type for an impl Trait is as follows.

* When type-checking functions or code within the *defining scope* of an impl Trait `X`:
    * Accumulate subtyping constraints from type check:
        * For example, `let var: X = 22_i32` would create the subtyping constraint that `i32` is a subtype of `X`.
        * Similarly, given `fn foo() -> X`, `let var: i32 = foo()` wold create the subtyping constraint that `X` be a subtype of `i32`.
    * The type `X` is assumed to implement the traits that appear in its bounds
        * Exception: auto traits. Proving an auto trait `X: Send` is done by "revealing" the hidden type and proving the auto trait against the hidden type.

## Terminology: Opaque type

An impl trait in [output position] is called an **opaque type**. We write the opaque type as `O<P0...Pn>`, where `P0...Pn` are the generic types on the opaque type. For a [type alias impl trait](./tait.md), the generic types are the generics from the type alias. For a [return position impl trait](./rpit.md), the generics are defined by the [RPIT capture rules](./rpit_capture.md).

## Terminology: Opaque type bounds

Given an opaque type `O<P1...Pn>`, let `B1..Bn` be the *bounds* on that opaque type, appropriately substituted for `P1..Pn`. For example, if the user wrote `type Foo<T> = impl PartialEq<T> + Debug + 'static` and we have the opaque type `Foo<u32>`, then the bounds would be `PartialEq<u32>`, `Debug`, and `'static`.

## Notation: applying bounds to a type to create a where clause

Given a type `T` and an opaque type bound `B1`, we write `T: B1` to signify the where clause that results from using `T` as the "self type" for the bound `B1`. For example, given the type `i32` and the bound `PartialEq<u32>`, the where clause would be `i32: PartialEq<u32>` Given the type `i32` and the bound `'static`, the where clause would be `i32: 'static`.

## Type checking opaque types

The following section describes how to type check functions or other pieces of code that reference opaque types.

### Rules that always apply

The following rules for type-checking opaque types apply to all code, whether or not it is part of the type's defining scope.

#### Reflexive equality for opaque types

Like any type, an opaque type `O<P0...Pn>` can be judged to be equal to itself. This does not require knowing the hidden type.

**Example.** The following program **compiles** because of this rule.

```rust
mod foo {
    type Foo = impl Clone;

    fn make_foo(x: u32) -> Foo {
        x
    }
}

fn main() {
    let mut x: Foo = make_foo(22);

    // Requires knowing that `Foo = Foo`.
    x = make_foo(44);
}
```

**Example.** The following program **does not compile** even though the hidden types are the same.

```rust
mod foo {
    type Foo<T: Clone> = impl Clone;

    fn make_foo<A>(x: u32) -> Foo<A> {
        x
    }
}

fn main() {
    let mut x: Foo<String> = make_foo::<String>(22);

    // `Foo<String>` is not assumed to be equal to `Foo<char>`.
    x = make_foo::<char>(44);
}
```

#### Method dispatch and trait matching

Given a method invocation `o.m(...)` where the method receiver `o` has opaque type `O<P1...Pn>`, methods defined in the opaque type's bounds are considered "inherent methods", similar to a `dyn` receiver or a generic type.

Similarly, the compiler can assume that `O<P1...Pn>: Bi` is true for any of the declared bounds `Bi`.

**Example.** The following program **compiles** because of this rule.

```rust
type Foo = impl Clone;

fn clone_foo(f: Foo) -> Foo {
    // Clone can be called without knowing the hidden type:
    f.clone()
}

fn make_foo() -> Foo {
    // Hidden type constrained to be `u32`
    22_u32
}
```

**Example.** The following program **does not compile** even though the hidden type for `Foo` is known within `clone_foo`; this is because the type of `f` is `impl Clone` and we interpret that as a wish to only rely on the fact that `Foo: Clone` and nothing else (modulo auto trait leakage, see next rule).

```rust
type Foo = impl Clone;

fn clone_foo() {
    let f: Foo = 22_u32;

    // We don't know that `Foo: Debug`.
    debug!("{:?}", f);
}
```

### Rules that only apply outside of the defining scope: auto trait leakage

The following type-checking rules can only be used when type-checking an item I that is **not** within the defining scope for the opaque type. When the type-checker for I wishes to employ one of these rules, it needs to "fetch" the hidden type for `O`. This may create a cycle in the computation graph if determining the hidden type for O requires type-checking the body of I (e.g., to compute the hidden type for another opaque type O1); cyclic cases result in a compilation error.

**Example.** The following program **compiles** because of this rule:

```rust
type Foo = impl Clone;

fn is_send<T: Send>() {}

fn clone_foo() {
    let f: Foo = 22_u32;

    // We can leak the hidden type of `u32` defined within this function.
    is_send::<Foo>();
}
```

**Example.** The following program **does not compile** because of a cycle:

```rust
fn is_send<T: Send>() { }        

mod foo {
    pub type Foo = impl Clone;

    pub fn make_foo() -> Foo {
        // Requires knowing hidden type of `Bar`:
        is_send::<crate::bar::Bar>();
        22_u32
    }
}

mod bar {
    pub type Bar = impl Clone;

    pub fn make_bar() -> Foo {
        // Requires knowing hidden type of `Foo`:
        is_send::<crate::foo::Foo>();
        22_u32
    }
}
```

#### Auto trait leakage

When proving `O<P1...Pn>: AT` for some auto trait `AT`, any item is permitted to use the hidden type `H` for `O<P1...Pn>` and show that `H: AT`.

### Rules that only apply within the defining scope: proposing an opaque type

The following type-checking rules can only be used when type-checking an item `I` that is within the defining scope for the opaque type `O`. If any of these rules are needed to type-check `I`, then `I` must compute a consistent type `H` to use as the hidden type for `O`. Any item `I` that relies on the rules in this section is said to *propose the hidden type `H` for the opaque type `O`*. The process for inferring a hidden type is described below.

#### Equality between an opaque type and its hidden type

An opaque type `O<P0...Pn>` can be judged to be equal to its hidden type `H`.

**Example.** The following program **compiles** because of this rule:

```rust
type Foo = impl Clone;

fn test() {
    // Requires that `Foo` be equal to the hidden type, `u32`.
    let x: Foo = 22_u32;
}
```

#### Auto trait leakage

When proving `O<P1...Pn>: AT` for some auto trait `AT`, `I` can use the hidden type `H` to show that `H: AT`.

**Example.** The following program **compiles** because of this rule:

```rust
type Foo = impl Clone;

fn is_send<T: Send>() {}

fn test() {
    let x: Foo = 22_u32;
    is_send::<Foo>();
}
```

**Example.** The following program **does not compile**. This is because it requires this rule to compile, but does not actually constrain the hidden type in any other way.

```rust
type Foo = impl Clone;

fn is_send<T: Send>() {}

fn test() {
    is_send::<Foo>();
}
```

## Determining the hidden type

To determine the hidden type for some opaque type `O`, we examine the set `C` of items that propose a hidden type `O`. If that set is an empty set, there is a compilation error. If two items in the set propose distinct hidden types for `O`, that is also an error. Otherwise, if all items propose the same hidden type `H` for `O`, then `H` becomes the hidden type for `O`.

### Computing the hidden type proposed by an item `I`

Computing the hidden type `H` for `O<P1..Pn>` that is required by the item `I` must be done independently from any other items within the defining scope of `O`. It can be done by creating an inference variable `?V` for `O<P1..Pn>` and unifying `?O` with all types that must be equal to `O<P1...Pn>`. This inference variable `?V` is called the *exemplar*, as it is an "example" of what the hidden type is when `P1..Pn` are substituted for the generic arguments of `O`. Note that a given function may produce multiple exemplars for a single opaque type if it contains multiple references to `O` with distinct generic arguments. Computing the actual hidden type is done by [higher-order pattern unification](#higher-order-pattern-unification). 

### Checking the hidden type proposed by an item `I`

Once the hidden type for an item `I` is determined, we also have to check that the hidden type is well-formed in the context of the type alias and that its bounds are satisfied. Since the where clauses that were in scope when the exemplar was computed can be different from those declared on the opaque type, this is not a given.

**Example.** The following program **does not compile** because of this check. Here, the exemplar is `T` and the hidden type is `X`. However, the type alias does not declare that `X: Clone`, so the `impl Clone` bounds are not known to be satisfied.

```rust
type Foo<X> = impl Clone;

fn make_foo<T: Clone>(t: T) {
    t
}
```

### Limitations on exemplars

Whenever an item `I` proposes a hidden type, the following conditions must be met:

* All of the exemplars from a given item `I` must map to the same hidden type `H`.
* Each exemplar type `E` must be completely constrained and must not involve unbound inference variables.

#### Examples

The following function has two exemplars but they both map to the same hidden type, so it is legal:

```rust
type Foo<T, U> = impl Debug;

fn compatible<A, B>(a: A, b: B) {
    // Exemplar: (A, B) for Foo<A, B>
    // Hidden type: (T, U)
    let _: Foo<A, B> = (a, b);

    // Exemplar: (B, A) for Foo<B, A>
    // Hidden type: (T, U)
    let _: Foo<B, A> = (b, a);
}
```

The following program is illegal because of the restriction against having incompatible examplars.

```rust
type Foo<T, U> = impl Debug;

fn incompatible<A, B>(a: A, b: B) {
    // Exemplar: (A, B)
    // Hidden type: (T, U)
    let _: Foo<A, B> = (a, b);

    // Exemplar: (B, A)
    // Hidden type: (U, T)
    let _: Foo<A, B> = (b, a);
}
```

The following program is illegal because of the restriction against having incomplete types. This is true even though the two functions, taken in aggregate, have enough information to compute the hidden type for `Foo`.

```rust
type Foo = impl Debug;

fn incomplete1() {
    // Computes an incomplete exemplar of `Result<(), _>`.
    let x: Foo = Ok(());
}

fn incomplete2() {
    // Computes an incomplete exemplar of `Result<_, ()>`.
    let x: Foo = Err(());
}
```

### Higher-order pattern unification

When an item `I` finishes its type check, it will have computed an exemplar type `E` for the opaque type `O<P1..Pn>`. This must be mapped back to the hidden type `H`. Thanks to the [limitations on type alias generic parameters](./tait_generics.md), this can be done by a process called *higher-order pattern unification* (a limited, tractable form of *higher-order unification*). Higher-order pattern unification is best explained by example.

Consider an opaque type `Foo`:

```rust
type Foo<T: Clone, U: Clone> = impl Clone;
```

and an item `make_foo` that contains `Foo`:

```rust
fn make_foo<X: Clone, Y: Clone>(x: X, y: Y) -> Foo<X, Y> {
    vec![(x, y)]
}
```

Here the opaque type `O<P1, P2>` is `Foo<X, Y>` (i.e., `P1` is `X` and `P2` is `Y`) and the exemplar type `E` is `Vec<(X, Y)>`. We wish to compute the *hidden* type, which will be `Vec<(T, U)>` -- note that the hidden type references the generics from the declaration of `Foo`, whereas the exemplar references the generics from `make_foo`. Computing the hidden type can be done by finding each occurence of a generic argument `Pi` (in this case, `X` and `Y`) and mapping to the corresponding generic parameter `i` from the opaque type declaration (in this case, `T` and `U` respectively).

This process is well-defined because of the [limitations on type alias generic parameters](./tait_generics.md). Thanks to those limitations, we know that:

* each generic argument `Pi` will be some generic type on `make_foo`;
* each generic argument `Pi` to the opaque type will be distinct from each other argument `Pj`.

In particular, these limitations mean that whenever we see an instance of some parameter `Pi` in the exemplar, the only way that `Pi` could appear in the final hidden type is if it was introduced by substitution for one of the generic arguments.

To see the ambiguity that is introduced without these limitations, consider what would mappen if you had a function like `bad_make_foo1`:

```rust
fn bad_make_foo1() -> Foo<u32, i32> {
    vec![(22_u32, 22_i32)]
}
```

Here the exemplar type is `Vec<(u32, i32)>`, but the hidden type is ambiguous. It could be `Vec<(T, U)>`, as before, but it could also just be `Vec<(u32, i32)>`. The problem is that this example violates the first restriction: the parameters `u32` and `i32` are not generics on `bad_make_foo1`. This means that they are types which can be named from both the definition of `type Foo` *and* from within the scope of `bad_make_foo1`, introducing ambiguity. 

A similar problem happens when type parameters are repeated, as illustrated by `bad_make_foo2`:

```rust
fn bad_make_foo2<X: Clone>(x: X) -> Foo<X, X> {
    vec![(x.clone(), x.clone())]
}
```

The exemplar type here is `Vec<(X, X)>`. The hidden type, however, could either be `Vec<(T, T)>` or `Vec<(T, U)>` or `Vec<(U, U)>`. All of them would be the same after substitution.


[output position]: ../glossary/output_impl_trait.md
[defining scope]: ../glossary/defining_scope.md
