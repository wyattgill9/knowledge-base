# Writing Idiomatic, Robust Nix: A 2024–2026 Practical Guide

**Bottom line up front:** In 2026, idiomatic Nix means flakes for project entry points, the NixOS module system for everything that composes (NixOS, Home Manager, **flake-parts** for flakes themselves), `stdenv.mkDerivation (finalAttrs: ...)` instead of `rec` for derivations, `nixfmt` (RFC 166) as the formatter, `statix` + `deadnix` as linters, `treefmt-nix` + `git-hooks.nix` as the CI glue, and `pkgs.callPackage` + overlays as the dependency-injection backbone. Hand-rolled `forAllSystems = lib.genAttrs lib.systems.flakeExposed` is now preferred over `flake-utils` for small flakes; `flake-parts` is preferred for anything non-trivial.

## TL;DR

- **Adopt this stack today**: flakes + flake-parts (or hand-rolled `forAllSystems`) + nixfmt v1 + statix + deadnix + treefmt-nix + git-hooks.nix + direnv/nix-direnv, with `stdenv.mkDerivation (finalAttrs: …)`, `final: prev:` overlays, and `pkgs/by-name`-style file layout. `flake-utils` still works but is no longer the community default for new code.
- **Three idioms that fix 80 % of bad Nix you'll read**: (1) replace `rec { }` and top-level `with pkgs;` with `let`-bindings and `finalAttrs:`; (2) use `lib.mkIf`, `lib.mkMerge`, `lib.mkDefault`/`mkForce` inside `config = …`, never naked `if/then/else` referencing `config`; (3) use `callPackage`/overlays for dependency injection — never `import ./foo.nix { inherit pkgs; }`.
- **Where the community is still split**: framework choice (flake-parts vs flake-utils vs hand-rolled), formatter (`nixfmt` is now official per RFC 166 but `alejandra` retains a following), and Python packaging (`uv2nix` is the recommended successor to `poetry2nix`, but `poetry2nix` is still maintained-by-committee and widely deployed).

---

## 1. Nix language idioms and anti-patterns

### 1.1 `let`/`in` over `with`, `rec`, and `<nixpkgs>`

The official guidance, from `nix.dev`'s *Best practices* page, is unambiguous: **prefer `let … in` to `rec` and avoid `with` except in the smallest scopes**. The page lists three direct anti-patterns:

> "Avoid `rec`. Use `let … in`."
> "Do not use `with` at the top of a Nix file. Explicitly assign names in a `let` expression."
> "`<...>` is special syntax that was introduced in 2011 to conveniently access values from the environment variable `$NIX_PATH`. This means the value of a lookup path depends on external system state."

(Source: `nix.dev` *Best practices*, current as of 2025.)

```nix
# ❌ Anti-idiomatic
with import <nixpkgs> {};
rec {
  version = "1.0";
  src = fetchurl { url = "..."; sha256 = "..."; };
  pkg  = stdenv.mkDerivation { inherit version src; };
}

# ✅ Idiomatic
{ pkgs ? import (fetchTarball "https://github.com/NixOS/nixpkgs/archive/<rev>.tar.gz") {} }:
let
  inherit (pkgs) stdenv fetchurl;
  version = "1.0";
  src = fetchurl { url = "..."; sha256 = "..."; };
in
stdenv.mkDerivation { inherit version src; }
```

