# Installing NixOS on a small VPS: a field report

**Goal:** Learn which install routes work for a low-memory VPS, which fail and why, and what to expect in the first weeks of a Nix newcomer's experience.

> [!NOTE]
> Source: Will Richardson, [Inevitably, NixOS](https://willhbr.net/2026/10/11/inevitably-nixos/) (11 October 2026), a first-person report from an Ubuntu and Alpine user. Commands are quoted from the post as the author used them. Our commentary is marked as advice.

## The setting

| Item | Detail |
|------|--------|
| Author's background | Ubuntu since about 2008, development over SSH to a server, a home server moved to Alpine in 2025 |
| Target | A small VPS (about 1 GB of memory) and later a home server |
| Motivation | **Declarative configuration**: a new machine uses the same config, and the config is the source of truth |
| Problem with the old way | Fixing a server over SSH with a one-off command (for example `ufw enable`) leaves that machine different from the others, so servers drift |

The server setup the author wanted: Tailscale, a separate no-sudo user for Podman, a firewall, node exporter and the right SSH public keys. Compare with [Firewall discipline](../05-services-and-security/05-firewall-discipline.md) and [Users and access](../03-nixos-host/04-users-and-access.md).

## Step 1: practise on a spare laptop

The author installed on an old ThinkPad first, so a configuration error could not lock out SSH. The graphical installer needed only a few clicks, offered several desktop environments, and set up wireless drivers correctly. Daily work was editing `/etc/nixos/configuration.nix` and running `sudo nixos-rebuild switch`. See [Rebuild and generations](../03-nixos-host/03-rebuild-and-generations.md).

> [!TIP]
> Our advice: practise on hardware or a VM where a mistake costs nothing, then repeat on the server.

## Step 2: four server install routes

| Route | Result | Why |
|-------|--------|-----|
| **Manual install from the wiki** (`fdisk`, `nixos-generate-config`, edit, `nixos-install`) | Abandoned | Partitioning by hand, including the DOS versus UEFI decision, felt unlike the guided installers of Ubuntu and Alpine |
| **nixos-up** script (Samuel Ainsworth) | Failed | Partitions, creates a user and calls `nixos-install` with a default config, but `nixos-install` evaluates the whole configuration on the small machine and ran out of memory |
| **nixos-infect** (convert an existing Ubuntu VM in place) | Worked, not trusted | The script had to be edited to avoid `/tmp` (a tmpfs, too small) and installed an older NixOS release, so the author did not want it in a server setup process |
| **Build locally, copy the closure** | Failed on 1 GB | See below |
| **nixos-anywhere with disko** | Worked | A fully declarative install including partitioning, then the same config for updates |

### Build locally, copy over

The idea is to evaluate and build on a powerful machine, so nothing is evaluated on the VPS:

```bash
nixos-rebuild build -I nixos-config=./configuration.nix
nix-store --export $(nix-store -qR result) | ssh root@target "nix-store --import --store /mnt"
ssh root@target "nixos-install --system $(readlink result)"
```

What went wrong:

- `nix-store --import` stalled, and the process had been **OOM killed**. It uses a lot of memory to validate input even when piped. Exporting to an archive first did not help.
- The source blog advised creating a **swap file** (its machine had 512 MB); the author skipped that, assuming 1 GB was enough.
- `nix-copy-closure` cannot override the destination store path, which matters when the target booted from an ISO and the new system is mounted at `/mnt/nix/store`.

> [!WARNING]
> Our advice: on a VPS with 1 GB or less, create swap before importing a large closure, or use a route that copies with `nix copy --to ssh-ng://...?root=/mnt`, which accepts an alternative root. Test the method on a throwaway machine first.

### The route that worked: nixos-anywhere

nixos-anywhere takes a flake that includes a disko disk layout and installs over SSH. The author booted a NixOS image on the VPS (kexec from another distribution is also possible), added a public key for root, set the right disk in the disko config and ran:

```bash
nix run github:nix-community/nixos-anywhere -- \
  --generate-hardware-config nixos-generate-config ./hardware-configuration.nix \
  --flake .#<flake name> \
  --target-host root@<ip address>
```

`--generate-hardware-config` writes `hardware-configuration.nix` from the real target and saves it locally; the tool then performs the `nixos-install` steps. The same config is used for later updates. This is the path the book teaches: see [disko](../07-ops-and-fleet/01-disko.md) and the install chapter ([Install](../03-nixos-host/01-install.md)).

The author estimated about $0.10 of cloud time for a few hours of tinkering, a cheap way to rehearse.

## Verdict: what surprised a newcomer

| Observation | What it means for you |
|-------------|-----------------------|
| Editing a file then `nixos-rebuild switch` is satisfying | The workflow is the main benefit |
| **Changing a user's UID did not apply** | NixOS would have to delete and recreate the account, and it silently did nothing. Plan UID changes as a manual migration |
| **Removing zsh left a user with zsh as login shell but no binary** | Set `users.users.<name>.shell` explicitly (for example `pkgs.bashInteractive`) whenever you remove a shell |
| Error messages feel like "undefined is not a function" | Errors are correct for the evaluator but not for the user; read the trace from the bottom for your file's line, or use `--show-trace` |
| CLI sits between old commands (`nixos-rebuild`, `nix-copy-closure`) and the unified `nix` CLI | Pick the one family per task and write it into your runbook |
| Everyone uses flakes, documentation pretends they do not exist | Flakes are still experimental. This book treats them as the working layout: [Flake host](../03-nixos-host/02-flake-host.md) |
| Starting from "install NixOS" rather than learning Nix first | Workable, but plan to read the language reference once you start sharing config between hosts |

The author's next task: structure the configurations so Podman, SSH and the firewall are shared and only hostname, mounts and partitions are per host. That is the pattern in [Shared modules](../07-ops-and-fleet/05-shared-modules.md).

## Checklist for your own small VPS

- [ ] Rehearsed on a laptop or throwaway VM
- [ ] Disk layout in disko, not typed into `fdisk`
- [ ] Install via nixos-anywhere with `--generate-hardware-config`
- [ ] Swap or a remote builder if the target has 1 GB of memory or less
- [ ] Login shells and UIDs set explicitly
- [ ] Before the data-holding home server moves: backups, plus a diff of the new config ([Diffing configurations](../07-ops-and-fleet/15c-nix-diff-between-revisions.md))
