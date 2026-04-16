## TL;DR recommended stack

If I were starting a greenfield Rust project on **NixOS-only** in April 2026 and optimizing for “native NixOS elegance” (not portability), I would standardize on:

**Flakes + flake-parts + rust-overlay toolchain (rust-toolchain.toml as source of truth) + crane for Nix builds/checks + direnv+nix-direnv for shell entry + Cachix/Attic for caching + (optional) NixOS-native debuginfod and remote builders.** citeturn24view0turn27view0turn37view0turn38view0turn6search10turn41search4turn14search2turn14search3

This is “best-in-class” for **serious solo hackers and small-to-mid teams** (1–30 engineers) who want maximal local ergonomics *and* CI parity, while remaining maintainable as the repo grows. For **larger orgs / monorepos**, keep the same core, then selectively add **crate-level build graph tooling (crate2nix) and distributed build infrastructure** when the workspace size forces it. citeturn32view0turn16view0turn14search2turn41search4

My ranking (strongly opinionated):

**Best default (2026):** flake-parts + rust-overlay + crane + nix-direnv + binary cache  
**Best “batteries included” alternative:** devenv (when you want processes/services/tasks as first-class Nix config)  
**Best for ultra-large workspaces where derivation granularity matters:** crate2nix (sometimes paired with devenv) citeturn24view0turn37view0turn38view0turn6search10turn21view0turn32view0

## Ecosystem reality in 2025–2026

### What serious NixOS Rust users converge on

The *center of gravity* in 2025–2026 is: **flake-based repos** and increasingly **flake-parts** to avoid bespoke glue. flake-parts explicitly frames itself as “core of a distributed framework” that uses the module system to reduce custom flake wiring and make configuration reusable. citeturn24view0

For day-to-day “enter the environment”, the ergonomic consensus is: **direnv + nix-direnv**. nix-direnv’s value prop is practical and NixOS-native: it caches the resulting shell derivation and pins it via gcroots so you don’t randomly lose build caches to GC (especially painful offline). citeturn6search10turn7search2

For Rust specifically, there’s also a well-understood split between:
- **fast local iteration**: run `cargo build/check/test` inside the devshell (so incremental compilation is real and persistent in your repo’s `target/`)  
- **hermetic builds and CI gating**: run the same actions via **Nix derivations** (so caches/substituters/CI parity become reliable)  

This split is implicitly encouraged by tooling like crane: it’s designed to build Cargo projects *with Nix caching semantics*, and to split “dependencies build once” from “run clippy/tests/docs/etc as separate derivations”. citeturn38view0turn39view0

### Where consensus breaks down

The main fracture lines in 2025–2026 aren’t “Rust vs Nix”. They are *dev environment UX and abstraction level*:

- **Plain flakes + mkShell** people want maximal transparency and minimal moving parts.
- **devenv** people want a higher-level module system, tasks, services, and turnkey editor/process integration; devenv markets fast shell reloads, incremental evaluation caching, and built-in functionality across many languages and services. citeturn18search0turn13view0
- **devshell / “naked shells”** vs **stdenv-based shells**: multiple community threads point out that “naked” shells can miss important stdenv hooks; Domen Kožar notes devenv tried the naked-shell route but it caused pain because stdenv hooks (used widely in nixpkgs packaging) didn’t trigger reliably. citeturn26view0

My take: on NixOS-only Rust, **stdenv hooks are not optional trivia**—they’re the difference between “bindgen works” and “bindgen explodes”, and between reproducible linker environment vs silent host leakage. If you want bleeding-edge but stable, prefer **mkShell/mkShellNoCC-style shells** (or abstractions built on them) over “naked shell” minimalism. citeturn26view0turn23view0

### Rising vs fading tools

The “modern stable” set:
- **crane** is firmly established as the mainstream “serious Rust + Nix” library. It advertises dependency vendoring, reusable dependency artifacts, and first-class integration with clippy/rustfmt/cargo-doc plus tools like cargo-nextest and cargo-llvm-cov; it also explicitly recommends pinning versions because breaking changes may land on `master`. citeturn38view0turn39view0
- **oxalica/rust-overlay** is still the default “power toolchain provider” on NixOS when you care about stable/beta/nightly and components/targets. It is explicitly designed to be pure in evaluation by prefetching hashes for toolchain components and auto-updating them daily. citeturn37view0
- **nix-direnv** has become “the pragmatic standard” for auto-activation and avoiding GC churn. citeturn6search10

