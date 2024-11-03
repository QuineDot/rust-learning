# Mutable slice iterator

The standard library has [an iterator over `&mut [T]`](https://doc.rust-lang.org/std/slice/struct.IterMut.html)
which is implemented (as of this writing) in terms of pointer arithmetic, presumably for the sake of optimization.
In this example, we'll show how one can implement their own mutable slice iterator with entirely safe code.

Here's the starting place for our implementation:
```rust
struct MyIterMut<'a, T> {
    slice: &'a mut [T],
    // ...maybe other fields for your needs...
}

impl<'a, T> Iterator for MyIterMut<'a, T> {
    type Item = &'a mut T;
    fn next(&mut self) -> Option<Self::Item> {
        todo!()
    }
}
```
Below are a few starting attempts at it.  Spoilers, they don't compile.
```rust,compile_fail
#struct MyIterMut<'a, T> { slice: &'a mut [T], }
# impl<'a, T> Iterator for MyIterMut<'a, T> {
#   type Item = &'a mut T;
    fn next(&mut self) -> Option<Self::Item> {
        // Eh, we'll worry about iterative logic later!
        self.slice.get_mut(0)
    }
#}
```
```rust,compile_fail
#struct MyIterMut<'a, T> { slice: &'a mut [T], }
# impl<'a, T> Iterator for MyIterMut<'a, T> {
#   type Item = &'a mut T;
    fn next(&mut self) -> Option<Self::Item> {
        // Actually, this method looks perfect for our iteration logic
        let (first, rest) = self.slice.split_first_mut()?;
        self.slice = rest;
        Some(first)
    }
#}
```
```rust,compile_fail
#struct MyIterMut<'a, T> { slice: &'a mut [T], }
# impl<'a, T> Iterator for MyIterMut<'a, T> {
#    type Item = &'a mut T;
    fn next(&mut self) -> Option<Self::Item> {
        // 🤔 Pattern matching??
        match &mut self.slice {
            [] => None,
            [first, rest @ ..] => Some(first),
        }
    }
#}
```

Yeah, the compiler really doesn't like any of that.  Let's take a minute to write out all the elided
lifetimes.  Some of them are in aliases, which we're also going to expand:
- `Item` is `&'a mut T`
- `&mut self` is short for `self: &mut Self`, and
  - `Self` is `MyIterMut<'a, T>`

Here's what it looks like with everything being explicit:
```rust
#struct MyIterMut<'a, T> { slice: &'a mut [T], }
# impl<'a, T> Iterator for MyIterMut<'a, T> {
#   type Item = &'a mut T;
    fn next<'s>(self: &'s mut MyIterMut<'a, T>) -> Option<&'a mut T> {
        todo!()
    }
#}
```
And remember that in `MyIterMut<'a, T>`, `slice` is a `&'a mut [T]`.

Ah, yes.  [We have a nested exclusive borrow](./st-invariance.md) here.

> You cannot get a `&'long mut U` through dereferencing a `&'short mut &'long mut U`.
> - You can only reborrow a `&'short mut U`.

There is no safe way to go *through* the `&'s mut self` and pull out a `&'a mut T`.

Are we stuck then?  No, there is actually a way forward! As it turns out,
slices are special.  In particular, the compiler understands that an *empty* slice
covers *no actual data*, so there can't be any memory aliasing concerns or data races,
et cetera.  So the compiler understands it's perfectly sound to pull an empty slice
reference out of no where, with any lifetime at all.  Even if it's an exclusive slice reference!
```rust
fn magic<T>() -> &'static mut [T] {
    &mut []
}
```

For our purposes, we don't even need the magic: the standard library
[has a `Default` implementation](https://doc.rust-lang.org/std/default/trait.Default.html#impl-Default-for-%26mut+%5BT%5D)
for `&mut [T]`.

Why does this unstick us?  With that implementation, we can conjure an empty `&mut [T]`
out of nowhere and *move our `slice` field out from behind `&mut self`:*
```rust
#struct MyIterMut<'a, T> { slice: &'a mut [T], }
# impl<'a, T> Iterator for MyIterMut<'a, T> {
#   type Item = &'a mut T;
    fn next(&mut self) -> Option<Self::Item> {
        let mut slice = std::mem::take(&mut self.slice);
        // Eh, we'll worry about iterative logic later!
        slice.get_mut(0)
    }
#}
```
[`std::mem::take`](https://doc.rust-lang.org/std/mem/fn.take.html) and `swap` and `replace` are
very useful and safe functions; don't be thrown off by them being in `std::mem` along side the
dangerous `transmute` and other low-level functions.  Note how we passed `&mut self.slice` --
that's a `&mut &mut [T]`.  `take` replaces everything inside of the outer `&mut`, which can have
an arbitrarily short lifetime -- just long enough to move the memory around.

So we're done aside from iterative logic, right?  This should just give us the first element forever?
```rust
#struct MyIterMut<'a, T> { slice: &'a mut [T], }
# impl<'a, T> Iterator for MyIterMut<'a, T> {
#   type Item = &'a mut T;
#   fn next(&mut self) -> Option<Self::Item> {
#       let mut slice = std::mem::take(&mut self.slice);
#       slice.get_mut(0)
#   }
#}
let mut arr = [0, 1, 2, 3];
let iter = MyIterMut { slice: &mut arr };
for x in iter.take(10) {
    println!("{x}");
}
```
Uh, it only gave us one item.  Oh right -- when we're done with the slice, we need to move it
back into our `slice` field.  We only want to *temporarily* replace that field with an empty slice.
```rust,compile_fail
#struct MyIterMut<'a, T> { slice: &'a mut [T], }
# impl<'a, T> Iterator for MyIterMut<'a, T> {
#   type Item = &'a mut T;
    fn next(&mut self) -> Option<Self::Item> {
        let mut slice = std::mem::take(&mut self.slice);
        // Eh, we'll worry about iterative logic later!
        let first = slice.get_mut(0);
        self.slice = slice;
        first
    }
#}
```
Uh oh, now what.
```rust

error[E0499]: cannot borrow `*slice` as mutable more than once at a time
9  |         let first = slice.get_mut(0);
   |                     ----- first mutable borrow occurs here
10 |         self.slice = slice;
   |                      ^^^^^ second mutable borrow occurs here
11 |         first
   |         ----- returning this value requires that `*slice` is borrowed for `'a`
```
Oh, right!  These are *exclusive* references.  We can't return the same item multiple
times -- that would mean someone could get multiple `&mut` to the same element if they
[collect](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect)ed the
iterator, for example.  Come to think of it, we can't punt on our iteration logic
either -- if we try to hold on to the entire `&mut [T]` while handing out `&mut T`
to the elements, that's *also* multiple `&mut` to the same memory!

This is what the error is telling us: We can't hold onto the entire `slice` and
return `first`.

(There's a pattern called "lending iterators" where you can hand out borrows of
data you own in an iterator-like fashion, but it's not possible with the current
`Iterator` trait; it is also a topic for another day.)

Alright, let's try [`split_first_mut`](https://doc.rust-lang.org/std/primitive.slice.html#method.split_first_mut)
again instead, that really did seem like a perfect fit for our iteration logic.
```rust
#struct MyIterMut<'a, T> { slice: &'a mut [T], }
# impl<'a, T> Iterator for MyIterMut<'a, T> {
#   type Item = &'a mut T;
    fn next(&mut self) -> Option<Self::Item> {
        let mut slice = std::mem::take(&mut self.slice);
        let (first, rest) = slice.split_first_mut()?;
        self.slice = rest;
        Some(first)
    }
# }

// ...

let mut arr = [0, 1, 2, 3];
let iter = MyIterMut { slice: &mut arr };
for x in iter {
    println!("{x}");
}
```

Finally, a working version!  `split_first_mut` is a form of [borrow splitting,](./lifetime-analysis.md#independently-borrowing-fields)
which we briefly mentioned before.

And for the sake of completion, here's the pattern based approach to borrow splitting:
```rust
#struct MyIterMut<'a, T> { slice: &'a mut [T], }
# impl<'a, T> Iterator for MyIterMut<'a, T> {
#   type Item = &'a mut T;
    fn next(&mut self) -> Option<Self::Item> {
        //    ....these are all that changed....
        //    vvvvvvvvvvvvvvvvvv               v
        match std::mem::take(&mut self.slice) {
            [] => None,
            [first, rest @ ..] => Some(first),
        }
     }
# }

// ...

let mut arr = [0, 1, 2, 3];
let iter = MyIterMut { slice: &mut arr };
for x in iter {
    println!("{x}");
}
```
Oh whoops, just one element again.  Right.  We need to put the `rest` back in `self.slice`:
```rust
#struct MyIterMut<'a, T> { slice: &'a mut [T], }
# impl<'a, T> Iterator for MyIterMut<'a, T> {
#   type Item = &'a mut T;
    fn next(&mut self) -> Option<Self::Item> {
        match std::mem::take(&mut self.slice) {
            [] => None,
            [first, rest @ ..] => {
                self.slice = rest;
                Some(first)
            }
        }
     }
# }

// ...

let mut arr = [0, 1, 2, 3];
let iter = MyIterMut { slice: &mut arr };
for x in iter {
    println!("{x}");
}
```
👍

