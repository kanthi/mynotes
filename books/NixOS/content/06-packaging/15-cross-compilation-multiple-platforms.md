---
title: Cross-Compilation for Multiple Platforms
---

# Cross-Compilation for Multiple Platforms

**Goal:** Master native cross-compilation in Nixpkgs—building C/C++, Rust, and Go binaries (and full NixOS system closures) from an `x86_64-linux` host to `aarch64-linux`, `riscv64-linux`, and static `musl` targets without slow QEMU emulation—by understanding `buildPlatform`, `hostPlatform`, `targetPlatform`, the 6-slot dependency matrix, and package splicing.

> [!NOTE]
> **Baseline:** Nix **2.34+**, NixOS / nixpkgs **26.05**, flakes-first workflows. Production platform engineering ships software to heterogeneous fleets—AWS Graviton / Ampere ARM64 servers, Raspberry Pi / edge boards, and RISC-V hardware—where native cross-compilation cuts CI latency and compute costs by an order of magnitude.

## Why it matters: why QEMU emulation fails at scale

Traditional CI/CD pipelines build multi-architecture artifacts using QEMU user-mode emulation (`binfmt_misc`):

```text
x86_64 CI Runner ──> [QEMU Software Translation] ──> Emulated aarch64 gcc / cargo / ld
```

While emulation is easy to toggle on (`boot.binfmt.emulatedSystems = [ "aarch64-linux" ];`), it fails under production workloads:

1. **Extreme performance penalty:** Translating ARM64 or RISC-V instructions on x86_64 slows CPU-bound compilers by **5x to 20x**. A Rust or C++ workspace that builds in 4 minutes natively can take over an hour under QEMU.
2. **Emulation instability:** QEMU user-mode periodically hangs or segfaults on multithreaded linkers (`lld`, `mold`), futex contention, Go/JVM garbage collectors, or complex signal handlers.
3. **Wasted cloud compute:** Organizations burn thousands of CI minutes running emulated compilers when a native x86_64 cross-compiler could emit the exact same ARM64 machine code at full silicon speed.

**The solution:** True **cross-compilation**—running compilers, linkers, and code generators natively on `buildPlatform` while emitting machine code and linking libraries for `hostPlatform`.

---

## The Nixpkgs platform model: build, host, target

Nixpkgs formalizes the GNU autotools three-platform model across every derivation and package set:

| Platform concept | Nix attribute | Definition | Practical example (compiling an ARM64 app on x86_64) |
|------------------|---------------|------------|------------------------------------------------------|
| **Build platform** | `stdenv.buildPlatform` | Where the build executes | `x86_64-linux` (your laptop or CI runner) |
| **Host platform** | `stdenv.hostPlatform` | Where the resulting package runs | `aarch64-linux` (AWS Graviton / Raspberry Pi) |
| **Target platform** | `stdenv.targetPlatform` | What a compiler/assembler emits code *for* | Only differs from `hostPlatform` when building a cross-compiler itself (Canadian cross) |

```text
Build Platform: x86_64-linux  (runs x86_64-to-aarch64 cross-gcc / rustc at native speed)
       │
       ▼
Host Platform:  aarch64-linux (target ELF binaries & shared libraries linked against aarch64 glibc/musl)
```

In native builds, `buildPlatform == hostPlatform == targetPlatform`. In standard cross-compilation, `buildPlatform != hostPlatform` and `hostPlatform == targetPlatform`.

### Checking executability in derivations

Never hardcode architecture strings when deciding whether a built binary can run during the build (e.g., generating man pages via `--help` or running `checkPhase`):

```nix
# True when native, OR when buildPlatform can execute hostPlatform (e.g., x86_64 running i686)
canRunHost = stdenv.buildPlatform.canExecute stdenv.hostPlatform;
```

Nixpkgs automatically disables `doCheck` when `!stdenv.buildPlatform.canExecute stdenv.hostPlatform`, unless you explicitly configure an emulator via `stdenv.hostPlatform.emulator buildPackages`.

---

## Instant cross-toolchains with `pkgsCross` and `pkgsStatic`

