---
title: Patching and overrides
---

# Patching and overrides

**Goal:** Fix or pin packages with `overrideAttrs`, `patches`, and overlays—without maintaining a permanent hard fork of nixpkgs.

## Why it matters

Upstream will be wrong for your window of time: a CVE pin, a broken release, a flag your hardware needs. The professional skill is **minimal, reviewable deltas** that compose, not copy-pasting half of nixpkgs into your flake.

---

## Theory 1 — Three layers of change

| Mechanism | Scope | Reuse | Maintenance |
|-----------|-------|-------|-------------|
| **`override` / `overrideAttrs`** | One package instance | Call-site | Low–medium |
| **Overlay** | Package set view for a flake/host | Shared modules | Medium |
| **Fork / `fetch` your tree** | Anything | High control | High—avoid until needed |

```text
nixpkgs  →  overlay(s)  →  pkgs.yourTool
                ↑
         overrideAttrs / patches
```

Overlays reshape `pkgs` for everyone who shares that package set. Call-site overrides affect one reference. Choose the smallest scope that works.

---

## Theory 2 — `overrideAttrs`

```nix
pkgs.hello.overrideAttrs (old: {
  patches = (old.patches or []) ++ [ ./patches/hello-msg.patch ];
  postPatch = (old.postPatch or "") + ''
    echo "packaging-craft" >> README
  '';
})
```

| Form | Notes |
|------|-------|
| `overrideAttrs (old: { … })` | Preferred; see previous attrs |
| `overrideAttrs { … }` | Easy to drop old patches by mistake |
| `package.override { stdenv = …; }` | Change **callPackage** args (dependencies), not phases |

### `override` vs `overrideAttrs`

```nix
# Change dependencies / function args
pkgs.ffmpeg.override { withXcb = true; }

# Change derivation attributes / phases
pkgs.ffmpeg.overrideAttrs (old: {
  configureFlags = (old.configureFlags or []) ++ [ "--enable-something" ];
})
```

Language builders often expose both (e.g. `overridePythonAttrs`).

### Composing safely

```nix
old.patches or []           # never assume patches exists
(old.postPatch or "") + ''  # string concat for hooks
(old.nativeBuildInputs or []) ++ [ pkgs.foo ]
```

Dropping upstream patches by accident is a classic security and build break.

---

## Theory 3 — Patches as first-class inputs

```nix
stdenv.mkDerivation {
  # …
  patches = [
    ./patches/0001-fix-build-on-musl.patch
    (fetchpatch {
      url = "https://github.com/org/proj/commit/abc123.patch";
      hash = "sha256-…";
    })
  ];
}
```

| Practice | Why |
|----------|-----|
| Numbered filenames | Apply order is list order |
| Keep patches small | Rebase pain scales with hunk size |
| Prefer upstream PRs | Local patches rot on every release |
| Test without patch once | Confirm the patch is still required |
| Record expiry | “Remove when nixpkgs ≥ rev …” |

`patchPhase` applies these automatically; fail logs show hunk rejects.

### Generating a patch

```bash
# Against unpacked sources or a git checkout of upstream
diff -u a/file b/file > patches/0001-msg.patch
# or
git format-patch -1 HEAD --stdout > patches/0001-msg.patch
```

Refresh patches when you bump `src.rev`—do not force with weaker reject settings as policy.

---

## Theory 4 — Overlay composition

```nix
# overlays/hello-patch.nix
final: prev: {
  hello = prev.hello.overrideAttrs (old: {
    patches = (old.patches or []) ++ [ ../patches/hello-msg.patch ];
  });
}
```

```nix
# flake.nix fragment
pkgs = import nixpkgs {
  inherit system;
  overlays = [
    (import ./overlays/hello-patch.nix)
    (import ./overlays/my-tool.nix)
  ];
};
```

### Order matters

```text
overlay1 final:prev → overlay2 final:prev → …
```

Later overlays see earlier results via the fixed-point.
Use **`final`** when depending on other packages you also overlay; use **`prev`** to modify the incoming package without recursion traps.

```nix
final: prev: {
  myApp = prev.myApp.overrideAttrs (_: { /* … */ });
  # depend on overlaid hello:
  myAppWrapped = final.hello;
}
```

### Infinite recursion patterns

```nix
# BAD
final: prev: {
  hello = final.hello.overrideAttrs (…); # often loops
}

# GOOD
final: prev: {
  hello = prev.hello.overrideAttrs (…);
}
```

### NixOS / HM consumption

```nix
nixpkgs.overlays = [
  (import ./overlays/hello-patch.nix)
];
```

Ensure the **same** `pkgs` instance that builds your host sees the overlay. A second `import nixpkgs { }` without overlays silently ignores your work.

---

## Theory 5 — Pinning and “don’t rebuild the world”

