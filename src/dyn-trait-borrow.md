# Generalizing borrows

A lot of core traits are built around some sort of *field projection,* where
the implementing type contains some other type `T` and you can convert a
`&self` to a `&T` or `&mut self` to a `&mut T`.
```rust
pub trait Deref {
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}

pub trait Index<Idx: ?Sized> {
    type Output: ?Sized;
    fn index(&self, index: Idx) -> &Self::Output;
}

pub trait AsRef<T: ?Sized> {
    fn as_ref(&self) -> &T;
}

pub trait Borrow<Borrowed: ?Sized> {
    fn borrow(&self) -> &Borrowed;
}

// `DerefMut`, `IndexMut`, `AsMut`, ...
```

There's generally no way to implement these traits if the type you want to
return is not contained within `Self` (except for returning a reference to
some static value or similar, which is rarely what you want).

However, sometimes you have a custom borrowing type which is *not*
actually contained within your owning type:
```rust
// We wish we could implement `Borrow<DataRef<'?>>`, but we can't
pub struct Data {
    first: usize,
    others: Vec<usize>,
}

pub struct DataRef<'a> {
    first: usize,
    others: &'a [usize],
}

pub struct DataMut<'a> {
    first: usize,
    others: &'a mut Vec<usize>,
}
```

This can be problematic when interacting with libraries and data structures
such as [`std::collections::HashSet`,](https://doc.rust-lang.org/std/collections/struct.HashSet.html)
which [rely on the `Borrow` trait](https://doc.rust-lang.org/std/collections/struct.HashSet.html#method.contains)
to be able to look up entries without taking ownership.

One way around this problem is to use
[a different library or type which is more flexible.](https://docs.rs/hashbrown/0.14.0/hashbrown/struct.HashSet.html#method.contains)
However, it's also possible to tackle the problem with a bit of indirection and type erasure.

## Your types contain a borrower

Here we present a solution to the problem by
[Eric Michael Sumner](https://orcid.org/0000-0002-6439-9757), who has
graciously blessed its inclusion in this guide.  I've rewritten the
original for the sake of presentation, and any errors are my own.

The main idea behind the approach is to utilize the following trait, which
encapsulates the ability to borrow `self` in the form of your custom borrowed
type:
```rust
#pub struct Data { first: usize, others: Vec<usize> }
#pub struct DataRef<'a> { first: usize, others: &'a [usize] }
pub trait Lend {
    fn lend(&self) -> DataRef<'_>;
}

impl Lend for Data {
    fn lend(&self) -> DataRef<'_> {
        DataRef {
            first: self.first,
            others: &self.others,
        }
    }
}

impl Lend for DataRef<'_> {
    fn lend(&self) -> DataRef<'_> {
        DataRef {
            first: self.first,
            others: self.others,
        }
    }
}

// impl Lend for DataMut<'_> ...
```

And the key insight is that any implementor can also coerce from
`&self` to `&dyn Lend`.  We can therefore implement traits like
`Borrow`, because every implementor "contains" a `dyn Lend` --
themselves!

```rust
#pub struct Data { first: usize, others: Vec<usize> }
#pub struct DataRef<'a> { first: usize, others: &'a [usize] }
#pub trait Lend { fn lend(&self) -> DataRef<'_>; }
#impl Lend for Data {
#    fn lend(&self) -> DataRef<'_> {
#        DataRef {
#            first: self.first,
#            others: &self.others,
#        }
#    }
#}
#impl Lend for DataRef<'_> {
#    fn lend(&self) -> DataRef<'_> {
#        DataRef {
#            first: self.first,
#            others: self.others,
#        }
#    }
#}
use std::borrow::Borrow;

