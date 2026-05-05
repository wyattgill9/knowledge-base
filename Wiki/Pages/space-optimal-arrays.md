---
tags:
  - performance
  - data-structures
  - architecture
sources:
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# Space-Optimal Dynamic Arrays

Theoretical computer science has solved the dynamic array space problem. Brodnik, Carlsson, Demaine, Munro, and Sedgewick proved in 1999 that any dynamic array maintaining O(1) amortized operations must waste at least Ω(√N) space — and exhibited a construction achieving this bound. Tarjan and Zwick (SOSA 2023) sharpened the result into a formal space-time tradeoff. Yet the empirical record consistently shows these structures are slower than `std::vector` in practice, because the indirection that buys you space efficiency defeats hardware prefetchers.

## The Brodnik et al. lower bound

The core result: any dynamic array supporting O(1) amortized push_back, pop_back, and indexing on N elements must use at least N + Ω(√N) space. Standard doubling vectors waste N + O(N) — up to 50% of allocated memory unused immediately after a grow. The √N bound is asymptotically tight.

Their construction is a two-level indexing scheme: an index block of √N pointers, each pointing to a data block of √N elements. Indexing is two memory accesses (compute block, then offset). Push_back is amortized O(1). Total wasted space is N + O(√N) — the optimal bound — because at any time only the last data block is partially filled.

## The Tarjan-Zwick generalization

Robert Tarjan and Uri Zwick's SOSA 2023 paper gave a more memory-efficient implementation and established a parameterized space-time tradeoff: a dynamic array using N + O(N^{1/r}) space must pay Ω(r) amortized growth cost. Setting r=2 recovers the √N bound; larger r trades growth speed for tighter memory. The result is a clean characterization of what is achievable, and demonstrates that the 1999 bound was not a quirk but an instance of a general principle.

## Hashed Array Trees

Sitarski (1996) introduced **Hashed Array Trees**: an index block of size √N pointing to data blocks of size √N. Wasted storage is O(√N), indexing is O(1) through one level of indirection, push_back is amortized O(1). HAT predates the formal lower bound and reaches it constructively.

The "hashed" name is misleading — there is no hashing. Sitarski borrowed the term from the resemblance to hash tables (an index pointing into bucket-sized blocks). HAT is sometimes called a "tabulated array" in modern literature.

## Tiered Vectors

Goodrich and Kloss (1999) generalized HAT to k-level tiered structures, providing O(N^{1/k}) **insertion anywhere in the array** — not just at the end. This is genuinely useful for workloads requiring mid-array insert, where standard `std::vector` is O(N) and linked lists destroy cache locality. Tiered Vectors are the right answer for "I need both random access and mid-array insert at decent speed."

## Why theory loses to practice

Katajainen's 2016 benchmarks ran the major space-optimal alternatives against `std::vector` and found them "measurably slower" for standard operations. The reasons compound:

1. **Indirection defeats prefetchers.** A two-level structure forces a pointer dereference between the index lookup and the data access. The hardware prefetcher cannot anticipate the second access from the first, so each indexing operation pays a cache miss penalty that the contiguous `std::vector` avoids entirely.
2. **Branch unpredictability.** The level-discrimination logic (which block am I in?) is not the kind of pattern modern branch predictors handle well, especially under varied access patterns.
3. **Constant factors are large.** The √N construction needs a metadata block, allocation tracking for each data block, and a slightly-more-complex indexing routine. For all but pathological N, the constant factor cost exceeds the asymptotic savings.

Katajainen's conclusion: a "sliced array with a few percent extra space" is the best practical worst-case dynamic array. In other words, [[fastest-dynamic-arrays|standard `std::vector` with geometric growth]] wins on real hardware.

## Where space-optimal designs still apply

The theoretical work remains valuable in narrow domains:

- **Memory-constrained embedded systems** where N + O(N) waste is genuinely prohibitive.
- **Persistent data structures** where the wasted slots translate to wasted disk or NVRAM.
- **Educational and theoretical contexts** where understanding the space-time frontier matters in itself.

For everything else — and this is the dominant case for production software — geometric growth, contiguous storage, and a [[mimalloc|good allocator]] win. This is the same lesson [[cache-oblivious-structures]] surfaces: theoretical optimality and benchmark-measurable speed often diverge, and the cache hierarchy is what reconciles them.

See [[fastest-dynamic-arrays]] for the broader hierarchy and [[fastest-data-structures]] for the same theme across other categories.
