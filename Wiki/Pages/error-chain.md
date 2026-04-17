---
tags:
  - rust
  - error-handling
  - crate
sources:
  - "Raw/Rust/Comparative Deep Research on Rust Error-Handling Crates, Including Snafu.md"
last_updated: 2026-04-16
---

# error-chain

error-chain is a **legacy, unmaintained** Rust error-handling crate. A Rust community announcement states plainly that it is "no longer maintained," and the repository has moved to `rust-lang-deprecated`. Last published version: 0.12.4 (2020).

**Migration paths:**
- Typed errors -> [[thiserror]] or [[snafu]]
- Application-level error boxing -> [[anyhow]] or [[eyre]]

Like [[failure]], the presence of error-chain in a dependency graph is a signal of age. See [[rust-error-crate-comparison]] for the modern landscape.
