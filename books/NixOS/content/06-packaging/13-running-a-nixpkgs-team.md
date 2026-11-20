---
title: Running a Nixpkgs team
---

# Running a Nixpkgs team

**Goal:** Design, charter, and operate a durable Nixpkgs maintainer team—connecting GitHub team governance, in-tree team sync (`maintainers/team-list.nix`), automated review tooling (`nixpkgs-review`, `nix-diff`, `ofborg`), supply-chain review checklists, and sustainable queue facilitation.

> [!NOTE]
> **Baseline:** Nix **2.35+** and **NixOS 26.05+**. Team sync automation from GitHub into the Nixpkgs tree landed in 2025; the policy that **teams are accountable individuals, not corporate brands** is current as of the Tweag 2026 write-up.

## Why it matters

Nixpkgs is the largest open-source software distribution in the world, with roughly a thousand active contributors every month across more than 100,000 packages. Most of that work is volunteer, and many critical pull requests still receive only a **single** committer glance before merge.

A **Nixpkgs team** turns “we care about these packages” into a durable, auditable review loop: a GitHub team under the `NixOS` organisation, in-tree team definitions wired into `meta.teams` / `meta.maintainers`, a public queue board outsiders can inspect, and an explicit norm—such as **blocking** defective or suspicious changes rather than racing to hit merge. Tweag’s 2026 write-up of their security-minded team provides a concrete operational blueprint that applies equally to ecosystem teams (Rust, Python, Robotics, Security, Kernel) and enterprise contributors.

---

## Mental model: governance & notification architecture

```text
  GitHub team (@NixOS/<team>)  ──sync bot──►  maintainers/team-list.nix
               │                                         │
               │                                         ▼
               │                       package meta.teams / meta.maintainers
               │                                         │
               ▼                                         ▼
  Public GitHub Project Board             ofborg / PR mention notifications
  + triage automation script                             │
               │                                         ▼
               ▼                          Local verification (nixpkgs-review,
  Matrix / Zulip standards room           nix-diff, upstream source audit)
  + private security checklist                           │
                                                         ▼
                                          Approve / Request changes / Block
```

| Component | Operational role |
|-----------|------------------|
| **GitHub team under `NixOS` org** | Any Nixpkgs **committer** can create one; membership defines the trust boundary |
| **In-tree team definitions (`maintainers/team-list.nix`)** | Automated GitHub → Nixpkgs sync keeps in-tree team membership aligned with GitHub without manual drift |
| **Package metadata (`meta.teams` / `meta.maintainers`)** | Connects packages to the team so `ofborg` and GitHub automatically ping reviewers when a PR touches those paths |
| **Public project board** | Exposes queue depth, reviewer assignment, and review SLA so contributors see work in flight |
| **Private review checklist** | Internal bar for security smells, patch forensics, and supply-chain checks—kept private so attackers cannot optimize specifically against a public rubric |
| **Explicit merge posture** | Tweag’s team **does not merge**; it **blocks** when something is wrong. Separating review from merge removes conflict-of-interest optics |

> [!IMPORTANT]
> **Individuals, not corporate brands:** Nixpkgs policy no longer permits corporate-branded teams as trust roots. Team members must be **accountable individuals** (with or without employer sponsorship). Organizations can fund engineering time, build farms, and review bandwidth, but trust in Nixpkgs accrues to named humans with track records.

---

## Setting up a team: step-by-step

### 1. Publish a public charter issue first

Before creating GitHub objects or touching `maintainers/`, open a concise Nixpkgs issue that defines the team’s contract:

1. **Mission & non-goals** (e.g., “review every PR touching our tracked security-sensitive packages and block unverified supply-chain or patch regressions”).
2. **Exact scope** (list of attribute paths, `pkgs/by-name/` prefixes, or NixOS module directories).
3. **Membership bar** (e.g., existing Nixpkgs commit access so community trust is already established, or a documented path for onboarding contributors after N quality reviews).
4. **Merge posture** (`review-and-block` vs `review-and-merge`—be explicit).
5. **Communication & cadence** (Matrix room, triage rotation, escalation path when a reviewer is away).

That issue doubles as your onboarding document and sets clear expectations for drive-by membership requests.

### 2. Create the GitHub team and verify in-tree sync

Committers create the team under the `NixOS` GitHub organization, add initial members, and let the **GitHub → Nixpkgs team sync** propagate the entry into `maintainers/team-list.nix` (do not maintain a parallel hand-edited roster in an external wiki).

