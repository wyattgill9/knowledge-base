---
tags:
  - performance
  - data-structures
  - architecture
sources:
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
last_updated: 2026-04-29
---

# Elastic Hashing

A January 2025 paper by Farach-Colton, Krapivin, and Kuszmaul that **disproved Yao's 1985 conjecture** on probe complexity for greedy (non-reordering) hash insertion. The result, featured in *Quanta Magazine*, is the most significant theoretical hash map development in 40 years and reopens a problem that the field had treated as closed.

## What Yao conjectured

In 1985, Andrew Yao conjectured that *uniform hashing* — picking a uniformly random probe sequence per key — was optimal among greedy insertion strategies (those that never relocate already-inserted elements). Under uniform hashing, expected probe count for a load factor close to 1 is **Θ(δ⁻¹)** where δ is the fraction of empty slots remaining. For a 99%-full table, that means ~100 expected probes worst-case. The conjecture stood for four decades and shaped how the community thought about open-addressing limits.

## What elastic hashing achieves

Elastic hashing achieves **O(log δ⁻¹)** worst-case expected probe complexity *without* reordering inserted elements. For the 99%-full case, that is **(log 100)² ≈ 21** worst-case probes versus the conjectured 100. The improvement compounds as tables fill: at 99.9% full, the gap is roughly 30 vs 1000.

A companion variant, **funnel hashing**, achieves **O(1) amortized** probes with matching lower bounds — proving the new bound is optimal in the amortized setting.

## Why it matters practically (eventually)

Production hash maps in 2025 ([[boost-unordered-flat-map]], [[abseil-flat-hash-map]], [[swiss-table]] generally) cap their load factor at ~0.875 specifically because probe costs explode near full. SIMD group probing makes that ceiling tolerable in practice, but the underlying math was assumed to be tight. Elastic hashing suggests that fundamentally new probing strategies could push that ceiling much higher — closer to memory bandwidth limits — without giving up greedy semantics.

The caveat: this is a theoretical breakthrough, not yet a benchmark winner. Translating O(log δ⁻¹) bounds into wall-clock wins requires the constants to be small and the access pattern to remain SIMD-friendly. Whether elastic or funnel hashing can be implemented with the cache and SIMD discipline of Swiss Tables is an open engineering question.

## In context

Other recent theoretical work includes [[iceberg-hashing]] (JACM 2023), which optimizes space, cache, and stability simultaneously for persistent memory workloads. [[folly-f14|Meta's F14]] uses overflow counting traceable to Amble-Knuth 1974 — sometimes the right move is to revisit very old ideas with new hardware in mind.

Elastic hashing is the rare result that tells the field "the limit you assumed was real is not." See [[fastest-hash-map-2025]] for surrounding context.
