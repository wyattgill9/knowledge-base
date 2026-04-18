---
tags:
  - rust
  - architecture
  - crate
sources:
  - "Raw/Rust/What A+ Rust design actually looks like.md"
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# Bevy

Bevy is a data-driven game engine built in Rust, notable for its [[ecs-pattern|ECS (Entity Component System)]] architecture and its extreme use of the [[facade-crate-pattern]]: 40+ internal crates (`bevy_ecs`, `bevy_render`, `bevy_audio`, etc.) behind a single `bevy` facade crate. This forces clean API boundaries between subsystems while providing users with a single, ergonomic dependency.

Bevy ECS uses archetype-based storage where entities sharing the same component set are co-located in [[soa-vs-aos|SoA]] tables — delivering excellent multi-component iteration cache behavior. This makes it one of the most prominent production examples of the [[ecs-pattern]], alongside EnTT (C++, Minecraft Bedrock Edition) and Flecs (C/C++).
