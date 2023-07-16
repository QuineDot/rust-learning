# Basic guidelines and subtleties

As a reminder, `dyn Trait` is a type constructor which is parameterized with a
lifetime; a fully resolved type includes the lifetime, such as `dyn Trait + 'static`.
The lifetime can be elided in many sitations, in which case the actual lifetime
used may take on some default lifetime, or may be inferred.

When talking about default trait object (`dyn Trait`) lifetimes, we're talking about
situations where the lifetime has been completely elided.  If the wildcard lifetime
is used (`dyn Trait + '_`), then [the normal lifetime elision rules](https://doc.rust-lang.org/reference/lifetime-elision.html#lifetime-elision-in-functions)
usually apply instead.  (The exceptions are rare, and you can usually be explicit
instead if you need to.)

For a completely elided `dyn Trait` lifetime, you can start with these
general guidelines for traits with no lifetime bounds (which are the vast majority):
- In function bodies, the trait object lifetime is inferred (i.e. ignore the following bullets)
- For references like `&'a dyn Trait`, the default is the same as the reference lifetime (`'a`)
- For `dyn`-supporting `std` types with lifetime parameters such as
  [`Ref<'a, T>`](https://doc.rust-lang.org/std/cell/struct.Ref.html), it is also `'a`
- For non-lifetime-parameter types like `Box<dyn Trait>`, and for bare `dyn Trait`, it's `'static`

And for the (rare) trait with lifetime bounds:
- If the trait has a `'static` bound, the trait object lifetime is always `'static`
- If the trait has only non-`'static` lifetime bounds, [you're better off being explicit](./dyn-elision-trait-bounds.md)


This is a close enough approximation to let you understand `dyn Trait`
lifetime elision most of the time, but there are exceptions to these
guidelines (which are explored on the next couple of pages).

There are also a few subtleties worth pointing out *within* these guidelines,
which are covered immediately below.

## Default `'static` bound gotchas

The most likely scenario to run into an error about `dyn Trait` lifetime is
when `Box` or similar is involved, resulting an implicit `'static` constraint.

Those errors can often be addressed by either adding an explicit `'static`
bound, or by overriding the implicit `'static` lifetime.  In particular, using
`'_` will usually result in the "normal" (non-`dyn Trait`) lifetime elision for
the given context.

```rust
trait Trait {}
impl<T: Trait> Trait for &T {}

// Remove `+ 'static` to see an error
fn with_explicit_bound<'a, T: Trait + 'static> (t: T) -> Box<dyn Trait> {
    Box::new(t)
}

// Remove `+ 'a` (in either position) to see an error
fn with_nonstatic_box<'a, T: Trait + 'a>(t: T) -> Box<dyn Trait + 'a> {
    Box::new(t)
}

// Remove `+ '_` to see an error
fn with_fn_lifetime_elision(t: &impl Trait) -> Box<dyn Trait + '_> {
    Box::new(t)
}
```

This can be particularly confusing within a function body, where a
`Box<dyn Trait>` variable annotation acts differently from a `Box<dyn Trait>`
function input parameter annotation:
```rust
# trait Trait {}
# impl Trait for &i32 {}
// In this context, the elided lifetime is `'static`
fn requires_static(_: Box<dyn Trait>) {}

fn example() {
    let local = 0;

    // In this context, the annotation means `Box<dyn Trait + '_>`!
    // That is why it can compile on it's own, with the local reference.
    let bx: Box<dyn Trait> = Box::new(&local);

    // So despite using the same syntax, this call cannot compile.
    // Uncomment it to see the compilation error.
    // requires_static(bx);
}
```

## `impl` headers

The `dyn Trait` lifetime elision applies in `impl` headers, which can lead to
implementations being less general than possible or desired:
```rust
trait Trait {}
trait Two {}

impl Two for Box<dyn Trait> {}
impl Two for &dyn Trait {}
```

`Two` is implemented for
- `Box<dyn Trait + 'static>`
- `&'a (dyn Trait + 'a)` for any `'a` (the lifetimes must match)

Consider using implementations like the following if possible, as they are
more general:
```rust
# trait Trait {}
# trait Two {}
// Implemented for all lifetimes
impl Two for Box<dyn Trait + '_> {}

// Implemented for all lifetimes such that the inner lifetime is
// at least as long as the outer lifetime
impl Two for &(dyn Trait + '_) {}
```

## Alias gotchas

Similar to `impl` headers, elision will apply when defining a type alias:
```rust,compile_fail
trait MyTraitICouldNotThinkOfAShortNameFor {}

// This is an alias to `dyn ... + 'static`!
type MyDyn = dyn MyTraitICouldNotThinkOfAShortNameFor;

// The default does not "override" the type alias and thus
// requires the trait object lifetime to be `'static`
fn foo(_: &MyDyn) {}

// As per the `dyn` elision rules, this requires the trait
// object lifetime to be the same as the reference...
fn bar(d: &dyn MyTraitICouldNotThinkOfAShortNameFor) {
    // ...and thus this fails as the lifetime cannot be extended
    foo(d);
}
```

More generally, elision does not "penetrate" or alter type aliases.
This includes the `Self` alias within implementation blocks.
```rust,compile_fail
trait Trait {}

impl dyn Trait {
    // Error: requires `T: 'static`
    fn f<T: Trait>(t: &T) -> &Self { t }
}

impl<'a> dyn Trait + 'a {
    // Error: requires `T: 'a`
    fn g<T: Trait>(t: &T) -> &Self { t }
}
```

See also [how type aliases with parameters behave.](./dyn-elision-advanced.md#iteraction-with-type-aliases)


## `'static` traits

When the trait itself is `'static`, the trait object lifetime has an implied
`'static` bound.  Therefore if you name the trait object lifetime explicitly,
the name you give it will also have an implied `'static` bound.  So here:
```rust,compile_fail
# use core::any::Any;
// n.b. trait `Any` has a `'static` bound
fn example<'a>(_: &'a (dyn Any + 'a)) {}

fn main() {
    let local = ();
    example(&local);
}
```

We get an error that the borrow of `local` must be `'static`.  The problem is
that `'a` in `example` has inherited the `'static` bound (`'a: 'static`), and
we also gave the outer reference the lifetime of `'a`.  This is a case where
we don't actually want them to be the same.

The most ergonomic solution is to always completely elide the trait object
lifetime when the trait itself has a `'static` bound.  Unlike other cases,
the trait object lifetime is independent of the outer reference lifetime when
the trait itself has a `'static` bound, so this compiles:
```rust
# use core::any::Any;
// This is `&'a (dyn Any + 'static)` and `'a` doesn't have to be `'static`
fn example(_: &dyn Any) {}

fn main() {
    let local = ();
    example(&local);
}
```

`Any` is the most common trait with a `'static` bound, i.e. the most likely
reason for you to encounter this scenario.

## `static` contexts

In some contexts like when declaring a `static`, it's possible to elide the
lifetime of types like references; doing so will result in `'static` being
used for the elided lifetime:
```rust
// The elided lifetime is `'static`
static S: &str = "";
const C: &str = "";
```

As a result, elided `dyn Trait` lifetimes will by default also be `'static`,
matching the inferred lifetime of the reference.  In contrast, this fails
due to the outer lifetime being `'static`:
```rust,compile_fail
# trait Trait {}
# impl Trait for () {}
struct S<'a: 'b, 'b>(&'b &'a str);
impl<'a: 'b, 'b> S<'a, 'b> {
    const T: &(dyn Trait + 'a) = &();
}
```

In this context, eliding all the lifetimes is again *usually* what you want.
