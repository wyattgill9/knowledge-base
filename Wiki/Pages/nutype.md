---
tags:
  - rust
  - type-theory
  - crate
sources:
  - "Raw/Rust/Type Driven Development in Rust.md"
last_updated: 2026-04-16
---

# nutype

nutype is a Rust proc macro crate (2.7M+ downloads) that generates validated newtypes with sanitization rules automatically. It eliminates the boilerplate of the [[newtype-pattern]] — private constructors, validation logic, trait impls — by declaring constraints declaratively.

This is [[parse-dont-validate]] made ergonomic: define a newtype with its validation rules, and nutype generates a type where invalid instances cannot exist. For manual newtype creation, [[derive-more]] handles the trait derivation boilerplate. See [[type-driven-development]] for the full methodology.
