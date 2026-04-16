## 1. Executive Summary

This document specifies a single, opinionated infrastructure architecture for a greenfield company rewrite, built entirely on Nix. The architecture uses **Nix flakes as the universal interface**, **NixOS as the sole server operating system**, **NixOS service hosts (not Kubernetes)** as the production substrate, **whole-system NixOS closures as the deployment artifact**, a **tiered binary cache** (local → org S3 → upstream), **agenix for secrets**, and a **promotion-based deployment model** using immutable system profiles with instant rollback via `nix-env --switch-generation` / `nixos-rebuild switch --rollback`.

Kubernetes is rejected. Containers are rejected as a deployment primitive. The entire stack — from developer laptops to production — speaks one language: Nix.

---

## 2. Restated Design Brief

Design the single best company-agnostic, pure-Nix infrastructure architecture for a greenfield rewrite under these conditions:

- Linux only, all environments
- Every engineer is a Nix expert
- Effectively infinite infrastructure budget
- Aggressive willingness to standardize
- Optimize for correctness, scalability, operability, and reproducibility
- Do not optimize for familiarity, onboarding, or incremental migration

---

## 3. Assumptions

1. **Team**: 20–200 engineers, all fluent in Nix, comfortable with NixOS module system.
2. **Scale**: Tens to low hundreds of distinct services; tens to low thousands of hosts.
3. **Workloads**: Typical SaaS — HTTP services, async workers, databases, caches, queues.
4. **Cloud**: Any IaaS provider (AWS, GCP, Hetzner, bare metal). Architecture is provider-agnostic.
5. **Languages**: Polyglot (Rust, Go, Python, TypeScript, etc.). All built through Nix.
6. **Compliance**: SOC 2 / ISO 27001 level auditability assumed as a baseline requirement.
7. **State**: Stateful services (Postgres, Redis, Kafka) run on dedicated NixOS hosts or managed services; they are not modeled as immutable deployments.
8. **No legacy**: Zero brownfield constraints. Every decision optimizes for the end state.

---

## 4. Architecture Principles

1. **One language, one tool**: Nix is the build system, package manager, configuration manager, and deployment tool. There is no Ansible, no Terraform for host configuration, no Docker, no Helm.
2. **Whole-system closures**: The deployment artifact is not a container image. It is a NixOS system closure — a complete, bootable, self-contained description of a machine, pinned to exact store paths.
3. **Reproducibility is non-negotiable**: Every build, everywhere, must produce the same output. Flakes enforce this with lockfiles. Impure builds are forbidden in CI and production.
4. **Promotion over mutation**: Artifacts are built once, tested, and promoted through environments. Nothing is rebuilt at deploy time. Nothing is mutated in place.
5. **Rollback is instant**: Every deployment is a new NixOS system profile generation. Rollback is switching to the previous generation. This is an atomic symlink swap, not a redeploy.
6. **Secrets are separate from derivations**: Secrets never enter the Nix store. They are encrypted at rest in the repo and decrypted on-host at activation time.
7. **The repo is the source of truth**: The entire system — services, hosts, networks, policies — is declared in a single monorepo. `git log` is the audit trail. `git diff` is the change review.

---

## 5. Final Recommended Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        MONOREPO (flake)                     │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌───────────┐  │
│  │ services/│  │  hosts/  │  │  modules/ │  │ overlays/ │  │
│  │          │  │          │  │           │  │           │  │
│  │ api/     │  │ prod-web │  │ base.nix  │  │ pins.nix  │  │
│  │ worker/  │  │ prod-wrk │  │ harden.nix│  │ patches/  │  │
│  │ frontend/│  │ staging/ │  │ monitor.nix│ │           │  │
│  └──────────┘  └──────────┘  └───────────┘  └───────────┘  │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐                 │
│  │ secrets/ │  │  tests/  │  │   lib/    │                 │
│  │ (agenix) │  │          │  │           │                 │
│  │ *.age    │  │ nixos-   │  │ mkService │                 │
│  │ keys/    │  │ tests    │  │ mkHost    │                 │
│  └──────────┘  └──────────┘  └───────────┘                 │
└─────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
   ┌───────────┐      ┌────────────┐      ┌─────────────┐
   │  Remote   │      │  Binary    │      │  Deployment  │
   │  Builders │      │  Cache     │      │  (colmena)   │
   │  (x86+arm)│      │  (S3+local)│      │              │
   └───────────┘      └────────────┘      └─────────────┘
