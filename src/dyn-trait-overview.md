# `dyn Trait` Overview

## What is `dyn Trait`?

`dyn Trait` is a compiler-provided type which implements `Trait`.  Any `Sized` implementor
of `Trait` can be coerced to be a `dyn Trait`, erasing the original base type in the process.
Different implementations of `Trait` may have different sizes, and as a result, `dyn Trait`
has no statically known size.  That means it does not implement `Sized`, and we call such
types "unsized", or "dynamically sized types (DSTs)".

Every `dyn Trait` value is the result of type erasing some other existing value.
You cannot create a `dyn Trait` from a trait definition alone; there must be an
implementing base type that you can coerce.

Rust currently does not support passing unsized parameters, returning unsized values, or
having unsized locals.  Therefore, when interacting with `dyn Trait`, you will generally
be working with some sort of indirection: a `Box<dyn Trait>`, `&dyn Trait`, `Arc<dyn Trait>`,
etc.

And in fact, the indirection is necessary for another reason.  These indirections are or
contain *wide pointers* to the erased type, which consist of a pointer to the value, and
a second pointer to a static *vtable*.  The vtable in turn contains data such as the size
of the value, a pointer to the value's destructor, pointers to methods of the `Trait`,
and so on.  The vtable [enables dynamic dispatch,](./dyn-trait-impls.md) by which
different `dyn Trait` values can dispatch method calls to the different erased base type
implementations of `Trait`.

`dyn Trait` is also called a "trait object".

