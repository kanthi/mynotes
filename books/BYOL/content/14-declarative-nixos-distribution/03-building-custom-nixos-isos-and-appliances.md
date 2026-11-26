# Building Custom NixOS ISOs and Appliances

In [Chapter 10](../10-iso-creation-and-installation/02-embedding-bootloader-kernel-squashfs.md), you assembled a bootable hybrid ISO by manually compiling a kernel, packing an initramfs, compressing a root filesystem into SquashFS, and running `xorriso`. NixOS turns that entire image-assembly pipeline into a pure derivation: the same `flake.nix` that defines your installed system can also emit a live ISO, a cloud VM image, or a declaratively partitioned bare-metal disk image.

## Compiling a Custom Live ISO Directly from a Flake

`nixpkgs` ships reusable image-builder modules under `nixos/modules/installer/cd-dvd/`. Importing `installation-cd-minimal.nix` wires up the live stage-1 initrd, SquashFS compression of the `/nix/store` closure, UEFI and legacy BIOS bootloaders, and `xorriso` ISO mastering automatically.

```nix
# flake.nix excerpt adding an ISO output
{
  description = "BYOL Custom NixOS Live ISO";
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";

  outputs = { self, nixpkgs }:
    let
      system = "x86_64-linux";
      isoSystem = nixpkgs.lib.nixosSystem {
        inherit system;
        modules = [
          "${nixpkgs}/nixos/modules/installer/cd-dvd/installation-cd-minimal.nix"
          ./profiles/base.nix
          ({ pkgs, lib, ... }: {
            byol.profile.role = "minimal";
            networking.hostName = "byol-live";

            # Fast, high-ratio SquashFS compression for smaller ISOs
            isoImage.squashfsCompression = "zstd -Xcompression-level 15";
            isoImage.volumeID = "BYOL_NIXOS_LIVE";
            isoImage.isoBaseName = "byol-nixos";

            # Pre-bake recovery and hardware inspection utilities
            environment.systemPackages = with pkgs; [
              btrfs-progs
              dosfstools
              gptfdisk
              nvme-cli
              parted
            ];

            # Keep the live image lightweight by omitting local docs
            documentation.enable = lib.mkForce false;
          })
        ];
      };
    in
    {
      packages.${system}.iso = isoSystem.config.system.build.isoImage;
    };
}
```

Running `nix build .#iso` evaluates the closure, builds the SquashFS image containing every referenced store path, packages the EFI System Partition and El Torito boot catalog, and writes the final `.iso` to `./result/iso/byol-nixos-*.iso`.

> [!NOTE]
> Because `system.build.isoImage` is a standard Nix derivation, rebuilding the ISO after adding a single CLI tool reuses every unchanged store path from `/nix/store` and only regenerates the SquashFS and ISO layers.

## Declarative Disk Partitioning with `disko`

Traditional installers separate OS configuration from disk layout: you click through a partitioner or maintain fragile `sfdisk` shell scripts before installing packages. **Disko** extends the NixOS module system so GPT tables, EFI System Partitions, Btrfs subvolumes, and `/etc/fstab` mount entries are generated from one Nix file.

```nix
# disko/standard-gpt.nix
{
  disko.devices = {
    disk.main = {
      type = "disk";
      device = "/dev/vda";
      content = {
        type = "gpt";
        partitions = {
          ESP = {
            priority = 1;
            name = "ESP";
            start = "1M";
            end = "512M";
            type = "EF00";
            content = {
              type = "filesystem";
              format = "vfat";
              mountpoint = "/boot";
              mountOptions = [ "umask=0077" ];
            };
          };
          root = {
            size = "100%";
            content = {
              type = "btrfs";
              extraArgs = [ "-f" ];
              subvolumes = {
                "@root" = {
                  mountpoint = "/";
                  mountOptions = [ "compress=zstd" "noatime" ];
                };
                "@nix" = {
                  mountpoint = "/nix";
                  mountOptions = [ "compress=zstd" "noatime" ];
                };
                "@var-log" = {
                  mountpoint = "/var/log";
                  mountOptions = [ "compress=zstd" "noatime" ];
                };
              };
            };
          };
        };
      };
    };
  };
}
```

