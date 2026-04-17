---
tags:
  - rust
  - error-handling
  - concurrency
  - crate
sources:
  - "Raw/Rust/Comparative Deep Research on Rust Error-Handling Crates, Including Snafu.md"
  - "Raw/Rust/SNAFU Deep Research Tutorial - Structured, Context-Rich Errors in Rust.md"
last_updated: 2026-04-16
---

# tracing-error

tracing-error enriches Rust errors with [[tracing]] span context, providing `SpanTrace` and `TracedError` types that capture the active span hierarchy at the point an error is created or propagated. In async systems, this is often more useful than backtraces, which show executor frames rather than logical application context.

## Why SpanTrace beats backtraces in async

In a [[tokio]]-based async service, a backtrace at the error site shows the executor's poll loop, task scheduler internals, and runtime plumbing — rarely what you want for debugging. A `SpanTrace` instead shows the logical hierarchy of instrumented spans (e.g., "handling request > parsing body > validating field X"), which directly maps to what the application was doing.

The `.in_current_span()` extension method makes it ergonomic to attach span context to futures:

```rust
use tracing_error::InstrumentResult;

async fn process() -> Result<(), MyError> {
    do_work().await.in_current_span()?;
    Ok(())
}
```

## Integration

tracing-error is designed to interop with `dyn Error` and [[tracing]] subscribers. It pairs naturally with [[color-eyre]] (which captures SpanTraces by default) and [[error-stack]] (which can include span context in its frame stack). The crate is explicitly marked as "experimental" by the tracing project.

### Integration with SNAFU

Because Rust's orphan rules prevent implementing [[snafu]]'s `GenerateImplicitData` for `SpanTrace` directly (both are external types), the documented pattern is a local newtype with `#[snafu(implicit)]`:

```rust
use snafu::prelude::*;
use tracing_error::SpanTrace;

#[derive(Debug)]
struct MySpanTrace(SpanTrace);

impl snafu::GenerateImplicitData for MySpanTrace {
    fn generate() -> Self { MySpanTrace(SpanTrace::capture()) }
}

#[derive(Debug, Snafu)]
pub struct Error {
    message: String,
    #[snafu(implicit)]
    span: MySpanTrace,
}
```

This captures the current span hierarchy automatically at error construction time — no manual plumbing needed. See [[snafu-tutorial]] for the full async integration story.

## When to use

Use tracing-error in **instrumented async services** where you already have [[tracing]] spans set up. If you're not using tracing, the crate provides no value. If you are, it's a low-cost way to make error reports dramatically more useful. See [[rust-error-crate-comparison]] for where it fits in the ecosystem layers.
