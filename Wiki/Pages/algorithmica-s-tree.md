---
tags:
  - performance
  - data-structures
  - cpp
sources:
  - "Raw/Fastest CS/The fastest ordered maps in computer science.md"
  - "Raw/Fastest CS/General.md"
last_updated: 2026-05-08
---

# Algorithmica S+ Tree

Sergey Slotin's reference SIMD B-tree, presented in *Algorithms for Modern Hardware*. Achieves **7–18× speedup over `std::set` on `lower_bound`** and 3–7× over [[abseil-flat-hash-map|Abseil]] `btree_set` on integer key lookups, in **under 150 lines of C++**. It is the canonical demonstration that the gap between mainstream ordered maps and what hardware allows is dominated by cache and SIMD, not algorithm.

## What makes it fast

- **AVX2 branchless intra-node search.** A node's keys fit in a small number of 256-bit registers; `_mm256_cmpgt_epi32` + `movemask` resolves the next child in a handful of instructions with no branches.
- **Compile-time constant tree height.** Static B+-tree shape baked at compile time eliminates loop overhead and lets the compiler unroll the descent fully.
- **[[huge-pages|Hugepage]] allocation.** 2 MB pages remove TLB pressure for working sets that would otherwise miss the TLB at every level.
- **No per-node heap allocation.** The whole tree lives in a single contiguous arena.

The design only handles static, integer-key sets — no inserts after construction. That is the deliberate trade. By giving up dynamic updates the implementation gains the structural assumptions that make the inner loop branchless.

## Where it sits

[[fastest-ordered-maps|Tier 1]] of the ordered-map hierarchy — alongside the more recent [[bs-tree]] (AVX-512, gapped nodes for branchless updates) and [[fb-plus-tree]] (variable-length keys with SIMD prefix matching). For a production-ready drop-in, [[abseil-flat-hash-map|Abseil]] `btree_map` is the right pick; for the absolute ceiling on a static integer index, the S+ tree is the reference.

The same techniques — SIMD parallel scanning, contiguous layout, branch elimination — appear in [[swiss-table]] (hash maps), [[adaptive-radix-tree]] Node16 (tries), and [[binary-fuse-filter|blocked Bloom filters]]. The S+ tree is the cleanest demonstration of the recipe applied to ordered search.
