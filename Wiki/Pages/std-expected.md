---
tags:
  - cpp
  - error-handling
  - design-patterns
sources:
  - "Raw/C++/Modern C++ Design Patterns (C++23 and Beyond).md"
last_updated: 2026-05-11
---

# `std::expected`

`std::expected<T, E>` is C++23's sum type for fallible computation — it holds either a `T` (success) or an `E` (error), with monadic combinators (`and_then`, `transform`, `or_else`) that chain operations and short-circuit on failure. It is the C++ side of the same "either/result" answer that Haskell's `Either`, Rust's `Result<T, E>`, and Swift's `Result` adopted: errors become values, control flow stays linear, and exceptions become reserved for genuinely exceptional cases.

## The dilemma it ends

For two decades C++ developers chose between two unsatisfying defaults:

- **Exceptions** — composable but invisible in the type signature, expensive to throw, banned in many performance-sensitive codebases (Google, embedded, game engines).
- **Error codes** — visible but force `if (err)` boilerplate everywhere, with no way to carry a typed payload.

`std::expected<T, E>` is visible in the type, zero-cost when the success path is taken, carries arbitrary error payloads, and composes with the rest of the type system.

## The monadic combinators

The shape of an `std::expected` pipeline mirrors [[railway-oriented-programming]]:

```cpp
enum class ParseError { InvalidFormat, OutOfRange };

std::expected<int, ParseError>    parse_int(std::string_view sv);
std::expected<double, ParseError> to_celsius(int fahrenheit);

auto result = parse_int(input)
    .and_then(to_celsius)
    .transform([](double c) { return std::format("{:.1f}°C", c); })
    .or_else([](ParseError e) -> std::expected<std::string, ParseError> {
        log_error(e);
        return std::unexpected(e);
    });
```

- **`and_then(f)`** — if success, calls `f(value)` which itself must return an `expected`; if error, passes the error through. This is monadic bind.
- **`transform(f)`** — if success, calls `f(value)` returning a plain `T'` which is wrapped back into `expected<T', E>`; if error, passes through. This is `fmap`.
- **`or_else(f)`** — if error, calls `f(err)` returning a (potentially recovered) `expected`; if success, passes through.

The pipeline never branches on error explicitly. The compiler-generated control flow is equivalent to chained `if (err) return err;` blocks but reads like a fluent API.

## Idioms that mature with it

**Boundary conversion.** At the edges of a codebase you still talk to exception-throwing libraries or C-style error codes. The standard idiom is to convert at the boundary and stay in `expected` everywhere inside:

```cpp
std::expected<File, IoError> open(std::filesystem::path p) {
    try { return File{open_throwing(p)}; }
    catch (const std::system_error& e) { return std::unexpected(IoError::from(e)); }
}
```

**Error type unification.** `std::expected` doesn't help when each layer has its own `E`. The same patterns as Rust apply: a single domain `Error` enum, or a wrapper similar to [[anyhow]] / [[eyre]] (C++ has no direct analog yet, but `std::expected<T, std::exception_ptr>` plus contextual wrapping is the rough equivalent).

**`.value_or(default)`** for the common "give me a fallback" case without writing an `or_else` lambda.

## The Rust parallel

`std::expected<T, E>` ≈ Rust `Result<T, E>`. `and_then` is `Result::and_then`. `transform` is `Result::map`. `or_else` is `Result::or_else`. `std::unexpected(e)` is `Err(e)`. The mental model transfers verbatim — including the same trap that Rust users learned in 2015, which is that combinator pipelines obscure stack traces and propagate context poorly. The Rust ecosystem solved that with [[snafu]] / [[thiserror]] / [[error-stack]]; C++ will need similar third-party libraries to layer context onto `expected` chains. As of 2026 nothing has emerged as the consensus pick — `tl::expected` (the pre-standard backport) lacks context-attachment idioms, and the proposed `std::expected` extensions in P2952 are not yet shipped.

See [[rust-error-handling]] for the four-layer architecture (typed definition / type-erased reporting / diagnostic rendering / observability) that the C++ side is on the early steps of replicating.

## Performance characteristics

`std::expected` is a tagged union with a discriminator byte, sized as `max(sizeof(T), sizeof(E)) + sizeof(discriminator)` with alignment padding. The success path is a single branch (almost always correctly predicted toward success). There is no allocation, no exception machinery, no RTTI. Benchmarks on GCC 14 and Clang 19 show `expected`-returning functions within 1–2% of plain return-by-value when the success path is taken, versus 50–100× cost for thrown exceptions on the failure path.

The implication for hot paths is straightforward: use `std::expected` everywhere that "this can fail" is a normal control-flow concern, and reserve exceptions for genuinely exceptional conditions (out-of-memory, programmer errors, contract violations — see [[cpp26-contracts]]).

## Position in modern C++

`std::expected` is the single most-adopted C++23 feature in the codebases that surveyed it. It slots cleanly into existing code, immediately removes a class of ugly boilerplate, and connects to the broader shift documented in [[modern-cpp-design-patterns]] — from imperative error handling to value-flow composition.
