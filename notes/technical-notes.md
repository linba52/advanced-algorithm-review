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

## Detailed Technical Understanding

### Why the Usual `log n` Lower Bound Does Not Apply

A normal pointer must identify one of `n` slots without help, so it needs
`log n` bits. A tiny pointer is deliberately not self-contained. The slot is
identified by:

```text
slot = Dereference(k, p)
```

where `k` is already known by the caller. Thus the information used to identify
the location is split between:

- the owner's key/context `k`;
- the short returned pointer `p`;
- the table's public/random metadata.

This is why the lower bound for standalone addresses is the wrong lower bound.
The right question is: how many bits must `p` contribute once `k` is already
known?

### Equivalence to Probe Sequences

For every key `k`, imagine an infinite probe sequence:

```text
h_1(k), h_2(k), h_3(k), ...
```

If `Allocate(k)` places `k` in `h_i(k)`, then the tiny pointer only has to encode
`i`, or enough information to reconstruct the corresponding local choice. The
value of `Dereference(k, p)` is the chosen `h_i(k)`.

In this view:

- keys are balls;
- slots are bins;
- a live assignment must be collision-free;
- pointer length is roughly `log i`;
- high load means the table has very few empty bins;
- the hard case is arbitrary deletion and reinsertion.

The "same key reappears" issue is important. On the first insertion of key `k`,
its hash/probe sequence is independent of the table state. After `k` is deleted
and later inserted again, the table state may already contain consequences of
the old placement of `k`. This history dependence is the reason simple probing
schemes are hard to analyze dynamically.

### Fixed-Size Construction in Operation-Level Detail

Goal: support load `1 - delta` with every tiny pointer having the same maximum
size:

```text
O(log log log n + log delta^{-1})
```

The construction combines two tables.

#### Primary Table

The primary table is a load-balancing table.

Parameters:

- table size `m = (1 - delta/2)n`;
- target failure slack about `delta^2`;
- bucket size `b = Theta(delta^{-2} log delta^{-1})`.

Allocation:

1. Hash key `k` to a bucket `h(k)`.
2. If some slot in the bucket is free, put the value there.
3. Return the local slot index `j` as the tiny pointer.
4. If the bucket is full, declare primary failure and send the allocation to
   the secondary table.

Dereference:

1. Recompute `h(k)`.
2. Interpret `p` as a local slot inside that bucket.

Free:

1. Recompute `h(k)`.
2. Mark the local slot indicated by `p` as free.

Why this is small:

```text
local pointer size = log b
                   = O(log(delta^{-2} log delta^{-1}))
                   = O(log delta^{-1})
```

This primary table alone is not enough, because it may fail on a small but
non-negligible fraction of allocations. Its purpose is to make the overflow set
small.

#### Secondary Table

The secondary table handles primary overflows.

Parameters:

- size `n' = delta n / 2`;
- sparse load about `Theta(1 / log log n')`;
- bucket size `Theta(log log n')`.

Allocation:

1. Hash `k` to two buckets.
2. Place the allocation in the bucket with more free slots.
3. Return one bit saying which of the two buckets was used plus the local slot.

Pointer size:

```text
1 + log(Theta(log log n')) = O(log log log n)
```

The primary table guarantees that only about `delta^2 n` live allocations
overflow. Since the secondary table has `Theta(delta n)` slots, the secondary
load is small enough for the power-of-two-choices analysis.

#### Combined Pointer Format

A fixed-size pointer stores:

```text
table tag | local pointer
```

The tag says primary vs secondary. The local pointer is padded to the maximum of
the two local formats.

Thus:

```text
size = O(1 + log delta^{-1} + log log log n)
```

#### Correctness Checklist

To verify correctness for an allocation:

1. If primary succeeds, only one key gets that primary slot because bucket free
   bits or free lists enforce exclusivity.
2. If primary fails, the allocation is inserted into the secondary table.
3. The secondary table also allocates only free slots.
4. The pointer contains enough information to recompute the same location from
   `k`.
