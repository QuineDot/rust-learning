# Advice to add bound which implies lifetime equality

The example for this one is very contrived, but [consider the output here:](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=aad18e17e4308fc4e94c5fd039e64d90)
```rust
fn f<'a, 'b>(s: &'a mut &'b mut str) -> &'b str {
    *s
}
```
```rust
  = help: consider adding the following bound: `'a: 'b`
```

With the nested lifetime in the argument, there's already an implied `'b: 'a` bound.
If you follow the advice and add a `'a: 'b` bound, then the two bounds together imply that `'a` and `'b` are in fact the same lifetime.
More clear advice would be to use a single lifetime.  Even better advice for this particular example would be to return `&'a str` instead.

Another possible pitfall of blindly following this advice is ending up with something like this:
```rust
impl Node<'a> {
    fn g<'s: 'a>(&'s mut self) { /* ... */ }
```
That's [the `&'a mut Node<'a>` anti-pattern](pf-self.md) in disguise!  This will probably be unusable and hints at a deeper problem that needs solved.

