---
title: Downstream Package Sets and Upstream Flow
---

# Downstream Package Sets and Upstream Flow

**Goal:** Master the operational craft of maintaining a production downstream package distribution (such as Flox, enterprise internal catalogs, or specialized distributions) layered on top of fast-moving upstream Nixpkgs, navigating channel synchronization, staging progression, patch lifecycle management, and upstreaming contributions.

## Why it matters

Consuming public `nixpkgs` directly works for personal workstations and small services. But shipping software products, developer platforms, or hardened runtime environments requires owning a **downstream package set**. 

Upstream Nixpkgs merges hundreds of commits daily across more than 100,000 packages. If you fork the tree blindly, you drown in rebase conflicts. If you rely solely on static overlays, evaluation performance degrades and subtle ABI mismatches slip into production. Engineering organizations like Flox, Chainguard, and enterprise platform teams maintain downstream distributions that remain pinned, patched, and reproducible while moving continuously with upstream innovation.

---

## The Upstream Ingestion & Staging Topology

Nixpkgs does not release packages on a per-package versioning cadence. Instead, it operates continuous integration loops that progress changes through branches and channels based on the blast radius of each change:

```text
  Individual PRs (Low Blast Radius, < 500 rebuilds)
       │
       ▼
  ┌──────────────────────────────────────────────────────────────┐
  │ master branch (Direct commits for non-mass rebuilds)         │
  └──────────────────────────────┬───────────────────────────────┘
                                 │
  Mass Rebuild PRs (> 500 rebuilds, glibc, stdenv, rust, python)
       │
       ▼
  ┌──────────────────────────────┴───────────────────────────────┐
  │ staging (Batched changes undergoing continuous realization)  │
  └──────────────────────────────┬───────────────────────────────┘
                                 │ Staging freeze
                                 ▼
  ┌──────────────────────────────────────────────────────────────┐
  │ staging-next (Evaluation stability & Hydra build farm check) │
  └──────────────────────────────┬───────────────────────────────┘
                                 │ Merged back to master
                                 ▼
  ┌──────────────────────────────────────────────────────────────┐
  │ Hydra Jobset Evaluation (trunk-combined / nixos:trunk-combined)│
  └──────────────────────────────┬───────────────────────────────┘
                                 │ Tests pass + Channel bump
                                 ▼
  ┌──────────────────────────────────────────────────────────────┐
  │ Channels (nixpkgs-unstable, nixos-unstable, nixos-26.05)     │
  └──────────────────────────────┬───────────────────────────────┘
                                 │ Downstream Sync
                                 ▼
  ┌──────────────────────────────────────────────────────────────┐
  │ Downstream Distribution (Carried Patches + Security Overlays)│
  └──────────────────────────────────────────────────────────────┘
```

### Upstream Branch and Channel Semantics

Understanding where upstream packages live is essential for deciding where to track, where to rebase, and where to submit patches:

| Branch / Channel | Purpose | Invalidation Scope | Downstream Role |
|------------------|---------|--------------------|-----------------|
| `master` | Primary trunk for day-to-day updates | Rebuilds < 500 dependents | Target for bug fixes and non-mass package updates |
| `staging` | Staging area for major toolchains and libraries | Affects thousands of derivations | Rebase point only when testing major toolchain transitions |
| `staging-next` | Stabilizing batch before merging into master | Build verification | Early warning system for upcoming rebuild freezes |
| `nixpkgs-unstable` | Rolling channel bumped once Hydra completes tests | Stable rolling binary cache | Primary upstream tracking branch for developer tools |
| `nixos-unstable` | Rolling channel validated by full NixOS VM tests | Fully verified system closures | Upstream tracking branch for full OS images and runtimes |
| `nixos-26.05` | Pinned stable release branch with backports | Conservative, stable ABI | Base for production fleets requiring minimal churn |

> [!IMPORTANT]
> Never track raw `master` directly in unattended production pipelines. Upstream `master` often moves faster than Hydra binary caches can build. Pointing a downstream pipeline at `master` causes local build farms to compile Chromium, LibreOffice, or Glibc from source when caches are missed. Track channel branches (`nixpkgs-unstable` or `nixos-unstable`), which only advance after Hydra has populated the public binary cache.

---

## Downstream Architecture: Forks vs. Overlays vs. Package Slices

When architecting a downstream distribution, you have three primary architectural choices:

