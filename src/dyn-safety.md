# `dyn` compatibility (object safety, `dyn` safety)

There exists traits for which you cannot create a `dyn Trait`:
```rust,compile_fail
let s = String::new();
let d: &dyn Clone = &s;
```

Instead of repeating all the rules here,
[I'll just link to the reference.](https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility)
You should go read that first.  (As the reference notes, `dyn` compatibility use
to be called object safety.  And an earlier version of this guide used the term "`dyn` safety".)

Note that as of this writing, the reference hasn't been updated to document that you
can opt to make associated types and <abbr title="generic associated types">GATs</abbr>
unavailable to trait objects by adding a `where Self: Sized` bound.  For now I'll
refer to this as opting the GAT (or associated type) out of being "`dyn`-usable".

What may not be immediately apparent is *why* these limitations exists.
The rest of this page explores some of the reasons.

## The `Sized` constraints

Before we get into the restrictions, let's have an aside about how the
`Sized` constraints work with `dyn Trait` and `dyn` compatibility.

Rust uses `Sized` to indicate that
- A trait is not `dyn` compatible
- An associated type or <abbr title="generic associated type">GAT</abbr> is not `dyn`-usable
- A method is not `dyn`-dispatchable
- An associated function is not callable for `dyn Trait`
  - Even though it never can be (so far), you have to declare this for the sake of being explicit and for potential forwards compatibility

This makes some sense, as `dyn Trait` is not `Sized`.  So a `dyn Trait`
cannot implement a trait with `Sized` as a supertrait, and a `dyn Trait`
can't call methods (or associated functions) that require `Sized` either.

However, it's still a hack as there are types which are not `Sized` but also
not `dyn Trait`, and we might want to implement our trait for those, *including*
some methods which are not `dyn`-dispatchable (such as generic methods).
Currently that's just not possible in Rust (the non-`dyn`-dispatchable methods
will also not be available for other unsized types which implement the trait).

<details>
<summary>Expand if you'd like to read a rant about the limitations of this hack.</summary>

Consider this example, where we've added a `Sized` bound in order to remain a `dyn` compatible trait:
```rust
# trait Bound<T: ?Sized> {}
trait Trait {
    // Opt-out of `dyn`-dispatchability for this method because it's generic
    fn method<T: Bound<Self>>(&self) where Self: Sized;
}
```

If you try to implement this trait for `str`, you won't have `method`
available, even if it would logically make sense to have it available.
You're allowed to write the implementation like so, but aren't allowed to
call the method, because the bounds on the trait take precedence.
```rust
# trait Bound<T: ?Sized> {}
# trait Trait {
#    fn method<T: Bound<Self>>(&self) where Self: Sized;
# }
// This compiles...
impl Trait for str {
    // N.b. we omitted the bound in our implementation.
    fn method<T: Bound<Self>>(&self) {
        println!("Hi there");
    }
}
```
```rust,compile_fail
# trait Bound<T: ?Sized> {}
# trait Trait {
#    fn method<T: Bound<Self>>(&self) where Self: Sized;
# }
# impl Trait for str {
#     fn method<T: Bound<Self>>(&self) {
#         println!("Hi there");
#     }
# }
# impl Bound<str> for i32 {}
// ...but the method is not callable.
"".method::<i32>()
```

So you might as well just leave it out.
```rust
# trait Bound<T: ?Sized> {}
# trait Trait {
#    fn method<T: Bound<Self>>(&self) where Self: Sized;
# }
impl Trait for str {
    // You can just omit the method since Rust 1.87
}
```

This is a pretty sad state of affairs.  Ideally, there would be a
distinct trait for opting out of `dyn` compatibility and dispatchability
instead of using `Sized` for this purpose; let's call it `NotDyn`.
Then we could have `Sized: NotDyn` for backwards compatibility,
change the bound above to be `NotDyn`, and have our implementation
for `str` be functional.

There has been some improvement to the situation, and are also some other future
possibilities that may improve the situation more:

