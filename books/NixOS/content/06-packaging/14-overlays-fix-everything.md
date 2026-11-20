---
title: Fixing Packages with Overlays
---

# Fixing Packages with Overlays

**Goal:** Master the fixed-point mechanics of Nixpkgs overlays (`final: prev:`), compose multi-layer overlays safely without infinite recursion or accidental mass rebuilds, override language package sets (`python3`, `haskellPackages`, `nodePackages`), and export tested overlays from flakes.

> [!NOTE]
> **Baseline:** Nix **2.34+**, NixOS / nixpkgs **26.05**, flakes-first workflows. Overlays are the primary mechanism for patching, pinning, and extending Nixpkgs without maintaining a permanent Git fork.

## Why it matters

Every non-trivial NixOS fleet or developer platform hits the same inflection point: upstream `nixpkgs` has a broken package, a missing build flag, a CVE that has not yet reached your channel, or an internal CLI that needs to live alongside upstream tools.

Forking `nixpkgs` gives you total freedom at the cost of perpetual rebase tax. **Overlays** give you surgical, composable deltas over the package set—propagating fixes automatically to every downstream package that depends on the overridden attribute, while keeping your upstream pin clean and fast to bump.

---

## Mental model: fixed-point evaluation (`final` vs `prev`)

Nixpkgs is not a static attribute set; it is a **lazy fixed point** constructed by `lib.fix` and extended with `lib.extends` / `lib.composeManyExtensions`.

```text
  nixpkgs base (stage 0)
         │
         ▼
  overlay 1 (final: prev: { ... })   ◄── prev = stage 0
         │
         ▼
  overlay 2 (final: prev: { ... })   ◄── prev = stage 0 + overlay 1
         │
         ▼
  overlay N (final: prev: { ... })   ◄── prev = stage 0 + ... + overlay N-1
         │
         ▼
       final ──────────────────────────► tied back into EVERY overlay's `final`
```

Every overlay receives two arguments:

| Argument | What it points to | When to use it |
|----------|-------------------|----------------|
| **`prev`** (historically `super`) | The package set **before** this overlay is applied | Accessing the original package you are modifying (`prev.curl.overrideAttrs ...`), reading `prev.lib`, `prev.stdenv`, or `prev.fetchFromGitHub` to avoid bootstrap cycles |
| **`final`** (historically `self`) | The **fully resolved** package set after *all* overlays have run | Calling `final.callPackage` for new packages, referencing sibling packages that *other* overlays may have patched, or wiring runtime dependencies that should pick up downstream overrides |

> [!IMPORTANT]
> **The Golden Rule of `final` vs `prev`:**
> Use `prev` on the right-hand side whenever you are modifying an attribute of the **same name** (`foo = prev.foo.overrideAttrs ...`). Use `final` when referencing **other** packages so your package benefits from the entire overlay stack.

### How `lib.fix` ties the knot

Understanding the 4-line fixed-point combinator demystifies almost every overlay error:

```nix
# Conceptual core of nixpkgs overlay evaluation (lib/fixed-points.nix)
let
  fix = f: let x = f x; in x;
  extends = overlay: f: final:
    let prev = f final;
    in prev // overlay final prev;
in
  fix (extends overlay2 (extends overlay1 basePkgs))
```

Because Nix is lazily evaluated, `final` can be passed into `overlay1` *before* `overlay2` finishes—**as long as** evaluating an attribute in `overlay1` does not strictly demand the same attribute from `final` before producing its own value. The moment an attribute depends strictly on its own `final` value, evaluation enters an endless cycle: `error: infinite recursion encountered`.

---

## Choosing the right override scope

Overlays are powerful precisely because they are **global** to the `pkgs` instance. That is also their biggest hazard: overriding a foundational library (`openssl`, `glibc`, `zlib`, `python3`) in a global overlay forces Nix to rebuild every dependent package from source, bypassing `cache.nixos.org`.

