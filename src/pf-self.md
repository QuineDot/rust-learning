# `&'a mut self` and `Self` aliasing more generally

`fn foo(&'a mut self)` is a red flag -- because if that `'a` wasn't declared on the function,
it's probably part of the `Self` struct.  And if that's the case, this is a `&'a mut Thing<'a>`
in disguise.  [As discussed in the previous section,](./pf-borrow-forever.md) this will make
`self` unusable afterwards, and thus is an anti-pattern.

More generally, `self` types and the `Self` alias include any parameters on the type constructor post-resolution.
[Which means here:](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=a46b294ec613acda99767069a744c488)
```rust
struct Node<'a>(&'a str);

impl<'a> Node<'a> {
    fn new(s: &str) -> Self {
        Node(s)
    }
}
```

`Self` is an alias for `Node<'a>`.  It is *not* an alias for `Node<'_>`.  So it means:
```rust
    fn new<'s>(s: &'s str) -> Node<'a> {
```
And not:
```rust
    fn new<'s>(s: &'s str) -> Node<'s> {
```
And you really meant to code one of these:
```rust
    fn new(s: &'a str) -> Self {
    fn new(s: &str) -> Node<'_> {
```

Similarly, using `Self` as a constructor will use the resolved type parameters.  So this won't work:
```rust
    fn new(s: &str) -> Node<'_> {
        Self(s)
    }
```
You need
```rust
    fn new(s: &str) -> Node<'_> {
        Node(s)
    }
```

