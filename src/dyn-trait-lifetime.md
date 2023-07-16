# `dyn Trait` lifetimes

As mentioned before, every `dyn Trait` has a "trait object lifetime".  Even though
it is often elided, the lifetime is always present.

The lifetime is necessary as types which implement `Trait` may not be valid everywhere.
For example, `&'s String` implements `Display` for any lifetime `'s`.  If you type
erase a `&'s String` into a `dyn Display`, Rust needs to keep track of that lifetime
so you don't try to print the value after the reference becomes invalid.

So you can coerce `&'s String` to `dyn Display + 's`, but not `dyn Display + 'static`.

Let's look at a couple examples:
```rust,compile_fail
# use core::fmt::Display;
fn fails() -> Box<dyn Display + 'static> {
    let local = String::new();
    // This reference cannot be longer than the function body
    let borrow = &local;
    // We can coerce it to `dyn Display`...
    let bx: Box<dyn Display + '_> = Box::new(borrow);
    // But the lifetime cannot be `'static`, so this is an error
    bx
}
```
```rust
# use core::fmt::Display;
// This is fine as per the function lifetime elision rules, the lifetime of the
// `dyn Display + '_` is the same as the lifetime of the `&String`, and we know
// the reference is valid for that long or it wouldn't be possible to call the
// function.
fn works(s: &String) -> Box<dyn Display + '_> {
    Box::new(s)
}
```

## When multiple lifetimes are involved

Let's try another example, with a `struct` that has more complicated lifetimes.
```rust
trait Trait {}

// We're using `*mut` to make the lifetimes invariant
struct MultiRef<'a, 'b>(*mut &'a str, *mut &'b str);

impl Trait for MultiRef<'_, '_> {}

fn foo<'a, 'b>(mr: MultiRef<'a, 'b>) {
    let _: Box<dyn Trait + '_> = Box::new(mr);
}
```

This compiles, but there's nothing preventing either `'a` from being longer than `'b`,
or `'b` from being longer than `'a`.  So what's the lifetime of the `dyn Trait`?  It
can't be either `'a` or `'b`:
```rust,compile_fail
# trait Trait {}
# #[derive(Copy, Clone)] struct MultiRef<'a, 'b>(*mut &'a str, *mut &'b str);
# impl Trait for MultiRef<'_, '_> {}
// These both fail
fn foo<'a, 'b>(mr: MultiRef<'a, 'b>) {
    let _: Box<dyn Trait + 'a> = Box::new(mr);
    let _: Box<dyn Trait + 'b> = Box::new(mr);
}
```

In this case, the compiler computes some lifetime, let's call it `'c`,
such that `'a` and `'b` are both valid for the entirety of `'c`.

That is, `'c` is contained in an intersection of `'a` and `'b`.

Any lifetime for which both `'a` and `'b` are valid over will do:
```rust
# trait Trait {}
# struct MultiRef<'a, 'b>(*mut &'a str, *mut &'b str);
# impl Trait for MultiRef<'_, '_> {}
// `'c` must be within the intersection of `'a` and `'b`
fn foo<'a: 'c, 'b: 'c, 'c>(mr: MultiRef<'a, 'b>) {
    let _: Box<dyn Trait + 'c> = Box::new(mr);
}
```

Note that this is not the same as `'a + 'b` -- that is the *union*
of `'a` and `'b`.  Unfortunately, there is no compact syntax
for the intersection of `'a` and `'b`.
