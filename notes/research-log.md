# Research Log

This file records the step-by-step research process for the review of
*Tiny Pointers*.

## Process Goals

- Identify the primary paper and its stable source.
- Record the main problem, model, claims, and technical contributions.
- Collect related work mentioned by the paper.
- Collect related work not explicitly discussed by the paper, including later
  work after the paper appeared.
- Develop an independent explanation of the construction, proof ideas, and
  significance.

## Log

- 2026-07-07: Initialized the repository structure for a process-preserving
  review. No technical claims are recorded yet.
- 2026-07-07: Identified the primary paper:
  - Title: *Tiny Pointers*
  - Authors: Michael A. Bender, Alex Conway, Martin Farach-Colton,
    William Kuszmaul, Guido Tagliavini
  - arXiv: 2111.12800, submitted 2021-11-24
  - arXiv link: https://arxiv.org/abs/2111.12800
  - DOI link: https://doi.org/10.48550/arXiv.2111.12800
  - DBLP record: https://dblp.org/rec/journals/corr/abs-2111-12800
  - DBLP classifies the record as CoRR abs/2111.12800 (2021), informal/other
    publication. I have not found a separate conference or journal version.

## First-Pass Reading Notes

The paper asks a deceptively simple question: how many bits are needed to store
a pointer? If a pointer is an independent absolute address into an array of size
`n`, then it needs `log n` bits. The paper argues that many data-structure
pointers are not independent absolute addresses: they are owned by a key, node,
user, or other context. A tiny pointer uses this context. The pair `(key, tiny
pointer)` identifies a location even when the tiny pointer alone would be far
too short to do so.

The paper's central object is a dereference table supporting:

- `Allocate(k)`: reserve a slot for key/user `k`, returning tiny pointer `p`.
- `Dereference(k, p)`: recover the reserved slot using both `k` and `p`.
- `Free(k, p)`: release the slot.

The main tradeoff is between load factor `1 - delta` and tiny-pointer size.
The paper proves tight bounds:

- Fixed-size tiny pointers: `Theta(log log log n + log delta^{-1})` bits.
- Variable-size tiny pointers: expected `Theta(1 + log delta^{-1})` bits, with
  a doubly exponential tail bound for large pointer sizes.

The paper then applies the abstraction to five problems:

- Relaxed data retrieval with tiny retrievers.
- Succinct dynamic binary search trees, including rotation-based trees.
- Stable dictionaries.
- Dictionaries with variable-size values.
- Internal-memory stashes for external-memory arrays.

The first technical translation I should keep in mind is:

> A key is like a ball, an array slot is like a bin, and a tiny pointer records
> which candidate bin in the key's probe sequence actually got used.

This translation makes the pointer size essentially the logarithm of the probe
index. The whole problem becomes: dynamically maintain assignments so that every
live key uses an early candidate in its own probe sequence, even under arbitrary
insertions, deletions, and reinsertion of the same key.

- 2026-07-07: Collected related-work directions. The paper itself connects tiny
  pointers to retrieval, succinct/dynamic trees, stable hashing/dictionaries,
  variable-size values, internal-memory stashes, and dynamic balls-and-bins.
  I also searched for later work. I did not find an obvious later paper whose
  main contribution is explicitly titled as a direct continuation of "tiny
  pointers"; later relevance appears mostly in adjacent areas such as succinct
  dictionaries, cell-probe lower bounds, retrieval structures, minimal perfect
  hashing, and dynamic rank/select.
