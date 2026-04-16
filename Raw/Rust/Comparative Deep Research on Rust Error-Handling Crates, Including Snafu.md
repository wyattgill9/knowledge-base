## Executive summary

Rust’s “error-handling crate ecosystem” has converged into a small set of dominant, complementary layers rather than a single “best crate”: (a) **typed error definition** for library APIs (notably `thiserror`, `snafu`, and increasingly `derive_more::Error`), (b) **type-erased reporting** at application boundaries (`anyhow` or `eyre`), (c) **diagnostic rendering** for user-facing, source-spanned errors (`miette`, plus alternatives like `ariadne`/`codespan-reporting`), and (d) **observability context** for instrumented async systems (`tracing-error`, plus `color-eyre`’s `SpanTrace` integration). A widely cited ecosystem overview explicitly recommends `thiserror`, `anyhow`, `snafu`, and `error-stack` as the “good crates,” and separately calls out `eyre` plus dedicated error-reporting crates like `miette`, `ariadne`, and `codespan`. It also marks `error-chain`, `failure`, `fehler`, and `err-derive` as historic/unrecommended today. citeturn16view0

Two strategic divides dominate decisions:

**Library vs application boundary**. `thiserror` emphasizes *not leaking into public API surface*—it generates the same `std::error::Error` impls you’d write by hand, so switching between handwritten impls and `thiserror` is not a breaking change. citeturn23search13 By contrast, `anyhow`’s README frames itself as “easy” when you *don’t care what error type your functions return*—typical for binaries—and advises `thiserror` when you do care (typical for libraries). citeturn47search1

**“String context” vs “structured context.”** `anyhow`/`eyre` are optimized for fast propagation with added context strings/macros but inevitably rely on dynamic erasure; `snafu` and `error-stack` push toward **structured, semantically named context**, which scales better in large systems at the cost of more upfront modeling and some boilerplate. `error-stack`’s own docs call out that it “adds some development overhead” by design, aiming to reduce debugging/observability overhead later. citeturn18search1turn21search7

Pragmatically, most production stacks combine crates rather than selecting only one:

- **Typical binary**: `anyhow` (or `eyre`) at the top-level + `thiserror`/`snafu` for internal modules/libraries; optionally `color-eyre` or `miette` for nicer output. citeturn47search1turn23search13turn42search2turn47search2  
- **Typed, context-rich systems**: `snafu` or `error-stack` where you want “semantic traces” and structured attachments; optionally integrate `tracing-error` to make async span context visible even when backtraces are noisy. citeturn33search21turn18search10turn43search6  
- **Legacy avoidance**: `failure` is explicitly deprecated and RustSec classifies it as end-of-life/unmaintained; `error-chain` is “no longer maintained” per Rust community announcements. citeturn44search0turn45search3turn46search2

## Scope, assumptions, and evaluation framework

This report scopes “error-handling crates” to: (1) typed error definition/derivation, (2) report/aggregation layers (type-erased or stacked), and (3) diagnostic rendering/reporting for user-facing errors. It includes the crates explicitly named in the prompt (including `snafu`) and extends coverage to additional ecosystem crates called out by authoritative ecosystem references and primary docs. citeturn16view0turn42search6turn42search4turn43search0

Assumptions (explicitly requested): Rust edition, MSRV, target platforms, and runtime environment (CLI vs server vs embedded) are **not specified**. Therefore, the analysis treats `std` vs `no_std`, allocations (`alloc`), and async/tracing integration as first-class dimensions rather than fixed constraints. citeturn31search3turn19search13turn39search1turn43search6

Evaluation dimensions (how comparisons are made):

- **Abstraction level and intended audience**: library API errors vs application reporting. citeturn47search1turn23search13turn16view0  
- **Ergonomics and boilerplate**: derive/macros, attribute expressiveness, and “how much ceremony” is needed for context. citeturn23search13turn50search2turn18search2turn47search2  
- **Composition/chaining and context attachment**: `source` chains, user-defined context frames, attachments, and semantic traces. citeturn47search1turn18search10turn19search1turn43search6  
- **Backtrace/span-trace support and cost model**: env-gated backtraces, stable vs nightly, and alternatives like `SpanTrace`. citeturn47search1turn21search8turn42search2turn43search6  
- **`std`/`no_std` support**: availability in embedded/wasm/no-stdlib contexts (often via `alloc`). citeturn31search3turn19search2turn19search13turn39search1  
- **Interoperability**: “just implements `Error`” vs “introduces a new report type,” and toolchain integration (e.g., `miette` deriving `Diagnostic`). citeturn23search13turn18search10turn47search2  
- **Maintenance & adoption**: release activity, explicit archive/deprecation status, and popularity proxies (GitHub stars, plus downloads where available). citeturn34search3turn21search5turn22search3turn47search2turn45search3turn46search2

