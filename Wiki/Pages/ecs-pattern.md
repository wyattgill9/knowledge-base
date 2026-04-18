---
tags:
  - architecture
  - performance
  - data-structures
  - design-patterns
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# Entity Component System (ECS)

A data-oriented architecture pattern that operationalizes [[soa-vs-aos|SoA (Structure of Arrays)]] at scale, primarily in game engines. ECS separates identity (entities), data (components), and behavior (systems), storing components in cache-friendly layouts that enable high-throughput iteration.

## Two storage models

**Archetype-based** (Flecs, [[bevy|Bevy ECS]]): Entities sharing the same component set are co-located in SoA tables. A "Player with Position + Velocity + Health" archetype stores all Positions contiguously, all Velocities contiguously, etc. Multi-component queries iterate densely packed memory — excellent cache behavior.

**Sparse-set-based** (EnTT): Each component type has its own packed array plus a sparse lookup array for O(1) entity-component access. Single-component queries and fast add/remove are faster than archetype systems, at the cost of multi-component query locality.

**Trade-off:** Archetype systems win for multi-component queries (the common case in game logic). Sparse-set systems win for single-component queries and when entities frequently change their component set.

## Production examples

**EnTT** (C++) — used by Minecraft Bedrock Edition. Sparse-set design with packed arrays for iteration.

**Flecs** (C/C++) — archetype-based with advanced features like query caching and relationship modeling.

**[[bevy|Bevy ECS]]** (Rust) — archetype-based, deeply integrated into the Bevy game engine. Demonstrates how ECS can be a general application architecture, not just a game engine pattern.

## Why ECS is fast

The performance advantage is entirely about memory layout. A traditional OOP game object stores all its data together — but a physics system only needs Position and Velocity, wasting cache on Health, Inventory, AI State, etc. ECS stores each component type contiguously, so the physics system iterates only the data it needs. This is [[soa-vs-aos|SoA]] applied to an entire application architecture.
