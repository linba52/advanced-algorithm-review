# Review of *Tiny Pointers*

This file will gradually become the final review. It starts as an empty
structure so that the commit history shows how the review develops from notes
into a complete essay.

## Working Thesis

The review will argue that *Tiny Pointers* is important because it separates two
ideas that are usually conflated: an address and the information needed to
recover an address in context. By making the owner key part of dereferencing,
the paper turns pointer compression into a dynamic balls-and-bins assignment
problem and gives tight bounds for the resulting abstraction.
