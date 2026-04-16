---
tags:
  - rust
  - data-structures
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# slotmap

slotmap provides generational-index arenas in Rust — a pattern where handles (keys) include a generation counter that detects use-after-free at runtime, preventing ABA problems. It ships three variants optimized for different access patterns.

## When to use it

Use slotmap when you need an arena with **stable handles and individual element removal** — entity-component systems, graph nodes, resource managers. Unlike [[bumpalo]] (which only supports bulk deallocation), slotmap lets you insert and remove individual elements while keeping existing handles valid.

## Generational indices

The key insight: each slot stores a generation counter. When an element is removed and the slot reused, the generation increments. Old keys with stale generations are detected on access, turning use-after-free into a safe, explicit error.
