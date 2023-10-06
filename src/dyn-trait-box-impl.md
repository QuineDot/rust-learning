# `impl Trait for Box<dyn Trait>`

Let's look at how one implements `Trait for Box<dyn Trait + '_>`.  One
thing to note off that bat is that most methods are going to involve
calling a method of the `dyn Trait` *inside* of our box, but if we just
use `self.method()` we would instantly recurse with the very method
we're writing (`<Box<dyn Trait>>::method`)!

***We need to take care to call `<dyn Trait>::method` and not `<Box<dyn Trait>>::method`
in those cases to avoid infinite recursion.***

Now that we've highlighted that consideration, let's dive right in:
```rust
trait Trait {
    fn look(&self);
    fn boop(&mut self);
    fn bye(self) where Self: Sized;
}

impl Trait for Box<dyn Trait + '_> {
    fn look(&self) {
        // We do NOT want to do this!
        // self.look()

        // That would recursively call *this* function!
        // We need to call `<dyn Trait as Trait>::look`.

        // Any of the below forms work, it depends on
        // how explicit you want to be.

        // Very explicit
        // <dyn Trait as Trait>::look(&**self)

        // Yay auto-deref for function parameters
        // <dyn Trait>::look(self)

        // Very succinct and a "makes sense once you've
        // seen it enough times" form.  The first deref
        // is for the reference (`&Self`) and the second
        // deref is for the `Box<_>`.
        (**self).look()
    }

    fn boop(&mut self) {
        // This is similar to the `&self` case
        (**self).boop()
    }

    fn bye(self) {
        // Uh... see below
    }
}
```

Oh yeah, that last one.  [Remember what we said before?](dyn-trait-impls.md#boxdyn-trait-and-dyn-trait-do-not-automatically-implement-trait)
`dyn Trait` doesn't have this method, but `Box<dyn Trait + '_>` does.
The compiler isn't going to just guess what to do here (and couldn't if,
say, we needed a return value).  We can't move the `dyn Trait` out of
the `Box` because it's unsized.  And we can't
[downcast from `dyn Trait`](./dyn-any.md#downcasting-methods-are-not-trait-methods)
either; even if we could, it would rarely help here, as we'd have to both
impose a `'static` constraint and also know every type that implements our
trait to attempt downcasting on each one (or have some other clever scheme
for more efficient downcasting).

Ugh, no wonder `Box<dyn Trait>` doesn't implement `Trait` automatically.

Assuming we want to call `Trait::bye` on the erased type, are we out of luck?
No, there are ways to work around this:
```rust
// Supertrait bound
trait Trait: BoxedBye {
    fn bye(self);
}

trait BoxedBye {
    // Unlike `self: Self`, this does *not* imply `Self: Sized` and
    // thus *will* be available for `dyn BoxedBye + '_`... and for
    // `dyn Trait + '_` too, automatically.
    fn boxed_bye(self: Box<Self>);
}

// We implement it for all `Sized` implementors of `trait: Trait` by
// unboxing and calling `Trait::bye`
impl<T: Trait> BoxedBye for T {
    fn boxed_bye(self: Box<Self>) {
        <Self as Trait>::bye(*self)
    }
}

impl Trait for Box<dyn Trait + '_> {
    fn bye(self) {
        // This time we pass `self` not `*self`
        <dyn Trait as BoxedBye>::boxed_bye(self);
    }
}
```

By adding the supertrait bound, the compiler will supply an implementation of
`BoxedBye for dyn Trait + '_`.  That implementation will call the implementation
of `BoxedBye` for `Box<Erased>`, where `Erased` is the erased base type.  That
is our blanket implementation, which unboxes `Erased` and calls `Erased`'s
`Trait::bye`.

The signature of `<dyn Trait as BoxedBye>::boxed_bye` has a receiver with the
type `Box<dyn Trait + '_>`, which is exactly the same signature as
`<Box<dyn Trait + '_> as Trait>::bye`.  And that's how we were able to
complete the implementation of `Trait` for `Box<dyn Trait + '_>`.

Here's how things flow when calling `Trait::bye` on `Box<dyn Trait + '_>`:
```rust,ignore
<Box<dyn Trait>>::bye      (_: Box<dyn Trait>) -- just passed -->
<    dyn Trait >::boxed_bye(_: Box<dyn Trait>) -- via vtable  -->
<      Erased  >::boxed_bye(_: Box<  Erased >) -- via unbox   -->
<      Erased  >::bye      (_:       Erased  ) :)
```

There's rarely a reason to implement `BoxedBye for Box<dyn Trait + '_>`, since
that takes a nested `Box<Box<dyn Trait + '_>>` receiver.

Any `Sized` implementor of `Trait` will get our blanket implementation of
the `BoxedBye` supertrait "for free", so they don't have to do anything
special.

---

The last thing I'll point out is how we did
```rust
# trait Trait {}
impl Trait for Box<dyn Trait + '_> {
// this:                     ^^^^
# }
```

[We didn't need to require `'static`, so this is more flexible.](dyn-elision-basic.md#impl-headers)
It's also very easy to forget.