Re-emerging / re-legitimized:
- **crate2nix**: after being quiet for ~2 years, crate2nix 0.15.0 was announced in January 2026 with notable features like private registries support and cross-compilation fixes—i.e., this is no longer safely dismissed as dead. citeturn32view0turn11search0

Fading / outdated / cargo-culted (in 2026):
- **mozilla/nixpkgs-mozilla** as a Rust toolchain source is basically legacy. Even its own README warns it may require `--impure` because it fetches from non-pinned URLs non-reproducibly. On NixOS-only, that’s a hard “no” for state-of-the-art. citeturn31view0turn30view0
- **dream2nix as the default Rust path** is still hard to recommend for greenfield Rust unless you *specifically* want its cross-language packaging framework. Dream2nix’s own site warns the software is unstable, APIs may break, and it’s mid-refactor to drv-parts with incomplete migration. citeturn35view0turn6search1

## Toolchain strategy on NixOS

### What I consider “state-of-the-art” toolchain pinning

State-of-the-art on NixOS is: **`rust-toolchain.toml` is the canonical toolchain spec** (because it’s what Rust tooling expects), and Nix consumes it rather than replacing it with Nix-only pinning.

`oxalica/rust-overlay` directly supports this with `rust-bin.fromRustupToolchainFile ./rust-toolchain.toml`. Under the hood, it cleanly maps the rustup toolchain file format (channel/profile/components/targets) into a Nix toolchain derivation. citeturn37view0turn27view0

This is the sweet spot for NixOS-only:
- Rust-native pinning semantics (works with Cargo, rust-analyzer, the Rust ecosystem)
- Nix purity and reproducibility (toolchain comes from Nix store, not mutable `~/.rustup`)
- No “hash yak shaving” for toolchain manifests (contrast fenix’s pure-eval requirement below) citeturn37view0turn36view0

### rust-overlay vs fenix vs nixpkgs toolchain

**rust-overlay (oxalica)** is my default recommendation in 2026 for “bleeding edge, but disciplined” NixOS Rust:

- It makes evaluation pure by shipping pre-fetched hashes for toolchain components and keeping them updated automatically. citeturn37view0  
- It has first-class support for stable/beta/nightly and custom toolchains (including from `rust-toolchain` files). citeturn37view0turn27view0  
- It explicitly warns against the brittle nightly pattern `rust-bin.nightly.latest` (components can be missing on some days). The recommended pattern is `rust-bin.selectLatestNightlyWith (...)` so your build selects a nightly that actually has required components. citeturn37view0  
- It also has a reality check: the repo keeps nightly/beta versions only back to `{current_year - 1}-01-01`, older via snapshot tags. If you want “nightly-2022-…” forever without snapshot discipline, you’ll fight entropy. citeturn37view0  

**fenix** remains excellent, especially when you care about rust-analyzer-nightly and want a “nightly but not too volatile” track:

- fenix provides the standard toolchain profiles and explicitly includes a nightly rust-analyzer and VSCode extension, plus a Cachix binary cache. citeturn36view0  
- fenix has a “monthly” branch updated on the 1st of every month—useful when you want nightly features but don’t want daily drift. citeturn36view0  
- The key tradeoff: fenix often requires a **sha256 for manifests in pure evaluation mode** (e.g., `fromToolchainFile { ...; sha256 = lib.fakeSha256; }`). This is *fine*, but it’s more operational overhead than rust-overlay’s “hashes are in-tree”. citeturn36view0  

**nixpkgs toolchain** is still totally valid on NixOS *if* you only need “the Rust in this nixpkgs revision”:

- The nixpkgs manual explicitly frames nixpkgs as the simple way to install rustc/cargo system-wide, and suggests rustup or community toolchains for nightly/beta. citeturn30view0  
- For state-of-the-art teams, nixpkgs-only becomes constraining quickly because nixpkgs generally tracks one stable toolchain version per revision; switching stable versions or mixing stable/nightly is where overlays shine. citeturn23view0turn30view0  

### rust-analyzer integration that doesn’t suck on NixOS

