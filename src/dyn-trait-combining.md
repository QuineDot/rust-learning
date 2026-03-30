# Combining traits

Rust has no support for directly combining multiple non-auto traits
into one `dyn Trait1 + Trait2`:
```rust,compile_fail
trait Foo { fn foo(&self) {} }
trait Bar { fn bar(&self) {} }

// Fails
let _: Box<dyn Foo + Bar> = todo!();
```

However, the methods of a supertrait are available to the subtrait.
What's a supertrait?  A supertrait is a trait bound on `Self` in the
definition of the subtrait, like so:
```rust
# trait Foo { fn foo(&self) {} }
# trait Bar { fn bar(&self) {} }
trait Subtrait: Foo
//    ^^^^^^^^^^^^^  A supertrait bound
where
    Self: Bar,
//  ^^^^^^^^^ Another one
{}
```
The supertrait bound is implied everywhere the subtrait bound is
present, and the methods of the supertrait are always available on
implementors of the subtrait.

Using these relationships, you can support something analogous to
`dyn Foo + Bar` by using `dyn Subtrait`.

```rust
# trait Foo { fn foo(&self) {} }
# trait Bar { fn bar(&self) {} }
# impl Foo for () {}
# impl Bar for () {}
trait Subtrait: Foo + Bar {}

// Blanket implement for everything that meets the bounds...
// ...including non-`Sized` types
impl<T: ?Sized> Subtrait for T where T: Foo + Bar {}

fn main() {
    let quz: &dyn Subtrait = &();
    quz.foo();
    quz.bar();
}
```

Note that despite the terminology, there is no sub/super *type* relationship
between sub/super traits, between `dyn SubTrait` and `dyn SuperTrait`,
between implementors of said traits, et cetera.
[Traits are not about sub/super typing.](./dyn-trait-overview.md#dyn-trait-is-not-a-supertype)

## Manual supertrait upcasting

[Supertrait upcasting](./dyn-trait-coercions.md#supertrait-upcasting) stabilized in Rust 1.86.
Before then, it was still possible, but you had to supply the implementation yourself.  This
section walks through how to supply the implementation yourself, as
- it's still applicable if you must support a version before Rust 1.86
- the general pattern of a supertrait with a blanket implementation is common
- having methods for upcasting can be more ergonomic than coercions

But if compatibility or compatibility isn't a concern, you can skip the rest of this page.
The language level supertrait upcasting is probably adequate for your needs.

On to the manual implementation!  For a start, we could build upcasting directly into our traits like so:
```rust
trait Foo {
    fn foo(&self) {}
    fn as_dyn_foo(&self) -> &dyn Foo;
}
```

But we can't supply a default function body, as `Self: Sized` is required to perform
the type erasing cast to `dyn Foo`.  We don't want that restriction or the method
won't be available on `dyn Supertrait`, which is not `Sized`.  It would also be
annoying if every implementation had to supply the function body, though.

So instead we can separate out the method and supply an implementation for all `Sized`
types, via another supertrait:
```rust
trait AsDynFoo {
    fn as_dyn_foo(&self) -> &dyn Foo;
}

trait Foo: AsDynFoo { fn foo(&self) {} }
```
And then supply the implementation for all `Sized + Foo` types:
```rust
# trait AsDynFoo { fn as_dyn_foo(&self) -> &dyn Foo; }
# trait Foo: AsDynFoo { fn foo(&self) {} }
impl<T: /* Sized + */ Foo> AsDynFoo for T {
    fn as_dyn_foo(&self) -> &dyn Foo {
        self
    }
}
```
The compiler will supply the implementation for both `dyn AsDynFoo` and `dyn Foo`.

When we put this altogether with the `Subtrait` from above, we can now utilize
an explicit version of supertrait upcasting:
```rust
trait Foo: AsDynFoo { fn foo(&self) {} }
trait Bar: AsDynBar { fn bar(&self) {} }

impl Foo for () {}
impl Bar for () {}

trait AsDynFoo { fn as_dyn_foo(&self) -> &dyn Foo; }
trait AsDynBar { fn as_dyn_bar(&self) -> &dyn Bar; }
impl<T: Foo> AsDynFoo for T { fn as_dyn_foo(&self) -> &dyn Foo { self } }
impl<T: Bar> AsDynBar for T { fn as_dyn_bar(&self) -> &dyn Bar { self } }

trait Subtrait: Foo + Bar {}
impl<T: ?Sized> Subtrait for T where T: Foo + Bar {}

fn main() {
    let quz: &dyn Subtrait = &();
    quz.foo();
    quz.bar();
    let _: &dyn Foo = quz.as_dyn_foo();
    let _: &dyn Bar = quz.as_dyn_bar();
}
```

For comparison, let's see how much cleaner built-in trait upcasting is:
```rust
trait Foo { fn foo(&self) {} }
trait Bar { fn bar(&self) {} }

impl Foo for () {}
impl Bar for () {}

trait Subtrait: Foo + Bar {}
impl<T: ?Sized> Subtrait for T where T: Foo + Bar {}

fn main() {
    let quz: &dyn Subtrait = &();
    quz.foo();
    quz.bar();
    let _: &dyn Foo = quz;
    let _: &dyn Bar = quz;
}
```

Less supertrait and blanket implementation noise, and direct coercions instead
of having to go through a method call.  Definitely an improvement to this example!

[We will later see](./dyn-trait-eq.md#comparison-with-manual-supertrait-upcasting)
that having the method can be more ergnomic due to method chaining in some
circumstances, however.
