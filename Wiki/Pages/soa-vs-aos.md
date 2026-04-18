---
tags:
  - performance
  - architecture
  - data-structures
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# SoA vs AoS (Structure of Arrays vs Array of Structures)

Perhaps the single most impactful optimization pattern in data structure design. When processing 1–2 fields across many entities, SoA delivers **1.4x to 12.7x speedup** depending on struct size and field ratio.

## The core idea

**AoS (Array of Structures):** `[{x,y,z,w}, {x,y,z,w}, ...]` — each entity's fields are contiguous. Good when you always access all fields of one entity.

**SoA (Structure of Arrays):** `{xs: [...], ys: [...], zs: [...], ws: [...]}` — each field is a contiguous array. Good when you access one field across many entities.

A Go benchmark showed **100,404 ns/op for AoS vs 7,913 ns/op for SoA** (12.7x faster) when structs were large and only a subset of fields was accessed. The reason: SoA loads only the data you need into cache lines, while AoS wastes cache space on fields you don't touch.

## SIMD vectorization

SoA enables direct [[simd-programming|SIMD]] vectorization since contiguous same-type arrays map perfectly to vector load instructions. You can process 4/8/16 elements per instruction with AVX2/AVX-512 — impossible with AoS layout where fields of different types are interleaved.

## The ECS pattern

The [[ecs-pattern|Entity Component System]] pattern in game engines operationalizes SoA at scale. [[bevy|Bevy ECS]] (Rust) and **Flecs** (C/C++) use archetype-based storage where entities sharing the same component set are co-located in SoA tables. **EnTT** (C++, used by Minecraft Bedrock Edition) uses sparse sets for O(1) entity-component lookup with packed arrays for iteration.

## When to use AoS

AoS wins when you typically access all fields of a single entity (random point lookups), when the struct is small enough to fit in one cache line, or when code clarity matters more than performance. The crossover point depends on struct size and access pattern — profile to decide.

## Connection to other performance patterns

This is the same cache-locality principle that makes [[b-tree]] beat red-black trees (packed vs pointer-chasing), [[csr-graph|CSR]] beat adjacency lists (contiguous vs scattered), and [[swiss-table]] beat chaining hash maps (flat vs linked). Contiguous memory layouts exploit hardware prefetchers, and modern CPUs can prefetch sequential access patterns but not pointer-chasing patterns.
