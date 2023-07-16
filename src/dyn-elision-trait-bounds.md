# Influences from trait lifetime bounds

When the trait itself has lifetime bounds, those bounds may influence the
behavior of `dyn Trait` lifetime elision.  Where and how the influence does
or does not take place is not properly documented, but we'll cover some
cases here.

The way trait object lifetime defaults behave in these scenarios is not
intuitive, and perhaps even arbitrary.  But to be clear, you will probably
never need to actually know the exact rules.  Traits with exotic lifetime
bounds are rare, and should you actually encounter one, you can usually
choose to be explicit instead of trying to figure out what lifetime is
the default when elided.

Which is to say, this subsection is more of an exploration of the
compiler's current behavior than something useful to learn.  If you're
trying to learn practical Rust, you should probably just skip it.

A very high level summary is:
- Trait bounds introduce implied bounds on the trait object lifetimes
- Elision in the presence of non-`'static` trait lifetime bounds is arbitrary, so prefer to be explicit
- Prefer not to add non-`'static` lifetime bounds to your own object safe traits
  - Avoid multiple lifetime bounds in particular

This section is also non-exhaustive.  Given how many exceptions I have ran across,
take my assertive statements in this section with a grain of salt.

## Trait lifetime bounds create an implied bound

The trait bound creates an implied bound on the `dyn Trait` lifetime:
```rust
pub trait LifetimeTrait<'a, 'b>: 'a {}

pub fn f<'b>(_: Box<dyn LifetimeTrait<'_, 'b> + 'b>) {}

fn fp<'a, 'b, 'c>(t: Box<dyn LifetimeTrait<'a, 'b> + 'c>) {
    // This compiles which indicates an implied `'c: 'a` bound
    let c: &'c [()] = &[];
    let _: &'a [()] = c;

    // This does not, demonstrating that `'c: 'b` is not implied
    // (i.e. the implied bound is on the trait object lifetime only, and
    // not on the other parameters.)
    //let _: &'b [()] = c;

    // This does not as it requires `'c: 'b` and `'b: 'a`
    //f(t);
}
```

This is similar to how `&'b &'a _` creates an implied `'a: 'b` bound.
It only applies to the trait object lifetime, and not the entirety
of the `dyn Trait` (e.g. it does not apply to trait parameters).

## The `'static` case

We've already summarized the behavior of trait object lifetime elision when
the trait itself has a `'static` bound as part of our basic guidelines: the
lifetime in this case is always `'static`.

This applies even to
- types with ambiguous (more than one) lifetime bounds
- types with a single lifetime bound like `&_`
  - i.e. the trait object lifetime (which is `'static`) becomes independent of the outer lifetime
- situations where a non-`'static` bound does *not* override the `&_` trait object lifetime default, as in some of the examples further below

This case applies even if there are multiple bounds and only one of them is
`'static`, in contrast with
[bounds considered ambiguous from the struct definition.](./dyn-elision-advanced.md#ambiguous-bounds)

## A single trait lifetime bound does not always apply

According to the reference, the default trait object lifetime for a trait
with a single lifetime bound in the context of a generic struct with no
lifetime bounds is always the lifetime in the trait's bound.

That's a mouthful, but the implication is that here:
```rust
trait Single<'a>: 'a {}
```
The elided lifetime of `Box<dyn Single<'a>>` is always `'a`.

However, this is not actually the case:
```rust
#trait Single<'a>: 'a {}
// The elided lifetime was `'static`, not `'a`, so this compiles
fn foo<'a>(s: Box<dyn Single<'a>>) {
    let s: Box<dyn Single<'a> + 'static> = s;
}
```

```rust,compile_fail
#trait Single<'a>: 'a {}
// In this case it *is* `'a`, so compilation fails
fn bar<'a: 'a>(s: Box<dyn Single<'a>>) {
    let s: Box<dyn Single<'a> + 'static> = s;
}
```

## When they apply, trait lifetime bounds override struct bounds

According to the reference, bounds on the trait never override bounds on
the struct.  But based on my testing, the opposite is true: *when* bounds
on the trait apply, they *always* override the bounds on the struct.

The complicated part is figuring out when they apply.

For example, the following compiles, but according to the reference it
should be ambiguous due to the multiple lifetime bounds on the struct.
It does not compile without the lifetime bound on the trait; the bound
on the trait is overriding the ambiguous bounds on the struct.
```rust
use core::marker::PhantomData;

// Remove `: 'a` to see the compile error
pub trait LifetimeTrait<'a>: 'a {}

pub struct Over<'a, T: 'a + 'static + ?Sized>(&'a T);

