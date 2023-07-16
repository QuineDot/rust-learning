# &mut inputs don't "downgrade" to &

Still talking about this signature:
```rust
fn quz(&mut self) -> &str { todo!() }
```
Newcomers often expect `self` to only be shared-borrowed after `quz` returns, because the return is a shared reference.
But that's not how things work; `self` remains *exclusively* borrowed for as long as the returned `&str` is valid.

I find looking at the exact return type a trap when trying to build a mental model for this pattern.
The fact that the lifetimes are connected is crucial, but beyond that, instead focus on the input parameter:
You *cannot call* the method *until you have created* a `&mut self` with a lifetime as long as the return type has.
Once that exclusive borrow (or reborrow) is created, the exclusiveness lasts for the entirety of the lifetime.
Moreover, you give the `&mut self` away by calling the method (it is not `Copy`), so you can't create any other
reborrows to `self` other than through whatever the method returns to you (in this case, the `&str`).


