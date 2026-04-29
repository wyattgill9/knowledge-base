---
tags:
  - cpp
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
last_updated: 2026-04-29
---

# boost::unordered_flat_map

The 2025 consensus fastest general-purpose hash map. A [[swiss-table]] variant by Joaquín M López Muñoz that narrowly edges Google's [[abseil-flat-hash-map]] across multiple independent benchmarks while keeping memory overhead to ~1.07 bytes per bucket. The March 2025 attractivechaos benchmark called it "one of the fastest hash table implementations in C/C++"; Jackson Allan's May 2024 suite confirmed it as the all-around leader.

## What sets it apart from Abseil

Two things, both meaningful.

**Overflow byte instead of tombstones.** Abseil marks deleted slots with a sentinel that accumulates over time, gradually degrading probe-chain performance under heavy insert/erase cycling. Boost instead tracks a single byte per group recording whether *any* overflow has ever occurred there. Lookups can terminate at non-overflowed groups instantly, even when the actual slot is empty. The author measured **3.2× fewer comparisons on unsuccessful lookups at high load factors** on AMD EPYC. This shows up directly in benchmarks: Boost is 1.36× the leader on integer erase, while Abseil is 2.17×.

**Almost-8-bit reduced hash.** Boost stores 7.99 bits of fingerprint in each metadata byte versus Abseil's 7. The extra entropy reduces false-positive key comparisons during SIMD probe-and-match.

It also automatically applies bit-mixing for poor hash functions, so users do not have to know whether their hash is high-entropy.

## Where it wins, where it loses

In the Jackson Allan benchmark (load factor 0.875, AMD Ryzen 7 5800H, normalized to fastest):

- **Insert (int→int)** — wins decisively (1.00×; Abseil 1.11×).
- **Lookup hit** — leads at 1.28× (Abseil 1.63×).
- **Lookup miss** — essentially tied with Abseil (1.02× vs 1.00×).
- **Erase** — clear win at 1.36× (Abseil 2.17×).
- **Iteration** — beaten by [[ankerl-unordered-dense]] (which stores entries in a contiguous `std::vector`).

For iteration-heavy workloads, prefer ankerl. For extreme load factors (0.999), [[emhash]]. For lowest memory footprint, [[folly-f14]]. For concurrent access, [[parlayhash]] dominates at high thread counts.

## The concurrent variant

`boost::concurrent_flat_map` (Boost 1.83+) reuses the SIMD-probed layout with group-level locks. It posts the best single-thread number of any concurrent map (25.3 Mops), but those group locks become bottlenecks past ~16 threads where it collapses to 58 Mops at 128 threads — vs [[parlayhash]]'s 1,130 Mops. Right choice when contention is bounded; wrong choice for raw scaling.

## When to reach for it

For C++ general-purpose use, boost::unordered_flat_map is the default. Abseil is the right pick if you are already in Google's ecosystem or want the reference implementation. See [[fastest-hash-map-2025]] for the full landscape.