```

---

## 6. Recommended Stack

| Concern                | Choice                          | Notes                                    |
| ---------------------- | ------------------------------- | ---------------------------------------- |
| Build system           | Nix (flakes, pure eval)         | Universal, all languages                 |
| OS                     | NixOS (unstable, pinned)        | Every host, every environment            |
| Deployment substrate   | NixOS service hosts             | No Kubernetes, no containers             |
| Deployment artifact    | NixOS system closure            | `/nix/store/...-nixos-system`            |
| Deployment tool        | Colmena                         | Parallel, fleet-aware NixOS deploy       |
| Secret management      | agenix                          | age-encrypted, in-repo, host-key decrypt |
| Binary cache           | Attic (self-hosted S3-backed)   | Tiered: local → org → upstream           |
| Remote builders        | Dedicated NixOS build machines  | Hydra or buildbot for CI orchestration   |
| CI orchestration       | Hydra                           | Nix-native, jobset-per-flake-output      |
| Monitoring             | Prometheus + Grafana + Loki     | NixOS modules, declarative               |
| Networking/firewall    | NixOS firewall + WireGuard      | Declarative, module-based                |
| Infrastructure provisioning | Terraform (provider-only)  | Provisions VMs/networks only; see §16    |
| Repo structure         | Single monorepo flake           | One lockfile, one truth                  |
| Testing                | NixOS VM tests                  | Full integration tests in Nix            |

---

## 7. Detailed System Design

### 7.1 Flakes Everywhere or Not

**Recommendation**: Flakes everywhere. No exceptions.

**Rationale**: Flakes provide the three things this architecture demands: hermetic evaluation (no `NIX_PATH`, no channels, no ambient state), a lockfile (`flake.lock`) that pins every input to an exact revision, and a standardized interface (`packages`, `nixosConfigurations`, `checks`, `devShells`) that tooling can rely on.

**Tradeoffs**: Flakes are still technically "experimental." The CLI is stable in practice. The eval caching is imperfect. `follows` semantics can be confusing. None of these matter for an expert team on a greenfield.

**Failure modes**: Lock drift if `flake.lock` is not updated deliberately. Mitigated by: automated weekly PRs to bump inputs, CI that builds against both current lock and latest to detect breakage early.

**Why alternatives lose**: Classic Nix (channels + `NIX_PATH`) is ambient, mutable, and unreproducible across machines. `niv` solves pinning but doesn't give you the standardized output schema. Flakes are the only option that gives you both.

### 7.2 NixOS Everywhere or Not

**Recommendation**: NixOS on every machine — developer laptops, CI builders, staging, production. No Ubuntu. No Alpine. No "Nix-on-Debian."

**Rationale**: NixOS is the only OS where the entire system configuration — kernel, services, users, firewall, mounts, cron, everything — is a single Nix expression. This means:
- A host is fully described by its `nixosConfiguration`.
- You can build a host's entire closure on any machine and push it.
- You can test a host's configuration in a VM using `nixos-test`.
- Rollback is switching a symlink.

**Tradeoffs**: Some vendor agents (Datadog, CrowdStrike) assume systemd unit files in `/etc/systemd`. NixOS puts them in the store. This requires writing NixOS modules for these agents. With an expert team this is a one-time cost.

**Failure modes**: An engineer installs something imperatively (`nix-env -i` on a server). Mitigated by: disabling `nix-env` on production hosts, enforcing that all changes go through the repo.

**Why alternatives lose**: Running Nix on Ubuntu means you manage two systems — apt for the base, Nix for your software. You lose atomic rollback of the whole system. You lose declarative firewall, users, mounts. You lose `nixos-test`. You halve the value of Nix.

### 7.3 Kubernetes vs NixOS Service Hosts

**Recommendation**: NixOS service hosts. No Kubernetes.

**Rationale**: Kubernetes solves problems that NixOS already solves — and solves them worse for this architecture. Specifically:

| Concern             | Kubernetes                         | NixOS service hosts                          |
| ------------------- | ---------------------------------- | -------------------------------------------- |
| Reproducibility     | Container images (layer cache, mutable tags) | Nix store paths (content-addressed, exact)   |
| Rollback            | Redeploy previous image (slow, may fail) | `switch-to-configuration` to prev generation (instant, atomic) |
| Service config      | YAML manifests + Helm + Kustomize  | NixOS modules (typed, composable, same language as build) |
| Secrets             | etcd (encrypted maybe), external-secrets | agenix (age-encrypted, in-repo, host-key decryption) |
| Networking          | CNI plugins, service mesh, ingress controllers | WireGuard + NixOS firewall + nginx/HAProxy modules |
| Health checks       | Liveness/readiness probes (reimplemented per-service) | systemd watchdog + socket activation         |
| Resource limits     | cgroups via container runtime      | systemd cgroups (same mechanism, less indirection) |
| Scheduling          | kube-scheduler                     | Explicit host assignment or simple fleet tooling |
| Observability       | Prometheus operator, Loki operator | Prometheus + Loki NixOS modules (same tools, declarative) |

Kubernetes adds a massive operational surface (etcd, API server, kubelet, container runtime, CNI, CSI, ingress controller, cert-manager, etc.) that provides no value when your deployment artifact is already a reproducible, atomic, rollbackable NixOS closure.

**Tradeoffs**: You lose automatic bin-packing and horizontal pod autoscaling. For bin-packing: at infinite budget, over-provision. For autoscaling: use systemd `DynamicUser=` + a simple load-balancer-aware scaler, or provision new NixOS VMs via cloud APIs (see §16). You also lose the "run anything in a container" escape hatch — every workload must be Nix-packaged. This is a feature, not a bug.

**Failure modes**: Scheduling becomes manual at scale. Mitigated by: Colmena's tagging system for fleet segmentation, and a thin custom scheduler for auto-provisioning if needed at 1000+ hosts.

**Why alternatives lose**: Kubernetes is a platform for running containers. If your deployment artifact is not a container, Kubernetes is overhead. You would be building NixOS closures, stuffing them into a container image (or running NixOS inside a VM inside a pod — yes, people do this), and then managing the Kubernetes cluster itself — which is a NixOS-managed fleet of machines anyway. You pay twice.

Nomad is closer — it can run raw binaries — but it still adds a scheduler, a gossip protocol, and a state store that duplicate what NixOS + Colmena already provide. It's a less-bad Kubernetes, not a good NixOS.

### 7.4 Deployment Artifact

**Recommendation**: The NixOS system closure. Specifically, the store path of the `config.system.build.toplevel` derivation for a given host's `nixosConfiguration`.

This is a self-contained directory tree in `/nix/store` that includes the kernel, initrd, systemd units, all service binaries, all configuration files, and the activation script. Deploying means copying this closure to the target host's Nix store and running `switch-to-configuration switch`.

**Rationale**: This is the most complete artifact possible. It is not "a binary" or "a container image" — it is the entire machine state, minus secrets and persistent data. Two hosts with the same closure are identical.

**Tradeoffs**: Closures are large (1-5 GB) because they include everything. Binary cache and store deduplication mitigate this — most of the closure is shared across hosts and cached.

**Failure modes**: A closure that passes CI but breaks in production due to secret misconfiguration or missing hardware drivers. Mitigated by: NixOS VM tests that simulate the full host (see §9), and canary deploys.

### 7.5 Remote Builders

**Recommendation**: Dedicated NixOS build machines, 2-4 per architecture (x86_64-linux, aarch64-linux), accessible via `nix build --builders` / Hydra's distributed build support.

**Architecture**:
```
Developer laptop ──► Remote builder pool (via nix distributed build)
                            │
