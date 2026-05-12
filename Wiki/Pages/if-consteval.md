---
tags:
  - cpp
  - design-patterns
  - type-theory
sources:
  - "Raw/C++/Modern C++ Design Patterns (C++23 and Beyond).md"
last_updated: 2026-05-11
---

# `if consteval`

`if consteval` is C++23's compile-time/runtime branch — a statement inside a `constexpr` function that picks a different code path depending on whether the call is being evaluated at compile time or at runtime. It replaces the older `if (std::is_constant_evaluated())` idiom, which had a subtle pitfall (you could nest it inside `if constexpr` and get wrong behavior), and enables a clean compile-time strategy pattern in a single function body.

## The pattern

```cpp
constexpr double fast_sqrt(double x) {
    if consteval {
        // Compile-time: Newton's method, because std::sqrt
        // wasn't constexpr before C++26.
        double guess = x / 2.0;
        for (int i = 0; i < 20; ++i)
            guess = (guess + x / guess) / 2.0;
        return guess;
    } else {
        // Runtime: hardware sqrt instruction.
        return std::sqrt(x);
    }
}

constexpr double a = fast_sqrt(2.0);  // Newton's method, at compile time
double b = fast_sqrt(some_input);     // SQRTSD instruction, at runtime
```

The compile-time branch produces an answer the compiler can use as a constant expression — folding into `constexpr` variables, `static_assert`, template arguments. The runtime branch uses the fast hardware path. The two branches don't need to compute the same way; they need only agree on the result for inputs that both can evaluate.

## Why it's better than `is_constant_evaluated()`

The C++20 predicate `std::is_constant_evaluated()` returns a `bool` — and a `bool` is just a value. The trap was that `if constexpr (std::is_constant_evaluated())` is *always true* (because the predicate itself is a constant expression evaluated by `if constexpr`), making both the syntactic and the intuitive use of the feature wrong. You had to write `if (std::is_constant_evaluated())` — a regular runtime `if` whose branches the compiler then folds. That worked but was fragile and easy to misread.

`if consteval` is a dedicated language construct, not a predicate. It is *only* valid as a statement, never wrapped in `if constexpr`, never returning a value. The trap is eliminated by syntax.

## Use cases

**Different algorithms for different contexts.** The Newton's-method-vs-hardware-sqrt example above. Hash functions: `constexpr` strings can use a slow-but-pure hash; runtime can use AES-NI–accelerated [[rapidhash]] or similar.

**Different precision or correctness guarantees.** Compile-time computation can afford a 20-iteration Newton iteration; runtime can use a single fast intrinsic.

**Compile-time diagnostics.** A `constexpr` function can use `if consteval` to do extra checking at compile time that would be too expensive at runtime — verifying invariants of literal-input ranges, validating string formats, etc.

**Library implementation.** Standard library implementations use `if consteval` to provide constexpr-friendly versions of operations that the language normally implements with non-constexpr machinery (memcpy, atomic intrinsics, SIMD).

## The deeper trajectory

Compile-time evaluation in C++ has been steadily expanding for fifteen years: `constexpr` functions (C++11), `constexpr` lambdas (C++17), `consteval` functions that *must* be compile-time (C++20), `constexpr` containers like `std::vector` (C++20), `constexpr` exceptions (C++26 proposed). `if consteval` is the connective tissue that lets a single function smoothly straddle the two regimes.

This connects to the broader theme of [[modern-cpp-design-patterns]] — moving correctness and computation toward the compiler. [[cpp26-static-reflection|Static reflection]] takes the next step by giving the compile-time computation access to the program's type structure. [[cpp26-contracts|Contracts]] take a parallel step by giving the compiler access to invariants. Together, they push the boundary between "compiler does this" and "runtime does this" measurably further toward the compiler than C++20 left it.

## The Rust comparison

Rust's `const fn` plays a similar role but currently lacks the `if consteval` analog — a `const fn` runs the same code at compile time and runtime, and any operation usable in `const` context must work uniformly. The Rust approach is stricter (no divergence allowed); the C++ approach is more flexible (different code allowed when justified). Both are slowly converging on a richer compile-time evaluation model; C++ is moving faster on this axis as of 2026 because its 35-year history of template metaprogramming gives it a head start on the implementation strategy.
