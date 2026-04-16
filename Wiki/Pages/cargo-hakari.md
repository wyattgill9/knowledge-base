---
tags:
  - rust
  - performance
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# cargo-hakari

cargo-hakari manages a workspace-hack crate that unifies feature flags across workspace members, preventing duplicate dependency compilations. It delivers up to **1.7x cumulative compile-time speedup** in large workspaces. Part of the recommended [[rust-build-tooling]] stack.
