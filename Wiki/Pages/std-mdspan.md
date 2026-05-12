---
tags:
  - cpp
  - performance
  - data-structures
  - design-patterns
sources:
  - "Raw/C++/Modern C++ Design Patterns (C++23 and Beyond).md"
last_updated: 2026-05-11
---

# `std::mdspan`

`std::mdspan` is C++23's non-owning multidimensional view over contiguous memory — a typed window onto a flat buffer with compile-time-known rank, optional compile-time-known extents, and a pluggable layout policy that decides how `(i, j, k)` maps to a linear offset. It replaces the custom matrix/tensor class that every numerical C++ codebase used to ship, removes the raw pointer arithmetic that made such code error-prone, and does so with zero runtime overhead.

## The pattern it ends

For decades, exposing a 2D view of a flat buffer meant one of three things: a `Matrix<T>` class that owned its storage and was incompatible with everyone else's `Matrix<T>`; a `std::vector<std::vector<T>>` that fragmented memory and broke cache locality; or raw pointer arithmetic with manual `i * cols + j` indexing repeated at every callsite. `std::mdspan` is the standard answer:

```cpp
void matrix_multiply(std::mdspan<const float, std::dextents<size_t, 2>> A,
                     std::mdspan<const float, std::dextents<size_t, 2>> B,
                     std::mdspan<float,       std::dextents<size_t, 2>> C) {
    for (size_t i = 0; i < C.extent(0); ++i)
        for (size_t j = 0; j < C.extent(1); ++j) {
            C[i, j] = 0;
            for (size_t k = 0; k < A.extent(1); ++k)
                C[i, j] += A[i, k] * B[k, j];
        }
}

std::vector<float> buf(rows * cols);
auto matrix = std::mdspan(buf.data(), rows, cols);
```

The function takes views, not containers. The caller passes whatever owns the memory — `std::vector`, `std::array`, a `std::span`, a memory-mapped file, a GPU staging buffer. No template explosion across container types, no copies, no ownership confusion.

The `[i, j]` syntax (the new C++23 multi-argument `operator[]`) is itself worth noting — it ended the convention of `m(i, j)` parenthesis-based indexing that every matrix library used to work around the limitation of `operator[]` to a single argument.

## Layout policies

The real depth of `mdspan` is its third template parameter, the layout policy. Two are standard: `std::layout_right` (row-major, C convention, the default) and `std::layout_left` (column-major, Fortran/BLAS convention). A third — `std::layout_stride` — supports arbitrary strides for slicing.

You can plug in your own layout. A tiled layout for cache blocking, a Morton-order layout for spatial locality across both dimensions, a transposed layout that swaps `i` and `j` without copying — all are user-defined classes that satisfy the `layout_mapping` concept. The algorithm written against `mdspan` doesn't change; the layout policy changes, and the memory access pattern changes with it. This is the same separation of concerns that NumPy `strides` and BLAS leading-dimensions encoded, finally first-class in C++.

## Zero-overhead claim

For statically-known extents (`std::extents<size_t, 1024, 1024>`), the compiler treats the bounds as compile-time constants. Loop bounds vectorize, the index computation folds, and the generated assembly is identical to hand-written pointer arithmetic. For dynamic extents (`std::dextents`) the bounds are runtime values stored in the mdspan handle (typically two `size_t`s for a 2D view), and the access is a multiply-add — the same code a hand-written `Matrix` would emit.

GCC 14, Clang 19, and MSVC 17.10 all ship optimized `mdspan` implementations; benchmark suites (e.g. the Kokkos team's, who designed the proposal) show matching or exceeding hand-rolled equivalents.

## Connection to existing wiki pages

`mdspan` is the latest entry in the long line of zero-cost view types: `std::span` (C++20) for 1D, `std::string_view` (C++17), and the Rust slice (`&[T]`). The pattern across all of them — separate ownership from view — is the same principle that makes [[soa-vs-aos|Structure of Arrays]] cleanly expressible, that underlies [[csr-graph|CSR graph]] views over flat edge arrays, and that lets [[simd-programming|SIMD code]] operate on subranges without copying.

The Kokkos library (the proposal's origin) uses `mdspan` to abstract over host-CPU, GPU, and accelerator memory in a single algorithm. C++26 will add `submdspan` for sliced views — `submdspan(m, std::full_extent, 5)` extracts a column without copying — completing the slicing toolkit.

## Position in the pattern shift

Within [[modern-cpp-design-patterns]] this is the data-layout entry: where [[std-expected]] reshapes error flow and [[deducing-this]] reshapes inheritance, `mdspan` reshapes the boundary between algorithm and memory layout. The same algorithm now works against any layout policy; the layout choice becomes orthogonal, configurable, sometimes runtime-chosen. This is the C++ expression of the [[mechanical-sympathy]] principle that the layout dominates the algorithm.
