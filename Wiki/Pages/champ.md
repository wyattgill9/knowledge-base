---
tags:
  - data-structures
  - performance
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# CHAMP (Compressed Hash-Array Mapped Prefix-tree)

A persistent (immutable) hash map variant delivering 10–100% improvement over classic HAMT for iteration. CHAMP powers Clojure's and Scala's immutable collections — the core data structure enabling efficient persistent maps in functional programming languages.

The improvement over HAMT comes from a more compact node layout that improves cache locality during iteration, the dominant operation for persistent collections (used in structural equality checks, serialization, and functional transformations).
