---
tags:
  - rust
  - crate
  - architecture
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# tracing

The tracing crate (tokio-rs) is the Rust ecosystem's standard for structured diagnostics — logging, distributed tracing, and instrumentation unified under one API. When a level is disabled, overhead is a single atomic load (~1 ns).

## Performance tuning

For hot paths, compile-time level gating eliminates all debug/trace instrumentation from the binary entirely:

```toml
[dependencies]
tracing = { version = "0.1", features = ["max_level_info"] }
```

This means zero-cost debug instrumentation: leave `tracing::debug!` calls in production code and they compile to nothing.

## Ecosystem

- **tracing-subscriber** — formatting, filtering, and layer composition for output.
- **[[tracing-error]]** — enriches errors with span context via `SpanTrace`. In async code, SpanTraces show the logical application hierarchy rather than executor frames, making them far more useful than backtraces. Integrates with [[color-eyre]] and [[error-stack]].
- **[[tokio-console]]** — real-time async task debugging powered by tracing data.
- **[[fastrace]]** — alternative for library authors who want zero overhead when tracing is completely disabled (tracing still pays the atomic load even when no subscriber is attached; fastrace compiles out entirely).

## For library authors

If writing a library crate, instrument with tracing spans and events. Application authors choose their subscriber. This separation of instrumentation from output is tracing's key architectural insight and why it won over `log`.
