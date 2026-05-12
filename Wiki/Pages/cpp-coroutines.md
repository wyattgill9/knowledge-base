---
tags:
  - cpp
  - concurrency
  - design-patterns
sources:
  - "Raw/C++/Modern C++ Design Patterns (C++23 and Beyond).md"
last_updated: 2026-05-11
---

# C++ Coroutines

C++ coroutines are stackless suspendable functions — a function containing `co_yield`, `co_await`, or `co_return` is compiled by the front-end into a state machine with a heap-allocated (or HALO-elided) frame, a promise object that customizes its behavior, and three keywords that the user writes to suspend, resume, and finalize. They shipped in C++20 as a low-level primitive without any standard library types using them; C++23 closed that gap with [[std-generator|`std::generator`]], and C++26 closes it further with [[senders-receivers|`std::execution` senders]] (which can be authored as coroutines via `task<T>`-style adapters).

## The unusual design choice

C++ coroutines are *library-customizable*. The keywords `co_yield`, `co_await`, `co_return` lower into calls on a user-supplied promise type, which the standard does not provide. To use coroutines productively you either (a) use a standard-library type that ships its own promise — `std::generator` does this — or (b) write your own coroutine type with its promise.

This is unlike Python (one generator protocol), Rust (one `Future` trait), C# (one `Task<T>` model). The C++ committee deliberately ceded the policy decisions to libraries because no single answer fit the spectrum of use cases — generators, futures, fibers, async I/O, GPU offload — each wants different promise semantics. The trade-off is that authoring a custom coroutine type is *legendarily* complex; consuming one is fine.

## The user-facing model

For the common case of using `std::generator`, the surface is small:

```cpp
std::generator<int> count_up(int n) {
    for (int i = 0; i < n; ++i) co_yield i;
}
```

`co_yield` produces a value and suspends the coroutine. The caller iterates via the generator's iterator interface. The compiler builds the state machine; the user writes what looks like sequential code.

For `co_await` (consuming a sender, awaiting a future), the surface is similar — `auto x = co_await some_future;` suspends until `some_future` resolves. The compiler turns the surrounding function into a state machine where each `co_await` is a possible suspend point.

## The frame-allocation question

A coroutine's local state lives in a frame, allocated by `operator new` on the coroutine's promise type by default. For generators and futures whose lifetimes don't escape the calling function, the optimizer performs Heap Allocation eLision Optimization (HALO) — it inlines the frame into the caller's stack and eliminates the allocation. HALO works reliably in Clang ≥ 17 and GCC ≥ 14 for the common cases; for cases where the coroutine is passed across function boundaries or stored in a container, the allocation persists and the cost is one `malloc`-equivalent per coroutine instance.

The custom allocator hook (`operator new` on the promise) lets libraries plug in arena allocators ([[bumpalo]]-equivalent), per-coroutine slab pools, or no-allocation-at-all designs for embedded targets. This is one of the few places where the customization complexity of C++ coroutines pays off — Rust's `Future` has no analog and embedded async-Rust users have had to invent their own conventions.

## The library ecosystem as of 2026

Standard:
- **[[std-generator|`std::generator<T>`]]** — C++23, the pull-based sequence.
- **[[senders-receivers|`std::execution::task<T>`]]** — C++26, when used with stdexec or libunifex's task adapters, exposes a coroutine interface to write senders as `co_await`-able async functions.

Third-party (libraries that ship their own coroutine types, used widely):
- **cppcoro** (Lewis Baker) — the original C++20 coroutine library. Provides `task<T>`, `generator<T>`, `async_generator<T>`, async I/O, sync primitives. Largely superseded by the standard now but remains the canonical reference.
- **folly::coro** — Meta's production coroutine library, used in Facebook's services.
- **libunifex** — the reference implementation of P2300 senders/receivers, including coroutine interop.
- **stdexec** — NVIDIA's standards-tracking implementation of senders/receivers, including CUDA scheduler integration.
- **Asio's coroutine support** (`asio::awaitable<T>`) — for network I/O coroutines.

## Common pitfalls

**Dangling references in coroutine frames.** A coroutine that captures a reference to a local of its caller can outlive the caller. The frame survives; the referenced storage does not. This is one of the most common Bug Bash items in adopting coroutines and has produced multiple proposals for compile-time lifetime checking, none yet shipping.

**Awaiting in destructors.** A destructor cannot `co_await`. This rules out using coroutines for resource cleanup, RAII-with-await, or other patterns that are routine in Rust async (which has the same restriction in `Drop`).

**Coroutine + exception interaction.** Exceptions propagate through `co_await` with subtle rules — exceptions thrown inside a coroutine are stored in the promise and re-thrown at the awaiter, but only at well-defined points. Libraries that use coroutines heavily often disable exceptions in coroutine code paths to simplify reasoning.

**Symmetric transfer.** A subtle but important optimization where one coroutine resumes another directly without returning to a scheduler. Used by `task<T>` chains and sender pipelines. Compilers implement it correctly but it's a frequent source of "why is my coroutine slow" issues — the answer is usually that symmetric transfer isn't kicking in, often because of an intervening shared_ptr or type erasure.

## Connection to the broader pattern shift

In [[modern-cpp-design-patterns]] coroutines are the underlying machinery for [[std-generator]] and [[senders-receivers]]. They are the language-level primitive; the patterns are the user-facing abstractions built on top. The principle is the same as `std::move_only_function` providing the primitive that the value-semantic polymorphism patterns require, or [[cpp26-static-reflection]] providing the primitive that derive-style code-generation patterns require: C++ ships the low-level mechanism, the standard library and ecosystem provide the high-level affordances.