## Actively maintained crates: feature-by-feature analysis

The ecosystem reference point (“Error Handling in Rust”) groups modern, commonly recommended crates into: `thiserror` (derive for typed errors), `anyhow` (type-erased error), `snafu` (derive + structured context), `error-stack` (stacked contexts/attachments), plus error-reporting crates like `eyre` and `miette`/`ariadne`/`codespan`. citeturn16view0

### Comparison table of key attributes

The table below summarizes the most commonly used and/or actively updated crates, plus “supporting” crates that frequently appear in real-world stacks. (Where a crate is primarily a helper/derive subcrate—e.g., `thiserror-impl`, `miette-derive`, `snafu-derive`—it is noted as such; typically you depend on the parent crate instead.) citeturn20search0turn20search11turn22search7turn50search2

| Crate | Primary role | Best fit | Context model | Backtrace / trace | `no_std` | Interop posture | Status notes | License |
|---|---|---|---|---|---|---|---|---|
| `thiserror` | derive typed errors | library APIs | `#[source]`, `#[from]`, `#[error(...)]` messages | optional backtrace attribute requires nightly (for `#[backtrace]`) | generally “std Error derive”; ecosystem uses forks for strict `no_std` | outputs normal `std::error::Error` impl; “does not appear in public API” | very active releases | MIT/Apache-2.0 citeturn23search13turn21search8turn34search3turn22search4 |
| `snafu` | typed errors + structured context selectors + reporting | libraries and large apps | generated “context selector” types (`.context(...)`, `.with_context(...)`) | has `report` support, can show chain + backtrace “if applicable” | explicit `alloc`/`std` feature split; many feature flags | errors remain typed and implement standard traits; encourages semantic layering | active (e.g., 0.9.0 in 2026-03) | MIT/Apache-2.0 citeturn22search3turn19search1turn39search8turn39search1turn50search1 |
| `anyhow` | type-erased application error (`anyhow::Error`) | binaries, CLI/tools, services (top layer) | string context (`Context`, `with_context`), `bail!`, `anyhow!` | captures/prints backtrace when enabled via std env vars; guarantees backtrace availability | supports `no_std` with default `std` feature off; allocator required | works with any error implementing `std::error::Error`; recommends `thiserror` for typed defs | very active | MIT/Apache-2.0 citeturn47search1turn31search3turn22search13 |
| `eyre` | type-erased application report (`eyre::Report`) w/ custom handlers | binaries needing customizable reporting | similar to `anyhow`, but handler-swappable | supports custom handlers; `simple-eyre` avoids capturing extra info | (not positioned as `no_std` in primary docs; treat as `std`-oriented) | warns against re-exporting in library public APIs | maintained; core repo updated 2025 | MIT/Apache-2.0 citeturn22search6turn22search2turn47search4turn21search10 |
| `color-eyre` | prettier `eyre` reports + panic hook | CLI apps, dev tooling | extends `eyre::Report` formatting + “sections” | captures `SpanTrace`/`Backtrace` (backtrace env-gated); notes SpanTrace cheaper | (primarily `std`) | integrates `tracing_error` and backtrace-formatting deps | repo is archived in org listing; still widely used | MIT/Apache-2.0 citeturn42search2turn25search5turn47search0turn20search2 |
| `miette` | diagnostics (`Diagnostic`) + reports | compilers, parsers, CLI UX | structured diagnostics: labels, help text, codes, source spans | provides report handler; “fancy” output feature pulls extra deps | (treated as `std`-centric; core trait depends on std Error) | can replace `anyhow`/`eyre`-like `Result`/`Report`; derive macro for metadata | active releases | Apache-2.0 citeturn47search2turn39search9turn20search3 |
| `error-stack` | stacked `Report` w/ contexts + attachments | large systems; observability | explicit frame stack of Context + Attachments | supports backtraces and rich formatting; community notes possible bloat | indicated as `no-std` capable in crate metadata | offers compatibility conversions; adds “development overhead” deliberately | maintained by HASH; very active | MIT/Apache-2.0 citeturn21search7turn18search10turn18search2turn18search1turn21search11turn18search15 |
| `tracing-error` | enrich errors with tracing span context | async services, instrumented systems | `SpanTrace`, `TracedError`, `in_current_span()` | SpanTrace is “relative of `Backtrace`” and can be *more useful in async* where stack traces show executor frames | (not primarily `no_std`) | designed to interop with `dyn Error` and tracing subscribers | explicitly “experimental” | MIT citeturn43search1turn43search6 |
| `displaydoc` | derive `Display` from doc comments | helper used with `thiserror`/others | doc-comment display templates | n/a | explicitly says it is `no_std` compatible (implements `core::fmt::Display`) | pure formatting helper | maintained | (license in crate docs; commonly dual) citeturn43search7 |
| `derive_more::Error` | derive many traits incl. `Error` | helper (esp. newtypes / enums) | derive attributes for source/backtrace detection | derive supports `no_std` except `provide()` due to `Backtrace` in std | explicitly discusses `no_std` behavior | outputs standard trait impls | active major releases | (see crate metadata) citeturn19search2turn48search1turn19search8 |
| `custom_error` | macro to define error types | small libs/apps; no-proc-macro preference | macro-based error definitions | n/a | docs.rs marks `nostd` feature (no-std support) | outputs standard trait impls | low activity (latest 2021; repo last commit ~4y) | BSD-2-Clause citeturn19search13turn19search9turn19search3 |
| `ariadne` | pretty diagnostics reporting | compilers/parsers; CLI UX | structured report builder | n/a | (typically `std`) | can complement typed errors | actively built | (see crate metadata) citeturn42search6 |
| `codespan-reporting` | diagnostic reporting | compilers/parsers; CLI UX | `Diagnostic` + labels + emit | n/a | (typically `std`) | alternative to `miette`; lists other alternatives | actively released (0.13.1 in 2025-10) | (see crate metadata) citeturn42search3turn42search1 |

