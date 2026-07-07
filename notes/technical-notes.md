# Technical Notes

This file records the evolving technical understanding of *Tiny Pointers*.

The final review should explain:

- What problem the paper solves.
- Why the problem is important.
- What model the paper introduces.
- What techniques the paper uses.
- What the key intuition is.
- How the construction works.
- Why the construction is correct.
- Why the bounds are strong or optimal.

## Initial Technical Map

### Model

There is an array/store of `n` slots. Each live key `k` may own at most one slot
at a time. A dereference table maintains the slot assignment. The key point is
that `p` is not an absolute slot number; instead, `Dereference(k, p)` computes a
slot from the owner key and the short pointer.

### Fixed-Size Upper Bound

Theorem 1 gives fixed-size tiny pointers of
`O(log log log n + log delta^{-1})` bits at load factor `1 - delta`.

The construction has two pieces:

1. A primary load-balancing table stores almost all allocations. It partitions
   the store into buckets of size `Theta(delta^{-2} log delta^{-1})`. Key `k`
   hashes to a bucket, and `p` records the slot inside that bucket. The table is
   allowed to fail on a small fraction of allocations.
2. A secondary sparse table stores primary-table overflow. This secondary table
   uses a power-of-two-choices style construction with buckets of size
   `Theta(log log n)`, so the local slot can be described in
   `O(log log log n)` bits.

The fixed-size pointer needs enough bits to say whether it points into the
primary or secondary table and then enough local information to recover the slot.
The `log delta^{-1}` term comes from the bucket size needed for high load; the
`log log log n` term comes from the sparse power-of-two-choices secondary
structure.

### Variable-Size Upper Bound

Theorem 2 removes the fixed-size `log log log n` barrier by allowing the pointer
length to vary. The construction first reduces the high-load case to a constant
load-factor construction, then builds a constant-load table whose pointer length
has constant expectation and doubly exponential tail.

The constant-load construction hashes keys into containers of expected size
`Theta(log n)`. Each container has levels. Level `i` has about half as many
buckets as level `i-1`, with constant bucket capacity. If allocation fails at
one level, the key is tried at the next level. Overflow arrays isolate bad
levels so that a local bad event cannot cascade and break the whole container.

The reason expectation stays small is geometric: reaching level `i` requires
failing repeatedly, and each level has only a constant fraction of full buckets.
The reason the tail is doubly exponential is that late overflows correspond to
very small level capacities, whose bad events are exponentially unlikely in the
level size.

### Lower Bounds

Theorem 3 proves variable-size tiny pointers need
`Omega(log delta^{-1})` expected bits. The proof uses a potential-like argument:
some slots may be "useful" to many keys, but there cannot be too many such
slots, and repeatedly inserting random keys forces many allocations to use less
useful slots, which require larger pointers.

Theorem 4 proves fixed-size tiny pointers need
`Omega(log log log n + log delta^{-1})` bits. After the
`Omega(log delta^{-1})` part, the new ingredient is a balls-and-bins lower bound:
if only `S = 2^s` pointer values are available per key, then each key has only
`S` predetermined candidate bins. A classic lower bound for sequential
balls-into-bins says that too few choices lead to an overfull bin. Avoiding this
forces `S = Omega(log log n)`, hence `s = Omega(log log log n)`.