You can also have objects such as `dyn Trait + Send + Sync`.  `Send` and `Sync` are
[auto-traits,](https://doc.rust-lang.org/reference/special-types-and-traits.html#auto-traits)
and a trait object can include any number of these auto traits as additional bounds.
Every distinct set of `Trait + AutoTraits` is a distinct type.

However, you can only have one *non*-auto trait in a trait object, so this will not
work:
```rust,compile_fail
trait Trait1 {}
trait Trait2 {};
struct S(Box<dyn Trait1 + Trait2>);
```

That being noted, [one can usually use a subtrait/supertrait pattern](./dyn-trait-combining.md)
to work around this restriction.

### The trait object lifetime

Confession: we were being imprecise when we said `dyn Trait` is a type.  `dyn Trait` is a
*type constructor*: [it is parameterized with a lifetime,](./dyn-trait-lifetime.md)
similar to how references are.  So `dyn Trait` on it's own isn't a type,
`dyn Trait + 'a` for some concrete lifetime `'a` is a type.

The lifetime can usually be elided, [which we will explore later.](./dyn-elision.md)
But it is always part of the type,
[just like a lifetime is part of every reference type,](./st-types.md)
even when elided.

### Associated Types

If a trait has non-generic associated types, those associated types become
named parameters of `dyn Trait`:
```rust
let _: Box<dyn Iterator<Item = i32>> = Box::new([1, 2, 3].into_iter());
```
We explore this some more [in a later section.](./dyn-trait-coercions.md#associated-types)

## What `dyn Trait` is *not*

### `dyn Trait` is not `Sized`

We mentioned the fact that `dyn Trait` is not `Sized` already.

However, let us take a moment to note that generic parameters
[have an implicit `Sized` bound.](https://doc.rust-lang.org/reference/special-types-and-traits.html#sized)
Therefore you may need to remove the implicit bound by using
`: ?Sized` in order to use `dyn Trait` in generic contexts.
```rust,compile_fail
# trait Trait {}
// This function only takes `T: Sized`.  It cannot accept a
// `&dyn Trait`, for example, as `dyn Trait` is not `Sized`.
fn foo<T: Trait>(_: &T) {}

// This function takes any `T: Trait`, even if `T` is not
// `Sized`.
fn bar<T: Trait + ?Sized>(t: &T) {
    // Demonstration that `foo` cannot except non-`Sized`
    // types:
    foo(t);
}
```


### `dyn Trait` is neither a generic nor dynamically typed

Given a concrete lifetime `'a`, `dyn Trait + 'a` is a statically known type.
For example, consider these two function signatures:
```rust
# trait Trait {}
fn generic<T: Trait>(_rt: &T) {}
fn not_generic(_dt: &dyn Trait) {}
```

In the generic case, a distinct version of the function will exist for every
type `T` which is passed to the function.  This compile-time generation of
new functions for every type is known as monomorphization.  (Side note,
lifetimes are erased during compilation, and not monomorphized.)

You can even create function pointers to the different versions like so:
```rust
# trait Trait {}
# impl Trait for String {}
# fn generic<T: Trait>(_rt: &T) {}
# fn main() {
let fp = generic::<String>;
# }
```
That is, the function item type is parameterized by some `T: Trait`.

In contrast, there will always only be only one `non_generic` function in
the resulting library.  The base implementors of `Trait` must be typed-erased
into `dyn Trait + '_` before being passed to the function.  The function type
is not parameterized by a generic type.

Similarly, here:
```rust
# trait Trait {}
fn generic<T: Trait>(bx: Box<T>) {}
```
`bx: Box<T>` is not a `Box<dyn Trait>`.  It is a thin owning pointer to a
heap allocated `T` specifically.  Because `T` has an implicit `Sized` bound
here, we could *coerce* `bx` to a `Box<dyn Trait + '_>`.  But that would be a
transformation to a different type of `Box`: a wide owning pointer which has
erased `T` and included the corresponding vtable pointer.

We'll explore more details on the interaction of generics and `dyn Trait`
[in a later section.](./dyn-trait-vs.md)

You may wonder why you can use the methods of `Trait` on a `&dyn Trait` or
`Box<dyn Trait>`, etc., despite not declaring any such bound.  The reason is
analogous to why you can use `Display` methods on a `String` without declaring
that bound, say: the type is staically known, and the compiler recognizes that
`dyn Trait` implements `Trait`, just like it recognizes that `String`
implements `Display`.  Trait bounds are needed for generics, not concrete types.

(In fact, [`Box<dyn Trait>` doesn't implement `Trait` automatically,](./dyn-trait-impls.md#boxdyn-trait-and-dyn-trait-do-not-automatically-implement-trait)
but deref coercion usually takes care of that case. For many `std` traits,
the trait is explicitly implemented for `Box<dyn Trait>` as well;
[we'll also explore what that can look like.](./dyn-trait-box-impl.md))

### `dyn Trait` is not a supertype

Because you can coerce base types into a `dyn Trait`, it is not uncommon for
people to think that `dyn Trait` is some sort of supertype over all the
coercible implementors of `Trait`.  The confusion is likely exacerbated by
trait bounds and lifetime bounds sharing the same syntax.

But the coercion from a base type to a `dyn Trait` is an unsizing coercion,
and not a sub-to-supertype conversion; the coercion happens at statically
known locations in your code, and may change the layout of the types
involved (e.g. changing a thin pointer into a wide pointer) as well.

Relatedly, `trait Trait` is not a class.  You cannot create a `dyn Trait`
without an implementing type (they do not have built-in constructors),
and a given type can implement a great many traits.  Due to the confusion it
can cause, I recommend not referring to base types as "instances" of the trait.
It is just a type that implements `Trait`, which exists independently of the
trait.  When I create a `String`, I'm creating a `String`, not "an instance
of `Display` (and `Debug` and `Write` and `ToString` and ...)".

When I read "an instance of `Trait`", I assume the variable in question is
some form of `dyn Trait`, and not some unerased base type that implements `Trait`.

Implementing something for `dyn Trait` does not implement it for all other
`T: Trait`.  In fact it implements it for nothing but `dyn Trait` itself.
Implementing something for `dyn Trait + Send` doesn't implement anything
for `dyn Trait` or vice-versa either; those are also separate, distinct types.

There are ways to *emulate* dynamic typing in Rust, [which we will explore later.](./dyn-any.md)
We'll also explore the role of [*supertraits*](./dyn-trait-combining.md) (which, despite the
name, still do not define a sub/supertype relationship).

The *only* subtypes in Rust involve lifetimes and types which are
higher-ranked over lifetimes.

<small>(Pedantic self-correction: trait objects have lifetimes and thus [are supertypes in that sense.](./dyn-covariance.md) However that's not the same concept that most Rust learners get confused about; there is no supertype relationship with the implementing types.)</small>

### `dyn Trait` is not universally applicable

We'll look at the details in their own sections, but in short, you cannot
always coerce an implementor of `Trait` into `dyn Trait`.  Both
[the trait](./dyn-safety.md) and [the implementor](dyn-trait-coercions.md)
must meet certain conditions.

## In summary

`dyn Trait + 'a` is
* a concrete, statically known type
* created by type erasing implementors of `Trait`
* used behind wide pointers to the type-erased value and to a static vtable
* dynamically sized (unsized, does not implement `Sized`)
* an implementor of `Trait` via dynamic dispatch
* *not* a supertype of all implementors
* *not* dynamically typed
* *not* a generic
* *not* creatable from all values
* *not* available for all traits