| Architecture | Mechanism | Strengths | Failure Modes & Costs |
|--------------|-----------|-----------|------------------------|
| **Pure Overlay Tree** | Downstream flake imports upstream nixpkgs and applies an overlay list | Trivial to update upstream pin; zero Git merge conflicts | High eval latency; fragile `overrideAttrs` across deep dependency trees; no structural reorganization |
| **Downstream Monorepo Fork** | Full Git clone of nixpkgs with downstream commits on a tracking branch | Complete control over package layout; instant tree-wide modifications | Extreme Git rebase friction; git log divergence; high maintenance overhead |
| **Hybrid Layered Distribution (Flox-style)** | Independent downstream package catalog consuming upstream via locked flakes, with selective in-tree overlays and custom builder engines | Clean isolation; modular package definitions; decoupled CI; explicit provenance | Requires dedicated tooling to re-evaluate downstream expressions against upstream changes |

### The Hybrid Downstream Flake Pattern

The modern industry standard separates downstream proprietary packages and customized open-source packages into an explicit flake catalog:

```nix
# downstream/flake.nix
{
  description = "Enterprise Secure Downstream Package Distribution";

  inputs = {
    # Pin to a validated upstream channel commit
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    flake-utils.url = "github:numtide/flake-utils";
  };

  outputs = { self, nixpkgs, flake-utils }:
    flake-utils.lib.eachDefaultSystem (system:
      let
        # Layer downstream overrides cleanly
        pkgs = import nixpkgs {
          inherit system;
          config = {
            allowUnfree = false;
            permittedInsecurePackages = [];
          };
          overlays = [
            (import ./overlays/security-patches.nix)
            (import ./overlays/custom-toolchains.nix)
            (import ./overlays/enterprise-packages.nix)
          ];
        };
      in {
        packages = {
          # Export downstream-maintained packages
          secure-ripgrep = pkgs.secure-ripgrep;
          custom-proxy   = pkgs.custom-proxy;
          data-pipeline  = pkgs.data-pipeline;
        };

        # Export overlays so consumers can integrate into existing hosts
        overlays.default = nixpkgs.lib.composeManyExtensions [
          (import ./overlays/security-patches.nix)
          (import ./overlays/custom-toolchains.nix)
          (import ./overlays/enterprise-packages.nix)
        ];
      }
    );
}
```

---

## The Patch Lifecycle: Carry, Upstream, Retire

Carrying private patches indefinitely without upstreaming them is the fastest way to accumulate crippling technical debt. Every local patch is a future merge conflict when upstream refactors.

```text
 ┌────────────────┐       ┌─────────────────┐       ┌─────────────────┐
 │ 1. CARRY       │       │ 2. UPSTREAM     │       │ 3. RETIRE       │
 │ Apply local    │ ────> │ Open PR to      │ ────> │ Upstream merged │
 │ patch in       │       │ Nixpkgs master  │       │ & channel bumped│
 │ downstream     │       │ (by-name)       │       │ Drop local fix  │
 └────────────────┘       └─────────────────┘       └─────────────────┘
```

### Stage 1: Carrying a Downstream Patch Safely

When applying a downstream fix, write the override defensively. Never assume upstream attributes or patches won't change:

```nix
# overlays/security-patches.nix
final: prev: {
  openssl_3 = prev.openssl_3.overrideAttrs (oldAttrs: {
    pname = "${oldAttrs.pname or "openssl"}-downstream-patched";

    # Preserve all existing patches and append downstream security fix
    patches = (oldAttrs.patches or []) ++ [
      (final.fetchpatch {
        name = "CVE-2026-9999-session-ticket-validation.patch";
        url = "https://internal-security.corp/patches/openssl-cve-2026-9999.patch";
        hash = "sha256-47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=";
      })
    ];

    # Record provenance in passthru for supply-chain audits
    passthru = (oldAttrs.passthru or {}) // {
      downstreamPatched = true;
      carriedPatchTicket = "SEC-4892";
      upstreamTrackingPr = "https://github.com/NixOS/nixpkgs/pull/345678";
    };
  });
}
```

### Stage 2: Upstreaming to Upstream Nixpkgs

When upstreaming your changes to Nixpkgs, adhere to the upstream repository standards:

1. **Use `pkgs/by-name`:** Modern Nixpkgs mandates placing new packages in `pkgs/by-name/<aa>/<pname>/package.nix`.
2. **Commit Style:** Adhere strictly to the conventional commit pattern used by Nixpkgs committers:
   - `<pname>: <version> -> <new-version>`
   - `<pname>: fix build on aarch64-linux`
   - `<pname>: fix CVE-2026-XXXX`
3. **Run Nixpkgs Architecture Checks:**
   ```bash
   # Validate by-name and formatting conventions
   nix-shell maintainers/scripts/check-by-name.sh
   nix-shell -p nixfmt-rfc-style --run "nixfmt-rfc-style pkgs/by-name/my/mypackage/package.nix"
   ```

