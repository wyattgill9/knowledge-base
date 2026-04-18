# Log

## [2026-04-17] ingest | What A+ Rust design actually looks like
- Source: `Raw/Rust/What A+ Rust design actually looks like.md`
- Created (13 pages):
  - [[expert-rust-design]] (source summary — mental model, compile-time techniques, dimensions of expert design)
  - [[rust-api-design]] — accept broad/return specific, de-generification, builder tiers, extension traits
  - [[rust-allocation-patterns]] — borrow → Cow → owned hierarchy, niche optimization, zero-cost caveats
  - [[serde-architecture]] — zero-allocation data model via traits, dual extensibility
  - [[axum-architecture]] — tower::Service, extractors, IntoResponse composability
  - [[rust-concurrency-patterns]] — Arc<Mutex> as last resort, data-oriented splitting, async task patterns
  - [[rust-abstraction-boundaries]] — enums vs generics vs trait objects, macros as last resort
  - [[facade-crate-pattern]] — workspace of specialized crates behind curated public API
  - [[rust-testing-patterns]] — layered testing and CI lint enforcement
  - [[bevy]] (stub) — data-driven game engine, 40+ crate facade
  - [[heapless]] (stub) — stack-allocated fixed-capacity collections for no_std
  - [[itertools]] (stub) — canonical Iterator extension trait crate
  - [[smallvec]] (stub) — stack-inline Vec with heap fallback
- Updated (4 pages):
  - [[newtype-pattern]] — added critical Deref warning for safety newtypes, added source
  - [[typestate-pattern]] — added source reference
  - [[type-driven-development]] — added source, cross-reference to expert-rust-design
  - [[rust-error-handling]] — added panic vs Result guidelines, RisingWave error formatting conventions, added source
- Index: added 13 new entries
- New wikilinks: [[expert-rust-design]], [[rust-api-design]], [[rust-allocation-patterns]], [[serde-architecture]], [[axum-architecture]], [[rust-concurrency-patterns]], [[rust-abstraction-boundaries]], [[facade-crate-pattern]], [[rust-testing-patterns]], [[bevy]], [[heapless]], [[itertools]], [[smallvec]], [[bon]]
- Open questions:
  - How does ripgrep's crate architecture compare to Bevy's in terms of inter-crate communication patterns?
  - What are the compile-time costs of de-generification vs full monomorphization in practice?
  - How do axum's extractors compare to other Rust web frameworks (actix-web, poem) in terms of type safety?

## [2026-04-16] ingest | Type Driven Development in Rust
- Source: `Raw/Rust/Type Driven Development in Rust.md`
- Created (7 pages):
  - [[type-driven-development]] (source summary — seven techniques, payoff, limits, pragmatic heuristic)
  - [[typestate-pattern]] — state machines checked at compile time via move semantics and PhantomData
  - [[newtype-pattern]] — zero-cost primitive wrapping with private constructors
  - [[parse-dont-validate]] — design principle: return refined types from validation
  - [[sealed-traits]] — closed type sets via private supertrait
  - [[nutype]] — validated newtype proc macro (2.7M+)
  - [[typed-builder]] — compile-time required field checking for builders
- Updated (1 page):
  - [[rust-pattern-matching]] — added exhaustive matching as verification tool section, type-driven development cross-references
- Index: added 7 new entries
- New wikilinks: [[type-driven-development]], [[typestate-pattern]], [[newtype-pattern]], [[parse-dont-validate]], [[sealed-traits]], [[nutype]], [[typed-builder]], [[bon]]
- Open questions:
  - How does the typestate crate (based on ACM paper "Retrofitting Typestates into Rust") compare to manual typestate implementation?
  - What's the compile-time cost of typed-builder vs bon for large structs?
  - Will Rust's type system ever get closer to dependent types (pattern types RFC)?

## [2026-04-16] ingest | Modern Rust — the definitive 2023–2026 feature and idiom guide
- Source: `Raw/Rust/Modern Rust - the definitive 2023–2026 feature and idiom guide.md`
- Created (8 pages):
  - [[modern-rust-features]] (source summary with feature timeline)
  - [[rust-2024-edition]] — RPIT capture, explicit unsafe, tighter temporaries, async prelude, resolver 3
  - [[rust-async-evolution]] — async fn in traits, async closures, trait_variant, JoinSet structured concurrency
  - [[gats]] — Generic Associated Types: lending iterators, pointer families
  - [[rust-pattern-matching]] — let-else, let-chains, exclusive ranges, exhaustive patterns
  - [[rust-workspace-patterns]] — virtual workspaces, dependency/lint inheritance, feature flags, cross-crate error chaining
  - [[trait-upcasting]] — &dyn Sub to &dyn Super coercion (1.86)
  - [[lazy-lock]] — LazyLock/LazyCell replacing lazy_static/once_cell
