# Understand borrows within a function

The analysis that the compiler does to determine lifetimes and borrow check
*within* a function body is quite complicated.  A full exploration is beyond
the scope of this guide, but we'll give a brief introduction here.

Your best bet if you run into an error you can't understand is to
ask for help on the forum or elsewhere.

## Borrow errors within a function

Here are some simple causes of borrow check errors within a function.

### Recalling the Basics

The most basic mechanism to keep in mind is that `&mut` references are exclusive,
while `&` references are shared and implement `Copy`.  You can't intermix using
a shared reference and an exclusive reference to the same value, or two exclusive
references to the same value.

```rust
# fn main() {
let mut local = "Hello".to_string();

// Creating and using a shared reference
let x = &local;
println!("{x}");

// Creating and using an exclusive reference
let y = &mut local;
y.push_str(", world!");

// Trying to use the shared reference again
println!("{x}");
# }
```

This doesn't compile because as soon as you created the exclusive reference,
any other existing references must cease to be valid.

### Borrows are often implicit

Here's the example again, only slightly rewritten.
```rust
# fn main() {
let mut local = "Hello".to_string();

// Creating and using a shared reference
let x = &local;
println!("{x}");

// Implicitly creating and using an exclusive reference
local.push_str(", world!");

// Trying to use the shared reference again
println!("{x}");
# }
```
Here, [`push_str` takes `&mut self`,](https://doc.rust-lang.org/std/string/struct.String.html#method.push_str)
so an implicit `&mut local` exists as part of the method call,
and thus the example can still not compile.

### Creating a `&mut` is not the only exclusive use

The borrow checker looks at *every* use of a value to see if it's
compatible with the lifetimes of borrows to that value, not
just uses that involve references or just uses that involve lifetimes.

For example, moving a value invalidates any references to the
value, as otherwise those references would dangle.

```rust
# fn main() {
let local = "Hello".to_string();

// Creating and using a shared reference
let x = &local;
println!("{x}");

// Moving the value
let _local = local;

// Trying to use the shared reference again
println!("{x}");
# }
```

### Referenced values must remain in scope

The effects of a value going out of scope are similar to moving the
value: all references are invalidated.
```rust
# fn main() {
let x;
{
    let local = "Hello".to_string();
    x = &local;
} // `local` goes out of scope here

// Trying to use the shared reference after `local` goes out of scope
println!("{x}");
# }
```

### Using `&mut self` or `&self` counts as a use of all fields

In the example below, `left` becomes invalid when we create `&self`
to call `bar`.  Because you can get a `&self.left` out of a `&self`,
this is similar to trying to intermix `&mut self.left` and `&self.left`.

```rust
#[derive(Debug)]
struct Pair {
    left: String,
    right: String,
}

impl Pair {
    fn foo(&mut self) {
        let left = &mut self.left;
        left.push_str("hi");
        self.bar();
        println!("{left}");
    }
    fn bar(&self) {
        println!("{self:?}");
    }
}
```

More generally, creating a `&mut x` or `&x` counts as a use of
everything reachable from `x`.

## Some things that compile successfully

Once you've started to get the hang of borrow errors, you might start to
wonder why certain programs are *allowed* to compile.  Here we introduce
some of the ways that Rust allows non-trivial borrowing while still being
sound.

### Independently borrowing fields

Rust tracks borrows of struct fields individually, so the borrows of
`left` and `right` below do not conflict.

```rust
# #[derive(Debug)]
# struct Pair {
#     left: String,
#     right: String,
# }
#
impl Pair {
    fn foo(&mut self) {
        let left = &mut self.left;
        let right = &mut self.right;
        left.push_str("hi");
        right.push_str("there");
        println!("{left} {right}");
    }
}
```

This capability is also called [splitting borrows.](https://doc.rust-lang.org/nomicon/borrow-splitting.html)

Note that data you access through indexing are not consider fields
per se; instead indexing is [an operation that generally borrows
all of `&self` or `&mut self`.](https://doc.rust-lang.org/std/ops/trait.Index.html)

```rust
# fn main() {
let mut v = vec![0, 1, 2];

// These two do not overlap, but...
let left = &mut v[..1];
let right = &mut v[1..];

// ...the borrow checker cannot recognize that
println!("{left:?} {right:?}");
# }
```

Usually in this case, one uses methods like [`split_at_mut`](https://doc.rust-lang.org/std/primitive.slice.html#method.split_at_mut)
in order to split the borrows instead.

Similarly to indexing, when you access something through "deref coercion", you're
exercising [the `Deref` trait](https://doc.rust-lang.org/std/ops/trait.Deref.html)
(or `DerefMut`), which borrow all of `self`.

There are also some niche cases where the borrow checker is smarter, however.
```rust
# fn main() {
// Pattern matching does understand non-overlapping slices (slices are special)
let mut v = vec![String::new(), String::new()];
let slice = &mut v[..];
if let [_left, right] = slice {
    if let [left, ..] = slice {
        left.push_str("left");
    }
    // Still usable!
    right.push_str("right");
}
# }
```
```rust
# fn main() {
// You can split borrows through a `Box` dereference (`Box` is special)
let mut bx = Box::new((0, 1));
let left = &mut bx.0;
let right = &mut bx.1;
*left += 1;
*right += 1;
# }
```

The examples are non-exhaustive 🙂.

### Reborrowing

[As mentioned before,](./st-reborrow.md) reborrows are what make `&mut` reasonable to use.
In fact, they have other special properties you can't emulate with a custom struct and
trait implementations.  Consider this example:
```rust
fn foo(s: &mut String) -> &str {
    &**s
}
```
Actually, that's too fast.  Let's change this a little bit and go step by step.
```rust
fn foo(s: &mut String) -> &str {
    let ms: &mut str = &mut **s;
    let rs: &str = &*s;
    rs
}
```
Here, both `s` and `ms` are going out of scope at the end of `foo`, but this doesn't
invalidate `rs`.  That is, reborrowing through references can impose lifetime constraints
on the reborrow, but the reborrow is not dependent on references staying in scope!  It is
only dependent on the borrowed data.

This demonstrates that reborrowing is more powerful than nesting references.

### Shared reborrowing

When it comes to detecting conflicts, the borrow checker distinguishes between shared
reborrows and exclusive ones.  In particular, creating a shared reborrow will invalidate
any exclusive reborrows of the same value (as they are no longer exclusive).  But it will
not invalidated shared reborrows:

```rust
struct Pair {
    left: String,
    right: String,
}

impl Pair {
    fn foo(&mut self) {
        // a split borrow: exclusive reborrow, shared reborrow
        let left = &mut self.left;
        let right = &self.right;
        left.push('x');

        // Shared reborrow of all of `self`, which "covers" all fields
        let this = &*self;

        // It invalidates any exclusive reborrows, so this will fail...
        // println!("{left}");

        // But it does not invalidate shared reborrows!
        println!("{right}");
    }
}
```

### Two-phase borrows

The following code compiles:
```rust
# fn main() {
let mut v = vec![0];
let shared = &v;
v.push(shared.len());
# }
```

However, if you're aware of the order of evaluation here, it probably seems like it shouldn't.
The implicit `&mut v` should have invalidated `shared` before `shared.len()` was evaluated.
What gives?

This is the result of a feature called two-phase borrows, which is intended to make
[nested method calls](https://rust-lang.github.io/rfcs/2025-nested-method-calls.html)
more ergonomic:
```rust
# fn main() {
let mut v = vec![0];
v.push(v.len());
# }
```
[In the olden days,](https://rust.godbolt.org/z/966j4Eh1f) you would have had to write it like so:
```rust
# fn main() {
let mut v = vec![0];
let len = v.len();
v.push(len);
# }
```

The implementation slipped, which is why the first example compiles too.  How far it slipped
is hard to say, as not only is there [no specification,](https://github.com/rust-lang/rust/issues/46901)
the feature doesn't even seem to be documented 🤷.