Nixpkgs exposes pre-configured cross-compilation package sets under `pkgsCross` (backed by `lib.systems.examples`) and fully static musl builds under `pkgsStatic`:

```bash
# Inspect available cross-compilation targets
nix eval --impure --expr 'builtins.attrNames (import <nixpkgs> {}).pkgsCross'

# Cross-compile GNU Hello from x86_64-linux to aarch64-linux
nix build nixpkgs#pkgsCross.aarch64-multiplatform.hello

# Cross-compile a fully static aarch64 musl binary (zero libc runtime dependency!)
nix build nixpkgs#pkgsCross.aarch64-multiplatform-musl.pkgsStatic.hello

# Verify ELF architecture and dynamic/static linking
file ./result/bin/hello
# ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked ...
```

| Package set | Target triple / ABI | Typical use |
|-------------|---------------------|-------------|
| `pkgsCross.aarch64-multiplatform` | `aarch64-unknown-linux-gnu` | ARM64 Linux servers (Graviton, Ampere, Raspberry Pi 4/5) |
| `pkgsCross.aarch64-multiplatform-musl` | `aarch64-unknown-linux-musl` | Lightweight ARM64 containers / embedded Linux |
| `pkgsCross.riscv64` | `riscv64-unknown-linux-gnu` | RISC-V 64-bit Linux boards and servers |
| `pkgsCross.armv7l-hf-multiplatform` | `armv7l-unknown-linux-gnueabihf` | 32-bit ARM hard-float embedded devices |
| `pkgsCross.mingwW64` | `x86_64-w64-mingw32` | Windows 64-bit `.exe` / `.dll` binaries built on Linux |
| `pkgsStatic` | Native arch + `musl` static linking | Single-file static Linux binaries with minimal closures |

---

## The 6-slot dependency matrix and splicing

Cross-compilation in Nixpkgs works because `mkDerivation` classifies every dependency by **(where it runs during the build, what platform it targets)** using a 2D offset matrix:

| Nixpkgs attribute | Runs on | Targets | What belongs here |
|-------------------|---------|---------|-------------------|
| **`depsBuildBuild`** | `build` | `build` | Build-time tools that run on `build` and produce code for `build` (e.g., `buildPackages.stdenv.cc` for a build script or `cmake` host-tool compiler) |
| **`nativeBuildInputs`** (`depsBuildHost`) | `build` | `host` | Cross-compilers, `pkg-config`, `cmake`, `meson`, `ninja`, `bison`, `protobuf`, `cargo` — tools running on the builder that produce or configure `host` artifacts |
| **`depsBuildTarget`** | `build` | `target` | Cross-compilers targeting `targetPlatform` (compiler packaging only) |
| **`depsHostHost`** | `host` | `host` | Libraries for `host` when building a cross-compiler |
| **`buildInputs`** (`depsHostTarget`) | `host` | `target` (= `host` for apps) | Libraries and headers linked into your final application (`openssl`, `zlib`, `sqlite`, `liburing`) |
| **`depsTargetTarget`** | `target` | `target` | Runtime libraries for a compiler's target (e.g., `libgcc` / `libstdc++` when building GCC) |

For 95% of application packages, two slots do almost all the work:
- **`nativeBuildInputs`** = tools that **execute** on the build machine (`cmake`, `pkg-config`, `protobuf`, `rustc`, `bindgen`).
- **`buildInputs`** = libraries that the resulting binary **links against** (`openssl`, `zlib`, `sqlite`).

### How `callPackage` splicing makes this automatic

When you write:

```nix
{ stdenv, cmake, pkg-config, openssl, zlib }:
stdenv.mkDerivation {
  nativeBuildInputs = [ cmake pkg-config ];
  buildInputs = [ openssl zlib ];
  # ...
}
```

How does `pkg-config` come from `pkgsBuildHost` (so it runs on `x86_64` while querying `aarch64` `.pc` files), while `openssl` comes from `pkgsHostTarget` (so its `.so` files are `aarch64`)?

