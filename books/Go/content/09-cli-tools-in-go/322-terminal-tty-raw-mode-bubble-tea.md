---
title: Terminal TTY Raw Mode, ANSI Rendering, and Bubble Tea Architecture
---

# Terminal TTY Raw Mode, ANSI Rendering, and Bubble Tea Architecture

Go powers many widely used command-line and terminal user interface (TUI) tools, including `kubectl`, `docker`, `gh`, `fzf`, `lazygit`, and `k9s`. Building a responsive, zero-flicker interactive CLI or TUI requires understanding what happens inside the OS kernel's TTY driver (`termios`), how raw keystrokes and ANSI escape sequences flow over file descriptors `0` (stdin) and `1` (stdout), why concurrent goroutines writing to `os.Stdout` corrupt terminal state, and how [Bubble Tea](https://github.com/charmbracelet/bubbletea) and [Lip Gloss](https://github.com/charmbracelet/lipgloss) achieve 60 FPS lock-free rendering.

## Kernel TTY Line Discipline: Canonical vs. Raw Mode

When a user launches a Go CLI inside a terminal emulator (iTerm2, Alacritty, Kitty, Ghostty, or `tmux`), `os.Stdin` is connected to a kernel pseudo-terminal (PTY) slave device (`/dev/pts/*`). By default, the kernel's TTY line discipline operates in **canonical ("cooked") mode**:

```text
1. Default Kernel "Cooked" (Canonical) Mode (ICANON | ECHO | ISIG | ONLCR)
+-------------------------------------------------------------------------------+
| User types: 'h' 'e' 'l' 'x' <Backspace> 'p' <Enter>                           |
| -> Kernel TTY driver buffers keystrokes internally, processes <Backspace>,    |
|    echoes characters to the screen, and converts Ctrl+C (0x03) into SIGINT.   |
| -> Go's os.Stdin.Read(buf) BLOCKS until <Enter> (\n) is pressed.              |
+-------------------------------------------------------------------------------+

2. Raw Terminal Mode via golang.org/x/term.MakeRaw(fd) (~ICANON & ~ECHO & ~ISIG)
+-------------------------------------------------------------------------------+
| -> Disables kernel line buffering, local echo, and signal translation via     |
|    ioctl(fd, TCSETS / TIOCSETA, &termios).                                    |
| -> Every keypress (arrow keys "\x1b[A", Ctrl+C 0x03) arrives IMMEDIATELY in   |
|    Go's os.Stdin.Read(buf) in sub-millisecond time.                           |
+-------------------------------------------------------------------------------+
```

### Kernel Flags Cleared by `term.MakeRaw(fd)`

