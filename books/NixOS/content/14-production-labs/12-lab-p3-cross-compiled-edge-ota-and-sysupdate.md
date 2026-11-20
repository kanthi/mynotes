---
title: 'Lab P3: Cross-Compiled ARM64 Edge Images, DeviceTree, and systemd-sysupdate OTA'
---

# Lab P3: Cross-Compiled ARM64 Edge Images, DeviceTree, and systemd-sysupdate OTA

Compiling an `aarch64-linux` kernel and robotics stack through QEMU `binfmt` emulation takes hours, and running `nixos-rebuild switch` over a lossy satellite or tactical radio link risks leaving a drone or edge box half-updated. The boring default for an embedded ARM64 / Jetson fleet is: **cross-compile the complete `aarch64-linux` system closure and SD image natively on `x86_64-linux` (`nixpkgs.crossSystem`), declare hardware buses via `hardware.deviceTree.overlays`, auto-enroll into a WireGuard mesh on first boot, and ship atomic A/B partition OTA images via `systemd-sysupdate`**.

## Mental model

```text
+-------------------------------------------------------------------+
|  x86_64 Build Host (CI Server)                                    |
|  - nixpkgs.buildPlatform = "x86_64-linux"                         |
|  - nixpkgs.hostPlatform  = "aarch64-linux" (Zero QEMU overhead)   |
|  - DeviceTree overlays (SPI/CAN/GPIO) + WireGuard + sysupdate     |
+---------------------------------+---------------------------------+
                                  |
                                  v (< 15 min cross-compile)
                      +-----------------------+
                      | Raw SD Image + A/B    |
                      | Partition Payloads    |
                      +-----------+-----------+
                                  |
                                  v
+---------------------------------+---------------------------------+
|  WireGuard Mesh Overlay Network (10.100.90.0/24)                  |
|                                                                   |
|  +------------------------------+   +---------------------------+ |
|  | edge-arm-01 (10.100.90.21)   |   | edge-arm-02 (10.100.90.22)| |
|  | - Active: Slot A (v26.05.1)  |   | - Active: Slot B (v26.05.2)| |
|  | - Standby: Slot B            |   | - Rollback: automatic     | |
|  +------------------------------+   +---------------------------+ |
+-------------------------------------------------------------------+
```

| Requirement | Mechanism | Why Not Standard Desktop NixOS? |
|---|---|---|
| Fast `aarch64` build on `x86_64` CI | `nixpkgs.buildPlatform` ≠ `nixpkgs.hostPlatform` | Native `aarch64-unknown-linux-gnu-gcc` cross-compiler is 8–10x faster than `binfmt` QEMU |
| Expose CAN / SPI / UART pins | `hardware.deviceTree.overlays` (`.dts`) | ARM64 SoCs discover peripherals from compiled `.dtb` trees, not PCIe enumeration |
| Resilient OTA over radio link | `systemd-sysupdate` + `systemd-boot` boot counting | Streams a compressed partition image into inactive Slot B; rolls back to Slot A if Slot B fails boot health |

## Worked examples

### Case 1: Pure cross-compiled `aarch64-linux` NixOS configuration in `flake.nix`

Setting `nixpkgs.buildPlatform = "x86_64-linux"` and `nixpkgs.hostPlatform = "aarch64-linux"` inside a NixOS module tells Nixpkgs to compile every package with the `x86_64 → aarch64` cross toolchain without invoking QEMU emulation.

Save as `flake.nix`:

```nix
# flake.nix
{
  description = "Lab P3 — Cross-Compiled ARM64 Edge Fleet";

  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";

  outputs = { self, nixpkgs }: {
    nixosConfigurations.edge-arm-01 = nixpkgs.lib.nixosSystem {
      modules = [
        "${nixpkgs}/nixos/modules/installer/sd-card/sd-image-aarch64.nix"
        ./hosts/edge-arm-01.nix
        {
          nixpkgs.buildPlatform = "x86_64-linux";
          nixpkgs.hostPlatform = "aarch64-linux";
        }
      ];
    };

    # Expose the flashable .img.zst directly on the x86_64 CI server
    packages.x86_64-linux.edge-arm-01-sd =
      self.nixosConfigurations.edge-arm-01.config.system.build.sdImage;
  };
}
```

### Case 2: DeviceTree overlay for SPI / CAN bus and custom kernel patch

On Raspberry Pi 4, Industrial CM4 carriers, or Jetson Orin boards, enable hardware buses by compiling `.dts` overlays via `hardware.deviceTree.overlays` and patching kernel config flags cleanly.

Save as `modules/edge-hardware.nix`:

```nix
# modules/edge-hardware.nix
{ pkgs, ... }:
{
  # Don't bloat the embedded closure with missing desktop storage drivers
  nixpkgs.overlays = [
    (final: prev: {
      makeModulesClosure = x:
        prev.makeModulesClosure (x // { allowMissing = true; });
    })
  ];

  # Custom kernel configuration for CAN bus & hardware watchdog
  boot.kernelPatches = [
    {
      name = "enable-can-and-watchdog";
      patch = null;
      extraConfig = ''
        CAN y
        CAN_RAW y
        CAN_DEV y
        CAN_MCP251X m
        WATCHDOG y
      '';
    }
  ];

  # Declarative DeviceTree overlay exposing SPI0 for an MCP2515 CAN controller
  hardware.deviceTree = {
    enable = true;
    overlays = [
      {
        name = "spi0-can0-overlay";
        dtsText = ''
          /dts-v1/;
          /plugin/;
          / {
            compatible = "brcm,bcm2711";
            fragment@0 {
              target-path = "/soc/spi@7e204000";
              __overlay__ {
                status = "okay";
              };
            };
          };
        '';
      }
    ];
  };
}
```

