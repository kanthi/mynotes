# script & scriptreplay

## Overview

`script` records everything printed to a terminal session (commands, outputs, colors, and optionally input/timing) into a typescript file. `scriptreplay` plays that recording back in real time (or sped up) using the saved timing file—making them invaluable for **audit trails during risky maintenance**, reproducible bug reports, and terminal demos. Part of `util-linux`.

## Syntax

```bash
script [options] [typescript_file]
scriptreplay [options] timing_file [typescript_file]
```

## Common Options

| Option | Description |
|--------|-------------|
| `-a`, `--append` | Append to an existing typescript file instead of overwriting |
| `-q`, `--quiet` | Suppress the `Script started` / `Script done` banner messages |
| `-c CMD`, `--command CMD` | Run a single command non-interactively and record its session |
| `-e`, `--return` | Return the exit status of the child process (useful in CI/automation) |
| `-f`, `--flush` | Flush output after every write (great for live tailing/audits) |
| `--log-out FILE` / `--log-timing FILE` | Modern `util-linux` flags to log output and timing separately |
| `-T FILE` / `-t[FILE]` | Record timing data for `scriptreplay` |

## Key Use Cases

1. Keep a complete verbatim log of a production database migration or server upgrade
2. Record a reproducible terminal session with timing (`scriptreplay`) for training or bug reports
3. Capture colorized/TTY-only output from tools that suppress color when piped

## Safety

- **Everything printed to stdout/stderr is saved verbatim**—avoid typing plaintext passwords or printing `.env` secrets while recording (and do not enable `--log-in` unless you specifically intend to capture keystrokes).
- Remember to type `exit` (or `Ctrl+D`) when finished so `script` closes the subshell and flushes the file.

## Examples with Explanations

### Record a maintenance session to a file

```bash
script -a /var/log/upgrade-2026-11-18.log
# Run your commands...
exit
```

### Record both output and timing for `scriptreplay`

```bash
script --log-out session.log --log-timing session.time
# Run interactive commands, then exit
scriptreplay --timing session.time --typescript session.log
```

### Replay at 2x speed (or cap pauses to 1 second max)

```bash
scriptreplay -d 2 -m 1 --timing session.time --typescript session.log
```

`-d 2` plays twice as fast; `-m 1` skips long idle pauses while thinking.

### Record a single command and preserve its exit code

```bash
script -qec "sudo apt update && sudo apt upgrade -y" /tmp/apt-run.log
```

### Force TTY/color output for a command in a non-interactive script

```bash
script -qec "ls --color=always /etc" /dev/null
```

## Notes & Pitfalls

- `script` spawns a subshell (`$SHELL`), so nested `script` sessions can get confusing—check `$SHLVL` or `ps` if you forget you are already inside one.
- Raw typescript files contain ANSI escape codes (`\e[32m`, `\r`); view them cleanly with `less -R` or strip ANSI sequences with `col -b < session.log`.

## Related Commands

- `tmux` (`capture-pane` / `pipe-pane`) — capture output inside persistent sessions
- `tee` — duplicate plain stdout/stderr without allocating a PTY
- `history` — recall shell commands (without their output)

## Additional Resources

- `man script`
- `man scriptreplay`
