---
tags:
  - rust
  - error-handling
  - architecture
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# Rust Error Handling

The Rust error-handling ecosystem has settled into a clear set of roles by 2026. The choice depends on whether you're writing a library or application, and how much structure you want in your error types.

## Typed errors (libraries and structured applications)

- **[[thiserror]] 2.0** — the ecosystem standard. Derive `Error` with minimal boilerplate (~2 lines per variant). `no_std` support since 2.0. Use this by default.
- **[[snafu]]** — context selectors enforce per-module error granularity. More verbose (~5 lines per variant) but prevents error type degradation in large multi-crate workspaces. Used by GreptimeDB, InfluxDB IOx, Iroh.

Neither has meaningful runtime overhead. The choice is purely API design philosophy.

## Dynamic errors (applications, prototyping, CLIs)

- **[[anyhow]]** — stable, simple `anyhow::Result` boxing. The default for application code where you want to propagate errors without defining types for every failure mode.
- **[[eyre]]** — richer error reports with custom handlers. Better when you want formatted, human-readable error output with context chains.

## Recommendation

For most projects: [[thiserror]] for library boundaries and structured error types, [[anyhow]] for application-level plumbing. Switch to [[snafu]] only if you're running a large workspace where error discipline matters more than familiarity.
