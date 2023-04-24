# Copy and reborrows

Shared references (`&T`) implement `Copy`, which makes them very flexible. Once you have one,
you can have as many as you want; once you've exposed one, you can't keep track of how many there are.

Exclusive references (`&mut T`) do not implement `Copy`.
Instead, you can use them ergonomically through a mechanism called *reborrowing*. For example here:

```rust
fn foo<'v>(v: &'v mut Vec<i32>) {
    v.push(0);         // line 1
    println!("{v:?}"); // line 2
}
```

You're not moving `v: &mut Vec<i32>` when you pass it to `push` on line 1, or you couldn't print it on line 2.
But you're not copying it either, because `&mut _` does not implement `Copy`. 
Instead `*v` is reborrowed for some shorter lifetime than `'v`, which ends on line 1.

An explicit reborrow would look like this:
```rust,no_compile
    Vec::push(&mut *v, 0);
```

`v` can't be used while the reborrow `&mut *v` exists, but after it "expires", you can use `v` again.

Though tragically underdocumented, reborrowing is what makes `&mut` usable; there's a lot of implicit reborrowing in Rust.
Reborrowing makes `&mut T` act like the `Copy`-able `&T` in some ways. But the necessity that `&mut T` is exclusive while
it exists leads to it being much less flexible.

Reborrowing is a large topic on its own, but you should at least understand that it exists, and is what enables Rust to
be usable and ergonomic while still enforcing memory safety.
