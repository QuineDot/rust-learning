# Clonable `Box<dyn Trait>`

What can you do if you want a `Box<dyn Trait>` that you can clone?
You can't have `Clone` as a [supertrait,](dyn-trait-combining.md)
because `Clone` requires `Sized` and that will make `Trait` be
[non-object-safe.](dyn-safety.md)

You might be tempted to do this:

```rust
trait Trait {
    fn dyn_clone(&self) -> Self where Self: Sized;
}
```

But then `dyn Trait` won't have the method available, and that will
[be a barrier to implementing `Trait` for `Box<dyn Trait>`.](dyn-trait-box-impl.md)

But hey, you know what?  Since this only really makes sense for
base types that implement `Clone`, we don't need a method that returns
`Self`.  The base types already have that, it's called `clone`.

What we ultimately want is to get a `Box<dyn Trait>` instead, like so:
```rust
trait Trait {
    fn dyn_clone<'s>(&self) -> Box<dyn Trait + 's> where Self: 's;
}

// example implementor
impl Trait for String {
    fn dyn_clone<'s>(&self) -> Box<dyn Trait + 's> where Self: 's {
        Box::new(self.clone())
    }
}
```

If we omit all the lifetime stuff, it only works with `Self: 'static`
due to the [default `'static` lifetime.](dyn-elision-basic.md)  And
sometimes, that's perfectly ok!  But we'll stick with the more general
version for this example.

The example implementation will make `dyn Trait` do the right thing
(clone the underlying base type via its implementation).  We can't have
a default body though, because the implementation requires `Clone`
and `Sized`, which again, we don't want as bounds.

But this is exactly the situation we had when we looked at
[manual supertrait upcasting](./dyn-trait-combining.md#manual-supertrait-upcasting)
and the [`self` receiver helper](./dyn-trait-box-impl.md)
in previous examples.  The same pattern
will work here: move the method to a helper supertrait and supply
a blanket implementation for those cases where it makes sense.

```rust
trait DynClone {
    fn dyn_clone<'s>(&self) -> Box<dyn Trait + 's> where Self: 's;
}

impl<T: Clone + Trait> DynClone for T {
    fn dyn_clone<'s>(&self) -> Box<dyn Trait + 's> where Self: 's {
        Box::new(self.clone())
    }
}

trait Trait: DynClone {}
```

Now we're ready for `Box<dyn Trait + '_>`.
```rust
#trait Trait: DynClone {}
#trait DynClone {
#    fn dyn_clone<'s>(&self) -> Box<dyn Trait + 's> where Self: 's;
#}
#impl<T: Clone + Trait> DynClone for T {
#    fn dyn_clone<'s>(&self) -> Box<dyn Trait + 's> where Self: 's {
#        Box::new(self.clone())
#    }
#}
impl Trait for Box<dyn Trait + '_> {}

impl Clone for Box<dyn Trait + '_> {
    fn clone(&self) -> Self {
    	// Important! "recursive trait implementation" style
        (**self).dyn_clone()
    }
}
```

It's important that we called `<dyn Trait as DynClone>::dyn_clone`!  Our
blanket implementation of `DynClone` was bounded on `Clone + Trait`, but
now we have implemented both of those for `Box<dyn Trait + '_>`.  If we
had just called `self.dyn_clone()`, the call graph would go like so:
```rust,ignore
<Box<dyn Trait> as Clone   >::clone()
<Box<dyn Trait> as DynClone>::dyn_clone()
<Box<dyn Trait> as Clone   >::clone()
<Box<dyn Trait> as DynClone>::dyn_clone()
<Box<dyn Trait> as Clone   >::clone()
<Box<dyn Trait> as DynClone>::dyn_clone()
...
```

Yep, infinite recursion.  Just like when implementing `Trait for Box<dyn Trait>`,
we need to call the `dyn Trait` method directly to avoid this.

---

There is also a crate for this use case: [the `dyn-clone` crate.](https://crates.io/crates/dyn_clone)

A comparison with the crate is beyond the scope of this guide for now.
