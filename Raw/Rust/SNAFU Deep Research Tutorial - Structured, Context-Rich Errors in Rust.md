## Executive summary

SNAFU is an ergonomic Rust error-handling library that focuses on **mapping underlying errors into domain-specific errors while adding structured context**, especially when the *same* underlying error type can occur in multiple conceptual situations. Its core mechanism is the `#[derive(Snafu)]` macro, which generates **context selector** types for each error variant; these selectors make it idiomatic to attach semantic context at call sites via `ResultExt::context` / `with_context` and related traits. citeturn3search2turn2search7turn3search1

Compared to “derive-only” crates (e.g., `thiserror`), SNAFU aims to reduce ambiguity at failure boundaries by encouraging **many small, module-scoped error types** and deliberate context layering. It also includes built-in support for **backtraces**, optional **user-friendly reporting** (`Report` and `#[snafu::report]`), compatibility with **`no_std`**, and optional extensions for **futures/streams** (`TryFutureExt` / `TryStreamExt`). citeturn3search2turn2search5turn3search0turn12search0

Assumptions (as requested): your project’s **Rust edition**, **MSRV**, and **target platforms** are unspecified. This tutorial therefore calls out where SNAFU depends on `std`, `alloc`, async runtimes, or additional tooling, and it shows patterns that scale from libraries to application boundaries. citeturn12search0turn9search5turn10search10

## What SNAFU is for and what it optimizes

SNAFU’s crate-level documentation defines its purpose succinctly: it is a library to “easily generate errors and add information to underlying errors,” with special emphasis on cases where the **same underlying error** can happen in different contexts. citeturn3search2 That emphasis is the key to understanding SNAFU’s design: rather than treating errors as just strings or a monolithic enum, SNAFU steers you toward **domain errors** with **structured fields** that capture the meaningful “what were we doing?” information at each boundary. citeturn3search2turn2search7

A telling default is that SNAFU-generated context selectors are **private** by default, reflecting the project’s opinion that modules should define cohesive error types locally; this keeps matches and error handling focused and reduces accidental coupling. citeturn8search3turn6search6 If you do export selectors, the docs warn that their API stability is not guaranteed—exporting selectors in a public API may create semver hazards if SNAFU internals evolve. citeturn10search8turn8search3

SNAFU is also explicitly designed to be usable in both libraries and apps. Its public metadata indicates it is `no_std` at the crate level (with features that enable `std` behavior), and it includes mechanisms ranging from “turnkey string errors” (`Whatever`) to fully custom typed error enums/structs and app-friendly reporting. citeturn13search0turn3search2turn12search0

### Maintenance and compatibility snapshot

SNAFU is actively maintained: version **0.9.0** was released on **2026-03-02**, and its changelog documents breaking changes and ongoing improvements (MSRV, default compatibility level, `Whatever` thread-safety, location interoperability). citeturn5search2 Its crate metadata specifies **edition = 2018**, **rust-version = 1.65**, and dual licensing **MIT OR Apache-2.0**. citeturn12search0

Compatibility is controlled by Cargo feature flags. The guide states SNAFU is tested back to Rust **1.65** and that the default compatibility target is **Rust 1.81**—notably using `core::error::Error` instead of `std::error::Error` when the `rust_1_81` feature is enabled. citeturn10search10turn12search0

## Core concepts you must internalize

### The `Snafu` derive and context selectors

`#[derive(Snafu)]` is the entrypoint for defining SNAFU-enhanced error types, intended to require little configuration for common cases while offering deep customization when needed. citeturn2search7 The derive generates **context selectors**—additional types used to construct errors ergonomically and to attach context while preserving sources. The guide’s “what code is generated” section explicitly describes these selectors and highlights that they are created per enum variant and strip out `source`/`backtrace` fields because SNAFU handles those automatically through the selector + conversion machinery. citeturn6search7

A critical ergonomic detail: when you call `.context(VariantSnafu { ... })`, the selector will call `Into::into` on each field, so your provided field types **do not need to exactly match** the error variant’s field types. This commonly means you can pass `&str` into a `String` field or similar, reducing boilerplate. citeturn9search7turn3search0

### `ResultExt`, `OptionExt`, and the `context` / `with_context` split

SNAFU’s `ResultExt` trait adds methods such as `context` and `with_context` for mapping errors into richer domain errors. citeturn0search5turn9search7 The guiding principle is:

