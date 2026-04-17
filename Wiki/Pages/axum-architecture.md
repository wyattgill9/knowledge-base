---
tags:
  - rust
  - architecture
  - design-patterns
sources:
  - "Raw/Rust/What A+ Rust design actually looks like.md"
last_updated: 2026-04-16
---

# Axum Architecture

Axum builds entirely on [[tower]]'s `Service` and `Layer` traits rather than inventing its own middleware system. This makes the entire [[tower]] middleware ecosystem — rate limiting, timeouts, authentication — available without adaptation. Axum uses `#![forbid(unsafe_code)]` throughout.

## Extractor-based handlers

Handlers are async functions that accept **extractors** as parameters — types implementing `FromRequest` or `FromRequestParts`. The type system determines what each handler needs, with no routing macros required:

```rust
async fn create_user(
    State(db): State<Database>,        // Extracted from app state
    Json(input): Json<CreateUserInput>, // Extracted from request body
) -> Result<Json<User>, ApiError> {
    // Type signatures alone define the contract
}
```

This is [[type-driven-development]] applied to web frameworks: the function signature *is* the specification. Adding a new extractor parameter automatically causes axum to extract that data from the request.

## IntoResponse composability

Return types implement `IntoResponse`, so tuples compose naturally: `(StatusCode, Json<T>)` works without ceremony. Routes are function compositions, not macro-decorated handlers.

## Architectural significance

Axum demonstrates how trait-driven design eliminates the need for framework-specific abstractions. By building on [[tower]]'s `Service` trait, axum is both a web framework and a thin layer over a generic async service ecosystem. See [[expert-rust-design]] for the broader design principles and [[serde-architecture]] for another example of traits-as-architecture.
