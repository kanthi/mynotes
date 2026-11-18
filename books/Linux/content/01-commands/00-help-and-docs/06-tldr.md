# tldr

## Overview

`tldr` (from the **tldr-pages** project) displays concise, community-maintained, **example-first cheat sheets** for command-line tools. Instead of scrolling through hundreds of lines of flags in a `man` page, `tldr` shows the 5–8 most common real-world invocations right away.

```bash
sudo apt install tldr        # or tealdeer / tlrc
sudo dnf install tldr
```

## Syntax

```bash
tldr [options] <command>
```

## Common Options

| Option | Description |
|--------|-------------|
| `-u`, `--update` | Update the local offline cache of cheat sheets |
| `-p`, `--platform` | Show platform-specific examples (`linux`, `osx`, `common`) |
| `-l`, `--list` | List all available pages in the local cache |
| `-r`, `--render FILE` | Preview a local Markdown tldr page |

## Key Use Cases

1. Remember the exact flags for `tar`, `find`, `ss`, or `ffmpeg` in 2 seconds
2. Quickly check common subcommands of complex tools (`git`, `systemctl`, `docker`)
3. Keep an offline-friendly quick reference alongside `man`

## Examples with Explanations

### Look up practical examples for a command

```bash
tldr tar
tldr ss
tldr rsync
```

### Look up a subcommand

```bash
tldr git-rebase
tldr ip-route
tldr systemctl
```

### Force Linux-specific examples and refresh cache

```bash
tldr -u
tldr -p linux lsof
```

## Notes & Pitfalls

- `tldr` is a **complement**, not a replacement for `man`: it intentionally omits rare flags, edge cases, and exit-code details. Always check `man` before running unfamiliar destructive flags.
- Several clients exist (`tlrc` in Rust, `tealdeer`, Python `tldr`, Node `tldr`); flag names like `--update` vs. `-u` vary slightly by client.

## Related Commands

- `man` — complete offline system manual
- `help` — documentation for shell builtins
- `apropos` / `whatis` — search man page descriptions

## Additional Resources

- `tldr tldr`
- `https://tldr.sh/`
