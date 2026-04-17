---
tags:
  - rust
  - performance
  - architecture
sources:
  - "Raw/Rust/What A+ Rust design actually looks like.md"
last_updated: 2026-04-16
---

# Rust Allocation Patterns

Expert Rust code follows a strict allocation avoidance hierarchy: **borrow → Cow → owned**. The fastest allocation is the one that doesn't happen.

## The hierarchy

### 1. Borrow first

Accept `&str` and `&[T]` in function parameters. No allocation, no copying, maximum flexibility. This is the default.

### 2. Cow for conditional ownership

When a function usually returns input unchanged but sometimes modifies it, `Cow<str>` avoids allocation on the common path:

```rust
fn escape_html(input: &str) -> Cow<str> {
    if !input.contains(['<', '>', '&']) {
        Cow::Borrowed(input)  // Zero allocation — most common path
    } else {
        Cow::Owned(input.replace('<', "&lt;").replace('>', "&gt;"))
    }
}
```

### 3. Owned only when necessary

Allocate `String`, `Vec<T>`, `Box<T>` only when data must outlive the current scope or be mutated independently. Prefer `String::with_capacity()` when the size is known or estimable.

## Buffer reuse

In loops, reuse allocations instead of creating new ones each iteration. `String::with_capacity()` then `clear()` reuses the underlying buffer — the allocation happens once, not per iteration. The same applies to `Vec::clear()`.

## SmallVec and stack allocation

[[smallvec]]`<[T; N]>` stores up to N elements inline on the stack before falling back to heap allocation. Ideal for collections that are usually small but occasionally grow.

## Arena allocation

[[bumpalo]] provides bump allocation (~2 ns per allocation) with instant bulk deallocation. Ideal for phase-oriented work: per-request processing in web servers, AST nodes in compilers, parse trees. See also [[slotmap]] for generational-index arenas with stable handles.

## Niche optimization

The Rust compiler exploits invalid bit patterns to store enum discriminants without additional space:

| Type | Size | Why |
|------|------|-----|
| `Option<Box<T>>` | pointer-sized | null represents `None` |
| `Option<NonZeroU32>` | 4 bytes | zero represents `None` |
| `Option<bool>` | 1 byte | uses unused bit pattern |
| `Option<Option<bool>>` | 1 byte | still fits in unused patterns |
| `Option<char>` | 4 bytes | `0x00110000` (past max Unicode) for `None` |

For large enums, `Box` the largest variant to prevent it from inflating all variants — Clippy's `large_enum_variant` lint catches this automatically.

## Zero-cost abstractions require release mode

Iterator chains compile to assembly identical to hand-written C loops — **but only in release mode**. Debug builds can be 10–50x slower. Verify zero-cost claims with `cargo asm` or Godbolt. Hidden costs to watch for: `dyn Trait` (vtable dispatch), `Box<dyn Fn>` (allocates), array indexing (bounds checks that iterators elide).

## Related pages

See [[expert-rust-design]] for the full design philosophy, [[rust-memory-allocators]] for global allocator choices, and [[bumpalo]] for arena allocation details.
