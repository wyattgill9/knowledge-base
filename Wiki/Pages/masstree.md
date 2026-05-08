---
tags:
  - concurrency
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest ordered maps in computer science.md"
last_updated: 2026-05-08
---

# Masstree

The concurrent ordered map of choice for **variable-length string keys under high contention**. A trie-of-B+-trees: keys are sliced into 8-byte chunks and indexed by a trie at the top with B+-trees handling the leaves. This combines trie efficiency for long keys with B+-tree cache locality on the search path.

## Cache craftiness

Masstree's defining principle is "cache craftiness" — read operations **never dirty shared cache lines**. Optimistic version-counter readers (the same idea behind [[art-olc|ART-OLC]]) verify rather than write, so even at high reader counts the cache-coherence traffic stays bounded. This is why Masstree achieves 6–10 Mops/sec on 16 cores under YCSB, only ~2.5× slower than a near-optimal hash table — a small price for range-query support.

## Where it wins, where it loses

Wins: long string keys, high contention, mixed-workload OLTP. Loses: fixed integer keys at low contention (where [[art-olc|ART-OLC]] is faster) and range-scan-heavy workloads (where [[bp-tree|BP-Tree]] now dominates by 7.4× on points and 30× on scans).

The 2025 ordered-map landscape sorts cleanly: [[art-olc]] for points on integers, **Masstree** for points on strings, [[bp-tree]] for ranges, Java `ConcurrentSkipListMap` as a portable fallback. The lock-free [[bw-tree]] is 1.5–4.5× *slower* than all three Masstree-class designs because cache-coherence traffic dwarfs the cost of the locks it avoids. See [[fastest-ordered-maps]].
