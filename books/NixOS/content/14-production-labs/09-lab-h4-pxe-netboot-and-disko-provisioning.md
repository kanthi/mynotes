---
title: 'Lab H4: Zero-Touch PXE Netboot and Disko Bare-Metal Provisioning'
---

# Lab H4: Zero-Touch PXE Netboot and Disko Bare-Metal Provisioning

Clicking through an ISO installer across twenty bare-metal servers or freshly created VMs does not scale. The boring default for zero-touch bare-metal provisioning is: **build a custom NixOS `netbootRamdisk` and iPXE script from your flake, serve DHCP/TFTP/HTTP from a gateway NixOS node running `dnsmasq` and `nginx`, and run an unattended `disko-install` systemd service on boot that formats the target disk and reboots into the installed OS**.

## Mental model

A NixOS configuration importing `netboot-minimal.nix` produces three artifacts in a single derivation (`config.system.build.netboot`):

1. `bzImage` — the Linux kernel
2. `initrd` — Stage-1 ramdisk containing the squashfs root store (`netbootRamdisk`)
3. `netboot.ipxe` — the iPXE boot script passing `init=/nix/store/.../init`

When a blank VM or bare-metal server powers on with network boot enabled, `dnsmasq` hands it an IP and chainloads `netboot.ipxe`. The node pulls `bzImage` and `initrd` over HTTP, boots straight into RAM, runs a one-shot `systemd` unit that executes `disko-install` against `/dev/vda`, and powers down or reboots into the freshly formatted disk.

```text
Blank Target VM (no disk format, PXE first)
    │
    ├── 1. DHCP + TFTP chainload  ──►  pxe-gw (10.100.60.1 · dnsmasq)
    ├── 2. HTTP GET bzImage + initrd ─► pxe-gw (http://10.100.60.1/netboot/)
    │
    ▼
Target boots NixOS Installer in RAM (squashfs store)
    │
    ├── 3. systemd unit: auto-provision.service
    │        ├── disko-install --flake .#target-node --disk main /dev/vda
    │        └── efibootmgr / reboot into local disk
    ▼
Target reboots into installed NixOS 26.05 on /dev/vda (zero prompts)
```

## Worked examples

### Case 1: Define the target node's Disko layout and OS configuration

First, declare the final NixOS host (`target-01`) that will be installed onto `/dev/vda`.

Save as `hosts/target-01.nix`:

```nix
# hosts/target-01.nix
{ ... }:
{
  networking.hostName = "target-01";
  system.stateVersion = "26.05";

  disko.devices.disk.main = {
    type = "disk";
    device = "/dev/vda";
    content = {
      type = "gpt";
      partitions = {
        ESP = {
          size = "512M";
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
            type = "filesystem";
            format = "ext4";
            mountpoint = "/";
          };
        };
      };
    };
  };

  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;

  services.openssh = {
    enable = true;
    settings.PasswordAuthentication = false;
  };
  users.users.root.openssh.authorizedKeys.keys = [
    "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... ops@desk"
  ];
}
```

### Case 2: Build the custom PXE netboot installer image with embedded closure

The netboot image embeds the **entire closure** of `nixosConfigurations.target-01.config.system.build.toplevel` inside its squashfs store so the unattended install needs zero external internet access. To prevent a re-provision loop if the VM leaves PXE first in its boot order, the installer checks a sentinel label before touching `/dev/vda`.

Save as `hosts/netboot-installer.nix`:

```nix
# hosts/netboot-installer.nix
{ config, lib, pkgs, modulesPath, targetSystem, flakeSelf, ... }:
{
  imports = [
    "${modulesPath}/installer/netboot/netboot-minimal.nix"
  ];

  system.stateVersion = "26.05";

  # Embed the target system's full store closure inside the netboot squashfs
  system.extraDependencies = [
    targetSystem.config.system.build.toplevel
    targetSystem.config.system.build.diskoScript
  ];

  systemd.services.auto-provision = {
    description = "Zero-Touch Disko Bare-Metal Provisioner";
    wantedBy = [ "multi-user.target" ];
    after = [ "network-online.target" ];
    wants = [ "network-online.target" ];
    path = [
      pkgs.disko
      pkgs.nixos-install-tools
      pkgs.util-linux
      pkgs.e2fsprogs
      pkgs.dosfstools
      pkgs.git
    ];
    serviceConfig = {
      Type = "oneshot";
      StandardOutput = "journal+console";
      StandardError = "journal+console";
    };
    script = ''
      set -euo pipefail
      if blkid | grep -q 'PARTLABEL="disk-main-root"'; then
        echo "Disk already provisioned (disk-main-root exists); skipping install."
        exit 0
      fi

      echo "Running Disko partition & format on /dev/vda..."
      ${targetSystem.config.system.build.diskoScript}

      echo "Installing target-01 closure from local squashfs store..."
      nixos-install \
        --root /mnt \
        --system ${targetSystem.config.system.build.toplevel} \
        --no-root-passwd \
        --no-channel-copy

      sync
      echo "Provisioning complete. Powering off / rebooting..."
      systemctl reboot
    '';
  };
}
```

Notice how `${targetSystem.config.system.build.diskoScript}` and `nixos-install --system ${targetSystem.config.system.build.toplevel}` install directly from the pre-populated squashfs store in RAM—finish time is under 45 seconds.

