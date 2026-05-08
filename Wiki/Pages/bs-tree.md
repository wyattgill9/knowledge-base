---
tags:
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest ordered maps in computer science.md"
last_updated: 2026-05-08
---

# BS-Tree

The 2026 SIMD B-tree (ICDE 2026) that pushes the [[algorithmica-s-tree|Algorithmica S+ tree]] design from AVX2 to AVX-512 and from static-only to dynamic. Each node packs **16 keys searched in two AVX-512 instructions**; gapped nodes enable **branchless updates**, removing the static-build restriction that defined earlier SIMD B-trees.

## The two-instruction node search

A 16-element node fits in a single 512-bit register. A broadcast of the search key followed by `_mm512_cmp_epi32_mask` yields a 16-bit mask in two instructions; a `tzcnt` resolves the child index. No branches, no scalar comparisons, one cache line per node — the limit case of the [[b-tree]] cache-line-sized-node argument.

## Gapped nodes for branchless updates

Earlier SIMD B-trees ([[algorithmica-s-tree|S+ tree]]) achieved their inner-loop simplicity by forbidding modification. BS-tree instead reserves slack in each node so insertion and deletion shift a known-bounded number of slots without triggering structural changes on the hot path. Combined with the SIMD search, this delivers the static-tree throughput on dynamic workloads.

## Where it sits

Top of [[fastest-ordered-maps|Tier 1]]. AVX-512 is now portable across **AMD Zen 5** (2024, native 512-bit execution — not double-pumped) and Intel Granite Rapids, so the BS-tree's microarchitectural assumptions hold on both vendors. For variable-length keys, the parallel design point is [[fb-plus-tree]]. For concurrent variants, see [[bp-tree]] and [[art-olc]].
