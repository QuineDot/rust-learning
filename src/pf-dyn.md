# `dyn Trait` lifetimes and `Box<dyn Trait>`

Every trait object (`dyn Trait`) has an elide-able lifetime with it's own defaults when completely elided.
This default is stronger than the normal function signature elision rules.
The most common way to run into a lifetime error about this is with `Box<dyn Trait>` in your function signatures, structs, and type aliases, where it means `Box<dyn Trait + 'static>`.

Often the error indicates that non-`'static` references/types aren't allowed in that context,
but sometimes it means that you should add an explicit lifetime, like `Box<dyn Trait + 'a>` or `Box<dyn Trait + '_>`.

The latter will act like "normal" lifetime elision; for example, it will introduce a new anonymous lifetime
parameter as a function input parameter, or use the `&self` lifetime in return position.

The reason the lifetime exists is that coercing values to `dyn Trait` erases their base type, including any
lifetimes that it may contain.  But those lifetimes have to be tracked by the compiler somehow to ensure
memory safety.  The `dyn Trait` lifetime represents the maximum lifetime the erased type is valid for.

---

Some short examples:
```rust
trait Trait {}

// The return is `Box<dyn Trait + 'static>` and this errors as there
// needs to be a bound requiring `T` to be `'static`, or the return
// type needs to be more flexible
fn one<T: Trait>(t: T) -> Box<dyn Trait> {
    Box::new(t)
}

// This works as we've added the bound
fn two<T: Trait + 'static>(t: T) -> Box<dyn Trait> {
    Box::new(t)
}

// This works as we've made the return type more flexible.  We still
// have to add a lifetime bound.
fn three<'a, T: Trait + 'a>(t: T) -> Box<dyn Trait + 'a> {
    Box::new(t)
}
```

For a more in-depth exploration,
[see this section of the `dyn Trait` tour.](./dyn-elision.md)
