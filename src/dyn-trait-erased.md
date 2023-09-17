# Erased traits

Let's say you have an existing trait which works well for the most part:
```rust
pub mod useful {
    pub trait Iterable {
        type Item;
        type Iter<'a>: Iterator<Item = &'a Self::Item> where Self: 'a;
        fn iter(&self) -> Self::Iter<'_>;
        fn visit<F: FnMut(&Self::Item)>(&self, mut f: F) {
            for item in self.iter() {
                f(item);
            }
        }
    }

    impl<I: Iterable + ?Sized> Iterable for &I {
        type Item = <I as Iterable>::Item;
        type Iter<'a> = <I as Iterable>::Iter<'a> where Self: 'a;
        fn iter(&self) -> Self::Iter<'_> {
            <I as Iterable>::iter(*self)
        }
    }

    impl<I: Iterable + ?Sized> Iterable for Box<I> {
        type Item = <I as Iterable>::Item;
        type Iter<'a> = <I as Iterable>::Iter<'a> where Self: 'a;
        fn iter(&self) -> Self::Iter<'_> {
            <I as Iterable>::iter(&**self)
        }
    }

    impl<T> Iterable for Vec<T> {
        type Item = T;
        type Iter<'a> = std::slice::Iter<'a, T> where Self: 'a;
        fn iter(&self) -> Self::Iter<'_> {
            <[T]>::iter(self)
        }
    }
}
```

However, it's not [`dyn` safe](./dyn-safety.md) and you wish it was.
Even if we get support for GATs in `dyn Trait` some day, there
are no plans to support functions with generic type parameters
like `Iterable::visit`.  Besides, you want the functionality now,
not "some day".

Perhaps you also have a lot of code utilizing this useful trait,
and you don't want to redo everything.  Maybe it's not even your
own trait.

This may be a case where you want to provide an "erased" version
of the trait to make it `dyn` safe.  The general idea is to use
`dyn` (type erasure) to replace all the non-`dyn`-safe uses such
as GATs and type-parameterized methods.

```rust
pub mod erased {
    // This trait is `dyn` safe
    pub trait Iterable {
        type Item;
        // No more GAT
        fn iter(&self) -> Box<dyn Iterator<Item = &Self::Item> + '_>;
        // No more type parameter
        fn visit(&self, f: &mut dyn FnMut(&Self::Item));
    }
}
```

We want to be able to create a `dyn erased::Iterable` from anything
that is `useful::Iterable`, so we need a blanket implementation to
connect the two:
```rust
#fn main() {}
#pub mod useful {
#    pub trait Iterable {
#        type Item;
#        type Iter<'a>: Iterator<Item = &'a Self::Item> where Self: 'a;
#        fn iter(&self) -> Self::Iter<'_>;
#        fn visit<F: FnMut(&Self::Item)>(&self, f: F);
#    }
#}
pub mod erased {
    use crate::useful;
#    pub trait Iterable {
#        type Item;
#        fn iter(&self) -> Box<dyn Iterator<Item = &Self::Item> + '_>;
#        fn visit(&self, f: &mut dyn FnMut(&Self::Item)) {
#            for item in self.iter() {
#                f(item);
#            }
#        }
#    }

    impl<I: useful::Iterable + ?Sized> Iterable for I {
        type Item = <I as useful::Iterable>::Item;
        fn iter(&self) -> Box<dyn Iterator<Item = &Self::Item> + '_> {
            Box::new(useful::Iterable::iter(self))
        }
        // By not using a default function body, we can avoid
        // boxing up the iterator
        fn visit(&self, f: &mut dyn FnMut(&Self::Item)) {
            for item in <Self as useful::Iterable>::iter(self) {
                f(item)
            }
        }
    }
}
```

We're also going to want to pass our `erased::Iterable`s to functions
that have a `useful::Iterable` trait bound.  However, we can't add
that as a supertrait, because that would remove the `dyn` safety.