| Technique | Blast radius | Cache friendliness | Best use case |
|-----------|--------------|--------------------|---------------|
| **Call-site `.override` / `.overrideAttrs`** | Single consumer | Highest (zero collateral rebuilds) | Customizing one app's flags or patching one leaf binary |
| **Scoped `callPackage` injection** | One package + its direct args | High | Passing a custom `openssl` or `ffmpeg` to one internal service without rebuilding the OS |
| **Leaf overlay** (`final: prev: { mytool = ...; }`) | Only consumers of `mytool` | High | Adding internal packages or patching leaf CLIs across a fleet |
| **Shared-library overlay** (`final: prev: { openssl = ...; }`) | Entire dependency graph | Low (triggers mass rebuilds) | Emergency fleet-wide security backports (paired with a private binary cache + remote builders) |

---

## Core overlay patterns

### 1. Patching or customizing an existing package (`overrideAttrs`)

Always use the two-argument `(finalAttrs: prevAttrs: ...)` or single-argument `(old: ...)` function form so you append to existing lists rather than silently wiping upstream patches or build inputs.

```nix
# overlays/curl-fix.nix
final: prev: {
  curl = prev.curl.overrideAttrs (old: {
    configureFlags = (old.configureFlags or [ ]) ++ [ "--disable-ares" ];
    patches = (old.patches or [ ]) ++ [
      ./patches/0001-curl-keepalive-fix.patch
    ];
  });
}
```

### 2. Adding new internal packages with `final.callPackage`

When introducing new attributes to `pkgs`, use `final.callPackage` so your package automatically receives any overridden dependencies from other overlays in the stack:

```nix
# overlays/internal-tools.nix
final: prev: {
  acme-cli = final.callPackage ../pkgs/acme-cli/package.nix { };
  acme-agent = final.callPackage ../pkgs/acme-agent/package.nix {
    # Explicit scoped override for just this package:
    openssl = final.openssl_3;
  };
}
```

### 3. Overriding package sets (Python, Haskell, Node, Perl)

Language ecosystems in Nixpkgs are **nested fixed points**. Naively doing `python3Packages = prev.python3Packages // { ... }` breaks internal cross-references inside the Python package set (other Python packages will still depend on the un-overridden version).

Instead, override the interpreter's `packageOverrides` fixed point:

```nix
# overlays/python-fixes.nix
final: prev: {
  python3 = prev.python3.override {
    packageOverrides = pyFinal: pyPrev: {
      # Patch an existing Python library across the entire python3Packages set
      requests = pyPrev.requests.overridePythonAttrs (old: {
        doCheck = false;
      });

      # Add a new internal Python library visible to all other Python packages
      acme-sdk = pyFinal.callPackage ../pkgs/acme-sdk/package.nix { };
    };
  };

  # Keep python3Packages in sync with the overridden interpreter
  python3Packages = final.python3.pkgs;
}
```

For `haskellPackages`, use `prev.haskellPackages.override { overrides = hFinal: hPrev: { ... }; }` or `prev.haskell.lib.compose.overrideCabal`.

### 4. Wrapping an existing binary without recompiling

If you only need to inject environment variables, default flags, or runtime `PATH` entries, **do not** rebuild a massive upstream package from source. Wrap the binary in a tiny derivation with `symlinkJoin` + `makeWrapper`:

```nix
# overlays/wrapped-tools.nix
final: prev: {
  kubectl = prev.symlinkJoin {
    name = "kubectl-wrapped-${prev.kubectl.version}";
    paths = [ prev.kubectl ];
    nativeBuildInputs = [ prev.makeWrapper ];
    postBuild = ''
      wrapProgram $out/bin/kubectl \
        --set KUBE_EDITOR "${final.neovim}/bin/nvim" \
        --prefix PATH : "${final.lib.makeBinPath [ final.kubelogin-oidc ]}"
    '';
    meta = prev.kubectl.meta;
  };
}
```

This builds in under a second, reuses the cached upstream `kubectl` binary, and keeps your runtime closure deterministic.

---

## Composing and exporting overlays in flakes

A well-structured flake exports reusable overlays under `outputs.overlays` and consumes them consistently across `packages`, `devShells`, and `nixosConfigurations`.

