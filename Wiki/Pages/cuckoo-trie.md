---
tags:
  - data-structures
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/The fastest ordered maps in computer science.md"
last_updated: 2026-05-08
---

# Cuckoo Trie

A 2021 trie design that exploits **memory-level parallelism** (MLP) to outperform state-of-the-art ordered indexes by **20–360%** at a smaller memory footprint. The insight is microarchitectural rather than algorithmic: modern out-of-order CPUs can have many independent DRAM accesses in flight simultaneously, but pointer-chasing trees serialize those accesses one cache miss at a time.

## The MLP argument

A traditional trie (or B-tree) lookup looks like:

```
load node A → wait for L3/DRAM →
load node B (depends on A) → wait for L3/DRAM →
load node C (depends on B) → wait for L3/DRAM
```

Each cache miss is ~100 ns and the chain serializes them. A modern CPU can have 10+ outstanding misses in flight, but only when they don't depend on each other. The Cuckoo Trie restructures lookups so multiple speculative memory accesses are issued in parallel — using cuckoo-hashing-style placement to make the candidate locations independent — so the CPU's memory parallelism is actually used. The 20–360% gain comes from converting serialized misses into parallel ones.

## Why this matters beyond the Cuckoo Trie

MLP exploitation is one of the deepest under-used levers in modern data structure design. Most published indexes are designed assuming the cost model is "instructions executed" or "cache lines touched." On real hardware the cost is closer to "longest dependency chain of misses." Indexes that explicitly produce independent miss streams — Cuckoo Trie, the [[swiss-table|Swiss Table]] family's group probing, [[binary-fuse-filter|blocked Bloom filters]] — recover throughput the dependency-chained alternatives leave on the table.

The same observation is why [[lcrq|LCRQ]] beats [[michael-scott-queue|Michael-Scott]] queues: LCRQ's [[faa-vs-cas|FAA]] producers issue independent memory operations rather than waiting on a CAS chain. See [[mechanical-sympathy]] for the broader principle and [[fastest-ordered-maps]] for context.
