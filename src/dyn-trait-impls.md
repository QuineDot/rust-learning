# `dyn Trait` implementations

In order for `dyn Trait` to be useful for abstracting over the base
types which implement `Trait`, `dyn Trait` itself needs to implement
`Trait`.  The compiler always supplies that implementation.  Here we
look at how this notionally works, and also touch on how this leads
to some related limitations around `dyn Trait`.

We also cover a few surprising corner-cases related to how the
implementation of `Trait` for `dyn Trait` works... or doesn't.

## How `dyn Trait` implements `Trait`

Let us note upfront: this is a rough sketch, and not normative.  What the
compiler actually does is an implementation detail.  But by providing a
sketch of how it *could* be implemented, we hope to provide some intuition
for `dyn Trait` being a concrete type, and some explanation of the
limitations that `dyn Trait` has.

With that disclaimer out of the way, let's look at what the compiler
implementation might look like for this trait:
```rust
trait Trait {
    fn look(&self);
    fn add(&mut self, s: String) -> i32;
}
```

Recall that when dealing with `dyn Trait`, you'll be dealing with
a pointer to the erased base type, and with a vtable.  For example,
we could imagine a `&dyn Trait` looks something like this:
```rust,ignore
#[repr(C)]
struct DynTraitRef<'a> {
    _lifetime: PhantomData<&'a ()>,
    base_type: *const (),
    vtable: &'static DynTraitVtable,
}

// Pseudo-code
type &'a dyn Trait = DynTraitRef<'a>;
```
Here we're using a thin `*const ()` to point to the erased base type.
Similarly, you can imagine a `DynTraitMut<'a>` for `&'a mut dyn Trait`
that uses `*mut ()`.

And the vtable might look something like this:
```rust
#[repr(C)]
struct DynTraitVtable {
    fn_drop: fn(*mut ()),
    type_size: usize,
    type_alignment: usize,
    fn_look: fn(*const ()),
    fn_add: fn(*mut (), s: String) -> i32,
}
```

And the implementation itself could look something like this:
```rust,ignore
impl Trait for dyn Trait + '_ {
    fn look(&self) {
        (self.vtable.fn_look)(self.base_type)
    }
    fn add(&mut self, s: String) -> i32 {
        (self.vtable.fn_add)(self.base_type, s)
    }
}
```

In summary, we've erased the base type by replacing references to the
base type with the appropriate type of pointer to the same data, both
in the wide references (`&dyn Trait`, `&mut dyn Trait`), and also in
the vtable function pointers.  The compiler guarantees there's no ABI
mismatch.

*Reminder:* This is just a rough sketch on how `dyn Trait` can be
implemented to aid the high-level understanding and discussion, and
not necessary exactly how they *are* implemented.