5. `Free(k, p)` uses the same decoding path and releases exactly the occupied
   slot.

To verify high-probability success:

1. Lemma 1 says the number of live primary failures is `O(delta^2 n)` with high
   probability.
2. The secondary table has `Theta(delta n)` slots, so the overflow load is much
   less than its capacity threshold.
3. Lemma 2 says the secondary allocations succeed with high probability.
4. A union bound over the table components gives high-probability success.

### Variable-Size Construction in Operation-Level Detail

Goal: support load `1 - delta` with expected tiny-pointer size:

```text
O(1 + log delta^{-1})
```

The high-load case is reduced to a constant-load case by reusing the
primary/secondary idea: the primary table absorbs almost everything; the
secondary table only needs to store a small overflow set at constant load.
Therefore the key new problem is:

```text
Build a constant-load dereference table whose pointer has O(1) expected length.
```

#### Containers

The construction hashes keys into about `n / log n` containers. Each container is
allowed to hold at most:

```text
s = c log n
```

items. By Chernoff bounds, choosing `c` large enough makes every container stay
below capacity with high probability.

Each container is independent and uses `O(log n)` slots, so all containers use
`O(n)` slots total.

#### Levels Inside a Container

Inside one container there are levels:

```text
i = 0, 1, ..., log_2 s - 1
```

Level `i` has:

```text
s_i = s / 2^i
```

buckets, each with constant capacity `b`.

The intended behavior is geometric:

- try level 0 first;
- if the selected bucket is full, try level 1;
- if that fails, try level 2;
- and so on.

Because each level has only constant bucket capacity but low effective load, the
probability of moving from one level to the next is at most a constant such as
`1/b`. Therefore reaching level `i` has probability at most about `b^{-i}`.

#### Why Overflow Arrays Are Needed

The naive level cascade can fail badly if several small levels happen to be
unlucky. Small levels do not individually have high-probability guarantees, and a
bad level could send too much work to all later levels.

The fix is to give each level `i` an overflow array of size `s_i`. The algorithm
tracks a counter `L_i`: the number of allocations that have reached level `i`,
including those that are later sent beyond it. If the next level would be asked
to handle too much, the allocation is diverted into the overflow array of the
current level.

The crucial invariant is:

```text
L_i <= s_i
```

Because the overflow array for level `i` also has size `s_i`, it cannot run out
of room. This isolates bad luck: if a level behaves unusually badly, the cost is
paid locally in an overflow array rather than propagating to the rest of the
container.

#### Allocation Procedure

For key `k`:

1. Hash `k` to a container.
2. If the container already has `s` items, fail.
3. For each level `i`:
   - increment `L_i`;
   - try to place `k` in the level-`i` load-balancing table;
   - if successful, return a pointer encoding:
     - level `i`;
     - "ordinary level table";
     - local bucket slot;
   - if the next level would exceed its limit, place `k` in the overflow array
     for level `i` and return a pointer encoding:
     - level counted from the back;
     - "overflow";
     - local overflow slot.

The "counted from the back" encoding for overflow is subtle and important. If
overflow happens late, `s_i` is small, so the local slot index is short. Encoding
`log s - 1 - i` rather than `i` makes the level identity short in exactly those
late-overflow cases.

#### Pointer Size Bound

If allocation succeeds in the level-`i` load-balancing table, the pointer size is
roughly:

```text
O(log i) + O(1)
```

If allocation goes to the level-`i` overflow array, the pointer size is roughly:

```text
O(log s_i)
```

Probability of ordinary level `i`:

```text
Pr[B_i] <= 1 / b^i
```

because to reach level `i`, all previous selected buckets must have been full.

Probability of overflow at level `i`:

```text
Pr[O_i] <= exp(-Omega(s_i))
```

using Lemma 1, because level `i` becomes dangerously overloaded only with
exponentially small probability in its size.

Thus the expected pointer length is:

