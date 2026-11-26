---
title: '332. Enhancing CLI Usability and Interactivity'
---

# Enhancing CLI Usability and Interactivity

Great developer CLIs serve both **humans** (colored tables, spinners, live-updating dashboards, shell autocomplete) and **scripts** (clean `--output json` or plain text with color automatically disabled when piped).

## Module Syllabus (7 Lessons)

1. New `--output` flag (`json`, `text`, `table`)
2. Use spinner bar for a single URL check
3. Monitoring table with dynamic updates
4. Add color for up/down statuses
5. Adding Autocomplete (Bash, Zsh, Fish, PowerShell)
6. Exercise
7. Exercise Solution

## Supporting Multiple Output Formats

Decouple check results from rendering so a single `Result` slice can be formatted as JSON, plain text, or an aligned table.

```go
package output

import (
	"encoding/json"
	"fmt"
	"io"
	"text/tabwriter"
	"time"
)

type CheckResult struct {
	URL        string        `json:"url"`
	StatusCode int           `json:"status_code"`
	Up         bool          `json:"up"`
	Latency    time.Duration `json:"latency_ms"`
}

func Render(w io.Writer, format string, results []CheckResult, useColor bool) error {
	switch format {
	case "json":
		enc := json.NewEncoder(w)
		enc.SetIndent("", "  ")
		return enc.Encode(results)
	case "text":
		for _, r := range results {
			fmt.Fprintf(w, "%s %d (%s)\n", r.URL, r.StatusCode, r.Latency.Round(time.Millisecond))
		}
		return nil
	case "table":
		tw := tabwriter.NewWriter(w, 0, 4, 2, ' ', 0)
		fmt.Fprintln(tw, "URL\tSTATUS\tSTATE\tLATENCY")
		for _, r := range results {
			state := "DOWN"
			if r.Up {
				state = "UP"
			}
			if useColor {
				if r.Up {
					state = "\033[32mUP\033[0m"
				} else {
					state = "\033[31mDOWN\033[0m"
				}
			}
			fmt.Fprintf(tw, "%s\t%d\t%s\t%s\n", r.URL, r.StatusCode, state, r.Latency.Round(time.Millisecond))
		}
		return tw.Flush()
	default:
		return fmt.Errorf("unsupported output format %q (use json, text, or table)", format)
	}
}
```

## Spinners, Dynamic Tables, Color Hygiene & Autocomplete

- **Spinners & dynamic tables:** Use `charmbracelet/bubbles` (`spinner`, `table`) or `pterm`/`briandowns/spinner` only when `stdout` is an interactive TTY (`term.IsTerminal`). When piped or in CI, skip animations and emit clean lines.
- **Color hygiene:** Honor the `NO_COLOR` environment variable and disable ANSI escape sequences whenever output is redirected to a file or pipe.
- **Shell autocomplete:** Cobra generates completion scripts out of the box via `root.GenBashCompletion`, `GenZshCompletion`, `GenFishCompletion`, and `GenPowerShellCompletionWithDesc`, and supports dynamic flag value completion via `RegisterFlagCompletionFunc("output", ...)`.

## Related Chapters

- Previous: [331. Structuring CLI Applications with Cobra](331-url-monitor-cobra-structure.md)
- Next: [333. Testing CLI Applications](333-testing-cobra-and-http-clis.md)
- Foundations: [322. Terminal TTY, ANSI & Bubble Tea](322-terminal-tty-raw-mode-bubble-tea.md), [323. Command Trees & `__complete` Engines](323-command-trees-kong-cobra-completions.md)
