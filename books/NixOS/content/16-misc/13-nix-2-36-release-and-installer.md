# Nix 2.36 and the official installer

**Goal:** Know what Nix 2.36.0 (announced 19 November 2026) changes for daily work, and which installer to use for a new machine.

> [!NOTE]
> Sources: the [Nix 2.36.0 release announcement](https://discourse.nixos.org/t/nix-2-36-0-released/81345) on NixOS Discourse (primary), and the [FOSSLinux write-up](https://www.fosslinux.com/164375/nix-2-36-new-installer-existing-installs.htm) (secondary commentary). [Current Nix and NixOS](01-current-nix-and-nixos.md) was checked on 13 November 2026 and still names 2.35 as current. Treat this chapter as the update to that pin.

## What happened

Nix 2.36 came about four months after 2.35. The Nix team describes it as many bugfixes and fixed performance blunders, plus substantial fuzzing work on core `libstore` components. The headline is that the **Rust-based `NixOS/nix-installer`** is now adopted **alongside** the existing scripted installer. It builds on code created by Determinate Systems, which licensed it liberally; an installer working group carried the project.

## Which installer for which machine

```bash
curl -sSfL https://channels.nixos.org/nix-latest/nix-installer | sh -s -- install
```

| Your machine | Use | Why |
|--------------|-----|-----|
| Fresh Linux box with systemd, multi-user | New Rust installer | It is the path 2.36 ships and documents |
| Apple Silicon Mac | New Rust installer | `aarch64-darwin` is a build target |
| Alpine or busybox, single-user | **Scripted installer** | The new installer is multi-user only; the script gained busybox single-user support in 2.36 |
| Existing Nix install | Neither | Nothing to reinstall; upgrade with `nix upgrade-nix` or your package manager |

Constraints from the release notes: binaries are built for `x86_64-linux`, `aarch64-linux` and `aarch64-darwin`, and **only multi-user installs are supported**. FOSSLinux notes the repository still labels the installer **Beta** and also documents WSL2, SteamOS and a root-only container mode. Treat it as a good default for new multi-user machines, not as a reason to rebuild working ones.

> [!IMPORTANT]
> The Fedora install chapter ([Fedora 44 install](../02-nix-on-linux/02-fedora-44-install.md)) uses the upstream installer of its day. Existing installs are unaffected. On the next fresh Fedora machine, compare both paths and record which you used.

## Changes that affect existing installs

| Change | Effect |
|--------|--------|
| **Daemon worker spawning rework** | Workers are started by executing the daemon binary in `--stdio` mode instead of a bare `fork()`, which fixes long-standing fork-safety issues, especially on macOS |
| `nix.conf` reloads | The daemon no longer needs a restart for `/etc/nix/nix.conf` changes to reach new workers. `allowed-users` and `trusted-users` are the exceptions |
| Worker process name | `argv[0]` is now `nix-daemon.worker-<peer pid>`; previously the peer PID was in `argv[1]`. **Update monitoring that matches process names** |
| Stabilised flags | `--force-trusted`, `--force-untrusted`, `--default-trust` and `--process-ops` are no longer experimental; the `daemon-trust-override` feature is stabilised |
| Build cgroups | With `use-cgroups` the daemon enables the `cpu`, `memory`, `io` and `pids` controllers, so builds expose `memory.peak` and `io.stat`. It needs the controllers delegated to the daemon's cgroup (for example systemd `Delegate=yes`) |
| Cgroup names | Now `nix-build@<drv-hash>-<uid>` (build user) or `nix-build@<drv-hash>-<pid>`, so external tools can find the cgroup of a derivation's build |

## Smaller improvements

| Change | Detail |
|--------|--------|
| Auto-call functions | `nix run`, `nix search` and `nix bundle` with `--file`/`--expr` call a function when every argument has a default or is passed with `--arg`/`--argstr` (like `nix build`) |
| `nix version` | New command; `nix version --json` includes library versions, handy for inventory scripts |
| Plain stdout | `nix profile list`, `nix flake show`, `nix search`, `nix store diff-closures` and others no longer emit ANSI styling when stdout is not a terminal or `NO_COLOR` is set. Set `CLICOLOR_FORCE` to keep it |
| Faster logging | Download progress in the `internal-json` logger is chunked and rate-limited, easing the JSON bottleneck for tools such as nix-output-monitor |
| `optimize` alias | `optimize` works as well as `optimise` |
| Darwin GC roots | Garbage collection finds runtime roots with libproc instead of `lsof` |
| Packaging | Cross-built binaries for `powerpc64-linux` and `powerpc64le-linux` |
| Scripted installer | Sets the correct store mtime |
| `nix develop` (per FOSSLinux) | No longer disables ASLR on Linux in the shell |

```bash
# inventory one-liner that is easier with 2.36
nix version --json
```

## What to do on a NixOS host

1. Do nothing on 26.05 unless you want 2.36: the channel Nix stays what nixpkgs provides. Check with `nix --version`.
2. To try it, pin through `nix.package` only after the attribute exists in your nixpkgs lock (see the pin pattern in [Current Nix and NixOS](01-current-nix-and-nixos.md)).
3. If you use `use-cgroups` with a custom daemon unit, add `Delegate=yes`.
4. Search your monitoring rules for `nix-daemon <pid>` command-line matches.
5. Remember tools that parse Nix output: [deploy-rs had to follow JSON changes in 2.32 and 2.33](../07-ops-and-fleet/03b-deploy-rs-2026-and-app-profiles.md), so run a deploy against a test host before upgrading a fleet's CLI.

## Checklist

- [ ] New machines: installer chosen per the table, choice recorded
- [ ] Existing machines: no reinstall; upgrade path documented
- [ ] Monitoring no longer depends on the worker command line
- [ ] Build cgroup delegation set if you read per-build metrics
