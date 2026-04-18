# Index

| Page | Description | Tags |
|------|-------------|------|
| [[ambassador]] | Most general-purpose Rust trait delegation crate; structs and enums, cross-crate, generics | rust, design-patterns, crate |
| [[anyhow]] | Type-erased application error with string context, backtraces on stable, and no_std support | rust, error-handling, crate |
| [[apache-iggy]] | Message streaming system; most documented production migration to shard-per-core (Compio) | rust, performance, architecture |
| [[ariadne]] | Pretty diagnostic rendering with source spans and fine-grained label control | rust, error-handling, crate |
| [[auto-enums]] | Auto-generate hidden dispatch enums for functions returning different concrete types | rust, design-patterns, crate |
| [[axum-architecture]] | How axum uses tower::Service, extractors, and IntoResponse for trait-driven web framework design | rust, architecture, design-patterns |
| [[bitcode]] | Fastest traditional serialization crate with smallest output size, complementary to rkyv | rust, performance, data-structures, crate |
| [[bitflags]] | C-style bitflag sets via macro; ~1.1B downloads, used by the Rust compiler | rust, data-structures, crate |
| [[bevy]] | Data-driven game engine with 40+ internal crates behind a facade; extreme example of workspace modularity | rust, architecture, crate |
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
| [[codespan-reporting]] | Structured diagnostic pipeline for compiler-style error messages with source spans | rust, error-handling, crate |
| [[color-eyre]] | Colorful eyre report handler with SpanTrace integration and panic hooks; repository archived | rust, error-handling, crate |
| [[compio]] | Recommended shard-per-core runtime for 2026; cross-platform, pluggable driver, validated by Apache Iggy | rust, concurrency, performance, crate |
| [[criterion]] | Mature statistical benchmarking with HTML reports; reduced maintainer activity | rust, performance, crate |
| [[crossbeam-channel]] | Proven synchronous MPMC message passing channel | rust, concurrency, crate |
| [[dashmap]] | Sharded RwLock concurrent HashMap with best write throughput and familiar API | rust, concurrency, data-structures, crate |
| [[derive-more]] | Swiss-army knife derive macro (200M+): From, Into, Display, Error, operators, utility methods | rust, data-structures, error-handling, crate |
| [[deterministic-simulation-testing]] | Testing methodology replacing I/O and time with deterministic mocks for reproducible distributed system testing | rust, architecture, concurrency |
| [[direct-io]] | O_DIRECT bypassing OS page cache; Glommio achieves 7.7x over buffered I/O on NVMe | performance, architecture |
| [[displaydoc]] | Derive Display from doc comments, no_std compatible | rust, error-handling, crate |
| [[divan]] | Modern benchmarking crate with ergonomic attribute macros and allocation counting | rust, performance, crate |
| [[either]] | Canonical Either<L, R> sum type with trait delegation | rust, data-structures, crate |
| [[enum-as-inner]] | Derive as_*(), into_*(), is_*() accessor methods for enum variants | rust, data-structures, crate |
| [[expert-rust-design]] | What separates A+ Rust from average: compile-time invariants, API design, crate architecture, knowing when to stop | rust, architecture, design-patterns |
| [[enum-dispatch]] | Static enum dispatch replacing dyn Trait at up to 10x speedup | rust, performance, design-patterns, crate |
| [[enum-map]] | O(1) array-backed map keyed by enum variants with exhaustive initialization | rust, data-structures, crate |
| [[error-chain]] | Legacy unmaintained error-handling crate; migrate to thiserror/snafu + anyhow/eyre | rust, error-handling, crate |
| [[error-stack]] | Stacked Report with typed contexts and arbitrary attachments for observability-first error handling | rust, error-handling, crate |
| [[eyre]] | Fork of anyhow with swappable report handlers for customizable error presentation | rust, error-handling, crate |
| [[facade-crate-pattern]] | Workspace of specialized crates behind a curated public API; used by Bevy, Tokio, ripgrep | rust, architecture, design-patterns |
| [[failure]] | Deprecated error-handling crate (RustSec EOL); migrate to anyhow/thiserror | rust, error-handling, crate |
| [[foldhash]] | Default hasher for hashbrown 0.15+, replacing ahash with better performance and smaller footprint | rust, performance, data-structures, crate |
| [[frunk]] | Haskell/Scala-style generic programming: HList, Coproduct, Generic/LabelledGeneric, Validated | rust, type-theory, data-structures, crate |
| [[gats]] | Generic Associated Types: lending iterators, pointer families, lifetime-parameterized associated types | rust, type-theory, architecture |
| [[generic-array]] | Arrays generic over length via typenum; foundation for RustCrypto ecosystem | rust, type-theory, data-structures, crate |
| [[glommio]] | Datadog's io_uring runtime with triple-ring architecture, proportional-share scheduling, and DMA storage I/O; effectively unmaintained | rust, concurrency, performance, crate |
| [[hashbrown]] | Swiss-table HashMap underlying std::HashMap, now using foldhash as default hasher | rust, data-structures, crate |
| [[heapless]] | Stack-allocated fixed-capacity collections via const generics for embedded and no_std | rust, performance, data-structures, crate |
| [[iai-callgrind]] | Deterministic instruction-count benchmarking via Valgrind for CI regression detection | rust, performance, crate |
| [[io-uring]] | Linux completion-based async I/O with cancellation safety, ecosystem incompatibility, and container security challenges | rust, performance, concurrency, architecture |
| [[itertools]] | Canonical extension trait crate for Iterator with chunks, tuple_windows, interleave, and dozens more | rust, crate |
| [[kanal]] | High-throughput message passing channel leading benchmarks at 8–16M msg/sec | rust, concurrency, crate |
| [[lazy-lock]] | LazyLock/LazyCell replacing lazy_static and once_cell in std (Rust 1.80) | rust, architecture |
| [[miette]] | Diagnostic-first errors with source spans, labels, help text, and rich rendering for compiler-style UX | rust, error-handling, crate |
| [[mimalloc]] | Microsoft's memory allocator delivering up to 5.3x faster allocation than glibc malloc | rust, performance, crate |
| [[modern-rust-features]] | Definitive guide to Rust 2023–2026: edition 2024, async evolution, type system, pattern matching, stdlib | rust, architecture |
| [[monoio]] | ByteDance's io_uring runtime with zero-copy slab-allocated I/O; best for Linux-only network proxies | rust, concurrency, performance, crate |
| [[newtype-pattern]] | Zero-cost primitive wrapping with private constructors for parse-don't-validate | rust, type-theory, design-patterns |
| [[num-enum]] | Safe integer-to-enum conversions with alternatives, catch-all, and TryFrom for FFI/protocols | rust, data-structures, crate |
| [[nutype]] | Proc macro for generating validated newtypes with sanitization rules (2.7M+) | rust, type-theory, crate |
| [[papaya]] | Lock-free concurrent HashMap with novel memory reclamation, ideal for async read-heavy caches | rust, concurrency, data-structures, crate |
| [[proptest]] | Property-based testing with automatic shrinking counterexamples | rust, crate |
| [[pulp]] | Portable SIMD on stable Rust with runtime CPU feature detection and dispatch | rust, performance, crate |
| [[rapidhash]] | Fastest non-cryptographic hash function overall (4.25 ns geometric mean) | rust, performance, data-structures, crate |
| [[recursion-schemes]] | Catamorphisms and anamorphisms with stack safety and arena-based traversal for AST-heavy code | rust, data-structures, design-patterns, crate |
| [[rkyv]] | Zero-copy deserialization framework with ~21 ns access time, uncontested in its niche | rust, performance, data-structures, crate |
| [[rust-abstraction-boundaries]] | Enums vs generics vs trait objects; macros as last resort; the overengineering trap | rust, architecture, design-patterns |
| [[rust-allocation-patterns]] | Borrow → Cow → owned hierarchy; niche optimization; buffer reuse; zero-cost abstraction caveats | rust, performance, architecture |
| [[rust-api-design]] | Accept broad, return specific; de-generification; builder tiers; extension traits; visibility control | rust, design-patterns, architecture |
| [[rust-2024-edition]] | The largest Rust edition: RPIT capture, explicit unsafe, tighter temporaries, async prelude, resolver 3 | rust, architecture |
| [[rust-async-evolution]] | Async fn in traits (1.75), async closures (1.85), structured concurrency with JoinSet | rust, concurrency, architecture |
| [[rust-build-tooling]] | Overview of cargo ecosystem tools for compilation, quality, and supply-chain security | rust, performance, architecture |
| [[rust-concurrency-patterns]] | Arc<Mutex> as last resort; decision framework; data-oriented state splitting; async task patterns | rust, concurrency, architecture |
| [[rust-concurrent-data-structures]] | Landscape of concurrent maps and channels beyond std | rust, concurrency, data-structures, architecture |
| [[rust-enum-crates]] | Survey of 60+ crates extending Rust enums: utilities, dispatch, type-level programming, recursion schemes | rust, data-structures, design-patterns, type-theory |
| [[rust-error-crate-comparison]] | Comprehensive comparative analysis of the Rust error-handling crate ecosystem with decision matrix | rust, error-handling, architecture |
| [[rust-error-handling]] | The four-layer Rust error-handling architecture: typed definition, type-erased reporting, diagnostics, and observability | rust, error-handling, architecture |
| [[rust-memory-allocators]] | Comparison of mimalloc and jemalloc as global allocator replacements | rust, performance, architecture |
| [[parse-dont-validate]] | Design principle: return refined types from validation, not booleans — the foundation of type-driven design | rust, type-theory, design-patterns |
| [[rust-pattern-matching]] | let-else, let-chains, exclusive ranges, exhaustive patterns — modern pattern matching ergonomics | rust, architecture |
| [[rust-testing-patterns]] | Layered testing (unit, integration, doc, property-based) and CI lint enforcement | rust, architecture, design-patterns |
| [[rust-workspace-patterns]] | Virtual workspaces with dependency/lint inheritance, feature flag best practices, cross-crate error chaining | rust, architecture |
| [[rustix]] | Safe syscall bindings with optional linux_raw backend bypassing libc | rust, performance, crate |
| [[scc]] | Scalable concurrent HashMap optimized for extreme write contention | rust, concurrency, data-structures, crate |
| [[sealed-traits]] | Closed type sets via private supertrait; essential for typestate and domain modeling | rust, type-theory, design-patterns |
| [[serde-architecture]] | Serde's zero-allocation data model via traits; dual extensibility without intermediate representations | rust, architecture, design-patterns |
| [[smallvec]] | Stack-inline Vec up to N elements with heap fallback; avoids allocation on the common small-collection path | rust, performance, data-structures, crate |
| [[seastar]] | ScyllaDB's C++ shard-per-core framework; ancestor of Glommio and the thread-per-core pattern | cpp, concurrency, performance, architecture |
| [[shard-per-core-runtimes-compared]] | Detailed comparison of Monoio, Compio, and Glommio with architecture table and decision framework | rust, concurrency, performance, architecture |
| [[slotmap]] | Generational-index arenas preventing ABA problems with stable handles | rust, data-structures, crate |
| [[snafu]] | Error-handling crate using context selectors for structured semantic backtraces, with progressive design from Whatever to opaque types | rust, error-handling, crate |
| [[snafu-tutorial]] | Deep-dive tutorial on SNAFU: context selectors, backtrace patterns, async integration, boundary conversions, and migration from thiserror | rust, error-handling |
| [[socket2]] | Ergonomic wrapper for advanced socket configuration (SO_REUSEPORT, TCP_CORK, BPF) | rust, performance, crate |
| [[strum]] | Standard enum ergonomics crate (100M+): Display, FromStr, EnumIter, EnumCount, metadata, discriminants | rust, data-structures, crate |
| [[thiserror]] | Ecosystem-standard derive macro (~858M downloads); invisible in public API, no_std since 2.0 via core::error::Error | rust, error-handling, crate |
| [[thread-per-core]] | Server architecture eliminating cache coherency costs; 2–3x Tokio at 16+ cores but 3x overprovisioning penalty | rust, performance, concurrency, architecture |
| [[tikv-jemallocator]] | jemalloc wrapper with most stable allocation latency and built-in heap profiling | rust, performance, crate |
| [[tokio]] | Rust's dominant async runtime with JoinSet structured concurrency; ecosystem moat makes it the correct default | rust, concurrency, crate, performance |
| [[tracing]] | Ecosystem standard for structured diagnostics with ~1 ns disabled overhead | rust, crate, architecture |
| [[tracing-error]] | Enriches errors with tracing span context; SpanTrace beats backtraces in async code | rust, error-handling, concurrency, crate |
| [[trait-upcasting]] | Coerce &dyn Sub to &dyn Super (Rust 1.86); works with Box, Arc, Rc | rust, type-theory |
| [[type-driven-development]] | Methodology: types as specifications, seven techniques from newtypes to typestates, pragmatic limits | rust, type-theory, design-patterns, architecture |
| [[typed-builder]] | Derive macro for builders with compile-time required field checking | rust, type-theory, crate |
| [[typestate-pattern]] | Crown jewel of type-driven Rust: state machines checked at compile time via move semantics | rust, type-theory, design-patterns |
| [[typenum]] | Type-level numbers with compile-time arithmetic; foundation for generic-array and RustCrypto | rust, type-theory, crate |
| [[zerocopy]] | Google's formally verified zero-cost byte conversions, superior to bytemuck for new code | rust, performance, crate |
