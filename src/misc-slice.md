# Slice layout

It's not uncommon for people on [the forum](https://users.rust-lang.org/)
to ask why it's conventional to have `&[T]` as an argument insteaed of
`&Vec<T>`, or to ask about the layout of slices more generally.  Or to
ask analogous questions about `&str` and `String`, et cetera.

This page exists to be a useful citation for such questions.

If you want, you can skip ahead to the [graphical layout.](#graphical-layout)

## What is a slice anyway?

The terminology around slices tends to be pretty loose.  I'll try to keep
it more formal on this page, but when you read something about a "slice"
elsewhere, keep in mind that it may be referring to any of `[T]`, `&[T]`,
`&mut [T]`, or even other types of references to `[T]` (`Box<[T]>`, `Arc<[T]>`, ...).

This is the case not just for casual material, but for
[official documentation](https://doc.rust-lang.org/std/primitive.slice.html)
and other technical material.  You just have to figure out which one or
ones they are specifically talking about from context.

With that out of the way, let me intoduce some terminology for this page:

- A slice, `[T]`, is a series of `T` in contiguous memory (layed out one after
another, with proper alignment). The length is only known at run time; we say
it is a dynamically sized type (DST), or an unsized type, or a type that does
not implement [`Sized`](https://doc.rust-lang.org/std/marker/trait.Sized.html).

- A shared slice, `&[T]`, is a shared reference to a slice. It's a wide
reference consisting of a pointer to the memory of the slice, and the number
of elements in the slice.

- An exclusive slice, `&mut [T]`, is like a shared slice, but the borrow is
exclusive (so you can e.g. overwrite elements through it).

- There are other wide pointer variations like boxed slices (`Box<[T]>`) and
so on; we'll mention a few more momentarily.

Note that while slices are unsized, the wide pointers to slices (like `&[T]`)
are sized.

### Where is a slice?

Slices can be on the heap, but also on the stack, or in static memory, or
anywhere else.  The type doesn't "care" where it is.  Therefore, you can't
be sure where a pointer to a slice points unless the pointer itself has
further guarantees.

For example, if you have a `Box<[T]>`, then any `T` within are on the
heap, because that's a guarantee of
[`Box<_>`.](https://doc.rust-lang.org/std/boxed/struct.Box.html)
(N.b. if `T` is zero sized, they are not actually stored anywhere.)
So in that *particular* case, we could say the slice `[T]` is on the heap.

## What is a `Vec<_>` anyway?

A [`Vec<T>`](https://doc.rust-lang.org/std/vec/struct.Vec.html) is growable buffer
that owns and stores `T`s in contiguous memory, on the heap.  You can conceptually
think of this as something that owns a slice `[T]` (or more accurately,
`[MaybeUninit<T>]`). You can index into a `Vec<T>` with a range and get back a
shared or exclusive slice.

[A `Vec<_>` consists of a pointer, capacity, and length.](https://doc.rust-lang.org/std/vec/struct.Vec.html#guarantees)

## Other types

A [`String`](https://doc.rust-lang.org/std/string/struct.String.html) is, under the hood,
like a `Vec<u8>` which has additional guarantees -- namely, that the bytes are valid UTF8.
A [`&str`](https://doc.rust-lang.org/std/primitive.str.html) is like a `&[u8]` that has the
same guarantee.  You can index into a `String` with a range and get back a `&str`
(or `&mut str`). Like `[u8]`, a `str` is unsized, which is why you're almost always working
with a `&str` or other pointer instead.

So the relationship betweeen `str` and `String` is the same as between `[T]` and `Vec<T>`.
There are other pairs of types with the same relationship:
- [`Path`](https://doc.rust-lang.org/std/path/struct.Path.html) and [`PathBuf`](https://doc.rust-lang.org/std/path/struct.PathBuf.html)
- [`OsStr`](https://doc.rust-lang.org/std/ffi/struct.OsStr.html) and [`OsString`](https://doc.rust-lang.org/std/ffi/struct.OsString.html)
- [`CStr`](https://doc.rust-lang.org/std/ffi/struct.CStr.html) and [`CString`](https://doc.rust-lang.org/std/ffi/struct.CString.html)

These `std` types generally have a [`ToOwned`](https://doc.rust-lang.org/std/borrow/trait.ToOwned.html#implementors)
relationship and a [`Borrow`](https://doc.rust-lang.org/std/borrow/trait.Borrow.html#implementors) relationship.

Even more data structures that can be considered a form of *owned* slices include:
- [`[T; N]`](https://doc.rust-lang.org/std/primitive.array.html) is an array with a
compile-time known length (i.e. it's a fixed-size array).  It is like a slice (`[T]`),
but it is `Sized`, as the the length is known at compile time.  The length is also part
of the type. It's not growable.

- `Box<[T]>`, a "boxed slice"; this is similar to a `Vec<T>` in that it owns the `T`
and stores them contiguously on the heap.  Unlike a `Vec<T>`, the buffer is not growable
(or shrinkable) through a `&mut Box<[T]>`; you would have to allocate new storage and
move the elements over.  The length of a boxed slice is stored at runtime, and isn't
known at compile time.  Therefore, like a shared slice, it consists of a pointer and
a length.

- `Arc<[T]>` and `Rc<[T]>` are shared owneship variations on `Box<[T]>`.

- There are similar variations for string-like types (`Box<str>`, `Arc<Path>`, `Rc<OsStr>`, ...)

- And other combinations too (`Box<[T; N]>`, etc.)

You can create shared slices to these other types of owned slices as well.

Technically, just a single `T` is like a `[T; 1]` (it has the same layout in memory).
So if you squint just right, every owned `T` is also a form of owned slice, but with
a compile-time known length of 1.  And indeed, you can create
[`&[T]`](https://doc.rust-lang.org/std/slice/fn.from_ref.html)
and [`&mut [T]`](https://doc.rust-lang.org/std/slice/fn.from_mut.html)
([and array versions too](https://doc.rust-lang.org/std/array/index.html#functions))
from `&T` and `&mut T`.

## Graphical layout

Here's a graphical representation of the layout of slices, shared slices,
`Vec<T>`, and `&Vec<T>`.

```
+---+---+---+---+---+---+---+---+
| Pointer       | Length        | &[T] (or &str, &Path, Box<[T]>, ...)
+---+---+---+---+---+---+---+---+
  |
  V
+---+---+---+---+---+---+---+---+
| D | A | T | A | . | . | . | ......    [T] (or str, Path, ...)
+---+---+---+---+---+---+---+---+
  ^
  |
+---+---+---+---+---+---+---+---+---+---+---+---+
| Pointer       | Length        | Capacity      | Vec<T> (or String, PathBuf, ...)
+---+---+---+---+---+---+---+---+---+---+---+---+
  ^
  |
+---+---+---+---+
| Pointer       | &Vec<T> (or &String, &PathBuf, ...)
+---+---+---+---+
```

One advantage of taking `&[T]` instead of `&Vec<T>` as an argument should be
immediately apparent from the diagram: a `&[T]` has less indirection.

However, there are other reasons:
- Everything useful for `&Vec<T>` is actually a method on `&[T]`
  - You can't check the capacity with a `&[T]`, but you can't change the capacity with a `&Vec<T>` anyway
- If you take `&[T]` as an argument, you can take shared slices that point to data
which isn't owned by a `Vec<T>` (such as static data, part of an array, into a `Box<[T]>`, et cetera)
  - So it is strictly and significantly more general to take `&[T]`

Similar advantages apply to taking a `&str` instead of a `&String`, et cetera.

In contrast, there are many things you can do with a `&mut Vec<T>` that you can't
do with a `&mut [T]`, so which you choose depends much more on what you need to
do with the borrowed data.

### Graphical layout for arrays

The layout of an array is the same as a slice, except the length is known.

```
+---+---+---+---+---+---+---+---+
| Pointer       | Length        | &[T] (or &str, &Path, Box<[T]>, ...)
+---+---+---+---+---+---+---+---+
  |
  V
+---+---+---+---+---+---+---+
| D | A | T | A | . | . | . | [T; N] (or str, Path, ...)
+---+---+---+---+---+---+---+
  ^
  |
+---+---+---+---+
| Pointer       | &[T; N] (or `Box<[T; N]>`, ...)
+---+---+---+---+
  ^
  |
+---+---+---+---+
| Pointer       | &Box<[T; N]> (or &&[T; N], ...)
+---+---+---+---+
```

Because `[T; N]` is sized, and because the length is part of the type,
pointers to it (like `&[T; N]`) are normal "thin" pointers, not "wide"
pointers.  But you can also create a `&[T]` that points to the array
(or to part of the array), as in the diagram.

Should you take a `&[T]` or a `&[T; N]` as a function argument?  If
you don't need a specific length, and aren't trying to generate code
that's optimized based on the specific length of the array, you probably
want `&[T]`.

