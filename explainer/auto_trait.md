# Auto traits and impl trait

![stable][]

{{#include ../badges.md}}

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