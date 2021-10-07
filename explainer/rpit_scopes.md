# Defining scopes

**Status:** Stable.

In cases like this, where the `impl Trait` is mapped to a single type that is determined by the body of a function or some other thing, we say that it is an **output impl Trait**. It occurs when the `impl Trait` appears in **output position**, such as the return type of a function (we'll see some other output positions later on).

Every output impl Trait has a **hidden type** and a **defining scope**. The **hidden type** is the actual type that this `impl Trait` is inferred to represent; in the example of `odd_integers`, this hidden type was `Filter<Range<u32>, _>` (you can see that the hidden type often includes anonymous types, like the types of closures, that don't have a user-writable syntax). We call this the "hidden" type because callers don't know what it's value is, they only know that it is *some type that impements `Iterator`*. 

The **defining scope** of an `impl Trait` is the region of code that is used to determine the hidden type for the `impl Trait`. The hidden type is inferred as part of type-checking the functions and other code blocks within that defining scope. The compiler tries to find some hidden type H such that replacing the `impl Trait` with that hidden type would still make the code type check.

In the case of `odd_integers`, the defining scope for the `-> impl Iterator` is the function `odd_integers` itself. We'll see examples later on where the defining scope of an output `impl Trait` can be larger, such as an impl or even a module, and we can dig into the inference of hidden types more then.

