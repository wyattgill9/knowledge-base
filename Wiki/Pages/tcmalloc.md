---
tags:
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# tcmalloc

Google's tcmalloc (thread-caching malloc) is a high-performance general-purpose allocator with two distinguishing strengths: per-CPU caches at warehouse scale and exceptional throughput on large allocations. The standout benchmark is **50× the throughput of system libmalloc on 4 MB allocations** — driven by tcmalloc's specialized large-allocation path that skips the small-allocation thread cache entirely.

## The architecture

Classical tcmalloc maintained a per-thread cache of free objects organized by size class. Each thread allocates and frees from its local cache without locking; the cache periodically synchronizes with a central allocator. This design eliminates lock contention on the common path and was Google's standard for years.

Modern tcmalloc has shifted to **per-CPU caches**, which scale better than per-thread caches in environments with many threads but bounded CPU parallelism. The per-CPU model relies on restartable sequences (Linux's `rseq` interface) to provide fast atomic allocation paths without traditional locking. The per-CPU design is documented to achieve approximately **6 ns per allocation** at warehouse scale.

## Where tcmalloc wins decisively

The 50× throughput advantage on 4 MB allocations comes from the large-allocation path: tcmalloc routes large requests directly to mmap-backed page heaps without going through the size-class machinery, while many other allocators force large requests through the same multi-stage logic as small ones. For workloads dominated by infrequent large allocations — buffer pools, large arena setups, multi-page caches — tcmalloc is the right answer.

For typical small-allocation workloads, [[mimalloc]] and [[jemalloc]] are competitive or faster. mimalloc was specifically measured 13% faster than tcmalloc on the Lean compiler benchmark; jemalloc handles long-running fragmentation better. The choice between the three is workload-dependent.

## In Rust

`tcmalloc-rs` and `tcmalloc2-rs` provide Rust bindings, but the Rust ecosystem has settled on [[mimalloc]] and [[tikv-jemallocator|jemalloc]] as the default global allocator replacements. tcmalloc's per-CPU `rseq` machinery is more invasive than the alternatives and integrates less cleanly with Rust's allocator interface. See [[rust-memory-allocators]] for the full picture.

## The broader lesson

The single most important fact about general-purpose allocators is that **picking a good one delivers more performance than almost any container choice**. mimalloc 13–22% over system malloc, jemalloc 30% higher throughput at high contention, tcmalloc 50× on large allocations — none of these require a code change beyond linking. See [[fastest-dynamic-arrays]] for how this insight reorders the dynamic-array hierarchy.
