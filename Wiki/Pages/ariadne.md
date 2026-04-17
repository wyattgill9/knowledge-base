---
tags:
  - rust
  - error-handling
  - crate
sources:
  - "Raw/Rust/Comparative Deep Research on Rust Error-Handling Crates, Including Snafu.md"
last_updated: 2026-04-16
---

# ariadne

ariadne is a Rust crate for rendering pretty, source-spanned diagnostic reports — the kind of output you see from compilers and linters pointing at specific lines and columns in source code. It serves the same niche as [[miette]] and [[codespan-reporting]] but with a focus on fine-grained label control and visual polish.

Use ariadne when building parsers, compilers, or any tool where errors need to reference specific source locations. It complements typed error crates like [[thiserror]] or [[snafu]] — those define *what* went wrong, ariadne shows *where*. See [[rust-error-crate-comparison]] for the full diagnostic rendering landscape.
