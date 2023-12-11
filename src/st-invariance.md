# Nested borrows and invariance

Now let's consider nested references:
* A `&'medium &'long U` coerces to a `&'short &'short U`
* A `&'medium mut &'long mut U` coerces to a `&'short mut &'long mut U`...
    * ...but *not* to a `&'short mut &'short mut U`

We say that `&mut T` is *invariant* in `T`, which means any lifetimes in `T` cannot change
(grow or shrink) at all. In the example, `T` is `&'long mut U`, and the `'long` cannot be changed.

Why not?  Consider this:
```rust
fn bar(v: &mut Vec<&'static str>) {
    let w: &mut Vec<&'_ str> = v; // call the lifetime 'w
    let local = "Gottem".to_string();
    w.push(&*local);
} // `local` drops
```
If `'w` was allowed to be shorter than `'static`, we'd end up with a dangling reference in `*v` after `bar` returns.

You will inevitably end up with a feel for covariance from using references with their flexible outer lifetimes,
but eventually hit a use case where invariance matters and causes some borrow check errors, because it's (necessarily) so much less flexible.
It's just part of the Rust learning experience.

---

Let's look at one more property of nested references you may run into:
* You can get a `&'long U` from a `&'short &'long U`:
  * Just copy it out!
* But you cannot get a `&'long mut U` through dereferencing a `&'short mut &'long mut U`.
  * You can only reborrow a `&'short mut U`.
  * Obtaining a `&'long mut U` is sometimes possible by swapping or
    temporarily moving out of the inner reference, for example with [`mem::swap`](https://doc.rust-lang.org/stable/std/mem/fn.swap.html),
    [`mem::replace`](https://doc.rust-lang.org/stable/std/mem/fn.replace.html),
    or [`replace_with`](https://docs.rs/replace_with/). See the [mutable slice iterator example](./lt-ex-mut-slice.md).

The reason is again to prevent memory unsafety.

Additionally,
* You cannot get a `&'long U` or *any* `&mut U` from a `&'short &'long mut U`
  * You can only reborrow a `&'short U`

Recall that once a shared reference exist, any number of copies of it could
simultaneously exist.  Therefore, so long as the outer shared reference exists
(and could be used to observe `U`), the inner `&mut` must not be usable in a
mutable or otherwise exclusive fashion.

And once the outer reference expires, the inner `&mut` is active and must
again be exclusive, so it must not be possible to obtain a `&'long U` either.
