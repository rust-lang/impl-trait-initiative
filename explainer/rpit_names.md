# Naming impl trait in return types

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