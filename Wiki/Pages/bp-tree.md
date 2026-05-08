---
tags:
  - performance
  - data-structures
  - concurrency
sources:
  - "Raw/Fastest CS/The fastest ordered maps in computer science.md"
last_updated: 2026-05-08
---

# BP-Tree

The 2023 VLDB concurrent B+-tree using optimistic lock coupling. Achieves **7.4× over [[masstree]] on point operations** and **30× on large range queries**, while matching Masstree's behavior on contention-heavy workloads. The current state-of-the-art for concurrent ordered maps where range scans matter.

## Where it wins

Range scans. A B+-tree's linked-leaf layout was always more scan-friendly than a trie, but [[masstree]] historically beat it on point throughput by combining trie efficiency with B+-tree leaves. BP-Tree closes the point-throughput gap (using fine-grained optimistic locking adapted from [[art-olc|ART-OLC]] principles) without giving up the leaf-scan advantage — so on workloads with significant range queries it is no longer a contest.

## Where it sits

In the 2025 concurrent ordered-map ranking from the OpenBw-Tree benchmarking work and successors:

1. [[art-olc|ART-OLC]] / [[congee]] — fastest for concurrent point ops on dense integer keys
2. [[masstree]] — best under high contention and for variable-length string keys
3. **BP-Tree** — dominant when range scans are part of the workload
4. Java `ConcurrentSkipListMap` — simplest, scales adequately

The recurring lesson across all four: optimistic readers that never write to shared cache lines — the same insight that makes [[rcu]] and [[faa-vs-cas|FAA-over-CAS]] win — beats lock-freedom that pays cache-coherence cost. The lock-free [[bw-tree|Bw-tree]] is 1.5–4.5× *slower* than these alternatives.

A related 2025 result: **B-skiplists** (ICPP 2025) reach 2–9× over traditional concurrent skip lists by packing multiple elements per node — the same B-tree-over-pointer-chasing argument applied to skip-list concurrency. See [[fastest-ordered-maps]] for the full hierarchy.
