---
tags:
  - performance
  - architecture
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# realloc and mremap

`realloc` is the C standard library's resize-or-relocate primitive; on Linux, glibc's implementation transparently invokes the `mremap` syscall for large buffers. `mremap` remaps virtual pages without copying physical memory — the page tables are updated, the physical frames stay where they are. The result is **O(1) reallocation regardless of buffer size**, which is the single most important advantage of C-style dynamic arrays over `std::vector`.

## What mremap actually does

A standard reallocation is: allocate fresh buffer, copy N bytes, free original. Cost is O(N) and bounded by memory bandwidth. `mremap` instead asks the kernel to remap the virtual address range — keeping the same physical pages, just pointing them at new virtual addresses (which may also be the same, if the kernel can extend the existing mapping). The cost is proportional to the page table size, not the data size, and physical memory traffic is zero.

On a benchmark reallocating a buffer of 256 million floats, `mremap` completely avoided copies from 0 bytes all the way through 256 MB. Against an O(N) bandwidth-bound copy, this is unboundedly faster as buffers grow.

## Why std::vector cannot use it

C++'s `new`/`delete` allocation model has no `realloc` analog. There is no operator-`renew`. The standard `std::allocator` interface offers `allocate` and `deallocate` and nothing else; an allocator cannot expose the "try to extend the existing region" capability that `realloc` provides. This is one of the rare places where C is genuinely faster than C++ at its strongest game, and it is why [[folly-fbvector]] and similar containers go around the standard interface — fbvector calls [[jemalloc|jemalloc's `xallocx()`]] directly, which serves the same purpose at the user-space allocator level.

For Rust, `Vec<T>` benefits from the underlying allocator if it implements `realloc` semantics: the `Allocator` trait has a `grow` method (currently unstable but used internally) that lets allocators perform in-place expansion when possible. Rust's standard `Vec` therefore inherits much of the `realloc`/`mremap` benefit when paired with [[mimalloc]] or [[jemalloc]].

## Practical implications

The "realloc + good allocator" pairing is the absolute performance ceiling for dynamic arrays of trivially copyable types. For C code, this is `realloc(p, new_size)` against a [[mimalloc]] or [[jemalloc]] global allocator — no library required. For C++ that needs `std::vector` API, [[folly-fbvector]] with jemalloc captures most of the win. For Rust, `Vec<T>` + `#[global_allocator]` capture it implicitly. See [[fastest-dynamic-arrays]] for the full hierarchy.

The technique only applies to types that can be relocated by raw byte copy. For non-trivially-relocatable types, the kernel's page remap is irrelevant because the C++ object model demands move constructors run. See [[trivial-relocatability]] for the standardization effort to expose which types qualify.
