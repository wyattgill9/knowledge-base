
**The serious NixOS Rust stack has converged around rust-overlay + crane + flake-parts + nix-direnv**, though meaningful debates persist on build granularity, flake architecture, and how much Nix to impose on the development loop. This guide synthesizes current practitioner experience, community consensus, and the real tradeoffs across every major tool in the ecosystem — from toolchain overlays to CI caching to frontier cross-compilation patterns.

---

## Toolchain overlays: rust-overlay has won the momentum war

Three options exist for managing Rust toolchains on NixOS, and the community has increasingly consolidated around one.

**oxalica/rust-overlay** (~1,400+ GitHub stars) is the current default choice. It works as a nixpkgs overlay adding a `rust-bin` attribute set, downloading pre-built binary toolchains from `static.rust-lang.org`. Its defining technical feature is that **all toolchain component hashes are pre-fetched and stored in-tree** — every stable version back to 1.29.0 and nightlies from the previous year. This makes evaluation fully pure with zero IFD. The tradeoff is repo size (~23 MiB), which adds latency to `nix flake update`. It reads `rust-toolchain.toml` natively via `rust-bin.fromRustupToolchainFile`, supports arbitrary version pinning by string (`rust-bin.stable."1.75.0".default`), and handles nightly component gaps elegantly with `selectLatestNightlyWith`. Cross-compilation has dedicated examples for aarch64, MinGW, and WASI targets.

The strongest signal of rust-overlay's dominance: **devenv.sh switched from fenix to rust-overlay in PR #1500** (mid-2025), citing superior maintenance. The bcachefs-tools project made the same switch in March 2025. oxalica, the sole maintainer (also author of nil, the Nix LSP), maintains an impressively active cadence — 270 active days in the past year with daily automated hash updates via GitHub Actions.

**nix-community/fenix** (~950+ stars) remains a credible alternative with genuine advantages. Its repo is just **836 KiB** versus rust-overlay's 23 MiB because it stores only latest manifests. Fenix's killer feature is providing **standalone nightly rust-analyzer builds** and a nightly VS Code extension — rust-overlay only offers the rust-analyzer component bundled with the toolchain. Fenix also ships on the nix-community Cachix cache for x86_64-linux, x86_64-darwin, and aarch64-darwin, giving it a potential speed edge for users without their own cache. The disadvantage is that pinning older versions requires workarounds (pinning the fenix flake revision or providing manifest SHAs), and some operations like `fromToolchainFile` with non-latest channels may require IFD.

**nixpkgs-native Rust** (just `pkgs.rustc` and `pkgs.cargo`) tracks only the latest stable. It works for simple use cases and is the only option for packages contributed to nixpkgs itself. No nightly, no version pinning, no `rust-toolchain.toml` support.

**nixpkgs-mozilla is dead.** If you see it referenced in tutorials, those tutorials are outdated. rust-overlay is its explicit drop-in replacement.

The practical recommendation: **use rust-overlay** unless you specifically need fenix's nightly rust-analyzer or its smaller flake input size. Pin your toolchain via `rust-toolchain.toml` for maximum portability between Nix and non-Nix developers:

```nix
rustToolchain = pkgs.rust-bin.fromRustupToolchainFile ./rust-toolchain.toml;
```

---

## Build frameworks: crane leads, crate2nix is resurgent

The build framework landscape has **eight or more options** — the single clearest sign of Nix-Rust ecosystem fragmentation. But a hierarchy has emerged.

**Crane** (~1,300 stars, v0.23.0 as of January 2026) is the community's preferred build framework for flake-based projects. Its core innovation is the **two-derivation caching model**: `buildDepsOnly` compiles all workspace dependencies using a dummy source (only `Cargo.toml` and `Cargo.lock` affect its hash), producing `cargoArtifacts` that can be shared across unlimited downstream derivations — `buildPackage`, `cargoClippy`, `cargoNextest`, `cargoTarpaulin`, and `cargoFmt`. This composability is crane's decisive advantage. In CI, dependencies build once and get pushed to cache; every subsequent check reuses them. Crane uses no IFD, no codegen, and parses `Cargo.lock` directly in pure Nix. It works seamlessly with both rust-overlay and fenix via `craneLib.overrideToolchain`, has dedicated cross-compilation templates, and maintains excellent documentation at crane.dev.

