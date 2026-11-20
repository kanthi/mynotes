---
title: Debugging Complex Build Systems in Nix
---

# Debugging Complex Build Systems in Nix

**Goal:** Master the diagnosis and repair of real-world build failures across CMake, Meson, Autotools, and Bazel within Nix's strict hermetic sandbox, troubleshooting compiler flags, missing pkg-config modules, rpath corruption, and dynamic linker failures.

## Why it matters

Upstream build systems are written under the assumption of an FHS (Filesystem Hierarchy Standard) Linux distribution—they search for headers in `/usr/include`, libraries in `/usr/lib64`, and helper binaries in `/bin` or `/usr/local/bin`. When dropped into Nix's hermetic sandbox, these assumptions shatter.

A Senior Packaging Engineer does not treat build errors as black-box mysteries. You must be able to read a 10,000-line `CMakeError.log`, debug an Autotools macro expansion failure in `config.log`, intercept Meson's wrap-file download requests, and reconcile Bazel's internal sandboxing engine with the Nix build sandbox.

---

## Build System Integration in Nixpkgs

Nixpkgs provides setup hooks and wrapper packages that adapt upstream build systems to the Nix environment:

| Build System | Nixpkgs Wrapper & Hook | Injected Environment & Flags | Primary Failure Modes in Sandbox |
|--------------|------------------------|------------------------------|-----------------------------------|
| **CMake** | `cmake` setup hook | Injects `-DCMAKE_INSTALL_PREFIX=$out`, toolchain files, `find_package` hints | Hardcoded search paths, ignoring `CMAKE_PREFIX_PATH`, rpath stripping |
| **Meson / Ninja** | `meson`, `ninja` setup hook | Injects `--prefix=$out`, cross-compilation definition files | Attempting to fetch wrap files/subprojects over network during configure |
| **Autotools** | `autoreconfHook`, GNU `stdenv` | Injects `--prefix=$out`, `--disable-dependency-tracking`, host triples | Stale `configure` scripts, missing m4 macros, libtool sysroot path mangling |
| **Bazel** | `bazel` | Strict sandboxing, workspace rule execution | Attempting to download external rules/dependencies during realization |

---

## Debugging CMake in Nix

The Nixpkgs `cmake` setup hook automatically sets up flags and paths so CMake searches `buildInputs` and `nativeBuildInputs`:

```text
  cmake setup hook
       │
       ▼
  Populates CMAKE_PREFIX_PATH = "/nix/store/...-zlib-dev;/nix/store/...-openssl-dev;..."
  Populates CMAKE_INSTALL_PREFIX = $out
  Populates CMAKE_BUILD_TYPE = "Release" (by default)
```

### Trap 1: CMake Cannot Find a Dependency (`find_package` Failure)

When CMake fails with `Could not find a package configuration file provided by "Foo"`:

```text
-- Could NOT find Foo (missing: Foo_DIR)
CMake Error at CMakeLists.txt:45 (find_package):
  Found package configuration file:
    /nix/store/.../lib/cmake/Foo/FooConfig.cmake
  but it set Foo_FOUND to FALSE so package "Foo" is considered to be NOT FOUND.
```

**Diagnostic Procedure:**
1. Check whether `foo` provides CMake config files or pkg-config files:
   ```bash
   nix-shell -p foo --run "find \$(echo \$foo | cut -d' ' -f1) -name '*cmake' -o -name '*pc'"
   ```
2. If `foo` only provides pkg-config files (`foo.pc`), ensure `pkg-config` is in `nativeBuildInputs` and `foo` is in `buildInputs`.
3. If CMake still misses the path, pass explicit hints via `cmakeFlags`:
   ```nix
   cmakeFlags = [
     "-DFoo_DIR=${pkgs.foo}/lib/cmake/Foo"
     "-DENABLE_TESTS=OFF"
     "-DCMAKE_POLICY_DEFAULT_CMP0074=NEW" # Honor Foo_ROOT hints
   ];
   ```

### Trap 2: CMake Overwrites RPATH on Install

By default, CMake may clear or replace RPATHs during the `install` phase:

```text
-- Set runtime path of "/nix/store/...-my-pkg/bin/tool" to ""
```

