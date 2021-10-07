# Defining scopes

The [defining scope] for a type alias is the enclosing module and its submodules:

[defining scope]: ./rpit_scopes.md

```rust
#![feature(type_alias_impl_trait)]

mod foo {
    type Foo = impl Clone;

    mod bar {
        // defines the value of `Foo`:
        fn test() -> super::Foo {
            22_i32
        }
    }
}
```

If the type alias appears in a function body, then the defining scope is the function body and any items declared within that function:

```rust
#![feature(type_alias_impl_trait)]

fn foo() {
    type Foo = impl Clone;
    
    // defines the value of `Foo`:
    fn test() -> Foo {
        22_i32
    }

    let x: Foo = test();
    let y = x.clone();
}

fn main() { }
```