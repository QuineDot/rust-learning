# Don't hide lifetimes

When you use lifetime-carrying structs (whether your own or someone else's),
the Rust compiler currently let's you elide the lifetime parameter when
mentioning the struct:
```rust
struct Foo<'a>(&'a str);

impl<'a> Foo<'a> {
    // No lifetime syntax needed :-(
    //                 vvv
    fn new(s: &str) -> Foo {
        Foo(s)
    }
}
```
This can make it non-obvious that borrowing is going on, and harder to figure
out where errors are coming from.  To save yourself some headaches, I recommend
using the `#![deny(elided_lifetimes_in_paths)]` lint:
```rust,compile_fail
#![deny(elided_lifetimes_in_paths)]
struct Foo<'a>(&'a str);

impl<'a> Foo<'a> {
    // Now this is an error
    //                 vvv
    fn new(s: &str) -> Foo {
        Foo(s)
    }
}
```
The first thing I do when taking on a borrow check error in someone else's code
is to turn on this lint.  If you have not enabled the lint and are getting errors
in your own code, try enabling the lint.  For every place that errors, take a moment
to pause and consider what is going on with the lifetimes.  Sometimes there's only
one possibility and you will just need to make a trivial change:
```diff
-    fn new(s: &str) -> Foo {
+    fn new(s: &str) -> Foo<'_> {
```
But often, in my experience, one of the error sites will be part of the problem
you're dealing with.