pub struct Invariant<T: ?Sized>(*mut PhantomData<T>);
unsafe impl<T: ?Sized> Sync for Invariant<T> {}

pub static OS: Invariant<Over<'_, dyn LifetimeTrait>> = Invariant(std::ptr::null_mut());
```

Further below are some examples where the trait bound overrides the
`&_` bounds as well, so it is not just ambiguous struct bounds which can
be overridden by trait bounds.

## Multiple trait bounds can be ambiguous or can apply

The following is considered ambiguous due to the multiple lifetime bounds
on the trait.
```rust,compile_fail
trait Double<'a, 'b>: 'a + 'b {}
fn f<'a, 'b, T: Double<'a, 'b> + 'static>(t: T) {
    let bx: Box<dyn Double<'a, 'b>> = Box::new(t);

    // This version works:
    let bx: Box<dyn Double<'a, 'b> + 'static> = Box::new(t);
}
```

The current documentation is silent on this point, but a multiple-bound
trait can still apply in such a way that it provides the default trait
object lifetime.
```rust
pub trait Double<'a, 'b>: 'a + 'b {}

fn x1<'a: 'a, 'b>(bx: Box<dyn Double<'a, 'b>>) {
    // This fails (the lifetime is not `'static`)
    //let bx: Box<dyn Double<'a, 'b> + 'static> = bx;

    // This also fails (the lifetime is not `'b` nor `'a + 'b`)
    //let bx: Box<dyn Double<'a, 'b> + 'b> = bx;

    // But this succeeds and we can conclude the lifetime is `'a`
    let bx: Box<dyn Double<'a, 'b> + 'a> = bx;
}
```

There's a subtle point here: the elided trait object lifetime is `'a`,
but there's an implied `: 'a + 'b` bound on the trait object lifetime
due to the trait bounds.  Therefore the function signature has an
implied `'a: 'b` bound, similar to when you have a `&'b &'a _`
argument.

## Trait bounds *always* apply in function bodies

Based on my testing, the default trait object lifetime for annotations
of `dyn Trait` in function bodies is *always* the trait bound.  And in
fact, this bound *even overrides the wildcard `'_` lifetime annotation*.

This is a surprising exception to the `'_` annotation restoring "normal"
lifetime elision behavior.

```rust,compile_fail
trait Single<'a>: 'a {}

fn baz<'long: 'a, 'a, T: 'long + Single<'a>>(s: T) {
    // This compiles with the assignment at the end:
    //let s: Box<dyn Single<'a> + 'long> = Box::new(s);

    // But none of these compile because `'a: 'long` does not hold:
    //let s: Box<dyn Single<'a>> = Box::new(s);
    //let s: Box<dyn Single<'_>> = Box::new(s);
    //let s: Box<dyn Single<'a> + '_> = Box::new(s);
    //let s: Box<dyn Single<'_> + '_> = Box::new(s);
    //let s: Box<dyn Single + '_> = Box::new(s);
    let s: Box<dyn Single> = Box::new(s);

    let s: Box<dyn Single<'_> + 'long> = s;
}
```

## When and how to trait lifetime bounds apply?

Now that we've seen a number of examples, we can theorize when and how
trait lifetime bounds apply.  As the examples have already illustrated,
there are very different rules for different contexts.

### In function signatures

This appears to be the most complex and arbitrary context for trait object lifetime elision.

