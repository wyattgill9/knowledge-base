---
tags:
  - rust
  - concurrency
  - architecture
sources:
  - "Raw/Rust/Modern Rust - the definitive 2023–2026 feature and idiom guide.md"
last_updated: 2026-04-16
---

# Rust Async Evolution

Two landmark features — **async fn in traits** (1.75) and **async closures** (1.85) — replaced the `async-trait` proc-macro crate's heap allocations with zero-cost compiler-generated futures. Together with structured concurrency patterns, they make async Rust in 2026 dramatically less boilerplatey than in 2023.

## Async fn in traits (Rust 1.75, December 2023)

`async fn` works directly in trait definitions, desugaring to `-> impl Future<Output = T>`:

```rust
trait Storage {
    async fn get(&self, key: &str) -> Option<String>;
    async fn set(&self, key: &str, value: &str) -> Result<(), std::io::Error>;
}

// Static dispatch works perfectly
async fn migrate(from: &impl Storage, to: &impl Storage) {
    if let Some(val) = from.get("key").await {
        to.set("key", &val).await.unwrap();
    }
}
```

**Key limitation:** async trait methods are *not* dyn-compatible. You cannot write `&dyn Storage`. For dynamic dispatch, use the `trait_variant` crate to generate a `Send`-bound version:

```rust
#[trait_variant::make(SendStorage: Send)]
trait Storage {
    async fn get(&self, key: &str) -> Option<String>;
}

// SendStorage requires Future: Send — safe to spawn on tokio
async fn spawn_work(storage: impl SendStorage + 'static) {
    tokio::spawn(async move { storage.get("key").await; });
}
```

## Async closures (Rust 1.85, February 2025)

First-class async closures with three new traits: `AsyncFn`, `AsyncFnMut`, `AsyncFnOnce`. They solve the longstanding "two-generic-parameter" problem where you needed separate `F` and `Fut` type parameters:

```rust
// OLD: two generic params, couldn't express higher-ranked lifetimes
fn old_style<F, Fut>(f: F) where F: Fn(&str) -> Fut, Fut: Future<Output = String> {
    todo!()
}

// NEW: clean and correct
async fn new_style(callback: impl async Fn(&str) -> String) {
    let local = String::from("hello");
    let result = callback(&local).await; // borrows local — works!
    println!("{result}");
}

// Usage
let greet = async |name: &str| -> String {
    tokio::time::sleep(Duration::from_millis(10)).await;
    format!("Hello, {name}!")
};
new_style(greet).await;
```

The `AsyncFn*` traits are added to the 2024 edition prelude. See [[rust-2024-edition]].

## Structured concurrency with JoinSet

Modern [[tokio]] structured concurrency relies on `JoinSet` for dynamic task groups:

```rust
use tokio::task::JoinSet;

async fn fetch_all(urls: Vec<String>) -> Vec<String> {
    let mut set = JoinSet::new();
    for url in urls {
        set.spawn(async move {
            reqwest::get(&url).await.unwrap().text().await.unwrap()
        });
    }
    let mut results = Vec::new();
    while let Some(res) = set.join_next().await {
        results.push(res.unwrap());
    }
    results // JoinSet cancels remaining tasks on drop
}
```

`tokio::join!` handles fixed concurrency, `select!` handles racing. JoinSet's drop-cancellation provides structured concurrency guarantees — no leaked tasks.

## What's still missing

- **Async iterators via `gen` blocks** — `gen` keyword reserved in the [[rust-2024-edition|2024 edition]]
- **Pin ergonomics** — making self-referential futures less painful
- **TAIT** (Type Alias Impl Trait) — would allow naming async trait return types

See [[modern-rust-features]] for the full timeline.