Hydra CI ──────────► Same builder pool (Hydra native support)
                            │
                            ▼
                     Binary cache (Attic on S3)
```

**Configuration**: Builders are declared in the monorepo as NixOS configurations. They are themselves deployed via Colmena. The `nix.buildMachines` setting on developer machines and Hydra points to them.

```nix
nix.buildMachines = [
  {
    hostName = "builder-01.internal";
    systems = [ "x86_64-linux" "i686-linux" ];
    maxJobs = 64;
    speedFactor = 10;
    supportedFeatures = [ "big-parallel" "kvm" "nixos-test" ];
    sshKey = "/etc/nix/builder-key";
  }
];
nix.distributedBuilds = true;
```

**Rationale**: Centralizing builds ensures that every derivation is built exactly once and cached. Developers never wait for a large build locally if the cache has it.

**Tradeoffs**: Single point of failure if builders go down. Mitigated by: multiple builders behind a round-robin, and developers can always fall back to local builds.

**Failure modes**: Builder disk fills up. Mitigated by: `nix.gc.automatic = true` with aggressive `nix.gc.options = "--delete-older-than 14d"`, monitoring on store size.

### 7.6 Binary Cache Layering

**Recommendation**: Three-tier cache:

1. **Local Nix store** (`/nix/store`): On every machine. Hits are free.
2. **Organization cache** (Attic on S3): Self-hosted, S3-backed, content-addressed. Every CI build pushes here. Every developer and production host pulls from here.
3. **Upstream cache** (`cache.nixos.org`): For nixpkgs derivations. Used as a substituter behind the org cache.

```nix
nix.settings = {
  substituters = [
    "https://cache.internal.company.com"  # Org cache (Attic)
    "https://cache.nixos.org"             # Upstream
  ];
  trusted-public-keys = [
    "company-cache:AAAA..."
    "cache.nixos.org-1:BBBB..."
  ];
};
```

**Rationale**: The org cache is the critical layer. It ensures that production deploys never rebuild from source, developer `nix build` commands hit cache for anything CI has already built, and the entire company shares build results.

**Tradeoffs**: Running Attic is operational overhead. Mitigated by: Attic itself is a NixOS service declared in the monorepo and deployed via Colmena. S3 storage is cheap. Attic supports garbage collection.

**Failure modes**: Cache poisoning (someone pushes a bad derivation). Mitigated by: only CI (Hydra) has push credentials. Developers are read-only. Signing keys are per-environment; production cache is separate from dev cache.

**Why not Cachix**: Cachix is a hosted Attic-like service, fine for open-source. For a company with compliance requirements, self-hosted Attic on your own S3 keeps data in your VPC and gives you full control over retention, access, and audit.

### 7.7 Secrets Handling

**Recommendation**: agenix.

**How it works**:
1. Secrets are age-encrypted files stored in the monorepo under `secrets/`.
2. `secrets.nix` maps each secret to the hosts and users whose public keys can decrypt it.
3. At NixOS activation time, agenix decrypts secrets to `/run/agenix/` (a tmpfs) using the host's SSH host key.
4. Services reference secrets via file paths (e.g., `config.age.secrets.db-password.path`), never environment variables baked into derivations.

```nix
# secrets/secrets.nix
let
  host-prod-web = "ssh-ed25519 AAAA...";
  host-prod-wrk = "ssh-ed25519 BBBB...";
  admin-alice   = "ssh-ed25519 CCCC...";