### Detailed analysis of the major “core” crates

**`thiserror`: minimal abstraction, maximal interoperability (typed errors for libraries).**  
`thiserror`’s design goal is low conceptual overhead: you write normal enums/structs and annotate them; the generated code is equivalent to handwritten implementations of `std::error::Error` and `Display`, and the crate “deliberately does not appear in your public API.” citeturn23search13turn31search4 This makes it particularly attractive for libraries because the choice of using `thiserror` is not forced on downstream consumers and is not a semver hazard. citeturn23search13 The tradeoff is that “context” is typically string-based or encoded in your error variants; there is no built-in concept of stacked frames/attachments beyond the standard error source chain (`source()`). citeturn23search13turn47search1

Backtrace support in `thiserror` exists but has an important nuance: the crate’s own documentation indicates the `#[backtrace]` attribute (for capturing/providing backtraces) requires a nightly compiler of a specified minimum version. citeturn21search8 In practice, teams frequently pair `thiserror` with a top-level reporter (`anyhow`, `eyre`, `miette`) that can capture backtraces in stable contexts. citeturn47search1turn22search2turn47search2

**`snafu`: “context selectors” encourage semantic traces (typed + structured context).**  
Snafu presents itself as a library to generate errors and add information to underlying errors. citeturn22search11turn25search17 Its signature abstraction is the generated *context selector* per error variant; these selectors make it ergonomic to attach structured context at the point an underlying error is mapped into a higher-level domain error. citeturn33search13turn50search2 This pushes codebases toward multiple small, meaningful error types rather than one monolithic enum—an advantage highlighted in community discussion: context selectors can act like a “semantic backtrace” (what the program was attempting at each layer), improving debuggability in large libraries. citeturn17search6

Snafu also provides a user-friendly reporting mechanism: it can print the main error and its chain, plus a backtrace “if applicable.” citeturn19search1 Recent releases show ongoing evolution and attention to interoperability and thread-safety—for example, Snafu’s 0.9.0 changelog (2026-03-02) includes changes like making `Whatever` implement `Send`/`Sync` (with implications for wrapped errors) and replacing `snafu::Location` with a type alias to Rust’s standard `Location` reference for better interoperability. citeturn39search8

