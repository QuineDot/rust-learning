# When not to name lifetimes

Sometimes newcomers try to solve borrow check errors by making things more generic,
which often involves adding lifetimes and naming previously-elided lifetimes:
```rust
# struct S; impl S {
fn quz<'a: 'b, 'b>(&'a mut self) -> &'b str { todo!() }
# }
```
But this doesn't actually permit more lifetimes than this:
```rust
# struct S; impl S {
fn quz<'b>(&'b mut self) -> &'b str { todo!() }
# }
```

Because in the first example, `&'a mut self` can coerce to `&'b mut self`.
And, in fact, you want it to -- because you generally don't want to exclusively borrow `self` any longer than necessary.

And at this point you can instead utilize lifetime elision and stick with:
```rust
# struct S; impl S {
fn quz(&mut self) -> &str { todo!() }
# }
```

As covariance and function lifetime elision become more intuitive, you'll build a feel for when it's
pointless to name lifetimes.  Adding surpuflous lifetimes like in the first example tends to make
understanding borrow errors harder, not easier.
