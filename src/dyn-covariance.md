# Variance

The `dyn Trait` lifetime is covariant, like the outer lifetime of a
reference.  This means that whenever it is in a covariant type position,
longer lifetimes can be coerced into shorter lifetimes.
```rust
# trait Trait {}
fn why_be_static<'a>(bx: Box<dyn Trait + 'static>) -> Box<dyn Trait + 'a> {
    bx
}
```
The trait object with the longer lifetime is a subtype of the trait object with
the shorter lifetime, so this is a form of supertype coercion.  [In the next
section,](./dyn-hr.md) we'll look at another form of trait object subtyping.

The idea behind *why* trait object lifetimes are covariant is that the lifetime
represents the region where it is still valid to call methods on the trait object.
Since it's valid to call methods anywhere in that region, it's also valid to restrict
the region to some subset of itself -- i.e. to coerce the lifetime to be shorter.

However, it turns out that the `dyn Trait` lifetime is even more flexible than
your typical covariant lifetime.

## Unsizing coercions in invariant context

[Earlier we noted that](./dyn-trait-coercions.md#the-reflexive-case)
you can cast a `dyn Trait + 'a` to a `dyn Trait + 'b`, where `'a: 'b`.
Well, isn't that just covariance?  Not quite -- when we noted this before,
we were talking about an *unsizing coercion* between two `dyn Trait + '_`.

And *that coercion can take place even in invariant position.*  That means
that the `dyn Trait` lifetime can act in a covariant-like fashion *even in
invariant contexts!*

For example, this compiles, even though the `dyn Trait` is behind a `&mut`:
```rust
# trait Trait {}
fn invariant_coercion<'m, 'long: 'short, 'short>(
    arg: &'m mut (dyn Trait + 'long)
) ->
    &'m mut (dyn Trait + 'short)
{
    arg
}
```

But as there are [no nested unsizing coercions,](./dyn-trait-coercions.md#no-nested-coercions)
this version does not compile:
```rust,compile_fail
# trait Trait {}
fn foo<'l: 's, 's>(v: *mut Box<dyn Trait + 'l>) -> *mut Box<dyn Trait + 's> {
    v
}
```

Because this is an unsizing coercion and not a subtyping coercion, there
may be situations where you must make the coercion explicitly, for example
with a cast.
```rust
# trait Trait {}
// This fails without the `as _` cast.
fn foo<'a>(arg: &'a mut Box<dyn Trait + 'static>) -> Option<&'a mut (dyn Trait + 'a)> {
    true.then(move || arg.as_mut() as _)
}
```

### Why this is actually a critical feature

We'll examine elided lifetime in depth [soon,](./dyn-elision.md) but let us
note here how this "ultra-covariance" is very important for making common patterns
usably ergonomic.

The signatures of `foo` and `bar` are effectively the same in the following example:
```rust
# trait Trait {}
fn foo(d: &mut dyn Trait) {}

fn bar<'a>(d: &'a mut (dyn Trait + 'a)) {
    foo(d);
    foo(d);
}
```

We can call `foo` multiple times from `bar` by [reborrowing](./st-reborrow.md)
the `&'a mut dyn Trait` for shorter than `'a`.  But because the trait object
lifetime must match the outer `&mut` lifetime in this case, *we also have
to coerce `dyn Trait + 'a` to that shorter lifetime.*

Similar considerations come into play when going between a `&mut Box<dyn Trait>`
and a `&mut dyn Trait`:
```rust
#trait Trait {}
#fn foo(d: &mut dyn Trait) {}
#fn bar<'a>(d: &'a mut (dyn Trait + 'a)) {
#    foo(d);
#    foo(d);
#}
fn baz(bx: &mut Box<dyn Trait /* + 'static */>) {
    // If the trait object lifetime could not "shrink" inside the `&mut`,
    // we could not make these calls at all
    foo(&mut **bx);
    bar(&mut **bx);
}
```
Here we reborrow `**bx` as `&'a mut (dyn Trait + 'static)` for some
short-lived `'a`, and then coerce that to a `&'a mut (dyn Trait + 'a)`.

## Variance in nested context

The supertype coercion of going from `dyn Trait + 'a` to `dyn Trait + 'b`
when `'a: 'b` *can* happen in deeply nested contexts, provided the trait
object is still in a covariant context.  So unlike the `*mut` version
above, this version compiles:
```rust
# trait Trait {}
fn foo<'l: 's, 's>(v: *const Box<dyn Trait + 'l>) -> *const Box<dyn Trait + 's> {
    v
}
```
