---
title: 'Lab H1: Cross-Arch Remote Builder and Harmonia Binary Cache'
---

# Lab H1: Cross-Arch Remote Builder and Harmonia Binary Cache

Compiling heavy derivations or `aarch64-linux` packages on every laptop wastes hours and battery. The boring default for a single-machine KVM lab is: **one dedicated builder VM (`builder-01`) running `binfmt` emulation and Harmonia (`services.harmonia`), signed with a local Ed25519 cache key so thin client VMs pull 100% pre-built store paths over HTTP**.

## Mental model

This lab separates **who evaluates** (`client-01`), **who compiles** (`builder-01` via SSH `nix-daemon`), and **who serves substitutes** (`builder-01` via Harmonia HTTP on `:5000`). A second clean client (`client-02`) with no build machines configured proves the cache works by pulling the compiled `aarch64-linux` closure without compiling a single line locally.

```text
┌────────────────────┐    1. SSH offload (.drv)    ┌─────────────────────────────────────────┐
│ client-01 (x86_64) ├────────────────────────────►│ builder-01 (x86_64 + binfmt aarch64)    │
│  nix.buildMachines │                             │  • boot.binfmt.emulatedSystems          │
│  --max-jobs 0      │                             │  • nix.sshServe (trusted)               │
└────────────────────┘                             │  • services.harmonia (:5000, signed)    │
                                                   └────────────────────┬────────────────────┘
┌────────────────────┐    2. HTTP NAR substitute                        │
│ client-02 (clean)  │◄─────────────────────────────────────────────────┘
│  substituters only │       zero local compilation
└────────────────────┘
```

| Node | IP (KVM lab net) | Role |
|---|---|---|
| `builder-01` | `10.100.50.10` | Remote builder (`x86_64-linux` + emulated `aarch64-linux`) & Harmonia cache |
| `client-01` | `10.100.50.11` | Submits multi-arch build with `--max-jobs 0` |
| `client-02` | `10.100.50.12` | Clean consumer that trusts Harmonia's public key and pulls pre-built closures |

## Worked examples

### Case 1: Generate the binary cache signing key pair

On your operator host (or inside `builder-01` setup), generate an Ed25519 key pair for Harmonia:

```bash
nix key generate-secret --key-name lab-cache-1 > cache-priv-key.pem
nix key convert-secret-to-public < cache-priv-key.pem > cache-pub-key.pem
cat cache-pub-key.pem
```

Output (shape):

```text
lab-cache-1:6x9bW1zF3pL8qR2vK0mN4jH7tY5cV8xZ1aB3dE6fG9h=
```

Store `cache-priv-key.pem` in SOPS (`sops.secrets."harmonia-sign-key"`) so it lands at `/run/secrets/harmonia-sign-key` on `builder-01`. Commit `cache-pub-key.pem`'s string directly in your shared NixOS module.

### Case 2: Configure `builder-01` with `binfmt` and Harmonia

Save as `hosts/builder-01.nix`:

```nix
# hosts/builder-01.nix
{ config, pkgs, ... }:
{
  networking.hostName = "builder-01";
  system.stateVersion = "26.05";

  # Emulate ARM64 userland via QEMU binfmt_misc for cross-arch derivations
  boot.binfmt.emulatedSystems = [ "aarch64-linux" ];

  # Dedicated SSH builder endpoint
  nix.sshServe = {
    enable = true;
    write = true;
    protocol = "ssh-ng";
    keys = [
      "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... root@client-01"
    ];
  };
  nix.settings.trusted-users = [ "nix-ssh" ];

  # Serve /nix/store over HTTP signed with lab-cache-1
  sops.secrets."harmonia-sign-key" = {
    mode = "0400";
    owner = "harmonia";
  };

  services.harmonia = {
    enable = true;
    signKeyPaths = [ config.sops.secrets."harmonia-sign-key".path ];
    settings.bind = "0.0.0.0:5000";
  };

  networking.firewall.allowedTCPPorts = [ 22 5000 ];
}
```

Rebuild `builder-01` and verify `binfmt` registration and Harmonia's `/nix-cache-info`:

```bash
ls /proc/sys/fs/binfmt_misc/aarch64-linux
curl -sS http://10.100.50.10:5000/nix-cache-info
```

### Case 3: Configure `client-01` to offload builds to `builder-01`

Save as `hosts/client-01.nix`:

