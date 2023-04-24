# Get a feel for borrow-returning methods

Here we look at how borrow-returning methods work.  Our examples will consider a typical
pattern:
```rust
fn method(&mut self) -> &SomeReturnType {}
```
