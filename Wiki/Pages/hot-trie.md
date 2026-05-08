---
tags:
  - data-structures
  - performance
sources:
  - "Raw/Fastest CS/The fastest ordered maps in computer science.md"
  - "Raw/Fastest CS/General.md"
last_updated: 2026-05-08
---

# HOT (Height Optimized Trie)

Binna et al., SIGMOD 2018. Improves on [[adaptive-radix-tree|ART]] for **string keys** by dynamically varying the number of bits used per node based on data distribution, reducing tree height further than ART's fixed byte-at-a-time decomposition. Outperforms ART, [[b-tree|B-trees]], and [[masstree]] for single-threaded string workloads.

## What ART misses

ART discriminates one byte per level. For string keys with long shared prefixes, this can cost a cache miss per byte of distinguishing information — a pessimistic case where the trie's structural advantage evaporates. HOT avoids this by selecting per-node a number of bits that maximizes branching factor given the actual key distribution at that subtree, so a node may handle 2 bits of key in one place and 8 bits in another. The result: shorter trees, fewer cache misses, and shorter hot paths.

## Where it sits

For single-threaded string-keyed ordered maps, HOT is the design point that beats ART. For *integer* keys, ART (or its concurrent variant [[art-olc|ART-OLC]]) is still preferred — HOT's per-node bit adaptation costs more than it saves on uniformly distributed integer keys.

For variable-length keys via SIMD B+-tree rather than trie, see [[fb-plus-tree]]. For the learned-index extension, **LITS** (2024) combines learned models with HOT-style tries and claims **2.4× over HOT** on point operations — the current research frontier for string-keyed ordered indexing. See [[learned-indexes]] and [[fastest-ordered-maps]].