```nix
# hosts/client-01.nix
{ ... }:
{
  networking.hostName = "client-01";
  system.stateVersion = "26.05";

  programs.ssh.knownHosts."10.100.50.10".publicKey =
    "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... host@builder-01";

  nix.distributedBuilds = true;
  nix.settings.builders-use-substitutes = true;

  nix.buildMachines = [
    {
      hostName = "10.100.50.10";
      sshUser = "nix-ssh";
      sshKey = "/root/.ssh/id_ed25519_builder";
      protocol = "ssh-ng";
      systems = [ "x86_64-linux" "aarch64-linux" ];
      maxJobs = 4;
      speedFactor = 2;
      supportedFeatures = [ "nixos-test" "benchmark" "big-parallel" "kvm" ];
    }
  ];
}
```

Test the builder connection before running a real build:

```bash
sudo nix store ping --store ssh-ng://nix-ssh@10.100.50.10
```

### Case 4: Configure `client-02` as a pure binary cache consumer

`client-02` has **no** `nix.buildMachines` and **no** `binfmt` emulation. It cannot build `aarch64-linux` derivations locally; it can only pull them from Harmonia on `builder-01`.

Save as `hosts/client-02.nix`:

```nix
# hosts/client-02.nix
{ ... }:
{
  networking.hostName = "client-02";
  system.stateVersion = "26.05";

  nix.settings = {
    substituters = [
      "http://10.100.50.10:5000"
      "https://cache.nixos.org"
    ];
    trusted-public-keys = [
      "lab-cache-1:6x9bW1zF3pL8qR2vK0mN4jH7tY5cV8xZ1aB3dE6fG9h="
      "cache.nixos.org-1:6NCHdD59X431o0gWypbMrAURkbJ16ZPMQFGspcDShjY="
    ];
  };
}
```

### Case 5: Verification — offload compile, then pull 100% cached

Define a custom `aarch64-linux` derivation in your lab flake so `cache.nixos.org` cannot satisfy it:

```nix
# fragment in flake.nix outputs
packages.aarch64-linux.lab-probe =
  nixpkgs.legacyPackages.aarch64-linux.runCommand "lab-h1-aarch64-probe" {} ''
    uname -m > "$out"
  '';
```

1. **On `client-01`**, trigger the build with `--max-jobs 0` (forcing remote offload to `builder-01`):

```bash
nix build .#packages.aarch64-linux.lab-probe --max-jobs 0 -L
cat ./result
```

Output:

```text
aarch64
```

2. **On `client-02`**, pull the exact derivation with `--max-jobs 0` (proving 100% cache substitution from Harmonia without local compilation or remote builders):

```bash
nix build .#packages.aarch64-linux.lab-probe --max-jobs 0 --option substitute true
cat ./result
```

Output shows `copying path '/nix/store/...-lab-h1-aarch64-probe' from 'http://10.100.50.10:5000'...` and prints `aarch64`.

## The trap

The trap is **testing remote builders with an `x86_64-linux` package on `client-01` without passing `--max-jobs 0`**. When a client can build `x86_64-linux` locally, the Nix daemon frequently chooses local compilation unless `--max-jobs 0` is set, fooling you into thinking the remote builder ran.

The second trap is **forgetting `programs.ssh.knownHosts` for root on `client-01`**. The `nix-daemon` runs as `root`, not your interactive user; if `/root/.ssh/known_hosts` does not trust `builder-01`'s host key, `nix-daemon` silently skips the remote builder with `cannot build on 'ssh-ng://...'`.

## The boring rule

- `nix-daemon` connects as `root`: put the builder's host key in `programs.ssh.knownHosts` and test with `sudo nix store ping --store ssh-ng://...`.
- Use `nix.sshServe` (`nix-ssh` user) on `builder-01` rather than handing `root` SSH access to every client.
- Sign Harmonia responses on the fly with `signKeyPaths` backed by `sops-nix`.
- Always verify cache consumption from a clean client with `--max-jobs 0`.

## Try this

1. Run `nix path-info --sigs .#packages.aarch64-linux.lab-probe` on `client-02` and confirm `lab-cache-1:...` is attached to the store path.
2. Temporarily change one character in `trusted-public-keys` on `client-02`, purge the store path (`nix store delete`), and confirm `nix build --max-jobs 0` rejects the unsigned/mismatched NAR from Harmonia.
3. Compare build wall-clock time on `builder-01` using `boot.binfmt.emulatedSystems` versus `pkgsCross.aarch64-multiplatform` cross-compilation.
