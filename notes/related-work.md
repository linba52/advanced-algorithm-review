# Related Work Notes

This file will collect related work for the review of *Tiny Pointers*.

The notes will be organized into:

- Earlier work that motivates the paper.
- Related work cited or discussed by the paper.
- Related work that is relevant but not central in the paper.
- Later work after the paper appeared.

## 1. Earlier and Contemporaneous Work Mentioned by the Paper

### Retrieval and Relaxed Retrieval

The paper's first application is a relaxation of the dynamic retrieval problem.
In static retrieval, a data structure stores a function on a set of keys and may
return anything outside the set. The paper cites:

- Stephen Alstrup, Gerth Brodal, and Theis Rauhe, "Optimal static range
  reporting in one dimension" (STOC 2001). The paper uses this line for lower
  bounds around retrieval.
- Erik Demaine, Friedhelm Meyer auf der Heide, Rasmus Pagh, and Mihai
  Patrascu, "De dictionariis dynamicis pauco spatio utentibus" (LATIN 2006).
  This is part of the dynamic succinct dictionary/retrieval background.
- Martin Dietzfelbinger and Rasmus Pagh, "Succinct data structures for retrieval
  and approximate membership" (ICALP 2008).
- Martin Dietzfelbinger and Stefan Walzer, "Constant-time retrieval with
  o(log m) extra bits" (STACS 2019).

The point for the review: ordinary dynamic retrieval has an
`Omega(n log log n)`-type metadata lower bound in the settings the paper cares
about. Tiny pointers do not contradict that lower bound; instead, the paper
changes the interface by giving the user a tiny retriever/hint to store.

### Succinct Trees and Rotation-Based BSTs

The paper emphasizes that succinct tree representations can encode the shape of
a binary tree in `O(n)` or `2n + o(n)` bits, but supporting rotations efficiently
has been a bottleneck. It cites:

- Rajeev Raman and Satti Srinivasa Rao, "Succinct dynamic dictionaries and
  trees" (ICALP 2003).
- J. Ian Munro, Venkatesh Raman, and Adam J. Storm, "Representing dynamic
  binary trees succinctly" (SODA 2001).
- Gonzalo Navarro and Kunihiko Sadakane, "Fully functional static and dynamic
  succinct trees" (ACM Transactions on Algorithms, 2014).
- Joshimar Cordova and Gonzalo Navarro, "Simple and efficient fully-functional
  succinct trees" (Theoretical Computer Science, 2016).

The review should stress that tiny pointers are useful here because a
rotation-based search tree is naturally pointer-heavy: parent/child navigation is
easy with ordinary pointers but too expensive in a succinct representation.

### Stable Hashing and Stable Dictionaries

The third application concerns stability: once a value is inserted, its location
should not move. This is important when external clients keep pointers or
references into the data structure. The paper connects this to:

- Don Knuth's early notes and *The Art of Computer Programming*, volume 3, on
  open addressing.
- Per-Ake Larson's work on uniform hashing and external hashing.
- Peter Sanders, "Hashing with Linear Probing and Referential Integrity"
  (arXiv:1808.04602, 2018).
- Demaine et al.'s dynamic succinct dictionaries work, where stable hashing has
  a known `Theta(log log n)` overhead in some regimes.
- Practical stable hash tables in mainstream systems such as Google Abseil and
  Facebook F14.

The review should distinguish "stable values" from "stable table cells":
Tiny Pointers lets the user keep a stable tiny pointer to the value even if the
dictionary's own metadata layout changes.

### Balls and Bins

The main technical engine is a dynamic balls-and-bins view. Earlier work:

- Azar, Broder, Karlin, and Upfal introduced the "power of two choices" result,
  showing an exponential improvement in maximum load compared with one random
  choice.
- Berthold Voecking studied asymmetric balanced allocations; the tiny-pointers
  paper uses a lower bound in this family for its fixed-size lower bound.
- Philipp Woelfel analyzed asymmetric balanced allocation with simple hash
  functions.
- Larson analyzed uniform hashing under random insertions/deletions, while
  general dynamic insertion/deletion sequences for simple probing schemes remain
  difficult.

The review should explain why the dynamic version is harder than the static
balls-into-bins version: when the same key is deleted and reinserted, the table
state already depends on that key's old probe sequence.

### Internal-Memory Stashes and Page Tables

The fifth application revisits stashes for locating objects in an external array.
The paper traces this line to:

- Gonnet and Larson's external hashing with limited internal storage.
- Larson and Kajla's one-access retrieval/file organization work.
- Modern address-translation/page-table motivation.
- Bender et al.'s adaptive filter work, which the paper adapts for the stash.

The review should describe this as a place where the pointer-compression problem
is extremely concrete: the internal stash is metadata whose job is to point into
a much larger external memory.

