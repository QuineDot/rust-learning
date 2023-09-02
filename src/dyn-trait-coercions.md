# `dyn Trait` coercions

Some `dyn Trait` coercions which are typical (in terms of what is being coerced) look like so:
```rust
# use std::sync::Arc;
# trait Trait {}
fn coerce_ref<'a, T: Trait + Sized + 'a>(t:    &T ) -> &(  dyn Trait + 'a) { t }
fn coerce_box<'a, T: Trait + Sized + 'a>(t: Box<T>) -> Box<dyn Trait + 'a> { t }
fn coerce_arc<'a, T: Trait + Sized + 'a>(t: Arc<T>) -> Arc<dyn Trait + 'a> { t }
// etc
```

These are more *syntactically noisy* than you will typically see in practice, as
I have included some explicit lifetimes and bounds which are normally implied
or not used.  For example the `Sized` bound on generic type parameters
[is usually implied,](https://doc.rust-lang.org/reference/special-types-and-traits.html#sized)
but I've made it explicit to emphasize that we're talking about `Sized` base types.

The key point is that given an [object safe `Trait`,](dyn-safety.md) and when
`T: 'a + Trait + Sized`, you can coerce a `Ptr<T>` to a `Ptr<dyn Trait + 'a>`
for the supported `Ptr` pointer types such as `&_` and `Box<_>`.

If we had wanted a `dyn Trait + Send + 'a`, naturally we would need `T: Send`
as well, and similarly for any other auto trait.

In the rest of this section, we look at cases beyond these typical examples,
as well as some limitations of coercions.

## Associated types

When a trait has one or more non-generic associated type, every concrete implementor of
the trait chooses a single, statically-known type for each associated type.  For base
types, this means the associated types are "outputs" of the implementing type and the
implemented trait: if you know the latter two, you can statically determine the
associated types as well.

So what should the associated types be in the implementation of `Trait` for
`dyn Trait`?

There is no single answer; they would need to vary based on the erased base types.

However, `dyn Trait` for traits with associated types is just too useful to
make traits with associated types inelligible for `dyn Trait`.  Instead, associated
types in the trait become, in essence, named *type parameters* of the `dyn Trait`
type constructor. (Recall it's already a type constructor due to the trait object lifetime.)

So given
```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```
We have
```rust,ignore
dyn Iterator<Item = String> + '_
dyn Iterator<Item = i32> + '_
dyn Iterator<Item = f64> + '_
```
and so on.  The associated types in `dyn Trait<...>` must be resolved to
concrete types in order for the `dyn Trait<...>` to be a concrete type.

Naturally, you can only coerce to `dyn Iterator<Item = String>` if you
both implement `Iterator`, and in your implementation, `type Item = String`.
The syntax mirrors that of associated type trait bounds:
```rust
fn takes_string_iter<Iter>(i: Iter)
where
    Iter: Iterator<Item = String>,
{
   // ...
}
```

The parameters being named has a number of benefits.  For one, it's
usually quite relevant, such as what `Item` an `Iterator` returns
(especially if the associated types are well named).  It also removes
the need to order the associated types in a well-defined way, such as
lexicographically or especially declaration order (which would be too fragile).

The named parameters must be specified after all ordered parameters, however.
```rust
trait AssocAndParams<T, U> { type Assoc1; type Assoc2; }

// The trait's ordered type parameters must be in declaration order
// (here, `String` then `usize`).  After that come the named associated
// type paramters, which can be reordered arbitrary amongst themselves.
fn foo(d: Box<dyn AssocAndParams<String, usize, Assoc1 = i32, Assoc2 = u32>>)
->
    Box<dyn AssocAndParams<String, usize, Assoc2 = u32, Assoc1 = i32>>
{
   d
}
```


## No nested coercions

An unsizing coercion needs to happen behind a layer of indirection (such as a
reference or in a `Box`) in order to accomodate the wide pointer to the erased
type's vtable (and because moving unsized types is not supported).

However, the unsizing coercion can only happen behind a *single* layer of
indirection.  For example, you can't coerce a `Vec<Box<T>>` to a `Vec<Box<dyn Trait>>`.
Why not?  `Box<T>` and `Box<dyn Trait>` have different layouts!  The former
is the size of one pointer, while the second is the size of two pointers.
The entire `Vec` would need to be reallocated to accomodate such a change:
```rust
# trait Trait {}
fn convert_vec<'a, T: Trait + 'a>(v: Vec<Box<T>>) -> Vec<Box<dyn Trait + 'a>> {
    v.into_iter().map(|bx| bx as _).collect()
}
```

In general, unsizing coercions consume the original pointer (reference, `Box`,
etc) and produce a new one, and this cannot happen in a nested context.

Internally, which coercions are possible are determined by the
[`CoerceUnsized`](https://doc.rust-lang.org/std/ops/trait.CoerceUnsized.html)
trait, and the (compiler-implemented) `Unsize` trait, as discussed in the
documentation.

### Except when you can

There are some material and some apparent exceptions where unsizing coercion
can occur in a nested context.

If you follow the link above, you'll see that [some types such as `Cell`
implement `CoerceUnsized` in a recursive manner.](https://doc.rust-lang.org/std/ops/trait.CoerceUnsized.html#impl-CoerceUnsized%3CCell%3CU%3E%3E-for-Cell%3CT%3E)
The idea is that `Cell` and the others have the same layout as their
generic type parameter.  As a result, outer layers of `Cell` don't count
as "nesting".
```rust
# use std::cell::Cell;
# trait Trait {}
// Fails :-(
//fn coerce_vec<'a, T: Trait + 'a>(v: Vec<Box<T>>) -> Vec<Box<dyn Trait + 'a>> {
//    v
//}

// Works! :-)
fn coerce_cell<'a, T: Trait + 'a>(c: Cell<Box<T>>) -> Cell<Box<dyn Trait + 'a>> {
    c
}
```

We'll cover the apparent exceptions (which are actually just supertype
coercions) [in an upcoming section.](./dyn-covariance.md#variance-in-nested-context)

## The `Sized` limitation

Base types must meet a `Sized` bound in order to be able to be coerced to
`dyn Trait`.  For example, `&str` cannot be coerced to `&dyn Display`
even though `str` implements `Display`, because `str` is unsized.

Why is this limitation in place?  `&str` is also a wide pointer; it consists
of a pointer to the UTF8 bytes, and a `usize` which is the number of bytes.
Similarly a slice reference `&[T]` is a pointer to the contiguous data, and
a count of the number of items.

A `&dyn Trait` created from a `&str` or `&[T]` would thus naively need to be
a "super-wide pointer", with a pointer to the data, the element count, *and*
the vtable pointer.  But `&dyn Trait` is a concrete type with a static layout
-- two pointers -- so this naive approach can't work.  Moreover, what if I
wanted to coerce a super-wide pointer?  Each recursive coercion requires
another pointer, making the size unbounded.

A non-naive approach would require special-casing how dynamic dispatch
works for erased non-`Sized` base types.  For example, once you've type
erased `str`, you've lost the information that `&str` is also a wide pointer,
and how to create that wide pointer.  However, the code would need to recreate
a wide pointer in order to perform dynamic dispatch.

So for `dyn Trait` to non-naively support unsized types, it would need need
to examine at run-time how to construct a pointer to the erased base type:
one possibility for thin pointers, and an additional possibility for each type
of wide pointer supported.  Not only that, but the metadata required (such as
the length of the `str`) has to be stored *somewhere*, and that can't be in
static memory like the vtable is.

Instead, unsized base types are simply not supported.

Sometimes you can work around the limitation by, for example, implementing
the trait for `&str` instead of `str`, and then coercing a `&'_ str` to
`dyn Trait + '_` (since references are always `Sized`).
```rust
# use std::fmt::Display;
// This fails as we cannot coerce `str` to `dyn Display`, so we cannot coerce
// `&str` to `&dyn Display`.
// let _: &dyn Display = "hi";

// However, `&str` also implements `Display`.  (If `T: Display`, then `&T: Display`.)
// Because `&str` is `Sized`, we can instead coerce `&&str` to `&dyn Display`:
let _: &dyn Display = &"hi";
```

`Sized` is also used as a sort of "not-`dyn`" marker,
[which we explore later.](dyn-safety.md#the-sized-constraints)

There is one broad exception to the `Sized` limitation: coercing between
forms of `dyn Trait` itself, which we look at immediately below.

## Discarding auto traits

You can coerce a `dyn Trait + Send` to a `dyn Trait`, and similarly discard
any other auto trait.

Although
[`dyn Trait` isn't a supertype of `dyn Trait + Send`,](./dyn-trait-overview.md#dyn-trait-is-not-a-supertype)
this is nonetheless referred to as *upcasting* `dyn Trait + Send` to `dyn Trait`.

Note that auto traits have no methods, and thus no change to the vtable is
required for these coercions.  They allow one to call a less restricted
function (that takes `dyn Trait`) from a more restrictive one (e.g. one that
requires `dyn Trait + Send`).  The coercion is necessary as, again, these are
(distinct) concrete types, and not generics nor subtypes nor dynamic types.

Although no change to the vtable is required, this coercion can still
[not happen in a nested context.](#no-nested-coercions)

## The reflexive case

You can cast `dyn Trait` to `dyn Trait`.

Sorry, we're being too imprecise again.  You can cast a `dyn Trait + 'a` to a `dyn Trait + 'b`,
where `'a: 'b`.  This is important for
[how borrowing works with `dyn Trait + '_`.](./dyn-covariance.md#unsizing-coercions-in-invariant-context)

As lifetimes are erased during compilation, the vtable is the same regardless of the lifetime.
Despite that, this unsizing coercion can still [not happen in a nested context.](#no-nested-coercions)

However, [in a future section](./dyn-covariance.md) we'll see
how variance can allow shortening the trait object lifetime even in nested context,
provided that context is also covariant.  [The section after that about higher-ranked
types](./dyn-hr.md) explores another lifetime-related coercion which could also be
considered reflexive.

## Supertrait upcasting

Though not supported on stable yet,
[the ability to upcast from `dyn SubTrait` to `dyn SuperTrait`](https://github.com/rust-lang/rust/issues/65991)
is a feature expected to be available some day.

It is, once again, explicitly a coercion and not a sub/super type relationship
(despite the terminology).  Although this is an implementation detail, the
conversion will probably involve replacing the vtable pointer (in contrast
with the last couple of examples).

Until the feature is stable,
[you can write your own "manual" supertrait upcasts.](./dyn-trait-combining.md#manual-supertrait-upcasting)

## Object-safe traits only

There are other restrictions on the *trait* which we have not discussed here,
such as not (yet) supporting traits with generic associated types (GATs).
[We cover those in the next section.](dyn-safety.md)
