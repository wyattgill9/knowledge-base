---
tags:
  - cpp
  - concurrency
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-29
---

# ParlayHash

CMU's concurrent hash map (2024) and the unambiguous state-of-the-art for high-thread-count workloads. On a 128-hyperthread Intel Xeon Ice Lake, ParlayHash hits **1,130 million ops/sec** — roughly 7× Meta's `folly::ConcurrentHashMap`, 10× phmap's parallel variant, 19× Intel TBB, and 39× libcuckoo. Memory cost is 26.3 bytes/element, also the lowest among concurrent maps.

## The benchmark

| Implementation | 1 thread (Mops) | 128 threads (Mops) | Memory (B/elt) |
|---|---|---|---|
| **ParlayHash** | 21.4 | **1,130** | 26.3 |
| boost::concurrent_flat_map | **25.3** | 58 | 37.9 |
| folly::ConcurrentHashMap | 12.2 | 157 | 91.9 |
| phmap::parallel_flat_hash_map | 22.0 | 112 | 36.0 |
| libcuckoo | 14.2 | 29 | 43.6 |
| Intel TBB concurrent_hash_map | 13.2 | 54 | — |

Note the shape: at 1 thread, `boost::concurrent_flat_map` is fastest. At 128 threads, ParlayHash is in another league. Most concurrent maps top out somewhere between — 50–150 Mops — because their lock or sharding strategy hits a contention wall.

## How it gets there

Two pieces. **Epoch-based memory reclamation** (similar to Rust's [[crossbeam-epoch]] or Linux [[rcu]]) lets readers proceed without atomic refcount updates or hazard pointers; freed memory is deferred to a quiescent epoch. **Parallel internal operations** via parlaylib mean rehashes and bulk operations can themselves use multiple threads. Together they remove the two bottlenecks that flatten everyone else past ~16 threads: contended atomics on every read, and serial rehash.

## The catch

Sequential maps are still much faster single-threaded. [[abseil-flat-hash-map|Abseil]] alone hits 40.1 Mops; [[boost-unordered-flat-map]] is comparable. ParlayHash at 1 thread is 21.4 Mops — half the throughput of a non-concurrent map. **Concurrent maps only justify their overhead above ~4–8 threads.** If your workload does not have that level of parallelism, you are paying for scaling you cannot use.

ParlayHash also depends on parlaylib and is C++ — no Rust binding, no production-stable language ports. For Rust concurrent loads see [[papaya]] (read-heavy, async-safe), [[dashmap]] (balanced), [[scc]] (extreme write contention).

## Architectural significance

ParlayHash demonstrates that the gap between sequential and concurrent hash maps is not fundamental — it is implementation. Most concurrent designs (lock-striping, RwLock sharding, group locks like `boost::concurrent_flat_map`) carry contention costs that grow with thread count. Epoch-based reclamation plus parallel internal ops is the recipe that escapes that asymptote. Expect future concurrent map work to converge here. See [[fastest-hash-map-2025]] for the full landscape.
