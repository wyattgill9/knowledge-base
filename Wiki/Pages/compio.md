---
tags:
  - rust
  - concurrency
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# Compio

Compio is the only cross-platform completion-based async runtime for Rust, supporting [[io-uring]] on Linux, IOCP on Windows, and a polling fallback elsewhere. It's the right choice when you want completion-based I/O semantics without being locked to Linux. See [[io-uring]] for the broader landscape.
