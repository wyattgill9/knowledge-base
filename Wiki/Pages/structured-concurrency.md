---
tags:
  - concurrency
  - architecture
  - design-patterns
sources:
  - "Raw/C++/Modern C++ Design Patterns (C++23 and Beyond).md"
last_updated: 2026-05-11
---

# Structured Concurrency

Structured concurrency is the discipline of guaranteeing that every spawned concurrent task has a bounded, lexically visible lifetime — that a function which spawns child tasks does not return until all of them have completed or been explicitly cancelled. Nathaniel J. Smith named the principle in 2018 ("Notes on structured concurrency, or: Go statement considered harmful"), but the underlying idea — that concurrency should compose the way function calls do, with clear parent-child hierarchies rather than detached "fire-and-forget" leaks — is older. By 2026 the discipline is the consensus design across Kotlin coroutines (`CoroutineScope`), Swift Concurrency (`Task` groups + `async let`), Trio and AnyIO in Python, Rust ([[tokio]] `JoinSet`, [[rust-async-evolution|async closures]]), and C++26 [[senders-receivers]]. The pattern is significant because it eliminates a category of bugs — detached tasks, cancellation leaks, lifetime-error race conditions — that had plagued every prior generation of concurrent programming model.

## The structural rule

A function that creates concurrent work must:

1. **Own** the work — the function holds a handle that conceptually contains the spawned tasks.
2. **Not return until** every child has either completed or been cancelled.
3. **Propagate cancellation downward** — when the parent is cancelled, every child receives the cancellation.
4. **Propagate errors upward** — if any child fails, the parent (after winding down siblings) propagates the failure.

This sounds like a small discipline; in practice it eliminates whole categories of failure mode. There are no "leaked tasks" because every task has a parent that waits for it. There is no "task survives past its caller" because the caller does not return until the task is done. Resource cleanup composes — `RAII` / `Drop` / `defer` semantics work uniformly because everything is on the stack.

## The "go statement considered harmful" framing

Smith's 2018 essay drew the analogy to Dijkstra's 1968 "Go To Statement Considered Harmful." Dijkstra's argument was that `goto` permitted arbitrary control flow that broke the correspondence between source-code structure and execution structure; structured programming (`if`/`while`/functions) restored that correspondence and made programs analyzable.

Smith argued that `go func(){...}` in Go (and `Thread.start`, `executor.submit`, `tokio::spawn`, `setTimeout` in JavaScript) is the concurrent analog of `goto` — it spawns work that escapes the source-level structure, with no syntactic indication that "this function leaves something running after it returns." Structured concurrency is the analog of structured programming: control transfers that *don't* respect the call hierarchy are forbidden.

This framing was influential — it gave a name to an intuition that was floating around the async-Python and Kotlin communities — and the resulting design discipline propagated rapidly across the language ecosystem over 2018–2025.

## Implementations across languages

**Kotlin coroutines** (`CoroutineScope`): every coroutine launches within a scope, and the scope's `coroutineScope { ... }` block does not return until all children have completed. Exceptions in children propagate to the scope. Cancellation of the scope cancels all children.

**Swift Concurrency** (`async let`, `withTaskGroup`): `async let` binds a child task whose lifetime is the enclosing function. `withTaskGroup` provides a scope that awaits all tasks before returning. The compiler enforces that no task escapes its scope.

**Trio / AnyIO** (Python): `async with trio.open_nursery() as nursery: nursery.start_soon(...)` — the nursery is a scope; tasks started in it are awaited at the `async with` block's exit. This is the design Smith specifically advocated for in the original essay.

**Rust [[tokio]] `JoinSet`**: a JoinSet is a collection of spawned tasks that the parent must drain before returning. This is structured concurrency made opt-in — `tokio::spawn` by itself is unstructured (the spawned task can outlive its parent), but JoinSet (and the newer `scope` proposal) provides the structured discipline.

**C++26 [[senders-receivers|`std::execution`]]**: every operation has an operation state, and the state's lifetime is bounded by the receiver that consumes it. `when_all` and similar adapters provide structured-concurrency primitives; the design is structured by default.

## The cancellation property

The single most under-discussed property of structured concurrency is **how cancellation propagates**. In an unstructured model, cancelling a task is best-effort — the task may keep running until it notices, or may never notice at all. In a structured model, the parent-child relationship gives cancellation a hierarchical structure:

- Cancelling a parent immediately marks all children for cancellation.
- Children check cancellation at every `await` / suspension point.
- The parent's cleanup runs after all children have noticed and exited.

This is what makes structured concurrency *composable*. A function that takes a structured-concurrency scope can safely launch child work; if the caller cancels the function, everything underneath cancels in order. Resources release in reverse order of acquisition. Try-finally analogs work. This is the property that made the design propagate so quickly — it solves problems that the unstructured model could only solve with discipline and convention.

## The Rust async parallel

Tokio's `JoinSet` (added in 1.21, expanded through 2023–2025) is Rust's flagship structured-concurrency primitive. The pattern:

```rust
let mut set = JoinSet::new();
for url in urls {
    set.spawn(async move { fetch(url).await });
}
while let Some(result) = set.join_next().await {
    process(result);
}
// set is dropped here; any tasks still running are aborted
```

The `JoinSet` drop guarantees that no task outlives the function. See [[rust-async-evolution]] for more on the async evolution path and how Rust is migrating from unstructured `tokio::spawn` to structured `JoinSet`/`scope` patterns. [[tokio]] itself describes JoinSet as the recommended default for new code.

## Position in modern concurrency

In [[modern-cpp-design-patterns]] structured concurrency is the principle that [[senders-receivers]] implement at the language-library level. Every property listed in the senders/receivers page — bounded lifetimes, cancellation propagation, scheduler-agnosticism, composability — is a direct expression of structured-concurrency discipline. C++ is somewhat unusual in arriving at structured concurrency *concurrently* with the rest of the field; the language and the design pattern reached maturity in roughly the same year (2026).

The deeper convergence: structured concurrency is the concurrent analog of [[type-driven-development|types-as-specifications]] — both make program structure visible to the compiler and the reader. Both eliminate categories of bugs by restricting what programs can express, in exchange for greater compositionality of what they can express. Both are products of the same shift in software engineering values over 2015–2025 toward correctness-by-construction rather than correctness-by-testing.
