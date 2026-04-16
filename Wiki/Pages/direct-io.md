---
tags:
  - performance
  - architecture
sources:
  - "Raw/Rust/Shard-per-core Rust runtimes - Monoio, Compio, and Glommio compared.md"
last_updated: 2026-04-15
---

# Direct I/O

Direct I/O (`O_DIRECT`) bypasses the OS page cache entirely, giving the application full control over buffering and caching. This avoids double-buffering (application buffer + page cache), eliminates page cache pollution from sequential scans, and reduces memory usage dramatically.

## In Glommio

[[glommio]] is the only Rust runtime with first-class Direct I/O support. Its `DmaFile` allocates from [[io-uring]]'s pre-registered buffer pool and returns reference-counted pointers for zero-copy device-to-user transfer. `DmaStreamReader` implements intelligent read-ahead that dispatches new reads as each buffer is consumed, maintaining constant I/O parallelism — unlike OS read-ahead which issues a large batch then waits.

Benchmarks on Intel Optane: sequential scans of a 53GB file (2.5x RAM) achieved **7.29 GB/s with Direct I/O vs 0.95 GB/s buffered** (7.7x). Random reads: **547K IOPS vs 238K IOPS**, using 80KB memory vs gigabytes of page cache.

## When to use it

Direct I/O is essential for storage engines, databases, and any workload that manages its own caching. It's counterproductive for general file I/O where the OS page cache is beneficial.
