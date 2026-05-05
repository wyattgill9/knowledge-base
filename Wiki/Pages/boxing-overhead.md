---
tags:
  - performance
  - architecture
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# Boxing Overhead

Boxing — wrapping a primitive value in a heap-allocated object — is the dominant performance penalty for dynamic arrays in Java and Python and the single largest reason their containers benchmark 40–100× slower than C++/Rust. The penalty has nothing to do with container design: every primitive in `Java ArrayList<Integer>` consumes ~28–36 bytes versus 4 bytes in `std::vector<int>`, and the cost is structural to the language's generic system. C# avoids it through reified generics; Java is locked into it by erasure.

## The Java cost structure

A `java.lang.Integer` object on a 64-bit JVM consumes:

- 12–16 bytes of object header (mark word + class pointer)
- 4 bytes of int payload
- 4 bytes of padding to 16-byte alignment

Total: 24 bytes per `Integer`. An `ArrayList<Integer>` adds an 8-byte pointer per slot, bringing per-element cost to **28–36 bytes** depending on JVM and alignment. Compared to 4 bytes for `int` in `std::vector<int>`, that is a 7–9× memory tax — and the corresponding cache-line, prefetch, and branch-prediction penalties make the runtime cost much worse than the memory cost suggests.

The benchmark numbers tell the story: Java `ArrayList<Long>` hits ~190 ns per push_back versus ~2 ns for C `libdynamic`. That is a ~95× ratio. The Trove primitive collection `TLongArrayList` reduces this to ~14.6 ns by storing `long` directly in a `long[]`, but it requires a separate type per primitive (TLongArrayList, TIntArrayList, TDoubleArrayList) and is a third-party library.

## Why Java cannot fix this without changing the language

Java generics use **type erasure**: `ArrayList<Integer>` and `ArrayList<String>` compile to the same runtime class operating on `Object` references. The compiler inserts boxing/unboxing automatically. There is no way to specialize the generic on a primitive without generating a separate class per primitive type — which is exactly what Project Valhalla aims to add but has not yet shipped.

The pre-Valhalla state is that `List<int>` simply does not exist in Java. Any "list of integers" is a list of pointers to Integer objects. The container can be perfectly optimized and the language tax remains.

## How C# avoids it

C# generics are **reified**: `List<int>` and `List<string>` are distinct types at runtime, with the JIT generating specialized code for each. A `List<int>` stores `int` values inline in the backing array — zero boxing, identical layout to `int[]`. The performance penalty Java pays does not exist in C#, which is why C# `List<T>` is competitive with C++/Rust for primitive workloads.

This is the cleanest demonstration of the cost of erasure. Same nominal language family, same nominal feature ("generic list of T"), an order of magnitude difference in performance because of the runtime representation choice.

## Python: boxing plus interpreter

Python's situation is worse: every integer in a Python list is a heap-allocated `PyLongObject` (28 bytes for small ints) plus an 8-byte pointer in the list — 36 bytes per element. On top of that, every operation runs through the CPython bytecode interpreter, which adds ~50–100× overhead versus compiled code. Python lists are at the bottom of the dynamic-array hierarchy not because the list implementation is bad but because every primitive is a heap allocation and every op is interpreted.

The mitigations are familiar: NumPy arrays for primitive numeric work (contiguous, unboxed, vectorized), `array.array` for the standard library's homogeneous array (rarely used), or moving the hot loop to C via Cython/Numba/PyO3.

## What this means for benchmarks

When comparing dynamic array performance across languages, separate three layers:

1. **Per-element cost structure.** Java/Python pay 28–36 B per primitive; C++/Rust/C# pay sizeof(T).
2. **Allocator quality.** glibc malloc vs. [[mimalloc]] vs. [[jemalloc]] vs. [[tcmalloc]]. Often dominates language choice.
3. **Container design.** Growth factor, [[realloc-mremap|in-place expansion]], [[trivial-relocatability]], [[small-buffer-optimization|SBO]]. Matters most when (1) and (2) are controlled.

The order is critical: you cannot optimize a Java ArrayList past its boxing tax with any container cleverness. See [[fastest-dynamic-arrays]] for the cross-language picture and where each language's ceiling is set.
