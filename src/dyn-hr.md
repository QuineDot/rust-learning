# Higher-ranked types

Another feature of trait objects is that they can be *higher-ranked* over
lifetime parameters of the trait:
```rust
// A trait with a lifetime parameter
trait Look<'s> {
    fn method(&self, s: &'s str);
}

// An implementation that works for any lifetime
impl<'s> Look<'s> for () {
    fn method(&self, s: &'s str) {
        println!("Hi there, {s}!");
    }
}

fn main() {
    // A higher-ranked trait object
    //           vvvvvvvvvvvvvvvvvvvvvvvv
    let _bx: Box<dyn for<'any> Look<'any>> = Box::new(());
}
```
The `for<'x>` part is a *lifetime binder* that introduces higher-ranked
lifetimes.  There can be more than one lifetime, and you can give them
arbitrary names just like lifetime parameters on functions, structs,
and so on.

You can only coerce to a higher-ranked trait object if you implement
the trait in question for *all* lifetimes.  For example, this doesn't
work:
```rust,compile_fail
# trait Look<'s> { fn method(&self, s: &'s str); }
impl<'s> Look<'s> for &'s i32 {
    fn method(&self, s: &'s str) {
        println!("Hi there, {s}!");
    }
}

fn main() {
    let _bx: Box<dyn for<'any> Look<'any>> = Box::new(&0);
}
```
`&'s i32` only implements `Look<'s>`, not `Look<'a>` for all lifetimes `'a`.

Similarly, this won't work either:
```rust,compile_fail
# trait Look<'s> { fn method(&self, s: &'s str); }
impl Look<'static> for i32 {
    fn method(&self, s: &'static str) {
        println!("Hi there, {s}!");
    }
}   

fn main() {
    let _bx: Box<dyn for<'any> Look<'any>> = Box::new(0);
}
```

Implementing the trait with `'static` as the lifetime parameter is not the
same thing as implementing the trait for any lifetime as the parameter.
Traits and trait implementations don't have something like variance; the
parameters of traits are always invariant and thus implementations are
always for the explicit lifetime(s) only.

## Subtyping

There's a relationship between higher-ranked types like `dyn for<'any> Look<'any>`
and non-higher-ranked types like `dyn Look<'x>` (for a single lifetime `'x`): the
higher-ranked type is a subtype of the non-higher-ranked types.  Thus you can
coerce a higher-ranked type to a non-higher-ranked type with any concrete lifetime:
```rust
# trait Look<'s> { fn method(&self, s: &'s str); }
fn as_static(bx: Box<dyn for<'any> Look<'any>>) -> Box<dyn Look<'static>> {
    bx
}

fn as_whatever<'w>(bx: Box<dyn for<'any> Look<'any>>) -> Box<dyn Look<'w>> {
    bx
}
```

Note that this still isn't a form of variance for the *lifetime parameter* of the
trait.  This fails for example, because you can't coerce from `dyn Look<'static>`
to `dyn Look<'w>`:
```rust
# trait Look<'s> { fn method(&self, s: &'s str); }
# fn as_static(bx: Box<dyn for<'any> Look<'any>>) -> Box<dyn Look<'static>> { bx }
fn as_whatever<'w>(bx: Box<dyn for<'any> Look<'any>>) -> Box<dyn Look<'w>> {     
    as_static(bx)
}
```

