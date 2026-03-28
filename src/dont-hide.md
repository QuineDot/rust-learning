# Don't hide lifetimes

When you use lifetime-carrying structs (whether your own or someone else's),
the Rust compiler sometimes lets you elide the lifetime parameter when
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
out where errors are coming from.  The above code used to be silently accepted,
but there are now a `mismatched_lifetime_syntaxes` lint which fires for the
most common troublesome patterns.  To save yourself some headaches, you may
even want to make the warning a hard error:

```rust,compile_fail
#![deny(mismatched_lifetime_syntaxes)]
struct Foo<'a>(&'a str);

impl<'a> Foo<'a> {
    // Now this is an error
    //                 vvv
    fn new(s: &str) -> Foo {
        Foo(s)
    }
}
```

There is also an allow-by-default lint called `elided_lifetimes_in_paths` which
fires on code patterns considered less likely to be problematic.

The first thing I do when taking on a borrow check error in someone else's code
is to check these lints.  If you have not enabled the lint and are getting errors
in your own code, try enabling the lint.  For every place that errors, take a moment
to pause and consider what is going on with the lifetimes.  Sometimes there's only
one possibility and you will just need to make a trivial change:
```diff
-    fn new(s: &str) -> Foo {
+    fn new(s: &str) -> Foo<'_> {
```
But often, in my experience, one of the error sites will be part of the problem
you're dealing with.  (This may become less common in the future now that
`mismatched_lifetime_syntaxes` is a warning by default.)

## Be aware of `impl Trait` and `async fn` capturing

[See this section](m-rpitit-alikes.md) which goes into the details.

TL;DR: When you see `-> impl` or `async fn`:
```rust,ignore
fn example(s: &str) -> impl Iterator<Item = Result<String, io::Error>> { ... }

async fn example(v: &mut Vec<String>) -> String { ... }
```
The return types may contain a lifetime from the inputs, even though there
was no `'_` or `&` to indicate it.

So think of `-> impl` and `async fn` as a sign that there may be a borrowing
relationship present, similarly to `-> Foo<'_>`.