```nix
# flake.nix
{
  description = "Composable Nixpkgs overlays for nixos-26.05";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";
  };

  outputs = { self, nixpkgs }:
    let
      systems = [ "x86_64-linux" "aarch64-linux" ];
      forAllSystems = nixpkgs.lib.genAttrs systems;

      # Combine multiple focused overlay files into one default overlay
      composedOverlay = nixpkgs.lib.composeManyExtensions [
        self.overlays.curl-fix
        self.overlays.python-fixes
        self.overlays.internal-tools
      ];
    in
    {
      # Export individual overlays and a combined default
      overlays = {
        curl-fix = import ./overlays/curl-fix.nix;
        python-fixes = import ./overlays/python-fixes.nix;
        internal-tools = import ./overlays/internal-tools.nix;
        default = composedOverlay;
      };

      packages = forAllSystems (system:
        let
          pkgs = import nixpkgs {
            inherit system;
            overlays = [ self.overlays.default ];
          };
        in
        {
          inherit (pkgs) acme-cli acme-agent curl;
          default = pkgs.acme-cli;
        }
      );

      # NixOS host consuming the exact same overlay set
      nixosConfigurations.prod-01 = nixpkgs.lib.nixosSystem {
        system = "x86_64-linux";
        modules = [
          { nixpkgs.overlays = [ self.overlays.default ]; }
          ./hosts/prod-01/configuration.nix
        ];
      };
    };
}
```

### `pkgs.extend` vs `pkgs.appendOverlays` vs `import nixpkgs { overlays = ...; }`

| Method | Behavior | Caveat |
|--------|----------|--------|
| `import nixpkgs { inherit system; overlays = [ ... ]; }` | Evaluates `nixpkgs` once with the overlay list | Preferred in flakes and NixOS (`nixpkgs.overlays`) |
| `pkgs.appendOverlays [ ov1 ov2 ]` | Re-evaluates `nixpkgs` with additional overlays appended | Evaluates a fresh `pkgs` fixed point—expensive if called repeatedly inside loops |
| `pkgs.extend ov` | Extends the fixed point with a single overlay | Same re-evaluation cost as `appendOverlays`; never call per-package inside a module |

> [!WARNING]
> **Evaluation performance trap:** Calling `import nixpkgs { ... }` or `pkgs.appendOverlays` inside every individual NixOS module multiplies memory consumption and evaluation time (often adding gigabytes of RAM to `nixos-rebuild`). Always attach overlays once via `nixpkgs.overlays` at the system or flake boundary.

---

## Diagnosing infinite recursion and overlay bugs

### Common infinite recursion traps

1. **Self-referential `final` lookup:**
   ```nix
   # BAD: demands final.hello to define final.hello
   final: prev: {
     hello = final.hello.overrideAttrs (old: { doCheck = false; });
   }

   # GOOD: reads from prev
   final: prev: {
     hello = prev.hello.overrideAttrs (old: { doCheck = false; });
   }
   ```
2. **Pulling `lib` or `stdenv` from `final` in a low-level overlay:**
   If an overlay uses `final.lib.optionals` or `final.stdenv` while also modifying core attributes, the fixed point can cycle through `stdenv` bootstrapping. Use `prev.lib` and `prev.stdenv` for helper functions inside overlay logic.
3. **Conditional attribute existence checks (`final ? foo`):**
   Checking `if final ? foo` forces the attribute names of the entire `final` set—which forces *your own overlay* to return its attribute names first, causing instant infinite recursion. Always branch on `prev ? foo` or explicit arguments, never on `final ? ...`.
4. **Cross-overlay cycles:**
   Overlay A defines `foo` using `final.bar`, while Overlay B defines `bar` using `final.foo`. Break the cycle by referencing `prev` for one side or merging the coupled definitions.

### Debugging commands

```bash
# 1. Get the full evaluation stack trace on infinite recursion
nix build .#acme-cli --show-trace

# 2. Verify whether your overlay actually took effect on the derivation
nix derivation show .#curl | jq '.[].env.configureFlags'

# 3. Check blast radius BEFORE building (counts how many derivations will build vs substitute)
nix build .#nixosConfigurations.prod-01.config.system.build.toplevel --dry-run

# 4. Inspect why a package depends on an overlaid library
nix why-depends .#acme-agent .#openssl
```