Nixpkgs **splices** packages together when instantiating `callPackage`: each spliced attribute carries internal references to its `pkgsBuildBuild`, `pkgsBuildHost`, and `pkgsHostTarget` variants. `stdenv.mkDerivation` inspects which list (`nativeBuildInputs` vs `buildInputs`) you placed the package into and selects the matching platform variant automatically.

> [!WARNING]
> **Don't break splicing with explicit `pkgs.*` indexing:** If you pass `pkgs` into a derivation and write `nativeBuildInputs = [ pkgs.pkg-config ]`, you may bypass splicing or pull the wrong host variant. Always request dependencies as top-level `callPackage` arguments, or explicitly reference `buildPackages.<tool>` when invoking a tool inside a shell snippet (`${lib.getExe buildPackages.protobuf}`).

### Using `stdenv.cc.targetPrefix` in custom build phases

If a Makefile or custom `buildPhase` hardcodes `gcc`, `ar`, or `strip`, it will invoke the wrong toolchain or fail to cross-compile. Pass the target-prefixed tools explicitly:

```nix
makeFlags = [
  "CC=${stdenv.cc.targetPrefix}cc"
  "CXX=${stdenv.cc.targetPrefix}c++"
  "AR=${stdenv.cc.targetPrefix}ar"
  "PKG_CONFIG=${stdenv.cc.targetPrefix}pkg-config"
];
```

On native builds, `stdenv.cc.targetPrefix` is `""` (empty string). On `aarch64-multiplatform`, it expands to `"aarch64-unknown-linux-gnu-"`.

---

## Language ecosystems in cross-compilation

### 1. C / C++ (CMake, Meson, Autotools)

Nixpkgs setup hooks automatically inject cross-compilation configuration when `buildPlatform != hostPlatform`:
- **Autotools:** passes `--build=<build> --host=<host>` to `./configure`.
- **CMake:** generates a toolchain file setting `CMAKE_SYSTEM_NAME`, `CMAKE_SYSTEM_PROCESSOR`, `CMAKE_C_COMPILER`, and `CMAKE_FIND_ROOT_PATH`.
- **Meson:** generates a `--cross-file` with `[binaries]`, `[host_machine]`, and `[properties]`.

```nix
# packages/sensor-service/package.nix
{
  lib,
  stdenv,
  fetchFromGitHub,
  cmake,
  pkg-config,
  openssl,
  zlib,
}:

stdenv.mkDerivation (finalAttrs: {
  pname = "sensor-service";
  version = "1.2.0";

  src = fetchFromGitHub {
    owner = "example-corp";
    repo = "sensor-service";
    rev = "v${finalAttrs.version}";
    hash = "sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=";
  };

  # Runs on buildPlatform (x86_64-linux)
  nativeBuildInputs = [
    cmake
    pkg-config
  ];

  # Linked for hostPlatform (aarch64-linux)
  buildInputs = [
    openssl
    zlib
  ];

  # Only run unit tests when the builder can execute host binaries
  doCheck = stdenv.buildPlatform.canExecute stdenv.hostPlatform;

  meta = {
    description = "C++ sensor processing service";
    platforms = lib.platforms.linux;
    mainProgram = "sensor-service";
  };
})
```

### 2. Rust (`rustPlatform.buildRustPackage`)

`rustPlatform.buildRustPackage` automatically configures `CARGO_BUILD_TARGET` and `CARGO_TARGET_<TRIPLE>_LINKER` when evaluated under `pkgsCross`.

Watch out for **build scripts (`build.rs`) and proc-macros**: they compile and execute on `buildPlatform`, not `hostPlatform`. If a crate's `build.rs` compiles C code with the `cc` crate or links a build-time library, add `buildPackages.stdenv.cc` to `depsBuildBuild`:

