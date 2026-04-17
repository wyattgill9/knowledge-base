---
tags:
  - rust
  - architecture
sources:
  - "Raw/Rust/Modern Rust - the definitive 2023–2026 feature and idiom guide.md"
last_updated: 2026-04-16
---

# Rust Workspace Patterns

The dominant pattern for non-trivial Rust projects is a **virtual workspace** with centralized dependency and lint management. Workspace inheritance (stable since 1.64, lint inheritance since 1.74) has become the non-negotiable foundation for any multi-crate project.

## Layout

```
project/
├── Cargo.toml            # workspace root (no [package])
├── rustfmt.toml
├── clippy.toml
├── deny.toml             # cargo-deny config
├── crates/
│   ├── core/
│   ├── api/
│   └── cli/
└── tests/                # workspace-level integration tests
```

## Root Cargo.toml

Centralizes everything: edition, MSRV, license, dependencies, and lints.

```toml
[workspace]
resolver = "3"                         # default for 2024 edition
members = ["crates/*"]

[workspace.package]
edition = "2024"
rust-version = "1.85.0"
license = "MIT OR Apache-2.0"

[workspace.dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
thiserror = "2.0"
anyhow = "1.0"
tracing = "0.1"
my-core = { path = "crates/core" }     # internal cross-references

[workspace.lints.rust]
unsafe_code = "forbid"
missing_docs = "warn"

[workspace.lints.clippy]
all = { level = "warn", priority = -1 }
pedantic = { level = "warn", priority = -1 }
unwrap_used = "warn"
module_name_repetitions = "allow"
```

## Member Cargo.toml

Inherits with a single line per field:

```toml
[package]
name = "my-api"
edition.workspace = true
version.workspace = true
rust-version.workspace = true
license.workspace = true

[dependencies]
tokio.workspace = true
serde.workspace = true
my-core.workspace = true
axum = "0.8"              # crate-specific deps declared normally

[lints]
workspace = true
```

## Feature flag best practices

Features should be **additive** (enabling a feature never removes functionality). Use the `dep:` prefix (stable since 1.60) to avoid leaking dependency names as implicit features:

```toml
[features]
default = ["json"]
json = ["dep:serde_json"]       # dep: prefix hides internal dep name
xml = ["dep:quick-xml"]
full = ["json", "xml"]
```

```rust
#[cfg(feature = "json")]
pub mod json;

#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub struct Config { pub host: String, pub port: u16 }
```

## Cross-workspace error chaining

Use [[thiserror]]'s `#[from]` to compose crate-level errors:

```rust
#[derive(thiserror::Error, Debug)]
pub enum ApiError {
    #[error("database error: {0}")]
    Database(#[from] my_db::DbError),
    #[error("config error: {0}")]
    Config(#[from] my_core::ConfigError),
}
```

See [[modern-rust-features]] for the full language evolution and [[cargo-profile-optimization]] for release profile tuning. For dependency auditing, see [[cargo-deny]]; for build speed, see [[cargo-hakari]].
