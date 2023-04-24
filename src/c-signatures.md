# Advice to change function signature when aliases are involved

[Here's a scenario from earlier in this guide.](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=ad6f395f748927ae66d06b2fd42603ea)  The compiler advice is:
```rust
error[E0621]: explicit lifetime required in the type of `s`
 --> src/lib.rs:5:9
  |
4 |     fn new(s: &str) -> Node<'_> {
  |               ---- help: add explicit lifetime `'a` to the type of `s`: `&'a str`
5 |         Self(s)
  |         ^^^^^^^ lifetime `'a` required
```
But [this works just as well:](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=9d89786c76424d66e7eb11ca7716645b)
```diff
-        Self(s)
+        Node(s)
```
And you may get this advice when implementing a trait, where you usually *can't* change the signature.

