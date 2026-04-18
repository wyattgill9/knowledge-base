---
tags:
  - data-structures
  - performance
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# Adaptive Radix Tree (ART)

The standout novel tree structure for variable-length key indexing. Introduced by Viktor Leis at TU Munich (ICDE 2013), ART uses four adaptive node sizes with [[simd-programming|SIMD]]-optimized search, path compression, and lazy expansion. It surpasses FAST trees, CSB+ trees, red-black trees, and even hash tables for main-memory database lookups, with worst-case overhead of only 52 bytes/key.

## Node types

ART's key insight is that different fanouts need different representations:

| Node type | Children | Lookup strategy | Size |
|-----------|----------|----------------|------|
| Node4 | 1–4 | Linear scan | Tiny |
| Node16 | 5–16 | SSE comparison of all 16 keys simultaneously | Small |
| Node48 | 17–48 | 256-byte index array → O(1) child lookup | Medium |
| Node256 | 49–256 | Direct array indexing | 2,064 bytes |

Node16 is the most interesting: it uses SSE instructions to compare all 16 keys simultaneously, making the same [[swiss-table]]-style SIMD parallel scanning work for tree search. Node48's 256-byte index array trades space for O(1) lookup without any comparison.

## Optimizations

**Path compression** collapses single-child chains into stored prefixes, dramatically reducing tree height for keys with common prefixes. **Lazy expansion** defers node creation until a split is actually needed, keeping memory usage low during sequential inserts.

## Production use

ART powers the **HyPer database system**. Its combination of space efficiency (52 bytes/key worst case), fast lookup, and sorted traversal makes it ideal for main-memory OLTP indexes where both point queries and range scans matter.

## Beyond ART

**HOT (Height Optimized Trie)** (Binna et al., SIGMOD 2018) improves on ART by dynamically varying bits per node, further reducing tree height. For sorted immutable string storage, [[fst-finite-state-transducer|Finite State Transducers]] share both prefixes and suffixes (ART shares only prefixes), achieving extraordinary space efficiency for static sets. **Tessil's HAT-trie** combines burst-trie design with cache-conscious array hash tables for dramatically lower memory than `std::unordered_map` while maintaining sort order.

## Relationship to other structures

ART competes with [[b-tree]] for ordered indexing but excels specifically at variable-length keys. For fixed-size integer keys, B-trees with SIMD node search (Algorithmica S+ Tree) are faster. For hash-based unordered access, [[swiss-table]] variants win. For immutable string sets, [[fst-finite-state-transducer|FSTs]] win on space.
