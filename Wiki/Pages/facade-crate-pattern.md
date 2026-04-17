---
tags:
  - rust
  - architecture
  - design-patterns
sources:
  - "Raw/Rust/What A+ Rust design actually looks like.md"
last_updated: 2026-04-16
---

# Facade Crate Pattern

The facade crate pattern is the dominant architecture for large Rust projects: a workspace of specialized internal crates with a top-level crate that re-exports a curated public API. The separation forces clean interfaces — if two modules are in different crates, they *must* communicate through explicit public APIs.

## Production examples

- **[[bevy]]** — 40+ internal crates (`bevy_ecs`, `bevy_render`, `bevy_audio`, etc.) behind a single `bevy` facade
- **[[tokio]]** — `tokio-macros`, `tokio-stream`, `tokio-util`, etc. behind the `tokio` facade with granular feature flags (`rt`, `net`, `io-util`, `time`, `sync`) plus a `full` convenience feature
- **ripgrep** — 9 crates (`grep-regex`, `grep-searcher`, `grep-printer`, `ignore`, `globset`), each with a single responsibility

## Feature flag discipline

Feature flags in facade crates must follow the **additive principle**: enabling more features must never break existing code. Mutually exclusive features are an anti-pattern. Use the `dep:` prefix (stable since Rust 1.60) to avoid leaking dependency names as implicit features. See [[rust-workspace-patterns]] for the full workspace configuration.

## Why it works

The pattern provides compile-time modularity (each crate compiles independently), enforcement of API boundaries (no reaching into internal modules), and independent versioning when needed. It scales from mid-sized projects to ecosystem-defining crates.

See [[expert-rust-design]] for the broader design philosophy and [[rust-workspace-patterns]] for workspace configuration details.
