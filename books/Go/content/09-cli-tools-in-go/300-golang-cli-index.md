---
title: Building CLI Tools in Go
---

# Building CLI Tools in Go

This part is a **hands-on curriculum for command-line tools**: flags, subcommands, exit codes, pipes, config, signals, tests—first with **stdlib only**, then with **Cobra**, **pflag**, **Viper**, and brief notes on other frameworks.

Go is one of the best languages for CLIs: single static binary, fast startup, excellent `flag` / `os` / `io` / `os/exec`, and a culture of small, composable tools.

> **Related parts.** Stdlib surface: [987 flag / exec / signal](../07-stdlib/987-flag-os-exec-signal.md). Pipeline philosophy: [143 Unix pipeline CLIs](../17-systems-programming/143-unix-pipeline-cli-tools.md). Capstones in this part: [317 Network CLI tools](317-project-network-cli-tools.md) (`ping`, `dns`, `scan`, `probe`) and [318 Network security mini-tools](318-project-network-security-mini-tools.md) (`tls`, `headers`, `banner`, …). More labs: [99 projects](../99-projects/99-index.md) (`goping`, linux clones, log shipper).

## Learning outcomes

After this part you can:

1. Ship a correct `main` that parses flags, writes stdout/stderr properly, and exits with useful codes
2. Implement subcommands with only `flag.FlagSet`
3. Read env vars and simple config files without a framework
4. Compose stdin/stdout pipelines and spawn subprocesses safely
5. Cancel work on SIGINT with `context` + `os/signal`
6. Test CLIs without flaky subprocess soup
7. Grow into Cobra when subcommands and completion matter
8. Wire Viper-style config only when env/file/flag layering earns its keep
9. Build multi-command **network diagnostic** tools (TCP ping, DNS, port scan, HTTP probe)
10. Build **simple network security** mini-tools (TLS cert, headers, banner, secrets hygiene)

## Path map

```text
  Stdlib core:   anatomy → flag → subcommands → exits/streams
  Real tools:    config → pipes → exec → signals → tests → layout
  Ecosystem:     Cobra → advanced Cobra → Viper → other libs
  Practice:      shipping → cookbook → netkit → netsec mini-tools
```

| Chapter | Focus |
|---------|--------|
| 300 | This overview |
| 301 | First tools with `os.Args` |
| 302 | `flag` package in depth |
| 303 | Subcommands with `FlagSet` |
| 304 | Exit codes, stdout vs stderr |
| 305 | Env vars and config files |
| 306 | Stdin, files, and pipes |
| 307 | `os/exec` orchestration |
| 308 | Signals and context |
| 309 | Testing CLIs |
| 310 | Project layout for larger CLIs |
| 311 | Cobra basics |
| 312 | Cobra advanced (persist flags, hooks, completion) |
| 313 | Viper and config layering |
| 314 | Other frameworks (urfave/cli, pflag, charm notes) |
| 315 | Version, install, completion, release |
| 316 | Cookbook: many end-to-end examples |
| 317 | **Project:** network CLI tools (ping, dns, scan, probe) |
| 318 | **Project:** simple network security mini-tools (tls, headers, …) |
| 319 | **Project:** text/log tools (`textkit`) |
| 320 | **Project:** HTTP debug CLI (`httpdbg`) |
| 321 | Cross-platform CLI notes (Windows/paths/signals) |
| 322 | [Terminal TTY raw mode, ANSI rendering, and Bubble Tea](322-terminal-tty-raw-mode-bubble-tea.md) |
| 323 | [Command trees, zero-alloc flags, and `__complete` engines](323-command-trees-kong-cobra-completions.md) |
| 324 | [Pseudo-terminals (PTY), subprocess multiplexing, and SSH TUIs](324-pty-subprocess-multiplexing-ssh-tui.md) |
| 326 | [CLI packaging, atomic self-updates, and ML-DSA verification](326-cli-packaging-self-update-mldsa.md) |
| 327 | [Structured CLI config, XDG directories, and secret keychains](327-structured-cli-config-xdg-keychains.md) |
| 328 | [High-throughput terminal streaming, `$PAGER`, and renderers](328-terminal-streaming-pagers-renderers.md) |
| 329 | [CLI plugin architectures (`go-plugin`, `wazero` Wasm, `plugin` pitfalls)](329-cli-plugin-architectures-go-plugin-wazero.md) |
| 330 | [URL Monitor CLI: Basic & Advanced Features with `flag`](330-url-monitor-flag-cli.md) |
| 331 | [Structuring CLI Applications with Cobra](331-url-monitor-cobra-structure.md) |
| 332 | [Enhancing CLI Usability and Interactivity](332-cli-usability-and-interactivity.md) |
| 333 | [Testing CLI Applications (`httptest` & Cobra `ExecuteC`)](333-testing-cobra-and-http-clis.md) |
| 334 | [Error Handling & Logging in CLIs (Custom Errors & Log Rotation)](334-cli-error-handling-and-log-rotation.md) |
| 335 | [Distributing Your CLI (Docker, GoReleaser, Homebrew, Chocolatey, Deb/RPM)](335-distributing-go-clis.md) |

## Suggested schedule

```text
Days 1–2   301–304     stdlib CLI hygiene
Days 3–4   305–308     config, pipes, exec, signals
Day 5      309–310     tests + package layout
Day 6      311–313     Cobra (+ Viper if needed)
Day 7      314–316     ecosystem + cookbook tools
Day 8      317         build netkit (ping/dns/scan/probe)
Day 9      318         build netsec (tls/headers/banner/…)
Day 10     319–321     textkit, httpdbg, portability
```

## Mental model: a CLI is a function

```text
  argv + env + stdin + files
           │
           ▼
     ┌───────────┐
     │   parse   │  flags / subcommand / config
     └─────┬─────┘
           ▼
     ┌───────────┐
     │    run    │  pure-ish domain work (testable)
     └─────┬─────┘
           ▼
     stdout (data) · stderr (diagnostics) · exit code
```

Keep **`run(args, stdin, stdout, stderr) error`** (or a small `App` struct) so tests do not need `os.Exit` or global `flag.CommandLine`.

## Stdlib first, libraries later

| Stay on stdlib when… | Reach for Cobra/Viper when… |
|----------------------|-----------------------------|
| One binary, few flags | Many nested subcommands |
| Internal tools / scripts | Public developer product CLI |
| You want zero deps | Shell completion + man-page ecosystem |
| Teaching / small utilities | Flag inheritance across command trees |

Most famous Go CLIs started simple. Add libraries when the **command tree** or **config surface** hurts—not on day one of a toy.

## Running examples

Unless noted, examples assume:

```bash
mkdir -p /tmp/gocli && cd /tmp/gocli
go mod init example.com/gocli
# paste main.go or package files
go run .
```
