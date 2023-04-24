# Quine Zine: Learning Rust

[Introduction](README.md)

---

# Practical suggestions for building intuition around borrow errors

- [Sectional introduction](lifetime-intuition.md)
- [Keep at it and participate in the community](community.md)
- [Prefer ownership over long-lived references](have-no-life.md)
- [Don't hide lifetimes](dont-hide.md)
- [Understand elision and get a feel for when to name lifetimes](elision.md)
- [Get a feel for variance, references, and reborrows](subtypes.md)
  - [The seed of a mental model](st-model.md)
  - [Reference types](st-types.md)
  - [Lifetime bounds](st-bounds.md)
  - [Reference lifetimes](st-references.md)
  - [Copy and reborrows](st-reborrow.md)
  - [Nested borrows and invariance](st-invariance.md)
  - [Invariance elsewhere](st-more-invariance.md)
  - [Non-references](st-non-references.md)
- [Get a feel for borrow-returning methods](methods.md)
  - [When not to name lifetimes](m-naming.md)
  - [Bound-related lifetimes "infect" each other](m-infect.md)
  - [`&mut` inputs don't "downgrade" to `&`](m-no-downgrades.md)
- [Understand function lifetime parameters](fn-parameters.md)
- [Borrow errors within a function](lifetime-analysis.md)
- [Learn some pitfalls and antipatterns](pitfalls.md)
  - [`dyn Trait` lifetimes and `Box<dyn Trait>`](pf-dyn.md)
  - [Conditional return of a borrow](pf-nll3.md)
  - [Borrowing something forever](pf-borrow-forever.md)
  - [`&'a mut self` and `Self` aliasing more generally](pf-self.md)
  - [Avoid self-referential structs](pf-meta.md)
- [Scrutinize compiler advice](compiler.md)
  - [Advice to change function signature when aliases are involved](c-signatures.md)
  - [Advice to add bound which implies lifetime equality](c-equality.md)
  - [Advice to add a static bound](c-static.md)
- [Circle back](circle.md)


