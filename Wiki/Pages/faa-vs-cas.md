---
tags:
  - concurrency
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# FAA vs CAS

The single most consequential primitive choice in modern lock-free data structure design is **Fetch-And-Add over Compare-And-Swap** at contention hotspots. CAS can fail and retry, producing a phenomenon called *contention meltdown* where throughput **decreases** as thread count rises. FAA always succeeds in hardware, so throughput plateaus rather than collapses. At 144 threads on x86, FAA completes in roughly **40 ns** versus **75 ns** for CAS — a 1.8× advantage that compounds across millions of operations.

## What each instruction does

- **CAS (`LOCK CMPXCHG`)** — atomically checks whether a memory location holds an expected value, and writes a new value only if it matches. If another thread modified the location first, CAS reports failure and the caller must retry.
- **FAA (`LOCK XADD`)** — atomically reads a memory location, adds an integer, writes the sum back, and returns the previous value. Always succeeds.

CAS is more general — you can implement any atomic update with CAS-retry — but generality has a cost. FAA is a constrained primitive that is implemented more directly in the cache coherence protocol.

## Why CAS melts down under contention

When N threads each want to increment a counter via CAS:

1. Thread 1 reads the counter, computes `+1`, attempts CAS.
2. Threads 2..N read the same value, compute `+1`, attempt CAS — all but one fail.
3. The failed threads retry. Half succeed (probabilistically), half fail again.
4. Throughput is **O(log N) operations per round** instead of O(N).

The cache line bounces between cores on every retry, each bounce costs 50–200 ns of [[cache-coherency|MESI traffic]], and contention compounds. Real benchmarks show throughput *decreasing* past ~32 threads on contended CAS — adding cores makes things slower.

## Why FAA scales

`LOCK XADD` is implemented natively in the cache coherence protocol on x86 (and equivalent on ARM via `LDADD`). The CPU acquires the cache line, increments, and releases — once per operation, no retries. Throughput plateaus at roughly the cache-coherence round-trip rate (one operation per ~10–40 ns per contended line) and stays flat with thread count rather than collapsing.

The 144-thread numbers from the [[the-fastest-queue|queue paper]]:

| Operation | Latency at 144 threads |
|-----------|-----------------------|
| FAA (`LOCK XADD`) | ~40 ns |
| CAS (`LOCK CMPXCHG`) | ~75 ns |

## When CAS is still right

FAA only handles addition, so it cannot replace CAS for:

- **Pointer swaps** — replacing a head/tail pointer with a new node ([[michael-scott-queue|Michael-Scott]] style).
- **Conditional updates** — "set X to Y only if X is currently Z."
- **Sentinel-aware operations** — clearing a flag while preserving the rest of the word.

The right pattern, used by [[lcrq|LCRQ]], [[scq|SCQ]], and [[lprq|LPRQ]], is FAA on the *contended* index, then CAS on the *uncontended* slot. The slot CAS rarely fails because each thread gets a unique slot from the FAA — so the CAS retry path almost never triggers.

## Defeating the FAA bottleneck itself

Even FAA serializes through cache coherence — the hardware physically cannot grant atomic increments faster than one cache-coherence round-trip. At extreme thread counts (100+), this becomes a bottleneck. [[aggregating-funnels|Aggregating Funnels]] (PPoPP 2025) addresses this by software-combining FAA operations: threads contend at funnel leaves, partial increments merge upward, and a single FAA at the root commits the combined increment. This gives up to 2.5× over plain FAA at high thread counts.

## See also

- [[lcrq]], [[scq]], [[lprq]], [[wcq]] — queues built around the FAA-then-slot-CAS pattern
- [[aggregating-funnels]] — the next layer up
- [[michael-scott-queue]] — the canonical CAS-retry queue this insight obsoleted
- [[cache-coherency]] — why contention costs what it costs
