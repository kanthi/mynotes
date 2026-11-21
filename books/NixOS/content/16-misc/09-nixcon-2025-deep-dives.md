# NixCon 2025 deep dives

**Goal:** Go beyond the talk abstracts for the eight NixCon 2025 topics with the most practical value, using the projects' own documentation and public write-ups.

> [!WARNING]
> The snippets below are **sketches** assembled from project documentation. Option names change between releases, so run `nix flake check` and read the current module source or manual before copying. Projects marked experimental can break without notice.

Overview of the whole conference: [NixCon 2025 field guide](05-nixcon-2025-field-guide.md).

## 1. Hydra Queue Runner in Rust

| Aspect | Old queue runner | New queue runner |
|--------|------------------|------------------|
| Builder transport | SSH, reading and writing the Nix daemon directly | gRPC with **bidirectional streams** |
| Builder reachability | Server connects out to builders | Builders connect in, so they need no public IP |
| Scheduling input | Static job limits | Heartbeats with CPU, memory and load metrics |
| Visibility | Limited | Tracing and a metrics/status HTTP interface |
| Queues | Shared | One queue per platform |

Why it matters to you: if you run Hydra or a large remote-builder pool, the old SSH model caps how many builders you can attach. Until the rewrite is in your Hydra version, keep builder counts moderate and see [Remote builders](../10-cicd/07-remote-builders.md).

## 2. A/B OTA updates with systemd

NixOS here acts as an **image generator**, not as an on-device `nixos-rebuild`:

1. `image.repart` builds a GPT disk image (ESP plus root partitions) with `systemd-repart` at build time.
2. On first boot, `systemd-repart` can grow the layout to fill the disk.
3. `systemd.sysupdate` polls a server, verifies a new versioned image and writes it to the **inactive** slot.
4. Boot counting in `systemd-boot` falls back to the previous slot if the new one fails.

```nix
{
  image.repart = {
    name = "appliance";
    partitions = {
      "10-esp".repartConfig = {
        Type = "esp";
        Format = "vfat";
        SizeMinBytes = "256M";
      };
      "20-root-a".repartConfig = {
        Type = "root";
        Format = "erofs";
        Label = "root-a";
        Minimize = "best";
      };
    };
  };

  systemd.sysupdate = {
    enable = true;
    transfers."10-root" = {
      Source = {
        Type = "url-file";
        Path = "https://updates.example.invalid/releases/";
        MatchPattern = "root_@v.raw.zst";
      };
      Target = {
        Type = "partition";
        Path = "auto";
        MatchPattern = "root-@v";
      };
    };
  };
}
```

On the device, `updatectl list` and `updatectl update` show and apply versions (these need a systemd version that ships `updatectl`). Related lab: [Lab P3](../14-production-labs/12-lab-p3-cross-compiled-edge-ota-and-sysupdate.md).

## 3. NixOps4

- The core only manages a **graph of resources** and their lifecycle (create, update, delete).
- Everything provider-specific lives in **resource providers**: external executables that speak a structured protocol over stdio, packaged with Nix.
- Configuration is written with the **Nix module system**, so resources and NixOS machine configurations share one model.
- State handling is configurable **per resource**, unlike one global state file.
- The talk's open question, "will it replace Terraform/OpenTofu?", is answered cautiously by the design: it can wrap rather than replace them. Until it matures, keep the boundary described in [Terraform boundary](../07-ops-and-fleet/07-terraform-boundary.md).

## 4. The dendritic pattern

Rule: **every Nix file except the entry point is a module**, loaded automatically.

```nix
# flake.nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";
    flake-parts.url = "github:hercules-ci/flake-parts";
    import-tree.url = "github:vic/import-tree";
  };

  outputs = inputs:
    inputs.flake-parts.lib.mkFlake { inherit inputs; }
      (inputs.import-tree ./modules);
}
```

```nix
# modules/ssh.nix : one feature, defined once, used by any host that includes it
{ ... }:
{
  flake.modules.nixos.ssh = {
    services.openssh.enable = true;
    services.openssh.settings.PasswordAuthentication = false;
  };
}
```

| Gain | Cost |
|------|------|
| No import lists; move files freely | Deeper stack traces |
| One file can touch NixOS, Home Manager and flake outputs | New readers must learn flake-parts first |

Adopt it on a branch first, as the workshop suggests.

## 5. jail.nix

Wrap a package in bubblewrap using combinators rather than raw flags.

```nix
let
  jail = inputs.jail-nix.lib.init pkgs;
in
jail "jailed-tool" pkgs.some-tool (with jail.combinators; [
  network
  mount-cwd
  (add-pkg-deps [ pkgs.git ])
])
```

Common combinators reported in its docs: `network`, `mount-cwd`, `readonly`/`readwrite` paths, `add-pkg-deps`, plus hooks to run code before launch or on cleanup. Treat the result as defence in depth, not a full security boundary (same kernel).

## 6. finix

finix swaps systemd for **finit** as PID 1 while keeping flakes, `nixos-rebuild`, `nixos-install` and much NixOS module scaffolding. Reported companions: seatd, mdevd or libudev-zero, and Wayland compositors such as niri. Services are Nix attributes mapped to finit runlevels, conditions and commands:

```nix
{
  finit.services.example = {
    description = "example daemon";
    runlevel = "2345";
    command = "${pkgs.example}/bin/example --foreground";
  };
}
```

Use it to learn what NixOS ties to systemd; do not treat it as a drop-in server OS.

## 7. nixos-compose

The garnix tool turns the `nixosConfigurations` of a flake into local VMs.

```bash
nix flake show                 # list configurations
nixos-compose up server        # build and boot one
nixos-compose ssh server       # preconfigured SSH access
nixos-compose ip server        # VM address
nixos-compose status           # running VMs
nixos-compose down             # stop everything
```

It handles SSH keys, a virtual switch and `/etc/hosts` so VMs reach each other by name. Compare with the NixOS test driver in [NixOS tests](../07-ops-and-fleet/08-nixos-tests.md): tests are scripted and disposable, compose is for interactive development.

## 8. Long-term support and the CRA

- Cyberus Technology's LTS project (first called Ctrl-OS, later reported renamed **Cyberus Linux**) offers **five years** of support on selected NixOS releases, with CVE tracking, SBOM generation and SLAs.
- The design principle is a **small, test-gated backport stream** on a fixed release so that certified products avoid frequent re-certification.
- DIY version of the same idea: pin one release branch, backport only security fixes, gate them with NixOS tests, and publish an SBOM per release. See [Vulnerability inventory](../05-services-and-security/12-vuln-inventory-across-profiles.md).

## Sources

- [Talk list](https://talks.nixcon.org/nixcon-2025/schedule/)
- Hydra: [NixOS/hydra](https://github.com/NixOS/hydra)
- sysupdate: [Nixcademy write-ups](https://nixcademy.com/) and the NixOS `image.repart` and `systemd.sysupdate` module options
- NixOps4: [nixops.dev](https://nixops.dev)
- Dendritic: [mightyiam/dendritic](https://github.com/mightyiam/dendritic)
- jail.nix: [sr.ht/~alexdavid/jail.nix](https://git.sr.ht/~alexdavid/jail.nix)
- finix: [finix-community/finix](https://github.com/finix-community/finix)
- nixos-compose: [garnix-io/nixos-compose](https://github.com/garnix-io/nixos-compose)
- Cyberus Technology: [cyberus-technology.de](https://cyberus-technology.de)