[Here's another blog post on the topic.](https://huonw.github.io/blog/2015/01/peeking-inside-trait-objects/)
Note that it was written in 2015, and some things in Rust have changed
since that time.  For example, [trait objects used to be "spelled" just
`Trait` instead of `dyn Trait`.](https://rust-lang.github.io/rfcs/2113-dyn-trait-syntax.html)
You'll have to figure out if they're talking about the trait or the
`dyn Trait` type from context.

## Other receivers

Let's look at one other function signature:
```rust
trait Trait {
    fn eat_box(self: Box<Self>);
}
```

How does this work?  Internally, a `Box<BaseType /* : Sized */>` is
a thin pointer, while a `Box<dyn Trait>` is wide pointer, very similar
to `&mut dyn Trait` for example (although the `Box` pointer implies ownership and
not just exclusivity).  The implementation for this method would be
similar to that of `&mut dyn Trait` as well:
```rust,ignore
// Still just for illustrative purpose
impl Trait for dyn Trait + '_ {
    fn eat_box(self: Box<Self>) {
        let BoxRepresentation { base_type, vtable } = self;
        let boxed_type = Box::from_raw(base_type);
        (vtable.fn_eat_box)(boxed_type);
    }
}
```

In short, the compiler knows how to go from the type-erased form
(like `Box<Self>`) into something ABI compatible for the base type
(`Box<BaseType>`) for every supported receiver type.

It's an implementation detail, but currently the way the compiler
knows how to do the conversion is via the
[`DispatchFromDyn`](https://doc.rust-lang.org/std/ops/trait.DispatchFromDyn.html)
trait.  The documentation lists the current limitations of supported
types (some of which are only available under the unstable
[`arbitrary_self_types` feature](https://github.com/rust-lang/rust/issues/44874)).

## Supertraits are also implemented

[We'll look at supertraits in more detail later,](./dyn-trait-combining.md) but
here we'll briefly note that when you have a supertrait:
```
trait SuperTrait { /* ... */ }
trait Trait: SuperTrait { /* ... */ }
```
The vtable for `dyn Trait` includes the methods of `SuperTrait` and the compiler
supplies an implementation of `SuperTrait` for `dyn Trait`, just as it supplies
an implementation of `Trait`.

## `Box<dyn Trait>` and `&dyn Trait` do not automatically implement `Trait`

It may come as a surprise that neither `Box<dyn Trait>` nor
`&dyn Trait` automatically implement `Trait`.  Why not?

In short, because it's not always possible.

[As we'll cover later,](dyn-safety.md#the-sized-constraints) a trait
may have methods which are not dispatchable by `dyn Trait`, but must
be implemented for any `Sized` type. One example is associated
functions that have no receiver:
```rust
trait Trait {
    fn no_receiver() -> String where Self: Sized;
}
```

There's no way for the compiler to generate the body of such an associated
function, and it can't provide a complete `Trait` implementation without
one.

Additionally, the receivers of dispatchable methods don't always make
sense:
```rust
trait Trait {
    fn takes_mut(&mut self);
}
```

A `&dyn Trait` can produce a `&BaseType`, but not a `&mut BaseType`, so
there is no way to implement `Trait::takes_mut` for `&dyn Trait` when
the only pre-existing implementation is for `BaseType`.

Similarly, an `Arc<dyn Trait>` has no way to call a `Box<dyn Trait>`
or vice-versa, and so on.

### Implementing these yourself

If `Trait` is a local trait, you can implement it for `Box<dyn Trait + '_>`
and so on just like you would for any other type.  Take care though, as it
can be easy to accidentally write a recursive definition!

[We walk through an example of this later on.](./dyn-trait-box-impl.md)

Moreover, `&T`, `&mut T`, and `Box<T>` are
[*fundamental*,](https://rust-lang.github.io/rfcs/2451-re-rebalancing-coherence.html#definitions)
which means that when it comes to the orphan rules (which gate which trait
implementations you can write), they act the same as `T`.  Additionally,
if `Trait` is a local trait, then `dyn Trait + '_` is a local type.

Together that means that *you can even implement **other** traits* for
`Box<dyn Trait + '_>` (and other fundamental wrappers)!

[We also have an example of this later on.](dyn-trait-clone.md)

Unfortunately, `Rc`, `Arc`, and so on are not fundamental, so this doesn't
cover every possible use case.

## The implementation cannot be directly overrode

The compiler provided implementation of `Trait` for `dyn Trait` cannot be
overrode by an implementation in your code.  If you attempt to define your
own definition directly, you'll get a compiler error:
```rust,compile_fail
trait Trait {}
impl Trait for dyn Trait + '_ {}
```

And if you have a blanket implementation to implement `Trait` and `dyn Trait`
happens to meet the bounds on the implementation, it will be ignored and the
compiler defined implementation will still be used:
```rust
#use std::any::type_name;
#
trait Trait {
    fn hi(&self) {
        println!("Hi from {}!", type_name::<Self>());
    }
}

// The simplest example is an implementation for absolutely everything
impl<T: ?Sized> Trait for T {}

let dt: &dyn Trait = &();
// Prints "Hi from ()!" and not "Hi from dyn Trait!"
dt.hi();
// Same thing
<dyn Trait as Trait>::hi(dt);
```

[This even applies with more complicated implementations,](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=66a0de00d42d915134f206ee73291136)
and applies to the supertrait implementations for `dyn Trait` as well.

[We'll see that this can be useful later.](./dyn-trait-erased.md)  But
unfortunately, [there are some compiler bugs around the compiler
implementation taking precedence over your blanket implementations.](https://github.com/rust-lang/rust/issues/57893#issuecomment-510690333)
How those bugs are dealt with is yet to be determined; it's possible
that certain blanket implementations will be disallowed, or that
some traits will no longer be `dyn`-safe.  (The *general* pattern,
such as the simple example above, is almost surely too widespread
to be deprecated.)

## The implementation cannot be indirectly bypassed

You may be aware that when a concrete type has an inherent method with
the same name and receiver as a trait method, the inherent method takes
precedence when performing method lookup:
```rust
trait Trait { fn method(&self) { println!("In trait Trait"); } }

struct S;
impl Trait for S {}
impl S { fn method(&self) { println!("In impl S"); } }

fn main() {
    let s = S;
    s.method();
    // If you wanted to use the trait, you can do this
    <S as Trait>::method(&s);
}
```

Unfortunately, this functionality is not available for `dyn Trait`.
You can write the implementation, but unlike the example above, they
will be considered ambiguous with the trait methods:
```rust,compile_fail
trait Trait {
    fn method(&self) {}
    fn non_dyn_dispatchable(&self) where Self: Sized {}
}

impl dyn Trait + '_ {
    fn method(&self) {}
    fn non_dyn_dispatchable(&self) {}
}

fn foo(d: &dyn Trait) {
    d.method();
    d.non_dyn_dispatchable();
}
```

Moreover, there is no syntax to call the inherent methods specifically
like there is for normal `struct`s.
[Even if you try to hide the trait,](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=6f55d2ebb8b44349034fc4df120e152f)
the inherent methods are unreachable, dead code.

Apparently the idea is that the trait methods "are" the inherent methods of
`dyn Trait`, but this is rather unfortunate as it prevents directly providing
something like the `non_dyn_dispatchable` override attempted above.
See [issue 51402](https://github.com/rust-lang/rust/issues/51402) for more
information.

Implementing methods on `dyn Trait` that don't attempt to shadow the
methods of `Trait` does work, however.
```rust
#trait Trait {}
impl dyn Trait + '_ {
    fn some_other_method(&self) {}
}

fn bar(d: &dyn Trait) {
    d.some_other_method();
}
```

## A niche exception to `dyn Trait: Trait`

Some bounds on traits aren't checked until you try to utilize the trait,
even when the trait is considered object safe.  As a result, [it is
actually sometimes possible to create a `dyn Trait` that does not implement
`Trait`!](https://github.com/rust-lang/rust/issues/88904)

```rust,compile_fail
trait Iterable
where
    for<'a> &'a Self: IntoIterator<
        Item = &'a <Self as Iterable>::Borrow,
    >,
{
    type Borrow;
    fn iter(&self) -> Box<dyn Iterator<Item = &Self::Borrow> + '_> {
        Box::new(self.into_iter())
    }
}

impl<I: ?Sized, Borrow> Iterable for I
where
    for<'a> &'a Self: IntoIterator<Item = &'a Borrow>,
{
    type Borrow = Borrow;
}

fn example(v: Vec<String>) {
    // This compiles, demonstrating that we can create `dyn Iterable`
    // (i.e. the trait is object safe and `v` can be coerced)
    let dt: &dyn Iterable<Borrow = String> = &v;

    // But this gives an error as `&dyn Iterable` doesn't meet the trait
    // bound, and thus `dyn Iterable` does not implement `Iterable`!
    for item in dt.iter() {
        println!("{item}");
    }
}
```

With this particular example, [it's possible to provide an implementation such that
`dyn Iterable` meets the bounds.](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=7b897bcb453c6d8b7b8e3461f70db7a6)
If that's not possible, you probably need to drop the bound or give up
on the trait being `dyn`-safe.
