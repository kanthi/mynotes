---
title: '330. URL Monitor CLI: Basic & Advanced Features with flag'
---

# URL Monitor CLI: Basic & Advanced Features with `flag`

Before reaching for a CLI framework, you can build a solid, testable URL checker using only Go's standard library (`flag`, `log/slog`, `net/http`, `os`).

## Module Syllabus (9 Lessons)

1. Introduction
2. Creating a Simple CLI
3. Adding a Logger
4. Adding `silent` & `verbose` flags
5. Intro to `flag.Parse()`
6. Handling a list of URLs
7. Customizing help output
8. Exercise
9. Exercise Walkthrough

## Implementing `flag.Parse()`, Verbosity, URL Lists & Custom Usage

Use a custom `flag.FlagSet` instead of global `flag.CommandLine` so the CLI remains easy to test, and customize `fs.Usage` to print clear examples and flag descriptions.

```go
package main

import (
	"flag"
	"fmt"
	"io"
	"log/slog"
	"net/http"
	"os"
	"strings"
	"time"
)

// urlList implements flag.Value so -url can be passed multiple times or comma-separated.
type urlList []string

func (u *urlList) String() string { return strings.Join(*u, ",") }
func (u *urlList) Set(val string) error {
	for _, part := range strings.Split(val, ",") {
		if trimmed := strings.TrimSpace(part); trimmed != "" {
			*u = append(*u, trimmed)
		}
	}
	return nil
}

func run(args []string, stdout, stderr io.Writer) error {
	fs := flag.NewFlagSet("urlmon", flag.ContinueOnError)
	fs.SetOutput(stderr)

	var urls urlList
	silent := fs.Bool("silent", false, "suppress non-error log output")
	verbose := fs.Bool("verbose", false, "enable debug-level logging")
	timeout := fs.Duration("timeout", 5*time.Second, "per-request HTTP timeout")
	fs.Var(&urls, "url", "URL to check (repeatable or comma-separated)")

	fs.Usage = func() {
		fmt.Fprintf(stderr, "Usage: urlmon [flags] [url ...]\n\nExamples:\n  urlmon -verbose https://example.com\n  urlmon -url https://a.dev,https://b.dev\n\nFlags:\n")
		fs.PrintDefaults()
	}

	if err := fs.Parse(args); err != nil {
		return err
	}
	// Accept positional URL arguments in addition to -url flags.
	urls = append(urls, fs.Args()...)
	if len(urls) == 0 {
		fs.Usage()
		return fmt.Errorf("at least one URL is required")
	}

	level := slog.LevelInfo
	if *verbose {
		level = slog.LevelDebug
	}
	logOut := stderr
	if *silent {
		logOut = io.Discard
	}
	logger := slog.New(slog.NewTextHandler(logOut, &slog.HandlerOptions{Level: level}))

	client := &http.Client{Timeout: *timeout}
	for _, u := range urls {
		logger.Debug("checking url", "url", u)
		resp, err := client.Get(u)
		if err != nil {
			logger.Error("request failed", "url", u, "err", err)
			continue
		}
		resp.Body.Close()
		fmt.Fprintf(stdout, "%s -> %d\n", u, resp.StatusCode)
	}
	return nil
}

func main() {
	if err := run(os.Args[1:], os.Stdout, os.Stderr); err != nil {
		os.Exit(1)
	}
}
```

## Key Takeaways

- **`flag.Parse()` stops at the first non-flag argument:** In Go's `flag` package, flags must appear *before* positional URLs (`urlmon -verbose https://example.com`, not `urlmon https://example.com -verbose`).
- **Keep logs on `stderr` and data on `stdout`:** So users can pipe `urlmon` output into `jq` or `grep` without log lines polluting the stream.

## Related Chapters

- Next: [331. Structuring CLI Applications with Cobra](331-url-monitor-cobra-structure.md)
- Foundations: [302. `flag` Package Deep Dive](302-flag-package-deep.md), [304. Exit Codes, Stdout & Stderr](304-exit-codes-stdout-stderr.md)