`with` *inside* a small expression is fine (`meta = with lib; { license = licenses.mit; maintainers = [ maintainers.alice ]; };`). The pathological cases are file-scope `with pkgs;` and `with lib;`, because **static analysis cannot tell where any unbound name comes from** and the scoping rules are non-intuitive (`with` doesn't shadow `let`).

`statix` will fix many of these automatically — its readme demonstrates rewriting `let mtl = pkgs.haskellPackages.mtl; in …` to `let inherit (pkgs.haskellPackages) mtl; in …`.

### 1.2 `inherit`, function arguments, and `@`-patterns

```nix
# Idiomatic function signature: open attrset pattern, default args, @-binding
{ lib
, stdenv
, fetchFromGitHub
, openssl
, enableSsl ? true
, ...
}@args:

stdenv.mkDerivation {
  pname = "foo";
  buildInputs = lib.optional enableSsl openssl;
  # `args` is available as a whole if you need to forward
}
```

Rules of thumb the nixpkgs CONTRIBUTING guide enforces:

1. Always use `{ ... }:` (with ellipsis) so callers may pass extra args that `callPackage` injects.
2. Put dependencies as named function arguments — never `pkgs.openssl` *inside* the body when `openssl` could have been an argument (this is what makes `callPackage`'s automatic threading work).
3. Use `lib.optional`/`lib.optionalString`/`lib.optionals` rather than `if … then … else []`/`""`/`[]`.

### 1.3 `//` vs `lib.recursiveUpdate`

`//` is a shallow update: `{ a.b = 1; a.c = 2; } // { a.b = 3; }` is `{ a = { b = 3; }; }` — the `c` is gone. Reach for `lib.recursiveUpdate` when merging nested attrsets that you control, but inside the module system **let the module system do it** via `mkMerge`/`mkDefault`. The `tweag/nix-ux` issue 20 summary captures the order: `mkOverride 1500` (`mkOptionDefault`) < `mkDefault` (1000) < normal (100) < `mkForce` (50) < `mkVMOverride` (10).

### 1.4 Pure vs impure, IFD

Impure language constructs (`builtins.getEnv`, `builtins.currentTime`, `builtins.currentSystem`, unpinned `<nixpkgs>`) are forbidden inside flakes — flakes are evaluated in pure mode. *Import-from-derivation* (IFD) is technically pure but pauses evaluation to realise a derivation, then reads its output as Nix code. The nixpkgs CI bans it; the Nix Reference Manual summarises the cost:

> "Evaluation can only finish when all required store objects are realised. Since the Nix language evaluator is sequential, it only finds store paths to read from one at a time. While realisation is always parallel, in this case it cannot be done for all required store paths at once, and is therefore much slower than otherwise." — *Nix Reference Manual*, "Import From Derivation"

Practical consequence: tools like `crane` (Rust), `poetry2nix`, and `dream2nix` use IFD, which is why nixpkgs itself uses hand-written `Cargo.lock`-derived expressions (`buildRustPackage`) instead. For your own project flakes, IFD is fine; for anything contributed to nixpkgs, it isn't.

### 1.5 Strings and assertions

```nix
# Indented string with proper escapes: '' begins, '' ends, ''${} escapes interpolation
shellHook = ''
  export FOO=${foo}        # interpolation
  echo "literal ''${bar}"  # NOT interpolated — bar is shell
'';

# Assertions ride the derivation
assert lib.versionAtLeast openssl.version "3";
stdenv.mkDerivation { … }

# throws give a typed error; abort kills evaluation hard (avoid in libraries)
foo = if cond then x else throw "foo requires cond to be true";
```

---

## 2. Flakes — modern idiomatic patterns

### 2.1 The current state (Nov 2025)

Flakes were introduced in Nix 2.4 (Nov 2021) and remain officially "experimental" in upstream Nix; **Determinate Nix ships them as stable**, and the recent Determinate Systems post *Nix flakes explained* (19 Nov 2025) lays out the direction:

> "Flakes close the loop on what Luc in the webinar calls the distributed input problem. Instead of each project and team pinning dependencies in ad-hoc ways, flakes provide a single, unified mechanism for determinism. … Every dependency is pinned in the flake.lock file."

Practically: every new Nix project in 2025–2026 should be a flake. The dissenting view, ably argued by Jade Lovelace in *Flakes aren't real and cannot hurt you* (jade.fyi), is that **most of your code should live in plain `.nix` files** and the flake should be a thin entry point:

> "Most of the Nix code in a project of any significant size should not be in flake.nix… there is as much rightward drift in flake.nix as in recent German and Dutch elections (concerningly much!)."

### 2.2 The four canonical flake shapes

**Shape A — Hand-rolled (smallest deps, most explicit, recommended by ayats.org & Determinate):**

```nix
{
  description = "My project";

  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";

  outputs = { self, nixpkgs }:
    let
      systems = [ "x86_64-linux" "aarch64-linux" "x86_64-darwin" "aarch64-darwin" ];
      forAllSystems = f: nixpkgs.lib.genAttrs systems
        (system: f nixpkgs.legacyPackages.${system});
    in {
      packages    = forAllSystems (pkgs: { default = pkgs.callPackage ./package.nix {}; });
      devShells   = forAllSystems (pkgs: { default = pkgs.callPackage ./shell.nix   {}; });
      formatter   = forAllSystems (pkgs: pkgs.nixfmt-rfc-style);
      checks      = forAllSystems (pkgs: { build = self.packages.${pkgs.system}.default; });
    };
}
```

**Shape B — `flake-utils` (still the most common in older code):**

```nix
outputs = { self, nixpkgs, flake-utils }:
  flake-utils.lib.eachDefaultSystem (system:
    let pkgs = nixpkgs.legacyPackages.${system}; in {
      packages.default = pkgs.callPackage ./package.nix {};
      devShells.default = pkgs.mkShell { packages = [ pkgs.go ]; };
    });
```

The problem ayats.org documents: `eachDefaultSystem` indiscriminately threads `system` into *every* output, so writing `nixosConfigurations.foo = ...` inside it produces `nixosConfigurations.x86_64-linux.foo`, which is wrong. Combine carefully with `//` for system-independent outputs, or — better — move on.

**Shape C — `flake-parts` (recommended for non-trivial flakes):**

```nix
{
  inputs.nixpkgs.url      = "github:NixOS/nixpkgs/nixos-unstable";
  inputs.flake-parts.url  = "github:hercules-ci/flake-parts";
  inputs.treefmt-nix.url  = "github:numtide/treefmt-nix";
  inputs.git-hooks.url    = "github:cachix/git-hooks.nix";

  outputs = inputs@{ flake-parts, ... }:
    flake-parts.lib.mkFlake { inherit inputs; } {
      systems = [ "x86_64-linux" "aarch64-linux" "x86_64-darwin" "aarch64-darwin" ];

      imports = [
        inputs.treefmt-nix.flakeModule
        inputs.git-hooks.flakeModule
      ];

      perSystem = { config, pkgs, system, self', inputs', ... }: {
        packages.default = pkgs.callPackage ./package.nix {};
        devShells.default = pkgs.mkShell {
          inputsFrom = [ config.treefmt.build.devShell ];
          packages   = [ pkgs.just ];
          shellHook  = config.pre-commit.installationScript;
        };

        treefmt.programs.nixfmt.enable = true;
        treefmt.programs.deadnix.enable = true;

        pre-commit.settings.hooks = {
          nixfmt.enable  = true;
          statix.enable  = true;
          deadnix.enable = true;
        };

        formatter = config.treefmt.build.wrapper;
      };

      # Flake-level (system-independent) outputs
      flake = {
        nixosModules.default = ./modules/default.nix;
        overlays.default     = import ./overlay.nix;
      };
    };
}
```

The flake-parts intro page (flake.parts) explicitly positions it as the NixOS-module-system applied to flakes:

> "flake-parts provides the options that represent standard flake attributes and establishes a way of working with system. … Flakes are configuration. The module system lets you refactor configuration into modules that can be shared."

In `perSystem`, the magic arguments are: `pkgs` (auto-imported nixpkgs), `system`, `self'` (the current flake with `system` pre-selected — note the prime), and `inputs'` (inputs with `system` pre-selected). `self'.packages.hello` is short for `self.packages.${system}.hello`.

**Shape D — `nix-systems` (the systems-as-an-input pattern):**

```nix
inputs.systems.url = "github:nix-systems/default";  # or default-linux, x86_64-linux, …

outputs = { systems, nixpkgs, ... }:
  let eachSystem = f: nixpkgs.lib.genAttrs (import systems)
                       (s: f nixpkgs.legacyPackages.${s});
  in { packages = eachSystem (pkgs: { default = pkgs.hello; }); };
```

The win: downstream consumers can `inputs.your-flake.inputs.systems.follows = "my-systems";` to restrict the system list — speeding up `nix flake show`/`check` on their side.

### 2.3 Managing inputs: `follows`, pinning, updates

```nix
inputs = {
  nixpkgs.url      = "github:NixOS/nixpkgs/nixos-unstable";
  flake-parts.url  = "github:hercules-ci/flake-parts";
  flake-parts.inputs.nixpkgs-lib.follows = "nixpkgs";   # avoid duplicate nixpkgs

  crane.url        = "github:ipetkov/crane";
  # do NOT do `crane.inputs.nixpkgs.follows = "nixpkgs"` for crane (it dropped its nixpkgs input)

  home-manager.url = "github:nix-community/home-manager";
  home-manager.inputs.nixpkgs.follows = "nixpkgs";
};
```

Use `nix flake update` to bump everything, `nix flake update nixpkgs` to bump one, `nix flake metadata` to inspect the lock, and `nix flake show` to enumerate outputs. Watch out: as Vladimir Timofeenko documented in his flake anatomy post, **NixOS 24.05 changed `nixpkgs` behavior so the input points to the version used to build the system**, which can break running such a flake on a different machine.

### 2.4 Common flake outputs (the schema in 2026)

| Output | Used by |
|---|---|
| `packages.<sys>.<name>` | `nix build .#name`, `nix shell` |
| `devShells.<sys>.<name>` | `nix develop .#name` |
| `apps.<sys>.<name>` | `nix run .#name` |
| `nixosConfigurations.<host>` | `nixos-rebuild switch --flake .#host` |
| `darwinConfigurations.<host>` | `darwin-rebuild` (nix-darwin) |
| `homeConfigurations.<user>` | `home-manager switch --flake .#user` (standalone HM) |
| `nixosModules.<name>` / `homeManagerModules.<name>` | imported by downstream configs |
| `overlays.<name>` (note: plural) | passed into `pkgs.extend` |
| `lib` | reusable library functions |
| `templates.<name>` | `nix flake init -t .#name` |
| `formatter.<sys>` | `nix fmt` |
| `checks.<sys>.<name>` | `nix flake check` (build farm gating) |

The legacy singular forms (`defaultPackage`, `defaultApp`, `nixosModule`, `overlay`, `devShell`) are deprecated; `nix flake check` will emit deprecation warnings.

---

## 3. Library development

### 3.1 How nixpkgs `lib` is organised

`nixpkgs/lib/default.nix` exposes a single attribute set built with `makeExtensible`, partitioned into categories: `trivial`, `fixedPoints`, `attrsets`, `lists`, `strings`, `stringsWithDeps`, `customisation` (overrides/callPackage), `derivations`, `types`, `modules`, `options`, `meta`, `debug`, `licenses`, `systems`, `path`, `fileset`, `generators`, `kernel`. The attributes are re-exported flat (so `lib.optional` exists even though the implementation lives in `lib/lists.nix`).

The `lib` attribute set is itself an extensible fixed-point: you can `lib.extend (final: prev: { myFn = …; })` to get a patched library — but the source comment is blunt about this being an escape hatch:

> "This functionality is intended as an escape hatch for when the provided version of the Nixpkgs library has a flaw. If you were to use it to add new functionality, you will run into compatibility and interoperability issues." — `nixpkgs/lib/default.nix`

NotAShelf's blog post *When, Why and How to Extend Nixpkgs' Standard Library* shows the canonical extension pattern: factor it out into `lib/default.nix`, pass it through `specialArgs` to NixOS configs. Pass new helpers into `_module.args.mylib` for module consumption rather than mutating `lib` globally.

### 3.2 `fix`, `extends`, `makeExtensible`

```nix
# Verbatim form from nixpkgs/lib/fixed-points.nix
fix = f: let x = f x; in x;

# extends: apply an overlay-shaped function to a fixed-point function
extends = overlay: rattrs: final:
  let prev = rattrs final; in prev // overlay final prev;

# makeExtensible: turn a fixed-point function into an attrset with .extend
obj = lib.makeExtensible (final: { foo = "foo"; });
# nix-repl> obj.extend (final: prev: { bar = prev.foo + "bar"; }))
```

This is the same mechanism nixpkgs `pkgs` uses internally (`final: prev:` overlays). Use it when building a library that you want users to be able to override piecewise.

### 3.3 Documentation conventions (RFC 145)

RFC 145, merged in November 2023 (per the treewide migration PR #262987's reference "hsjobeki mentioned this pull request · Nov 13, 2023"), and implemented in Nix via PR #11072, defines a doc-comment syntax: `/** … */` (note the **double asterisk**) before a binding, with CommonMark Markdown inside. `nixdoc` and Noogle consume these. The standard form for a library function in nixpkgs today:

```nix
{
  /**
    Concatenate two lists with a separator.

    # Inputs

    `sep` (string)
    : The separator.

    `xs` (list of strings)
    : The list.

    # Example

    ```
    intercalate ", " [ "a" "b" "c" ]
    => "a, b, c"
    ```

    # Type

    ```
    intercalate :: String -> [String] -> String
    ```
  */
  intercalate = sep: xs: /* … */;
}
```

`nix repl` renders these via `:doc lib.intercalate`. Legacy single-asterisk comments are auto-migrated by `nixdoc` but new code should always use `/** */`.

### 3.4 `lib.makeOverridable`

```nix
mkFoo = lib.makeOverridable ({ a, b }: { result = a + b; });

# Now:  (mkFoo { a = 1; b = 2; }).override { b = 10; }
#   =>  { result = 11; …; override = …; }
```

This is the mechanism behind `pkg.override { … }`. Wrap your library entry points in `makeOverridable` so consumers can tweak inputs without rebuilding from source.

---

## 4. Derivations and packaging

### 4.1 The modern idiom: `mkDerivation (finalAttrs: …)`

The single biggest change in derivation idioms since 2022 is the **`finalAttrs` pattern**, introduced in nixpkgs PR #119942 by Robert Hensing (merged into the 22.05 release). It replaces `rec` inside `mkDerivation`. Quoting the PR motivation verbatim:

> "Allows to get rid of confusing `rec { }` in `mkDerivation` calls. Overriding an attribute in this style causes derived attributes to be overridden as expected. … Fixes #119407, the problem where if you use `overrideAttrs`, the tests in `passthru` still use the old package."

The Nixpkgs manual (Stdenv chapter, "Fixed-point argument of mkDerivation") spells out the rationale:

> "Note that this does not use the `rec` keyword to reuse `withFeature` in `configureFlags`. The `rec` keyword works at the syntax level and is unaware of overriding. Instead, the definition references `finalAttrs`, allowing users to change `withFeature` consistently with `overrideAttrs`. `finalAttrs` also contains the attribute `finalPackage`, which includes the output paths, etc."

```nix
# Canonical 2026 form
{ lib, stdenv, fetchFromGitHub, testers }:

stdenv.mkDerivation (finalAttrs: {
  pname   = "hello";
  version = "2.12.1";

  src = fetchFromGitHub {
    owner = "owner";
    repo  = "hello";
    rev   = "v${finalAttrs.version}";   # self-reference, replaces rec
    hash  = "sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=";
  };

  # passthru can now reference the final package itself (fixes #119407):
  passthru.tests.version = testers.testVersion { package = finalAttrs.finalPackage; };

  meta = {
    description = "GNU Hello";
    license     = lib.licenses.gpl3Plus;
    mainProgram = "hello";
    maintainers = with lib.maintainers; [ ];
  };
})
```

**Footgun (issue #310373):** if you put `finalAttrs.version` inside `src.rev`, overriding `version` will not change `hash`, so the fetch will fail with a hash mismatch. Override `src` whenever you override `version`.

### 4.2 `callPackage`, `.override`, `.overrideAttrs`

```nix
# default.nix
let
  pkgs = import <nixpkgs> {};
in {
  hello       = pkgs.callPackage ./hello.nix { };                        # auto-thread deps
  hello-clang = pkgs.hello.override     { stdenv = pkgs.clangStdenv; };  # change inputs
  hello-patched = pkgs.hello.overrideAttrs (prev: {                      # change attrs
    patches = (prev.patches or []) ++ [ ./my.patch ];
  });
}
```

The mental model: `callPackage` does `f (builtins.intersectAttrs (functionArgs f) pkgs // args)`. `.override` re-runs `callPackage` with different args. `.overrideAttrs` re-runs `mkDerivation` with mutated attrs. Use `overrideAttrs (finalAttrs: previousAttrs: …)` for the two-argument form when you need both the originals and the post-override result.

### 4.3 Fetchers (which to use)

- `fetchFromGitHub` / `fetchFromGitLab` / `fetchFromSourcehut`: **first choice** for VCS sources — they produce a `.tar.gz` of the tree at a revision, hash is stable across git versions.
- `fetchgit`: only when you need submodules (`fetchSubmodules = true;`) or non-GitHub hosts not covered by a `fetchFrom*`.
- `fetchurl` / `fetchzip`: tarballs and zips. `fetchzip` strips one directory level by default.
- Always use SRI hashes: `hash = "sha256-…";` (not `sha256 = "…";`).
- For local sources in flakes: `src = ./.;` is fine, but consider `src = lib.fileset.toSource { root = ./.; fileset = lib.fileset.unions [ ./src ./Cargo.toml ./Cargo.lock ]; };` to avoid rebuilding when unrelated files change.

### 4.4 Language-specific builders (brief survey)

**Rust** — three options, all valid:
- `rustPlatform.buildRustPackage { src = …; cargoHash = "sha256-…"; }`: the **nixpkgs standard**, one derivation for the whole build, no IFD. Use this for anything going upstream.
- `crane` (Ivan Petkov): splits dependency build from source build, drastically faster incremental builds. Tweag's 22 September 2022 blog post *Building Nix flakes from Rust workspaces* concluded: *"Crane does seem to be an improvement over naersk, and because it delegates all the hard parts to Cargo, it can stay clear of all the problems that cargo2nix will need to handle."* Crane has since become the dominant choice for serious Rust+Nix projects.
- `naersk`: simpler than crane, still maintained, similar dependency-caching idea. Has compatibility issues with workspaces and non-default targets.

Pair any of them with `fenix` or `rust-overlay` to pin the exact rustc/cargo toolchain.

**Python** — the active debate:
- `buildPythonApplication` / `buildPythonPackage` from nixpkgs: works, but `nixpkgs`' python set is monolithic.
- `poetry2nix`: still works, but the original maintainer @adisbladis has explicitly stepped back, writing in issue #1865: "**My personal recommendation is to use uv & uv2nix instead**."
- `uv2nix` (built on `pyproject.nix`): the recommended successor for new projects. Same author. Still marked experimental in late 2025.
- `pyproject.nix`: lower-level building block for both poetry and uv tooling.

**Go**: `buildGoModule { src = …; vendorHash = "sha256-…"; }`. Set `vendorHash = lib.fakeHash;` first, build, copy the suggested hash. For pre-vendored projects, `vendorHash = null;`.

**Node.js**: `buildNpmPackage { src = …; npmDepsHash = "sha256-…"; }` is the nixpkgs standard. For pnpm projects, see `pnpm.fetchDeps` in nixpkgs (added in 2024). `yarn2nix` exists but is increasingly considered legacy.

**Haskell**: `haskellPackages.callCabal2nix` for Cabal projects; `haskellPackages.callPackage` for files already converted by `cabal2nix`.

### 4.5 Phases and wrappers

Standard phase order: `unpackPhase` → `patchPhase` → `configurePhase` → `buildPhase` → `checkPhase` → `installPhase` → `fixupPhase` → `installCheckPhase` → `distPhase`. Override individual phases with `*Phase = "…";` strings or `pre*`/`post*` hooks. Skip phases by setting them to `":"` (a no-op shell command), or by setting `dontConfigure = true;`/`dontBuild = true;`.

`wrapProgram` (from `makeWrapper`) is the canonical way to inject env vars or `PATH` into an installed binary:

```nix
nativeBuildInputs = [ makeWrapper ];
postInstall = ''
  wrapProgram $out/bin/myapp \
    --prefix PATH : ${lib.makeBinPath [ git curl ]} \
    --set MYAPP_DATA $out/share/myapp
'';
```

### 4.6 Cross-compilation

```nix
# In a flake:
packages.x86_64-linux.myapp-arm = pkgs.pkgsCross.aarch64-multiplatform.myapp;
```

Mark a package's tools (compilers, codegen) as `nativeBuildInputs` and its runtime libs as `buildInputs` — `stdenv` then knows which to build for the build platform vs the host platform. Use `lib.systems.examples` to enumerate cross targets.

### 4.7 `pkgs/by-name` layout (RFC 140)

For new nixpkgs packages and for personal monorepos that mirror nixpkgs conventions, the **RFC 140** structure — merged in 2023 and first enabled in nixpkgs by PR #237439 (merged on 2023-09-05 by infinisil with the Nixpkgs Architecture Team) — is now mandatory in nixpkgs for new top-level packages:

```
pkgs/by-name/he/hello/package.nix         # nixpkgs RFC 140 form
```

The file is a `callPackage`-shaped function (`{ stdenv, fetchurl, … }: stdenv.mkDerivation …`) and is auto-attached at the top level — no edit to `all-packages.nix` needed. The shard prefix (`he` for `hello`) is mandatory.

---

## 5. Overlays and overrides

### 5.1 Form: `final: prev:` (formerly `self: super:`)

```nix
# overlay.nix
final: prev: {
  myFoo = final.callPackage ./pkgs/myFoo {};         # uses overlaid pkgs
  hello = prev.hello.overrideAttrs (old: {            # patch existing
    patches = (old.patches or []) ++ [ ./hello.patch ];
  });
}
```

The rule of thumb from ayats.org's *Nix lectures* is the clearest statement of the `final` vs `prev` discipline:

> "When should you grab something from `prev` and when from `final`? The rule of thumb is the following: Take from `prev` what you want to modify, e.g. `pkg = prev.pkg.overrideAttrs …`. Take from `final` otherwise. As it will account for every modification from every package."

The names changed from `self: super:` to `final: prev:` around 2021 in nixpkgs convention (the older form still works; lint tools won't complain, but new code should use the new names).

### 5.2 Composing overlays

```nix
pkgs = import nixpkgs {
  inherit system;
  overlays = [
    inputs.rust-overlay.overlays.default
    (import ./overlays/myapp.nix)
    (final: prev: { foo = prev.foo.override { withGUI = true; }; })
  ];
};

# Programmatically:
combined = lib.composeManyExtensions [ overlayA overlayB ];
```

Overlays are applied left-to-right; later overlays see earlier ones via `final`. Common pitfall: an overlay that does `foo = final.bar.override { dep = final.foo; }` will recurse infinitely. Always trace one of the `foo`s back to `prev`.

### 5.3 When to use what

- **Use an overlay** when you want a modification to be visible to *every other package* that depends on the modified one (e.g., patching `openssl` system-wide).
- **Use `.override`/`.overrideAttrs`** when you only want a one-off modification for a single derivation.
- **Use a flake input + `callPackage`** when the dependency is conceptually a different project, not a patch to nixpkgs.

---

## 6. NixOS modules — idiomatic patterns

### 6.1 Anatomy

```nix
# modules/myservice.nix
{ config, lib, pkgs, ... }:
let
  cfg = config.services.myservice;   # short alias, conventional name
in {
  options.services.myservice = {
    enable = lib.mkEnableOption "myservice";

    package = lib.mkPackageOption pkgs "myservice" {};   # newer helper, RFC-ish

    port = lib.mkOption {
      type    = lib.types.port;
      default = 8080;
      description = "Port to listen on.";
    };

    settings = lib.mkOption {
      type = lib.types.submodule {
        freeformType = (pkgs.formats.toml {}).type;
        options.log_level = lib.mkOption {
          type    = lib.types.enum [ "debug" "info" "warn" "error" ];
          default = "info";
        };
      };
      default = {};
    };
  };

  config = lib.mkIf cfg.enable {
    systemd.services.myservice = {
      wantedBy = [ "multi-user.target" ];
      serviceConfig.ExecStart = "${cfg.package}/bin/myservice --port ${toString cfg.port}";
    };
    networking.firewall.allowedTCPPorts = [ cfg.port ];
  };
}
```

### 6.2 `mkIf`, `mkMerge`, `mkDefault`, `mkForce` — and when to use plain `if`

The official NixOS wiki page *The Nix Language versus the NixOS Module System* puts the rule plainly:

> "When your condition only references local values, it is okay to use native Nix `if-then` expressions. But if your condition references any part of the configuration tree, it is best to use `lib.mkIf`."

Why: `mkIf` produces `{ _type = "if"; condition = …; content = …; }`, which the module system unwraps during merge. A native `if` on `config.foo.enable` causes infinite recursion because `config` is computed from all modules' `config` blocks — you can't read it while still defining it. Three forms summarised by the NixOS & Flakes Book:

```nix
# ✅ Scenario 1: native if on already-resolved config — works fine
config.warnings = if config.foo then ["foo"] else [];

# ❌ Scenario 2: native if at the top of config — infinite recursion
config = if config.foo then { warnings = ["foo"]; } else {};

# ✅ Scenario 3: mkIf at the top — works fine
config = lib.mkIf config.foo { warnings = ["foo"]; };
```

Priorities (from `lib/modules.nix`): `mkOptionDefault` = 1500, `mkDefault` = 1000, normal = 100, `mkImageMediaOverride` = 60, `mkForce` = 50, `mkVMOverride` = 10. Lower number wins. So **`mkForce` overrides `mkDefault` overrides defaults baked into `mkOption`'s `default`**.

`mkMerge` combines several attrsets so one module can produce conditional + unconditional config in one expression:

```nix
config = lib.mkMerge [
  { environment.systemPackages = [ pkgs.git ]; }
  (lib.mkIf cfg.enable { systemd.services.myservice = …; })
  (lib.mkIf cfg.openFirewall { networking.firewall.allowedTCPPorts = [ cfg.port ]; })
];
```

### 6.3 Types

The canonical types from `lib.types`: `bool`, `int`, `float`, `str`, `path`, `package`, `port`, `enum [ … ]`, `nullOr <t>`, `listOf <t>`, `attrsOf <t>`, `submodule { options = …; }`, `oneOf [<t> …]`, `either <t> <u>`, `coerceTo <t> f <u>`, `unspecified`. Use `pkgs.formats.json {}` / `pkgs.formats.toml {}` / `pkgs.formats.yaml {}` and their `.type` for freeform config files; their `.generate` emits the file.

### 6.4 `specialArgs` vs `_module.args`

The nixpkgs source comment in `lib/modules.nix` is the definitive rule:

> "This should only be used for special arguments that need to be evaluated when resolving module structure (like in `imports`). For everything else, there's `_module.args`."

Practically:
- `specialArgs` (passed to `nixpkgs.lib.nixosSystem`/`evalModules`): usable in `imports = …` because available before module evaluation. Use it for things like `inputs`, `mylib`, `hostName`.
- `_module.args.foo = …`: usable as a module argument (`{ foo, … }:`) but **not** in `imports`.

```nix
nixosConfigurations.myhost = nixpkgs.lib.nixosSystem {
  system = "x86_64-linux";
  specialArgs = { inherit inputs; hostName = "myhost"; };
  modules = [
    ./configuration.nix
    ({ pkgs, _module, ... }: { _module.args.mylib = import ./lib { inherit pkgs; }; })
  ];
};
```

### 6.5 Testing modules with VM tests

The modern idiom is `pkgs.testers.runNixOSTest`, documented in the *Writing Tests* section of the NixOS manual:

> "A NixOS test is a module that has the following structure: `{ nodes = { vm1 = { config, pkgs, ... }: { … }; }; testScript = ''Python code…''; }`. … Outside the `nixpkgs` repository, you can use the `runNixOSTest` function from `pkgs.testers`."

```nix
# tests/nginx.nix
{ lib, ... }:
{
  name = "nginx-smoke";

  nodes.server = { pkgs, ... }: {
    networking.firewall.allowedTCPPorts = [ 80 ];
    services.nginx = {
      enable = true;
      virtualHosts."server".locations."/".return = "200 'hello'";
    };
  };
  nodes.client = { pkgs, ... }: {
    environment.systemPackages = [ pkgs.curl ];
  };

  testScript = ''
    start_all()
    server.wait_for_unit("nginx.service")
    server.wait_for_open_port(80)
    client.succeed("curl --fail http://server/ | grep -F hello")
  '';
}

# flake.nix
checks = forAllSystems (pkgs: {
  nginx-test = pkgs.testers.runNixOSTest ./tests/nginx.nix;
});
```

Run with `nix build .#checks.x86_64-linux.nginx-test -L`, or step through interactively with `nix build .#checks.x86_64-linux.nginx-test.driverInteractive && ./result/bin/nixos-test-driver`. The Python API (`machine.wait_for_unit`, `machine.succeed`, `machine.fail`, `machine.wait_until_succeeds`, `machine.screenshot`, `machine.send_chars`, etc.) is documented inline in the manual.

---

## 7. Dev shells / reproducible environments

### 7.1 `pkgs.mkShell` — the idiomatic baseline

```nix
# shell.nix or devShells.default
{ pkgs }:
pkgs.mkShell {
  # Use `packages` (alias for nativeBuildInputs), NOT buildInputs:
  packages = with pkgs; [ go gopls golangci-lint just ];

  # Inherit build inputs from other derivations (handy for "build my package + extra tools"):
  inputsFrom = [ pkgs.hello ];

  # Environment variables you set declaratively:
  GOFLAGS = "-mod=vendor";

  shellHook = ''
    echo "Welcome to the foo dev shell"
    export PROJECT_ROOT=$PWD
  '';
}
```

The footgun ayats.org documents: `buildInputs` does not run setup hooks for tools you're going to *use*, only for libraries you're going to *link against*. Always use `packages` (or `nativeBuildInputs` — they're the same option) for tools.

### 7.2 `direnv` + `nix-direnv`

In `~/.config/direnv/direnvrc`, add `source $HOME/.nix-profile/share/nix-direnv/direnvrc` (or install `nix-direnv` via Home Manager). Then in each project:

```bash
# .envrc
use flake          # or `use flake .#myShell`
```

`nix-direnv` caches the shell evaluation, so re-entering the directory is near-instant. CI and contributor onboarding then both just require Nix + direnv installed.

### 7.3 Alternatives: `devenv`, `numtide/devshell`

- **`numtide/devshell`** (zimbatm): replaces `stdenv.mkDerivation` with `builtins.derivation` to avoid C-toolchain pollution; ships a TOML-based config and a `menu` of project commands. Great for polyglot teams who don't want to write Nix.
- **`cachix/devenv`**: comprehensive — language toolchains, services (Postgres, Redis), `process-compose` integration, optional containerisation. Heavier (large input closure) but the highest ergonomics.
- **plain `mkShell`**: simplest, smallest closure, integrates trivially. Recommended for libraries / nixpkgs contributions.

### 7.4 Reproducible toolchains

- **Rust**: `rust-overlay` or `fenix`. Both expose pinned toolchains. Combine with `crane` for incremental builds:
  ```nix
  let
    toolchain = pkgs.rust-bin.fromRustupToolchainFile ./rust-toolchain.toml;
    craneLib  = (crane.mkLib pkgs).overrideToolchain toolchain;
  in craneLib.buildPackage { src = craneLib.cleanCargoSource ./.; }
  ```
- **Python**: `uv2nix` is the recommended path forward (per @adisbladis above); `poetry2nix` if you're already on Poetry.
- **Node**: `pnpm` via `pkgs.pnpm.fetchDeps` is the cleanest; `buildNpmPackage` for npm; `corepack` from nixpkgs handles yarn.

---

## 8. Tooling and workflow idioms

### 8.1 The CLI commands you'll use daily

| Command | Purpose |
|---|---|
| `nix develop` | Enter `devShells.default` |
| `nix develop .#foo` | Enter `devShells.foo` |
| `nix build .#foo` | Build `packages.<sys>.foo`; output at `./result` |
| `nix run .#foo -- arg1 arg2` | Run `apps.<sys>.foo` |
| `nix shell nixpkgs#hello` | Ephemeral env with `hello` on PATH |
| `nix flake check` | Build all `checks`, evaluate all outputs |
| `nix flake show` | List all outputs |
| `nix flake update` | Bump every input; `nix flake update foo` bumps one |
| `nix flake metadata` | Inspect the lock |
| `nix fmt` | Run `formatter.<sys>` |
| `nix repl` | REPL; use `:lf .` to load a flake's outputs |
| `nix eval --raw .#foo` | Evaluate without building |

### 8.2 Formatting: nixfmt is now official

RFC 166 was accepted on 4 March 2024 (NixOS Discourse FCP announcement: *"Barring any blocking issues it will be merged in 13 days (the next time the steering committee meets) on 2024-03-04T23:00:00Z (UTC)"*), establishing `nixfmt` as the official formatter, donated by Serokell to NixOS. The implementation tracking issue (NixOS/nixfmt#153), opened 12 March 2024, opens with *"Now that RFC 166 is accepted, let's commence with the implementation!"* The nixfmt README states the current status:

> "Nixfmt is the official formatter for Nix language code. … nixfmt was originally developed by Serokell and later donated to become an official Nix project with the acceptance of RFC 166."

Naming today:
- `pkgs.nixfmt-rfc-style` — the RFC 166–compliant style. **The one to use until 25.11.**
- `pkgs.nixfmt-classic` — the old pre-RFC style, kept around for legacy codebases.
- After NixOS **25.11**, `pkgs.nixfmt` *is* the RFC-style formatter. The NixOS 25.11 release notes confirm: *"The official Nix formatter nixfmt is now stable and available as `pkgs.nixfmt`, deprecating the temporary `pkgs.nixfmt-rfc-style` attribute. The classic nixfmt will stay available for some more time as `pkgs.nixfmt-classic`."* v1.0.0 of nixfmt was the corresponding upstream release.
- `pkgs.nixfmt-tree` — pre-wrapped with treefmt for whole-tree formatting.

Competitors that still have adherents: `alejandra` (opinionated, fast, predates the RFC) and `nixpkgs-fmt` (now deprecated). New projects should pick `nixfmt`.

Set it up in a flake:

```nix
formatter.${system} = pkgs.nixfmt-rfc-style;     # or pkgs.nixfmt on ≥25.11
                                                  # or pkgs.nixfmt-tree for `nix fmt .`
```

### 8.3 Linting and dead-code detection

- **`statix`** flags antipatterns (`with` in suspicious places, redundant `rec`, useless `let`, manual inherits, etc.). `statix check .` to lint, `statix fix .` to auto-fix.
- **`deadnix`** flags unused `let` bindings and unused function arguments. `deadnix .` to check, `deadnix -e .` to edit in place.
- **`nil`** or **`nixd`** as your LSP for editor integration.

### 8.4 `treefmt-nix` + `git-hooks.nix`

The de facto multi-formatter glue (`treefmt-nix`) and the de facto pre-commit framework (`git-hooks.nix`, formerly `pre-commit-hooks.nix`) are both flake-parts modules:

```nix
imports = [
  inputs.treefmt-nix.flakeModule
  inputs.git-hooks.flakeModule
];

perSystem = { config, pkgs, ... }: {
  treefmt = {
    projectRootFile = "flake.nix";
    programs.nixfmt.enable = true;
    programs.deadnix.enable = true;
    programs.prettier.enable = true;
  };

  pre-commit.settings.hooks = {
    nixfmt.enable  = true;
    statix.enable  = true;
    deadnix.enable = true;
    treefmt.enable = true;       # runs treefmt as a hook
  };

  devShells.default = pkgs.mkShell {
    shellHook = config.pre-commit.installationScript;
    packages  = config.pre-commit.settings.enabledPackages;
  };

  checks.formatting = config.treefmt.build.check self;
};
```

### 8.5 CI patterns

A minimal GitHub Actions workflow:

```yaml
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: DeterminateSystems/nix-installer-action@main   # or cachix/install-nix-action
      - uses: DeterminateSystems/magic-nix-cache-action@main # or cachix/cachix-action
      - run: nix flake check -L
```

For a multi-system build matrix, use `garnix.io` (a hosted Nix-aware CI), `Hercules CI`, or a GHA matrix on `ubuntu-latest` + `macos-latest`. For binary caching:

- **Cachix**: easiest, hosted, pay-per-GB.
- **attic** (zhaofengli/attic): self-hosted, multi-tenant, S3-backed.
- **FlakeHub Cache** (Determinate Systems): integrates with their flake registry.

---

## 9. Common anti-patterns and how to avoid them

| Anti-pattern | Why bad | Fix |
|---|---|---|
| `rec { … }` in a derivation | Doesn't survive overrides | `mkDerivation (finalAttrs: { … })` |
| `with pkgs;` at file top | Breaks tooling, scoping surprises | `let inherit (pkgs) a b c; in …` |
| `import ./foo.nix { inherit pkgs; }` | Tight coupling, no `.override`, no auto-threading | `pkgs.callPackage ./foo.nix {}` |
| `services.foo.bar = if config.x then … else …` at top of `config` | Infinite recursion | `config = lib.mkIf config.x { services.foo.bar = …; };` |
| `config = nixpkgs.lib.recursiveUpdate {…} {…}` in a module | Bypasses module merging, breaks priorities | Use `mkMerge` and let the system merge |
| `nixosModule = self: { … }` (singular, old) | Deprecated | `nixosModules.default = ./module.nix;` (plural) |
| Using `<nixpkgs>` | Depends on `NIX_PATH` | Flake input or `fetchTarball` pinned by hash |
| IFD in nixpkgs contributions | Banned by CI | Pre-generate lock-derived Nix, or use a builder that avoids IFD |
| `buildInputs = [ pkgs.git ];` in `mkShell` | Hooks for tools don't fire | `packages = [ pkgs.git ];` |
| Unbounded `overlays` chain referencing `final` | Infinite recursion | Take what you modify from `prev` |
| `let pkgs = import nixpkgs { system = "x86_64-linux"; }` baked into a flake | Won't build on Darwin/ARM | Use `forAllSystems` / `perSystem` |

---

## 10. Community resources & the current state

**Authoritative docs**
- nix.dev (official tutorials & best-practices) — https://nix.dev
- Nixpkgs manual — https://nixos.org/manual/nixpkgs/stable/
- NixOS manual — https://nixos.org/manual/nixos/stable/
- Official NixOS wiki — https://wiki.nixos.org (the new official wiki; `nixos.wiki` is the older mirror)
- flake-parts docs — https://flake.parts
- Zero-to-Nix — https://zero-to-nix.com (Determinate Systems' onboarding)
- Determinate Systems blog — https://determinate.systems/posts/
- Tweag blog — https://tweag.io/blog/
- Domen Kožar's writings (cachix author, devenv)
- Jade Lovelace (lf-) — https://jade.fyi
- Robert Hensing — flake-parts author, nixpkgs maintainer
- Eelco Dolstra — original Nix author, runs the architecture conversations

**Notable libraries (with their idiomatic role)**
- `nixpkgs.lib` — universal library
- `flake-parts` — flake module system
- `flake-utils` — older flake helpers (still works)
- `home-manager` — user-level config
- `disko` — declarative disk partitioning for NixOS installs
- `numtide/devshell` — TOML-driven dev environments
- `cachix/devenv` — batteries-included dev environments
- `cachix/git-hooks.nix` (formerly `pre-commit-hooks.nix`) — pre-commit framework
- `numtide/treefmt-nix` — multi-formatter
- `nix-systems` — extensible system lists
- `ipetkov/crane` — Rust builder
- `nix-community/fenix` / `oxalica/rust-overlay` — Rust toolchains
- `pyproject-nix/uv2nix` — Python via uv

**Recent RFCs & milestones**
- **RFC 140** (merged 2023; first nixpkgs implementation, PR #237439, merged on 2023-09-05 by infinisil with the Nixpkgs Architecture Team): the `pkgs/by-name` directory structure in nixpkgs. CI now enforces it for new top-level packages.
- **RFC 145** (merged November 2023, per the treewide migration PR #262987's reference *"hsjobeki mentioned this pull request · Nov 13, 2023"*; implemented in Nix via PR #11072): doc-comments — `/** … */` Markdown comments parsed by Nix and rendered in `nix repl`.
- **RFC 166** (accepted 2024-03-04, per the NixOS Discourse FCP announcement; tracking issue NixOS/nixfmt#153 opened 12 March 2024): nixfmt becomes the official formatter; v1.0.0 stabilised and shipped as `pkgs.nixfmt` in NixOS 25.11 (per the release notes), with the old style preserved as `nixfmt-classic`.
- **`mkDerivation (finalAttrs: …)`** (PR #119942, opened by Robert Hensing on 2 May 2021, merged for NixOS 22.05): replaces `rec` inside derivations.

---

## Recommendations

**Stage 1 — Start a new project (today, in 2026):**
1. `nix flake init -t github:hercules-ci/flake-parts#bare` (or hand-roll a minimal flake if you have ≤3 outputs).
2. Add `nixfmt-rfc-style` (or `nixfmt` on ≥25.11), `statix`, `deadnix` via `treefmt-nix` and wire as `formatter.<sys>` + a `check`.
3. Add `git-hooks.nix` with `nixfmt`/`statix`/`deadnix` hooks; `shellHook` installs them.
4. `direnv allow` with `use flake` in `.envrc`.

**Stage 2 — Packaging:**
1. Write `package.nix` as a `callPackage`-shaped function (`{ stdenv, fetchFromGitHub, … }: …`).
2. Use `stdenv.mkDerivation (finalAttrs: …)`; avoid `rec`.
3. Use `fetchFromGitHub` with `hash = "sha256-…";` (SRI form).
4. Add `meta.mainProgram` for `nix run`; add `passthru.tests = { … };` referencing `finalAttrs.finalPackage`.
5. If you're going upstream: place in `pkgs/by-name/<sh>/<name>/package.nix`.

**Stage 3 — NixOS modules:**
1. Always: `options.X = …; config = lib.mkIf cfg.enable { … };`.
2. `mkOption` with a real type from `lib.types` (not `types.unspecified`).
3. For freeform config files: `pkgs.formats.{json,toml,yaml} {}`.
4. Write at least one `pkgs.testers.runNixOSTest` test per module, exposed as `checks.<sys>.<module>-test`.
5. Pass non-`imports` data via `_module.args`, `imports`-required data via `specialArgs`.

**Stage 4 — Production / multi-developer:**
1. Set up a binary cache (Cachix for small teams, attic or FlakeHub for larger).
2. CI uses `nix flake check -L` plus `cachix watch-exec` (or attic's equivalent) to push closures.
3. Update inputs on a schedule (a weekly bot PR running `nix flake update`).
4. Use `flake-parts` once you have ≥3 modules in your flake or any cross-system output beyond `packages`/`devShells`.

**Triggers to change course:**
- **If your flake exceeds ~150 lines** → migrate to `flake-parts` with per-file modules.
- **If you're hitting evaluation > 10 s** → audit for IFD; consider `lazy-trees` (Determinate Nix) or splitting flakes.
- **If you're maintaining a library used by others** → adopt `lib.makeExtensible`, RFC 145 doc-comments, and `lib.makeOverridable` wrappers; expose a `lib` flake output.
- **If you're contributing to nixpkgs** → switch off IFD, use `pkgs/by-name`, run `nixpkgs-review` before submitting.

---

## Caveats

- **Flakes remain "experimental" in upstream Nix.** Determinate Nix ships them as stable; if you use upstream Nix, you must enable `experimental-features = nix-command flakes` in `nix.conf`. The community has been operating as if they were stable since 2022, but the CLI surface still occasionally changes (e.g., `nix profile` is being reworked, the `nix flake` output schema is in the process of being formalised via flake schemas).
- **There is genuine community disagreement** on (a) flake-utils vs flake-parts vs hand-rolled, (b) `nixfmt` vs `alejandra` (alejandra is faster and many find its output more readable, but `nixfmt` is now official), and (c) whether flakes should be encouraged at all for nixpkgs-style libraries (Jade Lovelace's position). The recommendations here reflect majority practice in late-2025 Determinate / Hercules / Nixpkgs-maintainer circles.
- **The `nixos.wiki` → `wiki.nixos.org` migration** has split documentation; the official one is `wiki.nixos.org`, but `nixos.wiki` is still online and search engines often return it. Prefer `wiki.nixos.org` when both are available.
- **`poetry2nix` is in caretaker mode**; its original maintainer has explicitly recommended `uv2nix` for new projects. `uv2nix` is still marked experimental.
- **IFD bans vary**: nixpkgs bans IFD on its CI; for personal projects, IFD is fine and often necessary (crane, dream2nix). Know which world you're in.
- **`pkgs/by-name` does not work in every case**: packages requiring code shared across multiple versions, or that aren't called via plain `callPackage` (e.g., Python package set members), still live in the old hierarchy. CI's `nixpkgs-vet` enforces specific allowed shapes.
- **The `finalAttrs.version` in `src.rev` footgun** (issue #310373) is real: overriding `version` does not change the hash, so the override fails. Override `src` whenever you override `version`, or accept that `version` is only authoritative when it isn't being overridden.
