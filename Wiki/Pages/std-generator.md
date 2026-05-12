---
tags:
  - cpp
  - design-patterns
  - concurrency
sources:
  - "Raw/C++/Modern C++ Design Patterns (C++23 and Beyond).md"
last_updated: 2026-05-11
---

# `std::generator`

`std::generator<T>` is C++23's standard coroutine-based lazy sequence type — a function body that uses `co_yield` to produce values one at a time, callable as if it were an iterator-returning function. It retires hand-rolled iterator classes, callback-based producers, and the visitor-style state machines that C++ programmers traditionally wrote when an algorithm wanted to stream values without materializing a container.

## The boilerplate it deletes

Before C++23, exposing a lazy sequence required writing an iterator class with `operator++`, `operator*`, `operator==`, and a sentinel — typically 50–100 lines for a non-trivial generator. The Fibonacci example is the canonical demonstration:

```cpp
std::generator<int> fibonacci() {
    int a = 0, b = 1;
    while (true) {
        co_yield a;
        auto next = a + b;
        a = b;
        b = next;
    }
}
```

That is the entire definition. The compiler builds a coroutine state machine, suspends at `co_yield`, and resumes on the next pull. The function is infinite — no precomputation, no upper bound — but it costs nothing until something consumes from it.

## Composition with ranges

`std::generator` models the `std::ranges::input_range` concept, which means it composes with every range adaptor in the standard:

```cpp
auto first_even_fibs = fibonacci()
    | std::views::filter([](int n) { return n % 2 == 0; })
    | std::views::take(10);

for (int n : first_even_fibs) { /* ... */ }
```

The whole pipeline is lazy. `take(10)` stops pulling after the tenth even Fibonacci number; `fibonacci()` is never asked for more. This is the same lazy-iterator algebra that defines idiomatic Rust (`Iterator` adapters), Haskell (`Data.List`), and Python (generator expressions) — C++23 simply joins the club.

## The cost model

A coroutine has frame state: local variables, the resume point, and a small amount of bookkeeping. On the heap by default, though most compilers (Clang 17+, GCC 14+) perform Heap Allocation eLision Optimization (HALO) when the generator's lifetime is locally provable. Once allocated, a `co_yield` is roughly the cost of a function return — save the frame, jump to the resume point of the consumer.

For tight inner loops over small generated sequences, this is slower than a hand-written loop by maybe 1.5–3× because the optimizer cannot inline across the suspend boundary as aggressively as a single function. For most use cases — parsing, traversal, pipeline stages, lazy I/O — the cost is invisible and the readability gain is large. The pattern shift is the same one [[std-expected]] represents for errors: prefer the structured construct unless profiling says otherwise.

## Where it's the wrong tool

**Compute-bound inner loops** that fit in cache. A coroutine's resume/suspend disrupts inlining; a plain function or a manual state machine wins.

**Producer-consumer across threads.** `std::generator` is single-threaded; sending values between threads needs a [[concurrent-queues|concurrent queue]] or, in C++26, the [[senders-receivers]] machinery.

**Truly recursive generators.** A `std::generator` that recursively yields from sub-generators works but is awkward — the proposed `std::generator::elements_of` smooths this, but the syntax remains less natural than Python's `yield from`.

## The broader pattern

`std::generator` is the first standard-library coroutine type, demonstrating the C++ coroutine machinery (which has shipped since C++20) for a real use case. See [[cpp-coroutines]] for the underlying mechanism. The same machinery underlies [[senders-receivers|`std::execution` senders]], which compose asynchronous operations the way generators compose synchronous sequences.

The connection to other patterns in [[modern-cpp-design-patterns]]: like [[std-expected]] for errors and [[std-mdspan]] for memory layout, this is a case of the standard library providing the abstraction that every nontrivial codebase had previously reinvented, badly.
