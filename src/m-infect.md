# Bound-related lifetimes "infect" each other

Separating `'a` and `'b` in the last section didn't make things any more flexible in terms of `self` being borrowed.
Once you declare a bound like `'a: 'b`, then the two lifetimes "infect" each other.
Even though the return type had a different lifetime than the input, it was still effectively a reborrow of the input.

This can actually happen between two input parameters too: if you've stated a lifetime relationship between two borrows,
the compiler assumes they can observe each other in some sense.  It's probably not anything you'll run into soon,
but if you do, the compiler errors tend to be drop errors ("borrow might be used here, when `x` is dropped"), or
sometimes read like "data flows from X into Y".


