# Appendix A: Inference details

**Status:** Under active development; stable features should be compatible with this description. This description varies slightly from the accepted RFCs and will have to be formalized.

The process to infer the hidden type for an impl Trait is as follows.

* When type-checking functions or code within the *defining scope* of an impl Trait `X`:
    * Accumulate subtyping constraints from type check:
        * For example, `let var: X = 22_i32` would create the subtyping constraint that `i32` is a subtype of `X`.
        * Similarly, given `fn foo() -> X`, `let var: i32 = foo()` wold create the subtyping constraint that `X` be a subtype of `i32`.
    * The type `X` is assumed to implement the traits that appear in its bounds
        * Exception: auto traits. Proving an auto trait `X: Send` is done by "revealing" the hidden type and proving the auto trait against the hidden type.
