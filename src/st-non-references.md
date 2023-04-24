# Non-references

The variance of lifetime and type parameters of your own `struct`s is automatically
inferred from how you use those parameters in your field.  For example if you have a
```rust
struct AllMine<'a, T>(&'a mut T);
```

Then `AllMine` is covariant in `'a` and invariant in `T`, just like `&'a mut T` is.