Crane's main limitation is **granularity**: any change to `Cargo.lock` rebuilds all dependencies as a single unit. Adding one small crate forces recompilation of your entire dependency graph. For projects with very large, slow-to-compile dependency trees, this can hurt.

**naersk** (~844 stars) pioneered the two-derivation approach that crane refined. It still works — no IFD, no codegen, minimal configuration. But it lacks crane's composability: running clippy, tests, and builds as separate cached derivations is difficult without duplicating dependency compilation. naersk's author nmattia and the community maintain it, but crane is widely viewed as its spiritual successor. **For new projects, choose crane over naersk.**

**buildRustPackage** is nixpkgs' built-in builder and the only option for packages contributed upstream. It uses a single monolithic derivation with a `cargoHash` (SRI hash) for vendored dependencies. No incremental caching — any source or dependency change triggers a full rebuild. The `cargoHash` must be manually updated after every `Cargo.lock` change (build with `lib.fakeHash`, fail, copy the correct hash). This is adequate for packaging releases but painful for active development. Note that nixpkgs 25.05 migrated from `fetchCargoTarball` to `fetchCargoVendor`, breaking some override patterns.

**crate2nix** is experiencing a resurgence. It generates **one derivation per crate**, offering the most granular caching possible. In August 2025, devenv's Domen Kožar chose crate2nix for the new `languages.rust.import` feature, stating that developers "don't want to compare crate2nix vs cargo2nix vs naersk vs crane — they want a tested solution that works." crate2nix now supports `--format json` for faster evaluation. The tradeoff: it requires either running `crate2nix generate` (committing `Cargo.nix`) or using IFD mode, and per-crate sharing across projects is limited in practice because different feature flag combinations prevent deduplication.

**dream2nix** lists all three of its Rust modules as "experimental." Multiple sources, including juspay's rust-flake documentation, explicitly warn against using it for production Rust. **cargo2nix** (~448 stars) is niche and less actively maintained, with known issues around Rust 2024 edition support.

| Framework | Derivations | IFD | Cache granularity | Best for |
|-----------|------------|-----|-------------------|----------|
| **crane** | 2 (deps + source) | No | Whole dep tree | Most projects, CI pipelines |
| **naersk** | 2 (deps + source) | No | Whole dep tree | Legacy projects already using it |
| **buildRustPackage** | 1 (monolithic) | No | None | nixpkgs contributions |
| **crate2nix** | Many (per-crate) | Optional | Per-crate | devenv users, huge dep trees |
| **cargo2nix** | Many (per-crate) | Yes (codegen) | Per-crate | Niche, declining |
| **dream2nix** | Varies | Varies | Varies | Avoid for Rust |

---

## Dev environment: the shell-entry workflow has matured

The standard 2025 dev environment pattern is **flakes + mkShell + nix-direnv**, with devenv.sh as a higher-level alternative.

**mkShell in a flake** remains the foundational approach. Define `devShells.default` in your flake, include your toolchain, native dependencies, and `RUST_SRC_PATH`, then enter via `nix develop` or direnv. The key best practice that has solidified: use `packages` (which sets `nativeBuildInputs`) rather than `buildInputs` for dev tools, and always include `rust-src` in your toolchain override so rust-analyzer finds the standard library without manual `RUST_SRC_PATH` configuration.

**nix-direnv is the universal shell-entry mechanism.** It replaces direnv's built-in `use_nix` with a faster, caching implementation that creates GC roots to prevent garbage collection. An `.envrc` with `use flake` is sufficient. Add `nix_direnv_watch_file rust-toolchain.toml` to trigger re-evaluation when the toolchain changes. lorri is effectively dead — no flake support, requires a daemon, and nix-direnv does everything better. The VS Code direnv extension picks up the Nix environment automatically, making rust-analyzer work without any editor-specific configuration.

