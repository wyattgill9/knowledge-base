---
tags:
  - cpp
  - concurrency
  - architecture
  - design-patterns
sources:
  - "Raw/C++/Modern C++ Design Patterns (C++23 and Beyond).md"
last_updated: 2026-05-11
---

# Senders and Receivers (`std::execution`)

`std::execution` — the C++26 standardization of paper P2300 — introduces a composable, structured concurrency model built around two abstractions: a **sender** describes an asynchronous operation that will eventually produce a value (success), an error, or a stopped signal (cancellation); a **receiver** is the continuation that consumes one of those three results. Senders compose with pipeline operators (`|`, `when_all`, `let_value`, `on`) and the whole pipeline is a value that can be passed around, transformed, and run on a chosen scheduler. It is the displacement of the ad-hoc `std::async` / `std::future` / `std::thread` patchwork that C++ has accumulated, replacing it with a single, scheduler-agnostic, cancellation-aware model.

## The pattern

```cpp
namespace ex = std::execution;

auto work = ex::when_all(
    ex::on(thread_pool.get_scheduler(),
        ex::just(url1) | ex::then(fetch) | ex::then(parse)),
    ex::on(thread_pool.get_scheduler(),
        ex::just(url2) | ex::then(fetch) | ex::then(parse))
)
| ex::then([](auto result1, auto result2) {
    return merge(result1, result2);
});

auto [merged] = ex::sync_wait(work).value();
```

Each `|` step is a sender adapter that transforms one sender into another. `then(f)` is `transform`-like — apply `f` to the value. `on(sched, s)` runs `s` on the given scheduler. `when_all(a, b)` runs `a` and `b` concurrently and produces a sender of their joint result. `sync_wait(s)` blocks the current thread until the pipeline completes. The whole construction is just values — no callbacks, no detached threads, no global state.

## Structured properties

The four properties senders enforce, individually or together, are the substance of what "structured concurrency" means in this context:

**Structured lifetimes.** A sender doesn't start running until something demands its result (`sync_wait`, `start_detached`, or composing into another sender that is itself driven). Once started, it cannot outlive its parent operation. The "dangling callback" failure mode of callback-chained futures cannot occur.

**Cancellation propagation.** Every sender carries a `stop_token` from its operation state. Calling `request_stop()` propagates downward — every in-flight `then`, `let_value`, `bulk` checks the token at suspension points and short-circuits to the stopped path. Cooperative cancellation is built into the protocol, not bolted on per operation.

**Scheduler-agnostic algorithms.** `when_all`, `then`, `let_value` are generic — they work with any scheduler. A pipeline written for a CPU thread pool runs unchanged on a GPU scheduler, an io_uring scheduler, a coroutine scheduler. The scheduler is a parameter, not a hard-coded dependency.

**Composable pipelines.** The output of `on(scheduler, sender)` is itself a sender. The output of `when_all(a, b)` is a sender. Senders form a closed algebra — you can build big pipelines from small ones, factor pipelines into reusable functions, and store them as values.

## What it replaces

`std::async` is anachronistic — it conflates "launch in a thread" with "deferred evaluation" and has surprising default policies. `std::future` / `std::promise` work but offer no composition (no `when_all`, no `then`). `std::thread` is the rawest primitive — useful but easy to misuse, with no structured-lifetime guarantees.

The 2010s C++ ecosystem filled the gap with executor libraries: Folly's `SemiFuture`, libunifex (the proposal's reference implementation), HPX, Eric Niebler's earlier P0443 executors work, Asio's executor concept, and the entire `std::experimental::executors_v1` TS that was eventually withdrawn in favor of senders/receivers. P2300 is the consolidation: one model, one vocabulary, in the standard.

## The Rust parallel

The structural overlap with Rust async is significant:

- A `sender<T>` corresponds roughly to a Rust `Future<Output = T>`.
- `on(scheduler, s)` corresponds to `tokio::spawn(s)` or `rayon::spawn(...)`.
- `when_all` corresponds to `futures::join!` or `tokio::join!`.
- `let_value` is similar to `.and_then()` on a future.
- The `stop_token` is the C++ analog of `tokio_util::sync::CancellationToken`.

The differences are substantive: Rust futures are *driven by polling*, while senders are *driven by callback-style continuations* (closer to JavaScript promises in shape, though all the resolution is direct and inline). Rust async has a single runtime ecosystem dominated by [[tokio]] with [[compio]] / [[monoio]] / [[glommio]] as [[shard-per-core-runtimes-compared|shard-per-core alternatives]]; C++ senders are explicitly multi-runtime-friendly from day one because the scheduler is a value parameter. See [[seastar]] for the C++ predecessor that pioneered this style at ScyllaDB before P2300 standardized it.

## Adoption status as of 2026

GCC 14 and Clang 19 ship experimental implementations under `<execution>`; MSVC 17.10 has the API but limited optimization. NVIDIA's CUDA C++ Core Libraries ship `cuda::std::execution` already, using senders to target GPU streams — a deliberate showcase of the scheduler-agnostic design. The Bloomberg, NVIDIA, and Codeplay teams that championed P2300 use it in production. Wider adoption depends on the third-party scheduler ecosystem maturing: io_uring senders, Asio interop, network/HTTP senders, file-I/O senders. The standard library does not yet ship those; libunifex and stdexec (NVIDIA's standards-tracking implementation) do.

## Position in the pattern shift

In [[modern-cpp-design-patterns]] this is the concurrency entry — the analog at the async layer of what [[std-expected]] is at the error-handling layer and [[overloaded-visit-pattern]] is at the dispatch layer. Each one replaces an imperative, side-effect-heavy idiom with a value-flow composition. Senders take this furthest: an entire asynchronous program becomes a value, transparent to inspection, composition, and substitution of its scheduler. The connection to [[structured-concurrency]] as a broader concept is direct — senders are the first standard-library expression of structured concurrency principles in C++.