As a supertype coercion, going from higher-ranked to non-higher-ranked can
apply even in a covariant nested context,
[just like non-higher-ranked supertype coercions:](./dyn-covariance.md#variance-in-nested-context)
```rust
# trait Look<'s> {}
fn foo<'l: 's, 's, 'p>(
    v: Vec<Box<dyn for<'any> Look<'any> + 'l>>
) -> Vec<Box<dyn Look<'p> + 's>>
{
    v
}
```

## `Fn` traits and `fn` pointers

The `Fn` traits ([`FnOnce`](https://doc.rust-lang.org/std/ops/trait.FnOnce.html),
[`FnMut`](https://doc.rust-lang.org/std/ops/trait.FnMut.html),
and [`Fn`](https://doc.rust-lang.org/std/ops/trait.Fn.html))
have special-cased syntax.  For one, you write them out to look more like
a function, using `(TypeOne, TypeTwo)` to list the input parameters and
`-> ResultType` to list the associated type.  But for another, elided
input lifetimes are sugar that introduces higher-ranked bindings.

For example, these two trait object types are the same:
```rust
fn identity(bx: Box<dyn Fn(&str)>) -> Box<dyn for<'any> Fn(&'any str)> {
    bx
}
```

This is similar to how elided lifetimes work for function declarations
as well, and indeed, the same output lifetime elision rules also apply:
```rust
// The elided input lifetime becomes a higher-ranked lifetime
// The elided output lifetime is the same as the single input lifetime
//     (underneath the binder)
fn identity(bx: Box<dyn Fn(&str) -> &str>) -> Box<dyn for<'any> Fn(&'any str) -> &'any str> {
    bx
}
```
```rust,compile_fail
// Doesn't compile as what the output lifetime should be is
// considered ambiguous
fn ambiguous(bx: Box<dyn Fn(&str, &str) -> &str>) {}

// Here's a possible fix, which is also an example of
// multiple lifetimes in the binder
fn first(bx: Box<dyn for<'a, 'b> Fn(&'a str, &'b str) -> &'a str>) {}
```

Function pointers are another example of types which can be higher-ranked
in Rust.  They have analogous syntax and sugar to function declarations
and the `Fn` traits.
```rust
fn identity(fp: fn(&str) -> &str) -> for<'any> fn(&'any str) -> &'any str {
    fp
}
```

### Syntactic inconsistencies

There are some inconsistencies around the syntax for function declarations,
function pointer types, and the `Fn` traits involving the "names" of the
input arguments.

First of all, only function (method) declarations can make use of the
shorthand `self` syntaxes for receivers, like `&self`:
```rust
# struct S;
impl S {
    fn foo(&self) {}
    //     ^^^^^
}
```
This exception is pretty unsurprising as the `Self` alias only exists
within those implementation blocks.

Each non-`self` argument in a function declaration is an
[irrefutable pattern](https://doc.rust-lang.org/reference/items/functions.html#function-parameters)
followed by a type annotation.  It is an error to leave out the pattern;
if you don't use the argument (and thus don't need to name it), you
still need to use at least the wildcard pattern.
```rust,compile_fail
fn this_works(_: i32) {}
fn this_fails(i32) {}
```
There is
[an accidental exception](https://rust-lang.github.io/rfcs/1685-deprecate-anonymous-parameters.html)
to this rule, but it was removed in Edition 2018 and thus is only
available on Edition 2015.

In contrast, each argument in a function pointer can be
- An *identifier* followed by a type annotation (`i: i32`)
- `_` followed by a type annotation (`_: i32`)
- Just a type name (`i32`)

So these all work:
```rust
let _: fn(i32) = |_| {};
let _: fn(i: i32) = |_| {};
let _: fn(_: i32) = |_| {};
```
But *actual* patterns [are not allowed:](https://doc.rust-lang.org/stable/error_codes/E0561.html)
```rust,compile_fail
let _: fn(&i: &i32) = |_| {};
```
The idiomatic form is to just use the type name.

It's also allowed [to have colliding names in function pointer
arguments,](https://github.com/rust-lang/rust/issues/33995) but this
is a property of having no function body -- so it's also possible in
a trait method declaration, for example.  It is also related to the
Edition 2015 exception for anonymous function arguments mentioned
above, and may be deprecated eventually.
```rust
trait Trait {
    fn silly(a: u32, a: i32);
}

let _: fn(a: u32, a: i32) = |_, _| {};
```

Finally, each argument in the `Fn` traits can *only* be a type name:
no identifiers, `_`, or patterns allowed.
```rust,compile_fail
// None of these compile
let _: Box<dyn Fn(i: i32)> = Box::new(|_| {});
let _: Box<dyn Fn(_: i32)> = Box::new(|_| {});
let _: Box<dyn Fn(&_: &i32)> = Box::new(|_| {});
```

Why the differences? One reason is that
[patterns are grammatically incompatible with anonymous arguments,
apparently.](https://github.com/rust-lang/rust/issues/41686#issuecomment-366611096)
I'm uncertain as to why identifiers are accepted on function pointers,
however, or more generally why the `Fn` sugar is inconsistent with
function pointer types.  But the simplest explanation is that function
pointers existed first with nameable parameters for whatever reason,
whereas the `Fn` sugar is for trait input type parameters which also
do not have names.

## Higher-ranked trait bounds

You can also apply higher-ranked trait bounds (HRTBs) to generic
type parameters, using the same syntax:
```rust
# trait Look<'s> { fn method(&self, s: &'s str); }
fn box_it_up<'t, T>(t: T) -> Box<dyn for<'any> Look<'any> + 't>
where
    T: for<'any> Look<'any> + 't,
{
    Box::new(t)
}
```

The sugar for `Fn` like traits applies here as well.  You've probably
already seen bounds like this on methods that take closures:
```rust
# struct S;
# impl S {
fn map<'s, F, R>(&'s self, mut f: F) -> impl Iterator<Item = R> + 's
where
    F: FnMut(&[i32]) -> R + 's
{
    // This part isn't the point ;-)
    [].into_iter().map(f)
}
# }
```

That bound is actually `F: for<'x> FnMut(&'x [i32]) -> R + 's`.

## That's all about higher-ranked types for now

Hopefully this has given you a decent overview of higher-ranked
types, HRTBs, and how they relate to the `Fn` traits.  There
are a lot more details and nuances to those topics and related
concepts such as closures, as you might imagine.  However, an
exploration of those topics deserves its own dedicated guide, so
we won't see too much more about higher-ranked types in this
tour of `dyn Trait`.
