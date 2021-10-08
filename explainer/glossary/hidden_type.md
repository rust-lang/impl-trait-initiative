# Hidden type

The **hidden type** of an impl Trait is the underlying type that the `impl Trait` represents. The compiler always infers the hidden type for an impl trait from the rest of the code. In some cases, there is just a single hidden type, but in other cases -- notably [argument position impl trait](../apit.md) -- there can be multiple hidden types (e.g., one per call site).