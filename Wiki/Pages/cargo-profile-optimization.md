---
tags:
  - rust
  - performance
  - architecture
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# Cargo Profile Optimization

Cargo's build profiles control optimization levels, LTO, codegen units, and debug info. The defaults are conservative — a tuned `Cargo.toml` can yield **10–20% runtime improvement** with no code changes.

## Recommended release profile

```toml
[profile.release]
opt-level = 3
lto = "fat"           # Whole-program LTO: 10-20% runtime improvement
codegen-units = 1     # Single codegen unit: better inlining/DCE
panic = "abort"       # No unwinding tables: smaller binary, slight speedup
strip = "symbols"     # Remove symbols: smaller binary
overflow-checks = false

[profile.dev]
opt-level = 1                    # Dramatically faster runtime than 0
debug = "line-tables-only"       # 20-40% faster dev compilation

[profile.dev.package."*"]
opt-level = 2                    # Optimize dependencies in dev mode

[profile.profiling]
inherits = "release"
debug = "line-tables-only"       # Keep line info for profilers
strip = false
```

## Fast linker

As of Rust 1.90, **lld became the default linker on x86_64 Linux**, but **mold** is still 30–70% faster for link times:

```toml
# .cargo/config.toml
[target.x86_64-unknown-linux-gnu]
linker = "clang"
rustflags = ["-C", "link-arg=-fuse-ld=mold", "-C", "target-cpu=native"]
```

`target-cpu=native` enables all SIMD extensions available on the build machine — significant for code using [[pulp]] or manual SIMD.

## Profile-Guided Optimization

**PGO via `cargo-pgo`** gives an additional **10%+ runtime improvement**: build an instrumented binary, run it under a representative workload, then recompile with the gathered profiles. This is the single highest-impact optimization most projects never apply. It's especially effective when combined with fat LTO and `codegen-units = 1`.
