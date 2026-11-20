---
title: 'Deployment Lifeline and Rapid Patching: Spare Tunnels, Eternal Terminal, tmux, and btop'
---

# Deployment Lifeline and Rapid Patching: Spare Tunnels, Eternal Terminal, tmux, and btop

Any major OS channel switch or emergency security rollout can restart `sshd`, reload `nftables`, bounce network interfaces, or push a machine into swap thrashing mid-activation. The boring default for remote NixOS deployments is: **never deploy into a host without an already-open spare lifeline session (`Eternal Terminal` or a second SSH tunnel) attached to a remote root `tmux` session where one pane monitors system pressure (`btop`) and a second pane sits pre-authenticated for instant rollback**.

## Mental model

When `switch-to-configuration` activates a new NixOS generation, it reloads firewall rules and restarts changed units. Two kernel and daemon properties determine whether you keep control or get locked out:

1. **Existing TCP sessions survive changes that block new logins.** A broken `sshd_config`, a failed PAM module, or a firewall error that drops `NEW` packets on port 22 will block every new `ssh` connection from your laptop—while an **already-established** session stays alive (because `sshd` forks a child process per connection and stateful firewalls allow `ct state established,related`).
2. **A raw SSH shell dies on network blips; `Eternal Terminal` (`et`) + `tmux` does not.** If `systemd-networkd` or `NetworkManager` bounces an interface during activation, a raw SSH command receives `SIGHUP` or hangs on a dead TCP socket. `Eternal Terminal` (`services.eternal-terminal`) reconnects the session automatically across interface bounces, and `tmux` keeps processes and root shells running on the host regardless of client state.

```text
Operator Workstation
  │
  ├── Session 1 (Deployer):  deploy .#host  /  nixos-rebuild switch
  │
  └── Session 2 (Lifeline — opened BEFORE Session 1 starts):
        et ops@host (TCP 2022)  OR  ssh -S ~/.ssh/lifeline-%r@%h:%p
             │
             ▼
        Remote tmux session ("lifeline", running as root)
        ┌───────────────────────────────────┬───────────────────────────────────┐
        │ Pane 0: Live Telemetry            │ Pane 1: Pre-Armed Rollback Shell  │
        │   btop                            │   # PREV=$(readlink -f            │
        │   • Watch /nix & /boot disk %     │       /run/current-system)        │
        │   • Watch nix-daemon / OOM        │   # Ready to run:                 │
        │   • Press 'q' (or Ctrl-b x)       │   $PREV/bin/switch-to-configuration│
        │     to yield terminal immediately │     switch                        │
        └───────────────────────────────────┴───────────────────────────────────┘
```

| Skill / Tool | Role During a Deployment | Failure Mode It Prevents |
|---|---|---|
| **Spare SSH tunnel / `ControlMaster`** | Pre-authenticated socket opened before activation begins | Broken PAM, `AuthorizedKeys`, or `sshd` restart blocking new connections |
| **Eternal Terminal (`et`)** | Roaming session daemon (`etserver` on TCP `2022`) with native `tmux` scrollback | Interface/VPN bounce hanging standard SSH TCP sockets mid-switch |
| **`tmux` on the target host** | Keeps shells and commands alive detached from the network socket | `SIGHUP` killing `switch-to-configuration` halfway through activation |
| **`btop` (and knowing how to quit/switch away)** | Real-time CPU, memory, disk I/O, and `/boot` + `/nix` capacity monitor | Silent OOM kills, runaway compile jobs, or 100% full `/boot` ESP partitions |
| **Direct `switch-to-configuration` rollback** | Invokes the previous generation's activation binary directly from `/nix/store` | Broken `nix` CLI or broken `nixos-rebuild` preventing standard rollback |

## Worked examples

### Case 1: Declarative lifeline module (`eternal-terminal`, `tmux`, `btop`)

Install the lifeline tools in your base server module so every host in your fleet has `etserver`, `tmux`, and `btop` before you ever need them in an emergency.

Save as `modules/ops-lifeline.nix`:

```nix
# modules/ops-lifeline.nix
{ pkgs, ... }:
{
  environment.systemPackages = [
    pkgs.btop
    pkgs.eternal-terminal
    pkgs.tmux
  ];

  # Persistent roaming terminal server (listens on TCP 2022 by default)
  services.eternal-terminal = {
    enable = true;
    port = 2022;
  };

  # Sensible tmux defaults on every remote host
  programs.tmux = {
    enable = true;
    clock24 = true;
    keyMode = "vi";
    historyLimit = 50000;
    extraConfig = ''
      set -g mouse on
      set -g status-right '#(hostname) | %Y-%m-%d %H:%M:%S'
    '';
  };

  # Open Eternal Terminal port alongside SSH (restrict to VPN/wg0 in production)
  networking.firewall.allowedTCPPorts = [ 22 2022 ];
}
```

### Case 2: The pre-deployment ritual — spare SSH tunnel, `et`, and the two-pane `tmux` layout

Before running any remote deployment (`deploy-rs`, `colmena`, or `nixos-rebuild`), open your spare lifeline session first:

```bash
# Option A: Connect via Eternal Terminal (survives IP/interface bounces)
et ops@10.100.10.41:2022

# Option B: Dedicated SSH master tunnel with server keepalives on a separate socket
ssh -M -S ~/.ssh/lifeline-sock -o ServerAliveInterval=5 -o ServerAliveCountMax=6 ops@10.100.10.41
```

Once inside the host, escalate to `root` **immediately** (do not wait until `sudo` or PAM breaks during a bad switch!) and launch the two-pane `tmux` layout:

