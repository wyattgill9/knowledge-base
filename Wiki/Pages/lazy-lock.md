---
tags:
  - rust
  - architecture
  - crate
sources:
  - "Raw/Rust/Modern Rust - the definitive 2023–2026 feature and idiom guide.md"
last_updated: 2026-04-16
---

# LazyLock

`std::sync::LazyLock` and `std::cell::LazyCell`, stabilized in Rust 1.80 (July 2024), replace the `lazy_static` and `once_cell` crates for lazy initialization. They are now part of the standard library — no external dependencies needed.

```rust
use std::sync::LazyLock;
use std::collections::HashMap;

static CONFIG: LazyLock<HashMap<String, String>> = LazyLock::new(|| {
    let mut m = HashMap::new();
    m.insert("host".into(), "localhost".into());
    m.insert("port".into(), "8080".into());
    m
});
```

- **`LazyLock`** — thread-safe (`Sync`), for `static` items and shared state
- **`LazyCell`** — single-threaded, for per-thread or non-shared lazy values

Use `LazyLock` in new code instead of `lazy_static!` or `once_cell::sync::Lazy`. See [[modern-rust-features]].