impl<'a> Borrow<dyn Lend + 'a> for Data {
    fn borrow(&self) -> &(dyn Lend + 'a) { self }
}

impl<'a, 'b: 'a> Borrow<dyn Lend + 'a> for DataRef<'b> {
    fn borrow(&self) -> &(dyn Lend + 'a) { self }
}

// impl<'a, 'b: 'a> Borrow<dyn Lend + 'a> for DataMut<'b> ...
```

This gives us a common `Borrow` type for both our owning and
custom borrowing data structures.  To look up borrowed entries
in a `HashSet`, for example, we can cast a `&DataRef<'_>` to
a `&dyn Lend` and pass that to `set.contains`; the `HashSet` can
hash the `dyn Lend` and then borrow the owned `Data` entries as
`dyn Lend` as well, in order to do the necessary lookup
comparisons.

That means we need to implement the requisite functionality such as
`PartialEq` and `Hash` for `dyn Lend`.  But this is a different use case
than [our general solution in the previous section.](./dyn-trait-eq.md)
In that case we wanted `PartialEq` for our already-type-erased `dyn Trait`,
so we could compare values across any arbitrary implementing types.

Here we don't care about arbitrary types, and we also have the ability
to produce a concrete type that references our actual data.  We can
use that to implement the functionality; there's no need for downcasting
or any of that in order to implement the requisite traits for `dyn Lend`.
We don't really *care* that `dyn Lend` will implement `PartialEq` and
`Hash` per se, as that is just a means to an end: giving `HashSet` and
friends a way to compare our custom concrete borrowing types despite the
`Borrow` trait bound.

First things first though, we need our concrete types to implement
the requisite traits themselves.  The main thing to be mindful of
is that we maintain
[the invariants expected by `Borrow`.](https://doc.rust-lang.org/std/borrow/trait.Borrow.html)
For this example, we're lucky enough that our borrowing
type can just derive all of the requisite functionality:
```rust
#[derive(Debug, Copy, Clone, Eq, PartialEq, Hash)]
pub struct DataRef<'a> {
    first: usize,
    others: &'a [usize],
}

#[derive(Debug, Clone)]
pub struct Data {
    first: usize,
    others: Vec<usize>,
}

#[derive(Debug)]
pub struct DataMut<'a> {
    first: usize,
    others: &'a mut Vec<usize>,
}
```
However, we haven't derived the traits that are semantically important
to `Borrow` for our other types.  We technically could have in
this case, because
- our fields are in the same order as they are in the borrowed type
- every field is present
- every field has a `Borrow` relationship when comparing with the borrowed type's field
- we understand how the `derive` works

But all those things might not be true for your use case, and even
when they are, relying on them creates a very fragile arrangement.
It's just too easy to accidentally break things by adding a field
or even just rearranging the field order.

Instead, we implement the traits directly by deferring to the borrowed type:
```rust
# struct Data; impl Data { fn lend(&self) {} }
// Excercise for the reader: `PartialEq` across all of our
// owned and borrowed types :-)
impl std::cmp::PartialEq for Data {
    fn eq(&self, other: &Self) -> bool {
        self.lend() == other.lend()
    }
}

impl std::cmp::Eq for Data {}

impl std::hash::Hash for Data {
    fn hash<H: std::hash::Hasher>(&self, hasher: &mut H) {
        self.lend().hash(hasher)
    }
}

// Similarly for `DataMut<'_>`
```
And in fact, this is exactly the approach we want to take for
`dyn Lend` as well:
```rust
#pub struct DataRef<'a> { first: usize, others: &'a [usize] }
#pub trait Lend { fn lend(&self); }
impl std::cmp::PartialEq<dyn Lend + '_> for dyn Lend + '_ {
    fn eq(&self, other: &(dyn Lend + '_)) -> bool {
        self.lend() == other.lend()
    }
}

impl std::cmp::Eq for dyn Lend + '_ {}

impl std::hash::Hash for dyn Lend + '_ {
    fn hash<H: std::hash::Hasher>(&self, hasher: &mut H) {
        self.lend().hash(hasher)
    }
}
```

Whew, that was a lot of boilerplate.  But we're finally at a
place where we can store `Data` in a `HashSet` and look up
entries when we only have a `DataRef`:
```rust,ignore
use std::collections::HashSet;
let set = [
    Data { first: 3, others: vec![5,7]},
].into_iter().collect::<HashSet<_>>();

assert!(set.contains::<dyn Lend>(&DataRef { first: 3, others: &[5,7]}))

// Alternative to turbofishing
let data_ref = DataRef { first: 3, others: &[5,7]};
assert!(set.contains(&data_ref as &dyn Lend));
```

[Here's a playground with the complete example.](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=cae538dbbf73e3e7692135bf2d397f39)

Another alternative to casting or turbofish is to add an
`as_lend(&self) -> &dyn Lend + '_` method, similar to many
of the previous examples.