On portability, Snafu’s feature matrix is explicit: docs.rs lists `std`, `alloc`, and other toggles; notably, `alloc` is a default feature but “does not enable additional features” by itself (it’s the allocator availability gate), and there are features for backtrace behavior and futures integration. citeturn39search1turn50search1 That flexibility makes Snafu one of the more realistic options for constrained environments, compared to UI-heavy diagnostic crates. citeturn39search1turn16view0

**`anyhow`: fastest path to “good enough” application errors (type erasure + context strings).**  
Anyhow’s README is unusually explicit about its positioning: it provides a concrete error type, used via `anyhow::Result<T>`, optimized for application code; it works with any type implementing `std::error::Error`, and it provides context attachment helpers (`Context`, `with_context`) and convenience macros like `anyhow!` and `bail!`. citeturn47search1turn31search3 The same README includes an explicit “Comparison to thiserror” that matches the common rule: `anyhow` for when you don’t care about the returned error type, `thiserror` for when you do. citeturn47search1

Backtraces are supported on stable Rust (with version-specific nuances) and are environment-variable gated (`RUST_BACKTRACE`, `RUST_LIB_BACKTRACE`) as described in the Anyhow docs. citeturn21search1turn47search1 Anyhow also supports `no_std` with the default `"std"` feature disabled, but it requires a global allocator, and older toolchains may require explicit `map_err(Error::msg)` in some cases because `?`-based conversions historically relied on `std::error::Error`. citeturn31search3turn47search1turn22search13 This is a meaningful portability advantage over many “application UX” crates, but it still implies heap usage and dynamic dispatch in the general case. citeturn47search1turn19search5

**`eyre` and `color-eyre`: customization and presentation (type-erased, handler-driven reporting).**  
`eyre` is a fork of `anyhow` with support for customized error reports; its docs emphasize a swap-able `Handler` and list companion crates such as `stable-eyre` (backtraces on stable) and `simple-eyre` (minimal handler capturing no additional info). citeturn22search6turn22search2turn22search2 Its documentation also explicitly advises against re-exporting `eyre` types in library public APIs, to avoid version coupling and semver hazards. citeturn21search10turn22search2

`color-eyre` is an `eyre` report handler plus panic hook installation that produces consistent, “colorful” reports; it documents how to disable tracing integration to reduce dependencies, and it distinguishes SpanTrace capture from Backtrace capture, stating that SpanTrace capture is significantly cheaper and is enabled by default. citeturn42search2 It also discusses performance in debug builds: `eyre` uses `std::backtrace::Backtrace` (precompiled with optimizations), while `color-eyre` depends on `backtrace::Backtrace`, whose debug build can be an order of magnitude slower to capture; it provides guidance for optimizing the `backtrace` crate in dev profiles. citeturn42search2 Finally, `color-eyre`’s docs clarify that pretty-printing comes largely from dependencies like `color-backtrace` and `color-spantrace`, which offer additional customization. citeturn25search5turn25search12

Maintenance nuance: the `eyre-rs` organization repo listing marks `color-eyre` as an archived repository (while still widely used). citeturn47search0

**`miette`: diagnostic-first errors (structured metadata + rich formatting).**  
Miette is positioned as an extension of `std::error::Error` for “pretty, detailed diagnostic printing” and encourages rich diagnostic metadata via its `Diagnostic` protocol and derive macro. citeturn47search2turn39search9 Its README includes an important ecosystem guideline: enable the `"fancy"` feature only in the top-level crate because it pulls in additional dependencies that libraries may not want. citeturn47search2 This guidance is a clear signal that `miette` is best seen as a **presentation/UX layer**, not as the fundamental definition of every library error type. Miette’s design also explicitly includes replacements for `anyhow`/`eyre`-style `Result`/`Report` and a `miette!` macro analogous to `anyhow!`. citeturn47search2turn39search9

**`error-stack`: explicit layered “Report” with typed contexts and attachments.**  
`error-stack` centers on constructing a `Report` as errors propagate; a `Report` contains a frame stack of `Context`s and attachments. citeturn21search7turn18search10 The extension trait `ResultExt` makes it ergonomic to attach printable or opaque payloads and to change context as the error crosses boundaries. citeturn18search2turn18search13 This is more than a `source()` chain: you can attach arbitrary user data (opaque) and additional printable messages, and later render or query them. citeturn18search10turn21search15

