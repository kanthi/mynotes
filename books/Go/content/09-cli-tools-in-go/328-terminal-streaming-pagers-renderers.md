---
title: High-Throughput Terminal Streaming, Pagers, and Table-Markdown Renderers
---

# High-Throughput Terminal Streaming, Pagers, and Table-Markdown Renderers

Developer CLIs like `git`, `gh`, `kubectl`, and `docker` spend a surprising share of CPU formatting output for `os.Stdout`. When a command emits 50,000 log lines, a wide resource table, or a Markdown changelog, naive rendering triggers three failures:

1. **Syscall thrashing:** Calling `fmt.Printf` per row triggers tens of thousands of unbuffered `write(2)` syscalls and visible flicker.
2. **Broken table alignment:** `len(s)` counts UTF-8 bytes and `fmt.Sprintf("%-20s", s)` counts runes—neither equals visible terminal columns once ANSI color escapes or wide Unicode graphemes appear.
3. **Broken-pipe (`EPIPE`) crashes with `$PAGER`:** Pressing `q` early in `less -FIRX` closes the pager's stdin pipe and surfaces `syscall.EPIPE`.

## Terminal Capability Detection: TTY vs. Pipe and `NO_COLOR`

```text
                      Output Mode Decision Tree (os.Stdout)
+-------------------------------------------------------------------------------+
| 1. Is os.Stdout a TTY (term.IsTerminal(int(out.Fd())))?                       |
|    - NO (Piped: mycli list | jq . / redirected to file):                      |
|      -> Disable colors, spinners, & $PAGER; emit raw TSV or NDJSON (json/v2)  |
|    - YES (Interactive Terminal):                                              |
|      -> Check NO_COLOR != "" or TERM == "dumb" (respects no-color.org spec)   |
|      -> Check CLICOLOR_FORCE != "" to allow forced color in CI logs           |
|      -> Query terminal dimensions (width, height := term.GetSize(fd))         |
|      -> If output lines > height - 2: route through $PAGER ("less -FIRX")     |
+-------------------------------------------------------------------------------+
```

## Bytes vs. Runes vs. Terminal Display Columns

| Cell String | `len(s)` (Bytes) | `utf8.RuneCountInString` | Actual Terminal Width |
|---|---|---|---|
| `"Running"` | 7 | 7 | **7 columns** |
| `"\x1b[32mRunning\x1b[0m"` | 16 | 16 | **7 columns** (9 invisible ANSI escape bytes) |
| `"🚀 Ready"` | 10 | 7 | **8 columns** (wide emoji = 2 columns) |
| `"👨‍👩‍👧"` (ZWJ emoji) | 18 | 5 | **2 columns** (1 grapheme cluster) |

