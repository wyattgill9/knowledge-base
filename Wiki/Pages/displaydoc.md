---
tags:
  - rust
  - error-handling
  - crate
sources:
  - "Raw/Rust/Comparative Deep Research on Rust Error-Handling Crates, Including Snafu.md"
last_updated: 2026-04-16
---

# displaydoc

displaydoc is a Rust derive macro that generates `Display` implementations from doc comments. Instead of writing `#[error("...")]` attributes, you write a doc comment on each variant and displaydoc uses it as the display template. It is `no_std` compatible (implements `core::fmt::Display`).

Commonly used as a helper alongside [[thiserror]] or other error-handling crates when you prefer doc-comment-driven display strings. A pure formatting utility with no runtime overhead. See [[rust-error-crate-comparison]].
