---
tags:
  - design-patterns
  - architecture
  - type-theory
sources:
  - "Raw/C++/Modern C++ Design Patterns (C++23 and Beyond).md"
last_updated: 2026-05-11
---

# Design by Contract

Design by Contract (DbC) is Bertrand Meyer's methodology, developed in the Eiffel language in the 1980s, of attaching machine-checkable **preconditions** (what a function requires of its caller), **postconditions** (what the function guarantees on return), and **class invariants** (what is always true of a well-formed object) to every public interface. The contracts are first-class language constructs, not comments or assertions, and they participate in inheritance — derived classes can weaken preconditions and strengthen postconditions (the language-level expression of Liskov Substitution). DbC is now arriving in mainstream languages thirty-five years after Eiffel pioneered it: C++ via [[cpp26-contracts]], Ada has had it since 2012, D since inception, and Rust is debating a contracts RFC.

## The core triple

A contracted function has three pieces, in addition to its signature:

- **Precondition** — the caller's obligation. `pre(size > 0)` says "the caller must ensure `size > 0` before calling this function."
- **Postcondition** — the function's promise. `post(r: r >= 0)` says "the function guarantees the return value will be non-negative."
- **Class invariant** — a property that is true before and after every public method call on an object. Eiffel writes `invariant balance >= 0;`; C++26's contracts (in P2900) deliberately omit class invariants for the initial proposal, deferring them to a later revision.

Together they form the function's *contract*: under the precondition, the function delivers the postcondition. Caller and callee each have responsibilities; correctness is the conjunction.

## The Liskov connection

The most theoretically interesting property of DbC is its formalization of subtyping. In Meyer's framework:

- A derived class may **weaken** preconditions (accept more inputs than the base required).
- A derived class may **strengthen** postconditions (promise more than the base did).
- A derived class must **preserve or strengthen** invariants.

These rules are exactly Barbara Liskov's substitution principle made operational. A `Square` cannot be a subclass of `Rectangle` (the classic counter-example) because `Square::set_width(w)` strengthens the postcondition `width == w` to also enforce `height == w`, which is not what `Rectangle::set_width`'s contract promised — and that strengthening of the postcondition implicitly strengthens what the *caller* can rely on, breaking substitutability.

C++26 contracts inherit this lineage but do not enforce the LSP rules at the language level (that requires the contracts-in-inheritance extension, deferred). For now the discipline is the developer's; the contracts express the intent without enforcing the substitution property.

## What DbC is good at

**Encoding informal preconditions formally.** Every function in every codebase has implicit preconditions ("don't pass null," "must call init() first," "size must be even"). DbC makes them part of the type.

**Catching contract violations early.** A violated precondition is a programmer error — the caller violated its part of the agreement. DbC makes this an explicit, debuggable event rather than a silent UB or eventual crash.

**Driving design discussions.** Writing the contracts before the body forces you to articulate what the function actually requires and promises. This is the methodological core of Meyer's book *Object-Oriented Software Construction*.

**Optimizer hints.** A precondition is a promise from the caller to the callee; the optimizer can use it as a free assumption when generating code for the callee. `pre(p != nullptr)` lets the compiler skip null checks throughout the body even in release builds.

## What it's not

**Proof.** A contract is a runtime check (or an assumption — in [[cpp26-contracts|C++26's terminology]], `enforce` vs `ignore` semantics). It does not statically prove correctness. For static proof you need a proof assistant like Lean 4 or a dependently-typed language like Idris — see the wiki's pages on those.

**Type system.** DbC complements types; it does not replace them. Types capture structural properties ("this is an integer"); contracts capture predicates over values ("this integer is positive"). The two are orthogonal axes of specification.

**Refinement types.** Languages like Liquid Haskell and F* express contract-like constraints as part of the type (`type Pos = { x : Int | x > 0 }`) and statically verify them. DbC is the dynamic-check analog of the same idea; refinement types are the static-proof version. The two converge in expressiveness but diverge sharply in implementation cost.

## Languages where DbC is first-class

- **Eiffel (1986)** — the original. Class invariants, inheritance-aware contracts, the whole package.
- **D (2001)** — `in`, `out`, `invariant` keywords; checked in debug builds.
- **Ada (2012, Ada 2012)** — pre/post aspects on subprograms, type invariants.
- **Spec# / Code Contracts (.NET, deprecated 2014)** — Microsoft's contract library; deprecated but spawned the design that became C# nullable reference types.
- **Java (JSR 305 → Checker Framework)** — third-party tools rather than language features.
- **C++26 (P2900)** — see [[cpp26-contracts]].
- **Rust (RFC pending)** — `#[requires]` / `#[ensures]` attributes via the `contracts` crate; an experimental language-level RFC exists but is not on a near-term track.

## Connection to type-driven design

DbC sits in the same family as [[type-driven-development]], [[parse-dont-validate]], and [[std-expected|monadic error handling]] — all of them push specification from comments and conventions into machine-checkable form. The differences:

- Type-driven design encodes invariants into types ([[newtype-pattern]], [[typestate-pattern]]). Verification is static; cost is at the type system.
- DbC encodes invariants into contracts. Verification is dynamic (or assumed); cost is at runtime or at the optimizer.
- Refinement types and dependent types unify the two: contracts that the compiler can statically check.

The right tool depends on what you can express in your type system and what you can afford to verify. In Rust, [[parse-dont-validate]] takes you a long way because the type system is expressive enough. In C++, [[cpp26-contracts]] cover the cases where the type system runs out. The combination of [[type-driven-development]] *and* contracts — types for structural invariants, contracts for value-level predicates — is the practical answer for most production code.

## The slow uptake question

DbC has been broadly understood for thirty-five years and is provably useful, yet adoption outside Eiffel remained marginal until C++26's contracts. Why?

The honest answer: contracts require organizational discipline — every function gets a contract, every contract is reviewed, violations are taken seriously. That's expensive process for an unclear payoff in most codebases. The languages that *required* contracts (Eiffel) never reached mass adoption; the languages that made them optional (Ada, D) saw them used in domains with already-strong correctness culture (avionics, safety-critical systems).

C++26's bet, and the bet behind the Rust RFC, is that *cheap, optional, optimizer-integrated* contracts can clear the adoption bar in a way that prior attempts couldn't. The first wave of production data should arrive in 2027–2028; whether the bet pays off is genuinely open.
