---
title: 'Project: Mini Process Supervisor (minisup)'
---

# Project: Mini Process Supervisor (`minisup`)

Build a lightweight process supervisor in Go that launches a child process in its own process group, restarts it with exponential backoff on unexpected exit, and forwards `SIGINT`/`SIGTERM` cleanly to the entire child process group on shutdown.

## Lifecycle State Machine

```text
Start -> Spawn Child (Setpgid: true) -> Wait
            ^                             |
            |    (unexpected exit)        +---> ctx.Done() -> SIGTERM (-pgid) -> grace timeout -> SIGKILL (-pgid)
            +-- Backoff (1s..30s) <-------+
```

## Core Supervisor Loop

```go
package main

import (
	"context"
	"log/slog"
	"os"
	"os/exec"
	"os/signal"
	"syscall"
	"time"
)

func supervise(ctx context.Context, name string, args ...string) error {
	backoff := time.Second
	const maxBackoff = 30 * time.Second

	for {
		cmd := exec.Command(name, args...)
		cmd.Stdout = os.Stdout
		cmd.Stderr = os.Stderr
		// Put child in its own process group so we can signal the whole tree.
		cmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}

		started := time.Now()
		if err := cmd.Start(); err != nil {
			return err
		}
		pgid := cmd.Process.Pid
		slog.Info("child started", "pid", pgid, "cmd", name)

		done := make(chan error, 1)
		go func() { done <- cmd.Wait() }()

		select {
		case <-ctx.Done():
			slog.Info("shutting down child process group", "pgid", pgid)
			_ = syscall.Kill(-pgid, syscall.SIGTERM)
			select {
			case <-done:
			case <-time.After(5 * time.Second):
				slog.Warn("grace period expired, sending SIGKILL", "pgid", pgid)
				_ = syscall.Kill(-pgid, syscall.SIGKILL)
				<-done
			}
			return nil

		case err := <-done:
			if time.Since(started) > 30*time.Second {
				backoff = time.Second // reset backoff after healthy run
			}
			slog.Warn("child exited, restarting", "err", err, "backoff", backoff)
			select {
			case <-ctx.Done():
				return nil
			case <-time.After(backoff):
				backoff = min(backoff*2, maxBackoff)
			}
		}
	}
}

func main() {
	if len(os.Args) < 2 {
		os.Exit(2)
	}
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()
	if err := supervise(ctx, os.Args[1], os.Args[2:]...); err != nil {
		slog.Error("supervisor failed", "err", err)
		os.Exit(1)
	}
}
```

## Verification

- Run `minisup sh -c 'echo hi; exit 1'` and observe exponential backoff restarts (`1s`, `2s`, `4s`).
- Send `SIGINT` (`Ctrl+C`) and verify no orphan child processes remain (`ps -ef`).