```nix
# packages/telemetry-agent/package.nix
{
  lib,
  stdenv,
  buildPackages,
  rustPlatform,
  fetchFromGitHub,
  pkg-config,
  openssl,
}:

rustPlatform.buildRustPackage (finalAttrs: {
  pname = "telemetry-agent";
  version = "0.4.1";

  src = fetchFromGitHub {
    owner = "example-corp";
    repo = "telemetry-agent";
    rev = "v${finalAttrs.version}";
    hash = "sha256-BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=";
  };

  cargoHash = "sha256-CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC=";

  # Needed when build.rs / proc-macros invoke a C compiler for the build machine
  depsBuildBuild = [ buildPackages.stdenv.cc ];

  nativeBuildInputs = [ pkg-config ];
  buildInputs = [ openssl ];

  # Expose host OpenSSL cleanly to openssl-sys under cross-compilation
  OPENSSL_NO_VENDOR = 1;

  doCheck = stdenv.buildPlatform.canExecute stdenv.hostPlatform;

  meta = {
    description = "Cross-compiled Rust telemetry agent";
    mainProgram = "telemetry-agent";
  };
})
```

### 3. Go (`buildGoModule` — pure Go vs CGO)

Go natively cross-compiles via `GOOS` and `GOARCH`, which `buildGoModule` sets from `stdenv.hostPlatform`.

- **Pure Go (`CGO_ENABLED = 0`):** Cross-compiles in seconds with no cross-C toolchain required.
- **CGO (`CGO_ENABLED = 1`):** Requires the cross C compiler (`stdenv.cc`) and host libraries in `buildInputs` (e.g., `sqlite`). `buildGoModule` wires `CC` automatically under `pkgsCross`.

```nix
# packages/gateway-api/package.nix
{
  lib,
  stdenv,
  buildGoModule,
  fetchFromGitHub,
}:

buildGoModule (finalAttrs: {
  pname = "gateway-api";
  version = "2.1.0";

  src = fetchFromGitHub {
    owner = "example-corp";
    repo = "gateway-api";
    rev = "v${finalAttrs.version}";
    hash = "sha256-DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDD=";
  };

  vendorHash = "sha256-EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE=";

  # Pure Go static cross-compilation; set to 1 + add buildInputs if linking C libs
  CGO_ENABLED = 0;

  doCheck = stdenv.buildPlatform.canExecute stdenv.hostPlatform;

  meta = {
    description = "Multi-arch Go API gateway";
    mainProgram = "gateway-api";
  };
})
```

---

## Flake pattern for multi-architecture & NixOS cross builds

### Exporting native, cross, and static packages

```nix
# flake.nix
{
  description = "Multi-Architecture Cross-Compilation Flake";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";
  };

  outputs = { self, nixpkgs }:
    let
      buildSystem = "x86_64-linux";
      pkgs = import nixpkgs { system = buildSystem; };
    in
    {
      packages.${buildSystem} = {
        # Native x86_64-linux build
        default = pkgs.callPackage ./packages/sensor-service/package.nix { };

        # Cross-compiled ARM64 glibc build (reuses pkgsCross directly)
        arm64 = pkgs.pkgsCross.aarch64-multiplatform.callPackage
          ./packages/sensor-service/package.nix { };

        # Cross-compiled static ARM64 musl build
        arm64-static = pkgs.pkgsCross.aarch64-multiplatform-musl.pkgsStatic.callPackage
          ./packages/sensor-service/package.nix { };

        # Cross-compiled RISC-V 64-bit build
        riscv64 = pkgs.pkgsCross.riscv64.callPackage
          ./packages/sensor-service/package.nix { };
      };

      # Cross-compiling a full NixOS ARM64 host configuration from an x86_64 builder
      nixosConfigurations.edge-arm64 = nixpkgs.lib.nixosSystem {
        modules = [
          {
            nixpkgs.buildPlatform = "x86_64-linux";
            nixpkgs.hostPlatform = "aarch64-linux";
          }
          ./hosts/edge-arm64/configuration.nix
        ];
      };
    };
}
```

Build directly from an `x86_64-linux` workstation or CI runner:

```bash
nix build .#arm64
nix build .#arm64-static
nix build .#riscv64
file ./result/bin/sensor-service
```

> [!TIP]
> **When to cross-compile vs use a remote native builder:** Cross-compiling a standalone C/Rust/Go service is fast and clean. Cross-compiling an *entire* NixOS desktop or heavy Python/Electron closure often hits unpatched upstream packages that try to execute host binaries at build time—for full OS images with large closures, a native ARM64 remote builder (see [Remote builders](10-remote-builders.md)) usually has higher cache hit rates from `cache.nixos.org`.

