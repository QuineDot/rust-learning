# Lifetime bounds

Here's a brief introduction to the lifetime bounds you may see on `fn` declarations and `impl` blocks.

## Bounds between lifetimes

A `'a: 'b` bound means, roughly speaking, `'long: 'short`.
It's often read as "`'a` outlives `'b`" and it sometimes called an "outlives bound" or "outlives relation".

I personally also like to read it as "`'a` is valid for (at least) `'b`".

Note that `'a` may be the same as `'b`, it does not have to be strictly longer despite the "outlives"
terminology.  It is analogous to `>=` in this respect.  Therefore, in this example:
```rust
fn example<'a: 'b, 'b: 'a>(a: &'a str, b: &'b str) {}
```

`'a` and `'b` must actually be the same lifetime.

When you have a function argument with a nested reference such as `&'b Foo<'a>`, a `'a: 'b` bound is inferred.

## Bounds between (generic) types and lifetimes

A `T: 'a` means that a `&'a T` would not be instantly undefined behavior.  In other words, it
means that if the type `T` contains any references or other lifetimes, they must be at least
as long as `'a`.

You can also read these as "(the type) `T` is valid for `'a`".

Note that this has nothing to do with the liveness scope or drop scope of a *value* of type `T`!
In particular the most common bound of this form is `T: 'static`.

This does not mean the value of type `T` must last for your entire program!  It just means that
the type `T` has no non-`'static` lifetimes.  `String: 'static` for example, but this doesn't
mean that you don't drop `String`s.

You can consider an outlives bound to apply recursively and syntactically.  For example:
```rust
fn example<'s, 'limit, T>(_s: &'s str, _v: Vec<T>)
where
    // (You wouldn't normally write a bound like this, but for illustration:)
    (&'s str, Vec<T>): 'limit,
{}
```
`(&'s str, Vec<T>): 'limit` holds if and only if
1. `&'s str: 'limit` holds, which happens if and only if
   - `'s: 'limit` holds
2. `Vec<T>: 'limit` holds, which happens if and only if
   - `T: 'limit` holds

(When you call the function with a concrete `T`, the process may continue.)

## Liveness scopes of values

For the above reasons, I prefer to never refer to the liveness or drop scope of a value as
the value's "lifetime".  Although there is a connection between the liveness scope of values
and lifetimes of references you take to it, conflating the two concepts can lead to confusion.

That said, not everyone follows this convention, so you may see the liveness scope of a value
referred to as the value's "lifetime".  The distinction is something to generally be aware of.

Rust lifetimes -- those `'a` things -- are much more closely related to *the duration of borrows*
than to a particular value's scope.
