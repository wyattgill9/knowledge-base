---
tags:
  - rust
  - architecture
sources:
  - "Raw/Rust/Modern Rust - the definitive 2023–2026 feature and idiom guide.md"
last_updated: 2026-04-16
---

# Rust 2024 Edition

The Rust 2024 edition, stabilized in Rust 1.85 (February 2025), is the largest edition yet. Opt in with `edition = "2024"` in `Cargo.toml`. It changes lifetime capture rules, scoping semantics, unsafe ergonomics, and adds `Future`, `IntoFuture`, and `AsyncFn*` to the prelude. The edition implies **resolver "3"**, which enables MSRV-aware dependency resolution. The `gen` keyword is reserved for future generators.

## RPIT lifetime capture

Return-position `impl Trait` now captures *all* in-scope generics and lifetimes by default. Use `+ use<>` syntax (stable since 1.82) for precise control:

```rust
// Rust 2024: captures both 'a and T by default
fn foo<'a, T: Display>(x: &'a T) -> impl Display { x }

// Opt out of capturing 'b
fn bar<'a, 'b>(x: &'a str, _y: &'b str) -> impl Display + use<'a> { x }
```

This is a breaking change from previous editions where RPIT captured only explicitly mentioned lifetimes. The new default is correct more often, and `use<>` handles the exceptions.

## Unsafe becomes more explicit

Three changes enforce safety discipline:

1. **`unsafe extern` blocks** — extern blocks require the `unsafe` keyword. Items can be marked `safe` explicitly.
2. **`unsafe(...)` attributes** — safety-critical attributes like `#[no_mangle]` require `#[unsafe(no_mangle)]`.
3. **Mandatory `unsafe {}` inside `unsafe fn`** — no more implicit unsafe scope in unsafe functions.

```rust
unsafe extern "C" {
    safe fn strlen(s: *const std::ffi::c_char) -> usize;
    fn free(ptr: *mut std::ffi::c_void); // unsafe by default
}

unsafe fn process(ptr: *mut i32) {
    unsafe { *ptr = 42; } // required — no implicit unsafe scope
}
```

## Tighter temporary scoping

`if let` temporaries and tail-expression temporaries now drop before local variables. This fixes subtle bugs with lock guards:

```rust
fn check(mutex: &std::sync::Mutex<Option<String>>) -> bool {
    if let Some(ref s) = *mutex.lock().unwrap() {
        println!("{s}");
        true
    } else {
        false
    }
    // Guard is already dropped here — safe to acquire again
}
```

In previous editions, the `MutexGuard` temporary could live until the end of the block, causing deadlocks.

## Prelude additions

The 2024 prelude adds `Future`, `IntoFuture`, and the `AsyncFn*` traits (`AsyncFn`, `AsyncFnMut`, `AsyncFnOnce`). See [[rust-async-evolution]] for details on [[rust-async-evolution|async closures]].

## What to do now

Projects should adopt edition 2024 for the improved scoping semantics, async prelude additions, and resolver v3. The migration is mostly mechanical — `cargo fix --edition` handles most changes. The RPIT capture change is the most likely source of breakage. See [[modern-rust-features]] for the full timeline.