On NixOS, rust-analyzer pain usually collapses to one issue: **std library sources**.

The NixOS Rust wiki calls out that some Rust tools need `RUST_SRC_PATH` set; it also notes that rust-analyzer from nixpkgs doesn’t require this, and that using rust-overlay with the `rust-src` extension is another fix. citeturn23view0

So, in a rust-overlay + custom toolchain world, the clean NixOS-native move is:
- include **`rust-src`** in your toolchain components (in `rust-toolchain.toml`)
- export `RUST_SRC_PATH` from the toolchain location (or let your tooling do it)

If you do choose fenix in 2026, fenix even documents the mapping explicitly: `rust-src` corresponds to `RUST_SRC_PATH = ".../lib/rustlib/src/rust/library"`. citeturn36view0

For VS Code specifically, the NixOS wiki suggests one pragmatic NixOS-native escape hatch: set `"rust-analyzer.server.path": "rust-analyzer"` so the extension uses the Nix-provided rust-analyzer binary instead of a bundled download. citeturn23view0

## Building and CI: what actually wins in practice

### The core philosophy that scales

For NixOS-only Rust, the “state-of-the-art” pattern is:

- **Cargo owns your inner loop** (incremental compile, fast edit/build/test, `target/` stays warm).
- **Nix owns your environment and CI contract** (toolchain pinned, native deps pinned, checks are hermetic derivations, binary caches accelerate everyone).

Trying to force “100% pure Nix builds for every keystroke” is a self-inflicted performance wound. The winning pattern is to use Nix as the *substrate* and Cargo as the *interactive* tool.

### crane vs naersk vs buildRustPackage vs cargo2nix/crate2nix vs dream2nix

**crane** is the best default in 2026.

Reasons (from crane itself):  
- It vendors dependencies in a Nix-friendly way and reuses dependency artifacts after building them once. citeturn38view0turn39view0  
- It’s designed for composability: clippy, docs, formatting, nextest, audit/deny can be separate derivations that all share the same dependency artifacts. citeturn39view0turn38view0  
- It explicitly supports cargo-nextest and cargo-llvm-cov out of the box. citeturn38view0turn39view0  
- It has a clear compatibility policy: pin versions because breaking changes can land on master; releases are semver-tagged and documented. citeturn38view0

The hidden “2026 frontier” detail: **monorepo/workspace scaling**.  
A 2026 deep-dive on optimizing crane builds for Cargo workspaces shows two real bottlenecks in naive workspaces: (1) changes in any crate invalidate all builds if the entire workspace is hashed as `src`, and (2) building shared `cargoArtifacts` for the whole workspace compiles dependencies some packages don’t need; the article demonstrates source isolation and per-package dependency artifacts, plus explicit tradeoffs like added complexity and possible IFD usage. citeturn16view0  
This is exactly where “state-of-the-art” teams push beyond boilerplate crane templates.

**naersk** is still good, but it is no longer “frontier”.

naersk is essentially “Cargo inside Nix” with minimal ceremony: it parses Cargo.lock and builds in the Nix sandbox, and importantly it doesn’t use IFD (useful for Hydra-style constraints). citeturn34view0  
However, naersk’s default story is “use the Rust version in nixpkgs” and it explicitly notes it ignores `rust-toolchain` files unless you wire a custom toolchain yourself. citeturn34view0  
In practice, teams reach for naersk when they want simplicity and “it just works”, but crane has eaten most of the “serious CI gating” use case.

**rustPlatform.buildRustPackage (nixpkgs)** is for packaging correctness, not developer happiness.

The nixpkgs manual is very clear: `buildRustPackage` needs either `cargoHash`/`cargoSha256` (tedious when Cargo.lock changes) *or* a `cargoLock`-driven flow that vendors from Cargo.lock; it also highlights caveats like Cargo.lock patching timing. citeturn30view0  
This is great when you’re upstreaming into nixpkgs or maintaining Nixpkgs-style packages; it’s less pleasant as a daily driver for a fast-moving Cargo workspace.

**cargo2nix** and **crate2nix** are about derivation granularity and monorepo economics.

