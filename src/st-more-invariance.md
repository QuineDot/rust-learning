# Invariance elsewhere

While behind a `&mut` is the most common place to first encounter invariance,
it's present elsewhere as well.

`Cell<T>` and `RefCell<T>` are also invariant in `T`.

Trait parameters are invariant too.  As a result, lifetime-parameterized traits can be onerous to work with.

Additionally, if you have a bound like `T: Trait<U>`, `U` becomes invariant becase it's a type parameter of the trait.
If your `U` resolves to `&'x V`, the lifetime `'x` will be invariant too.