Strip ANSI CSI/OSC-8 escape sequences first, then measure grapheme cluster display width with [rivo/uniseg](https://github.com/rivo/uniseg) (`uniseg.StringWidth`).

## Spawning `$PAGER` (`less -FIRX`) Safely

- **`-F` (`--quit-if-one-screen`):** Exits immediately if the entire output fits on one screen.
- **`-I` (`--ignore-case`):** Smart case-insensitive `/` search.
- **`-R` (`--RAW-CONTROL-CHARS`):** Passes ANSI color CSI sequences through cleanly.
- **`-X` (`--no-init`):** Prevents alternate-screen clearing when `-F` exits on short output.

> [!WARNING]
> Users often set `PAGER="less -R"` or `PAGER="bat --paging=always"` with embedded flags. Split `$PAGER` with `strings.Fields` before calling `exec.CommandContext`, and treat `errors.Is(err, syscall.EPIPE)` as a normal user-initiated early exit rather than an error.

## Complete Zero-Alloc Streaming Table and `$PAGER` Engine in Go 1.27

```go
package termstream

import (
	"bufio"
	"context"
	"errors"
	"fmt"
	"io"
	"os"
	"os/exec"
	"slices"
	"strings"
	"syscall"
	"time"

	"golang.org/x/term"
)

// OutputSession manages buffered streaming and optional $PAGER subprocess lifecycle.
type OutputSession struct {
	bw      *bufio.Writer
	pager   *exec.Cmd
	pipeW   io.WriteCloser
	isTTY   bool
	noColor bool
}

// OpenOutput inspects stdout, honors NO_COLOR, and optionally attaches a $PAGER
// subprocess (defaulting to "less -FIRX") when running interactively.
func OpenOutput(ctx context.Context, out *os.File, usePager bool) (*OutputSession, error) {
	isTTY := term.IsTerminal(int(out.Fd()))
	_, hasNoColor := os.LookupEnv("NO_COLOR")
	noColor := !isTTY || hasNoColor || os.Getenv("TERM") == "dumb"

	sess := &OutputSession{
		isTTY:   isTTY,
		noColor: noColor,
	}

	if !isTTY || !usePager {
		sess.bw = bufio.NewWriterSize(out, 32*1024)
		return sess, nil
	}

	pagerCmdStr := os.Getenv("PAGER")
	if pagerCmdStr == "" {
		pagerCmdStr = "less -FIRX"
	}
	parts := strings.Fields(pagerCmdStr)
	if len(parts) == 0 {
		sess.bw = bufio.NewWriterSize(out, 32*1024)
		return sess, nil
	}

	cmd := exec.CommandContext(ctx, parts[0], parts[1:]...)
	cmd.Stdout = out
	cmd.Stderr = os.Stderr
	cmd.Env = append(os.Environ(), "LESS=-FIRX")

	pipeW, err := cmd.StdinPipe()
	if err != nil {
		return nil, fmt.Errorf("create pager stdin pipe: %w", err)
	}
	if err := cmd.Start(); err != nil {
		_ = pipeW.Close()
		sess.bw = bufio.NewWriterSize(out, 32*1024)
		return sess, nil
	}

	sess.pager = cmd
	sess.pipeW = pipeW
	sess.bw = bufio.NewWriterSize(pipeW, 32*1024)
	return sess, nil
}

// Close flushes buffered bytes, closes the pager stdin pipe, and awaits pager exit.
func (s *OutputSession) Close() error {
	flushErr := s.bw.Flush()
	if errors.Is(flushErr, syscall.EPIPE) {
		flushErr = nil // User pressed 'q' in less early.
	}

	var closeErr, waitErr error
	if s.pipeW != nil {
		closeErr = s.pipeW.Close()
	}
	if s.pager != nil {
		waitErr = s.pager.Wait()
	}

	return errors.Join(flushErr, closeErr, waitErr)
}

type PodRow struct {
	Namespace string
	Name      string
	Status    string
	CreatedAt time.Time
}

// StreamTableRows renders rows into a reusable scratch buffer with zero per-row heap
// allocations using Go 1.24–1.27 encoding.TextAppender (time.Time.AppendText).
func (s *OutputSession) StreamTableRows(rows []PodRow) error {
	lineBuf := make([]byte, 0, 512)

	for _, row := range slices.All(rows) {
		lineBuf = lineBuf[:0]

		if !s.noColor && row.Status == "Running" {
			lineBuf = append(lineBuf, "\x1b[32m"...)
		}
		lineBuf = appendFixedCol(lineBuf, row.Namespace, 16)
		lineBuf = append(lineBuf, "  "...)
		lineBuf = appendFixedCol(lineBuf, row.Name, 28)
		lineBuf = append(lineBuf, "  "...)
		lineBuf = appendFixedCol(lineBuf, row.Status, 12)
		lineBuf = append(lineBuf, "  "...)

		var err error
		lineBuf, err = row.CreatedAt.AppendText(lineBuf)
		if err != nil {
			return err
		}

		if !s.noColor && row.Status == "Running" {
			lineBuf = append(lineBuf, "\x1b[0m"...)
		}
		lineBuf = append(lineBuf, '\n')

		if _, err := s.bw.Write(lineBuf); err != nil {
			if errors.Is(err, syscall.EPIPE) {
				return nil
			}
			return err
		}
	}
	return nil
}

//noalloc:root
func appendFixedCol(dst []byte, val string, width int) []byte {
	if len(val) >= width {
		return append(dst, val[:width]...)
	}
	dst = append(dst, val...)
	for i := len(val); i < width; i++ {
		dst = append(dst, ' ')
	}
	return dst
}
```

## ANSI Markdown and Structured Tables

- **[Glamour](https://github.com/charmbracelet/glamour):** Parses Markdown via `goldmark`, syntax-highlights fenced code via `chroma`, and word-wraps to terminal width while preserving ANSI styles. Construct `glamour.NewTermRenderer` once (or lazily via `sync.OnceValue`)—it takes ~3–5 ms per construction.
- **[Lip Gloss Table](https://github.com/charmbracelet/lipgloss):** Uses `StyleFunc` and `uniseg` grapheme widths to keep box borders aligned with nested ANSI colors and emojis.
- **Tooling verification:** Verify `pipeW.Close()` / `sess.Close()` on every branch with [gohawk](https://gohawk.dev/), enforce zero-allocation formatters with `noalloc`, and benchmark throughput gains with [benchstat](https://pkg.go.dev/golang.org/x/perf/cmd/benchstat).

## Related Chapters

- [306 Stdin, Files, and Pipes](306-stdin-pipes-files.md)
- [322 Terminal TTY Raw Mode, ANSI Rendering, and Bubble Tea Architecture](322-terminal-tty-raw-mode-bubble-tea.md)
- [983 I/O and bufio Composition](../07-stdlib/983-io-bufio-composition.md)
