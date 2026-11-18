---
title: Intro
---

# Intro

Keep shells alive across disconnects, split panes, share sessions, and record terminal audits. `tmux` is the usual default; `byobu` adds beginner-friendly function keys; `script` records verbatim terminal transcripts.

## Commands in this part

| Command | Role |
|---------|------|
| `tmux` | tmux is a terminal multiplexer: multiple windows and panes inside detachable sessions that survive SSH disconnects. |
| `screen` | screen is the classic terminal multiplexer: detachable sessions that survive hangups. |
| `byobu` | The byobu command is a text-based window manager and terminal multiplexer. |
| `tmate` | tmate is a fork of tmux focused on instant terminal sharing. |
| `abduco` | abduco is a minimal session attach/detach tool: it runs a process under a controlling terminal session you can… |
| `script` / `scriptreplay` | script records everything printed in a terminal session to a file; scriptreplay plays it back with timing. |

## Suggested starting points

1. Daily driver: `tmux` (sessions, windows, panes).
2. Legacy hosts: `screen`.
3. Convenience wrapper: `byobu`.
4. Remote pair/share: `tmate`; minimal attach tools: `abduco`.
5. Record maintenance logs & demos: `script` / `scriptreplay`.

## Related parts

- Networking — `ssh` into the host first
- Processes and jobs — reattach instead of losing long jobs
- Shell commands — history inside each pane

Continue with the individual command pages in this part.
