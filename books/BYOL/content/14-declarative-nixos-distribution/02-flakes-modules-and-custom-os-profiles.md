# Flakes, Modules, and Custom OS Profiles

Imperative Linux distributions assemble a system by running post-install scripts in sequence, mutating `/etc` and `/usr` until the machine reaches an approximate target state. NixOS replaces that script pipeline with a pure functional evaluation model. Two mechanisms make custom distributions practical at scale: **Nix Flakes**, which pin every upstream input to a cryptographic hash, and the **NixOS Module System**, which merges dozens of independent configuration files into a single validated attribute set before a single byte is built.

## Pinning the Entire OS Graph with `flake.nix`

A traditional distribution tracks dependencies across package repositories, release mirrors, and mutable branch names. A NixOS flake pins the exact Git revision and NAR hash of `nixpkgs`, custom overlays, and third-party modules inside `flake.lock`. Anyone who runs `nix build` against the same `flake.lock` evaluates the exact same Nix expressions and produces the same store derivation paths.

```nix
{
  description = "Custom BYOL declarative Linux distribution";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
    disko = {
      url = "github:nix-community/disko";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { self, nixpkgs, disko, ... }@inputs:
    let
      system = "x86_64-linux";
      pkgs = nixpkgs.legacyPackages.${system};
    in
    {
      nixosConfigurations.byol-workstation = nixpkgs.lib.nixosSystem {
        inherit system;
        specialArgs = { inherit inputs; };
        modules = [
          disko.nixosModules.disko
          ./profiles/base.nix
          ./profiles/kernel-hardening.nix
          ./hosts/workstation/configuration.nix
        ];
      };

      formatter.${system} = pkgs.nixpkgs-fmt;
    };
}
```

The `inputs.nixpkgs.follows = "nixpkgs"` directive is critical in multi-input distributions. Without it, each flake input evaluates its own pinned copy of `nixpkgs`, multiplying evaluation memory and compiling duplicate toolchain closures.

> [!TIP]
> Pass `specialArgs` only for values that must be available while resolving `imports` (such as `inputs`). For normal runtime configuration values, define typed module options instead so the module system can validate types and detect conflicts.

## How the NixOS Module System Merges Configuration

Every NixOS module is a function that receives `{ config, lib, pkgs, ... }` and returns an attribute set containing `imports`, `options`, and `config`. Under the hood, `lib.evalModules` collects every module in the import graph, evaluates all `options` declarations first, and then merges all `config` definitions using type-specific merge functions:

- **Attribute sets and lists** concatenate cleanly across modules (for example, `environment.systemPackages` merges lists from every profile).
- **Scalar types** (booleans, strings, integers) enforce uniqueness: if two modules set `networking.hostName` to different values at the same priority, evaluation fails immediately instead of silently letting the last file win.
- **Explicit priorities** resolve intentional overrides: `lib.mkDefault` sets priority `1000` (overridable baseline), normal assignments use `100`, and `lib.mkForce` sets priority `50` (hard override).

### Authoring a Typed Distribution Profile Module

When building your own distribution on NixOS, encapsulate your opinionated defaults inside a dedicated namespace (such as `byol.*`) with strict option types and documentation strings.

```nix
# profiles/base.nix
{ config, lib, pkgs, ... }:

let
  cfg = config.byol.profile;
in
{
  options.byol.profile = {
    role = lib.mkOption {
      type = lib.types.enum [ "minimal" "server" "workstation" ];
      default = "minimal";
      description = "High-level distribution role selecting default packages and services.";
    };

    enableSystemdInitrd = lib.mkOption {
      type = lib.types.bool;
      default = true;
      description = "Use systemd inside stage-1 initrd for parallel device discovery.";
    };

    extraPackages = lib.mkOption {
      type = lib.types.listOf lib.types.package;
      default = [ ];
      description = "Additional packages appended to the system environment.";
    };
  };

  config = {
    # Stage-1 initrd and kernel defaults
    boot.initrd.systemd.enable = lib.mkDefault cfg.enableSystemdInitrd;
    boot.kernelPackages = lib.mkDefault pkgs.linuxPackages_latest;
    boot.tmp.useTmpfs = lib.mkDefault true;

    # Deterministic system package closure
    environment.systemPackages =
      (with pkgs; [
        coreutils
        iproute2
        jq
        lsof
        procps
        util-linux
      ])
      ++ lib.optionals (cfg.role == "workstation") (with pkgs; [
        git
        ripgrep
        tmux
      ])
      ++ cfg.extraPackages;

    # Enable flakes and clean store links automatically
    nix.settings = {
      experimental-features = [ "nix-command" "flakes" ];
      auto-optimise-store = lib.mkDefault true;
    };

    system.stateVersion = "24.11";
  };
}
```