- Use **`context(selector)`** when the context is cheap to build or you already have the fields.
- Use **`with_context(|err| selector)`** when context is expensive (allocations, formatting, JSON snippets), because it is evaluated **only on error**, and the closure receives the underlying error by mutable reference (allowing inspection or extracting details). citeturn9search7turn0search5

Similarly, `OptionExt` converts `Option<T>` into `Result<T, E>` with structured context via `context` / `with_context`. This is extremely useful when “missing value” is a domain error, not just `None`. citeturn3search1

### Macro ergonomics: `ensure!`, `whatever!`, and friends

SNAFU includes macros for common patterns:

- `ensure!` returns early with a SNAFU-produced error if a condition fails.
- `whatever!` and `ensure_whatever!` provide “turnkey” string errors (useful at first, but often outgrown).
- `location!` constructs a `Location` for implicit source location capture. citeturn3search2turn5search1turn13search0

The crate-level docs explicitly recommend starting with `Whatever` and `whatever!` if you want minimal hassle, and then “graduate” to custom error types when you hit limitations. citeturn3search2turn5search6

### Error composition, `source()` chains, and structured context

SNAFU integrates with Rust’s standard error-chaining model: your error variants/structs typically include a `source` field (or a custom field marked `#[snafu(source)]`), and SNAFU uses that to implement `Error::source()` chaining. citeturn9search2turn5search0 The `ErrorCompat` trait provides `iter_chain()` to traverse an error and its sources recursively (starting at the current error); the docs recommend calling these methods in fully-qualified form for future-proofing. citeturn5search0turn13search0

SNAFU also provides tools to reduce redundant messages in displays: the upgrading guide notes that SNAFU’s default `Display` no longer includes the source’s `Display`, aligning with broader Rust error-handling guidance, and recommends using higher-level tools like `Report`/`report` or lower-level tools like `CleanedErrorText` to combine chain messages when you actually want the full story. citeturn7search3turn5search1

### Backtraces and “SpanTrace” thinking for async code

SNAFU’s `Backtrace` type is designed to be safe to include “liberally” in error types: capturing is gated via `RUST_BACKTRACE` and `RUST_LIB_BACKTRACE`, and the docs emphasize that capture can be memory-intensive/slow, so the env vars allow you to only pay the cost when needed. citeturn12search1 In the backtrace example guide, SNAFU recommends adding a `Backtrace` field to **leaf errors** (those without a `source`) and explains an important performance pattern: for errors created frequently (flow-control-ish situations), use `Option<Backtrace>` so a backtrace is captured only when an end user enables it. citeturn15search2

In async services, stack backtraces can be less useful because executor frames dominate; `tracing-error`’s `SpanTrace` captures *logical span context* and is described as lightweight and lazily formatted. citeturn3search4turn3search8 SNAFU does not “own” SpanTrace, but you can integrate it by capturing SpanTraces as structured fields (often via SNAFU’s implicit data mechanism and a local newtype). citeturn9search2turn3search4

## Setup and feature flags in practice

SNAFU’s `Cargo.toml` defines default features and key options. Default features are `["std", "rust_1_81"]`; `std` implies `alloc`. citeturn12search0 The guide also recommends keeping heavy/unstable features in applications rather than libraries (e.g., provider API, certain backtrace interoperability modes). citeturn15search14turn9search5turn12search0

### Example `Cargo.toml` configurations

```toml
[dependencies]
snafu = "0.9"
```

Use this for typical `std` environments (most apps and many libraries). Default features enable `std` and the `rust_1_81` compatibility mode. citeturn12search0turn10search10

```toml
[dependencies]
snafu = { version = "0.9", features = ["futures"] }
futures = "0.3"
```

Enable this when you want `TryFutureExt` / `TryStreamExt` context methods on futures/streams returning `Result`. citeturn12search0turn3search0turn3search3

```toml
[dependencies]
# no_std + alloc (e.g., embedded with an allocator)
snafu = { version = "0.9", default-features = false, features = ["alloc", "rust_1_81"] }
```

This aligns with the guide’s note that using `alloc` without `std` requires `rust_1_81`. citeturn9search5turn12search0turn10search10

```toml
[dependencies]
# Application-only: alias snafu::Backtrace to backtrace::Backtrace for interop
snafu = { version = "0.9", features = ["backtraces-impl-backtrace-crate"] }
```

This feature makes SNAFU’s `Backtrace` type become `backtrace::Backtrace`, improving interoperability with crates that require that concrete type; it’s recommended mainly for applications. citeturn9search5turn12search0

