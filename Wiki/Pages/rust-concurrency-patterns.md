---
tags:
  - rust
  - concurrency
  - architecture
sources:
  - "Raw/Rust/What A+ Rust design actually looks like.md"
last_updated: 2026-04-16
---

# Rust Concurrency Patterns

`Arc<Mutex<T>>` is the beginner default for shared mutable state, but experts reach for it **last**. The decision framework moves from least to most sharing:

## The decision framework

1. **Eliminate sharing entirely** — use ownership transfer. Move data into the task that needs it. This is the fastest and simplest option.
2. **Read-heavy access?** — use `RwLock`. Multiple readers proceed concurrently; only writers block.
3. **Message passing** — use channels ([[kanal]], [[crossbeam-channel]], `tokio::sync::mpsc`). See [[rust-concurrent-data-structures]] for the full channel comparison.
4. **Concurrent map needed?** — use [[dashmap]] (sharded locking), [[papaya]] (lock-free), or [[scc]] (extreme write contention).
5. **Simple counters?** — use `AtomicU64` or other atomics. No lock, no contention.
6. **Last resort** — `Arc<Mutex<T>>` when nothing above fits.

## Data-oriented state splitting

Split state into frequently-mutated and rarely-mutated components to minimize lock scope:

```rust
struct User {
    profile: UserProfile,                 // Immutable after creation — no lock
    stats: Arc<Mutex<UserStats>>,         // Frequently updated — small lock scope
}
```

This is data-oriented design applied to concurrency: separate hot mutable state from cold immutable state so locks protect only what changes.

## Async task patterns

For async code (OneSignal's production patterns):

| Pattern | Use case |
|---------|----------|
| `tokio::join!` | Fixed number of independent futures |
| `FuturesUnordered` | Dynamic batch of futures |
| `tokio::spawn` | Background tasks |
| `Semaphore` | Concurrency limiting |
| `JoinSet` | Task groups with drop-cancellation |

**Cancellation safety** is critical in async Rust: when a future is dropped at an `.await` point, any partial work must be handled correctly. [[tokio]]'s `JoinSet` provides structured concurrency guarantees — no leaked tasks.

## Related pages

See [[rust-concurrent-data-structures]] for the map and channel landscape, [[tokio]] for runtime-specific patterns, and [[thread-per-core]] for the shard-per-core architecture that eliminates shared state entirely.