---

## Lifecycle & retirement discipline

Every overlay is technical debt against future `nixpkgs` upgrades. Treat overlays like short-lived patches:

1. **Document the expiry condition** at the top of every overlay file (upstream PR URL, CVE ID, or target NixOS release).
2. **Add a version assertion** so Nix fails loudly when upstream catches up and the overlay can be retired:
   ```nix
   final: prev:
   assert prev.lib.assertMsg
     (prev.lib.versionOlder prev.curl.version "8.13.0")
     "Overlay curl-fix is obsolete: nixpkgs already has curl ${prev.curl.version}. Remove it.";
   {
     curl = prev.curl.overrideAttrs (old: {
       patches = (old.patches or [ ]) ++ [ ./patches/0001-curl-keepalive-fix.patch ];
     });
   }
   ```
3. **Never silently rename upstream attributes** inside an overlay—downstream modules and NixOS options expect canonical Nixpkgs names.
4. **Drop the overlay** the moment upstream merges and reaches your tracked channel.

---

## Lab — overlay composition and recursion forensics

1. **Create a two-overlay flake** targeting `nixos-26.05`:
   - Overlay 1 patches `hello` with a custom greeting via `overrideAttrs`.
   - Overlay 2 defines `hello-wrapped` using `final.hello` and `prev.makeWrapper`.
2. **Verify propagation:** Build `.#hello-wrapped` and confirm it prints the patched greeting from Overlay 1 (proving `final` resolution across overlays).
3. **Trigger and diagnose infinite recursion:** Temporarily change `prev.hello` to `final.hello` in Overlay 1, run `nix build .#hello --show-trace`, and locate the exact file/line in the trace.
4. **Measure blast radius:** Override `zlib` with a dummy comment in `postPatch` and run `nix build .#hello --dry-run` to observe how many derivations invalidate. Revert immediately.
5. **Add a retirement assertion:** Guard Overlay 1 with `lib.versionOlder` so bumping `nixpkgs` reminds you to delete it.

---

## Pitfalls

| Symptom | Root cause | Fix |
|---------|------------|-----|
| `error: infinite recursion encountered` | Used `final.pkg` to override `pkg`, or checked `final ? attr` | Switch self-references and attribute existence checks to `prev` |
| Overlay has no effect on NixOS host | Module imported a separate `nixpkgs` without `nixpkgs.overlays` | Apply overlays via `nixpkgs.overlays = [ ... ];` or pass `pkgs` consistently |
| Python override ignored by dependent apps | Overrode `python3Packages.foo` via `//` instead of `python3.override { packageOverrides = ...; }` | Use `packageOverrides` on the interpreter fixed point |
| `nixos-rebuild` suddenly compiles GCC/Glibc/Chromium | Overrode a low-level library globally in `nixpkgs.overlays` | Scope the override to the specific consumer via `callPackage`, or wrap without rebuilding |
| Upstream patches vanished after override | Used `overrideAttrs { patches = [ ./my.patch ]; }` (attribute set form) | Use function form `overrideAttrs (old: { patches = (old.patches or []) ++ [ ./my.patch ]; })` |

---

## Checkpoint

- [ ] Explained how `lib.fix` and `lib.extends` thread `final` and `prev` across an overlay list.
- [ ] Articulated when to use `prev` vs `final`, including why `final ? attr` causes infinite recursion.
- [ ] Overrode a Python package cleanly using `python3.override { packageOverrides = ...; }`.
- [ ] Wrapped an upstream binary with `symlinkJoin` + `makeWrapper` without triggering a source rebuild.
- [ ] Exported composed overlays from a flake with `lib.composeManyExtensions` and guarded an overlay with a retirement assertion.

## Sources

- [Nixpkgs Manual — Overlays](https://nixos.org/manual/nixpkgs/stable/#chap-overlays)
- [NixOS Manual — `nixpkgs.overlays`](https://nixos.org/manual/nixos/stable/#opt-nixpkgs.overlays)
- [nix.dev — Overlays tutorial](https://nix.dev/tutorials/nixos/overlays.html)
