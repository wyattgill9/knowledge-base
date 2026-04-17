---
tags:
  - rust
  - architecture
  - crate
sources:
  - "Raw/Rust/What A+ Rust design actually looks like.md"
last_updated: 2026-04-16
---

# Bevy

Bevy is a data-driven game engine built in Rust, notable for its ECS (Entity Component System) architecture and its extreme use of the [[facade-crate-pattern]]: 40+ internal crates (`bevy_ecs`, `bevy_render`, `bevy_audio`, etc.) behind a single `bevy` facade crate. This forces clean API boundaries between subsystems while providing users with a single, ergonomic dependency.