When this happens, the installed binary immediately fails with:
`error while loading shared libraries: libbar.so: cannot open shared object file`

**The Fix:**
Ensure `cmake` does not wipe the RPATH configured by the Nixpkgs compiler wrapper:

```nix
cmakeFlags = [
  "-DCMAKE_SKIP_BUILD_RPATH=FALSE"
  "-DCMAKE_BUILD_WITH_INSTALL_RPATH=FALSE"
  "-DCMAKE_INSTALL_RPATH_USE_LINK_PATH=TRUE"
];
```

---

## Debugging Meson and Ninja

Meson is designed for fast, modular builds, but it heavily pushes "WrapDB"—a mechanism to download external dependencies at configure time.

```text
[Meson Sandbox Violation]
Subproject foo is missing. Downloading from https://wrapdb.mesonbuild.com/...
Network is unreachable (inside sandbox)
```

### Resolving Meson Wrap Downloads

In Nix's hermetic build sandbox, outbound network calls are forbidden during normal builds.

**Solution 1: Provide the Dependency from Nixpkgs**
If `foo` exists in Nixpkgs, add it to `buildInputs`. Meson checks system packages via `dependency('foo')` (which uses `pkg-config`) before falling back to subprojects:

```nix
nativeBuildInputs = [ meson ninja pkg-config ];
buildInputs = [ foo bar zlib ];
```

**Solution 2: Pre-fetch Subproject into Fixed-Output Derivation**
If the subproject is a private fork or unreleased library, fetch it hermetically and link it into `subprojects/`:

```nix
postPatch = ''
  mkdir -p subprojects/foo
  cp -r ${fetchFromGitHub {
    owner = "upstream";
    repo = "foo";
    rev = "v1.2.0";
    hash = "sha256-...";
  }}/* subprojects/foo/
'';
```

---

## Debugging Autotools: `config.log` Forensics

Autotools projects run hundreds of small shell and C snippets during `./configure`. When configure fails, the error message in the build log is often completely misleading:

```text
configure: error: C compiler cannot create executables
```

This does **not** mean `gcc` is missing. It usually means an invalid flag or an incompatible library link failed during a test compilation.

### The Forensic Procedure

1. Run the build with `--keep-failed`:
   ```bash
   nix build .#broken-pkg --keep-failed -L
   ```
2. Navigate to the kept directory printed by Nix:
   ```bash
   cd /tmp/nix-build-broken-pkg-*/
   ```
3. Open `config.log` and search backwards from the end (`G` in `less`):
   ```bash
   less config.log
   ```
4. Look for the actual compilation failure before the exit message:
   ```text
   configure:3421: gcc -o conftest conftest.c -lz
   /nix/store/...-binutils/bin/ld: cannot find -lz: No such file or directory
   collect2: error: ld returned 1 exit status
   ```
   **Discovery:** The real issue was a missing `zlib` in `buildInputs`.

### Resolving Stale Autotools with `autoreconfHook`

Many older projects ship with pre-generated `configure` and `libtool` scripts from 2012 that fail to recognize modern Linux architectures (`aarch64`, `riscv64`) or modern Glibc versions.

**The Fix:**
Add `autoreconfHook` to `nativeBuildInputs`. This re-runs `aclocal`, `autoconf`, and `libtoolize` cleanly using modern toolchains:

```nix
nativeBuildInputs = [ autoreconfHook pkg-config ];
```

If specific m4 macros are missing:

```nix
# Pass include directories for external m4 macros
autoreconfFlags = [ "-I" "m4" "-I" "${pkgs.libtool}/share/aclocal" ];
```

---

## Debugging Bazel in Nix

Bazel poses unique packaging challenges because it is itself a complete hermetic build system that attempts to control the filesystem, download toolchains, and isolate workers. Running Bazel inside Nix requires reconciling two sandboxes.

```text
┌────────────────────────────────────────────────────────────┐
│ Nix Sandbox (Linux unshare, network disabled, pure store)   │
│   ┌──────────────────────────────────────────────────────┐ │
│   │ Bazel Driver (runs offline with pre-fetched deps)    │ │
│   │   ┌────────────────────────────────────────────────┐ │ │
│   │   │ C++ / Java Compiler Actions                    │ │ │
│   │   └────────────────────────────────────────────────┘ │ │
│   └──────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
```