| Goal | Technique |
|------|-----------|
| One package newer than channel | Overlay with `fetchFromGitHub` src bump + hashes |
| One package older (pin) | Overlay src to known-good rev; note security debt |
| Only your app sees the change | Don’t put overlay in global NixOS; inject via `callPackage` scope |
| Temporary CI fix | Overlay in flake only; ticket to drop it |

```nix
final: prev: {
  curl = prev.curl.overrideAttrs (old: rec {
    version = "8.11.1";
    src = prev.fetchurl {
      url = "https://curl.se/download/curl-${version}.tar.bz2";
      hash = "sha256-…";
    };
  });
}
```

Bumping shared libraries can rebuild huge closures—check before celebrating:

```bash
nix why-depends .#myService .#openssl
nix build .#myService --dry-run 2>&1 | head
```

---

## Theory 6 — Scoped injection without global overlays

```nix
# Only this package sees a special dependency
pkgs.callPackage ./pkgs/mytool.nix {
  openssl = pkgs.openssl.override { … };
}
```

```nix
# Overlay adds your package without changing world openssl
final: prev: {
  mytool = final.callPackage ../pkgs/mytool.nix { };
}
```

Prefer **narrow** scope for experimental flags; global overlays for org-wide pins with review.

---

## Theory 7 — Language-specific override helpers

| Ecosystem | Helper | Example use |
|-----------|--------|-------------|
| Generic | `overrideAttrs` | phases, patches, env |
| Generic | `override` | `callPackage` parameters |
| Python | `overridePythonAttrs` | `doCheck`, deps |
| Haskell | `overrideCabal` (ecosystem) | when you live there |
| Node | often `overrideAttrs` | `npmDepsHash`, patches |

```nix
python3.pkgs.numpy.overridePythonAttrs (old: {
  doCheck = false;
})
```

---

## Theory 8 — In-depth override & patching mechanics (26.05)

### 1. Two-argument `overrideAttrs (finalAttrs: prevAttrs: ...)`

When a package was written with `stdenv.mkDerivation (finalAttrs: { ... })`, the **two-argument** `overrideAttrs` form lets you update `version` and have `src`, `name`, and `passthru` recompute cleanly against `finalAttrs`:

```nix
pkgs.ripgrep.overrideAttrs (finalAttrs: prevAttrs: {
  version = "14.1.1";
  src = pkgs.fetchFromGitHub {
    owner = "BurntSushi";
    repo = "ripgrep";
    rev = finalAttrs.version;
    hash = "sha256-...";
  };
  patches = (prevAttrs.patches or [ ]) ++ [ ./patches/0001-local-tweak.patch ];
})
```

> [!WARNING]
> **Avoid `overrideDerivation`:** The legacy `overrideDerivation` function bypasses `mkDerivation`'s argument processing (so changing `buildInputs` or `version` does *not* re-run `stdenv` attribute normalization). Always use `override` (for `callPackage` args) or `overrideAttrs` (for `mkDerivation` attrs).

### 2. `substituteInPlace` and `replaceVars` (replacing `substituteAll`)

For tiny path or constant substitutions inside source files, maintaining a `.patch` file that breaks on surrounding context changes is fragile. Use Nixpkgs's built-in substitution helpers:

- **`substituteInPlace`** (in `postPatch`) with **`--replace-fail`** (never legacy `--replace`, which silently succeeds if the target string disappears in an upstream upgrade!):
  ```nix
  postPatch = ''
    substituteInPlace src/config.c \
      --replace-fail "/usr/bin/env" "${coreutils}/bin/env"
  '';
  ```
- **`replaceVars`** (in `patches` or `src` templates, replacing deprecated `substituteAll`): interpolates `@var@` placeholders at build time and fails loudly if any placeholder is left unsubstituted:
  ```nix
  patches = [
    (replaceVars ./patches/hardcode-store-paths.patch {
      inherit bash coreutils;
    })
  ];
  ```

### 3. Scoped package sets with `lib.makeScope` and `overrideScope`

When maintaining a family of interdependent internal packages (e.g., a shared library `libacme` plus three daemons that link it), `lib.makeScope` creates an isolated mini-fixed-point that consumers can override without touching global `pkgs`:

```nix
acmeScope = lib.makeScope pkgs.newScope (self: {
  libacme = self.callPackage ./pkgs/libacme.nix { };
  acme-server = self.callPackage ./pkgs/acme-server.nix { };
  acme-cli = self.callPackage ./pkgs/acme-cli.nix { };
});

# Override libacme ONLY inside acmeScope — acme-server and acme-cli pick it up automatically
acmeDebug = acmeScope.overrideScope (final: prev: {
  libacme = prev.libacme.overrideAttrs { dontStrip = true; };
});
```

---

