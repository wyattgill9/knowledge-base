---
tags:
  - rust
  - type-theory
  - architecture
sources:
  - "Raw/Rust/Modern Rust - the definitive 2023–2026 feature and idiom guide.md"
last_updated: 2026-04-16
---

# Generic Associated Types

Generic Associated Types (GATs), stabilized in Rust 1.65 (November 2022), enable lifetime-parameterized and type-parameterized associated types. They unlock two major patterns that were previously impossible: lending iterators and generic pointer families.

## Lending iterators

The canonical GAT use case — an iterator whose items borrow from the iterator itself:

```rust
trait LendingIterator {
    type Item<'a> where Self: 'a;
    fn next(&mut self) -> Option<Self::Item<'_>>;
}

struct WindowsMut<'w, T> {
    slice: &'w mut [T],
    pos: usize,
    size: usize,
}

impl<'w, T> LendingIterator for WindowsMut<'w, T> {
    type Item<'a> = &'a mut [T] where Self: 'a;

    fn next(&mut self) -> Option<Self::Item<'_>> {
        if self.pos + self.size > self.slice.len() { return None; }
        let start = self.pos;
        self.pos += 1;
        Some(&mut self.slice[start..start + self.size])
    }
}
```

Standard `Iterator` can't express this because `Item` has no lifetime parameter — the item can't borrow from `&mut self`. GATs fix this by letting `Item<'a>` vary with the borrow lifetime.

## Generic pointer families

Abstracting over `Rc` vs `Arc` (or any smart pointer) without code duplication:

```rust
trait PointerFamily {
    type Pointer<T>: Deref<Target = T>;
    fn new<T>(value: T) -> Self::Pointer<T>;
}

struct ArcFamily;
impl PointerFamily for ArcFamily {
    type Pointer<T> = Arc<T>;
    fn new<T>(value: T) -> Arc<T> { Arc::new(value) }
}
```

This pattern enables data structures that are generic over their sharing strategy — single-threaded (`Rc`) vs multi-threaded (`Arc`) — with zero runtime cost.

## Relationship to other type-level features

GATs complement RPITIT (return-position `impl Trait` in traits, 1.75) — RPITIT handles the common case of returning opaque types from trait methods without naming them, while GATs are needed when the associated type must be *named* or *parameterized*. Both are part of the type system completion tracked in [[modern-rust-features]]. For type-level computation with numbers and arithmetic, see [[typenum]] and [[frunk]].