**devenv.sh** provides a batteries-included alternative with `languages.rust.enable = true`. Since its 1.0 rewrite in Rust (March 2024), devenv offers automatic toolchain management (via rust-overlay), **mold/lld/wild linker integration**, cranelift backend support for faster dev builds, pre-commit hooks, process management via process-compose, and service orchestration (PostgreSQL, Redis). The new `languages.rust.import` feature (August 2025) adds zero-config packaging via crate2nix internally. devenv's philosophy is explicit: use cargo for local development, Nix for packaging and deployment. For teams where not everyone is a Nix expert, devenv dramatically reduces friction.

**numtide/devshell** occupies a niche between raw mkShell and devenv. It uses the NixOS module system, provides a categorized command menu on shell entry, supports TOML configuration, and has a flake-parts module. It's clean and well-designed but less widely adopted.

One critical NixOS-specific pain point deserves emphasis: **LD_LIBRARY_PATH and dynamic linking**. NixOS has no `/usr/lib`. Crates linking C libraries need `pkg-config` in `nativeBuildInputs` and the library in `buildInputs`. Using `lld` as the linker bypasses NixOS's ld wrapper and breaks RPATH, causing runtime linking failures. The workaround is `pkgs.llvmPackages.bintools` (which includes the wrapper) instead of raw `lld`. This matters more in 2025 because **Rust 1.90+ ships its own lld by default**.

---

## Flake architecture: flake-parts is becoming the standard

Raw flake outputs with `flake-utils.lib.eachDefaultSystem` still work, but the advanced community has moved toward **flake-parts**, which applies the NixOS module system to flakes. Instead of manually iterating over systems, you write `perSystem` modules that compose cleanly. The benefits compound with the ecosystem of flake-parts modules: treefmt-nix for formatting, devshell for shell management, and critically, **rust-flake** (by Juspay) which wraps crane into a declarative flake-parts module.

A concrete reference architecture for a serious Rust project using flake-parts + crane + rust-overlay + treefmt-nix:

```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    flake-parts.url = "github:hercules-ci/flake-parts";
    crane.url = "github:ipetkov/crane";
    rust-overlay = { url = "github:oxalica/rust-overlay"; inputs.nixpkgs.follows = "nixpkgs"; };
    treefmt-nix.url = "github:numtide/treefmt-nix";
  };
  outputs = inputs@{ flake-parts, ... }:
    flake-parts.lib.mkFlake { inherit inputs; } {
      systems = [ "x86_64-linux" "aarch64-linux" "aarch64-darwin" "x86_64-darwin" ];
      imports = [ inputs.treefmt-nix.flakeModule ];
      perSystem = { self', pkgs, system, ... }:
        let
          rustToolchain = pkgs.rust-bin.fromRustupToolchainFile ./rust-toolchain.toml;
          craneLib = (inputs.crane.mkLib pkgs).overrideToolchain rustToolchain;
          src = craneLib.cleanCargoSource ./.;
          cargoArtifacts = craneLib.buildDepsOnly { inherit src; };
        in {
          packages.default = craneLib.buildPackage { inherit cargoArtifacts src; };
          checks = {
            clippy = craneLib.cargoClippy {
              inherit cargoArtifacts src;
              cargoClippyExtraArgs = "-- --deny warnings";
            };
            nextest = craneLib.cargoNextest { inherit cargoArtifacts src; };
            fmt = craneLib.cargoFmt { inherit src; };
          };
          devShells.default = craneLib.devShell {
            checks = self'.checks;
            packages = with pkgs; [ cargo-watch rust-analyzer ];
          };
          treefmt.config = {
            projectRootFile = "flake.nix";
            programs.rustfmt.enable = true;
            programs.nixfmt-rfc-style.enable = true;
          };
        };
    };
}
```