cargo2nix’s core model is: generate a `Cargo.nix`, commit it, then build crates with fine-grained derivations; its README emphasizes caching and reproducibility benefits plus a `workspaceShell` to make `cargo build` work nicely in `nix develop`. citeturn33view0  
crate2nix similarly targets crate-by-crate builds; and importantly, crate2nix visibly regained momentum with the 0.15.0 release in Jan 2026, including private registries support and cross-compilation fixes. citeturn32view0turn11search4  

My 2026 opinion:  
- If you have a **huge** workspace where build invalidation and cache reuse across crates becomes a material cost (time + money), crate2nix-style “many derivations” can win.  
- Otherwise, crane’s “two derivations + composable checks” hits the best engineering ROI.

**dream2nix** is strategically interesting but operationally risky.

dream2nix is explicit that it’s unstable and APIs may break; it’s mid-refactor to drv-parts, with incomplete feature migration. citeturn35view0turn6search1  
That doesn’t mean “never”—it means: only choose it when you need its cross-language framework and you’re willing to own churn.

### devenv as the “batteries included” path (and why it matters)

Even if you don’t adopt devenv, you should understand how it’s shaping expectations.

devenv’s Rust module offers toolchain management via nixpkgs or rust-overlay channels, supports selecting components/targets, and exposes developer-speed features like enabling mold as a linker and enabling the Cranelift backend (nightly-only), including per-package LLVM fallbacks for problematic crates. citeturn13view0turn21view0  

It also demonstrates a key design goal: **one config for dev + packaging**. The 2025 blog post says devenv introduced `languages.rust.import` so users don’t have to choose among crate2nix/cargo2nix/naersk/crane; and it explicitly says devenv switched from fenix to rust-overlay for toolchains because rust-overlay was better maintained. citeturn10view0turn13view0  

One “bleeding edge” nuance: devenv moves fast enough that docs can lag implementation. The Rust module docs text says `languages.rust.import` uses cargo2nix, but the actual module code shows it wiring in **crate2nix** and even includes a 2026-03-08 changelog entry about using the configured toolchain; in 2026 you must occasionally read the module source, not just the rendered docs. citeturn13view0turn21view0

## Reference architecture for a world-class NixOS-native Rust setup

This is the architecture I’d ship for a serious Rust workspace on NixOS in 2026: **flake-parts + rust-overlay + crane + treefmt-nix + git-hooks-nix + nix-direnv**.

### Repo layout

A layout that scales without becoming a Nix ball of mud:

```text
.
├── flake.nix
├── flake.lock
├── rust-toolchain.toml         # canonical toolchain pin
├── Cargo.toml
├── Cargo.lock
├── crates/…                    # workspace members
└── nix/
    ├── rust.nix                # toolchain + crane lib glue
    ├── packages.nix            # package definitions
    ├── checks.nix              # clippy/fmt/nextest/audit/deny
    └── devshell.nix            # dev ergonomics (mold, sccache, env vars)
```

flake-parts exists specifically to let you split your flake into focused units and reuse module logic, rather than writing bespoke flake-utils loops forever. citeturn24view0

### Toolchain pinning strategy

1. **Commit `rust-toolchain.toml`** and treat it as canonical.
2. In Nix: use rust-overlay’s `fromRustupToolchainFile` to produce the toolchain derivation. citeturn37view0turn27view0
3. In crane: override the toolchain so *all* builds/checks use the exact same toolchain as the devShell.

crane supports overriding the whole toolchain via `overrideToolchain`, and Discourse examples show using `p: p.rust-bin.fromRustupToolchainFile ./rust-toolchain.toml`. citeturn40search8turn38view0turn27view0

### Annotated flake.nix skeleton

This is intentionally “realistic” rather than minimal. It uses flake-parts to keep the structure sane, and it wires packages/checks/devShell/formatter in the places Nix expects.

