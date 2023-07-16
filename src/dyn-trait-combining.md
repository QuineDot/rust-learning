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

[Supertrait upcasting is planned, but not yet stable.](./dyn-trait-coercions.md#supertrait-upcasting)
Until stabilized, if you need to cast something like `dyn Subtrait` to `dyn Foo`, you
have to supply the implementation yourself.

For a start, we could build it into our traits like so:
```rust
trait Foo {
    fn foo(&self) {}
    fn as_dyn_foo(&self) -> &dyn Foo;
}
```

But we can't supply a default function body, as `Self: Sized` is required to perform
the type erasing cast to `dyn Foo`.  We don't want that restriction or the method
won't be available on `dyn Supertrait`, which is not `Sized`.

Instead we can separate out the method and supply an implementation for all `Sized`
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