## Customizing the Kernel and Initrd Declaratively

In Chapters 2 and 4, you configured the Linux kernel with `make menuconfig` and assembled an initramfs by hand. In NixOS, those same kernel configuration knobs live directly in the module tree, and Nix rebuilds only the kernel derivation when `structuredExtraConfig` changes.

```nix
# profiles/kernel-hardening.nix
{ lib, pkgs, ... }:

{
  boot.kernelPatches = [
    {
      name = "byol-custom-kconfig";
      patch = null;
      structuredExtraConfig = with lib.kernel; {
        # Enable eBPF JIT and BTF metadata for modern observability tools
        BPF_JIT = yes;
        DEBUG_INFO_BTF = yes;
        # Trim legacy filesystems not needed on modern appliances
        REISERFS_FS = no;
        JFS_FS = no;
      };
    }
  ];

  # Load only required stage-1 kernel modules
  boot.initrd.availableKernelModules = [
    "nvme"
    "virtio_pci"
    "virtio_blk"
    "ext4"
  ];
}
```

> [!IMPORTANT]
> Changing `boot.kernelPatches` alters the kernel derivation hash, which means Nix cannot substitute a prebuilt kernel from `cache.nixos.org` and will compile the kernel locally (or pull it from your own binary cache). Test kernel config tweaks on a fast builder or use `boot.kernelParams` for runtime-only tuning when possible.

## Overlays: Patching Upstream Packages Without Forking

When your distribution needs a custom compile flag or patch on an upstream package, use a **nixpkgs overlay** instead of maintaining a forked repository. An overlay is a function `final: prev:` where `prev` is the original package set and `final` is the fixed-point package set after all overlays are applied.

```nix
# overlays/custom-tools.nix
final: prev: {
  byol-htop = prev.htop.overrideAttrs (old: {
    configureFlags = (old.configureFlags or [ ]) ++ [ "--enable-sensors" ];
  });
}
```

Because `overrideAttrs` derives a new package from `prev.htop`, every security update in `nixpkgs` flows into `byol-htop` automatically the next time you run `nix flake update`.

## Hands-On Lab: Validate and Inspect a Multi-Profile Flake

This lab builds a minimal NixOS flake with two profiles (`minimal` and `workstation`) and uses `nix eval` and `nix flake check` to verify module merging without needing root privileges or a full VM build.

### Step 1: Scaffold the Flake Directory

```bash
mkdir -p /tmp/byol-nix-lab/profiles
cd /tmp/byol-nix-lab
```

Create `profiles/base.nix` using the typed module above, then add a minimal `flake.nix` defining two host configurations:

```nix
# /tmp/byol-nix-lab/flake.nix
{
  description = "BYOL Module Merge Lab";
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";

  outputs = { self, nixpkgs }:
    let
      mkHost = role: nixpkgs.lib.nixosSystem {
        system = "x86_64-linux";
        modules = [
          ./profiles/base.nix
          {
            fileSystems."/".device = "/dev/vda1";
            boot.loader.grub.device = "nodev";
            byol.profile.role = role;
            networking.hostName = "byol-${role}";
          }
        ];
      };
    in
    {
      nixosConfigurations = {
        minimal = mkHost "minimal";
        workstation = mkHost "workstation";
      };
    };
}
```

### Step 2: Evaluate Merged Options Without Building

```bash
# Lock inputs and verify flake schema
nix flake check --no-build

# Inspect the resolved hostname and package count for each profile
nix eval .#nixosConfigurations.minimal.config.networking.hostName
nix eval .#nixosConfigurations.minimal.config.environment.systemPackages --apply builtins.length
nix eval .#nixosConfigurations.workstation.config.environment.systemPackages --apply builtins.length
```

### Step 3: Trigger and Diagnose a Priority Conflict

Add `networking.hostName = "conflict-node";` inside `profiles/base.nix` without `lib.mkDefault` and re-run `nix eval .#nixosConfigurations.minimal.config.networking.hostName`. Observe how NixOS reports the exact two file locations that defined conflicting values—then wrap the baseline in `lib.mkDefault` and confirm evaluation succeeds cleanly.

## Next Steps

With a pinned, modular OS definition in place, the next chapter compiles that flake directly into bootable hybrid ISOs, raw disk images, and declaratively partitioned appliances in [Building Custom NixOS ISOs and Appliances](./03-building-custom-nixos-isos-and-appliances.md).
