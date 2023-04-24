# Understand function lifetime parameters

First, note that elided lifetimes in function signatures are invisible lifetime parameters on the function.
```rust
fn zed(s: &str) {}
```
```rust
// same thing
fn zed<'s>(s: &'s str) {}
```
When you have a lifetime parameter like this, the *caller* chooses the lifetime.
But the body of your function is opaque to the caller: they can only choose lifetimes *just longer* than your function body.

So when you have a lifetime parameter on your function (without any further bounds), the only things you know are
* It's longer than your function body
* You don't get to pick it, and it could be arbitrarily long (even `'static`)
* But it could be *just barely longer* than your function body too; you have to support both cases

And the main corollaries are
* You can't borrow locals for a caller-chosen lifetime
* You can't extend a caller-chosen lifetime to some other named lifetime in scope
  * Unless there's some other outlives bound that makes it possible

---

Here's a couple of error examples related to function lifetime parameters:
```rust
fn long_borrowing_local<'a>(name: &'a str) {
    let local = String::new();
    let borrow: &'a str = &local;
}

fn borrowing_zed(name: &str) -> &str {
    match name.len() {
        0 => "Hello, stranger!",
        _ => &format!("Hello, {name}!"),
    }
}
```
