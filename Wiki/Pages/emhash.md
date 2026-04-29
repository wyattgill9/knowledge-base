---
tags:
  - cpp
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-29
---

# emhash

A C++ hash map family that wins **raw insert/erase throughput** and tolerates extreme load factors up to **0.999**. In Martin Ankerl's 2022 benchmark suite (29 hash maps × 6 hash functions × 11 benchmarks), `emhash7` clocked 3.6 ns/op for integer find-hit versus 15 ns for `std::unordered_map` — a 4.2× advantage and the fastest result on insert-and-erase workloads.

## What makes it fast

Three-way linear probing with x86 bit-scan instructions, no tombstones. Where [[abseil-flat-hash-map|Abseil]] degrades under heavy insert/erase cycling because of tombstone accumulation, emhash sidesteps the problem with active reorganization. Combined with linear probe groups it stays cache-friendly even when the table is nearly full.

The 0.999 load factor is the headline: most open-addressed hash maps cap at 0.875 ([[boost-unordered-flat-map]], [[abseil-flat-hash-map]]) because collision costs dominate above that. emhash's probing strategy keeps unsuccessful lookups bounded even at 99.9% occupancy, which makes it the tightest-packing option for memory-pressured workloads.

## The catch

Less community adoption and testing than [[boost-unordered-flat-map]] or [[abseil-flat-hash-map]]. The author has shipped multiple variants (emhash5, emhash6, emhash7, emhash8) with subtly different tradeoffs, and there is no official documentation explaining when to pick which. Production deployments are rarer; the API is less polished.

## When to use it

- Insert/erase throughput is the dominant cost.
- Memory pressure forces extreme load factors.
- You can afford to benchmark variants and pick.

For general-purpose C++ work, [[boost-unordered-flat-map]] is the safer default. emhash is a specialist's tool that delivers when you have the workload to justify the integration cost. See [[fastest-hash-map-2025]] for context.
