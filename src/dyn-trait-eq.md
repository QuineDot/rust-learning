# Downcasting `Self` parameters

Now let's move on to something a little more complicated.  We
[mentioned before](./dyn-safety.md#use-of-self-limitations) that
`Self` is not accepted outside of the receiver, such as when it's
another parameter, as there is no guarantee that the other
parameter has the same base type as the receiver (and if they
are not the same base type, there is no actual implementation to
call).

Let's see how we can work around this to implement
[`PartialOrd`](https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html)
for `dyn Trait`, despite the `&Self` parameter.  The trait is a good
fit in the face of type erasure, as we can just return `None` when
the types don't match, indicating that comparison is not possible.

`PartialOrd` requires `PartialEq`, so we'll tackle that as well.

## Downcasting with `dyn Any` to emulate dynamic typing

We haven't had to use `dyn Any` in the previous examples, because
we've been able to maneuver our implementations in such a way that
dynamic dispatch implicitly "downcasted" our erased types to their
concrete base types for us.  It's able to do this because the pointer
to the base type is coupled with a vtable that only accepts said base
type, and there is no need for actual dynamic typing or comparing types
at runtime.  The conversion is infallible for those cases.

However, now we have two wide pointers which may point to different
base types.  In this particular application, we only really need to
know if they have the same base type or not... though it would be
nice to have some *safe* way to recover the erased type of non-receiver
too, instead of whatever casting shenanigans might be necessary.

You might think you could somehow use the vtable pointers to see if
the base types are the same.  But unfortunately, [we can't rely on the
vtable to compare their types at runtime.](https://doc.rust-lang.org/std/ptr/fn.eq.html)

> When comparing wide pointers, both the address and the metadata are
tested for equality. However, note that comparing trait object pointers
(`*const dyn Trait`) is unreliable: pointers to values of the same
underlying type can compare unequal (because vtables are duplicated in
multiple codegen units), and pointers to values of different underlying
type can compare equal (since identical vtables can be deduplicated
within a codegen unit).

That's right, false negatives *and* false positives.  Fun!

So we need a different mechanism to compare types and know when we
have two wide pointers to the same base type, and that's where `dyn Any`
comes in.  [`Any`](https://doc.rust-lang.org/std/any/trait.Any.html) is
the trait to emulate dynamic typing, and
[many fallible downcasting methods](https://doc.rust-lang.org/std/any/trait.Any.html#implementations)
are supplied for the type-erased forms of `dyn Any`, `Box<dyn Any + Send>`,
et cetera.  This will allow us to not just compare for base type equality,
but also to safely recover the erased base type ("downcast").

The `Any` trait comes with a `'static` constraint for soundness reasons,
so note that our base types are going to be more limited for this example.

Additionally, the [lack of supertrait upcasting](dyn-trait-coercions.md#supertrait-upcasting)
is going to make things less ergonomic than they will be once that feature is
available.

One last side note, [we look at `dyn Any` in a bit more detail later.](./dyn-any.md)

Well enough meta, let's dive in!

## `PartialEq`

The general idea is that we're going to have a comparison trait, `DynCompare`,
and then implement `PartialEq` for `dyn DynCompare` in a universal manner.
Then our actual trait (`Trait`) can have `DynCompare` as a supertrait, and
implement `PartialEq` for `dyn Trait` by upcasting to `dyn DynCompare`.

In the implementation for `dyn DynCompare`, we're going to have to (attempt to)
downcast to the erased base type.  For that to be available we will need to
first be able to upcast from `dyn DynCompare` to `dyn Any`.

As the first step, we're going to use the "supertrait we can blanket implement"
pattern yet again to make a trait that can handle all of our supertrait upcasting needs.

Here it is, [similar to how we've done it before:](dyn-trait-combining.md#manual-supertrait-upcasting)
```rust
use std::any::Any;

trait AsDynCompare: Any {
    fn as_any(&self) -> &dyn Any;
    fn as_dyn_compare(&self) -> &dyn DynCompare;
}

// Sized types only
impl<T: Any + DynCompare> AsDynCompare for T {
    fn as_any(&self) -> &dyn Any {
        self
    }
    fn as_dyn_compare(&self) -> &dyn DynCompare {
        self
    }
}

trait DynCompare: AsDynCompare {
    fn dyn_eq(&self, other: &dyn DynCompare) -> bool;
}
```
There's an `Any: 'static` bound which applies to `dyn Any + '_`, so
[all of those `&dyn Any` are actually `&dyn Any + 'static`.](dyn-elision-trait-bounds.md#the-static-case)
I have also included an `Any` supertrait to `AsDynCompare`, so the
"always `'static`" property holds for `&dyn DynCompare` as well, even
though it isn't strictly necessary.  This way, we don't have to worry
about being flexible with the trait object lifetime at all -- it is
just always `'static`.

The downside is that only base types that satisfy the `'static` bound
can be supported, so there may be niche circumstances where you don't
want to include the supertrait bound.  However, given that we need to
upcast to `dyn Any`, this must mean you're pretending to be another
type, which seems quite niche indeed.  If you do try the non-`'static`
route for your own use case, note that some of the implementations in
this example could be made more general.

Anyway, let's move on to performing cross-type equality checking:
```rust
#use std::any::Any;
#
#trait AsDynCompare: Any {
#    fn as_any(&self) -> &dyn Any;
#    fn as_dyn_compare(&self) -> &dyn DynCompare;
#}
#
#impl<T: Any + DynCompare> AsDynCompare for T {
#    fn as_any(&self) -> &dyn Any {
#        self
#    }
#    fn as_dyn_compare(&self) -> &dyn DynCompare {
#        self
#    }
#}
#
#trait DynCompare: AsDynCompare {
#    fn dyn_eq(&self, other: &dyn DynCompare) -> bool;
#}
impl<T: Any + PartialEq> DynCompare for T {
    fn dyn_eq(&self, other: &dyn DynCompare) -> bool {
        if let Some(other) = other.as_any().downcast_ref::<Self>() {
            self == other
        } else {
            false
        }
    }
}

// n.b. this could be implemented in a more general way when
// the trait object lifetime is not constrained to `'static`
impl PartialEq<dyn DynCompare> for dyn DynCompare {
    fn eq(&self, other: &dyn DynCompare) -> bool {
        self.dyn_eq(other)
    }
}
```

Here we've utilized our `dyn Any` upcasting to try and recover a
parameter of our own base type, and if successful, do the actual
(partial) comparison.  Otherwise we say they're not equal.

This allows us to implement `PartialEq` for `dyn Compare`.

Then we want to wire this functionality up to our actual trait:
```rust
#use std::any::Any;
#
#trait AsDynCompare: Any {
#    fn as_any(&self) -> &dyn Any;
#    fn as_dyn_compare(&self) -> &dyn DynCompare;
#}
#
#impl<T: Any + DynCompare> AsDynCompare for T {
#    fn as_any(&self) -> &dyn Any {
#        self
#    }
#    fn as_dyn_compare(&self) -> &dyn DynCompare {
#        self
#    }
#}
#
#trait DynCompare: AsDynCompare {
#    fn dyn_eq(&self, other: &dyn DynCompare) -> bool;
#}
#impl<T: Any + PartialEq> DynCompare for T {
#    fn dyn_eq(&self, other: &dyn DynCompare) -> bool {
#        if let Some(other) = other.as_any().downcast_ref::<Self>() {
#            self == other
#        } else {
#            false
#        }
#    }
#}
#
#impl PartialEq<dyn DynCompare> for dyn DynCompare {
#    fn eq(&self, other: &dyn DynCompare) -> bool {
#        self.dyn_eq(other)
#    }
#}
trait Trait: DynCompare {}
impl Trait for i32 {}
impl Trait for bool {}

impl PartialEq<dyn Trait> for dyn Trait {
    fn eq(&self, other: &dyn Trait) -> bool {
        self.as_dyn_compare() == other.as_dyn_compare()
    }
}
```

The supertrait bound does most of the work, and we just use
upcasting again -- to `dyn DynCompare` this time -- to be
able to perform `PartialEq` on our `dyn Trait`.

[A blanket implementation in `std`](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html#impl-PartialEq%3CBox%3CT,+A%3E%3E-for-Box%3CT,+A%3E)
gives us `PartialEq` for `Box<dyn Trait>` automatically.

Now let's try it out:
```rust,compile_fail
#use std::any::Any;
#
#trait AsDynCompare: Any {
#    fn as_any(&self) -> &dyn Any;
#    fn as_dyn_compare(&self) -> &dyn DynCompare;
#}
#
#impl<T: Any + DynCompare> AsDynCompare for T {
#    fn as_any(&self) -> &dyn Any {
#        self
#    }
#    fn as_dyn_compare(&self) -> &dyn DynCompare {
#        self
#    }
#}
#
#trait DynCompare: AsDynCompare {
#    fn dyn_eq(&self, other: &dyn DynCompare) -> bool;
#}
#
#impl<T: Any + PartialEq> DynCompare for T {
#    fn dyn_eq(&self, other: &dyn DynCompare) -> bool {
#        if let Some(other) = other.as_any().downcast_ref::<Self>() {
#            self == other
#        } else {
#            false
#        }
#    }
#}
#
#impl PartialEq<dyn DynCompare> for dyn DynCompare {
#    fn eq(&self, other: &dyn DynCompare) -> bool {
#        self.dyn_eq(other)
#    }
#}
#
#trait Trait: DynCompare {}
#impl Trait for i32 {}
#impl Trait for bool {}
#
#impl PartialEq<dyn Trait> for dyn Trait {
#    fn eq(&self, other: &dyn Trait) -> bool {
#        self.as_dyn_compare() == other.as_dyn_compare()
#    }
#}
fn main() {
    let bx1a: Box<dyn Trait> = Box::new(1);
    let bx1b: Box<dyn Trait> = Box::new(1);
    let bx2: Box<dyn Trait> = Box::new(2);
    let bx3: Box<dyn Trait> = Box::new(true);

    println!("{}", bx1a == bx1a);
    println!("{}", bx1a == bx1b);
    println!("{}", bx1a == bx2);
    println!("{}", bx1a == bx3);
}
```
Uh... it didn't work, but for weird reasons.  Why is it trying to move out of the
`Box` for a comparison?  As it turns out, this is [a longstanding bug in the
language.](https://github.com/rust-lang/rust/issues/31740) Fortunately that issue
also offers a workaround that's ergonomic at the use site: implement `PartialEq<&Self>`
too.

```rust
#use std::any::Any;
#
#trait AsDynCompare: Any {
#    fn as_any(&self) -> &dyn Any;
#    fn as_dyn_compare(&self) -> &dyn DynCompare;
#}
#
#// Sized types only
#impl<T: Any + DynCompare> AsDynCompare for T {
#    fn as_any(&self) -> &dyn Any {
#        self
#    }
#    fn as_dyn_compare(&self) -> &dyn DynCompare {
#        self
#    }
#}
#
#trait DynCompare: AsDynCompare {
#    fn dyn_eq(&self, other: &dyn DynCompare) -> bool;
#}
#
#impl<T: Any + PartialEq> DynCompare for T {
#    fn dyn_eq(&self, other: &dyn DynCompare) -> bool {
#        if let Some(other) = other.as_any().downcast_ref::<Self>() {
#            self == other
#        } else {
#            false
#        }
#    }
#}
#
#impl PartialEq<dyn DynCompare> for dyn DynCompare {
#    fn eq(&self, other: &dyn DynCompare) -> bool {
#        self.dyn_eq(other)
#    }
#}
#
#trait Trait: DynCompare {}
#impl Trait for i32 {}
#impl Trait for bool {}
#
#impl PartialEq<dyn Trait> for dyn Trait {
#    fn eq(&self, other: &dyn Trait) -> bool {
#        self.as_dyn_compare() == other.as_dyn_compare()
#    }
#}
#
// New
impl PartialEq<&Self> for Box<dyn Trait> {
    fn eq(&self, other: &&Self) -> bool {
        <Self as PartialEq>::eq(self, *other)
    }
}

fn main() {
    let bx1a: Box<dyn Trait> = Box::new(1);
    let bx1b: Box<dyn Trait> = Box::new(1);
    let bx2: Box<dyn Trait> = Box::new(2);
    let bx3: Box<dyn Trait> = Box::new(true);

    println!("{}", bx1a == bx1a);
    println!("{}", bx1a == bx1b);
    println!("{}", bx1a == bx2);
    println!("{}", bx1a == bx3);
}
```
Ok, now it works.  Phew!

## `PartialOrd`

From here it's mostly mechanical to add `PartialOrd` support:
```diff
+use core::cmp::Ordering;

 trait DynCompare: AsDynCompare {
     fn dyn_eq(&self, other: &dyn DynCompare) -> bool;
+    fn dyn_partial_cmp(&self, other: &dyn DynCompare) -> Option<Ordering>;
 }

-impl<T: Any + PartialEq> DynCompare for T {
+impl<T: Any + PartialOrd> DynCompare for T {
     fn dyn_eq(&self, other: &dyn DynCompare) -> bool {
         if let Some(other) = other.as_any().downcast_ref::<Self>() {
             self == other
         } else {
             false
         }
     }
+
+    fn dyn_partial_cmp(&self, other: &dyn DynCompare) -> Option<Ordering> {
+        other
+            .as_any()
+            .downcast_ref::<Self>()
+            .and_then(|other| self.partial_cmp(other))
+    }
 }

+impl PartialOrd<dyn DynCompare> for dyn DynCompare {
+    fn partial_cmp(&self, other: &dyn DynCompare) -> Option<Ordering> {
+        self.dyn_partial_cmp(other)
+    }
+}

+impl PartialOrd<dyn Trait> for dyn Trait {
+    fn partial_cmp(&self, other: &dyn Trait) -> Option<Ordering> {
+        self.as_dyn_compare().partial_cmp(other.as_dyn_compare())
+    }
+}

+impl PartialOrd<&Self> for Box<dyn Trait> {
+    fn partial_cmp(&self, other: &&Self) -> Option<Ordering> {
+        <Self as PartialOrd>::partial_cmp(self, *other)
+    }
+}
```

[Here's the final playground.](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=8142d76abd2b2a4ef75950029e035371)
