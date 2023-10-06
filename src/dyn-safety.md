# `dyn` safety (object safety)

There exists traits for which you cannot create a `dyn Trait`:
```rust,compile_fail
let s = String::new();
let d: &dyn Clone = &s;
```

Instead of repeating all the rules here,
[I'll just link to the reference.](https://doc.rust-lang.org/reference/items/traits.html#object-safety)
You should go read that first.

What may not be immediately apparent is *why* these limitations exists.
The rest of this page explores some of the reasons.

## The `Sized` constraints

Before we get into the restrictions, let's have an aside about how the
`Sized` constraints work with `dyn Trait` and `dyn` safety.

Rust uses `Sized` to indicate that
- A trait is not `dyn` safe
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
will also not be available for other unsized types).

The next few paragraphs demonstrate (or perhaps rant about) how this can be an annoying limitation.
If you'd rather get on with learning practical Rust, [you may want to skip ahead 🙂.](#receiver-limitations)

Consider this example, where we've added a `Sized` bound in order to remain a `dyn`-safe trait:
```rust
# trait Bound<T: ?Sized> {}
trait Trait {
    // Opt-out of `dyn`-dispatchability for this method because it's generic
    fn method<T: Bound<Self>>(&self) where Self: Sized;
}
```

If you try to implement this trait for `str`, you won't have `method`
available, even if it would logically make sense to have it available.
Moreover, if you write the implementation like so:
```rust,compile_fail
# trait Bound<T: ?Sized> {}
# trait Trait {
#    fn method<T: Bound<Self>>(&self) where Self: Sized;
# }
impl Trait for str {
    // `Self: Sized` isn't true, so don't bother with `method`
}
```
You get an error saying you must provide `method`, even though the
bounds cannot be satisfied.  So then you can provide a perfectly
functional implementation:
```rust,compile_fail
# trait Bound<T: ?Sized> {}
# trait Trait {
#    fn method<T: Bound<Self>>(&self) where Self: Sized;
# }
impl Trait for str {
    fn method<T: Bound<Self>>(&self) where Self: Sized {
        // do logical `method` things
    }
}
```
Whoops, it doesn't accept that either! 😠 We have to implement it
without the bound, like so:
```rust
# trait Bound<T: ?Sized> {}
# trait Trait {
#    fn method<T: Bound<Self>>(&self) where Self: Sized;
# }
impl Trait for str {
    fn method<T: Bound<Self>>(&self) {
        // do logical `method` things
    }
}
```
And that compiles... but we can never actually call it.
```rust,compile_fail
# trait Bound<T: ?Sized> {}
# trait Trait {
#    fn method<T: Bound<Self>>(&self) where Self: Sized;
# }
# impl Trait for str {
#     fn method<T: Bound<Self>>(&self) {
#     }
# }
fn main() {
    "".method();
}
```
Alternatively, we can exploit the fact that higher-ranked bounds
are checked at the call site and not the definition site to sneak
in the unsatisfiable `Self: Sized` bound in a way that compiles:
```rust
# trait Bound<T: ?Sized> {}
# trait Trait {
#    fn method<T: Bound<Self>>(&self) where Self: Sized;
# }
impl Trait for str {
    // Still not callable, but compiles:   vvvvvvv  due to this binder
    fn method<T: Bound<Self>>(&self) where for<'a> Self: Sized {
        unreachable!()
    }
}
```
But naturally the method still cannot be called, as the bound is not satisfiable.

This is a pretty sad state of affairs.  Ideally, there would be a
distinct trait for opting out of `dyn` safety and dispatchability
instead of using `Sized` for this purpose; let's call it `NotDyn`.
Then we could have `Sized: NotDyn` for backwards compatibility,
change the bound above to be `NotDyn`, and have our implementation
for `str` be functional.

There also some other future possibilities that may improve the situation:
- [Some resolution of RFC issue 2829](https://github.com/rust-lang/rfcs/issues/2829)
or the duplicates linked within would allow omitting the method altogether
(but it would still not be callable)
- [RFC 2056](https://rust-lang.github.io/rfcs/2056-allow-trivial-where-clause-constraints.html)
will allow defining the method with the trivially unsatifiable bound without
exploiting the higher-ranked trick (but it will still not be callable)
- [RFC 3245](https://rust-lang.github.io/rfcs/3245-refined-impls.html) will allow
calling `<str as Trait>::method` and refined implementations more generally

But I feel removing the conflation between `dyn` safety and `Sized` would
be more clear and correct regardless of any future workarounds that may exist.

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

## Associated constant limitations

Similarly, supporting associated constants will require at least
[support for associated constant equality.](https://github.com/rust-lang/rust/issues/92827)

## History

[Object safety was introduced in RFC 0255,](https://rust-lang.github.io/rfcs/0255-object-safety.html)
and [RFC 0546](https://rust-lang.github.io/rfcs/0546-Self-not-sized-by-default.html) removed the
implied `Sized` bound on traits and added the rule that traits with (explicit) `Sized` bounds
are not object safe.

Both RFCs were implemented before Rust 1.0.
