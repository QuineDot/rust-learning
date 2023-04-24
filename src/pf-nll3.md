# Conditional return of a borrow


The compiler isn't perfect, and there are some things it doesn't yet accept which are in fact sound and could be accepted.
Perhaps the most common one to trip on is [conditional return of a borrow,](https://github.com/rust-lang/rust/issues/51545) aka NLL Problem Case #3.
There are some examples and workarounds in the issue and related issues.

The plan is still to accept that pattern some day.

More generally, if you run into something and don't understand why it's an error or think it should be allowed, try asking in a forum post.