## Progressive examples: idiomatic SNAFU in real code

The examples below build up from “typed errors” to “structured context,” then into async and boundary integration. They are intentionally short; you can paste them into a small project and expand. The SNAFU docs explicitly encourage reviewing guide examples and the generated `[src]` links for deeper understanding. citeturn15search6turn10search4

### Example: simple enum error with `ensure!` and `.fail()`

This is the smallest “custom error type” pattern: define an error enum and use `ensure!` (or `.fail()`) to construct it.

```rust
use snafu::prelude::*;

#[derive(Debug, Snafu)]
pub enum Error {
    #[snafu(display("ID must be >= 10, got {id}"))]
    InvalidId { id: u16 },
}

pub type Result<T> = std::result::Result<T, Error>;

pub fn validate_id(id: u16) -> Result<()> {
    ensure!(id >= 10, InvalidIdSnafu { id });
    Ok(())
}
```

Why this is idiomatic: SNAFU generates a context selector `InvalidIdSnafu` used with `ensure!` to create errors with minimal boilerplate. citeturn3search2turn13search0

### Example: adding structured context at call sites with `ResultExt::context`

This is the “SNAFU signature move”: map an underlying error (here `std::io::Error`) into different meaningful domain variants with structured fields (e.g., `path`), even though the source type is the same.

```rust
use snafu::prelude::*;
use std::{fs, io, path::PathBuf};

#[derive(Debug, Snafu)]
pub enum Error {
    #[snafu(display("Could not open config at {}", path.display()))]
    OpenConfig { path: PathBuf, source: io::Error },

    #[snafu(display("Could not write output to {}", path.display()))]
    WriteOutput { path: PathBuf, source: io::Error },
}

pub type Result<T> = std::result::Result<T, Error>;

pub fn load_then_write(config: &PathBuf, out: &PathBuf) -> Result<()> {
    let bytes = fs::read(config).context(OpenConfigSnafu { path: config.clone() })?;
    fs::write(out, bytes).context(WriteOutputSnafu { path: out.clone() })?;
    Ok(())
}
```

What you get: two distinct errors with the same `io::Error` source type, so downstream logs and debugging are immediately clearer about *what operation* failed. This is exactly the scenario emphasized in the crate docs. citeturn3search2turn2search7

### Example: using `with_context` for lazy context and fewer allocations

`with_context` is your default when context is expensive. SNAFU’s docs explicitly note this may not always be necessary (because fields use `Into::into`), but it’s valuable when you want to allocate only on failure. citeturn9search7turn3search0

```rust
use snafu::prelude::*;
use std::{fs, io, path::PathBuf};

#[derive(Debug, Snafu)]
pub enum Error {
    #[snafu(display("Reading file failed; tried path={path:?}"))]
    ReadFile { path: PathBuf, source: io::Error },
}

pub type Result<T> = std::result::Result<T, Error>;

pub fn read_file(path: PathBuf) -> Result<String> {
    fs::read_to_string(&path).with_context(|_source| ReadFileSnafu { path }) // alloc only on Err
}
```

Notice the closure gets access to the source error (`&_source`), which can be useful for building context based on error kind or other details. citeturn0search5turn9search7

### Example: backtraces with `Backtrace` and the “hot loop” `Option<Backtrace>` trick

SNAFU’s backtrace guide recommends:

- Put `Backtrace` on leaf errors by default.
- Use `Option<Backtrace>` for high-frequency errors where a backtrace is rarely needed; SNAFU will only capture it when the user enables backtraces via env vars. citeturn15search2turn15search13

```rust
use snafu::prelude::*;

#[derive(Debug, Snafu)]
pub enum Error {
    // Leaf failures: capture backtrace (as configured)
    #[snafu(display("Database connection failed"))]
    ConnectDb { backtrace: snafu::Backtrace },

    // Hot-path “error-like” outcome: only capture if RUST_LIB_BACKTRACE / RUST_BACKTRACE enables it
    #[snafu(display("Rate limited"))]
    RateLimited { backtrace: Option<snafu::Backtrace> },
}

pub type Result<T> = std::result::Result<T, Error>;

pub fn connect() -> Result<()> {
    ConnectDbSnafu.fail()
}
```

Backtrace collection behavior matters: `Backtrace::capture()` is a **noop unless** `RUST_BACKTRACE` or `RUST_LIB_BACKTRACE` is set appropriately; forcing capture is possible but potentially expensive. citeturn12search1turn15search13

