# Index

| Page | Description | Tags |
|------|-------------|------|
| [[anyhow]] | Simple dynamic error boxing for Rust application code | rust, error-handling, crate |
| [[apache-iggy]] | Message streaming system; most documented production migration to shard-per-core (Compio) | rust, performance, architecture |
| [[bitcode]] | Fastest traditional serialization crate with smallest output size, complementary to rkyv | rust, performance, data-structures, crate |
| [[blazingly-fast-rust-crate-stack]] | Survey of the high-performance Rust crate ecosystem for 2025–2026 | rust, performance, crate, architecture |
| [[bolero]] | Unified fuzzing and property testing under a single API | rust, crate |
| [[bumpalo]] | Bump allocator for phase-oriented work with ~2 ns allocation and instant bulk deallocation | rust, performance, data-structures, crate |
| [[bytemuck]] | Safe plain-data type casting, widely used in Solana/wgpu; superseded by zerocopy for new code | rust, performance, crate |
| [[bytes]] | Arc-backed byte buffers for zero-copy sharing throughout the Tokio ecosystem | rust, performance, crate |
| [[cache-coherency]] | MESI protocol costs and why thread-per-core eliminates them (50–200 ns/line under contention) | performance, concurrency, architecture |
| [[cargo-deny]] | Dependency graph auditor for licenses, vulnerabilities, duplicates, and banned crates | rust, crate |
| [[cargo-hakari]] | Workspace-hack crate manager that unifies feature flags for up to 1.7x compile speedup | rust, performance, crate |
| [[cargo-nextest]] | Parallel test runner replacing cargo test with per-process isolation | rust, crate |
| [[cargo-profile-optimization]] | Cargo.toml release profile tuning for 10–20% runtime improvement with no code changes | rust, performance, architecture |
| [[compio]] | Recommended shard-per-core runtime for 2026; cross-platform, pluggable driver, validated by Apache Iggy | rust, concurrency, performance, crate |
| [[criterion]] | Mature statistical benchmarking with HTML reports; reduced maintainer activity | rust, performance, crate |
| [[crossbeam-channel]] | Proven synchronous MPMC message passing channel | rust, concurrency, crate |
| [[dashmap]] | Sharded RwLock concurrent HashMap with best write throughput and familiar API | rust, concurrency, data-structures, crate |
| [[deterministic-simulation-testing]] | Testing methodology replacing I/O and time with deterministic mocks for reproducible distributed system testing | rust, architecture, concurrency |
| [[direct-io]] | O_DIRECT bypassing OS page cache; Glommio achieves 7.7x over buffered I/O on NVMe | performance, architecture |
| [[divan]] | Modern benchmarking crate with ergonomic attribute macros and allocation counting | rust, performance, crate |
| [[eyre]] | Dynamic error handling with rich reports and custom handlers | rust, error-handling, crate |
| [[foldhash]] | Default hasher for hashbrown 0.15+, replacing ahash with better performance and smaller footprint | rust, performance, data-structures, crate |
| [[glommio]] | Datadog's io_uring runtime with triple-ring architecture, proportional-share scheduling, and DMA storage I/O; effectively unmaintained | rust, concurrency, performance, crate |
| [[hashbrown]] | Swiss-table HashMap underlying std::HashMap, now using foldhash as default hasher | rust, data-structures, crate |
| [[iai-callgrind]] | Deterministic instruction-count benchmarking via Valgrind for CI regression detection | rust, performance, crate |
| [[io-uring]] | Linux completion-based async I/O with cancellation safety, ecosystem incompatibility, and container security challenges | rust, performance, concurrency, architecture |
| [[kanal]] | High-throughput message passing channel leading benchmarks at 8–16M msg/sec | rust, concurrency, crate |
| [[mimalloc]] | Microsoft's memory allocator delivering up to 5.3x faster allocation than glibc malloc | rust, performance, crate |
| [[monoio]] | ByteDance's io_uring runtime with zero-copy slab-allocated I/O; best for Linux-only network proxies | rust, concurrency, performance, crate |
| [[papaya]] | Lock-free concurrent HashMap with novel memory reclamation, ideal for async read-heavy caches | rust, concurrency, data-structures, crate |
| [[proptest]] | Property-based testing with automatic shrinking counterexamples | rust, crate |
| [[pulp]] | Portable SIMD on stable Rust with runtime CPU feature detection and dispatch | rust, performance, crate |
| [[rapidhash]] | Fastest non-cryptographic hash function overall (4.25 ns geometric mean) | rust, performance, data-structures, crate |
| [[rkyv]] | Zero-copy deserialization framework with ~21 ns access time, uncontested in its niche | rust, performance, data-structures, crate |
| [[rust-build-tooling]] | Overview of cargo ecosystem tools for compilation, quality, and supply-chain security | rust, performance, architecture |
| [[rust-concurrent-data-structures]] | Landscape of concurrent maps and channels beyond std | rust, concurrency, data-structures, architecture |
| [[rust-error-handling]] | The Rust error-handling ecosystem: typed vs dynamic, thiserror vs snafu | rust, error-handling, architecture |
| [[rust-memory-allocators]] | Comparison of mimalloc and jemalloc as global allocator replacements | rust, performance, architecture |
| [[rustix]] | Safe syscall bindings with optional linux_raw backend bypassing libc | rust, performance, crate |
| [[scc]] | Scalable concurrent HashMap optimized for extreme write contention | rust, concurrency, data-structures, crate |
| [[seastar]] | ScyllaDB's C++ shard-per-core framework; ancestor of Glommio and the thread-per-core pattern | cpp, concurrency, performance, architecture |
| [[shard-per-core-runtimes-compared]] | Detailed comparison of Monoio, Compio, and Glommio with architecture table and decision framework | rust, concurrency, performance, architecture |
| [[slotmap]] | Generational-index arenas preventing ABA problems with stable handles | rust, data-structures, crate |
| [[snafu]] | Error-handling crate using context selectors for enforced per-module error granularity | rust, error-handling, crate |
| [[socket2]] | Ergonomic wrapper for advanced socket configuration (SO_REUSEPORT, TCP_CORK, BPF) | rust, performance, crate |
| [[thiserror]] | Ecosystem-standard derive macro for custom error types, with no_std since 2.0 | rust, error-handling, crate |
| [[thread-per-core]] | Server architecture eliminating cache coherency costs; 2–3x Tokio at 16+ cores but 3x overprovisioning penalty | rust, performance, concurrency, architecture |
| [[tikv-jemallocator]] | jemalloc wrapper with most stable allocation latency and built-in heap profiling | rust, performance, crate |
| [[tokio]] | Rust's dominant async runtime; ecosystem moat makes it the correct default despite shard-per-core alternatives | rust, concurrency, crate, performance |
| [[tracing]] | Ecosystem standard for structured diagnostics with ~1 ns disabled overhead | rust, crate, architecture |
| [[zerocopy]] | Google's formally verified zero-cost byte conversions, superior to bytemuck for new code | rust, performance, crate |
