# Declarative Testing and Fleet Deployment

A custom Linux distribution is only as reliable as its verification and rollout loop. In traditional distributions, integration testing requires provisioning virtual machines with external scripts, and fleet upgrades risk half-applied package states when an upgrade is interrupted. NixOS solves both problems inside the same functional model: **declarative multi-VM integration tests** run as pure Nix derivations, and **atomic generation switches** roll out verified store closures across machines with instant rollback.

## Multi-VM Integration Tests with `pkgs.testers.runNixOSTest`

Because every NixOS machine is a pure expression, Nix can launch a virtual network of QEMU VMs inside a sandboxed build derivation and drive them with a Python test script. The VMs share the host `/nix/store` read-only over 9p/virtiofs, so booting two full Linux nodes takes seconds rather than minutes.

```nix
# tests/service-check.nix
{ pkgs }:

pkgs.testers.runNixOSTest {
  name = "byol-service-healthcheck";

  nodes = {
    app = { pkgs, ... }: {
      services.nginx = {
        enable = true;
        virtualHosts."byol.local".locations."/health".return = "200 'byol-ok'";
      };
    };

    client = { pkgs, ... }: {
      environment.systemPackages = [ pkgs.python3 ];
    };
  };

  testScript = ''
    start_all()
    app.wait_for_unit("nginx.service")
    app.wait_for_open_port(80)
    app.succeed("nginx -t")
  '';
}
```

Expose this test under `checks.x86_64-linux.service-healthcheck` in your `flake.nix`. Running `nix flake check` compiles the derivations, boots both QEMU nodes, runs the Python assertions, and caches the passing test result in `/nix/store`.

> [!TIP]
> Launch any NixOS VM test interactively for debugging with `nix run .#checks.x86_64-linux.service-healthcheck.driverInteractive`. This opens a Python REPL where you can step through node assertions and inspect journal logs live.

## Binary Caches: Build Once, Substitute Everywhere

Compiling custom kernels, patched packages, or system daemons on every target machine wastes CPU cycles and slows rollouts. A **Nix binary cache** stores signed `.narinfo` metadata and compressed `.nar` archives indexed by store path hash.

- **Build servers produce closures:** Your CI builder evaluates and compiles `nixosConfigurations.<host>.config.system.build.toplevel`, signs the resulting store paths, and uploads them to a self-hosted binary cache (such as Attic, Harmonia, or object storage).
- **Target hosts stay lean:** Production machines configure `nix.settings.extra-substituters` so they fetch prebuilt store paths directly instead of invoking a local compiler.

## Fleet Deployment Topologies Compared

Once your build pipeline has tested and cached a new system closure, you choose how target machines converge to that closure:

| Tool / Pattern | Push vs. Pull | Rollback Safety | Best Fit |
| :--- | :--- | :--- | :--- |
| `nixos-rebuild switch` | Push (direct trigger) | Manual reboot or `switch-to-configuration` | Small labs, workstations, rapid iteration |
| `deploy-rs` | Push (multi-profile) | Automatic rollback if activation fails confirmation | Critical servers and remote appliances |
| `colmena` | Push (parallel fleet) | Tag-based staged rollouts (`--on @canary`) | Medium-to-large fleets grouped by role |
| `system.autoUpgrade` / `comin` | Pull (GitOps polling) | Atomic generation switch + bootloader fallback | Edge nodes and intermittent-connectivity fleets |

### Atomic Activation Mechanics

Regardless of whether you push or pull, activating a new generation follows the same three-phase atomic sequence:

1. **Closure transfer (`nix copy`):** Every required `/nix/store/<hash>-*` path is copied or substituted before touching the running system. If the transfer aborts at 99%, the running system remains untouched.
2. **Profile symlink swap (`nix-env --set`):** `/nix/var/nix/profiles/system` atomically advances to a new numbered generation (`system-43-link`).
3. **Activation (`switch-to-configuration switch`):** Updates bootloader entries, reconciles `/etc` symlinks, reloads changed systemd units, and restarts only services whose unit definitions or store paths changed.

> [!IMPORTANT]
> For remote fleets, pair `nixos-rebuild test` (activates without writing a permanent boot entry) or `deploy-rs` automatic rollback with a hardware watchdog (`systemd.settings.Manager.RuntimeWatchdogSec = "30s"`). If a bad configuration disrupts connectivity, the machine automatically reverts to the previous generation.

## Hands-On Lab: Run a Sandboxed NixOS Integration Test

This lab wires `byol-service-healthcheck` into `/tmp/byol-nix-lab/flake.nix` and verifies an end-to-end service lifecycle inside ephemeral QEMU VMs.

### Step 1: Add the Test Check to `flake.nix`

Save `tests/service-check.nix` inside `/tmp/byol-nix-lab/tests/service-check.nix`, then expose it in `flake.nix`:

```nix
# Add inside outputs of /tmp/byol-nix-lab/flake.nix
checks.x86_64-linux.service-healthcheck =
  import ./tests/service-check.nix {
    pkgs = nixpkgs.legacyPackages.x86_64-linux;
  };
```

### Step 2: Execute the Test Suite

```bash
cd /tmp/byol-nix-lab
nix build .#checks.x86_64-linux.service-healthcheck --print-build-logs
```

Watch the build log as QEMU boots `app` and `client`, waits for `nginx.service` to bind port 80, and validates the configuration.

### Step 3: Verify Failure Detection

Intentionally change `app.wait_for_open_port(80)` to `app.wait_for_open_port(8080)` in `tests/service-check.nix` and re-run `nix build .#checks.x86_64-linux.service-healthcheck`. Confirm that the check times out and prevents the broken configuration from passing CI. Restore port `80` and verify the check passes again.

## Summary: Imperative BYOL vs. Declarative NixOS

Across Chapters 1 through 13, you built every layer of a Linux distribution imperatively—understanding compilers, kernels, initramfs, init systems, packages, and ISOs from bare metal up. Chapter 14 shows how NixOS takes those exact same Linux primitives and wraps them in content-addressed derivations, pure module evaluation, and atomic generations. Knowing both worlds gives you the mechanical intuition to debug any Linux failure and the architectural leverage to ship reproducible operating systems at scale.