in {
  "db-password.age".publicKeys = [ host-prod-web host-prod-wrk admin-alice ];
  "api-key.age".publicKeys     = [ host-prod-web admin-alice ];
}
```

**Rationale**: Secrets are in the repo (auditable, version-controlled, reviewable) but never in the Nix store (which is world-readable). Decryption uses the host's existing SSH key — no additional secret distribution mechanism. age is simpler and more auditable than GPG.

**Tradeoffs**: Secret rotation requires re-encrypting and redeploying. This is fine — redeployment is fast and atomic. For secrets that must rotate without redeployment (e.g., database credentials), use a secrets engine (Vault) with agenix providing the Vault token. This is the only place an external secrets tool is justified.

**Failure modes**: Host key compromise exposes all secrets encrypted to that host. Mitigated by: minimize the set of secrets per host (least privilege), rotate host keys periodically, re-encrypt after any key compromise. A host's secret set is visible in `secrets.nix` — easy to audit.

**Why not sops-nix**: sops-nix uses Mozilla SOPS, which supports GPG, age, AWS KMS, and GCP KMS. This flexibility is a liability — more backends means more things to break and audit. agenix uses age only, which is simpler, faster, and easier to reason about. Both solve the same problem; agenix solves it with fewer moving parts.

**Why not Vault directly**: Vault is a distributed system (HA mode needs Consul or Raft), requires its own unsealing ceremony, and adds a network dependency to every service startup. For static secrets (API keys, database passwords, TLS certs), agenix is simpler and more reliable. Use Vault only when you need dynamic secrets (e.g., short-lived database credentials), and even then, agenix provides the Vault bootstrap token.

### 7.8 Repo and Module Boundaries

**Recommendation**: Single monorepo. One flake. One lockfile.

**Structure**:
```
/
├── flake.nix                  # Top-level: inputs, outputs, nixosConfigurations
├── flake.lock                 # Pinned inputs
├── lib/                       # Shared Nix library functions
│   ├── mkService.nix          # Service builder abstraction
│   ├── mkHost.nix             # Host builder abstraction
│   └── testing.nix            # Test helpers
├── modules/                   # Reusable NixOS modules
│   ├── base.nix               # Common to all hosts
│   ├── hardening.nix          # Security baseline
│   ├── monitoring.nix         # Prometheus node-exporter, Loki
│   ├── networking.nix         # WireGuard mesh, firewall
│   └── services/              # Per-service NixOS modules
│       ├── api.nix
│       ├── worker.nix
│       └── frontend.nix
├── services/                  # Application source code
│   ├── api/
│   │   ├── src/
│   │   ├── Cargo.toml         # (or package.json, go.mod, etc.)
│   │   └── default.nix        # Package derivation
│   ├── worker/
│   └── frontend/
├── hosts/                     # Per-host NixOS configurations
│   ├── prod/
│   │   ├── web-01.nix
│   │   ├── web-02.nix
│   │   └── worker-01.nix
│   ├── staging/
│   └── dev/
├── overlays/                  # Nixpkgs overlays
│   └── default.nix
├── secrets/                   # agenix encrypted secrets
│   ├── secrets.nix            # Key mapping
│   ├── prod/
│   │   ├── db-password.age
│   │   └── api-key.age
│   └── staging/
├── tests/                     # NixOS VM integration tests
│   ├── api-test.nix
│   └── full-stack-test.nix
├── terraform/                 # Infrastructure provisioning ONLY
│   ├── main.tf
│   └── outputs.tf
└── colmena.nix                # Fleet deployment configuration
```

**Module boundaries**: The key abstraction is `lib/mkService.nix` — a function that takes a service's package derivation and returns a NixOS module. This enforces a standard interface: every service gets a systemd unit with sandboxing, a health check endpoint, log forwarding, metrics export, and firewall rules.

```nix
# lib/mkService.nix
{ name, package, port, healthPath ? "/health", extraConfig ? {} }:
{ config, pkgs, lib, ... }: {
  systemd.services.${name} = {
    description = "${name} service";
    wantedBy = [ "multi-user.target" ];
    after = [ "network-online.target" ];
    serviceConfig = {
      ExecStart = "${package}/bin/${name}";
      DynamicUser = true;
      StateDirectory = name;
      ProtectSystem = "strict";
      ProtectHome = true;
      NoNewPrivileges = true;
      Restart = "always";
      RestartSec = 5;
    };
  };
  networking.firewall.allowedTCPPorts = [ port ];
  # ... monitoring, logging, health check config
}
```

**Rationale**: A monorepo with one flake means one lockfile, one CI pipeline, one version of nixpkgs, one set of overlays. There is no diamond dependency problem. There is no "which version of the shared library does staging have." Cross-service refactors are atomic commits.

**Tradeoffs**: Large monorepos can be slow to evaluate. Mitigated by: Nix's eval caching, and structuring the flake so that `nixosConfigurations` for a single host import only that host's subtree. Also, `nix eval` is fast for targeted outputs.

**Failure modes**: A bad commit to `modules/base.nix` breaks every host. Mitigated by: CI runs `nix flake check` which builds every `nixosConfiguration` and runs every NixOS test. Nothing merges without green CI.

**Why not multi-repo**: Multi-repo means multiple lockfiles, version skew between repos, cross-repo integration testing that's harder to coordinate, and loss of atomic cross-service changes. The operational cost of a monorepo is lower than the coordination cost of multi-repo for a team that standardizes aggressively.

### 7.9 Promotion and Rollback

**Promotion model**: Build once, promote through environments.

```
  commit → CI build → staging closure → staging deploy → staging tests
                                                              │
                                                         (promote)
                                                              │
                                                    prod closure = same store paths
                                                              │
                                                         canary deploy (1 host)
                                                              │
                                                         full fleet deploy
```

**Key detail**: "Promotion" does not mean "rebuild for prod." The staging closure and the prod closure share the same application derivations (same store paths). The only difference is the NixOS configuration (different secrets, different hostnames, different scaling). The application binaries are byte-for-byte identical.

**Rollback**: Every `switch-to-configuration` creates a new NixOS system profile generation. Rollback is:

```bash
# On a single host:
nixos-rebuild switch --rollback

# Via Colmena for the fleet:
colmena exec -- nixos-rebuild switch --rollback
```

This is an atomic symlink swap. It takes less than a second. The previous generation's entire closure is still in the Nix store (garbage collection is on a multi-day delay). All binaries, all config, all systemd units — everything reverts.

**Rationale**: This is the safest deployment model possible. You are never in a state where "the new version is partially deployed." You are always on generation N or generation N-1. There is no "the rollback failed because we can't pull the old container image."

**Tradeoffs**: Rollback reverts the entire host, not a single service. This is intentional — partial rollbacks create inconsistent states. If you need per-service rollback, you're really asking for per-service hosts, which is a scaling decision (see §10).

**Failure modes**: Rollback to a generation whose secrets have since been rotated. The old generation's service configs will reference old secret paths that may no longer be valid. Mitigated by: secret rotation should be coordinated with deployment, and the old secrets can be kept in agenix until the old generation is garbage-collected.

---

## 8. Local Development

Every developer runs NixOS on their workstation. The monorepo's `devShells` output provides per-service development environments:

```nix
devShells.x86_64-linux = {
  default = pkgs.mkShell {
    packages = [ pkgs.nil pkgs.nixpkgs-fmt pkgs.colmena ];
  };
  api = pkgs.mkShell {
    inputsFrom = [ self.packages.x86_64-linux.api ];
    packages = [ pkgs.cargo-watch pkgs.sqlx-cli ];
  };
  worker = pkgs.mkShell { /* ... */ };
};
```

`nix develop .#api` drops you into a shell with every dependency for the API service. No Dockerfiles. No `brew install`. No version managers.