- Updated (4 pages):
  - [[thiserror]] — added 858M downloads, Error in core (1.81), cross-workspace error chaining pattern
  - [[rust-error-handling]] — added Error in core enabler note
  - [[tokio]] — added JoinSet structured concurrency section with code example, async evolution links
  - Index: added 8 new entries, revised thiserror/tokio descriptions
- New unresolved wikilinks: [[let-else]] (referenced from modern-rust-features timeline)
- Open questions:
  - When will TAIT and the never type stabilize (blocked on next-gen trait solver)?
  - What will gen blocks and async iterators look like in the 2027 edition?
  - How does the 2024 edition RPIT capture change affect real-world migration in practice?

## [2026-04-16] ingest | Rust crates that supercharge enums and algebraic data types
- Source: `Raw/Rust/Rust crates that supercharge enums and algebraic data types.md`
- Created (14 pages):
  - [[rust-enum-crates]] (source summary)
  - Detailed crate pages: [[strum]], [[bitflags]], [[enum-dispatch]], [[typenum]], [[frunk]], [[recursion-schemes]]
  - Stub pages: [[enum-map]], [[ambassador]], [[either]], [[num-enum]], [[auto-enums]], [[enum-as-inner]], [[generic-array]]
- Updated (1 page):
  - [[derive-more]] — expanded from error-only stub to full Swiss-army knife coverage: conversions, formatting, operators, utility methods (200M+ downloads)
- Index: added 14 new entries, revised derive-more description
- New unresolved wikilinks: none (all referenced crates either have pages or are mentioned inline)
- Open questions:
  - Will Rust get native anonymous sum types (`A | B | C`), obsoleting anon_enum/typeunion?
  - How do const generics evolution affect typenum/generic-array's relevance?
  - What's the compile-time cost of enum_dispatch vs ambassador in large codebases?

## [2026-04-16] ingest | SNAFU deep research tutorial
- Source: `Raw/Rust/SNAFU Deep Research Tutorial - Structured, Context-Rich Errors in Rust.md`
- Created (1 page):
  - [[snafu-tutorial]] (source summary)
- Updated (6 pages):
  - [[snafu]] — major expansion: selector visibility/privacy, Into::into auto-conversion, Whatever progressive design, backtrace patterns (leaf/hot-path/chained), #[snafu::report], async/futures integration, OptionExt, ensure!, opaque error pattern, boundary conversions, version/compatibility details
  - [[miette]] — added IntoDiagnostic boundary conversion pattern with SNAFU
  - [[tracing-error]] — added SNAFU integration via newtype + GenerateImplicitData + #[snafu(implicit)]
  - [[thiserror]] — added migration path to SNAFU
  - [[anyhow]] — added SNAFU boundary conversion via From/? pattern
  - [[rust-error-crate-comparison]] — strengthened SNAFU coverage with selector mechanics and tutorial link
- Index: added 1 new entry (snafu-tutorial), revised snafu description
- New wikilinks: [[snafu-tutorial]]
- Open questions:
  - How does snafu-virtstack compare to the newtype SpanTrace pattern for lightweight context?
  - What are the compile-time costs of SNAFU's context selector generation vs thiserror in large workspaces?

## [2026-04-16] ingest | Comparative deep research on Rust error-handling crates
- Source: `Raw/Rust/Comparative Deep Research on Rust Error-Handling Crates, Including Snafu.md`
- Created (11 pages):
  - [[rust-error-crate-comparison]] (source summary)
  - Crate pages: [[error-stack]], [[miette]], [[color-eyre]], [[tracing-error]], [[ariadne]], [[codespan-reporting]], [[displaydoc]], [[derive-more]]
  - Legacy pages: [[failure]], [[error-chain]]
