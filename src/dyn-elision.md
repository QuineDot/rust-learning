# Elision rules

The `dyn Trait` lifetime elision rules are an instance of fractal complexity
in Rust.  Some general guidelines will get you 95% of the way there, some
advanced guidelines will get you another 4% of the way there, but the deeper
you go the more niche circumstances you may run into.  And unfortunately,
there is no proper specification to refer to.

The good news is that you can override the lifetime elision behavior by
being explicit about the lifetime, which provides an escape hatch from
most of the complexity.  So when in doubt, be explicit!

In the following subsections, we present the current behavior of the compiler
in layers, to the extent we have explored them.

We occasionally refer to [the reference's documentation on trait object lifetime elision](https://doc.rust-lang.org/reference/lifetime-elision.html#default-trait-object-lifetimes).
However, our layered approach differs somewhat from the reference's approach,
as the reference is not completely accurate.

