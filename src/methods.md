# Get a feel for borrow-returning methods

Here we look at how borrow-returning methods work.  Our examples will consider a typical
pattern:
```rust,ignore
fn method(&mut self) -> &SomeReturnType {}
// AKA:
// fn method<'this>(&'this mut self) -> &'this SomeReturnType {}
```

The meaning of this signature is: uses of the returned value keep `*self`
exclusively borrowed.

That is the most important thing to keep in mind.  The following subsections
go into some more details which may be useful.