```nix
{
  description = "NixOS-native Rust workspace (2026 reference)";

  # Optional but recommended in NixOS-only teams:
  # Encode substituters in the flake so "nix develop" users get caches automatically.
  # (Requires accept-flake-config = true to avoid prompts.)
  nixConfig = {
    extra-substituters = [
      "https://myorg.cachix.org"
      # or your Attic cache endpoint
    ];
    extra-trusted-public-keys = [
      "myorg.cachix.org-1:AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA="
    ];
  };

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";

    flake-parts.url = "github:hercules-ci/flake-parts";

    rust-overlay = {
      url = "github:oxalica/rust-overlay";
      inputs.nixpkgs.follows = "nixpkgs";
    };

    crane.url = "github:ipetkov/crane";

    treefmt-nix.url = "github:numtide/treefmt-nix";
    git-hooks-nix.url = "github:cachix/git-hooks.nix";
  };

  outputs = inputs@{ flake-parts, ... }:
    flake-parts.lib.mkFlake { inherit inputs; } {

      systems = [ "x86_64-linux" "aarch64-linux" ];

      imports = [
        inputs.treefmt-nix.flakeModule
        inputs.git-hooks-nix.flakeModule
        ./nix/rust.nix
        ./nix/packages.nix
        ./nix/checks.nix
        ./nix/devshell.nix
      ];
    };
}
```

Why this design is “frontier” and not boilerplate:

- The `nixConfig` pattern is how advanced teams keep **CI parity with local dev**: everyone automatically uses the same substituters. The tradeoff is you should enable `accept-flake-config` in Nix to avoid interactive prompts (documented in the Nix config reference, and commonly used in per-project cache workflows). citeturn42search4turn28view0  
- treefmt-nix integrates with flakes by setting `formatter.<system>` so `nix fmt` is standardized, and it can also be checked in `nix flake check`. citeturn22search1turn22search4  
- git-hooks.nix exists to avoid slow, ad-hoc hook execution; it generates a pre-commit config, provides an activation script, and supplies a check—flake-parts exposes it cleanly as a module. citeturn22search0turn22search3  

### nix/rust.nix: rust-overlay toolchain + crane override

```nix
{ inputs, ... }:
{ perSystem = { pkgs, lib, system, ... }:
  let
    # Bring rust-overlay into pkgs
    pkgs' = import inputs.nixpkgs {
      inherit system;
      overlays = [ inputs.rust-overlay.overlays.default ];
    };

    rustToolchain = pkgs'.rust-bin.fromRustupToolchainFile ./rust-toolchain.toml;

    craneLib = (inputs.crane.mkLib pkgs').overrideToolchain (p: rustToolchain);
  in {
    _module.args = {
      inherit pkgs' rustToolchain craneLib;
    };
  };
}
```

Key points backed by upstream behavior:

- rust-overlay’s `fromRustupToolchainFile` implements the rustup toolchain file semantics (legacy string or TOML), and maps components/targets/profile into a toolchain derivation. citeturn27view0turn37view0  
- crane recommends a versioning strategy and provides `overrideToolchain` for overlaying an entire toolchain (cargo/rustc/clippy/rustfmt etc.). citeturn38view0turn40search1  

### nix/packages.nix: packages for the workspace

```nix
{ ... }:
{ perSystem = { pkgs', lib, craneLib, ... }:
  let
    src = craneLib.cleanCargoSource ./.; # keeps Cargo.toml/.lock and Rust sources
    commonArgs = {
      inherit src;
      strictDeps = true;

      nativeBuildInputs = [
        pkgs'.pkg-config
      ];

      buildInputs = [
        pkgs'.openssl
        # pkgs'.sqlite
        # pkgs'.zlib
      ];
    };

    cargoArtifacts = craneLib.buildDepsOnly commonArgs;

    mkCrate = pname: craneLib.buildPackage (commonArgs // {
      inherit cargoArtifacts;
      cargoExtraArgs = "-p ${pname}";
    });
  in {
    packages = {
      myapp = mkCrate "myapp";
      default = pkgs'.myapp or mkCrate "myapp";
    };
  };
}
```

Notes:
- `cleanCargoSource` / source filtering is a crane-best-practice to avoid hashing irrelevant files and thrashing caches; crane documents `cleanCargoSource` and `filterCargoSources` as the standard source filtering approach. citeturn29search2turn39view0  
- The `buildDepsOnly → cargoArtifacts → buildPackage` split is the canonical crane cache pattern; crane’s quick start explicitly calls out that building just dependencies allows reuse “e.g. via Cachix” in CI. citeturn39view0  

### nix/checks.nix: hermetic clippy/fmt/nextest/audit/deny