```text
sum_i Pr[B_i] O(log i) + sum_i Pr[O_i] O(log s_i) = O(1)
```

The tail bound is even stronger: the probability that pointer length exceeds
`ell` is doubly exponentially small in `ell`. Intuitively, a long pointer means
either reaching an exponentially late ordinary level or seeing an overflow in a
level whose capacity scale makes that event exponentially unlikely.

### Why Variable Size Beats Fixed Size

Fixed-size pointers must reserve enough bits for the worst live pointer that can
appear with high probability. That worst pointer has to handle rare but possible
deep placements, producing the `log log log n` term.

Variable-size pointers only pay for rare deep placements when they occur. Since
the probability of deep placement decays fast, the expected cost remains
constant (plus the `log delta^{-1}` high-load term).

This distinction is one of the paper's nicest conceptual points:

```text
fixed-size model: pay for the rare event on every pointer
variable-size model: pay for the rare event only when it happens
```

### Lower Bound for Variable-Size Pointers

Theorem 3 proves expected size `Omega(log delta^{-1})`.

Proof intuition:

1. Let `ell = Theta(delta^{-1})`.
2. If pointer value `i` means the slot `h_i(k)`, then short pointers only allow
   the first `ell` candidate slots.
3. Define the potential of a slot `j`:

```text
phi(j) = fraction of key/candidate pairs (k, i <= ell) that map to j
```

4. High-potential slots are useful to many keys, but there cannot be many of
   them, since the total potential over all slots is only `ell`.
5. Low-potential empty slots are plentiful, but a random new key is unlikely to
   have one of its first `ell` candidates among them.
6. Under a random alternating insertion/deletion workload at high load, a
   constant fraction of insertions therefore cannot be placed using the first
   `ell` candidate slots.
7. Those insertions need pointer value greater than `ell`, hence at least
   `Omega(log ell) = Omega(log delta^{-1})` bits.

This lower bound is not saying every pointer must be large. It says the expected
length cannot be too small because a constant fraction of random insertions are
forced out of the short-candidate region.

### Lower Bound for Fixed-Size Pointers

Theorem 4 adds `Omega(log log log n)` for fixed-size pointers.

If fixed pointer size is `s`, then every key has only:

```text
S = 2^s
```

possible tiny-pointer values. Therefore each key has only `S` candidate bins.

The lower-bound reduction:

1. Assume a dereference table with fixed pointer size `s = o(log log log n)`.
2. Then `S = 2^s = o(log log n)`.
3. Use the dereference table to define a sequential balls-into-bins process:
   each ball/key comes with its `S` candidate bins, derived from
   `Dereference(k, 1), ..., Dereference(k, S)`.
4. A classic balls-and-bins lower bound says any sequential process with only
   `S` choices has maximum load at least:

```text
Omega((log log m) / S)
```

5. Since `S = o(log log m)`, some bin gets `omega(1)` balls.
6. But the dereference table's successful allocations are collision-free in the
   original `n` slots; after grouping `n` slots into `m = Theta(n)` bins, every
   grouped bin can contain only `O(1)` balls.
7. Contradiction.

Therefore `S` must be at least `Omega(log log n)`, so:

```text
s >= Omega(log log log n)
```

Combining with the variable-size lower bound gives:

```text
s = Omega(log log log n + log delta^{-1})
```

for fixed-size pointers.

### What Is Technically Impressive

1. The abstraction is clean: it separates "what a pointer means" from "how many
   bits the pointer itself stores."
2. The bounds are tight in both the fixed-size and variable-size models.
3. The construction handles arbitrary dynamic updates, including deletion and
   reinsertion of the same key, where many simple probing schemes lack clean
   analyses.
4. The variable-size construction uses overflow arrays not as a hack but as a
   way to localize bad events and preserve a strong tail bound.
5. The paper turns the same tool into several black-box data-structure
   transformations, showing that pointer overhead is a recurring bottleneck
   rather than an isolated annoyance.
