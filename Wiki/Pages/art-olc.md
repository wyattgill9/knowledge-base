---
tags:
  - concurrency
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest ordered maps in computer science.md"
last_updated: 2026-05-08
---

# ART with Optimistic Lock Coupling (ART-OLC)

The fastest concurrent ordered map for point operations on dense integer keys. Combines [[adaptive-radix-tree|ART]]'s trie structure with **optimistic lock coupling**: readers verify version counters instead of acquiring locks, never writing to shared cache lines and therefore never triggering cache-coherence traffic between cores. This is the single highest-leverage technique in concurrent ordered map design — it produced ART-OLC's **4.8× fewer instructions and 3.6× fewer L3 cache misses** than B+-trees with traditional lock coupling.

## Optimistic lock coupling, briefly

Each node holds a version counter. A reader records the version, performs its lookup, and re-checks the version at the end; a mismatch means a writer interfered and the reader retries. Writers take an actual lock and bump the counter. Two consequences:

1. **Readers never write shared memory.** Cache lines stay clean across cores; a million readers cost what one reader costs.
2. **Writers serialize per-node, not per-tree.** Locality remains fine-grained.

This is the same architectural insight as [[rcu]] (zero read-side overhead) and [[faa-vs-cas|FAA-over-CAS]] (avoiding the CAS-retry cache-line ping). It is also why the lock-free [[bw-tree|Bw-tree]] *loses* to ART-OLC by 1.5–4.5×: lock-freedom does not imply cache-coherence-freedom, and the Bw-tree's CAS-heavy design generates exactly the coherence traffic OLC eliminates.

## Implementation

The reference implementation is C++ (Leis et al., follow-on to the original ICDE 2013 ART paper). The Rust port [[congee]] hits **150 Mops/sec on 32 cores** with 8-byte keys.

## Where it wins

Concurrent point operations on dense integer keys. For string keys at high contention, [[masstree]] is the design of choice. For range scans, [[bp-tree]] takes the throughput crown. For the full picture see [[fastest-ordered-maps]].
