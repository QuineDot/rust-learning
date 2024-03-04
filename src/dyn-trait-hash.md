# Hashable `Box<dyn Trait>`

Let's say we have a [`dyn Trait` that implements `Eq`](./dyn-trait-eq.md), but we also want it to implement `Hash`
so that we can use `Box<dyn Trait>` in a `HashSet` or as the key of a `HashMap` and so on.

<details>
<summary>Here's our starting point</summary>

The only update from [before](./dyn-trait-eq.md) is to require `Eq`:
```diff
-impl<T: Any + PartialOrd> DynCompare for T {
+impl<T: Any + PartialOrd + Eq> DynCompare for T {

+impl Eq for dyn DynCompare {}
+impl Eq for dyn Trait {}
```

The complete code:
```rust
use core::cmp::Ordering;
use std::any::Any;

trait AsDynCompare: Any {
    fn as_any(&self) -> &dyn Any;
    fn as_dyn_compare(&self) -> &dyn DynCompare;
}

// Sized types only
impl<T: Any + DynCompare> AsDynCompare for T {
    fn as_any(&self) -> &dyn Any {
        self
    }
    fn as_dyn_compare(&self) -> &dyn DynCompare {
        self
    }
}

trait DynCompare: AsDynCompare {
    fn dyn_eq(&self, other: &dyn DynCompare) -> bool;
    fn dyn_partial_cmp(&self, other: &dyn DynCompare) -> Option<Ordering>;
}

impl<T: Any + PartialOrd + Eq> DynCompare for T {
    fn dyn_eq(&self, other: &dyn DynCompare) -> bool {
        if let Some(other) = other.as_any().downcast_ref::<Self>() {
            self == other
        } else {
            false
        }
    }

    fn dyn_partial_cmp(&self, other: &dyn DynCompare) -> Option<Ordering> {
        other
            .as_any()
            .downcast_ref::<Self>()
            .and_then(|other| self.partial_cmp(other))
    }
}

impl Eq for dyn DynCompare {}
impl PartialEq<dyn DynCompare> for dyn DynCompare {
    fn eq(&self, other: &dyn DynCompare) -> bool {
        self.dyn_eq(other)
    }
}

impl PartialOrd<dyn DynCompare> for dyn DynCompare {
    fn partial_cmp(&self, other: &dyn DynCompare) -> Option<Ordering> {
        self.dyn_partial_cmp(other)
    }
}

trait Trait: DynCompare {}
impl Trait for i32 {}
impl Trait for bool {}

impl Eq for dyn Trait {}
impl PartialEq<dyn Trait> for dyn Trait {
    fn eq(&self, other: &dyn Trait) -> bool {
        self.as_dyn_compare() == other.as_dyn_compare()
    }
}

impl PartialOrd<dyn Trait> for dyn Trait {
    fn partial_cmp(&self, other: &dyn Trait) -> Option<Ordering> {
        self.as_dyn_compare().partial_cmp(other.as_dyn_compare())
    }
}

impl PartialEq<&Self> for Box<dyn Trait> {
    fn eq(&self, other: &&Self) -> bool {
        <Self as PartialEq>::eq(self, *other)
    }
}

impl PartialOrd<&Self> for Box<dyn Trait> {
    fn partial_cmp(&self, other: &&Self) -> Option<Ordering> {
        <Self as PartialOrd>::partial_cmp(self, *other)
    }
}
```

---

</details>

Similarly to [when we looked at `Clone`,](./dyn-trait-clone.md) `Hash` is not an object safe trait.  So we can't
just add `Hash` as a supertrait bound.  This time it's not because it requires `Sized`, though, it's because
[it has a generic method:](https://doc.rust-lang.org/std/hash/trait.Hash.html#tymethod.hash)
```rust
# use core::hash::Hasher;
pub trait Hash {
    fn hash<H>(&self, state: &mut H)
       where H: Hasher;
}
```

Fortunately, we can use an [erased trait](./dyn-trait-erased.md) approach:

```rust
use std::collections::HashSet;
use core::hash::{Hash, Hasher};

trait DynHash {
    fn dyn_hash(&self, state: &mut dyn Hasher);
}

// impl<T: ?Sized + Hash> DynHash for T {
impl<T: Hash> DynHash for T {
    fn dyn_hash(&self, mut state: &mut dyn Hasher) {
        self.hash(&mut state)
    }
}

impl Hash for dyn DynHash + '_ {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.dyn_hash(state)
    }
}
```
Now is a good time to point out a couple of things we're relying on:
- [`Hasher`](https://doc.rust-lang.org/std/hash/trait.Hasher.html) *is* object safe
    <br><br>If this wasn't the case, we couldn't take a `&mut dyn Hasher` in our `dyn_hash` method.

- The generic `H` in `Hash::hash<H: Hasher>` has an implicit `Sized` bound
    <br><br>If this wasn't the case, we couldn't coerce the `&mut H` to a `&mut dyn Hasher`
    in our implementation of `Hash` for `dyn DynHash`.

This demonstrates that *relaxing* a `Sized` bound can be a breaking change!


Moving on, we still need to wire up this new functionality to our own trait.
```diff
-trait Trait: DynCompare;
+trait Trait: DynCompare + DynHash {}
```
```rust
# use core::hash::{Hash, Hasher};
# trait DynHash { fn dyn_hash(&self, _: &mut dyn Hasher); }
# trait Trait: DynHash {}
// Same as what we did for `dyn DynHash`
impl Hash for dyn Trait {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.dyn_hash(state)
    }
}
```
[And that's it!](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=de3908f78bbeeb4da109ca0fa8e1d53b)
```rust,ignore
    let bx1a: Box<dyn Trait> = Box::new(1);
    let bx1b: Box<dyn Trait> = Box::new(1);
    let bx2: Box<dyn Trait> = Box::new(2);
    let bx3: Box<dyn Trait> = Box::new(true);

    let hm: HashSet<_> = HashSet::from_iter([bx1a, bx1b, bx2, bx3].into_iter());
    assert_eq!(hm.len(), 3);

```

## Closing remarks

[Borrowing a concrete type](dyn-trait-borrow.html) is probably a better approach if it applies
to your use case, since it doesn't require `Any + 'static`.

Although terribly inefficient, an implementation of `Hash` that returns the hash for everything
is a correct implementation.  So are other "rough approximations", like if we only hashed the
`TypeId`.  All that's required for logical behavior is that two equal values must also have
equal hashes.  So arguably we didn't need to go to such lengths to get the exact hashes of the
values.

You may have noticed this commented out line:
```rust,ignore
// impl<T: ?Sized + Hash> DynHash for T {
impl<T: Hash> DynHash for T {
```

The reason I went with the less general implementation is two-fold:
- It wasn't needed for the example
- Prudency: because we `impl Hash for dyn DynHash`, it technically overlaps with the compiler's implementation of `DynHash for dyn DynHash`.
[See the final paragraph of this subsection.](http://localhost:3000/dyn-trait-impls.html#the-implementation-cannot-be-directly-overrode)


