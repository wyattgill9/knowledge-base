---
tags:
  - cpp
  - design-patterns
  - architecture
sources:
  - "Raw/C++/Modern C++ Design Patterns (C++23 and Beyond).md"
last_updated: 2026-05-11
---

# C++26 Contracts

C++26 contracts — accepted into the working draft via paper P2900 — add language-level **preconditions** (`pre(...)`), **postconditions** (`post(r: ...)`), and **assertions** (`contract_assert(...)`) to function declarations and definitions. They replace the long tail of `assert()` macros, `if (...) throw std::logic_error(...)` checks, and undocumented invariants that defined Design-by-Contract in C++ for decades, and they do so as a real language feature visible to the compiler — meaning the optimizer can use them as assumptions in release builds and the type checker can refuse violations at compile time when they're evident.

## The pattern

```cpp
class CircularBuffer {
    int pop()
        pre(size_ > 0)
        post(r: r >= min_value_ && r <= max_value_)
    {
        --size_;
        return buffer_[head_++ % capacity_];
    }

    void push(int value)
        pre(size_ < capacity_)
        pre(value >= min_value_ && value <= max_value_)
        post(size_ > 0)
    {
        buffer_[tail_++ % capacity_] = value;
        ++size_;
    }
};
```

The `pre`/`post` clauses go between the parameter list and the function body — visible in the declaration, part of the function's type-level contract, not buried in the implementation. The `r:` in `post` names the return value; the postcondition expression can reference it.

## Configurable enforcement

The deepest design choice in P2900 is that contracts are **not** unconditionally checked. Each contract has a *semantic*: `ignore` (no check, behaves as if the contract weren't there), `observe` (check and report violations via the contract-violation handler without terminating), or `enforce` (check and terminate on violation, the moral equivalent of `abort()`). The semantic is set per build mode — typically `enforce` in debug, `observe` in test/staging, `ignore` in production-hot release.

This is the long-running debate that killed C++20 contracts (which had a much more complex model) resolved in favor of a simple three-mode dial. It also means contracts can simultaneously serve as:

- **Documentation** — the precondition is part of the signature; readers see it without reading the body.
- **Debug-time checks** — `enforce` mode behaves like an assertion.
- **Optimizer hints** — even in `ignore` mode, some compilers will treat the contract as `__builtin_assume`, propagating the assumption into code generation. Loops that the compiler can prove iterate at least once skip an iteration check; range checks on `pop()` consumers fold away.
- **Static analysis input** — clang-tidy, MSVC `/analyze`, and external tools can read contracts and verify them across call sites without running the code.

## What it replaces

**`assert()` macros** — the C-era predicate-check macro that vanishes in release builds. Contracts subsume it (`contract_assert(x > 0)`) but add documentation, configurability, and optimizer integration.

**Defensive `if-throw` patterns** — the `if (size == 0) throw std::logic_error("pop from empty");` pattern that pollutes function bodies. Contracts move this into the function signature.

**Implicit invariants documented only in comments** — every codebase has them; contracts make them machine-checkable.

**External contract libraries** — Boost.Contract, JSF++ macros, the various in-house assertion frameworks. Contracts are now standard.

## The Design-by-Contract lineage

The idea originated with Bertrand Meyer and Eiffel in the 1980s — function-level invariants as first-class language constructs. Meyer's argument was that types alone are insufficient to express what a function expects of its inputs (the precondition) and what it promises about its outputs (the postcondition), and that making these contractual obligations explicit and machine-checked closes a class of bugs that the type system on its own cannot. See [[design-by-contract]] for the broader concept.

Eiffel made contracts part of the inheritance contract too — derived classes could weaken preconditions and strengthen postconditions (Liskov Substitution as a language-level rule). C++26 contracts deliberately stop short of this; they are function-local. C++29 may extend them to class invariants and inheritance-aware contracts, but P2900 confined itself to the smaller, shippable subset.

## What it does *not* do

**Verify correctness.** A contract is a runtime check (or an optimizer hint), not a proof. There is no formal verification component — see [[lean-4]] or [[rocq]] for languages where the type system itself is the proof checker.

**Replace `std::expected`.** Contracts capture programmer-error conditions ("you passed a negative size") that should never happen in correct code; [[std-expected]] captures recoverable error states ("the file didn't exist") that are part of normal control flow. The dividing line: if `enforce` mode catching the violation would be a bug report, it's a contract; if it would be a normal "expected" outcome, it's an `expected`.

**Run at compile time.** Contracts are runtime in their enforcement semantics, though they participate in the compile-time assumption propagation. They are not [[cpp26-static-reflection|static-reflection]]-style compile-time facts.

## Performance characteristics

In `enforce` mode, a precondition is a branch — one or two cycles on a correctly-predicted always-true path. In `ignore` mode, it's nothing — except when the compiler treats it as `__builtin_assume`, in which case it can be net *negative* cost (the assumption enables optimizations that wouldn't otherwise be safe). The realistic measurement: enabling enforce-mode contracts in a typical codebase costs single-digit percent; the optimizer wins from assumption propagation often offset it.

## Position in the pattern shift

Within [[modern-cpp-design-patterns]], contracts are the precondition/postcondition entry — the analog at the function-signature layer of what [[std-expected]] is at the return-value layer. Both push correctness from "runtime hope" to "contract checked by the compiler or the runtime, with the choice configurable." They join [[cpp26-static-reflection]] and [[senders-receivers]] as the three C++26 features that together pull C++ much closer to the typed, contract-aware, structured-concurrency world that Rust, Swift, and Kotlin already inhabit — and that this wiki documents in depth on the Rust side under [[type-driven-development]].

## Adoption timeline

P2900 was accepted into C++26 in 2024. Implementations are landing through 2026: Clang has an experimental implementation behind `-fcontracts`, GCC's implementation is in progress with limited semantics, MSVC has announced support targeting late 2026. Production use will lag the implementation by a year or two while compiler stabilization, tooling integration (debugger support for inspecting violations), and library updates catch up.
