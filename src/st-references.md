# Reference lifetimes

Here's something you'll utilize in Rust all the time without thinking about it:
* A `&'long T` coerces to a `&'short T`
* A `&'long mut T` coerces to a `&'short mut T`

The technical term is "covariant (in the lifetime)" but a practical mental model is "the (outer) lifetime of references can shrink".

The property holds for values whose type is a reference, but it doesn't always hold for other types.
For example, we'll soon see that this property doesn't always hold for the lifetime of a reference
nested within another reference.  When the property doesn't hold, it's usually due to *invariance*.

Even if you never really think about covariance, you'll grow an intuition for it -- so much for so
that you'll eventually be surprised when you encounter invariance and the property doesn't hold.
We'll look at some cases soon.
