# Index

| Page | Description | Tags |
|------|-------------|------|
| [[abseil-flat-hash-map]] | Google's reference Swiss Table; production standard, narrowly trails Boost in 2025 benchmarks | cpp, performance, data-structures |
| [[absl-inlinedvector]] | Google's drop-in std::vector with inline storage; perfect API parity but ~5x random-access slowdown | cpp, performance, data-structures |
| [[adaptive-radix-tree]] | SIMD-optimized trie with four adaptive node sizes; O(k) lookup independent of n; concurrent variant ART-OLC | performance, data-structures |
| [[algorithmica-s-tree]] | Sergey Slotin's reference SIMD B-tree; 7–18× over std::set, 3–7× over Abseil, in 150 lines of C++ | cpp, performance, data-structures |
| [[aggregating-funnels]] | PPoPP 2025 software FAA combining; up to 2.5× over plain LCRQ at high thread counts | concurrency, performance, data-structures |
| [[ankerl-svector]] | Single-header SBO vector with 8-byte minimum sizeof; 7 inline bytes, 1 byte overhead | cpp, performance, data-structures |
| [[ankerl-unordered-dense]] | Single-header C++ hash map; uncontested iteration champion via dense std::vector storage | cpp, performance, data-structures |
| [[ambassador]] | Most general-purpose Rust trait delegation crate; structs and enums, cross-crate, generics | rust, design-patterns, crate |
| [[anyhow]] | Type-erased application error with string context, backtraces on stable, and no_std support | rust, error-handling, crate |
| [[apache-iggy]] | Message streaming system; most documented production migration to shard-per-core (Compio) | rust, performance, architecture |
| [[art-olc]] | ART with optimistic lock coupling; fastest concurrent ordered map on integer keys; readers never write shared cache lines | concurrency, performance, data-structures |
| [[ariadne]] | Pretty diagnostic rendering with source spans and fine-grained label control | rust, error-handling, crate |
| [[atomic-queue]] | C++ header-only lock-free MPMC; OptimistAtomicQueue at 200–500M+ ops/s with sub-100 ns latency | cpp, concurrency, performance |
| [[auto-enums]] | Auto-generate hidden dispatch enums for functions returning different concrete types | rust, design-patterns, crate |
| [[axum-architecture]] | How axum uses tower::Service, extractors, and IntoResponse for trait-driven web framework design | rust, architecture, design-patterns |
| [[bitcode]] | Fastest traditional serialization crate with smallest output size, complementary to rkyv | rust, performance, data-structures, crate |
| [[bitflags]] | C-style bitflag sets via macro; ~1.1B downloads, used by the Rust compiler | rust, data-structures, crate |
| [[bacq]] | Boundary-Aware Concurrent Queue (2024); 2–9× over previous GPU queue designs | concurrency, performance, data-structures |
| [[b-epsilon-tree]] | Fractal tree index; ~32× fewer write I/Os than B-trees via internal-node update buffering; principle survives in LSMs | data-structures, performance, architecture |
| [[bp-tree]] | VLDB 2023 concurrent B+-tree with OLC; 7.4× over Masstree on points, 30× on range scans | performance, data-structures, concurrency |
| [[bs-tree]] | ICDE 2026 SIMD B-tree; 16 keys/node in 2 AVX-512 instructions; gapped nodes for branchless updates | performance, data-structures |
| [[b-tree]] | Ordered container beating red-black trees by 7–18× through cache-line-sized nodes and SIMD search | performance, data-structures |
| [[bw-tree]] | Microsoft's lock-free B+-tree; 1.5–4.5× slower than ART-OLC because lock-freedom ≠ cache-coherence-freedom | concurrency, data-structures |
| [[bevy]] | Data-driven game engine with ECS architecture and 40+ internal crates behind a facade | rust, architecture, crate |
| [[binary-fuse-filter]] | Probabilistic set membership filter within 13% of information-theoretic minimum; surpasses Bloom | performance, data-structures |
| [[blazingly-fast-rust-crate-stack]] | Survey of the high-performance Rust crate ecosystem for 2025–2026 | rust, performance, crate, architecture |
| [[boost-unordered-flat-map]] | 2025 consensus fastest general-purpose C++ hash map; overflow byte beats Abseil's tombstones | cpp, performance, data-structures |
| [[bolero]] | Unified fuzzing and property testing under a single API | rust, crate |
| [[boxing-overhead]] | Why Java ArrayList is 40-100x slower than C++ vector; reified vs erased generics | performance, architecture, data-structures |
| [[bumpalo]] | Bump allocator for phase-oriented work with ~2 ns allocation and instant bulk deallocation | rust, performance, data-structures, crate |
| [[busy-spin]] | The PAUSE-loop discipline; sub-100 ns RTT at 100% core cost; 15× generational variance on Intel | concurrency, performance, architecture |
| [[bytemuck]] | Safe plain-data type casting, widely used in Solana/wgpu; superseded by zerocopy for new code | rust, performance, crate |
| [[bytes]] | Arc-backed byte buffers for zero-copy sharing throughout the Tokio ecosystem | rust, performance, crate |
| [[cache-coherency]] | MESI protocol costs, cache-friendly design, and why thread-per-core eliminates coherency traffic | performance, concurrency, architecture |
| [[cache-oblivious-structures]] | Data structures optimizing for every cache level simultaneously without knowing cache parameters | performance, data-structures, architecture |
| [[champ]] | Compressed Hash-Array Mapped Prefix-tree powering Clojure/Scala immutable collections; 10–100% over HAMT | data-structures, performance |
| [[concurrent-queues]] | The 2025 MPMC hierarchy from LCRQ+Funnels down to Michael-Scott; SPSC tier, channels, stacks | concurrency, performance, data-structures |
| [[congee]] | Rust port of ART-OLC; 150 Mops/sec on 32 cores; closest Rust gets to the global concurrent-ordered-map frontier | rust, concurrency, performance, data-structures, crate |
| [[crystalline-reclamation]] | PLDI 2024 wait-free reclamation with bounded memory simultaneously — previously thought impossible | concurrency, performance, data-structures, architecture |
| [[cuckoo-trie]] | 2021 trie exploiting memory-level parallelism; 20–360% over state-of-the-art via independent miss streams | data-structures, performance, architecture |
| [[core-pinning]] | taskset, isolcpus, SCHED_FIFO; cuts P99.9 latency from 120 µs to 30 µs; HFT foundation | performance, concurrency, architecture |
| [[crossbeam-array-queue]] | Vyukov-style bounded MPMC queue from crossbeam; closest Rust equivalent to LCRQ-class designs | rust, concurrency, data-structures, crate |
| [[crossbeam-epoch]] | Epoch-based memory reclamation for lock-free Rust data structures; RCU equivalent | rust, concurrency, crate |
| [[csr-graph]] | Compressed Sparse Row — fastest read-only graph representation; 40–250x faster than NetworkX | performance, data-structures, architecture |
| [[cargo-deny]] | Dependency graph auditor for licenses, vulnerabilities, duplicates, and banned crates | rust, crate |
| [[cargo-hakari]] | Workspace-hack crate manager that unifies feature flags for up to 1.7x compile speedup | rust, performance, crate |
| [[cargo-nextest]] | Parallel test runner replacing cargo test with per-process isolation | rust, crate |
| [[cargo-profile-optimization]] | Cargo.toml release profile tuning for 10–20% runtime improvement with no code changes | rust, performance, architecture |
| [[codespan-reporting]] | Structured diagnostic pipeline for compiler-style error messages with source spans | rust, error-handling, crate |
| [[color-eyre]] | Colorful eyre report handler with SpanTrace integration and panic hooks; repository archived | rust, error-handling, crate |
| [[compio]] | Recommended shard-per-core runtime for 2026; cross-platform, pluggable driver, validated by Apache Iggy | rust, concurrency, performance, crate |
| [[criterion]] | Mature statistical benchmarking with HTML reports; reduced maintainer activity | rust, performance, crate |
| [[crossbeam-channel]] | Proven synchronous MPMC message passing channel | rust, concurrency, crate |
| [[d-ary-heap]] | Fastest practical heap (d=4); 17–30% faster than binary heaps, decisively beats Fibonacci heaps | performance, data-structures |
| [[dashmap]] | Sharded RwLock concurrent HashMap with best write throughput and familiar API | rust, concurrency, data-structures, crate |
| [[derive-more]] | Swiss-army knife derive macro (200M+): From, Into, Display, Error, operators, utility methods | rust, data-structures, error-handling, crate |
| [[driftsort]] | Rust's standard stable sort since 2024; up to 4x faster than previous sort via lazy runs and branchless partitioning | rust, performance, data-structures |
| [[deterministic-simulation-testing]] | Testing methodology replacing I/O and time with deterministic mocks for reproducible distributed system testing | rust, architecture, concurrency |
| [[direct-io]] | O_DIRECT bypassing OS page cache; Glommio achieves 7.7x over buffered I/O on NVMe | performance, architecture |
| [[displaydoc]] | Derive Display from doc comments, no_std compatible | rust, error-handling, crate |
| [[divan]] | Modern benchmarking crate with ergonomic attribute macros and allocation counting | rust, performance, crate |
| [[either]] | Canonical Either<L, R> sum type with trait delegation | rust, data-structures, crate |
| [[elastic-hashing]] | Jan 2025 disproof of Yao's 1985 conjecture; O(log δ⁻¹) probes without reordering | performance, data-structures, architecture |
| [[emhash]] | C++ hash map for raw insert/erase throughput; tolerates 0.999 load factor with no tombstones | cpp, performance, data-structures |
| [[enum-as-inner]] | Derive as_*(), into_*(), is_*() accessor methods for enum variants | rust, data-structures, crate |
| [[expert-rust-design]] | What separates A+ Rust from average: compile-time invariants, API design, crate architecture, knowing when to stop | rust, architecture, design-patterns |
| [[ecs-pattern]] | Entity Component System — SoA at scale; archetype vs sparse-set storage for game engines | architecture, performance, data-structures, design-patterns |
| [[elimination-backoff-stack]] | Hendler-Shavit-Yerushalmi 2004 stack; matches push/pop in randomized array, bypasses head bottleneck | concurrency, performance, data-structures |
| [[enum-dispatch]] | Static enum dispatch replacing dyn Trait at up to 10x speedup | rust, performance, design-patterns, crate |
| [[enum-map]] | O(1) array-backed map keyed by enum variants with exhaustive initialization | rust, data-structures, crate |
| [[error-chain]] | Legacy unmaintained error-handling crate; migrate to thiserror/snafu + anyhow/eyre | rust, error-handling, crate |
| [[error-stack]] | Stacked Report with typed contexts and arbitrary attachments for observability-first error handling | rust, error-handling, crate |
| [[eyre]] | Fork of anyhow with swappable report handlers for customizable error presentation | rust, error-handling, crate |
| [[facade-crate-pattern]] | Workspace of specialized crates behind a curated public API; used by Bevy, Tokio, ripgrep | rust, architecture, design-patterns |
| [[faa-vs-cas]] | Why fetch-and-add beats compare-and-swap at contention hotspots; the foundational lock-free queue insight | concurrency, performance, architecture |
| [[failure]] | Deprecated error-handling crate (RustSec EOL); migrate to anyhow/thiserror | rust, error-handling, crate |
| [[false-sharing]] | The 128-byte cache-line rule; modern x86 adjacent-line prefetcher makes 64-byte padding insufficient | performance, concurrency, architecture |
| [[fastest-data-structures]] | Benchmarked survey of fastest implementations across 10 major data structure categories (2020–2026) | performance, data-structures, architecture |
| [[fastest-dynamic-arrays]] | The hierarchy: realloc/mremap + good allocator beats every container; allocator dominates language | performance, data-structures, architecture |
| [[fastest-hash-map-2025]] | The 2025 hash map landscape: Boost wins single-threaded, ParlayHash wins concurrent, elastic hashing reopens theory | performance, data-structures, architecture |
| [[fastest-linked-lists]] | The 2026 linked-list landscape: 125× memory-layout lever; plf::list leads single-threaded; "fastest linked list" no longer looks like one | performance, data-structures, architecture |
| [[fastest-ordered-maps]] | The 2026 ordered-map landscape: SIMD B-trees beat std::map by 7–18×; ART-OLC fastest concurrent; sub-log structures stay theoretical | performance, data-structures, architecture |
| [[fb-plus-tree]] | VLDB 2025 SIMD B+-tree for variable-length keys via AVX-512 byte-wise prefix matching | performance, data-structures |
| [[flat-combining]] | Hendler-Incze-Shavit-Tzafrir 2010; combiner thread batches operations; wins where lock-free CAS melts down | concurrency, performance, data-structures |
| [[folly-f14]] | Meta's chunked hash map family; lowest memory footprint at 24.7 B/elt; overflow counting | cpp, performance, data-structures |
| [[folly-fbvector]] | Facebook's optimized vector; jemalloc-aware sizing, xallocx in-place expansion, trivial relocatability | cpp, performance, data-structures |
| [[folly-small-vector]] | Bit-packed SBO vector with single-bit overhead; designed for billions of small vectors at fleet scale | cpp, performance, data-structures |
| [[frozen-dictionary]] | .NET 8's read-optimized immutable dictionary; 43–69% faster string lookups via build-time analysis | performance, data-structures |
| [[fst-finite-state-transducer]] | Immutable sorted string sets/maps sharing prefixes and suffixes; powers Lucene and Tantivy | data-structures, performance, rust |
| [[foldhash]] | Default hasher for hashbrown 0.15+, replacing ahash with better performance and smaller footprint | rust, performance, data-structures, crate |
| [[frunk]] | Haskell/Scala-style generic programming: HList, Coproduct, Generic/LabelledGeneric, Validated | rust, type-theory, data-structures, crate |
| [[fusion-tree]] | Fredman & Willard 1990 word-parallel predecessor structure; O(log n / log log n); no practical implementation competes | data-structures, performance |
| [[gats]] | Generic Associated Types: lending iterators, pointer families, lifetime-parameterized associated types | rust, type-theory, architecture |
| [[generic-array]] | Arrays generic over length via typenum; foundation for RustCrypto ecosystem | rust, type-theory, data-structures, crate |
| [[glommio]] | Datadog's io_uring runtime with triple-ring architecture, proportional-share scheduling, and DMA storage I/O; effectively unmaintained | rust, concurrency, performance, crate |
| [[gpu-queues]] | Warp-centric design mandatory; CAS melts down 1,112× at high thread counts; BACQ is current state of the art | concurrency, performance, architecture |
| [[harris-linked-list]] | Timothy Harris's 2001 marked-pointer lock-free list; Träff-Pöter (2020) and VBL fix the head-retraversal weakness | concurrency, data-structures |
| [[hazard-pointers]] | Bounded-memory safe reclamation primitive; standardized in C++26; complement to EBR for stalled-thread tolerance | concurrency, data-structures, architecture |
| [[growth-factor-analysis]] | 2x vs 1.5x vs adaptive growth; the memory reuse argument and why it's a third-order optimization | performance, data-structures, architecture |
| [[hashbrown]] | Swiss Table port to Rust; std::HashMap since 1.36; 2x over Robin Hood; foldhash default hasher | rust, data-structures, crate |
| [[iceberg-hashing]] | JACM 2023 hash table optimizing space, cache, and stability; built for persistent memory | performance, data-structures, architecture |
| [[inter-thread-communication]] | The 50–135 ns floor; topology > algorithm; full latency ladder from cache-line transfer to scheduler wake | concurrency, performance, architecture |
| [[intrusive-list]] | Linux kernel list.h pattern; pointers embedded in data structures; 5–29× over std::list via zero allocation | data-structures, performance, cpp, architecture |
| [[heapless]] | Stack-allocated fixed-capacity collections via const generics for embedded and no_std | rust, performance, data-structures, crate |
| [[hot-trie]] | Height Optimized Trie (Binna et al., SIGMOD 2018); varies bits per node by data distribution; beats ART on string keys | data-structures, performance |
| [[huge-pages]] | 2 MB / 1 GB pages eliminate TLB misses; 32× more address space per entry; allocator-level integration | performance, architecture |
| [[iai-callgrind]] | Deterministic instruction-count benchmarking via Valgrind for CI regression detection | rust, performance, crate |
| [[io-uring]] | Linux completion-based async I/O with cancellation safety, ecosystem incompatibility, and container security challenges | rust, performance, concurrency, architecture |
| [[ips4o]] | In-Place Super Scalar Samplesort — dominant parallel sort; 1.5x sequential, 3x parallel competitors | performance, concurrency, data-structures |
| [[itertools]] | Canonical extension trait crate for Iterator with chunks, tuple_windows, interleave, and dozens more | rust, crate |
| [[jemalloc]] | Facebook's allocator with xallocx in-place expansion; powers folly-fbvector; 30% throughput at 36 threads | performance, architecture |
| [[jumprope]] | World's fastest rope implementation (~35–40M edits/sec); skip-list spine + gap-buffer leaves | rust, data-structures, performance, crate |
| [[kanal]] | High-throughput message passing channel leading benchmarks at 8–16M msg/sec | rust, concurrency, crate |
| [[kernel-bypass]] | DPDK pattern: poll-mode drivers, no syscalls; 5 µs UDP vs 9 µs kernel; principles translate to inter-thread | performance, architecture, concurrency |
| [[lazy-lock]] | LazyLock/LazyCell replacing lazy_static and once_cell in std (Rust 1.80) | rust, architecture |
| [[lcrq]] | Linked Concurrent Ring Queue (PPoPP 2013); FAA-on-a-ring; 3× over Michael-Scott; foundation of modern fast MPMC | concurrency, performance, data-structures |
| [[learned-indexes]] | ML-based index structures (PGM-Index, RadixSpline) delivering 2–3x faster lookups than B-trees | performance, data-structures, architecture |
| [[lprq]] | Portable LCRQ (PPoPP 2023); drops CAS2 while matching LCRQ on x86; 1.6× over previous portable designs | concurrency, performance, data-structures |
| [[llvm-smallvector]] | LLVM's SBO vector with type-erased SmallVectorImpl base for template bloat avoidance | cpp, performance, data-structures |
| [[lmax-disruptor]] | Lock-free ring buffer achieving 52 ns/hop (630x over ArrayBlockingQueue); mechanical sympathy gold standard | concurrency, performance, architecture |
| [[masstree]] | Trie-of-B+-trees concurrent ordered map; cache-craftiness readers; best for string keys at high contention | concurrency, performance, data-structures |
| [[mechanical-sympathy]] | Martin Thompson's design philosophy; Java ping-pong 50 ns vs C++ 45 ns proves hardware sets the speed | performance, architecture, concurrency |
| [[memory-ordering]] | x86 TSO vs ARM relaxed; acquire/release free on x86, costs 10–20 ns DMB on ARM; the seq_cst trap | concurrency, performance, architecture |
| [[michael-scott-queue]] | Canonical 1996 lock-free MPMC linked-list queue; 3× slower than LCRQ; still in Java's ConcurrentLinkedQueue | concurrency, data-structures |
| [[miette]] | Diagnostic-first errors with source spans, labels, help text, and rich rendering for compiler-style UX | rust, error-handling, crate |
| [[mimalloc]] | Microsoft's memory allocator delivering up to 5.3x faster allocation than glibc malloc | rust, performance, crate |
| [[modern-rust-features]] | Definitive guide to Rust 2023–2026: edition 2024, async evolution, type system, pattern matching, stdlib | rust, architecture |
| [[monoio]] | ByteDance's io_uring runtime with zero-copy slab-allocated I/O; best for Linux-only network proxies | rust, concurrency, performance, crate |
| [[moodycamel-concurrent-queue]] | C++ MPMC with per-producer sub-queues; ~1.7× Boost.Lockfree at the cost of relaxed cross-producer FIFO | cpp, concurrency, performance |
| [[mpmc-queue]] | The 2013–2025 algorithm progression: LCRQ → SCQ → LPRQ → wCQ → Aggregating Funnels | concurrency, performance, data-structures |
| [[newtype-pattern]] | Zero-cost primitive wrapping with private constructors for parse-don't-validate | rust, type-theory, design-patterns |
| [[num-enum]] | Safe integer-to-enum conversions with alternatives, catch-all, and TryFrom for FFI/protocols | rust, data-structures, crate |
| [[numa-aware-queues]] | Nuddle delegation, SmartPQ; 1.87× over NUMA-oblivious designs on multi-socket systems | concurrency, performance, architecture |
| [[nutype]] | Proc macro for generating validated newtypes with sanitization rules (2.7M+) | rust, type-theory, crate |
| [[papaya]] | Lock-free concurrent HashMap with novel memory reclamation, ideal for async read-heavy caches | rust, concurrency, data-structures, crate |
| [[parlayhash]] | CMU concurrent hash map: 1,130 Mops at 128 threads via epoch-based reclamation; 39× libcuckoo | cpp, concurrency, performance, data-structures |
| [[perfect-hashing]] | Zero-collision hash construction for static key sets; PTHash, RecSplit; unbeatable on read-only data | performance, data-structures, architecture |
| [[persistent-queues]] | Functional queues: Hood-Melville (1981), Okasaki (1995), Kaplan-Tarjan (1999) with O(1) catenation | data-structures, type-theory, performance |
| [[proptest]] | Property-based testing with automatic shrinking counterexamples | rust, crate |
| [[pulp]] | Portable SIMD on stable Rust with runtime CPU feature detection and dispatch | rust, performance, crate |
| [[push-back-unchecked]] | Skipping the size==capacity branch enables vectorization; up to 5.9x speedup at small sizes on Clang | performance, data-structures |
| [[rapidhash]] | Fastest non-cryptographic hash function overall (4.25 ns geometric mean) | rust, performance, data-structures, crate |
| [[realloc-mremap]] | Linux's in-place virtual page remapping for O(1) array growth; the ceiling std::vector cannot reach | performance, architecture, data-structures |
| [[recursion-schemes]] | Catamorphisms and anamorphisms with stack safety and arena-based traversal for AST-heavy code | rust, data-structures, design-patterns, crate |
| [[ring-buffer]] | The foundation; 1.5× over std::queue, 2–10× over linked-list; underlies every fast queue | data-structures, performance, concurrency |
| [[rkyv]] | Zero-copy deserialization framework with ~21 ns access time, uncontested in its niche | rust, performance, data-structures, crate |
| [[rcu]] | Read-Copy-Update — zero read-side overhead concurrency; Linux kernel, liburcu, crossbeam-epoch | concurrency, performance, architecture |
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
| [[persistent-functional-list]] | Singly-linked cons list with structural sharing; Haskell/Clojure backbone; lock-free reads via immutability | data-structures, type-theory, performance |
| [[plf-list]] | Matt Bentley's drop-in std::list replacement; 333% faster insert, 6,500% faster clear via unrolled blocks and spatial-aware reinsertion | cpp, performance, data-structures |
| [[pdqsort]] | Pattern-Defeating Quicksort — fastest sequential unstable sort; Rust's sort_unstable; branchless variant | performance, data-structures |
| [[rust-pattern-matching]] | let-else, let-chains, exclusive ranges, exhaustive patterns — modern pattern matching ergonomics | rust, architecture |
| [[rust-testing-patterns]] | Layered testing (unit, integration, doc, property-based) and CI lint enforcement | rust, architecture, design-patterns |
| [[rust-workspace-patterns]] | Virtual workspaces with dependency/lint inheritance, feature flag best practices, cross-crate error chaining | rust, architecture |
| [[rtrb]] | Wait-free SPSC ring buffer for Rust; ~7 ns/op, 520M+ ops/s on Apple M4; real-time / audio DSP | rust, concurrency, performance, crate |
| [[rustix]] | Safe syscall bindings with optional linux_raw backend bypassing libc | rust, performance, crate |
| [[scc]] | Scalable concurrent HashMap optimized for extreme write contention | rust, concurrency, data-structures, crate |
| [[scq]] | Scalable Circular Queue (DISC 2019); portable LCRQ alternative; half the memory; wins on PowerPC | concurrency, performance, data-structures |
| [[slab-list]] | GPU warp-cooperative linked list (Ashkiani et al., IPDPS 2018); 512M updates/s, 937M searches/s on Tesla K40c | concurrency, performance, data-structures, architecture |
| [[sealed-traits]] | Closed type sets via private supertrait; essential for typestate and domain modeling | rust, type-theory, design-patterns |
| [[serde-architecture]] | Serde's zero-allocation data model via traits; dual extensibility without intermediate representations | rust, architecture, design-patterns |
| [[simd-programming]] | SIMD as the cross-cutting technique separating fastest data structures from the rest | performance, architecture |
| [[simdjson]] | SIMD-optimized JSON parser: 4x RapidJSON, 25x nlohmann; gigabytes/second on single core | performance, data-structures |
| [[small-buffer-optimization]] | The general SBO technique; benchmarks reveal it's overrated for typical workloads | performance, data-structures, architecture |
| [[soa-vs-aos]] | Structure of Arrays vs Array of Structures — 1.4–12.7x speedup for field-subset access patterns | performance, architecture, data-structures |
| [[succinct-data-structures]] | Near-information-theoretic-minimum space with efficient queries; SDSL-lite implements 40 publications | performance, data-structures |
| [[swiss-table]] | SIMD-parallel metadata probing hash map design; adopted by C++, Rust, and Go standard libraries | performance, data-structures, architecture |
| [[synchronization-primitives]] | The latency ladder: atomic spin (35 ns) → futex fast path (25 ns) → syscall (5 µs) → context switch (10 µs) | concurrency, performance, architecture |
| [[smallvec]] | Stack-inline Vec up to N elements with heap fallback; benchmarks show Vec+reserve often beats it by 5.5x | rust, performance, data-structures, crate |
| [[seastar]] | ScyllaDB's C++ shard-per-core framework; ancestor of Glommio and the thread-per-core pattern | cpp, concurrency, performance, architecture |
| [[shard-per-core-runtimes-compared]] | Detailed comparison of Monoio, Compio, and Glommio with architecture table and decision framework | rust, concurrency, performance, architecture |
| [[slotmap]] | Generational-index arenas preventing ABA problems with stable handles | rust, data-structures, crate |
| [[snafu]] | Error-handling crate using context selectors for structured semantic backtraces, with progressive design from Whatever to opaque types | rust, error-handling, crate |
| [[snafu-tutorial]] | Deep-dive tutorial on SNAFU: context selectors, backtrace patterns, async integration, boundary conversions, and migration from thiserror | rust, error-handling |
| [[socket2]] | Ergonomic wrapper for advanced socket configuration (SO_REUSEPORT, TCP_CORK, BPF) | rust, performance, crate |
| [[space-optimal-arrays]] | Brodnik 1999 √N lower bound, Tarjan-Zwick 2023, HAT, Tiered Vectors; theory loses to prefetchers | performance, data-structures, architecture |
| [[spsc-queue]] | Single-producer single-consumer privilege; 300–530M ops/s, no atomic RMW required | concurrency, performance, data-structures |
| [[static-vector]] | Fixed-capacity vector with zero heap and no SBO branch; boost::container, P0843 std::static_vector | cpp, performance, data-structures |
| [[strum]] | Standard enum ergonomics crate (100M+): Display, FromStr, EnumIter, EnumCount, metadata, discriminants | rust, data-structures, crate |
| [[tcmalloc]] | Google's allocator; 50x throughput on 4MB allocations; per-CPU caches via rseq | performance, architecture |
| [[the-fastest-queue]] | Survey of FIFO queue performance: SPSC ring buffer dominates 1P/1C, LCRQ+Funnels leads strict-FIFO MPMC | performance, concurrency, data-structures, architecture |
| [[thiserror]] | Ecosystem-standard derive macro (~858M downloads); invisible in public API, no_std since 2.0 via core::error::Error | rust, error-handling, crate |
| [[thread-per-core]] | Server architecture eliminating cache coherency costs; 2–3x Tokio at 16+ cores but 3x overprovisioning penalty | rust, performance, concurrency, architecture |
| [[tikv-jemallocator]] | jemalloc wrapper with most stable allocation latency and built-in heap profiling | rust, performance, crate |
| [[tokio]] | Rust's dominant async runtime with JoinSet structured concurrency; ecosystem moat makes it the correct default | rust, concurrency, crate, performance |
| [[tracing]] | Ecosystem standard for structured diagnostics with ~1 ns disabled overhead | rust, crate, architecture |
| [[tracing-error]] | Enriches errors with tracing span context; SpanTrace beats backtraces in async code | rust, error-handling, concurrency, crate |
| [[trait-upcasting]] | Coerce &dyn Sub to &dyn Super (Rust 1.86); works with Box, Arc, Rc | rust, type-theory |
| [[trivial-relocatability]] | P1144 proposal; memcpy-based vector growth instead of move construction; 2-10x reallocation speedup | cpp, performance, type-theory, data-structures |
| [[type-driven-development]] | Methodology: types as specifications, seven techniques from newtypes to typestates, pragmatic limits | rust, type-theory, design-patterns, architecture |
| [[typed-builder]] | Derive macro for builders with compile-time required field checking | rust, type-theory, crate |
| [[typestate-pattern]] | Crown jewel of type-driven Rust: state machines checked at compile time via move semantics | rust, type-theory, design-patterns |
| [[typenum]] | Type-level numbers with compile-time arithmetic; foundation for generic-array and RustCrypto | rust, type-theory, crate |
| [[unrolled-linked-list]] | Multiple elements per node sized to cache lines; 2–4× traversal speedup; concurrent variant 300% over alternatives | data-structures, performance |
| [[van-emde-boas-tree]] | O(log log U) predecessor structure; theoretically sub-logarithmic but loses to tuned B-trees on real hardware | data-structures, performance |
| [[vector-backed-list]] | Index-based linked list inside a std::vector; 7.8× std::forward_list with compaction; jsl::vector_list, orx-linked-list, GlueList | data-structures, performance |
| [[wcq]] | Wait-free MPMC queue (SPAA 2022); fast-path-slow-path matches SCQ throughput while bounding completion time | concurrency, performance, data-structures |
| [[zerocopy]] | Google's formally verified zero-cost byte conversions, superior to bytemuck for new code | rust, performance, crate |
