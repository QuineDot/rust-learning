# Citations

Finally we compare what we've covered about `dyn Trait` lifetime elision to the
current reference material, and supply some citations to the elision's storied
history.

## Summary of differences from the reference

The official documentation on trait object lifetime elision
[can be found here.](https://doc.rust-lang.org/reference/lifetime-elision.html#default-trait-object-lifetimes)

In summary, it states that `dyn Trait` lifetimes have a *default object lifetime bound* which varies based on context.
It states that the default bound only takes effect when the lifetime is *entirely* omitted.  When you write out `dyn Trait + '_`, the
[normal lifetime elision rules](https://doc.rust-lang.org/reference/lifetime-elision.html#lifetime-elision-in-functions)
apply instead.

In particular, as of this writing, the official documentation states that
> If the trait object is used as a type argument of a generic type then the containing type is first used to try to infer a bound.
> - If there is a unique bound from the containing type then that is the default
> - If there is more than one bound from the containing type then an explicit bound must be specified
>
> If neither of those rules apply, then the bounds on the trait are used:
> - If the trait is defined with a single lifetime bound then that bound is used.
> - If `'static` is used for any lifetime bound then `'static` is used.
> - If the trait has no lifetime bounds, then the lifetime is inferred in expressions and is `'static` outside of expressions.

Some differences from the reference which we have covered are that
- [inferring bounds in expressions applies to `&T` types unless annotated with a named lifetime](./dyn-elision-advanced.md#an-exception-to-inference-in-function-bodies)
- [inferring bounds in expressions applies to ambigous types](./dyn-elision-advanced.md#ambiguous-bounds)
- [when trait bounds apply, they override struct bounds, not the other way around](./dyn-elision-trait-bounds.md#when-they-apply-trait-lifetime-bounds-override-struct-bounds)
- [a `'static` trait bound always applies](./dyn-elision-trait-bounds.md#the-static-case)
- [otherwise, whether trait bounds apply or not depends on complicated contextual rules](./dyn-elision-trait-bounds.md#when-and-how-to-trait-lifetime-bounds-apply)
  - they always apply in `impl` headers, associated types, and function bodies
  - and technically in `static` contexts, with some odd cavaets
  - whether they apply in function signatures depends on the bounding parameters being late or early bound
    - a single parameter can apply to a trait bounds with multiple bounds in this context, introducing new implied lifetime bounds
- [trait bounds override `'_` in function bodies](./dyn-elision-trait-bounds.md#function-bodies)

And some other under or undocumented behaviors are that
- [aliases override struct definitions](./dyn-elision-advanced.md#iteraction-with-type-aliases)
- [trait bounds create implied bounds on the trait object lifetime](./dyn-elision-trait-bounds.md#trait-lifetime-bounds-create-an-implied-bound)
- [associated type and GAT bounds do not effect the default trait object lifetime](./dyn-elision-advanced.md#associated-types-and-gats)

## RFCs, Issues, and PRs

Trait objects, and trait object lifetime elision in particular, has undergone a lot of evolution over time.
Here we summarize some of the major developments and issues.

Reminder: a lot of these citations predate the [`dyn Trait` syntax.](https://rust-lang.github.io/rfcs/2113-dyn-trait-syntax.html)
Trait objects used to be just "spelled" as `Trait` in type position, instead of `dyn Trait`.

- [RFC 0192](https://rust-lang.github.io/rfcs/0192-bounds-on-object-and-generic-types.html#lifetime-bounds-on-object-types) first introduced the trait object lifetime
  - [including the "intersection lifetime" consideration](https://rust-lang.github.io/rfcs/0192-bounds-on-object-and-generic-types.html#appendix-b-why-object-types-must-have-exactly-one-bound)
- [RFC 0599](https://rust-lang.github.io/rfcs/0599-default-object-bound.html) first introduced *default* trait object lifetimes (`dyn Trait` lifetime elision)
- [RFC 1156](https://rust-lang.github.io/rfcs/1156-adjust-default-object-bounds.html) superceded RFC 0599 (`dyn Trait` lifetime elision)
- [PR 39305](https://github.com/rust-lang/rust/pull/39305) modified RFC 1156 (unofficially) to [allow more inference in function bodies](dyn-elision-advanced.md#an-exception-to-inference-in-function-bodies)
- [RFC 2093](https://rust-lang.github.io/rfcs/2093-infer-outlives.html#trait-object-lifetime-defaults) defined [how `struct` bounds interact with `dyn Trait` lifetime elision](dyn-elision-advanced.md#guiding-behavior-of-your-own-types)
- [Issue 100270](https://github.com/rust-lang/rust/issues/100270) notes that type aliases take precedent in terms of RFC 2093 `dyn Trait` lifetime elision rules
- [Issue 47078](https://github.com/rust-lang/rust/issues/47078) notes that being late-bound influences `dyn Trait` lifetime elision

