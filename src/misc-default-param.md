# Default parameter mechanics

Default parameters in Rust are not as convenient as one might wish.
[The RFC for default type parameters](https://rust-lang.github.io/rfcs/0213-defaulted-type-params.html)
was never fully completed; in particular, the "inference falls back to defaults"
parts have been delayed indefinitely.  As a result, there are times where default
parameters don't kick in, and you have to either be explicit or use other
workarounds.  It can also be unclear why the workarounds act differently.

[Default parameters are also not in the reference yet.](https://github.com/rust-lang/reference/issues/24)

This page exists to explain the mechanics behind default parameters as they
exist today, and to clear up exactly what the workarounds mean.  For an exploration
on how the interaction between inference and default parameters could be defined
in the future, I recommend
[this wonderful blog post by Gankra.](https://faultlore.com/blah/defaults-affect-inference/)

## Motivation

The most likely reason you'll run into default parameters not "working" is because
some expression desugars to replacing all type (and `const`) parameters with inference
variables, in combination with the fact that inference variables do not fall back to the
defaults.

What are inference variables?  For types, an inference variable is the same as the
"wildcard type" `_`, which tells the compiler to infer the type for you.
[`_` cannot be used for `const` parameters as of yet,](https://github.com/rust-lang/rust/issues/85077)
but they can still be inferred implicitly.

(For most of this guide, we'll be focused on types;
[there's a subsection about `const` parameters specifically later.](#const-parameters))

Let's see some examples of compilation failures involving defaulted parameters:
```rust,compile_fail
# use std::collections::HashSet;
// `HashSet` will be our running example for a type with both required
// (non-defaulted, non-lifetime) and defaulted parameters
// struct HashSet<Key, S = RandomState> { .. }

// The `insert` is enough for the compiler to infer the `Key` parameter, but
// not the `S` parameter
let mut hs = HashSet::default();
hs.insert(String::new());

// This means the same thing: *all* type (and const) parameters became
// inference variables
let mut hs = HashSet::<_, _>::default();
hs.insert(String::new());
```
This can be confusing because similar code just works:
```rust
# use std::collections::HashSet;
// This compiles, but the compiler can figure `Key` out on its own, so why?
let mut hs = HashSet::<String>::default();
hs.insert(String::new());

// And in fact... this compiles too!
let mut hs = HashSet::<_>::default();
hs.insert(String::new());

// `new` doesn't have this problem, which may also be confusing
let mut hs = HashSet::new();
hs.insert(String::new());
```
The errors can also arise when the type has defaults for of all the type (and `const`) parameters:
```rust,compile_fail
// This will be our running example for a type where all non-lifetime
// parameters have defaults
pub enum Foo<T = String> {
    Bar(T),
    Baz,
}

// This fails because the elided parameter desugars to an inference variable
let foo = Foo::Baz;

// So this means the exact same thing
let foo = Foo::<_>::Baz;
```
And some of the workarounds may be even more confusing:
```rust
#pub enum Foo<T = String> { Bar(T), Baz }
// This works!
let foo = <Foo>::Baz;
```

We want to explain exactly which expressions end up being problematic, and why
the workarounds solve the problem.

## The explanations in brief

First let's tackle why just wrapping the type in `<>` worked for that last example.
```rust
#pub enum Foo<T = String> { Bar(T), Baz }
let foo = <Foo>::Baz;
```
The leading `<Foo>::` notation is called a
"[qualified path type](#more-about-qualified-path-expressions)".
And the short answer to why it works is that, with respect to elided default
parameters, types in `<>`s act the same as type ascription:
```rust
#pub enum Foo<T = String> { Bar(T), Baz }
// Also works
let foo: Foo = Foo::Baz;
```
Type ascription uses default parameters in a way that's probably closer to your intuition.
([We explore the details below.](#type-position-mechanics-in-more-detail))
Note that types act like type ascription in `<>` elsewhere too, such as
within a turbofish, not just as a qualified path type.

As for the difference here:
```rust
#use std::collections::HashSet;
// This fails if we change `HashSet::new()` to `HashSet::default()`
let mut hs = HashSet::new();
hs.insert(String::new());
```
The example only works because [HashSet::new](https://doc.rust-lang.org/std/collections/struct.HashSet.html#implementations)
(and a number of other methods) is only defined for `HashSet<_, RandomState>`.
In contrast, `Default` is implemented for all possible `HashSet<_, _>`.  So
in a sense, this is a workaround on the side of the `HashSet` implementation!
[If inference and default parameters worked together,](https://faultlore.com/blah/defaults-affect-inference/)
`new` would presumably be defined for all possible hashers, too.

Finally, let's look at this workaround:
```rust
#use std::collections::HashSet;
// Remember, `HashSet::default()` fails
let mut hs = HashSet::<_>::default();
hs.insert(String::new());
```
The key difference here is that *if no required parameters are specified,*
then *all* the type (and `const`) parameters -- *including defaulted parameters*
-- are filled in with inference variables.  But if one or more non-lifetime
parameter is specified, it desugars to a qualified type path -- where default
parameters act the same as they do in type ascription.
```rust,compile_fail
#use std::collections::HashSet;
// These are all the same and fail
// let mut hs = HashSet::default();
// let mut hs = HashSet::<>::default();
// let mut hs = HashSet::<_, _>::default();
let mut hs = <HashSet::<_, _>>::default();
hs.insert(String::new());
```
```rust
#use std::collections::HashSet;
// These are the same and succeed.
// let mut hs = HashSet::<_>::default();
let mut hs = <HashSet<_>>::default();
hs.insert(String::new());
```

As is clear from the example, using `_` explicitly counts as specifying a type
parameter.  Also note that the desugaring to "all parameters are inference
variables" only happens when the type is not inside `<>`s.

## Type position mechanics in more detail

By "type position", we mean contexts where the language expects a type specifically.
This includes variable type ascription, implementation headers, type parameter
fields themselves, and qualified path types.

In type position, you can only elide default parameters.  Elided default parameters are
replaced by their default types (or `const` values) specifically (i.e. not inference variables).

Let's see some examples:
```rust
#use std::collections::HashSet;
#use std::hash::RandomState;
# enum Foo<T = String> { Bar(T), Baz, }
// These ascriptions mean the same thing
//     vvvvvvvvvvv
let e: Foo         = Foo::Baz;
let e: Foo<>       = Foo::Baz;
let e: Foo<String> = Foo::Baz;

// These ascriptions mean the same thing
//      vvvvvvvvvvvvvvvvvvvvvvvvvvvv
let hs: HashSet<String>              = Default::default();
let hs: HashSet<String, RandomState> = Default::default();
```
The following errors demonstrate that elided parameters aren't inference variables,
and that inference variables don't fall back to the defaults.
```rust,compile_fail
# enum Foo<T = String> { Bar(T), Baz, }
// Fails due to ambiguity
let e: Foo<_> = Foo::Baz;
```
```rust,compile_fail
#use std::collections::HashSet;
#use std::hash::RandomState;
// Fails due to ambiguity
let hs: HashSet<String, _> = Default::default();
```
```rust,compile_fail
# enum Foo<T = String> { Bar(T), Baz, }
// Fails because the elided type is exactly the default type (`String`)
let e: Foo = Foo::Bar(0);
```

The final example is the opposite situation from most of the examples we've seen:
it's a case where you want inference to override defaults.  If you made the ascription
`Foo<_>` it will compile (but a more trivial fix for this particular example is to
just remove the redundant ascription).

## More about qualified path expressions

Types inside of `<>`s are in type position, and that includes qualified path expressions.

A [qualified path expressions](https://doc.rust-lang.org/reference/paths.html#qualified-paths)
is when you have some path expression that starts with a segment contained in
`<>`.  They were defined in [RFC 0132](https://rust-lang.github.io/rfcs/0132-ufcs.html)
and they come in two different forms:
```rust
//      vvvvvvvv `<T>` where `T` is a type
let s = <String>::default();

//      vvvvvvvvvvvvvvvvvvv `<T as Tr>` where `T` is a type and `Tr` is a trait
let s = <String as Default>::default();
```
The first form can resolve to inherent functions or trait methods, whereas
the second form can only resolve to the named trait's methods.  Rust doesn't
have "trait inference variables", so the trait must be named; you can't use
`_` in place of the trait, for example.  (You can still use it in place of
the trait's type parameters.)

## Traits

Default parameters for traits work the same as default parameters for types,
both inside and outside of "type position".  When thinking of traits in paths
as sugar for qualified paths, the desugaring is like so:
```rust,compile_fail
trait Trait<One, Two = String>: Sized {
    fn foo(self) -> (Self, One, Two) where One: Default, Two: Default {
        (self, One::default(), Two::default())
    }
}

impl<T, U> Trait<T, U> for i32 {}
impl<T, U> Trait<T, U> for f64 {}

// Failing versions
//let _: (i32, (), _) = Trait::foo(0);
//let _: (i32, (), _) = Trait::<_, _>::foo(0);
let _: (i32, (), _) = <_ as Trait<_, _>>::foo(0);
//                    ^^^^^^^^^^^^^^^^^^
```
```rust
#trait Trait<One, Two = String>: Sized {
#    fn foo(self) -> (Self, One, Two) where One: Default, Two: Default {
#        (self, One::default(), Two::default())
#    }
#}
#impl<T, U> Trait<T, U> for i32 {}
#impl<T, U> Trait<T, U> for f64 {}
// Working versions
let _: (i32, (), _) = Trait::<_>::foo(0);
let _: (i32, (), _) = <_ as Trait<_>>::foo(0);
//                    ^^^^^^^^^^^^^^^
```
The only new thing of note is that the implementing type is an inference variable
in this case.

### Mostly historical side note

Before edition 2021, it's possible to leave the `dyn` off of `dyn Trait` types
(although it does fire a lint).  This means that the same name can refer to either
a trait, or a type (the trait object type).  Which one is used depends on the
context.

For example:
```rust,ignore
let _: i32 = Trait::name(0.0);

// If `Trait` has a method called `name`, that is is
let _: i32 = <_ as Trait>::name(0.0);

// But if it does not, and `dyn Trait` has a method called `name`, this is
let _: i32 = <dyn Trait>::name(0.0)

// And the following line is always referring to `dyn Trait`
let _: i32 = <Trait>::name(0.0);
```

## More about types in expressions

In this section, "types in expressions" refers to types which are in expressions
but not within `<>` (e.g. not a qualified path type or a type parameter).  These
are the positions where it is *required* to use turbofish (e.g. `Vec::<String>`)
instead of just appending the parameter list (e.g. `Vec<String>`).

In these positions, it is always allowed to elide all the type and `const`
parameters, even if there are required (i.e. non-defaulted, non-lifetime)
parameters.  When you do so -- even if all the type and `const` parameters
have defaults -- the behavior is the same as using type inference variables
(`_`) for *all* the parameters.

If you do not elide all non-lifetime parameters -- that is, if you specify one or
more type parameter or `const` parameter -- then you must specify all required
parameters. Or in other words: if you specify at least one type or `const`
parameter, you can only elide defaulted parameters (and lifetimes).

The behavior of elided defaulted parameters is as follows:
- If you specify zero non-lifetime parameters
  - Inference variables are used for *all* type and `const` parameters
- If you specify one or more non-lifetime parameters
  - Defaults are used for elided type and `const` parameters

Above, we phrased the different default parameter behavior for types in expressions in
terms of desugaring to [qualified type paths.](#more-about-qualified-path-expressions)
However, the behavior applies in other contexts too, such as `struct` expression syntax:
```rust,compile_fail
struct Two<T, U = String> { t: T, u: U }

// This is ambiguous
let _ = Two { t: (), u: Default::default() };
```
```rust
# struct Two<T, U = String> { t: T, u: U }
// But this works
let _ = Two::<_> { t: (), u: Default::default() };
```
Qualified path types are not allowed in this postion, so not all of the
workarounds we discussed for paths are applicable.
```rust,compile_fail
struct One<T = String> { t: T }

// Ambiguous
let _ = One { t: Default::default() };
```
```rust,compile_fail
#struct One<T = String> { t: T }
// Not accepted grammatically
let _ = <One> { t: Default::default() };
```

Finally, there is no way to syntactically represent inferred but
non-defaulted `const` parameters in qualified path types (or any
other type-annotation-like position).
```rust
struct Pixel<const N: usize>([u8; N]);
impl<const N: usize> Default for Pixel<N> {
    fn default() -> Self {
        Self([0; N])
    }
}

// Works
let pixel = Pixel::default();

// These fail because `_` cannot be used for const parameters yet
// let pixel = <Pixel<_>>::default();
// let pixel: Pixel<_> = Default::default();

drive_inference(pixel);
fn drive_inference(_: Pixel<3>) {}
```

## Non-type generic parameters

This guide has mostly concentrated on type parameters.  We've tried to
be careful with our wording throughout the guide, but let's take a moment
to talk specifically at how non-type parameters work with regards to
defaults.

### Lifetime parameters

Lifetime paramters can not be given defaults, and do not change any of the
default parameter behavior we've discussed.

I've tried to take care to use phrases like "specify one or mmore non-lifetime
parameter" instead of phrase like "empty parameter list".  But just to make
things more explicit: the inclusion or elision of lifetime parameters doesn't
change how parameter defaults work.

For example, the below are still cases of specifying no required parameters,
and thus uses an inference variable (which then fails as ambiguous).
```rust,compile_fail
pub enum Foo2<'a, T = String> {
    Bar(&'a T),
    Baz,
}

let foo = Foo2::<'_>::Bar;
let foo = Foo2::<'static>::Bar;
```

### `const` parameters

[`const` parameters defaults were stabilized in 1.59,](https://github.com/rust-lang/rust/pull/90207)
along with the ability to intermix `const` and type parameters
in the parameter list (as otherwise the presence of a defaulted
type parameter would force all `const` parameters to also have
defaults, for example).

Generally speaking, defaulted `const` parameters act just like defaulted
type parameters.  However, one important difference is that
[`_` cannot be used for `const` parameters as of yet.](https://github.com/rust-lang/rust/issues/85077)

This does mean that some of the workarounds we've seen cannot be applied:
```rust,compile_fail
// We'll use this analogously to our `HashSet` examples
struct MyArray<const N: usize, T = String>([T; N]);

impl<T: Default, const N: usize> Default for MyArray<N, T> {
    fn default() -> Self {
        Self(std::array::from_fn(|_| T::default()))
    }
}

// Ambiguous for the usual reasons
// let arr = MyArray::default();

// Here's what we did when `HashSet` had this problem.
// But it fails because we can't use `_` for the `const` parameter!
let arr = MyArray::<_>::default();

// Explicitness it is then
// let arr = MyArray::<16>::default();

// (These parts are just here to make everything above work like
// our `HashSet` examples worked.)
drive_inference_of_length(&arr);
fn drive_inference_of_length<T>(_: &MyArray<16, T>) {}
```

## A warning about implementations and function arguments

Default type parameters work the same in implementation headers and
function argument lists as they do in other
"[type positions](#type-position-mechanics-in-more-detail)".  This
may be surprising with compared to elided lifetime parameters.

In implementations and function argument lists, eliding a lifetime
parameter introduces a new, independent generic lifetime parameter.
But eliding a type parameter *never* means "introduce a new generic".
Elided type parameters always resolve to a single type (or error),
whether that type comes from inference or a default type.

```rust,compile_fail
pub enum Foo<T = String> { Bar(T), Baz }

// This is an implementation for `Foo<String>` only
impl Foo {
    fn papers_please(&self) {}
}

// This is an implemenetation for all (`Sized`) `T`
impl<T> Foo<T> {
    fn welcome(&self) {}
}

let foo = Foo::Bar(0);

// Works
foo.welcome();

// Fails
foo.papers_please();
```

(Eliding lifetimes in other positions sometimes means `'static` and
sometimes means "infer this for me", but that's a topic for another
day.  Lifetime *parameters* cannot have defaults.)

## Default type parameters elsewhere

Declaring default parameters that are not on types, traits, or trait
aliases either results in an error, or fires a deny-by-default lint stating that
[support will be removed.](https://github.com/rust-lang/rust/issues/36887)

Despite the lint, default parameters on functions work the same as default
parameters on type declarations.  However, every function has a unique type
(a "function item type") which cannot be named.  Because the function item
type cannot be named, most of the workarounds we've talked about cannot be
applied.

That being said, the case where you use a turbofish with one or more
non-elided type parameter still works:
```rust
#[allow(invalid_type_param_default)]
fn example<X, Y: Default = String>() -> Y {
    Y::default()
}

let s = example::<()>();
println!("{}", std::any::type_name_of_val(&s));
```

Default parameters on `impl` headers do not serve any purpose as far as I'm
aware.  Implementations don't have names at all (which is why the parameters
are on the `impl` keyword).
```rust
#struct MyStruct;
#trait WhyThough<T, U> { }
#[allow(invalid_type_param_default)]
impl<T = String> WhyThough<i32, T> for MyStruct {}
```

Default parameters on GATs are currently just denied, even if the lint is allowed.
```rust,compile_fail
#![allow(invalid_type_param_default)]
trait MyTrait {
    type Gat<T = String>;
}
```