```text
Setup verification checklist:
[ ] GitHub team exists under the NixOS org with individual members
[ ] In-tree team entry appears in maintainers/team-list.nix after sync
[ ] Team is attached to target packages in meta.teams (or lib.teams.<name>)
[ ] Test PR against a tracked package triggers actual team mention notifications
[ ] Project board automation ingests new PRs touching scoped files
```

### 3. Wire team ownership in package metadata

Check `maintainers/team-list.nix` and nearby `pkgs/by-name/` packages for the exact schema on your target Nixpkgs revision. Modern packages declare teams via `meta.teams` alongside individual `meta.maintainers`:

```nix
# pkgs/by-name/ex/example/package.nix (excerpt)
meta = {
  description = "Example security-critical daemon";
  homepage = "https://example.org";
  license = lib.licenses.mit;
  maintainers = with lib.maintainers; [
    alice
    bob
  ];
  teams = with lib.teams; [
    security-review
  ];
  mainProgram = "exampled";
};
```

Validate metadata against the repository’s own checks (`nixpkgs-lint` / maintainer schema tests in `CONTRIBUTING.md`) before opening the wiring PR.

---

## Technical review workflow: what reviewers actually run

A credible Nixpkgs team does not rubber-stamp diffs in the GitHub web UI. Reviewers verify builds, reverse dependencies, and derivation diffs locally or on a dedicated builder.

### 1. Build the PR and its reverse dependencies with `nixpkgs-review`

`nixpkgs-review` checks out the PR, computes every package whose `.drv` changed compared to the target branch, builds them in a sandbox, and optionally posts a structured build report directly to the PR:

```bash
# Review PR #345678 from a local nixpkgs checkout, building all affected packages
nixpkgs-review pr 345678

# On a builder with more resources, cap parallel jobs and post results to GitHub
nixpkgs-review pr 345678 --num-parallel-evals 4 --post-result

# Review only a specific platform or limit reverse-dependency rebuilds
nixpkgs-review pr 345678 --systems x86_64-linux --package mypkg
```

### 2. Inspect derivation-level diffs with `nix-diff`

When a PR claims to be a minor bump or formatting cleanup, compare the `.drv` before and after to verify nothing unexpected leaked into `configureFlags`, `patches`, `env`, or transitive inputs:

```bash
# Compare old and new derivations for a tracked package
nix-diff \
  $(nix eval --raw .#mypkg.drvPath) \
  $(nix eval --raw github:NixOS/nixpkgs/pull/345678/head#mypkg.drvPath)
```

### 3. Blast-radius and branch-target check

Every PR must target the right branch (`master` vs `staging`):

| Estimated rebuild count | Correct base branch | Reviewer action if mis-targeted |
|-------------------------|---------------------|---------------------------------|
| **1 – ~500 derivations** | `master` | Normal review + `nixpkgs-review` |
| **> 500 derivations** (core libs, Python/Rust/glibc/stdenv) | `staging` | Request retarget to `staging`; do **not** merge to `master` (overloads Hydra and breaks `nixos-unstable` cache hit rates) |
| **Security hotfix / regression** on stable release | `nixos-26.05` (backport) | Verify minimal diff, check ABI compatibility, backport cleanly after `master` |

### 4. High-signal security & supply-chain checks

During review, focus attention on the surfaces where bugs and malicious changes hide:

- **Source provenance:** Does `src` point to the canonical upstream repository and signed release tag, or did a contributor quietly redirect `owner`/`url` to a personal fork or unverified tarball?
- **FOD hash changes:** Never accept a hash change without verifying *why* the bytes changed (compare upstream release diff between old `rev` and new `rev`).
- **Build-phase hooks:** Audit every addition to `prePatch`, `postPatch`, `preConfigure`, `postInstall`, `preFixup`, and `setupHook`—these execute arbitrary shell code or alter runtime wrappers.
- **Patched files & vendored blobs:** Inspect every `.patch` file line-by-line; flag pre-compiled binaries, minified blobs, or `doCheck = false` additions that silence failing test suites without justification.
- **Closure drift:** Compare `nix path-info -rS` before and after to catch accidental `-dev` leaks or unexpected new runtime dependencies.

---

## Operating the queue: visibility, metrics, and sustainability