This pattern gives you `nix build`, `nix flake check` (running clippy, tests, and fmt as parallel derivations sharing cached dependency artifacts), `nix fmt`, and `nix develop` — all from one composable file. The **srid/rust-nix-template** (242+ stars) and **akirak/flake-templates#rust** are well-maintained starting points using this approach.

For teams wanting even more abstraction, **juspay/rust-flake** reads `Cargo.toml`, auto-wires packages, checks, and devShells, and generates clippy checks and rustdoc outputs declaratively. It explicitly warns against nix-cargo-integration (which uses dream2nix) in favor of its direct crane integration.

---

## Caching and CI: the two-layer strategy

The practical CI caching strategy has two layers: **Nix binary caches for dependency derivations, and crane's artifact splitting for build composability**.

**Cachix** remains the default hosted binary cache (~12K developers, 521 TB transferred monthly). The free tier gives 5 GB for public caches. For CI, `cachix/cachix-action@v17` with daemon mode pushes store paths as they're built. The critical interaction with crane: once `buildDepsOnly` is pushed to Cachix, every CI run and every developer machine skips the entire dependency compilation. Only your source code gets rebuilt.

**Magic Nix Cache** (DeterminateSystems) nearly died in early 2025 when GitHub deprecated its v1 Cache API. A community member reverse-engineered the new gRPC API, and it's functional again — but it runs on undocumented APIs that GitHub could break. It's free, zero-config, and adequate for open source projects that can tolerate occasional breakage. For anything more reliable, **FlakeHub Cache** ($20/org member/month) offers cross-environment caching with per-flake access control.

**Attic** is the primary self-hosted alternative. It uses S3-compatible storage, supports multi-tenant caches with JWT tokens, and provides global chunk-level deduplication. It describes itself as an "early prototype" but is actively used by self-hosters avoiding Cachix pricing.

A standard GitHub Actions CI configuration:

```yaml
steps:
  - uses: actions/checkout@v5
  - uses: cachix/install-nix-action@v31
  - uses: cachix/cachix-action@v17
    with: { name: mycache, authToken: "${{ secrets.CACHIX_AUTH_TOKEN }}" }
  - run: nix build -L
  - run: nix flake check -L
```

CI parity with local development is the core principle: the same flake, same `nix build`, same `nix flake check`. **Remote builders** via nixbuild.net (free tier: 25 CPU hours/month) or self-hosted NixOS machines handle cross-platform builds. nixbuild.net's remote store mode eliminates closure copying entirely — their benchmarks show **20 minutes → 20 seconds** for typical builds.

---

## Where consensus fractures and what's overrated

The community agrees on rust-overlay + crane as the default stack but disagrees on several fronts.

