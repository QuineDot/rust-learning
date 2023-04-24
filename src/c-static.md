# Advice to add a static bound

The compiler is gradually getting better about this, but when it suggests to use a `&'static` or that a lifetime needs to outlive `'static`, it usually *actually* means either
* You're in a context where non-`'static` references and other non-`static` types aren't allowed
* You should add a lifetime parameter somewhere

Rather than try to cook up my own example, [I'll just link to this issue.](https://github.com/rust-lang/rust/issues/50212)
Although it's closed, there's still room for improvement in some of the examples within.


