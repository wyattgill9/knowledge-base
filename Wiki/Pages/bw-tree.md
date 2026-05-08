---
tags:
  - concurrency
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest ordered maps in computer science.md"
last_updated: 2026-05-08
---

# Bw-Tree

A lock-free B+-tree (Microsoft Research, ICDE 2013) using **delta chains** — a node update appends a delta record rather than modifying the page in place, with periodic consolidation merging deltas back. The design was once held up as the future of concurrent ordered maps. Subsequent benchmarking has been unkind.

## The OpenBw-Tree result

The OpenBw-Tree paper (Wang et al., SIGMOD 2018) reimplemented and benchmarked the Bw-tree against lock-based alternatives head-to-head. Result: the lock-free Bw-tree was **1.5–4.5× slower** than [[art-olc|ART-OLC]], [[masstree]], and lock-coupled B+-trees. The two compounding causes:

1. **Delta chains add pointer chasing.** Each lookup may traverse a chain of delta records before reaching the consolidated state — exactly the cache-miss cost B-trees were designed to avoid.
2. **CAS-heavy updates generate cache-coherence traffic.** Every CAS dirties a shared cache line, invalidating the corresponding line in every other core's L1. At high core counts this dominates throughput. The same dynamic produces the [[faa-vs-cas|CAS retry meltdown]] in queues.

## The lesson

Lock-freedom is not the same as cache-coherence-freedom, and on modern multi-core hardware the latter is what matters. Optimistic version-counter readers ([[art-olc|ART-OLC]], [[masstree]]) write *nothing* to shared memory on the read path — they win precisely because they avoid the work the Bw-tree spent its complexity budget on.

This insight generalizes: the reason [[rcu]] dominates read-dominated workloads is identical, the reason [[faa-vs-cas|FAA beats CAS]] in queues is identical, and the reason [[abseil-flat-hash-map|Abseil]] beats traditional concurrent maps under read-heavy access is closely related. Cache coherence is the substrate; algorithms that fight it lose.

The Bw-tree remains historically important and survives in some Microsoft systems, but for new designs the field moved on. See [[fastest-ordered-maps]] for the post-2018 ranking.
