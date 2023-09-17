# `dyn Trait` vs. alternatives

When getting familiar with Rust, it can be hard at first to
recognize when you should use `dyn Trait` versus some other
type mechanism, such as `impl Trait` or generics.

In this section we look at some tradeoffs, depending on the
use case.

## Generic functions and argument position `impl Trait`

### Preliminaries: What is argument position `impl Trait`?

When we talk about argument position `impl Trait`, aka APIT,
we're talking about functions such as this:
```rust
# use std::fmt::Display;
fn foo(d: impl Display) { println!("{d}"); }
// APIT:  ^^^^^^^^^^^^
```

That is, `impl Trait` as an argument type of a function.

APIT is, so far at least, mostly the same as a generic parameter:
```rust
# use std::fmt::Display;
fn foo<D: Display>(d: D) { println!("{d}"); }
```

The main difference is that generics allow
- the function writer to refer to `D`
  - e.g. `D::to_string(&d)`
- other utilizers to turbofish the function
  - e.g. `let function_pointer = foo::<String>;`

Whereas the `impl Display` parameter is not nameable inside nor
outside the function.

There may be more differences in the future, but for now at least,
generics are the more flexible and thus superior form -- unless you
have a burning hatred against the `<...>` syntax, anyway.

At any rate, comparing `dyn Trait` against APIT is essentially the
same as comparing `dyn Trait` against a function with a generic type
parameter.

### Tradeoffs between generic functions and `dyn Trait`

Here, we're talking about choosing between signatures like so:
```rust
# trait Trait {}
// Owned or borrowed generics
fn foo1<T: Trait>(t: T) {}
fn bar1<T: Trait + ?Sized>(t: &T) {}

// Owned or borrowed `dyn Trait`
fn foo2(t: Box<dyn Trait + '_>) {}
fn bar2(t: &dyn Trait) {}
```

When a function has a generic parameter, the parameter is *monomorphized*
for every concrete type which is used to call the function (after lifetime
erasure).  That is, every type the parameter takes on results in a distinct
function in the compiled code.  (Some of the resulting functions may be
eliminated or combined by optimization if possible).  There could be many
copies of `foo1` and `bar1`, depending on how it's called.

But (after lifetime erasure), `dyn Trait` is a singular concrete type.
There will only be one copy of `foo2` and `bar2`.

Yet in your typical Rust program, generic arguments are preferred over
`dyn Trait` arguments.  Why is that?  There are a number of reasons:
- Each monomorphized function can typically be optimized better
- Trait bounds are more general than `dyn Trait`
  - No `dyn` safety concerns (`T: Clone` is possible)
  - No single trait restriction (`T: Trait1 + Trait2` is allowed)
- Less indirection through dynamic dispatch
- No need for boxing in the owned case
  - `Box` isn't even available in `#![no_std]` programs

The `dyn Trait` versions do have the following advantages:
- Smaller code size
- Faster code generation
- [Do not make traits `dyn`-unsafe](./dyn-trait-erased.md)

In general, you should prefer generics unless you have a specific
reason to opt for `dyn Trait` in argument position.

## Return position `impl Trait` and TAIT

### Preliminaries: What are return position `impl Trait` and TAIT?

When we talk about return position `impl Trait`, aka RPIT, we're talking
about functions such as this:
```rust
// RPIT:                vvvvvvvvvvvvvvvvvvvvvvv
fn foo<T>(v: Vec<T>) -> impl Iterator<Item = T> {
    v.into_iter().inspect(|t| println!("{t:p}"))
}
```

Unlike APIT, RPITs are not the same as a generic type parameter.  They
are instead opaque type aliases or opaque type alias constructors.  In
the above example, the RPIT is an opaque type alias constructor which
depends on the input type parameter of the function (`T`).  For every
concrete `T`, the RPIT is also an alias of a ***singular*** concrete
type.

The function body and the compiler still know what the concrete type is,
but that is opaque to the caller and other code.  Instead, the only ways
you can use the type are those which are compatible with the trait or
traits in the `impl Trait`, plus any auto traits which the concrete type
happens to implement.

The singular part is key: the following code does not compile because
it is trying to return two distinct types.  Rust is strictly and
statically typed, so this is not possible -- the opacity of the RPIT
does not and cannot change that.
```rust,compile_fail
# use std::fmt::Display;
fn foo(b: bool) -> impl Display {
    if b { 0 } else { "hi!" }
}
```

