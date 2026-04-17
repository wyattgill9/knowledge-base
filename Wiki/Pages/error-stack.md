---
tags:
  - rust
  - error-handling
  - crate
sources:
  - "Raw/Rust/Comparative Deep Research on Rust Error-Handling Crates, Including Snafu.md"
last_updated: 2026-04-16
---

# error-stack

error-stack is a Rust [[rust-error-handling|error-handling]] crate that builds a layered `Report` with a frame stack of typed contexts and arbitrary attachments as errors propagate. Maintained by HASH, it deliberately trades development overhead for debugging and observability payoff — the crate does not allow string-like root errors, pushing you toward typed contexts from the start.

## Core model

Unlike [[anyhow]] or [[eyre]] which erase the error type behind a single dynamic report, error-stack constructs a `Report<C>` containing a stack of `Context` frames and attachments. The `ResultExt` extension trait makes it ergonomic to build this stack:

```rust
use error_stack::{Report, ResultExt};

#[derive(Debug)]
struct ConfigCtx;
impl error_stack::Context for ConfigCtx {}

fn load_config(path: &str) -> Result<String, Report<ConfigCtx>> {
    std::fs::read_to_string(path)
        .map_err(Report::from)
        .attach_printable(format!("path={path}"))
        .change_context(ConfigCtx)
}
```

Attachments can be printable (strings, formatted messages) or opaque (arbitrary user data queryable later). This goes beyond the standard `source()` chain — you can carry structured payloads through the error path and render or inspect them at the boundary.

## When to use error-stack

error-stack shines in **large systems where observability matters more than development speed**. If you find yourself wishing [[anyhow]]'s context strings were queryable or typed, error-stack is the answer. It pairs well with [[tracing-error]] for async span context.

The tradeoff is real: the crate explicitly acknowledges that it "adds some development overhead." Community feedback also notes that depending on configuration, error-stack's backtrace dependency can contribute to code size bloat. For smaller projects or rapid prototyping, [[anyhow]] is faster to ship with.

## Comparison with snafu

Both [[snafu]] and error-stack push toward structured error context, but they differ in approach. Snafu generates context selectors per variant — the structure lives in your error type definitions. error-stack keeps the error type simple and builds structure in the report's frame stack at propagation time. Snafu is more natural for library APIs; error-stack is more natural for application-level error aggregation.

## Portability

error-stack indicates `no_std` capability in crate metadata. Dual-licensed MIT/Apache-2.0.