**Local testing**: Developers can build and run any host's full NixOS configuration in a local VM:

```bash
# Build and run the staging-web host in a QEMU VM
nixos-rebuild build-vm --flake .#staging-web-01
./result/bin/run-*-vm
```

This boots a full NixOS system with the exact same configuration as staging — including services, firewall rules, and monitoring — in a local VM. This is the single most powerful testing capability NixOS provides.

**Editor integration**: Use `nil` (Nix language server) for IDE support. `nixpkgs-fmt` or `alejandra` for formatting. Both are pinned in the flake and provided via `devShells.default`.

---

## 9. CI/CD

### CI: Hydra

**Recommendation**: Self-hosted Hydra instance, running on a dedicated NixOS host, declared in the monorepo.

**Jobsets**: Hydra evaluates the flake and builds every output:
- All `packages.*` (service binaries)
- All `nixosConfigurations.*` (host closures)
- All `checks.*` (NixOS VM tests, linting, formatting)

A merge to `main` triggers a full evaluation. Feature branches trigger evaluation of affected outputs only (Hydra supports this via input change detection).

**NixOS VM tests**: The killer feature of NixOS CI. You write a test that spins up multiple NixOS VMs, configures them, and asserts behavior:

```nix
# tests/api-test.nix
nixosTest {
  name = "api-integration";
  nodes = {
    server = { ... }: {
      imports = [ self.nixosModules.api ];
      services.api.enable = true;
      services.api.database = "postgresql://db:5432/test";
    };
    db = { ... }: {
      services.postgresql.enable = true;
      services.postgresql.ensureDatabases = [ "test" ];
    };
  };
  testScript = ''
    db.wait_for_unit("postgresql")
    server.wait_for_unit("api")
    server.wait_for_open_port(8080)
    server.succeed("curl -f http://localhost:8080/health")
    server.succeed("curl -f http://localhost:8080/api/v1/status | grep ok")
  '';
}
```

This runs actual NixOS VMs with actual systemd, actual networking, actual Postgres. Not mocks. Not containers pretending to be VMs. Full machines. This catches configuration bugs, firewall issues, service ordering problems, and integration failures that unit tests and container-based CI miss.

**Build flow**:
```
1. Developer pushes branch
2. Hydra evaluates flake, identifies changed outputs
3. Remote builders build all affected derivations
4. Results pushed to org binary cache (Attic)
5. NixOS VM tests run
6. Green ✓ → PR is mergeable
7. Merge to main → Hydra rebuilds, pushes to cache
8. Deployment pipeline picks up new closures
```

### CD: Colmena

**Recommendation**: Colmena for fleet deployment.

Colmena is a NixOS deployment tool that evaluates your flake's `nixosConfigurations`, builds the closures (or pulls from cache), copies them to target hosts, and runs `switch-to-configuration`. It supports parallel deployment, canary deployment, and tagging for fleet segmentation.

```nix
# colmena.nix
{
  meta = {
    nixpkgs = import inputs.nixpkgs { system = "x86_64-linux"; };
  };

  defaults = { ... }: {
    imports = [ ./modules/base.nix ];
    deployment.allowLocalDeployment = false;
  };

  prod-web-01 = { name, nodes, ... }: {
    imports = [ ./hosts/prod/web-01.nix ];
    deployment.targetHost = "10.0.1.10";
    deployment.tags = [ "prod" "web" ];
  };

  prod-web-02 = { ... }: { /* ... */ };
  prod-worker-01 = { ... }: { /* ... */ };
}
```

**Deployment commands**:
```bash
# Deploy to all prod web hosts
colmena apply --on @web --on @prod

# Canary: deploy to one host, wait, then the rest
colmena apply --on prod-web-01
# ... verify ...
colmena apply --on @prod --on @web

# Rollback all prod web hosts
colmena exec --on @web --on @prod -- nixos-rebuild switch --rollback
```

**Why not NixOps**: NixOps manages state (an SQLite database tracking deployments). This state can desync, corrupt, or conflict between operators. Colmena is stateless — it evaluates the flake, diffs against the target, and deploys. No state file to manage.

**Why not deploy-rs**: deploy-rs is solid but less mature for large fleets. Colmena's tagging, parallel deployment, and `colmena exec` for ad-hoc fleet commands give it the edge for operational use.

---

## 10. Production Platform

### Host Architecture

Each host runs NixOS and serves a specific role. Roles are defined by which NixOS modules are imported:

- **Web hosts**: nginx reverse proxy + application service(s). Module: `modules/services/api.nix` + `modules/services/frontend.nix`.
- **Worker hosts**: Background job processors. Module: `modules/services/worker.nix`.
- **Data hosts**: Postgres, Redis, Kafka. Module: `modules/data/postgres.nix`, etc. These are the only stateful hosts.
- **Infra hosts**: Hydra, Attic, Prometheus, Grafana, Loki. Module: `modules/infra/*.nix`.

**One service per host vs. multiple**: Default to one primary service per host. This gives you per-service rollback (by rolling back that host), per-service scaling (by adding more hosts of that role), and blast radius containment. Colocate services on a host only when they are tightly coupled and always deployed together.

### Networking

