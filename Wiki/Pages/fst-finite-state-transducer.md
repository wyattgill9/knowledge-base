---
tags:
  - data-structures
  - performance
  - rust
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# Finite State Transducer (FST)

Extraordinarily space-efficient immutable sorted string sets and maps. Unlike tries that share only prefixes, FSTs share both prefixes and suffixes, achieving much higher compression for natural-language and URL-like key sets.

## Key implementations

**BurntSushi's Rust `fst` crate** can index **1.6 billion keys** by sharing both prefixes and suffixes. The resulting structure can be memory-mapped for zero-copy queries, making it practical for datasets that far exceed RAM when only the index needs to be in memory.

**Lucene's Java FST** builds 9.8 million Wikipedia terms in ~8 seconds into a 69 MB structure. FSTs power the term dictionaries in both Lucene and Tantivy (Rust) search engines — the core data structure enabling fast term lookup in full-text search.

## How they work

An FST is a deterministic finite automaton where each transition is labeled with an input byte and an output value. For a sorted set, outputs are unused (just accept/reject). For a sorted map, outputs encode associated values along transition edges. The key insight: a minimal FST for a sorted input can be constructed in linear time in a single pass, and the resulting automaton is typically orders of magnitude smaller than the input.

## Trade-offs

FSTs are **immutable** — you build them once from sorted input. Any mutation requires rebuilding. They are also **sorted-access only** — you can do prefix queries, range queries, and exact lookups, but not arbitrary substring search. For mutable string indexing, [[adaptive-radix-tree|ART]] or HAT-tries are better choices.

## Relationship to other structures

FSTs complement [[adaptive-radix-tree|ART]] (mutable, in-memory) and [[succinct-data-structures]] (general-purpose compressed representations). For probabilistic membership (no need for exact key retrieval), [[binary-fuse-filter|Binary Fuse filters]] are faster and simpler.
