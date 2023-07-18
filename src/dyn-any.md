# More about `dyn Any`

We've taken a lot of care to emphasize that `dyn Trait` isn't
a supertype nor a form of dynamic typing, but
[in one of the examples](dyn-trait-eq.md#downcasting-with-dyn-any-to-emulate-dynamic-typing)
we saw that `dyn Any` *can* "downcast" back to the erased
base type.  Or [to quote the official documentation,](https://doc.rust-lang.org/std/any/trait.Any.html)
`Any` is
> A trait to emulate dynamic typing.

Here we take a closer look at `dyn Any` specifically.

## The general idea

The `Any` trait is implemented for all types which satisfy a `'static` bound.
It supplies a method `type_id`, which returns
[an opaque but unique identifier](https://doc.rust-lang.org/std/any/struct.TypeId.html)
of the implementing type.  We also have
[`TypeId::of::<T>`,](https://doc.rust-lang.org/std/any/struct.TypeId.html#method.of)
which lets us look up the `TypeId` of any `'static` type.

Together, this allows fallible downcasting by doing [things along these lines:](https://doc.rust-lang.org/src/core/any.rs.html)
```rust
    pub fn downcast_ref<T: Any>(&self) -> Option<&T> {
        if self.is::<T>() {
            // SAFETY: just checked whether we are pointing to the correct type, and we can rely on
            // that check for memory safety because we have implemented Any for all types; no other
            // impls can exist as they would conflict with our impl.
            unsafe { Some(self.downcast_ref_unchecked()) }
        } else {
            None
        }
    }
```
And [`is`](https://doc.rust-lang.org/std/any/trait.Any.html#method.is) simply compares
`TypeId::of::<T>()` to `self.type_id()`.

Details of specific downcasts aside, that's pretty much it!  All that was
needed is the global identifier (`TypeId`).  The standard library provides
the various downcasting methods in order to encapsulate the required
`unsafe`ty.

[As a reminder from before,](dyn-trait-eq.md#downcasting-with-dyn-any-to-emulate-dynamic-typing)
the vtable pointers themselves are not suitable to use as a global identifier
of erased types.  The same trait can have multiple vtables due to codegen
units and linker implementations, and different traits can have the same
vtable due to deduplication optimizations.

[Where exactly the language goes with respect to comparing vtable pointers
is an open question.](https://github.com/rust-lang/rust/issues/106447)  It's
not unimaginable that all vtables will gain some lifetime-erased version of
`TypeId`, but [related to some discussion below,](#why-static) this may not
be as straightforward as it may sound.

## Downcasting methods are not trait methods

Note that the *only* method available in the `Any` trait is `type_id`.
[All of the downcasting methods](https://doc.rust-lang.org/std/any/trait.Any.html#implementations)
are implemented on the erased `dyn Any` directly, or on `Box<dyn Any>`, or
on `dyn Any + Send`, etc.  Downcasting doesn't generally make sense for a
non-erased base type -- you already know what it is!

Another good reason for this is that
[types like `Box<dyn Any>` implement `Any`,](https://doc.rust-lang.org/std/any/index.html#smart-pointers-and-dyn-any)
making easy to accidentally call the `Box<dyn Any>` implementation instead of
the `dyn Any` implementation in the case of `type_id`.  It would be much more
fraught if `downcast_ref` worked like this, for example.

However, this does mean that having `Any` as a supertrait does not allow
downcasting for your own `dyn Trait`s.  [Instead you have to first upcast
to dyn Any,](./dyn-trait-combining.md#manual-supertrait-upcasting) and then
downcast.  Once we have [built-in supertrait upcasting,](dyn-trait-coercions.md#supertrait-upcasting)
the process will involve much less boilerplate when an `Any` suptertrait
bound is acceptable.

## Some brief examples

[In our other example,](dyn-trait-eq.md#downcasting-with-dyn-any-to-emulate-dynamic-typing)
we used [manual supertrait casting](./dyn-trait-combining.md#manual-supertrait-upcasting) to
turn a `dyn DynCompare` into a `dyn Any`.  This was a case where we really just wished we
could attempt to downcast `dyn DynCompare` itself.

Here we instead look at some simple examples of type erasing and downcasting concrete
types directly.

### The basics

Getting a `dyn Any` isn't any different than any other kind of type erasure:
```rust
# use std::any::Any;
let mut i = 0;
let rf: &dyn Any = &();
let mt: &mut dyn Any = &mut i;
let bx: Box<dyn Any> = Box::new(String::new());
```
You have to keep in mind the `'static` requirement though:
```rust,compile_fail
# use std::any::Any;
let local = ();
let borrow = &local;
// fails because `borrow` is not `'static`
let _: &dyn Any = &borrow;
```
On the upside, [`dyn Any` is *always* `dyn Any + 'static`,](dyn-elision-trait-bounds.md#the-static-case)
which makes many trait object related borrow check errors impossible.

Although `Any` is implemented for unsized types, and unsized types can
have `TypeId`s too, the `Sized` restriction for type erasure still applies:
```rust,compile_fail
# use std::any::Any;
// fails because `str` is not `Sized`
let _: &dyn Any = "";
```

Fallible downcasting is pretty straightforward as well.  For references
the return is an `Option`:
```rust
# use std::any::Any;
# let mut i = 0;
# let rf: &dyn Any = &();
# let mt: &mut dyn Any = &mut i;
assert_eq!( rf.downcast_ref::<()>(), Some(&()) );
assert_eq!( rf.downcast_ref::<String>(), None );

assert!( mt.downcast_mut::<i32>().is_some() );
assert_eq!( mt.downcast_mut::<String>(), None );
```

For `Box`es, the return type is a `Result` so that the you can keep
ownership of the `Box<dyn Any>` if the downcast is not applicable.
The `Ok` variant is a `Box<T>` so that you can choose whether it's
appropriate to unbox the type or not.
```rust
# use std::any::Any;
# let bx: Box<dyn Any> = Box::new(String::new());
if let Err(bx) = bx.downcast::<i32>() {
    println!("Hmm, not an `i32`.");
    if let Ok(bx) = bx.downcast::<String>() {
        let s: String = *bx;
        println!("Yep, it was a `String`.");
    }
}
```

That's it for the basics!

### The `TypeMap` pattern

For an example with a more practical bent, let's say you wanted to store a distinct
value for each distinct type you may encounter, for some reason.  Maybe you're
storing callbacks for types which are likewise type erased, say, and the callback
for a `Dog` would be different than that for a `Cat`, and you might not even have
a callback for the `Mouse`.

One way to do this would be to have a data structure that maps a `TypeId` to the
values.  A "type map", if you will:
```rust
# use std::any::Any;
# use std::any::TypeId;
# use std::collections::HashMap;
pub struct TypeMap<V> {
    map: HashMap<TypeId, V>,
}

impl<V> TypeMap<V> {
    pub fn insert<T: Any>(&mut self, value: V) -> Option<V> {
        let id = TypeId::of::<T>();
        self.map.insert(id, value)
    }
    pub fn get_mut<T: Any>(&mut self) -> Option<&mut V> {
        self.get_mut_of(&TypeId::of::<T>())
    }
    pub fn get_mut_of(&mut self, id: &TypeId) -> Option<&mut V> {
        self.map.get_mut(id)
    }
    // ...
}
```

This could be used for the callback idea:
```rust
# use std::any::Any;
# use std::any::TypeId;
# use std::collections::HashMap;
# pub struct TypeMap<V> {
#     map: HashMap<TypeId, V>,
# }
# impl<V> TypeMap<V> {
#     pub fn insert<T: Any>(&mut self, value: V) -> Option<V> {
#         let id = TypeId::of::<T>();
#         self.map.insert(id, value)
#     }
#     pub fn get_mut<T: Any>(&mut self) -> Option<&mut V> {
#         self.get_mut_of(&TypeId::of::<T>())
#     }
#     pub fn get_mut_of(&mut self, id: &TypeId) -> Option<&mut V> {
#         self.map.get_mut(id)
#     }
# }
pub struct Visitor {
    map: TypeMap<Box<dyn FnMut(&dyn Any)>>,
}

impl Visitor {
    // Because we return closures we have previously created,
    // we should take care to not *assume* that the parameter
    // in the callback is of the correct type.  If we never
    // let our closures escape to the outside world, we could
    // safely assume that the parameter was, in fact, `T`.
    //
    // It would be sound to `panic` if the parameter was not
    // `T` even if we let the closures escape, but it would
    // not be sound to use the unstable `downcast_ref_unchecked`
    // so long as we're letting the closure escape.
    pub fn register<T, F>(&mut self, mut callback: F) -> Option<Box<dyn FnMut(&dyn Any)>>
    where
        T: Any,
        F: 'static + FnMut(&T),
    {
        let callback = Box::new(move |any: &dyn Any| {
            if let Some(t) = any.downcast_ref::<T>() {
                callback(t);
            }
        });

        self.map.insert::<T>(callback)
    }

    pub fn get_callback<T: Any>(&mut self) -> Option<impl FnMut(&T) + '_> {
        self.map
            .get_mut::<T>()
            .map(|f| {
                |t: &T| f(t)
            })
    }

    pub fn visit<T: Any>(&mut self, value: &T) -> bool {
        if let Some(mut callback) = self.get_callback::<T>() {
            callback(value);
            true
        } else {
            false
        }
    }

    pub fn visit_erased(&mut self, value: &dyn Any) -> bool {
        if let Some(callback) = self.map.get_mut_of(&value.type_id()) {
            callback(value);
            true
        } else {
            false
        }
    }
}
```
Above we have also type erased our callback signatures, since we needed
a single type for our values.  This is somewhat the data structure version
of [erasing a trait.](./dyn-trait-erased.md)

For whatever reason you might want to map by types, this is
[an existing pattern in the ecosystem.](https://lib.rs/keywords/typemap)

## Why `'static`?

The `Any` trait is implemented for all types which satisfy a `'static` bound,
but no other types; in fact, it has a `'static` bound and thus *cannot* be implemented
for types that do not meet a `'static` bound.  This means that emulating dynamic
typing with `Any` cannot be done for borrowing types (except those that borrow for
`'static`), for example.

Why such a harsh restriction?  In short, lifetimes are erased before runtime, types
with different lifetimes would have to have the same `TypeId` identifier, and thus
downcasting based on the `TypeId` would ignore lifetimes and be *wildly unsound*.
Lifetimes are a part of types and certain relationships must be preserved for
soundness, but as the lifetimes have been erased before runtime, it's not possible
to preserve the relationships dynamically.

Thus there is just no sound way to use `TypeId` or any similar lifetime-unaware
identifier to perform non-`'static` downcasts directly.

[There is more information in this RFC PR,](https://github.com/rust-lang/rfcs/pull/1849)
for the curious.  Note that the PR was accepted but then later removed, and was never
about non-`'static` downcasting; it was about a non-`'static` `type_id` method.  The
idea was to get a "type" identifier that ignored lifetimes.

It was withdrawn in large part because if such a thing existed, [the chances of it
being abused in some wildly unsound way are about 100%.](https://internals.rust-lang.org/t/pre-rfc-non-footgun-non-static-typeid/17079/10)

An alternative (as presented in that thread) is to have some way to dynamically check
if two types (which are perhaps generic) are equal without imposing a `'static` bound.
The check could only be meaningful for types that were "inherently `'static`", that is,
types that do not involve any lifetime parameters at all.  That would be possible
without actually exposing a non-`'static` `TypeId` or otherwise enabling downcasting.

The tradeoff results in pretty unintuitive behavior: `&'static str` cannot be
compared to `&'static str` with this approach, because there *is* a lifetime parameter
involved with `&str`!

Another alternative is to provide some sort of "type lambda" which is itself `'static`,
but can soundly map erased lifetimes back to their proper position.  [A sketch is
provided here,](https://github.com/sagebind/castaway/pull/6#issuecomment-1150952050)
but an in-depth exploration is out of scope for this guide.

## A potential footgun around subtypes (subtitle: why not `const`?)

Let's take a minute to talk about types that *do* have a sub and supertype
relationship in Rust!  Types which are [higher-ranked](./dyn-hr.md) have this
relationship.  For example:
```rust
// More explicitly, this is a `for<'any> fn(&'any str)` function pointer.
// The type is higher-ranked over the lifetime.
let fp: fn(&str) = |_| {};

// This type is a supertype of the higher-ranked type.
let fp: fn(&'static str) = fp;

// This errors because you can't soundly downcast the types.
// let fp: fn(&str) = fp;
```

And as it turns out, it is possible for two Rust types which are more than
superficially syntactically different to be *subtypes of one another.*  Some
parts of the language consider the existence of such a relationship to mean
that the two types are equal.  Let's say they are semantically equal.

Below is an example.  Due to covariance, it's always possible to call
either of the functions from the other, which helps explain why they are
considered subtypes of one another.
```rust
let one: for<'a    > fn(&'a str, &'a str) = |_, _| {};
let two: for<'a, 'b> fn(&'a str, &'b str) = |_, _| {};
let mut fp = one;
fp = two;
let mut fp = two;
fp = one;
```

[However, these two types have different `TypeId`s!](https://github.com/rust-lang/rust/issues/97156)

So different parts of Rust currently disagree about what types are equal or not.

As the issue explains, this is a bit of a footgun if you were expecting consistency.
Additionally, it's a blocker for [a `const type_id` function](https://github.com/rust-lang/rust/issues/77125)
as it is possible to cause UB in safe code with a `const type_id` function so long
as this inconsistency remains.

How the language will evolve around this is unclear.  People want the `const` feature
bad enough that [some version with caveats about false negatives](https://github.com/rust-lang/libs-team/issues/231)
may be pursued.  Personally I feel making the type system consistent would be the
better solution and worth waiting for.

## More considerations around higher-ranked types

Even if the issue discussed above gets resolved and Rust becomes consistent about
what types are equal, [higher-ranked types](./dyn-hr.md) introduce some nuance to
be aware of.  For example, when considering these two types:
```rust
trait Look<'s> {}
type HR = dyn for<'any> Look<'any> + 'static;
type ST = dyn Look<'static> + 'static;
```
`HR` is a *subtype* of `ST`, but not *the same type*.
However, they both satisfy a `'static` bound:
```rust
#trait Look<'s> {}
#type HR = dyn for<'any> Look<'any> + 'static;
#type ST = dyn Look<'static> + 'static;
fn assert_static<T: ?Sized + 'static>() {}

assert_static::<HR>();
assert_static::<ST>();
```
As `'static` types, they have `TypeId`s.  As distinct types, their
`TypeId`s are different, even though one is a subtype of the other.

And this in turn means that you can't stop thinking about sub-and-super types
by simply applying a `'static` bound.  If you need to "disable" sub/super type
coercions in a generic context for soundness, you must make that context invariant
or take other steps to avoid a soundness hole, even if you have a `'static` bound.

[See this issue](https://github.com/rust-lang/rust/issues/85863) for a real-life
example of such a soundness hole, and
[this comment in particular](https://github.com/rust-lang/rust/issues/85863#issuecomment-872536139)
exploring the sub/super type relationships of higher-ranked function pointers.

## The representation of `TypeId`

`TypeId` is intentionally opaque and subject to change.  It was internally represented
by a `u64` for quite some time; as of Rust 1.72
[the representation is a `u128`.](https://github.com/rust-lang/rust/pull/109953)
At some future time it could be [something more exotic.](https://github.com/rust-lang/rust/pull/95845)

Long story short, you're not meant to rely on the exact representation of `TypeId`,
only it's type comparing properties.
