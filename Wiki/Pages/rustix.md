---
tags:
  - rust
  - performance
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# rustix

rustix provides safe Rust bindings to POSIX-like syscalls with an optional **linux_raw backend** that bypasses libc entirely, issuing direct syscalls with vDSO optimization. This makes it both safer (Rust types instead of C casts) and potentially faster than raw libc calls.

## When to use it

Use rustix for any syscall interaction beyond socket configuration (which [[socket2]] handles better). File I/O, process management, signal handling, memory mapping — rustix covers the full POSIX surface. The linux_raw backend is particularly valuable for static musl builds and minimal-dependency environments.

## Complementary with socket2

[[socket2]] provides the ergonomic `Socket` wrapper for advanced socket options. rustix provides everything else. Both are keep-worthy in a production stack.