- Updated (6 pages):
  - [[rust-error-handling]] — rewritten from two-section overview to four-layer architecture (typed definition, type-erased reporting, diagnostic rendering, observability context); added legacy crates section, decision matrix
  - [[snafu]] — added code example, semantic backtrace concept, 0.9.0 changes, portability details, comparison with error-stack
  - [[thiserror]] — added backtrace nuance (nightly-only), "invisible in public API" design goal, companion crate pairings
  - [[anyhow]] — expanded with code example, no_std details, positioning vs eyre and error-stack
  - [[eyre]] — expanded with handler ecosystem, public API warning, comparison with anyhow
  - [[tracing]] — added tracing-error to ecosystem section
- Index: added 11 new entries, revised descriptions for 6 updated entries
- New wikilinks: [[error-stack]], [[miette]], [[color-eyre]], [[tracing-error]], [[ariadne]], [[codespan-reporting]], [[displaydoc]], [[derive-more]], [[failure]], [[error-chain]], [[rust-error-crate-comparison]]
- Open questions:
  - How does error-stack's code size bloat manifest in practice? Worth benchmarking.
  - What's the migration story from color-eyre now that it's archived? Is miette the successor for CLI UX?
  - How do snafu-virtstack and similar niche crates perform in allocation-constrained environments?

## [2026-04-15] ingest | Shard-per-core Rust runtimes compared
- Source: `Raw/Rust/Shard-per-core Rust runtimes - Monoio, Compio, and Glommio compared.md`
- Created (6 pages):
  - [[shard-per-core-runtimes-compared]] (source summary)
  - [[apache-iggy]], [[seastar]], [[deterministic-simulation-testing]], [[cache-coherency]], [[direct-io]]
- Updated (6 pages):
  - [[monoio]] — expanded with ownership-transfer buffer model, slab allocation, scheduler details, Monolake benchmarks
  - [[compio]] — expanded with pluggable architecture, Apache Iggy validation, boxing trade-off, recommended default status
  - [[glommio]] — expanded with triple-ring architecture, proportional-share scheduling, DMA storage I/O, maintenance concerns
  - [[io-uring]] — added container security blocker, buffer ownership model, decision framework
  - [[thread-per-core]] — added queueing theory penalty (3x overprovisioning), Seastar lineage, cache coherency motivation, failure modes
  - [[tokio]] — added ecosystem moat section, "poor person's thread-per-core" pattern
- Index: added 6 new entries, revised descriptions for 6 updated entries
- Unresolved wikilinks added: [[axum]], [[tonic]], [[hyper]], [[tower]], [[tokio-console]]
- Open questions:
  - Will Rust's async trait evolution (async closures, trait impls) close the ownership-transfer gap?
  - Is there a path to io_uring re-enablement in container runtimes?

## [2026-04-15] ingest | The blazingly fast Rust crate stack for 2025–2026
- Source: `Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md`
- Created (35 pages):
  - [[blazingly-fast-rust-crate-stack]] (source summary)
  - Crate pages: [[tokio]], [[snafu]], [[thiserror]], [[zerocopy]], [[bytemuck]], [[rkyv]], [[rapidhash]], [[foldhash]], [[socket2]], [[rustix]], [[mimalloc]], [[tikv-jemallocator]], [[divan]], [[iai-callgrind]], [[tracing]], [[papaya]], [[dashmap]], [[kanal]], [[bumpalo]], [[slotmap]], [[bytes]], [[pulp]], [[monoio]], [[bitcode]], [[cargo-nextest]], [[anyhow]], [[eyre]], [[glommio]], [[compio]], [[scc]], [[crossbeam-channel]], [[criterion]], [[cargo-hakari]], [[cargo-deny]], [[hashbrown]], [[bolero]], [[proptest]]
  - Concept pages: [[io-uring]], [[thread-per-core]], [[cargo-profile-optimization]], [[rust-build-tooling]], [[rust-error-handling]], [[rust-memory-allocators]], [[rust-concurrent-data-structures]]
- Updated: none (first ingest, wiki was empty)
- Unresolved wikilinks (page creation queue): [[axum]], [[tonic]], [[hyper]], [[tower]], [[tokio-console]], [[fastrace]], [[wide]], [[dhat-rs]], [[cargo-flamegraph]], [[samply]]
- Open questions:
  - How does the recommended stack differ for `no_std` targets?
  - What's the current state of Rust's Project Safe Transmute and its timeline?
  - How do these crate choices change for embedded/WASM targets?