- **`ICANON` (Canonical Input):** Disabling `ICANON` delivers bytes to `os.Stdin.Read` immediately without waiting for `\n`.
- **`ECHO` (Local Echo):** Disabling `ECHO` stops the kernel from printing typed characters onto the screen so your TUI controls every cell.
- **`ISIG` and `IXON` (Signal & Flow Control):** Disabling `ISIG` prevents the kernel from converting `Ctrl+C` (`0x03`), `Ctrl+Z` (`0x1A`), and `Ctrl+\` (`0x1C`) into OS signals—delivering them as raw bytes to your input parser instead.
- **`ONLCR` (Output Newline Translation):** Disabling `ONLCR` means writing `\n` (`0x0A`) moves the cursor down one row **without returning to column 0**. In raw mode, always emit `\r\n` (`0x0D 0x0A`) to start a new line at column 0.

## Alternate Screen Buffer and Restoration Invariant

Full-screen TUIs (`k9s`, `lazygit`, `htop`, `vim`) preserve the user's shell scrollback by emitting ANSI CSI (Control Sequence Introducer) escape sequences:

| Escape Sequence | Name | Terminal Action |
|---|---|---|
| `\x1b[?1049h` | Enter Alternate Screen | Saves cursor position and switches to a clean, scrollback-free viewport buffer. |
| `\x1b[?1049l` | Exit Alternate Screen | Restores the user's original shell screen and cursor position. |
| `\x1b[?25l` / `\x1b[?25h` | Hide / Show Cursor | Hides the hardware blinking cursor during TUI rendering and restores it on exit. |
| `\x1b[2J\x1b[H` | Clear Viewport & Home | Clears the active viewport and moves the cursor to row 1, column 1. |

> [!WARNING]
> **The broken-terminal hazard:** If a raw-mode Go program returns on an error or panics without calling `term.Restore(fd, oldState)` and writing `\x1b[?25h\x1b[?1049l`, the user's shell stays stuck in raw mode with `ECHO` disabled—typed characters are invisible and `<Enter>` produces staircase text. Always pair `term.MakeRaw(fd)` immediately with a `defer` block that restores both `termios` and the ANSI screen buffer.

## Lock-Free Elm Architecture in Bubble Tea

If Goroutine A writes `\x1b[31mError\x1b[0m` at the same microsecond Goroutine B writes `\x1b[2J`, the OS kernel can interleave bytes (`\x1b[31\x1b[2JmError...`), printing corrupted control sequences across the screen.

[Bubble Tea](https://github.com/charmbracelet/bubbletea) eliminates terminal races and UI mutexes via the Elm Architecture:

```text
                   Bubble Tea Lock-Free Elm Event Loop
+-----------------------------------------------------------------------+
|  [ Model (Single-Owner State Struct — Zero Mutexes Required) ]        |
|        |                                                  ^           |
|        v View() string                                    |           |
|  [ Frame-Capped Diff Renderer (60 FPS) ]             Update(msg)      |
|  (Buffers & writes ONLY changed lines in 1 Write)         |           |
|                                                           |           |
|  [ Async tea.Cmd Goroutines / Keypresses / SIGWINCH ] ----+ (tea.Msg) |
+-----------------------------------------------------------------------+
```

1. **Single-Goroutine State Ownership (`Model`):** Only the Bubble Tea event loop calls `Update(msg)` and `View()`. Your UI state struct requires zero `sync.Mutex` locks.
2. **Asynchronous I/O via `tea.Cmd`:** Never run blocking HTTP calls, disk I/O, or `time.Sleep` inside `Update` or `View`. Return a `tea.Cmd` (`func() tea.Msg`); the runtime runs it on a background goroutine and sends the resulting `tea.Msg` back into `Update`.
3. **Frame-Capped Buffered Diff Rendering:** Even if workers emit 10,000 messages/sec, Bubble Tea coalesces updates and renders `View()` at most once per 16.6 ms (60 FPS), flushing a single buffered `os.Stdout.Write` per frame.

## Complete Raw-Mode TUI Engine in Go 1.27

This runnable Unix implementation switches stdin to raw mode via `golang.org/x/term`, handles `SIGWINCH` terminal resizes, buffers frame writes into a single atomic `Write`, and guarantees panic-safe TTY restoration:

```go
//go:build unix

package tuiengine

import (
	"bytes"
	"context"
	"fmt"
	"io"
	"os"
	"os/signal"
	"syscall"

	"golang.org/x/term"
)

const (
	enterAltScreen = "\x1b[?1049h\x1b[?25l"
	exitAltScreen  = "\x1b[?25h\x1b[?1049l"
	clearAndHome   = "\x1b[2J\x1b[H"
)

// RunRawTUI switches stdin into Raw Mode, handles SIGWINCH window resizes,
// buffers frame writes to prevent flicker, and guarantees TTY restoration.
func RunRawTUI(ctx context.Context, in *os.File, out io.Writer) error {
	fd := int(in.Fd())
	if !term.IsTerminal(fd) {
		return fmt.Errorf("fd %d is not a terminal TTY", fd)
	}

	oldState, err := term.MakeRaw(fd)
	if err != nil {
		return fmt.Errorf("enable raw mode: %w", err)
	}

	// Restore terminal state on EVERY exit or panic path.
	_, _ = io.WriteString(out, enterAltScreen)
	defer func() {
		_, _ = io.WriteString(out, exitAltScreen)
		_ = term.Restore(fd, oldState)
	}()

	winchCh := make(chan os.Signal, 1)
	signal.Notify(winchCh, syscall.SIGWINCH)
	defer signal.Stop(winchCh)

	var frameBuf bytes.Buffer
	width, height, _ := term.GetSize(fd)
	flushFrame(out, &frameBuf, width, height, "Ready — press 'q' or Ctrl+C to quit")

	keyCh := make(chan byte, 16)
	errCh := make(chan error, 1)
	go func() {
		var b [1]byte
		for {
			n, readErr := in.Read(b[:])
			if n > 0 {
				select {
				case keyCh <- b[0]:
				case <-ctx.Done():
					return
				}
			}
			if readErr != nil {
				errCh <- readErr
				return
			}
		}
	}()

	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-winchCh:
			width, height, _ = term.GetSize(fd)
			flushFrame(out, &frameBuf, width, height, "Viewport resized")
		case key := <-keyCh:
			// In Raw Mode, Ctrl+C arrives as byte 0x03 (ETX).
			if key == 0x03 || key == 'q' {
				return nil
			}
			flushFrame(out, &frameBuf, width, height, fmt.Sprintf("Key byte: 0x%02x (%q)", key, key))
		case readErr := <-errCh:
			return readErr
		}
	}
}

func flushFrame(out io.Writer, buf *bytes.Buffer, width, height int, status string) {
	buf.Reset()
	// Always use \r\n in raw mode because kernel ONLCR is disabled.
	fmt.Fprintf(buf, "%s=== Go 1.27 Raw TUI (%dx%d) ===\r\nStatus: %s\r\n",
		clearAndHome, width, height, status)
	// Single Write prevents intermediate frame tearing.
	_, _ = out.Write(buf.Bytes())
}
```

## Double-SIGINT Graceful Shutdown and SIGPIPE Handling

For non-interactive CLIs piped into `head` or `less`, two POSIX signal patterns are essential:

### Double-SIGINT (Graceful on 1st Ctrl+C, Force-Exit on 2nd Ctrl+C)

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

// Unregister the signal trap as soon as the 1st Ctrl+C cancels ctx,
// so a 2nd Ctrl+C terminates immediately via the OS default handler.
context.AfterFunc(ctx, func() {
	stop()
})
```

### Handling Broken Pipes (`my-cli | head -n 5`)

When `head -n 5` reads 5 lines and closes its pipe, the next write to `os.Stdout` returns `syscall.EPIPE` (`broken pipe`). Check `errors.Is(err, syscall.EPIPE)` on stdout writes and exit cleanly (`os.Exit(0)`) instead of printing an error trace.

## Ecosystem Tooling and Verification

- **Compile-time TTY restoration checks with [gohawk](https://gohawk.dev/):** Calling `term.MakeRaw(fd)` or opening a PTY acquires stateful kernel resources. Running `go tool gohawk ./...` applies interprocedural SSA `resourcelifetime` analysis to verify that TTY restoration runs on every return branch.
- **Declarative styling and deterministic TUI tests:** Pair [Lip Gloss](https://github.com/charmbracelet/lipgloss) for CSS-like box layout and adaptive color profiles (`TrueColor` / `ANSI256` / `NoTTY`) with `charmbracelet/x/exp/teatest` or [VHS](https://github.com/charmbracelet/vhs) `.tape` recordings for automated TUI regression tests.

## Related Chapters

- [308 Signals and Context](308-signals-context-cli.md)
- [324 Pseudo-Terminals (PTY), Subprocess Multiplexing, and SSH TUI Apps](324-pty-subprocess-multiplexing-ssh-tui.md)
- [328 High-Throughput Terminal Streaming, Pagers, and Renderers](328-terminal-streaming-pagers-renderers.md)
