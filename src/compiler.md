# Scrutinize compiler advice


The compiler gives better errors than pretty much any other language I've used, but it still does give some poor suggestions in some cases.
It's hard to turn a borrow check error into an accurate "what did the programmer mean" suggestion.
So suggested bounds are an area where it can be better to take a moment to try and understand what's going on with the lifetimes,
instead of just blindly applying compiler advice.

I'll cover a few scenarios here.