When you want to print backtraces, the guide recommends using `ErrorCompat::backtrace` in fully-qualified form. citeturn15search2turn15search12

### Example: app-friendly reporting with `#[snafu::report]`

By default, `Display` prints only the top-level error message; SNAFU’s reporting mechanism prints the error plus its chain (and backtrace when applicable). The crate docs show using `#[snafu::report]` on `main` as the intended ergonomic entry point. citeturn3search2turn2search5

```rust
use snafu::prelude::*;

#[derive(Debug, Snafu)]
#[snafu(display("Could not load configuration file {path}"))]
struct ConfigError {
    source: std::io::Error,
    path: String,
}

fn read_config(path: &str) -> Result<String, ConfigError> {
    std::fs::read_to_string(path).context(ConfigSnafu { path: path.to_string() })
}

#[snafu::report]
fn main() -> Result<(), ConfigError> {
    let _cfg = read_config("missing.ini")?;
    Ok(())
}
```

The `report` macro is documented as sugar over returning `Report`; it can be used with other common proc macros and has been tested with `tokio::{main,test}` and `async_std::{main,test}`. citeturn2search5turn2search2

### Example: async ergonomics via SNAFU’s `futures` feature

When you enable the `futures` feature, SNAFU provides `TryFutureExt` and `TryStreamExt` traits so you can attach context in combinator chains for futures and streams returning `Result`. citeturn3search3turn3search0turn15search14

```rust
use futures::future::TryFuture;
use snafu::prelude::*;

#[derive(Debug, Snafu)]
enum Error {
    #[snafu(display("auth failed for {user_name} ({user_id})"))]
    Authenticating { user_name: String, user_id: i32, source: ApiError },
}

type ApiError = std::io::Error;

fn another_function() -> impl TryFuture<Ok = i32, Error = ApiError> {
    futures::future::err(std::io::Error::new(std::io::ErrorKind::Other, "nope"))
}

fn example() -> impl TryFuture<Ok = i32, Error = Error> {
    another_function().context(AuthenticatingSnafu {
        user_name: "admin",
        user_id: 42,
    })
}
```

This pattern is straight from SNAFU’s `TryFutureExt` docs and is one of the cleanest ways to keep structured context in async code without manual `map_err`. citeturn3search0turn3search3

## Integrating SNAFU with `anyhow`, `miette`, and `tracing-error`

SNAFU is very compatible with the rest of the Rust ecosystem because it builds on standard error traits (and can target `core::error::Error` under `rust_1_81`). citeturn10search10turn12search0 Integration is best thought of as a **layering** strategy: use typed, structured errors inside libraries/modules; then convert at the application boundary to a reporting type (e.g., `anyhow::Error` or `miette::Report`) if desired.

### Mermaid diagram: error flow from library to boundary to report

```mermaid
flowchart TB
  subgraph Lib[Library / domain modules]
    A[Typed SNAFU errors\n(enum/struct + source/backtrace fields)]
    A --> B[ResultExt::context / with_context\nadds structured context selectors]
    B --> C[Error::source() chain\nErrorCompat::iter_chain()]
  end

  subgraph Boundary[Application boundary]
    C --> D[Convert to app report type\n(anyhow::Error / miette::Report)]
  end

  subgraph Output[Presentation]
    D --> E1[SNAFU Report / #[snafu::report]\n(opinionated chain output)]
    D --> E2[miette rendering\n(span/labels if Diagnostic)]
    D --> E3[tracing-error SpanTrace\n(logical async context)]
  end
```

This reflects SNAFU’s own emphasis on structured domain errors plus optional app-friendly reporting. citeturn3search2turn2search2turn3search4

### Boundary conversion to `anyhow`

`anyhow::Error` is a dynamic error wrapper intended for application code. It implements `From<E>` (and provides `Error::new`) for any underlying error `E` that implements `std::error::Error + Send + Sync + 'static`. citeturn4search1turn4search0 That means SNAFU-defined errors can usually be propagated into `anyhow::Result` via `?` in `main`.

```rust
use anyhow::Result;

mod mylib {
    use snafu::prelude::*;

    #[derive(Debug, Snafu)]
    pub enum Error {
        #[snafu(display("bad input"))]
        BadInput,
    }

    pub fn run() -> std::result::Result<(), Error> {
        BadInputSnafu.fail()
    }
}

fn main() -> Result<()> {
    mylib::run()?; // converted into anyhow::Error via From<E>
    Ok(())
}
```

