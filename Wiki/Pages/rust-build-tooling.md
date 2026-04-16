---
tags:
  - rust
  - performance
  - architecture
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# Rust Build Tooling

An overview of the cargo ecosystem tools that accelerate compilation, enforce quality, and secure the supply chain.

## Compile-time acceleration

1. **Fast linker** — mold (30–70% faster link times) or lld (default on x86_64 Linux since Rust 1.90). See [[cargo-profile-optimization]].
2. **sccache** — compilation caching, especially valuable in CI with GitHub Actions integration.
3. **`[profile.dev.package."*"] opt-level = 2`** — avoid recompiling optimized dependencies during development.
4. **[[cargo-hakari]]** — manages a workspace-hack crate that unifies feature flags, preventing duplicate dependency compilations for up to **1.7x cumulative speedup** in large workspaces.
5. **Parallel frontend** (nightly) — `-Z threads=8` can halve compile times.

## Linting and auditing

- **Clippy config in `Cargo.toml`** (Rust 1.74+) — enable `pedantic` and `perf` groups as warnings, selectively allow noisy lints (`module_name_repetitions`, `must_use_candidate`).
- **[[cargo-deny]]** — audits dependencies for license violations, known vulnerabilities, duplicate versions, and banned crates.
- **cargo-audit** — scans `Cargo.lock` against the RustSec advisory database.
- **cargo-vet** (Mozilla) — formal audit trails for supply-chain security.

## Testing and fuzzing

- **[[cargo-nextest]]** — parallel test runner, the standard replacement for `cargo test`.
- **cargo-fuzz** (libFuzzer) — crash-finding for parsers, deserializers, untrusted input processing.
- **[[bolero]]** — unifies fuzzing and property testing under a single API.
- **[[proptest]]** — property-based testing with shrinking counterexamples.
- **rstest** — fixtures and parameterized tests.

## Utility subcommands

- **cargo-udeps** — find unused dependencies.
- **cargo-bloat** — identify binary size contributors.
- **cargo-expand** — inspect macro expansion.
- **cargo-outdated** — track dependency freshness.
