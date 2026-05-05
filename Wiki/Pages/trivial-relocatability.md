---
tags:
  - cpp
  - performance
  - type-theory
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# Trivial Relocatability

A type is **trivially relocatable** if relocating an instance from one address to another can be performed by a raw byte copy (`memcpy`) followed by treating the source storage as uninitialized — without running a move constructor or destructor. This is strictly stronger than what the C++ standard exposes, and the gap between "trivially relocatable" and `std::is_trivially_copyable` is where most of the standard library lives. Recognizing this delivers 2–10× speedup on container reallocation.

## Why it matters

When a vector grows beyond its capacity, every existing element must be transferred to the new buffer. The C++ standard requires move-constructing each element at the new location and destroying the original — a per-element function call pair, even when the moves are trivial. `std::is_trivially_copyable` lets the compiler optimize this to `memcpy`, but it is too restrictive: it forbids any user-defined constructor, destructor, or assignment.

Yet the *operation* of relocating — equivalent to "move and destroy" — is bitwise-safe for vastly more types. `std::unique_ptr<T>` holds a raw pointer; bitwise copy gives the destination ownership and the source becomes a moved-from null state, which the destructor handles trivially. `std::string` (in most implementations, via SSO with a self-pointer to inline buffer) is bitwise-relocatable as long as the buffer is heap-allocated. Most of the standard library — every container, every smart pointer, most user types — qualifies.

## How folly exposes it

`folly::IsRelocatable<T>` is a trait that types can specialize to opt in. The macro `FOLLY_ASSUME_FBVECTOR_COMPATIBLE(T)` is the convenient form. When [[folly-fbvector]] reallocates, it checks the trait: if true, it uses `memcpy`; otherwise it falls back to standard move-construct-and-destroy. The same machinery powers parts of [[folly-f14]] and Folly's other containers.

The 2–10× speedup applies to the reallocation operation itself, not amortized push_back. For workloads that reallocate frequently — short-lived vectors, vectors of large objects — the wins are immediate.

## P1144 standardization

P1144 ("Object relocation in terms of move plus destroy") is the long-running proposal to add `std::is_trivially_relocatable<T>` to the standard. The mechanics are settled; the holdup is the language committee's appetite for a concept that requires careful integration with the existing object model. Once standardized, every container in the standard library can adopt it without breaking changes, and the gap between `std::vector` and [[folly-fbvector]] closes substantially.

Until then, `std::is_trivially_copyable` is the conservative trait that production code can rely on. For everything else, library authors annotate manually — and library users get whatever the implementer chose.

## In Rust

Rust's ownership model already makes every type trivially relocatable: `Vec<T>` always uses `ptr::copy_nonoverlapping` (effectively `memcpy`) when reallocating, regardless of `T`. Move semantics in Rust are bitwise by definition; there is no analog of C++'s user-defined move constructor that might do work. This is one of the structural reasons Rust's `Vec<T>` is competitive with C-style arrays despite running through a higher-level standard library — the language closes the trivial-relocatability gap automatically. See [[fastest-dynamic-arrays]] for the comparative numbers.