- **WireGuard mesh**: All hosts connect via a WireGuard mesh network. Peer configurations are declared in NixOS modules and derived from the host definitions in the repo.
- **NixOS firewall**: Declarative, per-host, per-service firewall rules. When a service module opens a port, it does so explicitly in its NixOS module.
- **Load balancing**: HAProxy or nginx on dedicated edge hosts, configured via NixOS modules.
- **DNS**: Internal DNS via CoreDNS on infra hosts, declared in NixOS.

### Process Management

All services run under systemd. The NixOS module system gives you:
- `DynamicUser=true` for automatic user isolation
- `ProtectSystem=strict` + `ProtectHome=true` for filesystem sandboxing
- `NoNewPrivileges=true` for privilege escalation prevention
- `Restart=always` + `RestartSec=5` for automatic recovery
- Socket activation for zero-downtime restarts (where applicable)
- Cgroup resource limits (`MemoryMax=`, `CPUQuota=`)

This provides 90% of what Kubernetes gives you for workload isolation, using systemd — which is already there.

---

## 11. Build / Cache / Performance

### Build Performance

1. **Remote builders with high core count**: 64+ core machines (e.g., Hetzner AX162 or c7a.16xlarge). Nix parallelizes within a derivation via `enableParallelBuilding` and across derivations via `max-jobs`.
2. **Content-addressed derivations** (experimental): Enable `ca-derivations` in `nix.conf`. This means two derivations that produce identical outputs share a single store path, regardless of their input derivation graph. This dramatically improves cache hit rates for rebuilds where only metadata changed.
3. **Eval caching**: `nix.settings.eval-cache = true`. Flakes cache evaluation results, so re-evaluating an unchanged flake is near-instant.

### Cache Strategy

```
Build request
    │
    ▼
Local /nix/store hit? ──yes──► Done
    │ no
    ▼
Org Attic cache hit? ──yes──► Download, done
    │ no
    ▼
cache.nixos.org hit? ──yes──► Download, done
    │ no
    ▼
Build from source on remote builder
    │
    ▼
Push result to Attic ──► Available for all future requests
```

### Garbage Collection

- **Builders**: Aggressive GC (`--delete-older-than 7d`). Builders are ephemeral; the cache is the source of truth.
- **Production hosts**: Keep last 5 generations (for rollback). GC everything else weekly.
- **Developer machines**: Keep last 3 generations. GC on demand.
- **Attic cache**: Retain derivations referenced by any active deployment. GC unreferenced derivations after 30 days.

---

## 12. Secrets / Trust Model

### Trust Boundaries

```
┌─────────────────────────────────────────┐
│              Trust Zone: Repo           │
│  Secrets encrypted at rest (age)        │
│  Decryptable only by listed host keys   │
│  Viewable by anyone with repo access    │
│  (encrypted form only)                  │
└─────────────────────────────────────────┘
         │
         ▼ (deploy)
┌─────────────────────────────────────────┐
│          Trust Zone: Host               │
│  Secrets decrypted to /run/agenix/      │
│  tmpfs (never written to disk)          │
│  Readable only by designated services   │
│  Host SSH key is the trust anchor       │
└─────────────────────────────────────────┘
```

### Key Management

- **Host keys**: Each NixOS host generates an SSH ed25519 host key at first boot. This key is added to `secrets/secrets.nix` and committed. For new hosts, the workflow is: provision VM → extract host key → add to secrets.nix → re-encrypt relevant secrets → deploy.
- **Admin keys**: Platform engineers have personal age keys for encrypting secrets (and for emergency decryption). These are also listed in `secrets.nix`.
- **CI keys**: Hydra does not decrypt secrets. It builds closures. Secrets are decrypted only on target hosts at activation time.
- **Cache signing keys**: Attic uses a separate ed25519 key for signing store paths. This key lives only on the Attic server. It is provisioned via agenix.

### Secret Categories

| Category              | Mechanism        | Example                          |
| --------------------- | ---------------- | -------------------------------- |
| Static secrets        | agenix           | API keys, DB passwords, TLS certs |
| Dynamic secrets       | Vault + agenix   | Short-lived DB creds, cloud tokens |
| TLS certificates      | agenix + ACME    | Let's Encrypt via NixOS ACME module |
| SSH keys              | NixOS host keys  | Already present, used by agenix   |
| Cache signing         | Attic config     | Ed25519 key in agenix            |

---

## 13. Fleet / Environment Management

### Environments

Three environments, all declared in the same repo:

| Environment | Purpose                  | Hosts                        | Secrets       | Cache     |
| ----------- | ------------------------ | ---------------------------- | ------------- | --------- |
| dev         | Developer VMs, local testing | Ephemeral VMs via `nixos-rebuild build-vm` | Dev secrets   | Org cache |
| staging     | Pre-production validation | 1:1 mirror of prod (smaller) | Staging secrets | Org cache |
| prod        | Production               | Full fleet                   | Prod secrets  | Prod cache (subset of org) |

**Staging mirrors prod structurally**: Same NixOS modules, same service configurations, different host count and secrets. The staging `web-01.nix` and prod `web-01.nix` both import `modules/services/api.nix` — the only differences are secrets paths and scaling parameters.

### Fleet Segmentation (Colmena Tags)

```nix
deployment.tags = [ "prod" "web" "us-east" "canary" ];
```

Deploy by any combination:
```bash
colmena apply --on @prod --on @web           # All prod web hosts
colmena apply --on @canary                   # Canary hosts only
colmena apply --on @us-east --on @worker     # US-East workers
```

### Scaling

- **Horizontal**: Add a new host definition to `hosts/prod/`, add its key to `secrets.nix`, run Terraform to provision the VM, deploy with Colmena.
- **Vertical**: Change the VM spec in Terraform, reboot the host, Colmena re-deploys (closure is the same).
- **Auto-scaling**: For workloads that need it, a thin custom controller watches metrics and provisions/decommissions VMs via cloud API + Colmena. This is the one place where a small amount of custom orchestration is justified.