Volunteer capacity fluctuates. Teams die quietly when PRs pile up with no visible owner.

### Public board + lightweight automation

Pair a **GitHub Project board** (`Incoming` → `In Review` → `Changes Requested` → `Done`) with a small scheduled script (e.g., GitHub Actions + `gh api graphql`) that automatically adds every open PR touching your tracked packages to `Incoming`:

```bash
# Quick CLI triage: list open PRs touching a team's tracked packages
gh pr list --repo NixOS/nixpkgs \
  --search 'is:open label:"6.topic: security"' \
  --json number,title,author,updatedAt,reviewDecision \
  --jq '.[] | "#\(.number) [\(.reviewDecision // "PENDING")] \(.title) (@\(.author.login))"'
```

### Honest success metrics (the Tweag 2026 benchmark)

Between March and September 2026, Tweag’s security-focused Nixpkgs team reviewed **every** PR touching their package set—more than **250 pull requests** in six months. They did not claim theatrical zero-day interceptions; they **did** catch real engineering defects (misapplied patches, broken flags, code-quality regressions) continuously.

**That is the real operational bar:** predictable, boring coverage over 100% of your scope beats heroic incident response after a broken build hits `nixos-unstable`.

---

## Pitfalls

| Trap | Why it hurts | Mitigation |
|------|--------------|------------|
| **Corporate-branded teams** | Violates Nixpkgs individual-accountability policy and erodes community trust | Register individual contributors; sponsor their time transparently without making a brand the trust root |
| **Zero-bar membership on day one** | Dilutes review quality and creates trust ambiguity | Start with an explicit bar (e.g., existing committers) and publish a clear graduation path for new reviewers |
| **Security team that also merges by default** | Creates conflict-of-interest optics and pressures reviewers into speed over rigor | Adopt a **review-and-block** posture unless your charter explicitly requires merge authority |
| **Publishing the exact internal detection rubric** | Gives opportunistic contributors or attackers a checklist to game | Keep queue status and tracked package lists **public**; keep adversarial heuristics internal |
| **No facilitator / rotation** | Volunteer intermittency strands PRs silently until the team becomes nominal-only | Assign a rotating weekly triage facilitator who nudges stalled reviews and keeps the board accurate |

---

## Lab — launch a scoped review loop

1. **Scope estimation:** Pick **three** packages you rely on in production. Measure their 90-day PR volume using GitHub search (`repo:NixOS/nixpkgs is:pr path:pkgs/by-name/<xx>/<pkg>`).
2. **Charter draft:** Write a one-page charter issue covering mission, exact package list, membership rule, merge posture, and Matrix/escalation norms.
3. **Local review drill:** Pick one open Nixpkgs PR touching any of those packages and run `nixpkgs-review pr <num>` plus `nix-diff` on the resulting `.drv` paths.
4. **Metadata wiring:** Check `maintainers/team-list.nix` on a current Nixpkgs checkout, verify how `meta.teams` is structured, and inspect `ofborg` notification behavior on a sample PR.
5. **Triage board:** Set up a three-column project board (`Incoming` / `Reviewing` / `Done`) and write a one-line `gh pr list` query to audit stale PRs older than 7 days.

---

## Checkpoint

- [ ] Explained how GitHub teams sync into `maintainers/team-list.nix` and trigger PR review notifications via `meta.teams`.
- [ ] Run `nixpkgs-review pr <num>` and `nix-diff` to verify a package change and its reverse dependencies.
- [ ] Distinguished when a PR belongs on `master` vs `staging` based on rebuild blast radius.
- [ ] Listed the high-risk supply-chain surfaces to inspect in every Nixpkgs PR (`src`, FOD hashes, patches, phase hooks, `doCheck`).
- [ ] Designed a sustainable queue posture (public board, rotating facilitator, clear block-vs-merge charter).

## Sources

- [How to set up and run a Nixpkgs team](https://www.tweag.io/blog/2026-10-08-running-nixpkgs-team/) (Aleksandr Bantev / Tweag, 2026-10-08)
- [Nixpkgs CONTRIBUTING.md](https://github.com/NixOS/nixpkgs/blob/master/CONTRIBUTING.md) (maintainer and contribution norms)
- [nixpkgs-review](https://github.com/Mic92/nixpkgs-review) (automated PR build and review tool)
- Nixpkgs `maintainers/team-list.nix` and `maintainers/README.md` in-tree documentation
