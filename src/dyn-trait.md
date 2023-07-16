# Sectional introduction

Rust's type-erasing `dyn Trait` offers a way to treat different implementors
of a trait in a homogenous fashion while remaining strictly and statically
(i.e. compile-time) typed.  For example: if you want a `Vec` of values which
implement your trait, but they might not all be the same base type, you need
type erasure so that you can create a `Vec<Box<dyn Trait>>` or similar.

`dyn Trait` is also useful in some situations where generics are undesirable,
or to type erase unnameable types such as closures into something you need to
name (such as a field type, an associated type, or a trait method return type).

There is a lot to know about when and how `dyn Trait` works or does not, and
how this ties together with generics, lifetimes, and Rust's type system more
generally.  It is therefore not uncommon to get somewhat confused about
`dyn Trait` when learning Rust.

In this section we take a look at what `dyn Trait` is and is not, the limitations
around using it, how it relates to generics and opaque types, and more.