```nix
{ inputs, ... }:
{ perSystem = { pkgs', craneLib, lib, config, ... }:
  let
    src = craneLib.cleanCargoSource ./.;
    commonArgs = { inherit src; strictDeps = true; };
    cargoArtifacts = craneLib.buildDepsOnly commonArgs;
  in {
    checks = {
      # Build is a check (useful as a fast smoke test)
      build = craneLib.buildPackage (commonArgs // { inherit cargoArtifacts; });

      clippy = craneLib.cargoClippy (commonArgs // {
        inherit cargoArtifacts;
        cargoClippyExtraArgs = "--all-targets -- --deny warnings";
      });

      fmt = craneLib.cargoFmt { inherit src; };

      # cargo-nextest is the modern “serious workspace” test runner.
      nextest = craneLib.cargoNextest (commonArgs // {
        inherit cargoArtifacts;
        partitions = 1;
        partitionType = "count";
      });

      # Optional: supply-chain hygiene
      audit = craneLib.cargoAudit {
        inherit src;
        advisory-db = inputs.advisory-db or null;
      };

      deny = craneLib.cargoDeny { inherit src; };
    };
  };
}
```

Why this is “state-of-the-art”:

- crane is explicitly designed so checks like clippy/fmt/nextest are separate derivations that reuse the same dependency artifacts, which makes CI caching extremely effective. citeturn38view0turn39view0  
- cargo-nextest itself is explicitly designed to be faster than `cargo test` (up to 3×), with per-test isolation and CI support; it also supports workspace-level configuration in `.config/nextest.toml`. citeturn29search10turn29search11  

### nix/devshell.nix: developer speed as a first-class goal

```nix
{ ... }:
{ perSystem = { pkgs', rustToolchain, craneLib, ... }:
  {
    devShells.default = pkgs'.mkShell {
      nativeBuildInputs = [
        rustToolchain

        # Iteration speed
        pkgs'.mold
        pkgs'.clang
        pkgs'.sccache

        # Better testing and profiling UX
        pkgs'.cargo-nextest
        pkgs'.cargo-llvm-cov

        # Debugging/profiling primitives
        pkgs'.gdb
        pkgs'.lldb
        pkgs'.perf
      ];

      # The NixOS wiki explicitly calls out RUST_SRC_PATH as a fix for tools needing stdlib sources.
      # If rust-analyzer is coming from the toolchain, you usually want this.
      RUST_SRC_PATH = "${rustToolchain}/lib/rustlib/src/rust/library";

      shellHook = ''
        export RUSTC_WRAPPER=sccache
        # mold is documented as "several times quicker than LLVM lld" and targets edit-build cycles.
        export RUSTFLAGS="-C link-arg=-fuse-ld=mold"

        echo "Rust toolchain: $(rustc --version)"
      '';
    };
  };
}
```

Justification for these “speed knobs” with primary sources:

- mold describes itself as a faster drop-in linker, “several times quicker than LLVM lld”, specifically to reduce edit/rebuild cycle time. citeturn17search0  
- sccache is a compiler wrapper that caches compilation outputs locally or in remote backends, and explicitly supports Rust. citeturn17search2turn42search8  
- cargo-llvm-cov is the modern LLVM source-based coverage wrapper around `-C instrument-coverage`, and it supports `cargo nextest` among other modes. citeturn29search14turn29search7  
- The NixOS Rust wiki explicitly documents setting `RUST_SRC_PATH` (and notes rust-overlay with `rust-src` as an alternative fix). citeturn23view0  

## Frontier patterns and unresolved pain points

### Purity vs developer speed: the real tradeoff

The ecosystem has effectively accepted a two-tier model:

- **Tier A (fast dev loop):** Cargo builds in the repo, incremental compilation stays warm, LSP is responsive.
- **Tier B (true reproducibility / CI parity):** Nix derivations for build + checks; binary caches make this feasible at scale.

If you try to collapse Tier A into Tier B, you’ll burn time: Nix builds are isolated by design, and you won’t get “persistent incremental compilation” unless you intentionally engineer around it (and most teams shouldn’t).

The frontier work here is not “make Nix behave like Cargo”. It’s **make the Nix layer cache-efficient** so CI and teammates don’t pay for your dependency rebuilds.

The 2026 crane-workspace optimization writeup shows what this looks like in practice: per-crate source filtering and per-package dependency artifacts to avoid invalidating sibling crates and compiling irrelevant dependencies; it also bluntly documents the costs (complexity, more derivations, evaluation-time reads, sometimes IFD). citeturn16view0