Separating `@nix` into its own Btrfs subvolume lets you take snapshots of `@root` or wipe ephemeral root state on boot without discarding the immutable `/nix/store`.

## Verified Boot and Immutable Appliance Patterns

For edge appliances and workstations that require cryptographic boot integrity (extending the chain of trust from [Chapter 9](../09-security-foundations/03-secure-boot-and-chain-of-trust.md)), NixOS integrates cleanly with UEFI Secure Boot via **Lanzaboote** and systemd-based Unified Kernel Images (UKIs):

| Appliance Layer | Traditional BYOL Mechanism | Declarative NixOS Equivalent |
| :--- | :--- | :--- |
| Bootloader & UKI | Manual `ukify` + `sbsign` scripts | `lanzaboote` module signing every generation's UKI automatically |
| Disk Layout | Imperative `parted` + `mkfs` scripts | `disko.devices` generating both partition script and `fileSystems` |
| Root Filesystem | Mutable `/etc` and `/var` | Ephemeral `tmpfs` or rolled-back Btrfs `@root` + persistent `/nix` |
| System Updates | `pacman`/`apt` in-place mutation | Atomic generation switch (`nixos-rebuild switch` or `systemd-sysupdate`) |

```nix
# Example Lanzaboote configuration snippet for signed UKIs
{ lib, ... }:

{
  boot.loader.systemd-boot.enable = lib.mkForce false;
  boot.lanzaboote = {
    enable = true;
    pkiBundle = "/var/lib/sbctl";
  };
}
```

> [!WARNING]
> Before enabling UEFI Secure Boot enforcement in firmware, enroll your custom Platform Key (PK), Key Exchange Key (KEK), and Signature Database (db) keys while firmware is in Setup Mode, and test booting a signed generation at least once.

## Generating Multiple Image Formats from One Configuration

Beyond live ISOs, community tooling such as `nixos-generators` (or built-in `nixpkgs` image modules) compiles the exact same NixOS configuration into target-specific formats:

- **QEMU/KVM (`qcow2`)**: Import `${nixpkgs}/nixos/modules/virtualisation/qemu-vm.nix` and build `.config.system.build.vm` for instant local VM launches that share the host `/nix/store` over virtiofs.
- **Raw Disk Image (`raw-efi`)**: Import `disko`'s image builder (`system.build.diskoImages`) to produce a pre-partitioned raw image ready to flash directly onto NVMe or SD media.
- **OCI Container (`docker`)**: Build minimal service closures with `pkgs.dockerTools.buildLayeredImage` sharing the same pinned `pkgs` revision as your host OS.

## Hands-On Lab: Build and Boot a Custom NixOS Live ISO in QEMU

This lab builds the custom live ISO defined above and boots it inside QEMU to verify the SquashFS root, kernel version, and pre-installed tools.

### Step 1: Build the ISO Derivation

```bash
cd /tmp/byol-nix-lab
# Ensure flake.nix exposes packages.x86_64-linux.iso
nix build .#iso --print-build-logs
ls -lh result/iso/
```

### Step 2: Inspect the ISO Layout Without Booting

Verify that the output is a hybrid ISO containing an El Torito boot catalog and SquashFS payload:

```bash
ISO_PATH=$(ls result/iso/*.iso | head -n 1)
file "$ISO_PATH"
xorriso -indev "$ISO_PATH" -toc 2>/dev/null | head -n 25
```

### Step 3: Boot the Live ISO in QEMU

```bash
qemu-system-x86_64 \
  -m 2048 \
  -smp 2 \
  -cdrom "$ISO_PATH" \
  -nographic \
  -serial mon:stdio
```

Once the live shell appears, verify your custom profile settings inside the guest:

```bash
hostname
uname -r
which nvme parted gdisk
df -h / /nix/.ro-store
```

You should see `byol-live`, your selected kernel release, all pre-baked disk tools, and the read-only SquashFS mounted under `/nix/.ro-store` overlaid with a writable tmpfs.

## Next Steps

Building an ISO proves the image boots once; maintaining a custom distribution over time requires automated multi-VM integration tests, binary caching, and safe fleet rollouts. Continue to [Declarative Testing and Fleet Deployment](./04-declarative-testing-and-fleet-deployment.md).
