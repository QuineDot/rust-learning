# `dyn Trait` examples

Here we provide some "recipes" for common `dyn Trait` implementation
patterns.

In the examples, we'll typically be working with `dyn Trait`, `Box<dyn Trait>`,
and so on for the sake of brevity.  But note that in more practical code, there
is a good chance you would also need to provide implementations for
`Box<dyn Trait + Send + Sync>` or other variations across auto-traits.  This
may be in place of implementations for `dyn Trait` (if you always need the
auto-trait bounds) or in addition to implementations for `dyn Trait`
(to provide maximum flexibility).