This pattern matches Anyhow’s documented positioning: “use Anyhow if you don’t care what error type your functions return,” and use typed errors (often via derive crates) in library-like parts of the codebase. citeturn14search1turn14search0

### Boundary conversion to `miette`

Miette is a diagnostics/reporting library. Its `IntoDiagnostic` trait converts a `Result<T, E>` where `E: std::error::Error + Send + Sync + 'static` into `Result<T, miette::Report>`. citeturn3search7 This is convenient when your library errors are plain `Error` types (SNAFU or otherwise) and you want miette’s reporting at the application layer.

```rust
use miette::{IntoDiagnostic, Result};

fn main() -> Result<()> {
    mylib::run().into_diagnostic()?; // convert Error -> miette::Report
    Ok(())
}

mod mylib {
    use snafu::prelude::*;

    #[derive(Debug, Snafu)]
    pub enum Error {
        #[snafu(display("operation failed"))]
        Failed,
    }

    pub fn run() -> std::result::Result<(), Error> {
        FailedSnafu.fail()
    }
}
```

Be aware of miette’s own warning: converting an error that *already* implements `Diagnostic` into a `Report` this way may discard richer diagnostic info. But for plain error types, it’s a clean boundary adapter. citeturn3search7

### Using `tracing-error` SpanTrace alongside SNAFU

`tracing-error` documents `SpanTrace` as a “relative of `std::backtrace::Backtrace`” that captures the current tracing span context, which can be more useful in async systems where backtraces show executor frames instead of logical call context. It’s also described as fairly lightweight and lazily formatted. citeturn3search4turn3search8

Because Rust’s orphan rules prevent implementing SNAFU’s `GenerateImplicitData` for `SpanTrace` directly (both are external), a common pattern is to define a local newtype:

```rust
use snafu::prelude::*;
use tracing_error::SpanTrace;

#[derive(Debug)]
struct MySpanTrace(SpanTrace);

impl snafu::GenerateImplicitData for MySpanTrace {
    fn generate() -> Self {
        MySpanTrace(SpanTrace::capture())
    }
}

#[derive(Debug, Snafu)]
pub struct Error {
    #[snafu(display("request failed"))]
    message: String,

    #[snafu(implicit)]
    span: MySpanTrace,
}
```

This leverages SNAFU’s `#[snafu(implicit)]` mechanism (which generates data without arguments for types implementing `GenerateImplicitData`) to capture trace context at the error construction site. citeturn9search2turn13search2turn3search4

## Migration tips, common pitfalls, and performance guidance

### Migration tips from `thiserror` or manual error enums

`thiserror` is a derive macro that generates standard `std::error::Error` impls and “deliberately does not appear in your public API.” citeturn14search0 If you already have `thiserror`-based errors, moving to SNAFU is less about “better derives” and more about adopting **context selectors** and **structured context at call sites**.

A pragmatic migration path looks like:

- Start by keeping your error shapes (enums/structs) and replace `#[derive(Error)]` + `#[error("...")]` with `#[derive(Snafu)]` + `#[snafu(display("..."))]`. SNAFU’s derive is intended to be low-config for typical usage. citeturn2search7turn14search0
- Replace ad-hoc `map_err(|e| MyError::Variant { source: e, ... })` with `.context(VariantSnafu { ... })`. This aligns with SNAFU’s “same source type, different contexts” emphasis. citeturn3search2turn2search7
- Where you previously relied on “transparent” wrappers for composition, consider `#[snafu(transparent)]` to delegate `Display` and `Error::source` to the underlying error and avoid redundant nesting. citeturn9search2
- If your public API needs to remain stable/opaque, adopt SNAFU’s “opaque error type” pattern: a public newtype wrapping an internal SNAFU error enum, with delegated traits and an easy `From` conversion. citeturn9search1turn9search6

### Common pitfalls and how to avoid them

The most common beginner confusion is that `.context(...)` expects a **context selector**, not the enum variant path. In other words, you write `.context(ReadConfiguration { ... })` (or `.context(ReadConfigurationSnafu { ... })` depending on the selector name), not `.context(Error::ReadConfiguration { ... })`. A community explanation summarizes this as: SNAFU generates a separate struct used to map the source error into the enum variant. citeturn6reddit47turn6search7

Visibility is the next pitfall: selectors are private by default. If you move error types across modules or want to use a selector outside its defining module, you likely need `#[snafu(visibility(...))]` on the error type or specific variants/selectors. citeturn8search3turn6search6 The docs also note that if you use `#[snafu(module)]` to avoid selector name conflicts, the generated module begins with `use super::*`, so required names must be in scope (sometimes requiring more explicit imports). citeturn0search0turn10search8