If you were paying close attention, you may have noticed that we occasionally had
trivial bounds like `'a: 'a` in the examples above, and that affected whether the
trait bounds applied or not.  A lifetime parameter of a function with no
*explicit* bounds is known as a late-bound parameter, and
[whether or not a lifetime is late-bound influences when the trait bounds apply
in function signatures.](https://github.com/rust-lang/rust/issues/47078)
Parameters which are not late-bound are early-bound.

Let us call a lifetime parameter of a trait which is also a bound of the trait
a "bounding parameter".  My hypothesis on the behavior is as follows:
- if any trait bound is `'static`, the default lifetime is `'static`
- if any bounding parameter is explicitly `'static`, the default lifetime is `'static`
- if exactly one bounding parameter is early-bound, the default lifetime is that lifetime
  - including if it is in multiple positions, such as `dyn Double<'a, 'a>`
- if more than one bounding parameter is early-bound, the default lifetime is ambiguous
- if no bounding parameters are early-bound, the default lifetime depends on the `struct`
  bounds (the same as they do for a trait without bounds)

Note that in any case, the implied bounds on the trait object lifetime
that exist due to the trait bounds are still in effect.

The requirement that exactly one of the bounding parameters is early-bound
or that any of them are `'static` are syntactical requirements, rather than
semantic ones.  For example:
```rust,compile_fail
pub trait Double<'a, 'b>: 'a + 'b {}

// Semantically, `'a` and `'b` must be `'static`.  However the
// parameters were not explicitly `'static` and thus this
// trait object lifetime is considered ambiguous (even though,
// due to the implied bounds, it must be `'static` too).
fn foo<'a: 'static, 'b: 'static>(d: Box<dyn Double<'a, 'b>>) {}

// Semantically, `'a` and `'b` must be the same.  They are also
// early-bound parameters due to the bounds.  However the parameters
// are not syntatically the same lifetime and thus this trait
// object lifetime is considered ambiguous.
fn bar<'a: 'b, 'b: 'a>(d: &dyn Double<'a, 'b>) {}
```
But if you change either example to `Double<'a, 'a>`, then
exactly one of the bounding parameters is early-bound, and they
will compile:
```rust
#pub trait Double<'a, 'b>: 'a + 'b {}
fn foo<'a: 'static, 'b: 'static>(d: Box<dyn Double<'a, 'a>>) {}
fn bar<'a: 'b, 'b: 'a>(d: &dyn Double<'a, 'a>) {}
```

#### Implicit bounds do not negate being late-bound

Note that when considering `&dyn Trait` there is always an *implied* bound between the
outer reference's lifetime and the `dyn Trait` (in addition to the implied bound from
the trait itself).  However, these implied bounds are not enough to make the trait
bound apply on their own.  A lifetime can be late-bound even when there are implied bounds.
```rust
pub trait LifetimeTrait<'a>: 'a {}
impl LifetimeTrait<'_> for () {}

// All of these compile with the `fp` function below, indicating that
// the trait bound does in fact apply and results in a trait object
// lifetime independent of the reference lifetime
pub fn f<'a: 'a>(_: &dyn LifetimeTrait<'a>) {}
//pub fn f<'a: 'a>(_: &'_ dyn LifetimeTrait<'a>) {}
//pub fn f<'r, 'a: 'a>(_: &'r dyn LifetimeTrait<'a>) {}
//pub fn f<'r: 'r, 'a: 'a>(_: &'r dyn LifetimeTrait<'a>) {}
//pub fn f<'r, 'a: 'r + 'a>(_: &'r dyn LifetimeTrait<'a>) {}
//pub fn f<'r: 'r, 'a: 'r>(_: &'r dyn LifetimeTrait<'a>) {}

// However none of these compile with `fp`, indicating that the elided trait
// object lifetime is defaulting to the reference lifetime "per normal".
//pub fn f(_: &dyn LifetimeTrait) {}
//pub fn f(_: &'_ dyn LifetimeTrait) {}
//pub fn f<'r>(_: &'r dyn LifetimeTrait) {}
//pub fn f<'r: 'r>(_: &'r dyn LifetimeTrait) {}

//pub fn f(_: &dyn LifetimeTrait<'_>) {}
//pub fn f(_: &'_ dyn LifetimeTrait<'_>) {}
//pub fn f<'r>(_: &'r dyn LifetimeTrait<'_>) {}
//pub fn f<'r: 'r>(_: &'r dyn LifetimeTrait<'_>) {}

//pub fn f<'a>(_: &dyn LifetimeTrait<'a>) {}
//pub fn f<'a>(_: &'_ dyn LifetimeTrait<'a>) {}
//pub fn f<'r, 'a>(_: &'r dyn LifetimeTrait<'a>) {}
//pub fn f<'r: 'r, 'a>(_: &'r dyn LifetimeTrait<'a>) {}

// n.b. `'a` is invariant due to being a trait parameter
fn fp<'a>(t: &(dyn LifetimeTrait<'a> + 'a)) {
    f(t);
}
```

The above examples also demonstrate that when trait bounds apply,
they do override non-ambiguous struct bounds (such as those of `&_`).

#### Implied bounds and default object bounds interact

The interaction between what the default object lifetime is for a given
signature can interact in potentially surprising ways.  Consider this example:
```rust
#pub trait LifetimeTrait<'a>: 'a {}
// The implied bounds in `&'outer (dyn Lifetime<'param> + 'trait)` are:
// - `'param: 'outer` (validity of the reference)
// - `'trait: 'outer` (validity of the reference)
// - `'trait: 'param` (from the trait bound)
//
// And as the trait bound does not apply to the elided parameter in this
// case, we also have `'outer = 'trait` due to the "normal" default
// lifetime behavior of `&_`.  Adding that equality to the above bounds
// results in a requirement that *all three lifetimes are the same*.
//
// And thus this compiles:
pub fn g<'r, 'a>(d: &'r dyn LifetimeTrait<'a>) {
    let r: [&'r (); 1] = [&()];
    let a: [&'a (); 1] = [&()];
    let _: [&'a (); 1] = r;
    let _: [&'r (); 1] = a;
    let _: &'r (dyn LifetimeTrait<'r> + 'r) = d;
    let _: &'a (dyn LifetimeTrait<'a> + 'a) = d;
}
```

The results can be even more surprising with more complex bounds:
```rust
trait Double<'a, 'b>: 'a + 'b {}

fn h<'a, 'b, T>(bx: Box<dyn Double<'a, 'b>>, t: &'a T)
where
    &'a T: Send, // this makes `'a` early-bound
{
    // `bx` is `Box<dyn Double<'a, 'b> + 'a>` as per the rules above,
    // so this does not compile:
    //let _: Box<dyn Double<'a, 'b> + 'static> = bx;

    // However, the implied bounds still apply, which means:
    // - `'a: 'a + 'b`
    // - So `'a: 'b`
    //
    // Which is why this can compile even though that bound
    // is not declared anywhere!
    let t: &'b T = t;

    // The lifetimes are still not the same, so this fails
    let _: &'a T = t;
}
```

The only reason that `'a: 'b` is an implied bound in the above example
is the interaction between
- the implied `: 'a + 'b` bound on the trait object lifetime
- the default trait object lifetime being `'a`
  - due to `'a` being early-bound and `'b` being late-bound

If `'b` was also early-bound, the default trait object lifetime would
be ambiguous.  If `'a` wasn't early-bound, the default trait object
lifetime would be `'static` and there would be no implied `'a: 'b`
bound.

#### The wildcard lifetime still introduces a fresh inference lifetime

Based on my testing, using `'_` will behave like typical lifetime elision,
introducing a fresh inference lifetime in input position, and following
the function signature elision rules in output position.

#### Higher-ranked lifetimes are late-bound

Based on my testing, `for<'a> dyn Trait...` lifetimes act the same as
late-bound lifetimes.

### Function bodies

As mentioned above, trait object bounds always apply in function bodies,
similar to function signatures where every lifetime is early-bound.  This
is true irregardless of whether the lifetimes are early or late bound in
the function signature.
```rust
trait Single<'a>: 'a {}

fn foo<'r, 'a>(bx: Box<dyn Single<'a> + 'static>, rf: &'r (dyn Single<'a> + 'static)) {
    // Here it is `'a`, and not `'static` nor inferred
    let bx: Box<dyn Single<'a>> = bx;
    // So this fails
    //let _: Box<dyn Single<'a> + 'static> = bx;

    // Here it is `'a`, and not the same as the reference lifetime nor inferred
    let a: &dyn Single<'a> = rf;
    // So this succeeds
    let _: &(dyn Single<'a> + 'a) = a;
    // And this fails
    //let _: &(dyn Single<'a> + 'static) = a;

    // Same behavior when the reference lifetime is explicit
    let a: &'r dyn Single<'a> = rf;
    let _: &'r (dyn Single<'a> + 'a) = a;
    //let _: &'r (dyn Single<'a> + 'static) = a;

    // This also fails, demonstrating that `'r` is not `'a`
    //let _: &'a &'r () = &&();
}
```

And unlike elsewhere, using `'_` in place of complete trait object
lifetime elision in the function body does not restore the normal
lifetime elision behavior (which would be inferring the lifetime).
All three of the examples above behave identically if `'_` is used.

```rust
#trait Single<'a>: 'a {}
fn foo<'r, 'a>(bx: Box<dyn Single<'a> + 'static>, rf: &'r (dyn Single<'a> + 'static)) {
    let bx: Box<dyn Single<'a> + '_> = bx;
    // Fails
    //let _: Box<dyn Single<'a> + 'static> = bx;

    let a: &(dyn Single<'a> + '_) = rf;
    let _: &(dyn Single<'a> + 'a) = a;
    // Fails
    //let _: &(dyn Single<'a> + 'static) = a;

    let a: &'r (dyn Single<'a> + '_) = rf;
    let _: &'r (dyn Single<'a> + 'a) = a;
    // Fails
    //let _: &'r (dyn Single<'a> + 'static) = a;
}
```

In combination with the behavior of function signatures, this can
lead to some awkward situations.
```rust,compile_fail
trait Double<'a, 'b>: 'a + 'b {}

// Here in the signature, `'_` acts like "normal" and creates an
// independent lifetime for the trait object lifetime; let us call
// it `'c`.  Though independent, it is related due to the implied
// bounds: `'c: 'a + 'b`
fn foo<'a, 'b>(bx: Box<dyn Double<'a, 'b> + '_>) {
    // Here in the body, the default trait object lifetime is
    // considered ambiguous, and `'_` does not override this.
    //
    // Moreover, there is no way to name `'c` since it was
    // elided in the signature.  We could annotate this as
    // either `'a` or `'b`, but cannot "preserve" the full
    // lifetime unless we change the function signature to
    // give the lifetime a name.
    let bx: Box<dyn Double<'a, 'b> + '_> = bx;
}
```

### Static contexts

In most static contexts, any elided lifetimes (not just trait object
lifetimes) default to the `'static` lifetime.

```rust
#use core::marker::PhantomData;
trait Single<'a>: 'a + Send + Sync {}
trait Halfie<'a, 'b>: 'a + Send + Sync {}
trait Double<'a, 'b>: 'a + 'b + Send + Sync {}

static BS: PhantomData<Box<dyn Single<'_>>> = PhantomData;
static BH: PhantomData<Box<dyn Halfie<'_, '_>>> = PhantomData;
static BD: PhantomData<Box<dyn Double<'_, '_>>> = PhantomData;

static S_BS: PhantomData<Box<dyn Single<'static> + 'static>> = BS;
static S_BH: PhantomData<Box<dyn Halfie<'static, 'static> + 'static>> = BH;
static S_BD: PhantomData<Box<dyn Double<'static, 'static> + 'static>> = BD;

const CS: PhantomData<Box<dyn Single<'_>>> = PhantomData;
const CH: PhantomData<Box<dyn Halfie<'_, '_>>> = PhantomData;
const CD: PhantomData<Box<dyn Double<'_, '_>>> = PhantomData;

const S_CS: PhantomData<Box<dyn Single<'static> + 'static>> = CS;
const S_CH: PhantomData<Box<dyn Halfie<'static, 'static> + 'static>> = CH;
const S_CD: PhantomData<Box<dyn Double<'static, 'static> + 'static>> = CD;
```

In a context where non-`'static` lifetimes can be named, those lifetimes
act like early-bound lifetimes in function signatures.
```rust
#use core::marker::PhantomData;
#trait Single<'a>: 'a + Send + Sync {}
struct L<'l, 'm>(&'l str, &'m str);
impl<'a, 'b> L<'a, 'b> {
    const CS: PhantomData<Box<dyn Single<'a>>> = PhantomData;

    // Fails
    //const S_CS: PhantomData<Box<dyn Single<'a> + 'static>> = Self::CS;
    const S_CS: PhantomData<Box<dyn Single<'a> + 'a>> = Self::CS;
}
```
Elided lifetimes are still inferred to be `'static`...
```rust
#use core::marker::PhantomData;
#trait Single<'a>: 'a + Send + Sync {}
#struct L<'l, 'm>(&'l str, &'m str);
impl<'a, 'b> L<'a, 'b> {
    const ECS: PhantomData<Box<dyn Single<'_>>> = PhantomData;
    const SCS: PhantomData<Box<dyn Single<'static>>> = PhantomData;

    const S_ECS: PhantomData<Box<dyn Single<'static> + 'static>> = Self::ECS;
    const S_SCS: PhantomData<Box<dyn Single<'static> + 'static>> = Self::SCS;
}
```
*...however,* this inference only seems to take effect after the
definition itself for elided trait object lifetimes, as demonstrated
by cases such as this being ambiguous:
```rust
#use core::marker::PhantomData;
#trait Double<'a, 'b>: 'a + 'b + Send + Sync {}
#struct L<'l, 'm>(&'l str, &'m str);
impl<'a, 'b> L<'a, 'b> {
    const EBCD: PhantomData<Box<dyn Double<'a, '_>>> = PhantomData;
}
```
...and cases such this saying that the reference lifetime is longer
than the trait object, even when they have the same anonymous
lifetime:
```rust
#use core::marker::PhantomData;
#trait Single<'a>: 'a + Send + Sync {}
struct R<'l, 'm, 'r>(&'l str, &'m str, &'r ());
impl<'a, 'b, 'r> R<'a, 'b, 'r> where 'a: 'r, 'b: 'r {
    const ECS: PhantomData<&dyn Single<'_>> = PhantomData;
    const RECS: PhantomData<&'r dyn Single<'_>> = PhantomData;
}
```

### `impl` headers

Trait bounds always apply in `impl` headers.

```rust
trait Single<'a>: 'a {}
trait Halfie<'a, 'b>: 'a {}
trait Double<'a, 'b>: 'a + 'b {}

struct S<T>(T);

// The trait bounds apply
impl<'a> S<Box<dyn Single<'a>>> { fn f01() {} } // 'a (not 'static)
impl<'a, 'r> S<&'r dyn Single<'a>> { fn f02() {} } // 'a (not 'r)
impl<'a, 'b> S<Box<dyn Halfie<'a, 'b>>> { fn f03() {} } // 'a (not 'static)
impl<'a, 'b, 'r> S<&'r dyn Halfie<'a, 'b>> { fn f04() {} } // 'a (not 'r)
// Ambiguous (uncomment for error)
// impl<'a, 'b> S<Box<dyn Double<'a, 'b>>> { fn f05() {} }
// impl<'a, 'b, 'r> S<&'r dyn Double<'a, 'b>> { fn f05() {} }

// Try `+ 'static` or `+ 'r` for errors
fn f<'a, 'b, 'r>(_: &'r &'a str, _: &'r &'b str) {
    S::<Box<dyn Single<'a> + 'a>>::f01();
    S::<&'r (dyn Single<'a> + 'a)>::f02();
    S::<Box<dyn Halfie<'a, 'b> + 'a>>::f03();
    S::<&'r (dyn Halfie<'a, 'b> + 'a)>::f04();
}
```

As in function signatures, but unlike function bodies, the wildcard lifetime
`'_` acts like normal elision (introducing a new anonymous lifetime variable).
```rust
#trait Single<'a>: 'a {}
#trait Halfie<'a, 'b>: 'a {}
#trait Double<'a, 'b>: 'a + 'b {}
#struct S<T>(T);
// The wildcard lifetime `'_` introduces an independent lifetime
// (covering all cases including `'static`) as per normal
impl<'a> S<Box<dyn Single<'a> + '_>> { fn f26() {} }
impl<'a, 'r> S<&'r (dyn Single<'a> + '_)> { fn f27() {} }
impl<'a, 'b> S<Box<dyn Halfie<'a, 'b> + '_>> { fn f28() {} }
impl<'a, 'b, 'r> S<&'r (dyn Halfie<'a, 'b> + '_)> { fn f29() {} }
impl<'a, 'b> S<Box<dyn Double<'a, 'b> + '_>> { fn f30() {} }
impl<'a, 'b, 'r> S<&'r (dyn Double<'a, 'b> + '_)> { fn f31() {} }

fn f<'a, 'b, 'r>(_: &'r &'a str, _: &'r &'b str) {
    S::<Box<dyn Single<'a> + 'static>>::f26();
    S::<&'r (dyn Single<'a> + 'static)>::f27();
    S::<Box<dyn Halfie<'a, 'b> + 'static>>::f28();
    S::<&'r (dyn Halfie<'a, 'b> + 'static)>::f29();
    S::<Box<dyn Double<'a, 'b> + 'static>>::f30();
    S::<&'r (dyn Double<'a, 'b> + 'static)>::f31();
}
```

### Associated types

Similar to `impl` headers, trait bounds always apply to associated types.
```rust
#use core::marker::PhantomData;
trait Single<'a>: 'a {}
trait Halfie<'a, 'b>: 'a {}
trait Double<'a, 'b>: 'a + 'b {}

trait Assoc {
    type A01: ?Sized + Default;
    type A02: ?Sized + Default;
    //type A03: ?Sized + Default;
    type A04: ?Sized + Default;
    type A05: ?Sized + Default;
    //type A06: ?Sized + Default;
}

impl<'r, 'a, 'b> Assoc for (&'r &'a (), &'r &'b ()) {
    // '_ is not allowed here
    // & /* elided */ is not allowed here
    type A01 = PhantomData<Box<dyn Single<'a>>>;
    type A02 = PhantomData<Box<dyn Halfie<'a, 'b>>>;
    // ambiguous
    // type A03 = PhantomData<Box<dyn Double<'a, 'b>>>;
    type A04 = PhantomData<&'r dyn Single<'a>>;
    type A05 = PhantomData<&'r dyn Halfie<'a, 'b>>;
    // ambiguous
    // type A06 = PhantomData<&'r dyn Double<'a, 'b>>;
}

fn f<'r, 'a: 'r, 'b: 'r>() {
    // 'a (not `'static`, `'r`, `'b`)
    let _: PhantomData<Box<dyn Single<'a> + 'a>> = <(&'r &'a (), &'r &'b ()) as Assoc>::A01::default();
    let _: PhantomData<Box<dyn Halfie<'a, 'b> + 'a>> = <(&'r &'a (), &'r &'b ()) as Assoc>::A02::default();
    let _: PhantomData<&'r (dyn Single<'a> + 'a)> = <(&'r &'a (), &'r &'b ()) as Assoc>::A04::default();
    let _: PhantomData<&'r (dyn Halfie<'a, 'b> + 'a)> = <(&'r &'a (), &'r &'b ()) as Assoc>::A05::default();
}
```

Note: I have not performed extensive tests with GATs or associated types which themselves
have lifetime bounds in combination with bounded traits.
