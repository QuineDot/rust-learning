# The seed of a mental model

Some find it helpful to think of shared (`&T`) and exclusive (`&mut T`) references like so:

* `&T` is a compiler-checked `RwLockReadGuard`
  * You can have as many of these at one time as you want
* `&mut T` is an compiler-checked `RwLockWriteGuard`
  * You can only have one of these at one time, and when you do, you can have no `RwLockReadGuard`

The exclusivity is key.

`&mut T` are often called "mutable references" for obvious reasons.  And following from that, `&T`
are often called "immutable references".  However, I find it more accurate and consistent to call
`&mut T` an exclusive reference and to call `&T` a shared reference.

This guide doesn't yet cover shared mutability, more commonly called interior mutability, but you'll
run into the concept sometime in your Rust journey.  The one thing I will mention here is that it
enables mutation behind a `&T`, and thus "immutable reference" is a misleading name.

There are other situations where the important quality of a `&mut T` is the exclusivity that it
guarantees, and not the ability to mutate through it.  If you find yourself annoyed at getting
borrow errors about `&mut T` when you performed no actual mutation, it may help to instead
consider `&mut T` as a directive to the compiler to ensure *exclusive* access instead of
*mutable* access.
