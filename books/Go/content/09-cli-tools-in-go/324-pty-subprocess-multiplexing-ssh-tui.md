---
title: Pseudo-Terminals (PTY), Subprocess Multiplexing, and SSH TUI Apps
---

# Pseudo-Terminals (PTY), Subprocess Multiplexing, and SSH TUI Apps

When building container CLIs (`docker exec -it`, `kubectl exec -it`), terminal multiplexers, web-based cloud shells, CI/CD log runners, or SSH-hosted terminal apps, standard `os/exec.Cmd` pipes (`cmd.StdoutPipe()`) immediately hit a wall. When a child process inspects `isatty(1)` (`ioctl(fd, TCGETS)`) and sees a FIFO pipe instead of a terminal character device:

1. **Buffered output stalls:** `libc` switches stdout from line-buffered (`_IOLBF`) to 4 KB block-buffered (`_IOFBF`), delaying log lines until 4 KB accumulates.
2. **Colors and progress bars vanish:** CLI tools disable ANSI color and interactive spinners when `!term.IsTerminal(1)`.
3. **Interactive programs refuse to run:** `vim`, `htop`, `bash`, and `psql` fail with `ENOTTY` (`Inappropriate ioctl for device`).

The kernel **Pseudo-Terminal (PTY)** subsystem solves this—paired in Go with [creack/pty](https://github.com/creack/pty), [Wish](https://github.com/charmbracelet/wish) for SSH-hosted TUIs, and [Huh](https://github.com/charmbracelet/huh) for accessible interactive forms.

## Kernel PTY Architecture: `/dev/ptmx` and `/dev/pts/N`

```text
Parent Go Process (Terminal Controller)            Child Subprocess (bash / vim / top)
+-----------------------------------------+        +-----------------------------------------+
| Holds PTY Master FD (ptmx *os.File)     |        | Stdin(0), Stdout(1), Stderr(2) bound    |
| - Writes user keystrokes into ptmx      |        |   to PTY Slave FD (/dev/pts/4)          |
| - Reads ANSI screen output from ptmx    |        | - term.IsTerminal(0/1/2) == true        |
| - Calls ioctl(ptmx, TIOCSWINSZ, &ws)    |        | - Receives SIGWINCH on window resize    |
+--------------------+--------------------+        +--------------------+--------------------+
                     |                                                  ^
                     v                                                  |
+-----------------------------------------------------------------------+--------------------+
|                         OS Kernel PTY Driver & Line Discipline (termios)                   |
|             /dev/ptmx (Master Multiplexer)  <===>  /dev/pts/4 (Slave Character Device)     |
+--------------------------------------------------------------------------------------------+
```

### The Five Kernel Steps in `pty.StartWithSize`

1. **`posix_openpt` (`open("/dev/ptmx", O_RDWR|O_NOCTTY)`):** Allocates a new virtual master file descriptor (`ptmx`) and slave node `/dev/pts/<N>`.
2. **`grantpt` and `unlockpt` (`ioctl(ptmx, TIOCSPTLCK, 0)`):** Sets slave permissions and unlocks `/dev/pts/<N>`.
3. **`ioctl(ptmx, TIOCSWINSZ, &Winsize)`:** Sets initial row/column geometry before the child process starts—avoiding the startup race where a child reads `0x0` or `80x24`.
4. **Bind streams:** Opens `/dev/pts/<N>` and assigns `cmd.Stdin`, `cmd.Stdout`, and `cmd.Stderr`.
5. **Session leader & controlling terminal:** Sets `cmd.SysProcAttr = &syscall.SysProcAttr{Setsid: true, Setctty: true}` so the child calls `setsid(2)` and `ioctl(0, TIOCSCTTY, 0)` to own `Ctrl+C` and `SIGWINCH`, then closes the parent's copy of the slave FD.

## Two Production PTY Traps in Go

> [!WARNING]
> **Trap 1 — Linux `syscall.EIO` on child exit:** On macOS and BSD, when the child exits and closes the last slave FD, `ptmx.Read(buf)` returns `io.EOF`. On **Linux**, `ptmx.Read(buf)` returns `syscall.EIO` (`input/output error`). Always check `errors.Is(err, syscall.EIO)` alongside `io.EOF` when copying from a PTY master.

> [!IMPORTANT]
> **Trap 2 — Double-echo on the outer terminal:** If you relay `os.Stdin` into `ptmx`, both the outer terminal and the inner PTY slave have `ECHO` and `ICANON` enabled by default. Switch the outer `os.Stdin` into raw mode (`term.MakeRaw`) so keystrokes (`Ctrl+C`, arrows, `<Tab>`) pass straight through to the inner PTY's line discipline.

## Complete Interactive PTY Attach Engine in Go 1.27

```go
//go:build unix

package ptyrunner

import (
	"context"
	"errors"
	"fmt"
	"io"
	"os"
	"os/exec"
	"os/signal"
	"sync"
	"syscall"

	"github.com/creack/pty"
	"golang.org/x/term"
)

// RunInteractivePTY spawns cmd attached to a new kernel Pseudo-Terminal,
// synchronizes SIGWINCH window resizes, puts the host stdin into Raw Mode,
// and cleanly handles Linux EIO on subprocess exit.
func RunInteractivePTY(ctx context.Context, cmd *exec.Cmd) error {
	stdinFD := int(os.Stdin.Fd())
	if !term.IsTerminal(stdinFD) {
		return fmt.Errorf("stdin (fd %d) must be a terminal TTY", stdinFD)
	}

	initialSize, err := pty.GetsizeFull(os.Stdin)
	if err != nil {
		return fmt.Errorf("get terminal size: %w", err)
	}

	ptmx, err := pty.StartWithSize(cmd, initialSize)
	if err != nil {
		return fmt.Errorf("start pty: %w", err)
	}
	defer func() { _ = ptmx.Close() }()

	oldTermState, err := term.MakeRaw(stdinFD)
	if err != nil {
		return fmt.Errorf("make raw host tty: %w", err)
	}
	defer func() { _ = term.Restore(stdinFD, oldTermState) }()

	winchCh := make(chan os.Signal, 1)
	signal.Notify(winchCh, syscall.SIGWINCH)
	defer signal.Stop(winchCh)

	stopCtx, cancel := context.WithCancel(ctx)
	defer cancel()

	var wg sync.WaitGroup
	// Go 1.25+: WaitGroup.Go manages goroutine lifecycles without Add/Done boilerplate.
	wg.Go(func() {
		for {
			select {
			case <-stopCtx.Done():
				return
			case <-winchCh:
				_ = pty.InheritSize(os.Stdin, ptmx)
			}
		}
	})

	// Stream child PTY output -> host os.Stdout (ignoring Linux EIO on child exit).
	wg.Go(func() {
		_, copyErr := io.Copy(os.Stdout, ptmx)
		if copyErr != nil && !errors.Is(copyErr, io.EOF) && !errors.Is(copyErr, syscall.EIO) {
			// Log unexpected I/O error to stderr if needed.
		}
	})

	// Stream host os.Stdin -> child PTY master.
	go func() {
		_, _ = io.Copy(ptmx, os.Stdin)
	}()

	waitErr := cmd.Wait()
	cancel()
	// Close ptmx before wg.Wait() so io.Copy unblocks even if a grandchild kept the slave open.
	_ = ptmx.Close()
	wg.Wait()

	return waitErr
}
```

## Serving Multi-User TUIs Over SSH with Wish and Forms with Huh

- **Zero-install TUI apps via [Wish](https://github.com/charmbracelet/wish):** Wish wraps a pure-Go SSH server. When a client connects with `ssh portal.internal`, the SSH protocol negotiates `pty-req` (`TERM`, columns, rows) and forwards `window-change` resize messages. Wish's `bubbletea.Middleware` bridges each `ssh.Session` (`io.Reader`/`io.Writer`) into an isolated per-connection `tea.Program` goroutine.
- **Accessible CLI wizards via [Huh](https://github.com/charmbracelet/huh):** Composable forms (`huh.NewSelect`, `huh.NewMultiSelect`, `huh.NewInput`, `huh.NewConfirm`) built on Bubble Tea that bind directly to Go structs and degrade cleanly into screen-reader mode (`WithAccessible(os.Getenv("ACCESSIBLE") != "")`).

## Compile-Time PTY Verification

PTY runners combine three leak-prone resources: the master `/dev/ptmx` descriptor (exhausting `kernel.pty.max`, typically 4,096 system-wide), the mutated host `termios` state, and relay goroutines. Run [gohawk](https://gohawk.dev/) (`go tool gohawk ./...`) to verify both `resourcelifetime` (`ptmx.Close()` and `term.Restore()`) and `goroutineownership` (`wg.Wait()`) across every return branch.

## Related Chapters

- [152 PTY Terminals](../17-systems-programming/152-pty-terminals.md)
- [322 Terminal TTY Raw Mode, ANSI Rendering, and Bubble Tea Architecture](322-terminal-tty-raw-mode-bubble-tea.md)
- [307 os/exec Orchestration](307-os-exec-orchestration.md)
