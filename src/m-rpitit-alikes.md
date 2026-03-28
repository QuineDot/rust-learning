# `async` and returning `impl Trait`

A lot can be said about `async fn` and returning `impl Trait`; more than can be covered here.
But something to be particularly aware of is how they nearly invisibly introduce borrow
relationships in function signatures.

## About `-> impl Trait` and implicit capturing

You can use `impl Trait` as a return type to return something which the caller
knows satisfies the trait, without actually naming the type.  This is also
called "`return` position `impl Trait`", or RPIT.

When you use `-> impl Trait`, it's possible for lifetimes (and type generics)
to flow into the return type almost invisibly:

```rust,ignore,mdbook-runnable
# #![deny(elided_lifetimes_in_paths)]
# use either::Either;
# use std::fs::File;
# use std::io::{self, BufRead, BufReader};
# use std::iter;
fn example(s: &str) -> impl Iterator<Item = Result<String, io::Error>> {
    match File::open(s) {
        Ok(file) => Either::Left(BufReader::new(file).lines()),
        Err(e) => Either::Right(iter::once(Err(e))),
    }
}
```

The signature is actually syntactic sugar for:
```rust,ignore
//                          vvvvvvvvv
fn example(s: &str) -> impl use<'_> + Iterator<Item = Result<String, io::Error>> {
```

This is called "capturing" the generic lifetime.  In this case it means that callers
will treat the returned value as if it contains the `&str` which we passed in.
However, we're not actually doing that!  So this may cause unexpected borrow checker
errors, like so:
```rust,compile_fail
# #![deny(elided_lifetimes_in_paths)]
# use either::Either;
# use std::fs::File;
# use std::io::{self, BufRead, BufReader};
# use std::iter;
# fn example(s: &str) -> impl Iterator<Item = Result<String, io::Error>> {
#     match File::open(s) {
#         Ok(file) => Either::Left(BufReader::new(file).lines()),
#         Err(e) => Either::Right(iter::once(Err(e))),
#     }
# }
fn main() {
    let local = String::from("filename.txt");
    let iter = example(&local);
    let _move_of_local = local; // Errors because `local` is still borrowed...
    for _ in iter {}            // ...as if `&local` ended up in `iter`
}
```
The return value keeps `local` borrowed, as that's what the function API tells the
compiler to enforce.  Capturing the lifetime is part of the API contract.

By default, *all* generics are captured (all lifetimes and all types).  But we can
write out our own `use` clause manually to only capture the lifetimes we actually need.
(So far, you are always required to capture all *type* generics.)

In this example we don't need any lifetimes -- we don't actually use `s` in our
returned value -- so the change looks like this:
```diff
-fn example(s: &str) -> impl Iterator<Item = Result<String, io::Error>> {
+fn example(s: &str) -> impl use<> + Iterator<Item = Result<String, io::Error>> {
```
And now this compiles:
```rust,ignore,mdbook-runnable
# #![deny(elided_lifetimes_in_paths)]
# use either::Either;
# use std::fs::File;
# use std::io::{self, BufRead, BufReader};
# use std::iter;
fn example(s: &str) -> impl use<> + Iterator<Item = Result<String, io::Error>> {
    // ...
#     match File::open(s) {
#         Ok(file) => Either::Left(BufReader::new(file).lines()),
#         Err(e) => Either::Right(iter::once(Err(e))),
#     }
}

fn main() {
    let local = String::from("filename.txt");
    let iter = example(&local);
    let _move_of_local = local;
    for _ in iter {}
}
```

As far as I know there is no lint to require making the `use<..>` clause explicit.
Instead, I recommend trying to get in the habit of seeing `-> impl` as a signal that
a borrow relationship may be happening, similar to how you may see `-> &_`.

<details>
<summary>Expand for some historical notes.</summary>

[RPIT outside of traits do not capture lifetimes by default on older editions.](https://doc.rust-lang.org/nightly/edition-guide/rust-2024/rpit-lifetime-capture.html)
On those editions, someone may use `+ use<..>` to capture *more* lifetimes instead of less!
(But ideally, you should just use the newest stable edition of Rust.)

Like the edition guide states, you may also see `-> impl Trait + 'lifetime` instead of
`+ use<'lifetime>`.  The pattern became somewhat common as it was stable before the
creation of the `use<..>` clause, but `use<..>` is usually the correct choice.

</details>

## `async fn` capturing

`async fn` uses the RPIT capabilities under the hood.  As a result, the return type of an
`async` function captures all of its generic parameters, including any lifetimes.  So here:
```rust
async fn example(v: &mut Vec<String>) -> String {
    "Hi :-)".to_string()
}

// Notionally the same as:
// fn example(v: &mut Vec<String>) -> impl use<'_> + Future<Output = String> {
//     async move {
//         // Always capture everything!
//         let v = v;
//         "Hi :-)".to_string()
//     }
// }
```
The future returned by the `async fn` implicitly reborrows the `v` input, and
"carries" the same lifetime, just like the other examples we saw.

So you should view `async fn` similarly to how you view `-> impl`: a flag
that the return type might be holding onto borrows from the inputs.

If you run into a case where the capturing is unnecessary, you can rewrite the
`async fn` as an `impl Trait` returning normal `fn` instead.
```rust
// Note the empty `use<>`...
fn example(v: &mut Vec<String>) -> impl use<> + Future<Output = String> {
    let _ignore = v;
    // ...and don't move or otherwise use the borrow inside the `async` block
    async {
        "Hi :-)".to_string()
    }
}
```