---

## 14. Migration Plan

Since this is a greenfield rewrite, there is no brownfield migration. The sequence is:

1. **Week 1-2**: Bootstrap monorepo. Set up flake structure, `lib/mkService.nix`, `lib/mkHost.nix`, base NixOS modules.
2. **Week 2-3**: Stand up infra hosts: Hydra, Attic, Prometheus/Grafana/Loki. All declared in the repo, deployed with Colmena.
3. **Week 3-4**: Stand up remote builders. Configure developer machines to use them.
4. **Week 4-6**: Implement first service (API) end-to-end: source in `services/api/`, package in `default.nix`, NixOS module in `modules/services/api.nix`, host config in `hosts/staging/web-01.nix`, NixOS VM test in `tests/api-test.nix`. This is the template for all future services.
5. **Week 6+**: Implement remaining services following the same pattern. Each service is a PR that adds source, package, module, host config, and tests.
6. **Ongoing**: Production deployment begins once staging validation is green. Canary → full fleet.

---

## 15. Risks and Anti-Patterns

### Risks

1. **Nixpkgs breakage**: A nixpkgs update breaks a dependency. **Mitigation**: Pin nixpkgs in `flake.lock`. Update deliberately with CI validation. Maintain overlays for critical patches.
2. **Eval performance**: Monorepo flake evaluation slows as the repo grows. **Mitigation**: Structure the flake to minimize import graph per host. Use `nix eval` targeted at specific outputs. Consider eval daemon for CI.
3. **Single-repo coupling**: A bad commit to a shared module breaks everything. **Mitigation**: Comprehensive CI (every `nixosConfiguration` builds, every test runs) before merge. Feature flags via NixOS module options, not branches.
4. **Nix store disk pressure**: `/nix/store` fills up on hosts. **Mitigation**: Automated GC, monitoring, alerting on disk usage. Keep last N generations, GC the rest.
5. **Bus factor**: The architecture assumes Nix expertise. If key engineers leave, the team may struggle. **Mitigation**: Exhaustive internal documentation. The monorepo is self-documenting (every host, service, and module is Nix code). But this is a real risk, acknowledged by the design brief's "do not optimize for onboarding."
6. **Upstream Nix churn**: Flakes are still "experimental." CLI may change. **Mitigation**: Pin the Nix version itself. The flake schema is stable in practice. Monitor nixpkgs releases for breaking changes.

### Anti-Patterns to Avoid

1. **Imperative state on NixOS hosts**: Never `nix-env -i` on a server. Never `systemctl edit`. Never manually edit `/etc/`. If it's not in the repo, it doesn't exist.
2. **Rebuilding at deploy time**: Deployments pull from cache. If a derivation isn't cached, the deploy should fail, not build. CI builds; Colmena deploys.
3. **Secrets in derivations**: Never pass secrets as build arguments, environment variables in `mkDerivation`, or anything that ends up in a `.drv` or store path. Secrets are runtime-only, via file paths from agenix.
4. **IFD (Import From Derivation)**: Avoid `import (pkgs.runCommand ...)` in module evaluation. IFD forces builds during eval, breaking Hydra's eval/build split and causing mysterious CI timeouts.
5. **Nix as a general-purpose programming language**: Nix is a configuration language. Don't write business logic in it. Don't build complex data transformation pipelines in it. If you need a script, write it in Bash or Python and package it with Nix.
6. **Over-abstracting modules**: NixOS modules are powerful. Resist the urge to create six layers of abstraction. `mkService` is one layer. Host configs import modules directly. Two layers max between a service's source and its running systemd unit.

---

## 16. What Should NOT Be Modeled in Nix

This is critical. Nix is the right tool for building, configuring, and deploying software. It is the wrong tool for:

1. **Cloud infrastructure provisioning**: Use Terraform (or Pulumi, OpenTofu) for VMs, VPCs, security groups, DNS zones, S3 buckets, managed databases. Nix cannot talk to cloud APIs. Terraform's state model is designed for this. The boundary is clear: Terraform creates the VM; NixOS configures what runs on it. Terraform outputs (IP addresses, hostnames) are consumed by Colmena and agenix.

2. **Runtime state management**: Database migrations, feature flags, A/B test configurations, user-facing content. These change at runtime without redeployment. Use appropriate tools (a migration framework, LaunchDarkly, a CMS).

3. **Application-level orchestration**: Request routing, circuit breaking, rate limiting, service discovery. These are runtime concerns handled by your load balancer, service mesh (if you must), or application code. NixOS configures the load balancer; it doesn't make routing decisions.

4. **Monitoring dashboards and alert rules**: While Prometheus scrape configs and Grafana installation are NixOS modules, the actual dashboard JSON and alerting rules should live in version control but be loaded by the tools themselves (Grafana provisioning, Prometheus rule files), not generated by Nix expressions. Nix builds the monitoring stack; it doesn't define what "high latency" means for your business.

5. **Data pipelines and ETL**: Nix builds the binaries. Airflow/Dagster/your-scheduler runs the DAGs. Don't model DAG dependencies in Nix.

6. **Multi-cloud abstractions**: Don't try to make Nix abstract over AWS vs GCP vs Azure. Use Terraform with provider-specific modules. Nix doesn't care which cloud the VM runs on — it configures the OS the same way regardless.

---

## 17. Rejected Alternatives

