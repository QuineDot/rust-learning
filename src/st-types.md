# Reference types

Let's open with a question: is `&str` a type?

When not being pendantic or formal, pretty much everyone will say yes, `&str` is a type.
However, it is technically a *type constructor* which is parameterized with a generic
lifetime parameter.  So `&str` isn't technically a type, `&'a str` for some concrete
lifetime is a type.  

Similarly, `Vec<T>` for a generic `T` is a type constructor, but `Vec<i32>` is a type.

By "concrete lifetime", I mean some compile-time determined lifetime.  The exact
definition of "lifetime" is suprisingly complicated and beyond the scope of this
guide, but here are a few examples of `&str`s and their concrete types.

```rust
// The exact lifetime of `'a` is determined at each call site.  We'll explore
// what this means in more depth later.
//
// The lifetime of `b` works the same, we just didn't give it a name.
fn example<'a>(a: &'a str, b: &str) {
    // Literal strings are `&'static str`
    let s = "literal";

    // The lifetime of local borrows are determined by compiler analysis
    // and have no names (but it's still a single lifetime).
    let local = String::new();
    let borrow = local.as_str();

    // These are the same and they just tell the compiler to infer the
    // lifetime.  In this small example that means the same thing as not
    // having a type annotation at all.
    let borrow: &str = local.as_str();
    let borrow: &'_ str = local.as_str();
}
```