- [Since Rust 1.87,](https://github.com/rust-lang/rust/pull/135480) you can
simply omit the uncallable method (before that you had to supply the method)
- [RFC 2056](https://rust-lang.github.io/rfcs/2056-allow-trivial-where-clause-constraints.html)
will also allow defining the method with the trivially unsatifiable bound without
[exploiting higher-ranked bound tricks](https://github.com/rust-lang/rust/issues/48214#issuecomment-1150463333)
(but it will still not be callable)
- [RFC 3245](https://rust-lang.github.io/rfcs/3245-refined-impls.html) or
an expansion thereof may allow calling `<str as Trait>::method` and refined
implementations more generally

But I feel removing the conflation between `dyn` compatibility and `Sized` would
be more clear and correct regardless of any future workarounds that may exist.

</details>

## Receiver limitations

The requirement for some sort of `Self`-based receiver on `dyn`-dispatchable
methods is to ensure the vtable is available.  Some wide pointer to `Self`
needs to be present in order to
[find the vtable and perform dynamic dispatch.](dyn-trait-impls.md#how-dyn-trait-implements-trait)

Arguably this could be expanded to methods that take a single,
non-receiver `&Self` and so on.

As for the other limitation on receiver types, [the compiler has to know
how to go backwards from type erased version to original
version](./dyn-trait-impls.md#other-receivers) in order to
implement `Trait`.  This may be generalized some day, but for
now it's a restricted set.

## Generic method limitations

In order to support type-generic methods, there would need to be
a function pointer in the vtable for every possible type that the
generic could take on.  Not only would this create vtables of
unwieldly size, it would also require some sort of global analysis.
After all, every crate which uses your trait might define new types
that meet the trait bounds in question, and they (or you) might also
want to call the method using those types.

You can sometimes work around this limitation by type erasing the
generic type parameter in question (in the main method, as an
alternative method, or in a different "erased" trait).
[We'll see an example of this later.](./dyn-trait-erased.md)

## Use of `Self` limitations

Methods which take some form of `Self` other than as a receiver
can depend on the parameter being exactly the same as the
implementing type.  But this can't be relied upon once the base
types have been erased.

For example, consider [`PartialEq<Self>`:](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html)
```rust
// Simplified
pub trait PartialEq {
    fn partial_eq(&self, rhs: &Self);
}
```
If this were implemented for `dyn PartialEq`, the `rhs` parameter
would be a `&dyn PartialEq` like `self` is.  But there is no
guarantee that the base types are the same!  Both `u8` and `String`
implement `PartialEq` for example, but there's no facility to
compare them for equality (and Rust has no interest in handling
this in an arbitrary way).

You can sometimes work around this by supplying your own implementations
for some *other* `dyn Trait`, perhaps utilizing the `Any` trait
to emulate dynamic typing and reflection.
[We give an example of this approach later.](./dyn-trait-eq.md)

[The `impl Clone for Box<dyn Trait>` example](./dyn-trait-clone.md)
demonstrates handling a case where `Self` is the return value.

## GAT limitations

GATs are too new to support type erasing as-of-yet.  We'll need
some way to embed the GAT into the `dyn Trait` as a parameter,
[similar to how is done for non-generic associated types.](./dyn-trait-coercions.md#associated-types)

[As of Rust 1.72,](https://github.com/rust-lang/rust/pull/112319/)
you can opt out of GATs being `dyn`-usable, and thus out of the
necessity of naming the GAT as a parameter, by adding a
`Self: Sized` bound.

This is similar to [the same ability on non-generic associated types.](dyn-trait-coercions.md#opting-out-of-dyn-usability)
Interestingly, it allows specifying not only *specific* GAT equalities...
```rust
trait Trait {
    type Gat<'a> where Self: Sized;
}

impl Trait for () {
    type Gat<'a> = &'a str;
}

let _: &dyn Trait<Gat<'static> = &'static str> = &();
```
...but also higher-ranked GAT equality:
```rust,compile_fail
# trait Trait {
#    type Gat<'a> where Self: Sized;
# }
# impl Trait for () {
#    type Gat<'a> = &'a str;
# }
// This syntax is still not supported
let _: &dyn Trait<for<'a> Gat<'a> = &'a str> = &();
```
```rust
# trait Trait {
#    type Gat<'a> where Self: Sized;
# }
# impl Trait for () {
#    type Gat<'a> = &'a str;
# }
// However, with `dyn Trait`, you can move the binder to outside the `Trait`:
let _: &dyn for<'a> Trait<Gat<'a> = &'a str> = &();
```
However, as with the non-generic associated type case, making any use of the
equality would have to be done indirectly, as the `dyn Trait` itself cannot
define a GAT in its own implementation.

## Associated constant limitations

Similarly, supporting associated constants will require at least
[support for associated constant equality.](https://github.com/rust-lang/rust/issues/92827)

## Return position `impl Trait` limitations

Trait methods utilizing
[<abbr title="return position impl traits">RPITs</abbr>](dyn-trait-vs.md#return-position-impl-trait-and-tait)
are, notionally at least, sugar for declaring an opaque associated
type or generic associated type.  Additionally, even if the RPIT
captures no generic parameters and thus corresponds to returning
an associated type, there is currently no way to name that associated
type.

Similar to [generic methods,](#generic-method-limitations) you can
sometimes work around this limitation by type erasing the return
type.  (Note that there are some trade-offs, but a discussion of
such is more suited to a dedicated guide about RPITs.)

## History

[Object safety was introduced in RFC 0255,](https://rust-lang.github.io/rfcs/0255-object-safety.html)
and [RFC 0546](https://rust-lang.github.io/rfcs/0546-Self-not-sized-by-default.html) removed the
implied `Sized` bound on traits and added the rule that traits with (explicit) `Sized` bounds
are not object safe.

Both RFCs were implemented before Rust 1.0.