### Monorepo scaling: when “two derivations” stops being enough

crane’s default pattern (one shared deps derivation + one package derivation per crate) is a massive improvement over “everything rebuilds always”, but large workspaces can still suffer.

The frontier options in 2026 are:

- **Advanced crane patterns** (source isolation + per-package deps artifacts) as above. citeturn16view0  
- **crate2nix when derivation granularity dominates**: crate2nix targets “crate-by-crate” hermetic builds, and its 2026 release explicitly calls out improvements like private registries support and cross compilation fixes, which are exactly the friction points that previously limited adoption. citeturn32view0turn11search4  
- **cargo2nix as a mature “generate Cargo.nix and commit it” pipeline** when you’re comfortable with codegen and want fine-grained derivations plus a `workspaceShell` for local dev. citeturn33view0  

### CI, caching, and distributed builds: what “serious teams” do on NixOS

A “state-of-the-art” NixOS Rust setup treats caches as first-class production infra.

Binary caches:
- **Cachix** remains the default managed service; it is explicitly a CLI client for Nix binary cache hosting. citeturn41search8  
- **Attic** is the most prominent self-hosted option: it’s a self-hostable binary cache backed by S3-compatible storage, with global deduplication and garbage collection; its docs call it an early prototype looking for testers. citeturn41search4turn41search0  

Distributed builds:
- Nix’s distributed builds are standard practice on NixOS when you have heterogeneous hardware or want to offload builds; nix.dev provides a tutorial for setting up distributed builds. citeturn14search2  
- Past a certain scale, you start caring about remote-store semantics (`ssh-ng://`) and build scheduling; nixbuild.net’s docs highlight `ssh-ng://` as the newer protocol and discuss tradeoffs of remote builders vs remote stores, especially when multiple clients share builders. citeturn14search9  
- On the bleeding edge, tools like **rio-build** attempt to make Nix derivation DAGs run efficiently across Kubernetes clusters by speaking Nix’s remote store/builder protocol. citeturn14search13  

A subtle but very real 2026 performance concern: *too many substituters*. One practical pattern is to enable only the official cache globally and selectively enable project-specific caches via flake `nixConfig` or direnv; this is motivated by the cost of querying many caches for every build. citeturn28view0

### Debugging ergonomics: NixOS-native debuginfod

A classic Nix pain point is debug symbols: even if packages have separate debug outputs, you might not automatically download them, and debuggers may not find them.

nixseparatedebuginfod exists specifically to solve this by fetching debug symbols/sources for Nix packages so gdb can use them; the project documents an easy NixOS module (`services.nixseparatedebuginfod.enable = true`) and discusses the underlying “separateDebugInfo exists but you don’t download it or wire it into gdb” problem. citeturn14search3turn14search6

If you care about “low-friction debugging” as a first-class requirement, this is one of the most NixOS-native, high-leverage wins you can add.

### Where I land in 2026

For a greenfield NixOS Rust project:  
I would choose **flake-parts + rust-overlay toolchain from rust-toolchain.toml + crane checks + nix-direnv + (Cachix or Attic) cache**, plus mold + sccache + nextest for iteration speed. citeturn24view0turn37view0turn38view0turn6search10turn17search0turn17search2turn29search10turn41search4

For a team:  
Same stack, but I would be stricter about:
- per-project substituter discipline (flake `nixConfig`, `accept-flake-config`) citeturn28view0turn42search4  
- remote builders (nix.distributedBuilds + buildMachines) once builds become expensive citeturn14search2  
- adding nixseparatedebuginfod in the base NixOS config so debugging doesn’t degrade into archaeology citeturn14search3turn14search6  

What I would avoid:  
- mozilla/nixpkgs-mozilla for Rust toolchains (explicitly non-reproducible fetch behavior / impure workflows). citeturn31view0turn30view0  
- dream2nix as the default Rust path for a team that wants stability (it tells you it’s unstable and mid-refactor). citeturn35view0  
- “cargo-cult purity” where developers are forced to run all builds through Nix for every edit; it burns the main advantage of Cargo (incrementality) while not improving the CI contract beyond what crane already gives you. citeturn38view0turn39view0