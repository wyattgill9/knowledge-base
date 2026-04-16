
These two documents converge strongly on the same core recommendations while offering complementary depth on different aspects. Let me walk you through the synthesized picture, starting from the high-level consensus and working down into the nuances where it gets interesting.

## The Stack That Has Won

Both documents agree that the Rust-on-NixOS ecosystem has consolidated around a clear default stack, and if you're starting a new project in 2026, this is what you should reach for:

**rust-overlay** for toolchain management, **crane** for Nix builds and CI checks, **flake-parts** for flake architecture, and **nix-direnv** for shell entry. Layer on a binary cache (Cachix for managed hosting, Attic for self-hosted), and you have a setup that scales from solo hacking to a mid-sized team without rearchitecting.

The reason this stack won is that each piece solves a specific problem cleanly without stepping on the others. rust-overlay gives you pure evaluation with pre-fetched hashes for every toolchain component, so you never deal with IFD just to get your compiler. crane splits your build into two derivations — dependencies and your source — so CI caches are effective and checks like clippy, formatting, and tests all share the same pre-built dependency artifacts. flake-parts applies the NixOS module system to your flake, which means you can split configuration into focused files instead of growing one enormous `flake.nix`. And nix-direnv caches your shell environment and pins it with GC roots, so you don't lose your build environment to garbage collection (a genuinely painful experience, especially offline).

## The Core Mental Model: Two Tiers of Building

Perhaps the most important insight both documents emphasize is the **two-tier development model** that the ecosystem has settled on. This is worth internalizing because it resolves a tension that trips up many newcomers.

**Tier A is your fast development loop.** You enter your Nix-provided devShell (via `nix develop` or direnv), and then you use `cargo build`, `cargo check`, and `cargo test` directly. This gives you Cargo's incremental compilation, a warm `target/` directory, and responsive LSP. Nix's job here is only to provide the *environment* — the right toolchain, the right native libraries, the right environment variables.

**Tier B is your hermetic, reproducible build.** This is `nix build` and `nix flake check`, which run inside Nix's sandbox. These are what CI runs, what produces your release artifacts, and what guarantees that your build is reproducible. crane makes this layer cache-efficient by separating dependency compilation from source compilation.

The critical lesson both documents stress: **do not try to collapse Tier A into Tier B.** Nix derivations don't persist a `target/` directory between invocations — that's by design, not a bug. Forcing every edit through `nix build` burns Cargo's main advantage (incrementality) without improving your CI contract beyond what crane already provides.

## Toolchain Pinning: How It Actually Works

The state-of-the-art pattern is to make `rust-toolchain.toml` your single source of truth for the Rust toolchain. This file is understood by Cargo, rust-analyzer, and the broader Rust ecosystem. Then, on the Nix side, rust-overlay consumes it directly:

```nix
rustToolchain = pkgs.rust-bin.fromRustupToolchainFile ./rust-toolchain.toml;
```

This gives you the best of both worlds: Rust-native pinning semantics (so non-Nix developers can also use the same file with rustup), plus Nix purity and reproducibility (the toolchain comes from the Nix store, not a mutable `~/.rustup` directory).

One practical subtlety both documents flag: if you use nightly toolchains, **don't use `rust-bin.nightly.latest`** because components can be missing on some days. Instead, use `rust-bin.selectLatestNightlyWith`, which finds the most recent nightly that actually has all the components you need.

