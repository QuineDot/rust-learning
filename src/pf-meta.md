# Avoid self-referential structs

By self-referential, I mean you have one field that is a reference, and that reference points to another field (or contents of a field) in the same struct.
```rust
struct Snek<'a> {
    owned: String,
    // Like if you want this to point to the `owned` field
    borrowed: &'a str,
}
```

The only safe way to construct this to be self-referential is to take a `&'a mut Snek<'a>`, get a `&'a str` to the `owned` field, and assign it to the `borrowed` field.
```rust
#struct Snek<'a> {
#    owned: String,
#    // Like if you want this to point to the `owned` field
#    borrowed: &'a str,
#}

impl<'a> Snek<'a> {
    fn bite(&'a mut self) {
        self.borrowed = &self.owned;
    }
}
```

[And as was covered before,](pf-borrow-forever.md) that's an anti-pattern because you cannot use the self-referencial struct directly ever again.
The only way to use it at all is via a (reborrowed) return value from the method call that required `&'a mut self`.

So it's technically possible, but so restrictive it's pretty much always useless.

Trying to create self-referential structs is a common newcomer misstep, and you may see the response to questions about them in the approximated form of "you can't do that in safe Rust".

