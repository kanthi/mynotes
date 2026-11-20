---
title: 'Lab H3: Lanzaboote Secure Boot, TPM2 Auto-Unlock, and CycloneDX SBOM'
---

# Lab H3: Lanzaboote Secure Boot, TPM2 Auto-Unlock, and CycloneDX SBOM

Defense and autonomous hardware platforms require cryptographic proof of what booted and an encrypted disk that unlocks unattended only when the firmware and kernel signatures remain intact. The boring default for testing this in a single-machine KVM lab is: **emulate UEFI OVMF + `swtpm` in QEMU, sign Unified Kernel Images (UKIs) with `sbctl` and `lanzaboote`, bind LUKS2 auto-unlock to TPM2 PCRs `0+7` via `systemd-cryptenroll`, and emit a CycloneDX SBOM of the exact system derivation with `sbomnix`**.

## Mental model

Standard `systemd-boot` loads an unsigned kernel and initrd from `/boot`. **Lanzaboote** replaces that flow by packaging the kernel, initrd, and kernel command line into a single PE binary (**Unified Kernel Image / UKI**) signed by your Database Key (`db`). At boot:

1. **UEFI Secure Boot** verifies the UKI signature against `PK`/`KEK`/`db` stored in OVMF NVRAM (measured into **PCR 7**).
2. **Stage-1 `systemd` initrd** requests the LUKS2 volume key from the **TPM2** chip (`swtpm`), which unseals the secret only if **PCR 0** (firmware) and **PCR 7** (Secure Boot policy & keys) match the enrolled state.
3. An unsigned kernel or tampered Secure Boot state changes PCR 7 (or fails UEFI verification), refusing to unlock the disk without the offline recovery passphrase.

```text
OVMF UEFI Firmware (PCR 0) + Enrolled PK/KEK/db (PCR 7)
    │
    ├──► Refuses unsigned kernel image immediately
    ▼
Signed Lanzaboote UKI (kernel + systemd initrd + cmdline)
    │
    ▼
systemd-cryptsetup @ Stage 1  ──►  TPM2 (swtpm) unseals LUKS2 key iff PCR 0+7 match
    │
    ▼
Hardened NixOS Userland (usb-storage blacklisted, CycloneDX SBOM archived)
```

## Worked examples

### Case 1: Launch a KVM guest with OVMF Secure Boot and `swtpm`

On your KVM host, start a virtual TPM2 socket (`swtpm`) and boot a QEMU guest with OVMF (`OVMF_CODE.secboot.fd`) and a writable `OVMF_VARS.fd` in Setup Mode:

```bash
mkdir -p /tmp/lab-h3-tpm
swtpm socket --tpm2 \
  --tpmstate dir=/tmp/lab-h3-tpm \
  --ctrl type=unixio,path=/tmp/lab-h3-tpm/swtpm-sock \
  --daemon

# In QEMU args (or libvirt domain XML <tpm model='tpm-crb' backend='emulator' version='2.0'/>):
# -chardev socket,id=chrtpm,path=/tmp/lab-h3-tpm/swtpm-sock
# -tpmdev emulator,id=tpm0,chardev=chrtpm
# -device tpm-tis,tpmdev=tpm0
```

Partition `/dev/vda` with a `512M` EFI System Partition (`/boot`) and a LUKS2-encrypted root partition (`/dev/vda2` → `/dev/mapper/cryptroot`).

### Case 2: Configure Lanzaboote, systemd Stage 1, and kernel hardening in the flake

Save as `flake.nix`:

```nix
# flake.nix
{
  description = "Lab H3 — Hardened Lanzaboote + TPM2 NixOS Host";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";
    lanzaboote = {
      url = "github:nix-community/lanzaboote";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { self, nixpkgs, lanzaboote }: {
    nixosConfigurations.anduril-lab = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        lanzaboote.nixosModules.lanzaboote
        ./hosts/anduril-lab.nix
      ];
    };
  };
}
```

Save as `hosts/anduril-lab.nix`:

```nix
# hosts/anduril-lab.nix
{ lib, pkgs, ... }:
{
  networking.hostName = "anduril-lab";
  system.stateVersion = "26.05";

  environment.systemPackages = [ pkgs.sbctl pkgs.tpm2-tools ];

  # 1. Lanzaboote replaces stock systemd-boot installer
  boot.loader.systemd-boot.enable = lib.mkForce false;
  boot.lanzaboote = {
    enable = true;
    pkiBundle = "/var/lib/sbctl";
  };

  # 2. systemd Stage 1 initrd for native TPM2 LUKS2 auto-unlock
  boot.initrd.systemd.enable = true;
  boot.initrd.luks.devices."cryptroot" = {
    device = "/dev/disk/by-partlabel/disk-main-luks";
    crypttabExtraOpts = [ "tpm2-device=auto" ];
  };

  # 3. OS & physical peripheral hardening
  boot.blacklistedKernelModules = [
    "usb-storage"
    "uas"
    "firewire-core"
    "thunderbolt"
  ];
  boot.kernelParams = [ "lockdown=confidentiality" ];

  # Disable serial getty consoles and tighten sudo
  systemd.services."serial-getty@ttyS0".enable = false;
  security.sudo.execWheelOnly = true;
  security.sudo.extraConfig = "Defaults timestamp_timeout=0";
}
```

