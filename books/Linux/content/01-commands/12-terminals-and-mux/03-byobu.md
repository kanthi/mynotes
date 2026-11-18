# byobu

## Overview

`byobu` is a user-friendly wrapper and configuration layer on top of **`tmux`** (default) or **`screen`**. It provides ready-to-use function-key shortcuts (`F2`–`F12`), a live system-status bar (CPU, memory, disk, IP, load), and effortless session persistence over SSH without needing to memorize `Ctrl+b` prefix chords or write a custom `.tmux.conf`.

```bash
sudo apt install byobu   # Debian/Ubuntu
sudo dnf install byobu   # Fedora/RHEL
```

## Syntax

```bash
byobu [options] [command]
byobu-enable            # auto-launch byobu on every SSH/console login
```

## Common Options & Function Keys

| Option / Key | Description |
|--------------|-------------|
| `byobu` | Start a new session or reattach to an existing one |
| `byobu new -s NAME` | Start a named session |
| `byobu list-sessions` | List running sessions |
| `byobu attach -t NAME` | Reattach to a specific named session |
| `byobu-select-backend` | Choose between `tmux` (recommended) and `screen` |
| `F2` | Create a new window |
| `F3` / `F4` | Switch to previous / next window |
| `F6` | Detach from the session (keeps processes running) |
| `F7` | Enter scrollback / copy mode (use arrows or `/` to search) |
| `F8` | Rename the current window |
| `F9` | Open the interactive configuration & status-bar menu |
| `Shift-F2` / `Ctrl-F2` | Split pane horizontally / vertically |

## Key Use Cases

1. Keep long-running admin tasks alive across dropped SSH connections (`F6` to detach)
2. Get an instant `tmux` setup with a hardware/network status bar out of the box
3. Auto-attach into a persistent workspace on every SSH login (`byobu-enable`)

## Examples with Explanations

### Start, name, and reattach to sessions

```bash
byobu new -s deploy
# Press F6 to detach safely, then later:
byobu list-sessions
byobu attach -t deploy
```

### Enable automatic launch on SSH login (and disable it)

```bash
byobu-enable
byobu-disable
```

### Switch backend to `tmux` or `screen`

```bash
byobu-select-backend
```

### Start a session running a specific command

```bash
byobu new-session -d -s logs 'journalctl -f'
```

Launches a detached `byobu` session tailing `journalctl -f` in the background.

### Share a session read-only or collaboratively (via underlying `tmux`)

```bash
byobu attach -t deploy -r   # read-only attach
```

## Notes & Pitfalls

- On macOS or certain terminal emulators, function keys (`F2`–`F12`) may be intercepted by the OS or terminal—use `Shift-F1` inside `byobu` to see alternate `Ctrl-a` / `Alt` bindings, or switch to plain `tmux`.
- `byobu` passes unknown subcommands straight through to its backend (`tmux`), so standard `tmux` commands like `kill-session -t NAME` work seamlessly.

## Related Commands

- `tmux` — underlying modern terminal multiplexer
- `screen` — classic terminal multiplexer
- `tmate` — instant SSH-based terminal sharing
- `abduco` — minimal session detach/attach tool

## Additional Resources

- `man byobu`
- `man byobu-config`
