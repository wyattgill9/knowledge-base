---
tags:
  - data-structures
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/The fastest ordered maps in computer science.md"
last_updated: 2026-05-08
---

# Bε-Tree (Fractal Tree Index)

A write-optimized external-memory ordered index. Achieves **O(log_B N / B^(1−ε))** amortized I/Os per insert — for typical parameters that's **~32× fewer write I/Os than a [[b-tree|B-tree]]** — by buffering updates inside internal nodes and propagating them in batches. Reads pay slightly more I/Os in exchange.

## The buffering idea

A standard B-tree applies every insert immediately to a leaf, paying one disk I/O per write. A Bε-tree adds a **buffer** to each internal node: an insert is appended to the root's buffer cheaply, and only when a buffer fills does it flush downward in batches. Each flush amortizes its I/O across many inserts, so the per-insert I/O cost drops sublinearly with the buffer size.

The asymptotic improvement comes from the parameter ε ∈ (0, 1] tuning the buffer-to-fanout ratio. ε = 1 recovers a B-tree (no buffering); ε close to 0 maximizes write amortization at modest read cost.

## Production lineage

[[https://en.wikipedia.org/wiki/TokuDB|TokuDB]] used Bε-trees in production for MySQL until its 2022 deprecation. The principle did not die with TokuDB — it lives in **LSM-tree variants**, which can be viewed as a coarse-grained Bε-tree where the buffer is a sequence of sorted on-disk runs. Modern systems like RocksDB, ScyllaDB, and CockroachDB inherit the write-amortization argument even when they do not use a literal Bε-tree.

## When to reach for write-optimized

Bε-trees and LSMs win when:

- Writes dominate the workload.
- The working set spills past RAM (the I/O cost gap matters).
- Read latency variance is acceptable (LSMs especially can have multi-level reads).

For in-memory or read-dominated workloads, conventional B-trees and the [[fastest-ordered-maps|SIMD B-tree]] family remain faster. [[cache-oblivious-structures|COLAs]] reach the same write-amortization regime without parameter tuning. See [[fastest-ordered-maps]] for the broader landscape.
