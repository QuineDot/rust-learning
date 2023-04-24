# Practical suggestions for building intuition around borrow errors

Ownership, borrowing, and lifetimes covers a lot of ground in Rust, and thus this section is somewhat long.
It is also hard to create a succinct overview of the topic, as what aspects of the topic a newcomer
first encounters depends on what projects they take on in the process of learning Rust. The genesis of
this guide was advice for someone who picked a zero-copy regular expression crate, and as such they ran
into a lot of lifetime issues off the bad.  Someone who chose a web-frame work would be more likely to
run into issues around `Arc` and `Mutex`, those who start with `async` projects will likely run into
many `async`-specific errors, and so on.

Despite the length of this section, there are entire areas it doesn't yet touch on, such as destructor
mechanics, shared ownership and shared mutability, and so on.

Therefore, it's not expected that anyone will absorb everything in this guide all at once.  Instead,
it's intended to be a broad introduction to the topic.  Skim it on your first pass and take time on
the parts that seem relevant, or use them as pointers to look up more in-depth documentation on your
own.  Hopefully you will get a feel for what kind errors can arise even if you haven't ran into them
yourself yet, so that you're not completely baffled when they do arise.

In general, your mental model of ownership and borrowing will go through a few stages of evolution.
It will pay off to return to any given area of the topic with fresh eyes every once in awhile.