## Worked examples

### Example A — local patch on `hello`

```diff
# patches/hello-msg.patch
--- a/src/hello.c
+++ b/src/hello.c
@@ -59,7 +59,7 @@
-  printf ("Hello, world!\n");
+  printf ("Hello, packaging craft!\n");
```

(Exact hunks must match the version you patch—regenerate against real sources.)

```nix
# overlays/hello-craft.nix
final: prev: {
  hello = prev.hello.overrideAttrs (old: {
    patches = (old.patches or []) ++ [ ../patches/hello-msg.patch ];
  });
}
```

### Example B — overlay adding your ecosystem A package

```nix
final: prev: {
  hello-go = prev.callPackage ../pkgs/hello-go.nix { };
}
```

### Example C — `packageOverrides` legacy note

Older blogs use `packageOverrides`. Prefer **overlays** in modern flakes; recognize the old name when reading.

### Example D — flake export

```nix
{
  description = "overrides lab";
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";
  outputs = { self, nixpkgs }:
    let
      system = "x86_64-linux";
      pkgs = import nixpkgs {
        inherit system;
        overlays = [ (import ./overlays/hello-craft.nix) ];
      };
    in {
      packages.${system}.hello = pkgs.hello;
      packages.${system}.hello-callsite =
        (import nixpkgs { inherit system; }).hello.overrideAttrs (old: {
          patches = (old.patches or []) ++ [ ./patches/hello-msg.patch ];
        });
    };
}
```

---

## Lab — multi-step

Suggested workspace: `~/lab/nixos/packaging/overrides`

### Step 1 — Baseline

```bash
nix build nixpkgs#hello -o baseline-hello
./baseline-hello/bin/hello
```

### Step 2 — Write a patch

Create a minimal patch (message change or README touch). If patching C is painful on your platform, patch a script package you own from the stdenv chapter instead.

### Step 3 — `overrideAttrs` only (no overlay)

```nix
pkgs.hello.overrideAttrs (old: {
  patches = (old.patches or []) ++ [ ./patches/hello-msg.patch ];
})
```

```bash
nix build .#hello-patched -L
./result/bin/hello
```

### Step 4 — Promote to overlay

Move the same change to `overlays/…`. Import overlay into flake `pkgs`. Confirm drv path differs from unpatched when appropriate.

### Step 5 — Stack two overlays

1. Overlay patches `hello`
2. Overlay adds `myTool` depending on `final.hello`

Prove `myTool` sees the patched hello if it references it (even a trivial wrap).

### Step 6 — Dependency override

Pick a package with a boolean flag (choose something not enormous if you can):

```nix
pkgs.ffmpeg_7.override { /* one flag — or smaller package */ }
```

Build **or** evaluate and document why you stopped short if closure is huge.

### Step 7 — Maintenance note

In `NOTES.md`: expiry conditions for the patch (“remove when nixpkgs ≥ rev …” / “upstream PR #”).

### Step 8 — Intentional drop of upstream patches

Temporarily set `patches = [ ./only-ours.patch ];` without `old.patches`. Observe risk; restore proper composition.

---

## Exercises

1. Write a table: call-site override vs overlay vs nixpkgs PR—when each wins.
2. Refresh a patch against a newer `hello` version; document the conflict resolution.
3. Use `lib.optionals stdenv.isLinux` for a Linux-only patch.
4. Find a package in your flake that should **not** use a global overlay; refactor to call-site.
5. Draft a PR description as if upstreaming your patch (problem, approach, testing).

---

## Common gotchas

| Symptom / mistake | What to do |
|-------------------|------------|
| Patch rejects | Refresh patch against actual `src` version |
| Overlay silently not applied | Ensure same `pkgs` instance reaches the host/package |
| Infinite recursion in overlay | Use `prev` for the package you modify |
| Dropped upstream patches | Always `(old.patches or []) ++ [ ours ]` |
| Rebuilt half of nixpkgs | You overrode a low-level lib; narrow scope |
| Unmaintainable mega-patch | Split; upstream; or wait for release |
| Darwin vs Linux patch drift | Conditional patches with `lib.optionals` |
| Two nixpkgs imports, one overlay | Unify package set construction |
| Security pin without expiry note | Add ticket date and risk acceptance |

---

## Checkpoint

- [ ] `overrideAttrs` patch builds and changes behavior
- [ ] Overlay form works from flake
- [ ] Two overlays composed with known order
- [ ] Difference `override` vs `overrideAttrs` written down
- [ ] Patch expiry / upstream plan noted
- [ ] Avoided infinite recursion pattern
- [ ] Understood blast radius of low-level library overrides

### Commit

```bash
git add .
git commit -m "packaging: patches, overrideAttrs, overlays"
```

Write three personal gotchas before multiple outputs.