## 2. Related Work Not Central in the Paper

### Hash Tables and Succinct Dictionaries as a Broad Context

The paper is part of a larger agenda: making dynamic hashing and dictionaries
approach information-theoretic space while retaining fast operations. Closely
related context includes:

- Pagh and Rodler's cuckoo hashing.
- Pagh and Pagh's uniform hashing in constant time and optimal space.
- Arbitman, Naor, and Segev's de-amortized/backyard cuckoo hashing.
- Bender, Farach-Colton, John Kuszmaul, William Kuszmaul, and Liu's "On the
  Optimal Time/Space Tradeoff for Hash Tables" (arXiv:2111.00602), which gives
  a tradeoff between operation time and wasted bits per key.

This matters for the review because tiny pointers are not merely "short
addresses"; they are a mechanism for reducing the metadata overhead that keeps
dynamic structures away from information-theoretic space.

### Minimal Perfect Hashing and Monotone Minimal Perfect Hashing

Minimal perfect hashing is not the same problem as tiny pointers, because it is
typically static and maps keys collision-free into a compact range. But it is
adjacent because both areas ask: how much extra information is needed to recover
an index or rank from a key? Relevant later work includes:

- Sepehr Assadi, Martin Farach-Colton, and William Kuszmaul, "Tight Bounds for
  Monotone Minimal Perfect Hashing" (arXiv:2207.10556; SODA 2023).
- Dmitry Kosolobov, "Simplified Tight Bounds for Monotone Minimal Perfect
  Hashing" (arXiv:2403.07760).

These works are useful comparison points for the review's discussion of
`log log log` terms: such terms often appear when a data structure stores just
enough order/index information and no more.

## 3. Later Work After *Tiny Pointers*

### Dynamic Succinct Dictionaries and Lower Bounds

Li, Liang, Yu, and Zhou's "Tight Cell-Probe Lower Bounds for Dynamic Succinct
Dictionaries" (arXiv:2306.02253; FOCS 2023) proves matching lower bounds for
the time/space tradeoff achieved by the state-of-the-art dynamic succinct
dictionary of Bender et al. It is not a direct tiny-pointer sequel, but it
clarifies the broader landscape: near-optimal space in dynamic dictionaries
comes with unavoidable update-time tradeoffs.

Source: https://arxiv.org/abs/2306.02253

### Practical Retrieval Structures

Dillinger, Huebschle-Schneider, Sanders, and Walzer's "Fast Succinct Retrieval
and Approximate Membership using Ribbon" (arXiv:2109.01892; SEA 2022) and
Becht, Lehmann, and Sanders's "Brief Announcement: Parallel Construction of
Bumped Ribbon Retrieval" (arXiv:2411.12365) show that retrieval structures have
an active practical line focused on very low overhead, locality, and parallel
construction.

Sources:

- https://arxiv.org/abs/2109.01892
- https://arxiv.org/abs/2411.12365

This is relevant because *Tiny Pointers* uses retrieval as one of its application
targets, but its contribution is more theoretical and dynamic: it changes the
interface by adding a user-stored hint.

### Minimal Perfect Hashing Survey Work

Lehmann, Mueller, Pagh, Pibiri, Sanders, Vigna, and Walzer's 2025/2026 survey
"Modern Minimal Perfect Hashing: A Survey" (arXiv:2506.06536) documents recent
practical progress in perfect hashing. Again this is adjacent rather than a
direct continuation: minimal perfect hashing is static, while tiny pointers are
dynamic and owner-contextual. But both are about encoding positions/indexes with
very little extra information.

Source: https://arxiv.org/abs/2506.06536

### Dynamic Rank/Select and Succinct Ordered Dictionaries

Kuszmaul, Liang, and Zhou's "Succinct Dynamic Rank/Select: Bypassing the
Tree-Structure Bottleneck" (arXiv:2510.19175; SODA 2026) attacks a related
succinct dynamic representation bottleneck. It is especially relevant to the
review's discussion of succinct trees: after *Tiny Pointers*, the broader area
continued to look for ways around the fact that maintaining dynamic structure
often costs as much space as the data itself.

Source: https://arxiv.org/abs/2510.19175

## 4. Provisional Viewpoint for the Review

The strongest way to position *Tiny Pointers* is not as a one-off trick for one
data structure. It is a "metadata reframing" paper. Many succinct data-structure
problems are hard because metadata pointers are too large. The paper's move is
to observe that metadata often has context: the owner key, the tree node, or the
logical item is already known. Once dereferencing is allowed to depend on that
context, the information-theoretic lower bound for standalone pointers no longer
applies.

The related work also clarifies the paper's limitation. Tiny pointers do not
make arbitrary independent addresses shorter. They are powerful specifically
when the surrounding data structure controls allocation, access, and ownership.