### Stage 3: Retiring Downstream Overrides

To prevent dead code and maintain clean evaluation graphs, downstream teams implement automated retirement guards:

```nix
# overlays/retirement-guards.nix
final: prev: {
  curl = 
    let
      # If upstream has already upgraded to 8.12.0 or higher, alert or retire
      upstreamVersion = prev.curl.version;
      needsRetirement = builtins.compareVersions upstreamVersion "8.12.0" >= 0;
    in
      if needsRetirement then
        # Warn or pass through unchanged
        builtins.trace "WARNING: Downstream patch on curl can be retired; upstream is at ${upstreamVersion}" prev.curl
      else
        prev.curl.overrideAttrs (old: {
          patches = (old.patches or []) ++ [ ./patches/curl-http2-downgrade-fix.patch ];
        });
}
```

---

## Upstream Synchronization and Rebase Runbook

Downstream maintainers execute regular sync cycles (weekly or bi-weekly). When synchronizing:

```text
[Fetch Upstream Remote] ──> [Run Lock Diff] ──> [Evaluate Package Set] ──> [Matrix Build in CI] ──> [Promote Channel]
```

### 1. Inspect What Changed Between Channel Bumps

```bash
# Compare flake lock inputs before updating
nix flake lock --update-input nixpkgs

# Run nvd (Nix Version Diff) against the evaluated outputs
nix-shell -p nvd --run "nvd diff /nix/var/nix/profiles/downstream-prod-12-link ./result"
```

The `nvd` tool prints a clean breakdown of package version transitions:
```text
<<< /nix/var/nix/profiles/downstream-prod-12-link
>>> ./result
[C] .curl: 8.10.1 -> 8.11.0
[U] .glibc: 2.40 -> 2.40-36 (rebuilds 4,210 packages)
[+] .new-utility: 1.0.0
[-] .deprecated-lib: 0.8.2
```

### 2. Identifying Mass Rebuild Cascades

Before merging an upstream bump, check how many downstream packages will need local compilation:

```bash
# Query the closure size and check substitutability against upstream cache
nix path-info --recursive --size --closure-size .#packages.x86_64-linux.my-product

# Verify if upstream binary cache already has the paths
nix check-substituters .#packages.x86_64-linux.my-product
```

If substitute check fails for fundamental dependencies (`glibc`, `openssl`), upstream Hydra has not finished publishing cache entries for your system architecture. Halt the synchronization until the cache is fully populated.

---

## Comparing Distribution Engineering Models

Understanding traditional Linux packaging paradigms helps communicate tradeoffs to infrastructure and security leaders:

| Dimension | Red Hat (RPM / Koji) | Debian / Ubuntu (deb / Launchpad) | Alpine / Wolfi (apk / Melange) | Nixpkgs / Flox (Nix / Hydra) |
|-----------|----------------------|-----------------------------------|--------------------------------|------------------------------|
| **Package Unit** | SPEC file + tarball + patches | `debian/rules` + control files | YAML declarative pipelines | Declarative Nix expressions (`package.nix`) |
| **Dependency Model** | Global system namespace (`/usr`) | Global system namespace (`/usr`) | Minimal chroot filesystem root | Content-addressed cryptographic store paths (`/nix/store`) |
| **Coexistence** | Multi-version conflict hell | Alternatives system (`update-alternatives`) | Single version per container | Arbitrary side-by-side multi-version closures |
| **Build System** | Koji build farm (mock chroots) | Launchpad / sbuild | Ephemeral bubblewrap / docker | Hermetic unshare sandboxes (`stdenv`) |
| **Channel Promotion** | Bodhi update gates | Testing migration (`excuses`) | Rolling git tags | Hydra jobset evaluation + test suite gates |

---

## Checkpoint & Self-Review

Before shipping a downstream package set sync:

- [ ] Upstream pin references a channel branch (`nixpkgs-unstable`, `nixos-unstable`, or `nixos-26.05`), never unverified `master`.
- [ ] Every carried patch contains a tracking header (CVE number, internal ticket, upstream PR URL).
- [ ] Carried patches append to `oldAttrs.patches or []` rather than overwriting upstream patches.
- [ ] Upstream package submissions adhere to `pkgs/by-name` sharding conventions.
- [ ] Downstream expressions include automated version retirement traces.
- [ ] Evaluation diff (`nvd`) verified before pushing to downstream binary caches.
- [ ] No undeclared build-time substituter misses on fundamental base libraries.