### The Bazel Packaging Pattern

1. **Disable Bazel's Internal Sandboxing:** Bazel's sandbox conflicts with Nix's Linux namespaces (`unshare`). Run Bazel with `--spawn_strategy=standalone`.
2. **Pre-fetch External Dependencies:** Build rules cannot reach the internet. Dependencies must be fetched via Nix fixed-output derivations and passed via Bazel's `--distdir`:

```nix
{ lib, stdenv, fetchurl, bazel_6, jdk17 }:

let
  # Pre-fetch external tarballs required by WORKSPACE
  rulesCc = fetchurl {
    url = "https://github.com/bazelbuild/rules_cc/releases/download/0.0.9/rules_cc-0.0.9.tar.gz";
    hash = "sha256-...";
  };
in
stdenv.mkDerivation {
  pname = "bazel-service";
  version = "1.0.0";

  nativeBuildInputs = [ bazel_6 jdk17 ];

  buildPhase = ''
    runHook preBuild

    # Create local distribution directory containing external dependencies
    mkdir -p distdir
    cp ${rulesCc} distdir/rules_cc-0.0.9.tar.gz

    # Execute Bazel build in standalone offline mode
    bazel \
      --output_user_root=$TMPDIR/bazel_root \
      build \
      --distdir=distdir \
      --spawn_strategy=standalone \
      --curses=no \
      //src:service

    runHook postBuild
  '';

  installPhase = ''
    mkdir -p $out/bin
    cp bazel-bin/src/service $out/bin/
  '';
}
```

---

## Dynamic Linking, ELF Diagnostics, and `patchelf`

Even when compilation completes, binary execution can fail due to Linux ELF linking mechanics:

```bash
# Inspect runtime shared library dependencies
patchelf --print-needed ./result/bin/my-app
# Output:
#   libssl.so.3
#   libcrypto.so.3
#   libc.so.6

# Inspect embedded RPATH / RUNPATH
patchelf --print-rpath ./result/bin/my-app
# Output:
#   /nix/store/...-openssl-3.0.15/lib:/nix/store/...-glibc-2.40/lib
```

### DT_RPATH vs DT_RUNPATH

- `DT_RPATH` takes precedence over `LD_LIBRARY_PATH`.
- `DT_RUNPATH` can be overridden by `LD_LIBRARY_PATH`.

Nix's `fixupPhase` automatically runs `patchelf` and shrinks RPATHs to only those store paths actually referenced by the ELF binary.

### Automating Pre-Compiled Binaries with `autoPatchelfHook`

When packaging proprietary or pre-compiled vendor binaries:

```nix
{ stdenv, autoPatchelfHook, zlib, openssl, libGL }:

stdenv.mkDerivation {
  pname = "vendor-tool";
  version = "2.0.0";
  src = ./vendor-binaries.tar.gz;

  # autoPatchelfHook scans all ELF files and sets RPATH automatically
  nativeBuildInputs = [ autoPatchelfHook ];

  # Libraries that autoPatchelfHook should search for needed .so files
  buildInputs = [ zlib openssl libGL ];

  installPhase = ''
    mkdir -p $out/bin
    cp vendor-tool $out/bin/
  '';
}
```

---

## Checkpoint & Self-Review

When debugging and fixing complex build systems:

- [ ] CMake `find_package` failures checked against both `cmake` config files and `pkg-config` (`.pc`).
- [ ] CMake RPATH preserved during installation (`CMAKE_SKIP_BUILD_RPATH=FALSE`).
- [ ] Meson wrap-file downloads eliminated by declaring dependencies in `buildInputs` or pre-fetching to `subprojects/`.
- [ ] Autotools failure causes identified by reading `config.log` forensics rather than high-level error summaries.
- [ ] Outdated Autotools build scripts regenerated using `autoreconfHook`.
- [ ] Bazel configured with `--distdir` and `--spawn_strategy=standalone` for pure offline builds.
- [ ] Installed ELF binaries verified with `patchelf --print-rpath` and `patchelf --print-needed`.