### Case 3: Wire the PXE Gateway VM (`pxe-gw`) with `dnsmasq` and HTTP netboot assets

In `flake.nix`, combine `bzImage`, `initrd` (`netbootRamdisk`), and `netbootIpxeScript` using `pkgs.symlinkJoin` and serve them from `pxe-gw`:

```nix
# fragment in flake.nix outputs
let
  system = "x86_64-linux";
  pkgs = nixpkgs.legacyPackages.${system};

  target01 = nixpkgs.lib.nixosSystem {
    inherit system;
    modules = [ disko.nixosModules.disko ./hosts/target-01.nix ];
  };

  netbootNode = nixpkgs.lib.nixosSystem {
    inherit system;
    specialArgs = { targetSystem = target01; flakeSelf = self; };
    modules = [ ./hosts/netboot-installer.nix ];
  };

  netbootBundle = pkgs.symlinkJoin {
    name = "netboot-bundle";
    paths = with netbootNode.config.system.build; [
      netbootRamdisk
      kernel
      netbootIpxeScript
    ];
  };
in
{
  packages.${system}.netboot-bundle = netbootBundle;
}
```

Configure `hosts/pxe-gw.nix` to serve DHCP/TFTP via `services.dnsmasq` and fast HTTP transfers for the large `initrd` via `services.nginx`:

```nix
# hosts/pxe-gw.nix
{ pkgs, netbootBundle, ... }:
{
  networking.hostName = "pxe-gw";
  system.stateVersion = "26.05";

  networking.interfaces.eth1.ipv4.addresses = [
    { address = "10.100.60.1"; prefixLength = 24; }
  ];

  services.dnsmasq = {
    enable = true;
    settings = {
      interface = "eth1";
      bind-interfaces = true;
      dhcp-range = [ "10.100.60.50,10.100.60.150,12h" ];
      enable-tftp = true;
      tftp-root = "${pkgs.ipxe}";
      # Chainload iPXE over HTTP to avoid slow UDP TFTP for the 800MB initrd
      dhcp-match = [ "set:ipxe,175" ];
      dhcp-boot = [
        "tag:!ipxe,ipxe.efi"
        "tag:ipxe,http://10.100.60.1/netboot/netboot.ipxe"
      ];
    };
  };

  services.nginx = {
    enable = true;
    virtualHosts."10.100.60.1".locations."/netboot/" = {
      alias = "${netbootBundle}/";
    };
  };

  networking.firewall.interfaces.eth1 = {
    allowedTCPPorts = [ 80 ];
    allowedUDPPorts = [ 53 67 69 ];
  };
}
```

### Case 4: Verification — boot a blank VM over PXE

Create a blank KVM guest with an unformatted 20 GB `/dev/vda` disk attached to the `10.100.60.0/24` bridge and boot it:

```bash
qemu-img create -f qcow2 /tmp/target-01.qcow2 20G
# Boot VM on bridge attached to pxe-gw eth1 with serial console (-nographic)
```

Observe the serial log:
1. iPXE receives `10.100.60.51` from `dnsmasq` and fetches `http://10.100.60.1/netboot/netboot.ipxe`.
2. `bzImage` and `initrd` stream over HTTP in 3 seconds and boot the NixOS netboot ramdisk.
3. `auto-provision.service` formats `/dev/vda` via Disko, installs `target-01`'s closure with `nixos-install --no-channel-copy`, and reboots.
4. On the second boot, the node boots `target-01` from `/dev/vda` and accepts SSH on `ops@desk`.

## The trap

The trap is **serving the 800 MB `initrd` squashfs over raw TFTP instead of chainloading iPXE HTTP**. Standard UDP TFTP uses tiny block sizes and frequently times out or takes 15+ minutes on virtual and physical switches. Always serve `ipxe.efi` via TFTP and fetch `netboot.ipxe`, `bzImage`, and `initrd` over HTTP (`nginx`).

The second trap is **an infinite wipe loop when the BIOS boot order keeps PXE above the local disk**. Without the `blkid | grep -q 'PARTLABEL="disk-main-root"'` guard in `auto-provision.service` (or a MAC-specific iPXE handoff on `pxe-gw`), every reboot re-triggers `diskoScript` and wipes the machine again.

## The boring rule

- Embed `targetSystem.config.system.build.toplevel` and `diskoScript` in `system.extraDependencies` of the netboot image so installs never hit the internet.
- Chainload iPXE to fetch `bzImage` and `initrd` over HTTP, never raw TFTP.
- Guard `auto-provision.service` against already-formatted disks (`PARTLABEL="disk-main-root"`).
- Always pass `--no-root-passwd --no-channel-copy` to `nixos-install` inside automated scripts.

## Try this

1. Build `.#packages.x86_64-linux.netboot-bundle` and inspect `cat result/netboot.ipxe` to see the exact kernel parameters (`init=...`, `initrd=initrd`).
2. Test the netboot image directly in QEMU without `dnsmasq` first using `qemu-system-x86_64 -kernel result/bzImage -initrd result/initrd -append "$(grep -o 'init=.*' result/netboot.ipxe)"`.
3. Add a post-install webhook (`curl -X POST http://10.100.60.1/done`) so the gateway switches the host's MAC address to local-disk boot after `nixos-install` succeeds.