### Kubernetes
Rejected. See §7.3. TL;DR: Kubernetes is a platform for containers. This architecture doesn't use containers. Kubernetes adds a massive operational surface (etcd, API server, kubelet, CNI, CSI, ingress) that duplicates what NixOS + Colmena already provide, but worse (mutable state, YAML sprawl, container image mutability, slow rollback). The only thing Kubernetes does better is automatic bin-packing and HPA — both solvable at lower cost with over-provisioning and a simple custom scaler.

### Containers as Deployment Artifact
Rejected. A container image is a tar of filesystem layers. It is mutable (tags can be overwritten), not content-addressed (two identical builds may produce different layer hashes), and incomplete (it doesn't describe the host — networking, firewall, kernel parameters are external). A NixOS system closure is content-addressed, immutable, complete, and includes the host configuration. Containers are a strict downgrade.

### Nix + Docker (Building Container Images with Nix)
Rejected. This is a common "compromise" architecture: use Nix to build reproducible container images, then deploy them to Kubernetes. This gives you reproducible builds but throws away reproducible deployments. You still need Kubernetes, Helm, container registries, and all their failure modes. If you're building with Nix, deploy with Nix.

### NixOps
Rejected. NixOps manages deployment state in an SQLite database. This state can corrupt, desync, and conflict. It doesn't support parallel deployment to large fleets well. Colmena is stateless and fleet-aware.

### Ansible/Puppet/Chef for Configuration
Rejected. These are imperative or convergent configuration management tools. NixOS is declarative and atomic. Ansible can't roll back. Puppet can't guarantee a host matches its declaration (it converges toward it). NixOS guarantees it — the system closure is the truth.

### Nix on Non-NixOS (e.g., Nix + Ubuntu)
Rejected. See §7.2. You lose half the value of Nix: no atomic system rollback, no declarative system configuration, no `nixos-test`, no unified system closure. You manage two package managers and two configuration systems.

### Multi-Repo with Shared Flake Inputs
Rejected. Multiple repos mean multiple lockfiles, version skew, and cross-repo integration testing complexity. The coordination overhead exceeds the organizational cost of a monorepo for a team willing to standardize aggressively.

### Cachix (Hosted Binary Cache)
Rejected for primary use. Cachix is fine for open-source. For a company, self-hosted Attic on your own S3 keeps data in your VPC, gives you full control over retention and access, and avoids a third-party dependency for a critical path (builds).

### Nomad
Rejected. Nomad is a better Kubernetes for non-container workloads, but it still adds a scheduler, a gossip protocol (Serf), and a state store (Raft) that are unnecessary when Colmena already handles fleet deployment and NixOS handles service management. Nomad makes sense if you need dynamic scheduling of short-lived batch jobs across a shared cluster; for long-running services on dedicated hosts, it's overhead.

---

## 18. Recommended Defaults

These are opinionated defaults for the monorepo. Every one can be overridden per-service or per-host, but the defaults should cover 90% of cases.

| Setting                             | Default                                                  |
| ----------------------------------- | -------------------------------------------------------- |
| Nix version                         | Latest stable, pinned in flake                           |
| Nixpkgs channel                     | `nixos-unstable`, pinned in `flake.lock`                 |
| NixOS release                       | Tracks nixpkgs pin (unstable = latest)                   |
| Nix eval mode                       | Pure (`--pure-eval`, enforced in CI)                     |
| Flake checks                        | All `nixosConfigurations` build, all `checks` pass       |
| System architecture                 | `x86_64-linux` primary, `aarch64-linux` supported        |
| Service user                        | `DynamicUser = true` (systemd allocates)                 |
| Service sandboxing                  | `ProtectSystem=strict`, `ProtectHome=true`, `NoNewPrivileges=true` |
| Service restart                     | `Restart=always`, `RestartSec=5`                         |
| Logging                             | journald → Loki (via promtail)                           |
| Metrics                             | Prometheus node-exporter + service-specific exporters     |
| Firewall                            | Default deny. Services explicitly open ports.            |
| Secrets                             | agenix, decrypted to `/run/agenix/`, tmpfs               |
| TLS                                 | Let's Encrypt via NixOS ACME module                      |
| DNS                                 | Internal CoreDNS for service discovery                   |
| Time sync                           | `services.chrony.enable = true`                          |
| SSH access                          | Ed25519 keys only, password auth disabled, root SSH disabled |
| Nix GC (servers)                    | Weekly, keep last 5 generations                          |
| Nix GC (builders)                   | Daily, keep last 7 days                                  |
| Formatter                           | `alejandra` (deterministic, fast)                        |
| Linter                              | `statix` for Nix anti-pattern detection                  |
| Language server                     | `nil`                                                    |
| Deployment parallelism              | 10 hosts concurrent (Colmena default, tunable)           |
| Canary policy                       | Deploy to 1 tagged canary host, wait 10 min, then fleet  |

---

## 19. Final Verdict

The architecture is: **Flakes + NixOS + NixOS service hosts + Colmena + Attic + agenix + Hydra + monorepo**.

There is no Kubernetes. There are no containers. There is no Ansible. There is no Helm. There is no Docker. There is one tool that builds, configures, and deploys everything: Nix. There is one OS: NixOS. There is one repo. There is one lockfile. There is one truth.

Every host is a Nix expression. Every service is a Nix derivation wrapped in a NixOS module. Every deployment is an atomic switch to a new system generation. Every rollback is an atomic switch back. Every build is reproducible. Every secret is encrypted at rest and decrypted on-host. Every change is a git commit, reviewed in a PR, validated by CI that builds every host and runs integration tests in real NixOS VMs.

This is the architecture an elite platform team would run if they could start from zero, had infinite budget, universal Nix expertise, and the will to standardize. It is the most correct, most reproducible, most operable infrastructure architecture possible with today's tooling.

It is not easy. It is not familiar. It is right.
