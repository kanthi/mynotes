# systemd-analyze

## Overview

`systemd-analyze` diagnoses **boot performance**, validates unit files, tests calendar expressions, and audits service **sandboxing security**. Reach for it whenever a server boots slowly, after editing a `.service` or `.timer` file, or when hardening systemd units.

## Syntax

```bash
systemd-analyze [options] [subcommand]
```

## Common Subcommands

| Subcommand | Description |
|------------|-------------|
| `time` | Print kernel, initrd, and userspace boot times |
| `blame` | List units sorted by initialization duration (slowest first) |
| `critical-chain [UNIT]` | Show the blocking dependency chain to `default.target` |
| `plot > boot.svg` | Generate an SVG waterfall timeline of the entire boot |
| `verify FILE...` | Lint unit files for syntax errors, missing binaries, or cycles |
| `calendar SPEC` | Parse and preview next run times for `OnCalendar=` expressions |
| `security [UNIT]` | Audit sandboxing exposure and print a hardening score (`0.0` to `10.0`) |
| `dot [UNIT]` | Export unit dependency graph in Graphviz `dot` format |

## Key Use Cases

1. Pinpoint which service or mount delayed system boot (`time`, `blame`, `critical-chain`)
2. Catch typos and broken paths in new `.service` files before restarting (`verify`)
3. Test `OnCalendar=` timer schedules before deploying (`calendar`)
4. Audit how exposed a daemon is (`security nginx.service`) and what hardening directives to add

## Examples with Explanations

### Diagnose a slow boot

```bash
systemd-analyze time
systemd-analyze blame | head -n 15
systemd-analyze critical-chain
```

`blame` shows raw durations (including parallel units); `critical-chain` highlights the actual **blocking** path that delayed reaching login/target.

### Generate a visual SVG boot timeline

```bash
systemd-analyze plot > /tmp/boot.svg
```

Open `/tmp/boot.svg` in a browser to see parallel vs. serialized startup visually.

### Validate a custom unit file after editing

```bash
sudo systemd-analyze verify /etc/systemd/system/myapp.service
```

Flags unknown keys, invalid `ExecStart=` paths, and ordering cycles.

### Preview `OnCalendar=` timer schedules

```bash
systemd-analyze calendar 'Mon..Fri *-*-* 09:00:00'
systemd-analyze calendar --iterations=3 'weekly'
```

### Audit service sandboxing posture

```bash
systemd-analyze security ssh.service | head -n 25
```

Lists each hardening option (`ProtectSystem=`, `PrivateTmp=`, `NoNewPrivileges=`) and whether the service restricts it.

## Understanding Output

In `systemd-analyze critical-chain`:

* Units printed in **red** (or with `@TIME +DURATION`) indicate the time spent waiting before the unit became active (`@` = start timestamp, `+` = time taken).
* A slow unit in `blame` is **not** slowing boot unless it also appears on `critical-chain`.

## Notes & Pitfalls

- First boot after a kernel or package upgrade can inflate `blame` times (e.g., `ldconfig`, `mandb`, or `dkms` rebuilds).
- `systemd-analyze` requires systemd as PID 1; inside Docker/Podman containers without systemd, boot metrics are unavailable.
- `NetworkManager-wait-online.service` or `systemd-networkd-wait-online.service` and slow NFS mounts are the most common culprits in `critical-chain`.

## Related Commands

- `systemctl` — inspect and manage units
- `journalctl -b` — read full boot logs
- `systemd-run` — run transient units and timers
- `bootctl` — inspect UEFI systemd-boot status

## Additional Resources

- `man systemd-analyze`
