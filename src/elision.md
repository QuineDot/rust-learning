# Understand elision and get a feel for when to name lifetimes

[Read the function lifetime elision rules.](https://doc.rust-lang.org/reference/lifetime-elision.html)
They're intuitive for the most part.  The elision rules are built around being what you need in the common case,
but they're not always the solution.  For example, they assume an input of `&'s self` means you mean to return a
borrow of `self` or its contents with lifetime `'s`, but this is often *not* correct for lifetime-carrying
data structures.

If you get a lifetime error involving elided lifetimes, try giving all the lifetimes names. This can improve
the compiler errors; if nothing else, you won't have to work so hard to mentally track the numbers used in
errors for illustration, or what "anonymous lifetime" is being talked about.

Take care to refer to the elision rules when naming all the lifetimes, or you may inadvertently change the
meaning of the signature.  If the error changes *drastically,* you've probably changed the meaning of the
signature.

Once you have a good feel for the function lifetime elision rules, you'll start developing intuition for
when you need to name your lifetimes instead.