### Case 3: Enroll Secure Boot keys (`sbctl`) and bind LUKS2 to TPM2 PCR 0+7

Inside `anduril-lab`, generate the Secure Boot PKI bundle (`PK`, `KEK`, `db`), rebuild so Lanzaboote signs the UKIs in `/boot/EFI/Linux/`, and enroll the keys into OVMF NVRAM:

```bash
# 1. Generate PK, KEK, db in /var/lib/sbctl
sudo sbctl create-keys

# 2. Rebuild so Lanzaboote signs all boot generations with /var/lib/sbctl
sudo nixos-rebuild switch --flake .#anduril-lab
sudo sbctl verify

# 3. Enroll keys into UEFI firmware (include --microsoft only if OVMF option ROMs need it)
sudo sbctl enroll-keys --microsoft
```

Reboot the VM so UEFI transitions into `SecureBoot: enabled` mode. Once booted with Secure Boot active, enroll the TPM2 chip into a LUKS2 keyslot bound to **PCR 0** (UEFI firmware) and **PCR 7** (Secure Boot state):

```bash
# Verify Secure Boot is active before sealing to PCR 7
bootctl status | grep -E "Secure Boot|TPM2"

# Enroll TPM2 against PCR 0 and PCR 7
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 /dev/disk/by-partlabel/disk-main-luks
```

Reboot a second time: `systemd-cryptsetup` in Stage 1 unseals the volume automatically via TPM2 without prompting for a passphrase.

### Case 4: Verification — test unsigned kernel rejection and generate a CycloneDX SBOM

1. **Prove Secure Boot rejects unsigned images:** Copy a signed UKI on the ESP to a test entry and strip its PE authenticodes signature with `sbctl remove-file` / `objcopy`:

```bash
sudo cp /boot/EFI/Linux/nixos-generation-1-*.efi /boot/EFI/Linux/unsigned-test.efi
sudo objcopy --remove-section .certs /boot/EFI/Linux/unsigned-test.efi
```

Select `unsigned-test.efi` in the boot menu: OVMF immediately halts execution with `Security Violation` and refuses to boot the unsigned kernel.

2. **Generate a CycloneDX SBOM for the exact system derivation:**

```bash
nix run github:tiiuae/sbomnix#sbomnix -- \
  .#nixosConfigurations.anduril-lab.config.system.build.toplevel \
  --cdx sbom-anduril-lab.cdx.json

jq '{bomFormat, specVersion, componentsCount: (.components | length)}' sbom-anduril-lab.cdx.json
```

Output (shape):

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.5",
  "componentsCount": 642
}
```

## The trap

The trap is **running `systemd-cryptenroll --tpm2-pcrs=0+7` *before* enrolling Secure Boot keys and rebooting into Secure Boot Active mode**. PCR 7 hashes whether Secure Boot is currently enabled and which `PK`/`KEK`/`db` certificates were used. If you enroll the TPM while Secure Boot is still in Setup/Disabled mode, enabling Secure Boot on the next reboot changes PCR 7 and forces a fallback to the recovery passphrase.

The second trap is **binding to PCR 4 or PCR 8/9 without signed PCR policies (`systemd-measure`)**. PCR 4 changes on every kernel update (`nixos-rebuild switch` with a new kernel). Binding raw PCR values to `0+7` keeps the disk tied to firmware + your Secure Boot `db` signing key across normal NixOS generation updates.

## The boring rule

- Always keep an offline LUKS recovery passphrase slot before enrolling TPM2.
- Sequence strictly: `sbctl create-keys` → `nixos-rebuild switch` (Lanzaboote signs UKIs) → `sbctl enroll-keys` → reboot with SB enabled → `systemd-cryptenroll --tpm2-pcrs=0+7`.
- Use `boot.initrd.systemd.enable = true` so `crypttabExtraOpts = [ "tpm2-device=auto" ]` works natively in Stage 1.
- Blacklist unused external DMA/storage modules (`usb-storage`, `thunderbolt`, `firewire-core`) on unattended hardware.
- Generate and archive `sbom-*.cdx.json` in CI alongside every production image build.

## Try this

1. Run `sbctl status` and `sbctl verify` inside the VM to inspect every signed `.efi` binary on `/boot`.
2. Run `systemd-cryptenroll /dev/disk/by-partlabel/disk-main-luks` (with no flags) to list active LUKS2 keyslots (`password` vs `tpm2`).
3. Verify `modprobe usb-storage` fails when `usb-storage` is listed in `boot.blacklistedKernelModules`.
4. Query `sbom-anduril-lab.cdx.json` with `jq '.components[] | select(.name == "systemd")'` to inspect the exact CPE and PURL emitted by `sbomnix`.