The purpose of our `erased::Iterable` is to be able to type-erase to
`dyn erased::Iterable` anyway though, so instead we just implement
`useful::Iterable` directly on `dyn erased::Iterable`:
```rust
#fn main() {}
#pub mod useful {
#    pub trait Iterable {
#        type Item;
#        type Iter<'a>: Iterator<Item = &'a Self::Item> where Self: 'a;
#        fn iter(&self) -> Self::Iter<'_>;
#        fn visit<F: FnMut(&Self::Item)>(&self, f: F);
#    }
#}
pub mod erased {
    use crate::useful;
#    pub trait Iterable {
#        type Item;
#        fn iter(&self) -> Box<dyn Iterator<Item = &Self::Item> + '_>;
#        fn visit(&self, f: &mut dyn FnMut(&Self::Item)) {
#            for item in self.iter() {
#                f(item);
#            }
#        }
#    }
#    impl<I: useful::Iterable + ?Sized> Iterable for I {
#        type Item = <I as useful::Iterable>::Item;
#        fn iter(&self) -> Box<dyn Iterator<Item = &Self::Item> + '_> {
#            Box::new(useful::Iterable::iter(self))
#        }
#        fn visit(&self, f: &mut dyn FnMut(&Self::Item)) {
#            for item in <Self as useful::Iterable>::iter(self) {
#                f(item)
#            }
#        }
#    }

    impl<Item> useful::Iterable for dyn Iterable<Item = Item> + '_ {
        type Item = Item;
        type Iter<'a> = Box<dyn Iterator<Item = &'a Item> + 'a> where Self: 'a;
        fn iter(&self) -> Self::Iter<'_> {
            Iterable::iter(self)
        }
        // Here we can choose to override the default function body to avoid
        // boxing up the iterator, or we can use the default function body
        // to avoid dynamic dispatch of `F`.  I've opted for the former.
        fn visit<F: FnMut(&Self::Item)>(&self, mut f: F) {
            <Self as Iterable>::visit(self, &mut f)
        }
    }
}
```

Technically our blanket implementation of `erased::Iterable` now applies to
`dyn erased::Iterable`, but things still work out due to some
[language magic.](./dyn-trait-impls.md#the-implementation-cannot-be-directly-overrode)

The blanket implementations of `useful::Iterable` in the `useful` module gives
us implementations for `&dyn erased::Iterable` and `Box<dyn erased::Iterable>`,
so now we're good to go!

## Mindful implementations and their limitations

You may have noticed how we took care to avoid boxing the iterator when possible
by being mindful of how we implemented some of the methods, for example not
having a default body for `erased::Iterable::visit`, and then overriding the
default body of `useful::Iterable::visit`.  This can lead to better performance
but isn't necessarily critical, so long as you avoid things like accidental
infinite recursion.

How well did we do on this front?
[Let's take a look in the playground.](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=31dbe5ae8a7a6b7677ed942962424e03)

Hmm, perhaps not as well as we hoped!  `<dyn erased::Iterable as useful::Iterable>::visit`
avoids the boxing as designed, but `Box<dyn erased::Iterable>`'s `visit` still boxes the
iterator.

Why is that?  It is because the implementation for the `Box` is supplied by the `useful`
module, and that implementation uses the default body.  In order to avoid the boxing,
[it would need to recurse to the underlying implementation instead.](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=02ae6370cdd7f7b5b1586e0281e090d2)
That way, the call to `visit` will "drill down" until the implementation for
`dyn erased::Iterable::visit`, which takes care to avoid the boxed iterator.  Or
phrased another way, the recursive implementations "respects" any overrides of the
default function body by other implementors of `useful::Iterable`.

Since the original trait might not even be in your crate, this might be out of your
control.  Oh well, so it goes; maybe submit a PR 🙂.  In this particular case you
could take care to pass `&dyn erased::Iterable` by coding `&*boxed_erased_iterable`.

Or maybe it doesn't really matter enough to bother in practice for your use case.

## Real world examples

Perhaps the most popular crate to use this pattern is
[the `erased-serde` crate.](https://crates.io/crates/erased-serde)

Another use case is [working with `async`-esque traits,](https://smallcultfollowing.com/babysteps/blog/2021/10/15/dyn-async-traits-part-6/)
which tends to involve a lot of type erasure and unnameable types.
