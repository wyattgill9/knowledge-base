---
tags:
  - rust
  - performance
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# socket2

socket2 provides an ergonomic `Socket` wrapper for advanced socket configuration in Rust — `SO_REUSEPORT`, `TCP_CORK`, BPF filters, and other options that `std::net` doesn't expose.

## Relationship with rustix

socket2 and [[rustix]] are complementary, not competing. Use socket2 for socket-level configuration (it wraps the socket API specifically and does it well), and rustix for broader syscall coverage including its optional **linux_raw backend** that bypasses libc entirely for direct syscalls with vDSO optimization. Both belong in a high-performance networking stack alongside [[tokio]].
