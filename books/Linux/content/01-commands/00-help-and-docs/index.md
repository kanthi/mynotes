---
title: Intro
---

# Intro

This part covers how to find answers **on the system itself** (and in concise offline cheat sheets) before searching the web. Local docs match the versions you actually installed — critical when flags change between releases.

## Commands in this part

| Command | Role |
|---------|------|
| `man` | man displays the system manual. |
| help (shell builtin) | help shows documentation for shell builtins — commands implemented inside the shell itself (cd, export, read, [[,… |
| `info` | info reads documentation in Info (Texinfo) format. |
| `apropos` | apropos searches man page names and short descriptions for keywords. |
| `whatis` | whatis prints single-line descriptions from the man-db whatis index. |
| `tldr` | tldr displays concise, example-first cheat sheets for common CLI commands. |

## Which doc tool should I use?

| Need | Best Choice | Why |
|------|-------------|-----|
| Top 5 real-world examples fast | `tldr cmd` | Skips flag walls, shows copyable patterns |
| Exact flags, exit codes, and edge cases | `man cmd` | Authoritative for your installed version |
| Builtin syntax (`read`, `set`, `trap`, `[[`) | `help cmd` | Builtins do not have separate `man` pages |
| Forgot the command name | `apropos keyword` | Searches man-db short descriptions |

## Suggested starting points

1. Quick examples: `tldr cmd` or `cmd --help`.
2. Quick one-line summary: `whatis cmd`.
3. Full story: `man cmd` (try section 5 for configs, 8 for admin tools).
4. Forgot the name: `apropos keyword`, then open the best hit with `man`.
5. Shell builtin: `type cmd`, then `help cmd` if it is a builtin.
6. GNU deep dive: `info` for coreutils, bash, and related tools.

## Related parts

- `type` / `command -v` (Shell commands) — resolve names
- Book introduction — how pages are structured

Continue with the individual command pages in this part.
