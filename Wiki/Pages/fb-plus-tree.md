---
tags:
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest ordered maps in computer science.md"
last_updated: 2026-05-08
---

# FB+-Tree

The 2025 VLDB SIMD B+-tree that extends the [[bs-tree|BS-tree]] / [[algorithmica-s-tree|Algorithmica S+]] design to **variable-length keys** using AVX-512 byte-wise prefix matching. SIMD B-trees historically depended on fixed-size integer comparisons; FB+-tree breaks that restriction.

## Byte-wise prefix matching

For variable-length keys, the lookup compares discriminative byte prefixes packed into a node's metadata, broadcast into a 512-bit register, and matched against the probe key in parallel. The same SIMD instruction mask that resolves a child in [[bs-tree]] resolves a candidate slot here; the actual key comparison happens only on prefix-match, much like [[swiss-table]]'s H2 fingerprint dance.

## Why it matters

The string-key ordered-map design point used to bifurcate cleanly: tries (fast for shared prefixes, painful for range scans and arbitrary keys) versus B+-trees (slow inner-loop string comparisons but cache-friendly leaves). FB+-tree closes the SIMD gap on the B+-tree side, putting it in direct competition with [[hot-trie|HOT]] and [[adaptive-radix-tree|ART]] for general-purpose string-keyed ordered maps.

For the sibling fixed-key design see [[bs-tree]]. For trie alternatives see [[hot-trie]] and [[adaptive-radix-tree]]. For the broader landscape see [[fastest-ordered-maps]].
