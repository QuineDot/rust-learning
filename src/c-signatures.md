# Advice to change function signature when aliases are involved

[Here's a scenario from earlier in this guide.](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=54379d0c026ccdf73eb3fdde3292b4fb)  The compiler advice is:
```rust,ignore
error[E0621]: explicit lifetime required in the type of `s`
 --> src/lib.rs:5:9
  |
5 |         Self(s)
  |         ^^^^^^^ lifetime `'a` required
  |
help: add explicit lifetime `'a` to the type of `s`
  |
4 |     fn new(s: &'a str) -> Node<'_> {
  |                ++
```

But [this works just as well:](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=268f52ef82439f2036f787a5490aa114)
```diff
-        Self(s)
+        Node(s)
```

And you may even get the above compiler advice when implementing a trait, where you usually *can't* change the signature.