**The granularity debate is unresolved.** Crane's two-derivation model is simpler but rebuilds all dependencies as a unit. crate2nix's per-crate model is more granular but adds evaluation overhead and codegen/IFD complexity. Ivan Petkov (crane's author) has argued that per-crate benefits are "overstated" because different feature flag combinations prevent sharing between projects anyway. devenv chose crate2nix regardless. The practical answer: crane for most projects, crate2nix (via devenv) for projects with massive dependency trees where single-crate changes are common.

**flake-utils vs flake-parts** splits along the simplicity/composability axis. Advanced users prefer flake-parts for its module system extensibility; others find it unnecessary abstraction. Multiple blog posts explicitly recommend removing flake-utils in favor of flake-parts.

Several **cargo-culted patterns** persist in the ecosystem. Using `rustup` inside Nix shells breaks reproducibility. referencing nixpkgs-mozilla (deprecated years ago) still appears in outdated tutorials. dream2nix for Rust is recommended by some guides despite being explicitly marked experimental by its own documentation. Over-engineering flake.nix with excessive inputs and abstractions when a simple `buildRustPackage` + `mkShell` suffices for straightforward projects is another common trap.

**What's genuinely overrated**: the idea that Nix should manage your entire development loop. The pragmatic consensus is to use `cargo build` and `cargo check` inside `nix develop` for fast iteration, reserving `nix build` for CI and packaging. Fighting Nix's sandbox to get incremental compilation is a losing battle — Nix derivations don't persist a `target/` directory between invocations. Accept the dual workflow.

---

## Pain points that remain genuinely unresolved

**Eight-plus build tools with no single winner** is the ecosystem's original sin. The NixOS Wiki dutifully lists them all with a comparison table, but this creates decision paralysis for newcomers and maintenance burden for the community. Convergence is happening (toward crane) but slowly.

**IFD remains a fundamental tension.** Tools that parse `Cargo.lock` need either IFD (disallowed in nixpkgs and Hydra), codegen (maintenance burden of committing generated files), or precomputed hashes (workflow friction on every dependency change). Crane and naersk solve this by parsing Cargo.lock in pure Nix, but this limits them to two-derivation granularity rather than per-crate.

**Cross-compilation works for simple cases but breaks down fast.** Pure Rust projects cross-compile via `pkgsCross.*` or crane's cross templates. Add C dependencies and everything gets complex: correctly splitting `nativeBuildInputs` from `buildInputs`, ensuring pkg-config finds the right target libraries, and avoiding full-toolchain recompilation. WASM is particularly painful — using `wasm32-unknown-unknown` via nixpkgs' `crossSystem` triggers building rustc from source with WASM as the only target, which fails. The workaround is manually overriding the build step. **Flakebox** (used by Fedimint in production) is the most complete cross-compilation solution, handling Linux, Android, Darwin, and iOS targets with non-trivial C dependencies.

**NixOS dynamic linking** continues to trip people up. Rust 1.90+ shipping its own lld by default compounds the existing RPATH issues on NixOS. Any use of lld without the NixOS ld wrapper produces binaries that fail at runtime. rustup-installed toolchains are dynamically linked and simply don't work on NixOS without patching.

---

## Frontier patterns worth watching

**Tvix and Snix** — Rust reimplementations of Nix itself — are the most significant long-term development. devenv is transitioning its evaluator to Tvix/Snix, meaning the Nix ecosystem is becoming Rust-native from the ground up. Lix (a Nix fork) plans gradual Rust rewrites as well.

**Flakebox** represents the most mature attempt at production-grade Nix+Rust infrastructure. Created by dpc from professional experience on the Fedimint Bitcoin project, it wraps crane with integrated cross-compilation, CI generation, and developer experience tooling. It's opinionated but battle-tested.

**devenv's `languages.rust.import`** (August 2025) is the clearest attempt to eliminate the build framework choice entirely — developers write `languages.rust.import = true` and devenv internally handles everything via crate2nix, including producing deployable Nix derivations from Cargo projects. This "managed complexity" approach mirrors how devenv already handles toolchain selection by defaulting to rust-overlay.

NixOS-native deployment patterns are maturing around `pkgs.dockerTools.buildLayeredImage` for minimal container images from Nix-built Rust binaries, and NixOS module system integration for systemd services. NixOS 25.11's Rust-based `nixos-init` and the COSMIC desktop environment demonstrate Rust's viability at the systems level within NixOS.

## Conclusion

The 2025–2026 Nix+Rust ecosystem has a clear default stack — **rust-overlay for toolchains, crane for builds, flake-parts for structure, nix-direnv for shell entry** — but this default obscures genuine tradeoffs that practitioners should understand rather than cargo-cult. crate2nix's resurgence via devenv represents a credible alternative philosophy (per-crate granularity over composable simplicity). The deepest unresolved tensions are structural: IFD policy prevents ideal build granularity in pure evaluation, cross-compilation with C dependencies remains expert-level work, and NixOS's dynamic linking model creates unique friction that Rust's evolving linker defaults exacerbate. The frontier is converging on managed abstractions (devenv, rust-flake) that hide these complexities, and Rust reimplementations of Nix itself (Tvix/Snix) that may eventually reshape the entire foundation.