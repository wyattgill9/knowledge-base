# Log

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