### Case 3: Automated WireGuard mesh enrollment and `systemd-sysupdate` A/B slots

When the flashed board powers on in the field, it loads its pre-provisioned WireGuard key from the persistent `/persist` partition, joins the `10.100.90.0/24` fleet overlay, and checks the internal update server for new signed A/B partition images via `systemd-sysupdate`.

Save as `hosts/edge-arm-01.nix`:

```nix
# hosts/edge-arm-01.nix
{ config, pkgs, ... }:
{
  imports = [ ../modules/edge-hardware.nix ];

  networking.hostName = "edge-arm-01";
  system.stateVersion = "26.05";

  # 1. Automatic WireGuard mesh enrollment on boot
  networking.wireguard.interfaces.wg0 = {
    ips = [ "10.100.90.21/24" ];
    privateKeyFile = "/persist/secrets/wg0.key";
    peers = [
      {
        publicKey = "wX8yZ9aB0cD1eF2gH3iJ4kL5mN6oP7qR8sT9uV0wXyZ=";
        allowedIPs = [ "10.100.90.0/24" ];
        endpoint = "203.0.113.10:51820";
        persistentKeepalive = 25;
      }
    ];
  };

  # 2. systemd-sysupdate rule writing OTA rootfs payloads into the inactive A/B partition slot
  environment.etc."sysupdate.d/50-rootfs.transfer".text = ''
    [Transfer]
    ProtectVersion=%A

    [Source]
    Type=url-file
    Path=http://10.100.90.1/ota/
    MatchPattern=edge-arm-rootfs_@v.raw.zst

    [Target]
    Type=partition
    Path=/dev/disk/by-diskseq/1
    MatchPattern=root-@v
    MatchPartitionType=root-arm64
    ReadOnly=yes
  '';

  # 3. Boot health gate: mark A/B slot good only after wg0 and telemetry are healthy
  systemd.services.ota-boot-bless = {
    description = "Bless current A/B boot slot after mesh & telemetry health check";
    wantedBy = [ "multi-user.target" ];
    after = [ "wireguard-wg0.service" ];
    serviceConfig.Type = "oneshot";
    script = ''
      ${pkgs.iputils}/bin/ping -c 3 -W 5 10.100.90.1
      ${pkgs.systemd}/lib/systemd/systemd-bless-boot good
    '';
  };

  # Hardware watchdog reboots into the previous healthy A/B slot if boot hangs
  systemd.settings.Manager = {
    RuntimeWatchdogSec = "30s";
    RebootWatchdogSec = "2m";
  };
}
```

### Case 4: Verification — cross-compile under 15 minutes and test A/B OTA rollback

1. **Cross-compile the flashable SD image on the `x86_64-linux` host:**

```bash
time nix build .#packages.x86_64-linux.edge-arm-01-sd -L
ls -lh result/sd-image/*.img.zst
```

Verify that no QEMU `binfmt` processes ran during compilation and that the compressed `.img.zst` finishes building in under 15 minutes.

2. **Verify A/B boot counting rollback:**
   When `systemd-sysupdate update` writes version `26.05.2` into inactive Slot B, it labels the boot entry with a retry counter (`edge-arm-26.05.2+3-0.efi` — 3 tries left, 0 boots tried).
   - If `26.05.2` boots and `ota-boot-bless.service` successfully pings the WireGuard gateway `10.100.90.1`, `systemd-bless-boot good` renames the entry to `edge-arm-26.05.2.efi` (permanently blessed).
   - If `26.05.2` has a broken kernel or broken WireGuard config, `ota-boot-bless.service` fails, the hardware watchdog reboots the board, the counter decrements to `+0-3`, and `systemd-boot` automatically falls back to Slot A (`26.05.1`).

## The trap

The trap is **accidentally referencing `pkgs.foo` from the build platform or running a package's native test suite (`doCheck = true`) during cross-compilation**. If `boot.binfmt.emulatedSystems` is enabled on your CI server while you test `nixpkgs.crossSystem`, a broken derivation that mistakenly executes a host binary at build time will silently fall back to slow QEMU emulation instead of failing fast. Disable `binfmt` on dedicated cross-compilation runners so cross boundary bugs are caught immediately.

The second trap is **shipping A/B `systemd-sysupdate` updates without configuring `RuntimeWatchdogSec` and `systemd-bless-boot`**. Without boot counting (`+3-0` suffix on the loader entry) and a hardware watchdog, a kernel panic in Slot B leaves the remote device frozen until someone physically power-cycles and switches slots.

## The boring rule

- Use `nixpkgs.buildPlatform = "x86_64-linux"` and `nixpkgs.hostPlatform = "aarch64-linux"` for embedded ARM64 image builds; keep `binfmt` off on the cross-compile CI runner.
- Declare board peripherals (`SPI`, `I2C`, `CAN`, `UART`) in `hardware.deviceTree.overlays` inside git, never by editing `/boot/config.txt` on the SD card by hand.
- Pair `systemd-sysupdate` A/B partition transfers with `systemd-bless-boot` gated on WireGuard mesh reachability (`ping 10.100.90.1`) and a hardware watchdog (`RuntimeWatchdogSec = "30s"`).

## Try this

1. Inspect the cross-compiler triple used by the system closure with `nix eval .#nixosConfigurations.edge-arm-01.pkgs.stdenv.cc.targetPrefix`.
2. Decompress the built SD image (`zstd -d result/sd-image/*.img.zst -o /tmp/edge.img`) and inspect its partitions with `fdisk -l /tmp/edge.img`.
3. Run `/lib/systemd/systemd-sysupdate --verify=no list` inside a test VM to see how `sysupdate.d/*.transfer` matches active vs. standby partition slots.
