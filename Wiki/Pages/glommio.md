---
tags:
  - rust
  - concurrency
  - performance
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# Glommio

Glommio is Datadog's [[io-uring]]-native async runtime for Rust. It offers better ergonomics than [[monoio]] with a three-ring architecture and share-based task scheduling, though at slightly lower peak throughput. Like all io_uring runtimes, it faces the [[io-uring#cancellation-safety|cancellation safety crisis]] and is Linux-only.