The crate is maintained by HASH (noted in docs) and is under MIT/Apache dual licensing. citeturn21search11turn18search17 It also explicitly acknowledges tradeoffs: “This crate adds some development overhead” and, notably, does **not** allow string-like root errors (promoting typed contexts). citeturn18search1 Community feedback highlights a potential downside: depending on configuration, `error-stack`’s backtrace dependency can contribute to code size (“code bloat”) in some client applications. citeturn18search15

## Legacy and niche crates: status and migration notes

This section focuses on crates that appear in older codebases, tutorials, or dependency graphs, and how they relate to modern choices.

**`failure` (legacy; deprecated/unmaintained).**  
The docs.rs page for `failure` includes an explicit deprecation notice recommending `anyhow` as a replacement for `failure::Error` and `thiserror` as a near drop-in replacement for `#[derive(Fail)]`. citeturn44search0 RustSec lists `failure` as “officially deprecated/unmaintained” and explicitly says there will be no updates/maintenance going forward, while suggesting alternatives including `anyhow`, `eyre`, `fehler`, `snafu`, and `thiserror`. citeturn45search3turn46search0  
Migration path: `failure::Error` → `anyhow::Error`/`eyre::Report`; custom `Fail` derive → `thiserror::Error` derive (or `snafu` for structured context). citeturn44search0turn47search1turn22search6

**`error-chain` (legacy; no longer maintained).**  
A Rust community announcement states plainly: “Error-chain is no longer maintained,” advising new projects not to use it and noting it was moved toward `rust-lang-deprecated`. citeturn46search2turn44search2 The crate’s docs show the last published version 0.12.4 in 2020 and link to `rust-lang-deprecated/error-chain`. citeturn44search2turn46search5  
Migration path: move to `thiserror`/`snafu` for typed errors + `anyhow`/`eyre` at app boundaries, depending on whether you want typed returns or erased reports. citeturn16view0turn47search1turn22search3

**`quick-error` (older macro-based typed errors; low-churn).**  
`quick-error` remains available and documents a macro for defining error enums with concise `Display`/`Error`/`From` generation. citeturn44search5turn49search2 However, its release history shows the most recent stable release in 2021 (2.0.1), and modern codebases typically prefer derive-driven solutions (`thiserror`, `snafu`, `derive_more`) for clearer syntax, IDE support, and ecosystem alignment. citeturn49search2turn23search13turn50search2turn19search2  
Migration path: macro-defined enums → `#[derive(Error)]` in `thiserror` or `derive_more`, or `#[derive(Snafu)]` if you want structured context selectors. citeturn23search13turn19search2turn50search2

**`fehler` (experiment; generally treated as historic).**  
The ecosystem overview lists `fehler` as “an experiment which is no longer maintained.” citeturn16view0 Its docs describe the `#[throws]` attribute, `throw!` macro, and even include a TODO mentioning async blocks and broader `Try` support. citeturn45search0  
Recommendation: avoid introducing `fehler` into new long-lived code unless you have strong reasons and accept the macro/maintenance risk; prefer explicit `Result` return types plus modern error crates. citeturn16view0turn47search1turn22search3

**`err-derive` (historic; mostly superseded).**  
`err-derive` positions itself as a “failure-like derive macro” for `std::error::Error`, derived from code copied from `failure-derive`, and it supports `no_std` by disabling the default `std` feature (deriving only `From`/`Display` in that mode). citeturn45search2 The ecosystem overview explicitly lists `err-derive` as historic because its support is built into the recommended crates. citeturn16view0  
Migration path: `err-derive` → `thiserror` or `derive_more::Error` (or `snafu` where structured context is valuable). citeturn16view0turn23search13turn19search2

**Niche/derivative crates worth knowing about.**  
Some crates build directly atop the “core” ecosystem for specialized needs; for example, `snafu-virtstack` describes combining SNAFU-style error handling with “virtual stack traces” to produce meaningful context without system-backtrace overhead, and it includes a “Performance” section in its README. citeturn39search5 These can be valuable in performance- or size-sensitive environments but are typically less standardized than the mainstream crates.

