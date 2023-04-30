# Borrowing something forever

An anti-pattern you may run into is to create a `&'a mut Thing<'a>`.  It's an anti-pattern because it translates
into "take an exclusive reference of `Thing<'a>` for the entire rest of it's validity (`'a`)".  Once you create
the exclusive borrow, you cannot use the `Thing<'a>` ever again, except via that borrow.

You can't call methods on it, you can't take another reference to it, you can't move it, you can't print it, you
can't use it at all.  You can't even call a non-trivial destructor on it; if you have a non-trivial destructor,
your code won't compile in the presence of `&'a mut Thing<'a>`.

So avoid `&'a mut Thing<'a>`.

---

Examples:
```rust
#[derive(Debug)]
struct Node<'a>(&'a str);
fn example_1<'a>(node: &'a mut Node<'a>) {}

struct DroppingNode<'a>(&'a str);
impl Drop for DroppingNode<'_> { fn drop(&mut self) {} }
fn example_2<'a>(node: &'a mut DroppingNode<'a>) {}

fn main() {
    let local = String::new();

    let mut node_a = Node(&local);
    // You can do this once and it's ok...
    example_1(&mut node_a);
    
    let mut node_b = Node(&local);
    // ...but then you can't use the node directly ever again
    example_1(&mut node_b);
    println!("{node_b:?}");
    
    let mut node_c = DroppingNode(&local);
    // And this doesn't work at all
    example_2(&mut node_c);
}
```
