# `async` and returning `impl Trait`

The return type of an `async` function captures all of its generic parameters,
including any lifetimes.  So here:
```rust
async fn example(v: &mut Vec<String>) -> String {
    "Hi :-)".to_string()
}
```
The future returned by the `async fn` implicitly reborrows the `v` input, and
"carries" the same lifetime, just like the other examples we saw.

The same is true when you use `return`-position `impl Trait` (RPIT) in traits (RPITIT):
```rust
# struct MyStruct {}
impl MyStruct {
    fn iter(&self) -> impl Iterator<Item = String> {
        ["Hi :-)"].into_iter().map(ToString::to_string)
    }
}
```

`self` will remained borrowed here for as long as the iterator is alive.  Note that
this is true even if it's not required by the body!  The implicit lifetime capture
is considered part of the API contract.

For both of these cases, all generic *types* are also captured.

---

Not that RPIT *outside* of traits does *not* implicitly capture lifetimes!  At least,
not as of this writing -- the plan is that RPIT outside of traits will act like RPITIT
and implicitly capture all lifetimes in edition 2024 and beyond.  But for now, they
only implicitly capture type parameters.

```rust
# use std::fmt::Display;
// Required: edition 2021 or before
fn no_capture(s: &str) -> impl Display {
    s.to_string()
}

// This wouldn't compile if `no_capture` reborrowed `*s`
fn check() {
    let mut s = "before".to_string();
    let d = no_capture(&s);
    s ="after".to_string();
    println!("{d}");
}
```

```rust
# use std::fmt::Display;
// This fails on edition 2021 or before, because it tries to
// return a reborrow of `*s`, but that requires capturing the lifetime
fn no_capture(s: &str) -> impl Display {
    s
}   
```

```rust
# use std::fmt::Display;
// This allows it to work again (but `+ '_` is too restrictive
// for every situation, which is part of why edition 2021 will
// change the behavior of RPIT outside of traits)
//
//                                     vvvv
fn no_capture(s: &str) -> impl Display + '_ {
    s            
}
```

---

A lot can be said about `async` and RPITs; more than can be covered here.  But their
implicit capturing nature is something to be aware of, given how invisible it is.