## Decision matrix: recommended crates by use case

Rather than selecting a single crate for an entire organization, it is generally more robust to standardize a **two-layer policy**: (1) how libraries define errors, and (2) how applications report them. This mirrors the ecosystem’s “good crates” split: `thiserror`/`snafu`/`error-stack` for defining and structuring errors, and `anyhow`/`eyre`/`miette` for application reporting. citeturn16view0turn47search1turn22search6turn47search2

### Matrix

| Use case | Recommended primary choice | Why | “Avoid / be cautious” |
|---|---|---|---|
| CLI apps (human-facing, wants nice output) | `anyhow` + (`color-eyre` *or* `miette`) | `anyhow` is optimized for “don’t care about the error type” application code and easy context; `color-eyre` installs report + panic hooks and provides multi-verbosity output; `miette` provides structured diagnostics and flashy UX | `color-eyre` repo is archived; treat as stable but watch maintenance; `miette`’s “fancy” feature pulls deps—enable only at top-level citeturn47search1turn42search2turn47search2turn47search0 |
| Libraries (public API stability) | `thiserror` *or* `snafu` | `thiserror` stays out of public API and generates standard impls; `snafu` adds structured context selectors that scale well with many boundary crossings | returning `anyhow::Error`/`eyre::Report` in public APIs (type-erased coupling); `eyre` explicitly warns against re-exporting in library APIs citeturn23search13turn22search11turn21search10 |
| Production services (observability, debugging) | `snafu` or `error-stack` (structured layering) + `tracing-error` | `SpanTrace` can be more informative than backtraces in async systems (executor-frame noise); `error-stack` attachments support rich payloads; `snafu` semantic contexts reduce “what step failed?” ambiguity | unstructured “single giant error enum” that becomes a dump; relying solely on string context when you need structured telemetry citeturn43search6turn18search10turn17search6turn19search1 |
| Low-allocation / `no_std` / embedded | `snafu` (with `alloc`/feature tuning), `derive_more::Error`, small hand-rolled enums; optionally `custom_error` | Snafu and `derive_more::Error` explicitly discuss `no_std` behavior; `custom_error` documents a `nostd` feature; simplest is often “define an enum and implement traits” | `miette`/`color-eyre` (UX-focused, heavier deps), and type-erased report crates if heap/dyn dispatch are unacceptable citeturn39search1turn19search2turn19search13turn47search2turn42search2 |
| Prototyping / internal tooling | `anyhow` (or `eyre`) | maximum speed of development; macros and context help quickly pinpoint failures | using legacy crates (`failure`, `error-chain`) adds long-term migration burden citeturn47search1turn22search2turn44search0turn46search2 |
| Parser/compiler-like UX (source spans, labels) | `miette` or `ariadne`/`codespan-reporting` | built for “show the user where and why”; `codespan-reporting` explicitly positions itself as making compiler-style diagnostics easier | mixing “fancy output” dependencies into libraries; follow guidance to keep heavy rendering in top-level crates citeturn47search2turn42search6turn42search3 |

### Recommended organizational patterns

A simple and scalable policy is:

- **Library crates**: define domain errors with `thiserror` (minimum ceremony) or `snafu` (structured context selectors), and keep error types stable and intentional. citeturn23search13turn22search11turn17search6  
- **Binary/application crates**: convert to `anyhow::Result` (or `eyre::Result`) at the boundary and add human-readable context. If you need compiler-grade UX, use `miette` (optionally with `thiserror` errors that also derive `Diagnostic`). citeturn47search1turn22search2turn47search2  
- **Observability-first services**: prefer `tracing-error` + SpanTraces for async context, and consider `error-stack` attachments when you want to carry opaque, structured payloads through the error path. citeturn43search6turn18search10turn18search2

## Idiomatic usage patterns and examples

### Popularity chart

The chart below uses GitHub stars (a rough popularity proxy) for major crates where primary GitHub pages plainly list stars. Data points are approximate (rounded “k” values) and reflect the crawl-time snapshots of the cited sources. citeturn47search1turn47search3turn47search2turn47search4turn47search0  

![Popularity proxy: GitHub stars for major Rust error-handling crates](sandbox:/mnt/data/rust_error_crates_github_stars.png)

### Mermaid diagram of a recommended layered architecture

