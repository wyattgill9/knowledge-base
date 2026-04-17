---
tags:
  - rust
  - architecture
  - design-patterns
sources:
  - "Raw/Rust/What A+ Rust design actually looks like.md"
last_updated: 2026-04-16
---

# Rust Testing Patterns

Expert Rust projects employ layered testing and enforce quality through CI configuration that catches issues before they reach code review.

## Testing layers

- **Unit tests** in `#[cfg(test)]` modules test internal logic with full access to private functions
- **Integration tests** in `tests/` exercise only the public API, catching interface regressions
- **Doc tests** in `///` comments serve as both documentation and regression tests — the Rust API Guidelines mandate using `?` instead of `unwrap()` to show callers the correct error-handling pattern
- **Property-based testing** with [[proptest]] generates random inputs to verify invariants hold universally, catching edge cases hand-written tests miss. [[bolero]] unifies fuzzing and property testing under a single API.

## CI lint enforcement

Production Rust CI pipelines run `cargo clippy -- -D warnings` with pedantic and selected restriction lints. Key restriction lints:

| Lint | Purpose |
|------|---------|
| `clippy::unwrap_used` | Force explicit error handling |
| `clippy::expect_used` | Same, for `expect` calls |
| `clippy::needless_clone` | Catch unnecessary allocations |
| `clippy::ptr_arg` | Catch `&Vec<T>` where `&[T]` suffices |

Testing with both `--all-features` and `--no-default-features` catches feature-flag interaction bugs. `cargo audit` scans for known vulnerabilities. See [[cargo-deny]] for comprehensive dependency auditing.

## Related pages

See [[expert-rust-design]] for the broader design philosophy, [[cargo-nextest]] for parallel test execution, and [[rust-workspace-patterns]] for workspace-level CI configuration.