---

## Running cross-compiled tests selectively with an emulator

If you *want* to verify a cross-compiled binary locally without an ARM64 machine, you can run it under QEMU user-mode as a **post-build verification step** (compilation still ran at native x86_64 speed):

```bash
# Compile natively at x86_64 speed
nix build .#arm64

# Smoke-test the resulting ARM64 binary using qemu-user
nix run nixpkgs#qemu-user -- ./result/bin/sensor-service --version
```

Inside a Nix expression, `stdenv.hostPlatform.emulator buildPackages` resolves to the right QEMU user binary (or empty string on native platforms).

---

## Common cross-compilation pitfalls

| Failure symptom | Root cause | Fix |
|-----------------|------------|-----|
| `Exec format error` during build | A `hostPlatform` binary executed on `buildPlatform` (unit tests, code generator, or `help2man`) | Guard tests with `doCheck = stdenv.buildPlatform.canExecute stdenv.hostPlatform;` and move code generators to `nativeBuildInputs` (or invoke `buildPackages.<tool>`) |
| `cannot find -lssl` or wrong ELF class | Target library placed in `nativeBuildInputs` instead of `buildInputs` | Move all linked libraries into `buildInputs` |
| `pkg-config` returns x86_64 paths or fails to find host `.pc` files | Build script invoked unprefixed `pkg-config` or `pkg-config` was placed in `buildInputs` | Put `pkg-config` in `nativeBuildInputs` and pass `PKG_CONFIG=${stdenv.cc.targetPrefix}pkg-config` if the build system ignores `PKG_CONFIG` |
| Rust `build.rs` fails with linker error or wrong architecture | `build.rs` compiled C code using the cross-compiler instead of the build compiler | Add `depsBuildBuild = [ buildPackages.stdenv.cc ];` |
| `configure: error: cannot run C compiled programs` | Autotools `AC_RUN_IFELSE` check cannot execute cross-compiled test binary | Supply the cached autoconf result via `configureFlags = [ "ac_cv_func_malloc_0_nonnull=yes" ];` |
| Cross-build compiles hundreds of core packages from scratch | `pkgsCross.*` has lower coverage on `cache.nixos.org` than native `x86_64-linux` / `aarch64-linux` | Push cross-toolchain closures to a shared team binary cache (Attic/Cachix) so CI only builds them once |

---

## Lab — native vs cross vs static verification

1. **Cross-compile an upstream package:** Run `nix build nixpkgs#pkgsCross.aarch64-multiplatform.hello` and inspect `./result/bin/hello` with `file` and `readelf -d`.
2. **Compare dynamic vs static closure size:** Build `nixpkgs#pkgsCross.aarch64-multiplatform-musl.pkgsStatic.hello`, compare `nix path-info -rS` against the glibc build, and verify `readelf -l` shows no `PT_INTERP` dynamic loader.
3. **Break and fix `nativeBuildInputs`:** In a small C/CMake or Rust package, move `pkg-config` into `buildInputs`, observe the cross-build failure, then restore it to `nativeBuildInputs`.
4. **Smoke-test with QEMU user-mode:** Run the cross-compiled ARM64 binary using `nix run nixpkgs#qemu-user -- ./result/bin/hello`.

---

## Checkpoint

- [ ] Distinguished `buildPlatform`, `hostPlatform`, and `targetPlatform` and explained when `targetPlatform != hostPlatform`.
- [ ] Classified dependencies accurately across `depsBuildBuild`, `nativeBuildInputs` (`depsBuildHost`), and `buildInputs` (`depsHostTarget`).
- [ ] Cross-compiled C/C++, Rust, or Go binaries using `pkgsCross.aarch64-multiplatform` and `pkgsStatic` without QEMU during compilation.
- [ ] Used `stdenv.cc.targetPrefix` and `stdenv.buildPlatform.canExecute stdenv.hostPlatform` cleanly in derivations.
- [ ] Decided accurately when to cross-compile vs offload to a native ARM64 remote builder.
