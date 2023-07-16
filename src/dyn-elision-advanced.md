# Advanced guidelines

In this section, we cover how to guide elision behavior for your own
generic data types, and point out some exceptions to the basic guidelines
presented in the previous section.

## Guiding behavior of your own types

When you declare a custom type with a lifetime parameter `'a` and a trait parameter `T: ?Sized`,
including an *explicit* `T: 'a` bound will result in elision behaving the same as
it does for references `&'a T` and for `std` types like `Ref<'a, T>`:
```rust
// When `T` is replaced by `dyn Trait` with an elided lifetime, the elided lifetime
// will default to `'a` outside of function bodies
struct ExplicitOutlives<'a, T: 'a + ?Sized>(&'a T);
```

If your type has no lifetime parameter, or if there is no bound between the type
parameter and the lifetime parameter, the default for elided `dyn Trait` lifetimes
will be `'static`, like it is for `Box<T>`.  *This is true even if there is an
**implied** `T: 'a` bound.*  For example:
```rust,compile_fail
trait Trait {}
impl Trait for () {}

// There's an *implied* `T: 'a` bound due to the `&'a T` field (RFC 2093)
struct InferredOutlivesOnly<'a, T: ?Sized>(&'a T);

// Yet this function expects an `InferredOutlivesOnly<'a, dyn Trait + 'static>`
fn example<'a>(ioo: InferredOutlivesOnly<'a, dyn Trait>) {}

// Thus this fails to compile
fn attempt<'a>(ioo: InferredOutlivesOnly<'a, dyn Trait + 'a>) {
    example(ioo);
}
```

If you make `T: 'a` explicit in the definition of the `struct`, the
example will compile.

If `T: 'a` is an inferred bound of your type, and `T: ?Sized`, I recommend
including the explicit `T: 'a` bound.

## Ambiguous bounds

If you have more than one lifetime bound in your type definition, the
bound is considered ambiguous, even if one of the lifetimes is `'static`
(or more generally, even if one lifetime is known to outlive the other).
Such structs are rare, but if you have one, you usually must be explicit
about the `dyn Trait` lifetime:
```rust,compile_fail
# trait Trait {}
# impl Trait for () {}
struct S<'a, 'b: 'a, T: 'a + 'b + ?Sized>(&'a &'b T);

// error[E0228]: the lifetime bound for this object type cannot be deduced
// from context; please supply an explicit bound
const C: S<dyn Trait> = S(&&());
```

However, in function bodies, the lifetime is still inferred; moreover it
is inferred independent of any annotation of the lifetime types:
```rust
# trait Trait {}
# impl Trait for () {}
struct Weird<'a, 'b, T: 'a + 'b + ?Sized>(&'a T, &'b T);

fn example<'a, 'b>() {
    // Either of `dyn Trait + 'a` or `dyn Trait + 'b` is an error,
    // so the `dyn Trait` lifetime must be inferred independently
    // from `'a` and `'b`
    let _: Weird<'a, 'b, dyn Trait> = Weird(&(), &());
}
```

(This is contrary to the documentation in the reference, and
[ironically more flexible than non-ambiguous types.](#an-exception-to-inference-in-function-bodies)
In this particular example, the lifetime will be inferred
[analogously to the lifetime intersection mentioned previously.](dyn-trait-lifetime.md#when-multiple-lifetimes-are-involved))

## Iteraction with type aliases

When you use a type alias, the bounds between lifetime parameters and type
parameters *on the `type` alias* determine how `dyn Trait` lifetime elision
behaves, overriding the bounds on the aliased type (be they stronger or weaker).

```rust,compile_fail
# trait Trait {}
// Without the `T: 'a` bound, the default trait object lifetime
// for this alias is `'static`
type MyRef<'a, T> = &'a T;

// So this compiles
fn foo(mr: MyRef<'_, dyn Trait>) -> &(dyn Trait + 'static) {
   mr
}

// With the `T: 'a` bound, the default trait object lifetime for
// this alias is the lifetime parameter
type MyOtherRef<'a, T: 'a> = MyRef<'a, T>;

// So this does not compile
fn bar(mr: MyOtherRef<'_, dyn Trait>) -> &(dyn Trait + 'static) {
   mr
}
```

[See issue 100270.](https://github.com/rust-lang/rust/issues/100270)
This is undocumented.

## Associated types and GATs

`dyn Trait` lifetime elision applies in this context.  There are some
things of note, however:
* Bounds on associated types and GATs don't seem to have any effect
* Eliding non-`dyn Trait` lifetimes is not allowed

For example:
```rust
# trait Trait {}
trait Assoc {
    type T;
}

impl Assoc for () {
    // Box<dyn Trait + 'static>
    type T = Box<dyn Trait>;
}

impl<'a> Assoc for &'a str {
    // &'a (dyn Trait + 'a)
    type T = &'a dyn Trait;
    // This is a compilation error as the reference lifetime is elided
    // type T = &dyn Trait;
}
```

```rust,compile_fail
# trait Trait {}
# impl Trait for () {}
trait BoundedAssoc<'x> {
    type BA: 'x + ?Sized;
}

// Still `Box<dyn Trait + 'static>`
impl<'x> BoundedAssoc<'x> for () { type BA = Box<dyn Trait>; }

// Fails
fn bib<'a>(obj: Box<dyn Trait + 'a>) {
    let obj: <() as BoundedAssoc<'a>>::BA = obj;
}
```

## An exception to inference in function bodies

There is also an exception to the elided `dyn Trait` lifetime being inferred
in function bodies.  If you have a reference-like type, and you annotate the
lifetime of the non-`dyn Trait` lifetime with a named lifetime, then the
elided `dyn Trait` lifetime will be the same as the annotated lifetime
(similar to how things behave outside of a function body):
```rust,compile_fail
trait Trait {}
impl Trait for () {}

fn example<'a>(arg: &'a ()) {
    let dt: &'a dyn Trait = arg;
    // fails
    let _: &(dyn Trait + 'static) = dt;
}
```

According to the reference, `&dyn Trait` should always behave like this.
However, if the outer lifetime is elided or if `'_` is used for the outer lifetime,
*the `dyn Trait` lifetime is inferred **independently** of the reference lifetime:*
```rust
# trait Trait {}
# impl Trait for () {}
fn example() {
    let local = ();

    // The outer reference lifetime cannot be `'static`...
    let obj: &dyn Trait = &local;

    // Yet the `dyn Trait` lifetime is!
    let _: &(dyn Trait + 'static) = obj;
}
```

This is not documented anywhere and is in conflict with the reference.
[It was implemented here,](https://github.com/rust-lang/rust/pull/39305)
with no team input or FCP.  🤷

However, the chances that you will run into a problem due to this behavior
is low, as it's rare to annotate lifetimes within a function body.

## What we are not yet covering

To the best of our knowledge, this covers the behavior of `dyn Trait` lifetime elision
*when there are no lifetime bounds on the trait itself.*  Non-`'static` lifetime bounds
on the trait itself lead to some more nuanced behavior; we'll cover some of them in the
[next section.](./dyn-elision-trait-bounds.md)

## Advanced guidelines summary

So all in all, we have three common categories of `dyn Trait` lifetime elision
when ignoring lifetime bounds on traits:

- `'static` for type parameters with no (explicit) lifetime bound
  - E.g. `Box<dyn Trait>` (`Box<dyn Trait + 'static>`)
  - E.g. `struct Unbounded<'a, T: ?Sized>(&'a T)`
- Another lifetime parameter for type parameters with a single (explicit) lifetime bound
  - E.g. `&'a dyn Trait` (`&'a dyn Trait + 'a`)
  - E.g. `Ref<'a, dyn Trait>` (`Ref<'a, dyn Trait + 'a>`)
  - E.g. `struct Bounded<'a, T: 'a + ?Sized>(&'a T)`
- Ambiguous due to multiple bounds (rare)
  - E.g. `struct Weird<'a, T: 'a + 'static>(&'a T);`

And the behavior in various contexts is:

| |  `static` | `impl` | \[G\]AT | `fn` in | `fn` out | `fn` body |
| - | - | - | - | - | - | - |
| `Box<dyn Trait>` | `'static` | `'static` | `'static` | `'static` | `'static` | Inferred
| `&dyn Trait` | Ref | Ref | E0637 | Ref | Ref | Inferred
| `&'a dyn Trait` | Ref | Ref | Ref | Ref | Ref | Ref
| Ambig. | E0228 | E0228 | E0228 | E0228 | E0228  | Inferred

With the following notes:
- `type` alias bounds take precedence over the aliased type bounds
- Associated type and GAT bounds do not effect the default

For contrast, the "normal" elision rules work like so:

|  | `static` | `impl` | \[G\]AT | `fn` in | `fn` out | `fn` body |
| - | - | - | - | - | - | - |
| `Box<dyn Tr + '_>` | `'static` | Fresh | E0637 | Fresh | Elision | Inferred
| `&(dyn Trait + '_`) | `'static` | Fresh | E0637 | Fresh | Elision | Inferred
| `&'a (dyn Trait + '_`) | `'static` | Fresh | E0637 | Fresh | Elision | Inferred
| Ambig. with `'_` | `'static` | Fresh | E0637 | Fresh | Elision | Inferred