`type` alias `impl Trait`, or TAIT, is a generalization of RPIT which
[is not yet stable,](https://github.com/rust-lang/rust/issues/63063)
but may become stable before *too* much longer.  TAIT allows one to
define aliases for opaque types, which allows them to be named and
to be used in more than one location.
```rust,nightly
#![feature(type_alias_impl_trait)]
type MyDisplay = impl std::fmt::Display;

fn foo() -> MyDisplay { "hello," }
fn bar() -> MyDisplay { " world" }
```

Notionally (and hopefully literally), RPIT desugars to a TAIT in
a manner similar to this:
```rust,nightly
#![feature(type_alias_impl_trait)]
# use std::fmt::Display;
fn foo1() -> impl Display { "hi" }

// Same thing... or so
type __Unnameable_Tait = impl Display;
fn foo2() -> __Unnameable_Tait { "hi" }
```

TAITs must still be an alias of a singular, concrete type.

### Tradeoffs between RPIT and `dyn Trait`

RPITs and `dyn Trait` returns share some benefits for the function writer:
- So long as the bounds don't change, you can change the concrete or base type
- You can return unnameable types, such as closures
- It simplifies complicated types, such as long iterator combinator chains

`dyn Trait` does have some limitations and downsides:
- Only one non-auto-trait is supportable without [subtrait boilerplate](./dyn-trait-combining.md)
  - In contrast, you can return `impl Trait1 + Trait2`
- Only `dyn`-safe traits are supportable
  - In contrast, you can return `impl Clone`
- Boxing is required to returned owned types
- You pay the typical optimization penalties of not knowing the base type and performing dynamic dispatch

However, RPITs also have their downsides:
- As an opaque alias, you can only return one actual, concrete type
- For now, the return type is unnameable, which can be awkward for consumers
  - e.g. you can't store the result as a field in your struct
  - ...unless the opaque type bounds are `dyn`-safe and you can type erase it yourself
- Auto-traits are leaky, so it's easy for the function writer to accidentally break semver
  - Whereas auto-traits are explicit with `dyn Trait`
- RPIT is not yet supported in traits
  - It is planned ("RPITIT" (ugh)), but it is uncertain how far away the functionality is

RPITs also have some rather tricky behavior around type parameter and lifetime capture.
The planned `impl Trait` functionalities deserve their own exploration independent of
`dyn Trait`, so I'll only mention them in brief:
- RPIT captures all type parameters ([and their implied lifetimes](https://github.com/rust-lang/rust/issues/42940))
- RPIT captures specific lifetimes and not [the intersection of all lifetimes](./dyn-trait-lifetime.md#when-multiple-lifetimes-are-involved)
  - And thus it is [tedious to capture an intersection of input lifetimes](https://github.com/danielhenrymantilla/fix_hidden_lifetime_bug.rs) instead of a union

Despite all these downsides, I would say that RPIT has a *slight* edge over `dyn Trait`
in return position *when applicable,* especially for owned types.  The advantage between
`dyn Trait` and a (named) TAIT will be even greater, once that is available:
- You can give the return type a name
- You can be explicit about what type parameters are captured
- You can be more explicit about lifetime capture as well
  - Though without a syntax for lifetime intersection, it will probably still be a pain to do so

But `dyn Trait` is still sometimes the better option:
- where RPIT is not supported, like in traits (so far)
- when you need to type erase and return distinct types

However, there is often a third possibility available, which we explore below:
return a generic struct.

### An alternative to both: nominal generic structs

Here we can take inspiration from the standard library.  One of the more popular
situations to use RPIT or return `dyn Trait` is when dealing with iterators
(as iterator chains have long types and often involve unnameable types such as
closures as well).
[So let's look at the Iterator methods.](https://doc.rust-lang.org/std/iter/trait.Iterator.html)

You may notice a pattern with the combinators:
```rust,ignore
fn chain<U>(self, other: U) -> Chain<Self, <U as IntoIterator>::IntoIter>
where
    Self: Sized,
    U: IntoIterator<Item = Self::Item>,
{ todo!() }

fn filter<P>(self, predicate: P) -> Filter<Self, P>
where
    Self: Sized,
    P: FnMut(&Self::Item) -> bool,
{ todo!() }

fn map<B, F>(self, f: F) -> Map<Self, F>
where
    Self: Sized,
    F: FnMut(Self::Item) -> B,
{ todo!() }
```
The pattern is to have a function which is parameterized by a generic type
return a concrete (nominal) struct, also parameterized by the generic type.
This is possible even if the parameter itself is unnameable -- for example,
in the case of `map`, the `F: FnMut(Self::Item) -> B` parameter might well
be an unnameable closure.

The downside is much more boilerplate if you opt to follow this pattern
yourself: You have to define the struct, and (for examples like these)
implement the `Iterator` trait for them, and perhaps other traits such
as `DoubleEndedIterator` as desired.  This will probably involve storing
the original iterator and calling `next().map(|item| ...)` on it, or
such.

The upside is that you (and the consumers of your method) get many of the upsides of both RPIT and `dyn Trait`:
- No dynamic dispatch penalty
- No boxing penalty
- No concrete-type specific optimization loss
- No single trait limitation
- No `dyn`-safe limitation
- Applicable in traits
- Ability to be specific about captures
- Ability to change your implementation within the API bounds
- Nameable return type

You do retain some of the downsides:
- Auto-traits are leaky and still a semver hazard, as with RPIT
- Multiple concrete types aren't possible (without also utilizing type erasure), as with RPIT

And incur some unique ones as well:
- Variance of data types are leaky too
- Unnameable types that aren't input type parameters can't be supported (without also utilizing type erasure)

On the whole, when using a nominal type is possible, it is the best option for
consumers of the function.  But it's also the most amount of work for the function
implementor.

I recommend nominal types for general libraries (i.e. intended for wide consumption)
when possible, following the lead of the standard library.

## Generic structs

In the last section, we covered how generic structs can often be used as an
alternative to RPIT or returning `dyn Trait` in some form.   A related question
is, when should you use type erasure within your data types?

The main reason to use type erasure in your data types are when you want to
treat implementors of a trait as if they were the same type, for instance when
storing a collection of callbacks.  In this case, the decision to use type
erasure is a question of functionality, and not really much of a choice.

However, you may also want to use type erasure in your data types in order
to make your own struct non-generic.  When your data type is generic, after
all, those who use your data type in such a way that the parameter takes on
more than one type will have to propagate the use of generics themselves, or
face the decision of type erasing your data type themselves.

This can not only be a question of ergonomics, but also of compile time and
even run time performance.  Compiling strictly more code by having all your
methods monomorphized will naturally tend to result in longer compile times,
and the increase in actual code size *can sometimes be slower at runtime*
than a touch of dynamic dispatch in the right areas.

Unfortunately, there is no silver bullet when it comes to choosing between
being generic and using type erasure.  However, a general principle is that
your optimization sensitive, call-heavy code areas should not be type erased,
and instead push type erasure to a boundary outside of your heavy computations.

For example, the failure to devirtualize and inline a call to
`<dyn Iterator>::next` in a tight loop may have a relatively large impact,
whereas a dynamic callback that only fires occasionally (and then dispatches
to the optimized, non-type-erased implementation) is not likely to be
noticeable at all.

## `enum`s

Finally we'll mention one other alternative to type erasure:
just put all of the implementing types in an `enum`!

This clearly only applies when you have a fixed set of types that you
expect to implement your trait.  The downside of using an `enum` is that
it can involve a lot of boilerplate, since you're frequently having to check
which variant you are instead of relying on dynamic dispatch to perform
that function for you.

The upside is avoiding practically all of the downsides of type erasure
and the other alternatives such as opaque types.

Macros can help ease the pain of such boilerplate, and there are also
[crates in the ecosystem](https://crates.io/crates/ambassador) aimed at
reducing the boilerplate.

In fact, [there are also crates for this pattern as a whole.](https://crates.io/crates/enum_dispatch)

In particular, if you find yourself in a situation where you've
[chosen to use `dyn Any`](./dyn-any.md) but you find yourself with
a bunch of attempted downcasts against a known set of types, you
should *strongly* consider just using an `enum`.  It won't be much
less ergonomic (if at all) and will be more efficient.