```bash
sudo -i
tmux new -s lifeline

# Inside tmux:
# 1. Record the known-good generation store path before touching anything
export GOOD_GEN=$(readlink -f /run/current-system)
echo "Known-good generation: $GOOD_GEN"

# 2. Split the window horizontally (Ctrl-b %) and start btop in the second pane
btop
```

Now trigger your deployment from a **separate** terminal window on your workstation (`deploy .#host` or `nixos-rebuild test`). Watch `btop` in the lifeline window as activation runs.

### Case 3: Quitting `btop`, diagnosing failures, and rolling back through the surviving tunnel

During activation, watch four indicators in `btop`:
- **Disk box (`/` and `/boot`)**: If `/boot` hits `100%`, kernel/UKI installation fails mid-switch.
- **Mem box**: If available RAM drops toward zero and swap I/O spikes to 100%, an unanticipated local compilation or service memory leak is about to trigger the kernel OOM killer.
- **Proc box**: Filter with `f` → `systemd` or `nix` to see which unit or derivation is stalling.

When a deployment goes wrong, **get out of `btop` and back to your root shell cleanly**:

1. **Normal exit:** Press **`q`** (or **`Esc`** → select **Quit**, or **`Ctrl-C`**) to exit `btop` immediately.
2. **If `btop` is frozen on blocked disk I/O or a graphical menu is stuck:** Because you started inside `tmux`, you are never trapped by a hung TUI. Press **`Ctrl-b ;`** (or **`Ctrl-b o`**) to jump focus to your pre-armed root shell pane, **`Ctrl-b c`** to spawn a fresh root window, or **`Ctrl-b x` → `y`** to kill the `btop` pane cleanly.

From your surviving root prompt in Pane 0, diagnose and roll back:

```bash
# 1. See which systemd units failed during switch-to-configuration
systemctl --failed
journalctl -xe --no-pager | tail -n 40

# 2. Fast standard rollback to previous generation
nixos-rebuild switch --rollback

# 3. Direct store-path rollback (works even if nix-daemon or nixos-rebuild is broken!)
"$GOOD_GEN/bin/switch-to-configuration" switch

# 4. Or pick a specific numbered generation from the system profile
ls -l /nix/var/nix/profiles/system-*-link
/nix/var/nix/profiles/system-41-link/bin/switch-to-configuration switch
```

Invoking `$GOOD_GEN/bin/switch-to-configuration switch` directly requires zero Nix evaluation, zero network access, and zero working `nixos-rebuild` wrappers. It atomically restores `/etc`, restarts `sshd` and `nftables` with the previous working configuration, and updates the bootloader entry.

### Case 4: Rapid zero-day patch velocity under pressure

Being able to build, verify, and ship an update across your fleet in minutes—without fear of bricking nodes—is what separates deterministic infrastructure from fragile snowflakes during an emergency CVE wave:

```bash
# 1. Pull the upstream security bump (or apply an overrideAttrs patch)
nix flake update nixpkgs

# 2. Build locally/on builder and read the exact package delta before activating
nixos-rebuild build --flake .#web-01
nix store diff-closures /run/current-system ./result

# 3. Test-activate without changing boot default while watching the lifeline tmux session
deploy .#web-01 --test

# 4. Verify health in the lifeline pane, then promote to permanent boot default
deploy .#web-01
```

Because the lifeline tunnel is open and `$GOOD_GEN/bin/switch-to-configuration switch` is pre-staged in `tmux`, even a regression in an emergency security patch can be reverted in three seconds.

## The trap

The trap is **opening your spare SSH session as an unprivileged user and waiting until *after* the deployment breaks to type `sudo -i`**. If the broken deployment corrupted PAM, `/etc/sudoers`, or NSS user resolution, `sudo` will reject you even though your SSH session survived. Always escalate to `root` (`sudo -i`) inside the lifeline `tmux` session **before** starting the deployment.

The second trap is **running `btop` directly inside a bare SSH session without `tmux`**. If you have only one surviving SSH connection and `btop` occupies it (or stalls polling a hung NFS/Ceph mount), you have no second pane to run `switch-to-configuration switch`. Always run `btop` inside a `tmux` pane next to an idle root prompt.

## The boring rule

- Every remote host runs `tmux`, `btop`, and `services.eternal-terminal` (or a separate SSH `ControlMaster` lifeline socket).
- Before any major switch or upgrade: open the lifeline session, `sudo -i` to root, start `tmux`, store `GOOD_GEN=$(readlink -f /run/current-system)`, and run `btop` in a split pane.
- Know the three ways out of `btop`: `q` (normal quit), `Ctrl-b o` (switch `tmux` pane), and `Ctrl-b x` (kill pane).
- Memorize `/nix/var/nix/profiles/system-*-link/bin/switch-to-configuration switch`—it rolls back the machine even when `nix` and `nixos-rebuild` are broken.

## Try this

1. Open `tmux`, split into two panes (`Ctrl-b %`), launch `btop` in one pane, practice switching panes (`Ctrl-b o`), and quit `btop` cleanly with `q`.
2. In a root shell on a lab VM, record `GOOD_GEN=$(readlink -f /run/current-system)`, deploy a trivial change to `/etc/issue`, and roll back directly using `"$GOOD_GEN/bin/switch-to-configuration" switch` without invoking `nixos-rebuild`.
3. Enable `services.eternal-terminal.enable = true` on a lab VM, connect with `et`, restart `sshd.service` and bounce the network interface inside the VM, and confirm your `et` session resumes automatically without dropping your `tmux` workspace.