```mermaid
flowchart TB
  subgraph L[Library crates]
    A[thiserror or snafu\nDomainError enums/structs] --> B[std::error::Error + source() chain]
  end

  subgraph S[Service / app core]
    B --> C[Boundary conversion\n(anyhow::Error or eyre::Report)]
    C --> D[Add context strings\nContext / WrapErr]
  end

  subgraph UX[User-facing reporting]
    D --> E1[color-eyre\n(pretty reports + SpanTrace)]
    D --> E2[miette\n(Diagnostic + rich rendering)]
  end

  subgraph OBS[Observability]
    C --> F[tracing-error\nSpanTrace / TracedError]
    F --> E1
  end
```

This diagram reflects the ecosystem’s explicit separation of concerns between typed error definition (`thiserror`, `snafu`), application-level reporting (`anyhow`, `eyre`), and diagnostic presentation (`miette`, `color-eyre`) plus tracing integration (`tracing-error`). citeturn16view0turn47search1turn22search2turn47search2turn43search6

### Idiomatic code examples for representative crates

The snippets below are schematic, focusing on idioms rather than exhaustive edge cases.

#### `thiserror` for library error types

```rust
use thiserror::Error;
use std::io;

#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("failed to read config file: {path}")]
    Read { path: String, #[source] source: io::Error },

    #[error("invalid config: {0}")]
    Parse(String),
}
```

Why this is idiomatic: `thiserror` is designed to generate the same `Error` and `Display` impls you would write by hand, without surfacing the crate in public API types beyond the fact that you implement standard traits. citeturn23search13turn31search4

#### `anyhow` for application boundaries and rapid context

```rust
use anyhow::{Context, Result};

fn load_config(path: &str) -> Result<String> {
    std::fs::read_to_string(path)
        .with_context(|| format!("reading config from {path}"))
}

fn main() -> Result<()> {
    let cfg = load_config("app.toml")?;
    println!("{cfg}");
    Ok(())
}
```

Why this is idiomatic: Anyhow emphasizes `anyhow::Result<T>` + context helpers and documents environment-variable controlled backtraces (`RUST_BACKTRACE`, `RUST_LIB_BACKTRACE`). citeturn47search1turn21search1

#### `snafu` with context selectors

```rust
use snafu::prelude::*;
use std::{fs, io};

#[derive(Debug, Snafu)]
pub enum Error {
    #[snafu(display("unable to read config file {path}"))]
    ReadConfig { path: String, source: io::Error },

    #[snafu(display("config was invalid"))]
    InvalidConfig,
}

pub fn load_config(path: &str) -> Result<String, Error> {
    fs::read_to_string(path).context(ReadConfigSnafu { path: path.to_string() })
}
```

Why this is idiomatic: the `Snafu` derive macro is described as the “entrypoint” and generates context selectors; Snafu also provides reporting facilities that print chains and backtraces when applicable. citeturn35search12turn50search2turn19search1

#### `error-stack` for stacked contexts and attachments

```rust
use error_stack::{Report, ResultExt};
use std::io;

#[derive(Debug)]
struct ConfigCtx;
impl error_stack::Context for ConfigCtx {}

fn load_config(path: &str) -> Result<String, Report<ConfigCtx>> {
    std::fs::read_to_string(path)
        .map_err(Report::from)
        .attach_printable(format!("path={path}"))
        .change_context(ConfigCtx)
}
```

Why this is idiomatic: `error-stack` centers on building a `Report` with a frame stack of contexts and attachments, and `ResultExt` provides ergonomic methods like `attach_*` and `change_context`. citeturn21search7turn18search10turn18search2

#### `miette` for user-facing diagnostics

```rust
use miette::{Diagnostic, NamedSource, Result, SourceSpan};
use thiserror::Error;

#[derive(Debug, Error, Diagnostic)]
#[error("parse error")]
pub struct ParseError {
    #[source_code]
    src: NamedSource,

    #[label("here")]
    span: SourceSpan,

    #[help]
    help: Option<String>,
}

fn main() -> Result<()> {
    // ... create ParseError with a span into src ...
    Ok(())
}
```

Why this is idiomatic: miette’s core value is `Diagnostic` metadata (labels, help, codes) plus report handlers; it explicitly warns to enable the `"fancy"` output feature only in top-level crates due to added dependencies. citeturn47search2turn39search9