Finally, avoid overusing `#[snafu(context(false))]`. It exists for cases where there is truly no useful additional context and you want to use `?` directly, but SNAFU explicitly urges thinking about end users: context is often what makes an error actionable. citeturn7search2turn10search8

### Performance and allocation considerations

Much of SNAFU’s runtime cost is under your control because context is primarily “whatever fields you store.” The most important built-in cost center is **backtrace capture**:

- `Backtrace::capture()` is env-gated: it is a no-op unless `RUST_BACKTRACE` or `RUST_LIB_BACKTRACE` is set. Capturing can be memory-intensive and slow, so this gating is explicitly intended to make “liberal use” reasonable. citeturn12search1turn15search4
- For high-frequency errors, prefer `Option<Backtrace>` so you don’t capture unless the end user explicitly enables it. citeturn15search2turn15search13
- Prefer `with_context` when context requires allocations or formatting: it is called only on the error path. citeturn9search7turn0search5

Benchmarks: I did not find widely-cited, standalone benchmark suites comparing SNAFU’s overhead to alternatives in a way that isolates allocations and backtrace capture across identical workloads. The primary sources instead emphasize cost-control mechanisms (env-gated backtraces, `Option<Backtrace>`, lazy `with_context`) and recommend patterns accordingly. citeturn12search1turn15search2turn9search7

### Decision matrix: when SNAFU is the right tool

| If you are building… | Prefer SNAFU when… | Prefer alternatives when… |
|---|---|---|
| A library with a public API | You want stable, typed errors with structured context and encourage module-local error design; optionally provide an opaque wrapper type for API stability | You only want a lightweight derive for `Error`/`Display` and don’t need context selectors (often `thiserror`) citeturn14search0turn9search1 |
| A CLI or app | You want typed, structured errors internally and can use `#[snafu::report]` or convert to `anyhow`/`miette` at the boundary | You want to move fast and don’t care about typed errors across layers (often `anyhow`) citeturn14search1turn2search5 |
| Async services | You want structured context and can augment with backtraces and/or tracing `SpanTrace` (better logical context than stack traces) | You want an error stack/attachment model different from source-chaining (consider other ecosystems), or you want only tracing context and not typed errors citeturn3search4turn3search0 |
| `no_std` targets | You need `no_std` compatibility with optional `alloc`, and you want to keep typed error modeling | You require extremely minimal dependencies and will hand-roll error enums/traits (possible but more manual) citeturn13search0turn12search0turn9search5 |

## Concise comparison: SNAFU vs `thiserror` vs `anyhow` for library vs app use

| Dimension | SNAFU | thiserror | anyhow |
|---|---|---|---|
| Primary intent | Typed errors **plus structured context selectors** at call sites citeturn3search2turn2search7 | Typed error derive, minimal abstraction, doesn’t appear in public API citeturn14search0 | App-oriented dynamic error wrapper (`anyhow::Error`) for “easy” propagation citeturn14search1turn4search1 |
| Best fit | Libraries and large apps that benefit from semantic, structured contexts citeturn3search2turn8search3 | Libraries that want clean, stable error types with derive convenience citeturn14search0 | Application boundaries and prototypes where error type design is not a priority citeturn14search1turn14search5 |
| Context attachment style | Structured fields via selectors (`.context(...)`, `.with_context(...)`) citeturn0search5turn6search7 | Mostly via manual variants/fields; no selector-based mapping layer citeturn14search0 | Mostly string context (`Context`, `with_context`) citeturn14search1turn14search5 |
| Backtraces | Built-in `Backtrace` type with env-gated capture; `Option<Backtrace>` pattern for hot loops citeturn12search1turn15search2 | Provider-based backtrace support requires nightly for `Backtrace` field support citeturn14search0 | Guarantees a backtrace exists (captures if absent), env-gated for display; `Error::new` requires `Send + Sync + 'static` citeturn4search1turn4search0 |
| no_std story | `#![no_std]` crate with `std`/`alloc` features; `rust_1_81` compatibility uses `core::error::Error` citeturn13search0turn10search10turn12search0 | Primarily `std`-error derive (community maintains forks for strict `no_std`) citeturn14search0turn14search9 | Explicit no_std mode with allocator (disable default `std` feature) citeturn14search1turn14search5 |