**fenix** remains a credible alternative, and it has one genuine advantage: it provides standalone nightly rust-analyzer builds and a nightly VS Code extension, which rust-overlay doesn't offer separately from the toolchain. fenix's repo is also dramatically smaller (836 KiB vs. rust-overlay's 23 MiB). The tradeoff is that fenix sometimes requires you to provide SHA256 hashes for manifest files in pure evaluation mode, which adds operational friction. The first document notes that devenv itself switched from fenix to rust-overlay in 2025, citing better maintenance — a strong signal.

**nixpkgs-mozilla is dead.** Both documents are emphatic about this. If you encounter it in tutorials, those tutorials are outdated.

## rust-analyzer: Making It Work Cleanly

On NixOS, rust-analyzer problems almost always collapse to one issue: it can't find the standard library sources. The fix is straightforward — include `rust-src` in your toolchain components (in `rust-toolchain.toml`) and, if needed, export `RUST_SRC_PATH` in your devShell pointing to the toolchain's library source directory. For VS Code specifically, setting `"rust-analyzer.server.path": "rust-analyzer"` ensures the extension uses the Nix-provided binary rather than downloading its own.

## The Build Framework Landscape

Both documents agree that **crane** is the clear default, but they offer useful complementary analysis on when you might want something else.

crane's two-derivation model (one for dependencies, one for your source) is the sweet spot for most projects. It requires no IFD, no code generation, and it parses `Cargo.lock` directly in pure Nix. The composability story is what makes it dominant: clippy, rustfmt, cargo-nextest, cargo-audit, and cargo-deny can all be separate check derivations that share the same pre-built dependency artifacts. In CI, this means dependencies build once, get cached, and every subsequent check skips that work entirely.

The limitation is granularity. Any change to `Cargo.lock` rebuilds all dependencies as a single unit. For most projects, this is fine. But if you have a massive dependency tree and you're frequently adding or updating individual crates, rebuilding everything can hurt.

This is where **crate2nix** becomes relevant. It generates one derivation per crate, offering the most granular caching possible. The second document highlights that crate2nix experienced a genuine resurgence in 2025-2026: devenv chose it for its `languages.rust.import` feature, and the 0.15.0 release in January 2026 added private registries support and cross-compilation fixes. The first document adds nuance: crane's author has argued that per-crate benefits are overstated because different feature flag combinations prevent deduplication across projects anyway. The practical synthesis is that crane works for most projects, and you should consider crate2nix only when your dependency tree is large enough that the single-unit rebuild becomes a material cost in time or money.

**naersk** pioneered the two-derivation approach that crane refined, but it lacks crane's composability for checks. Both documents recommend crane over naersk for new projects. **buildRustPackage** from nixpkgs is a monolithic single-derivation builder — appropriate for contributing packages to nixpkgs itself, but painful for active development because the `cargoHash` must be manually updated after every `Cargo.lock` change. **dream2nix** explicitly labels its Rust modules as experimental, and both documents recommend avoiding it for production Rust work.

## Developer Speed: The Practical Knobs

Both documents converge on the same set of tools for making the inner development loop fast. **mold** as a linker is dramatically faster than the default for edit-rebuild cycles. **sccache** as a compiler cache avoids redundant compilation. **cargo-nextest** as a test runner is up to 3× faster than `cargo test` with per-test isolation. And **cargo-llvm-cov** provides modern source-based code coverage.

The devShell wires these in via environment variables (`RUSTC_WRAPPER=sccache`, `RUSTFLAGS="-C link-arg=-fuse-ld=mold"`), keeping the configuration declarative and reproducible.

One NixOS-specific pain point the second document highlights with particular clarity: **dynamic linking and lld**. NixOS has no `/usr/lib`, so crates linking C libraries need `pkg-config` in `nativeBuildInputs` and the library in `buildInputs`. Using lld directly bypasses NixOS's ld wrapper and breaks RPATH, causing runtime linking failures. This matters more now because Rust 1.90+ ships its own lld by default. The workaround is using `pkgs.llvmPackages.bintools` (which includes the NixOS wrapper) instead of raw lld.

## Caching and CI: First-Class Infrastructure

Both documents treat binary caches as production infrastructure, not an afterthought. The strategy has two layers: crane's artifact splitting provides build composability (dependencies build once and are reused across all checks), and binary caches distribute those artifacts to CI runners and developer machines.

**Cachix** is the default managed service. **Attic** is the primary self-hosted alternative, backed by S3-compatible storage with chunk-level deduplication — it describes itself as an early prototype but is actively used. The first document adds an important operational detail: too many substituters can degrade performance because Nix queries each one for every build. The recommended pattern is to enable only the official cache globally and selectively add project-specific caches via flake `nixConfig`.

For CI, the pattern is simple: the same `nix build` and `nix flake check` commands run locally and in CI. The flake's `nixConfig` block can encode substituters so everyone — local developers and CI runners — automatically uses the same caches.

## devenv: The Batteries-Included Alternative

Both documents recognize **devenv** as a legitimate, increasingly polished alternative to the raw flake-parts + crane stack. devenv provides `languages.rust.enable = true` and handles toolchain management (via rust-overlay), linker integration (mold, lld), cranelift backend support for faster dev builds, pre-commit hooks, process management, and service orchestration. Its `languages.rust.import` feature (August 2025) adds zero-config packaging via crate2nix internally, so developers don't have to choose a build framework at all.

The first document positions devenv as "best batteries-included alternative" — ideal when you want processes, services, and tasks as first-class Nix config, especially on teams where not everyone is a Nix expert. The tradeoff is that devenv moves fast enough that documentation can lag implementation, and you may occasionally need to read module source code rather than relying on rendered docs.

## What's Genuinely Unresolved

Both documents are honest about the ecosystem's remaining pain points. **Cross-compilation works for pure Rust but breaks down with C dependencies** — correctly splitting build inputs, ensuring pkg-config finds the right target libraries, and avoiding full-toolchain recompilation is expert-level work. **IFD remains a fundamental tension**: tools that want per-crate granularity need either IFD (disallowed in nixpkgs and Hydra), code generation, or precomputed hashes. And **the sheer number of build tools** (eight or more) creates decision paralysis for newcomers, even though convergence toward crane is happening.

## The Bottom Line

If you're starting a greenfield Rust project on NixOS in 2026, the first document's recommendation holds up well after synthesis with the second: **flake-parts + rust-overlay (pinned via `rust-toolchain.toml`) + crane + nix-direnv + binary cache**, with mold, sccache, and cargo-nextest for iteration speed. Use Cargo for your fast loop, Nix for your environment and CI contract, and resist the temptation to force every keystroke through a Nix derivation. If you want a higher-level abstraction that hides the framework choices, devenv is the mature option. And if you outgrow crane's two-derivation model in a large monorepo, crate2nix is the resurgent tool to evaluate.