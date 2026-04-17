---
tags:
  - rust
  - type-theory
sources:
  - "Raw/Rust/Modern Rust - the definitive 2023–2026 feature and idiom guide.md"
last_updated: 2026-04-16
---

# Trait Upcasting

Trait upcasting, stabilized in Rust 1.86 (April 2025), lets you coerce `&dyn Sub` to `&dyn Super` — a long-requested feature that previously required manual workarounds.

```rust
trait Animal: std::fmt::Debug + std::any::Any {
    fn name(&self) -> &str;
}

fn print_and_downcast(animal: &dyn Animal) {
    let debuggable: &dyn std::fmt::Debug = animal;  // upcast to supertrait
    println!("{debuggable:?}");

    let any: &dyn std::any::Any = animal;            // upcast to Any
    if let Some(dog) = any.downcast_ref::<Dog>() {
        println!("It's a dog named {}!", dog.0);
    }
}

// Works with Box, Arc, Rc too
fn upcast_box(animal: Box<dyn Animal>) -> Box<dyn std::fmt::Debug> {
    animal
}
```

This completes a gap in Rust's trait object model that previously required either downcasting hacks or the `as-any` pattern. Combined with [[gats]] and RPITIT, the Rust type system is now nearly feature-complete for practical use. See [[modern-rust-features]] for the timeline.
