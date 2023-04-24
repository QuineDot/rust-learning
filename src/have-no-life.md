# Prefer ownership over long-lived references

A common newcomer mistake is to overuse references or other lifetime-carrying
data structures by storing them in their own long-lived structures.  This is
often a mistake as the primary use-case of non-`'static` references is for
short-term borrows.

If you're trying to create a lifetime-carrying struct, stop and consider if
you could use a struct with no lifetime instead, for example by replacing
`&str` with `String`, or `&[T]` with `Vec<T>`.


