---
title: '334. Error Handling & Logging in CLIs'
---

# Error Handling & Logging in CLIs

Production CLIs need **domain-specific error types** that map cleanly to exit codes, **structured rotating log files** for long-running monitors, and **graceful user-facing error messages** that never dump raw stack traces on expected user mistakes.

## Module Syllabus (6 Lessons)

1. Introduction
2. Creating Custom Errors
3. Advanced logging techniques (log rotation, multi-sink output)
4. Graceful errors & interactivity
5. Exercise
6. Exercise Solution

## Custom Errors with Exit Codes

Attach an exit code and user-friendly hint to custom error types so `main()` can translate failures cleanly via `errors.As`.

```go
package clierr

import (
	"errors"
	"fmt"
)

type ExitError struct {
	Code int
	Hint string
	Err  error
}

func (e *ExitError) Error() string { return e.Err.Error() }
func (e *ExitError) Unwrap() error { return e.Err }

func ExitCode(err error) int {
	if err == nil {
		return 0
	}
	var exitErr *ExitError
	if errors.As(err, &exitErr) && exitErr.Code != 0 {
		return exitErr.Code
	}
	return 1
}

func FormatForUser(err error) string {
	var exitErr *ExitError
	if errors.As(err, &exitErr) && exitErr.Hint != "" {
		return fmt.Sprintf("Error: %s\nHint:  %s\n", exitErr.Err, exitErr.Hint)
	}
	return fmt.Sprintf("Error: %s\n", err)
}
```

## Log Rotation for Long-Running Monitor CLIs

When `urlmon monitor` runs for days in the background, writing unbounded logs fills the disk. Combine `log/slog` with a rotating writer such as `gopkg.in/natefinch/lumberjack.v2`:

```go
rotator := &lumberjack.Logger{
	Filename:   "/var/log/urlmon/monitor.log",
	MaxSize:    50, // megabytes before rotation
	MaxBackups: 5,  // old files to retain
	MaxAge:     14, // days
	Compress:   true,
}
logger := slog.New(slog.NewJSONHandler(rotator, &slog.HandlerOptions{
	Level: slog.LevelInfo,
}))
```

## Graceful Errors & Interactivity

- Set `SilenceUsage: true` and `SilenceErrors: true` on Cobra's root command so runtime network errors do not flood the terminal with the entire `--help` text.
- Stop active spinners and restore terminal cursor state (`defer spinner.Stop()`) before printing an error message.

## Related Chapters

- Previous: [333. Testing CLI Applications](333-testing-cobra-and-http-clis.md)
- Next: [335. Distributing Your CLI](335-distributing-go-clis.md)
- Foundations: [34. Error Wrapping](../05-errors/34-error-wrapping.md), [161. Structured Logging](../19-observability-sre/161-structured-logging-correlation.md)
