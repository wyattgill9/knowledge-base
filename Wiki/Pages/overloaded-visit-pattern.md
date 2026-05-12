---
tags:
  - cpp
  - design-patterns
  - type-theory
sources:
  - "Raw/C++/Modern C++ Design Patterns (C++23 and Beyond).md"
last_updated: 2026-05-11
---

# `std::visit` + Overloaded Pattern

The `std::visit` + `overloaded` idiom is the modern C++ replacement for virtual dispatch over a *closed* set of types: model the cases as `std::variant<Move, Attack, Heal, Quit>`, visit with a variadic lambda overload set, and let the compiler enforce exhaustiveness. It is value-semantic, supports any types (no inheritance required), and produces the closest thing C++ has to Rust's `match` or Swift's `switch` on enums-with-payload. C++23's deduction improvements made the pattern read cleanly enough that it has become the de facto way to dispatch over algebraic data types until full pattern matching (P2688 `inspect`) ships.

## The two pieces

```cpp
// 1. The overload-set helper — three lines that subsume the visitor pattern
template <class... Ts>
struct overloaded : Ts... { using Ts::operator()...; };

// 2. The variant + visit + per-alternative lambda
using Command = std::variant<Move, Attack, Heal, Quit>;

void handle(const Command& cmd) {
    std::visit(overloaded{
        [](const Move& m)   { player.move_to(m.target); },
        [](const Attack& a) { combat.resolve(a); },
        [](const Heal& h)   { player.hp += h.amount; },
        [](const Quit&)     { engine.shutdown(); }
    }, cmd);
}
```

The `overloaded` template is three lines of C++17 — a struct that inherits from each lambda's anonymous type and pulls each `operator()` into its own overload set. C++17 also enabled the class-template argument deduction that lets you write `overloaded{...}` without spelling out template parameters. C++23 cleaned up a few remaining rough edges (deduction guides became implicit in more cases).

## What it replaces

**Virtual dispatch over a closed hierarchy.** The GoF answer was an abstract base class with a `virtual void execute() = 0;` and one derived class per command. Disadvantages: heap allocation per command, vptr per object, inheritance constraint, hard to compose. The variant approach stores each command by value in the same `Command`, dispatches via a jump table, and lets you add behaviors as free functions or lambdas rather than as methods.

**The Visitor pattern** in the literal GoF sense. `std::visit` is the visitor; `overloaded` is the visitor's class. Where the GoF visitor required every concrete class to implement `accept(Visitor&)` and the visitor to declare a `visit(Concrete&)` overload per case, the modern version is one `std::visit` call and a brace-initialized lambda set.

**Manual `if-else` chains over a tag.** The pre-variant equivalent was a struct with an enum discriminator and a union or `std::any`; dispatch was a sequence of `if (cmd.kind == ...) ...` branches. The modern version is exhaustive by construction — adding a `Triangle` to the variant breaks compilation at every `std::visit` site that doesn't handle it, exactly the property [[type-driven-development]] points at as "make illegal states unrepresentable."

## Exhaustiveness

`std::visit` requires that the overload set cover every type in the variant. If you add `Sleep` to the `Command` variant and forget to add a `[](const Sleep&) { ... }` lambda, the compilation fails — not at the call site as in some languages, but inside the visit machinery, with an error pointing to the missing overload. This is the C++ equivalent of Rust's "non-exhaustive match" compile error, with the same load-bearing property: you can refactor a variant freely and the compiler tells you everywhere you missed a case.

For deliberately partial matching, use a catch-all `[](const auto&) { /* default */ }` as the last overload — `overloaded` will match anything else through the templated `operator()`.

## Where it's awkward

**Cross-cutting state.** If every case needs access to a shared resource, you end up with N lambdas all closing over the same variables. Sometimes a regular visitor class with members is cleaner.

**Returning values.** `std::visit` returns whatever the visitor returns; if different lambdas return different types, you need a common type or `std::variant` again. C++23 added `std::expected`-based visit overloads but the ergonomics still lag Rust's `match` on this axis.

**Open hierarchies.** If the set of types is genuinely open (plugins, user-extensible kinds), virtual dispatch or `std::move_only_function`-based polymorphism is the better tool. See [[std-move-only-function]] for the open-set counterpart.

## What's still missing

C++26 was on track to ship `inspect` (proposal P2688) — true pattern matching with destructuring, guards, and nested patterns. It missed the C++26 standard cutoff and is now targeted at C++29. When it lands, the verbose `std::visit` + `overloaded` idiom will look the way `std::function` does after `std::move_only_function` arrived: still functional, but obviously the older way of doing it.

Until then, the variant + visit pattern is the practical answer to algebraic data type dispatch in C++ — and it is good enough that production C++ codebases (LLVM, Clang, MongoDB, every recent compiler frontend) have built around it.

## Position in the pattern shift

In [[modern-cpp-design-patterns]] this is the closed-set dispatch entry: where [[std-move-only-function]] handles open-set dispatch and [[deducing-this]] handles compile-time-known derived types, `std::visit` + `overloaded` handles the case where you know all the alternatives statically and want compile-time exhaustiveness. Together the three retire the dominant uses of virtual dispatch in modern C++ code, leaving virtual functions for the genuinely-open-hierarchy case where they were always the best fit.
