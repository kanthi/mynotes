---
title: '331. Structuring CLI Applications with Cobra'
---

# Structuring CLI Applications with Cobra

As a CLI grows from a single command into subcommands like `check`, `monitor`, and `version`, **Cobra** provides flag inheritance, lifecycle hooks, GNU-style flag parsing (`pflag`), and automatic help generation.

## Module Syllabus (8 Lessons)

1. Introduction
2. Setting up a new structured logger
3. Adding `check` & `monitor` Commands
4. Adding Command Hooks (`PersistentPreRunE`, `PreRunE`, `PostRunE`)
5. Using Context to Manage Global Cancellation
6. Creating a `version` Command
7. Exercise
8. Exercise Solution

## Command Tree & Lifecycle Hooks

```text
urlmon (root: --verbose, --silent, --output)
  ├── check <url...>     (one-shot HTTP probe)
  ├── monitor <url...>   (periodic polling with --interval, canceled via Ctrl+C)
  └── version            (prints ldflags-injected version, commit, build date)
```

Cobra executes `PersistentPreRunE` on the root command before any child command's `RunE`, making it the ideal place to initialize the structured `slog.Logger` from persistent flags.

## Root Command with Context Cancellation & Version

```go
package cmd

import (
	"context"
	"fmt"
	"log/slog"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/spf13/cobra"
)

var (
	Version = "dev"
	Commit  = "none"
	Date    = "unknown"
)

func NewRootCmd() *cobra.Command {
	var (
		verbose bool
		logger  *slog.Logger
	)

	root := &cobra.Command{
		Use:   "urlmon",
		Short: "Check and monitor HTTP endpoints from the terminal",
		PersistentPreRunE: func(cmd *cobra.Command, _ []string) error {
			level := slog.LevelInfo
			if verbose {
				level = slog.LevelDebug
			}
			logger = slog.New(slog.NewTextHandler(cmd.ErrOrStderr(), &slog.HandlerOptions{Level: level}))
			return nil
		},
	}
	root.PersistentFlags().BoolVarP(&verbose, "verbose", "v", false, "enable debug logs")

	checkCmd := &cobra.Command{
		Use:   "check [urls...]",
		Short: "Run a one-shot check on one or more URLs",
		Args:  cobra.MinimumNArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			logger.Debug("running check", "count", len(args))
			return nil
		},
	}

	var interval time.Duration
	monitorCmd := &cobra.Command{
		Use:   "monitor [urls...]",
		Short: "Continuously monitor URLs until interrupted",
		Args:  cobra.MinimumNArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			ticker := time.NewTicker(interval)
			defer ticker.Stop()
			for {
				select {
				case <-cmd.Context().Done():
					logger.Info("monitoring stopped")
					return nil
				case <-ticker.C:
					logger.Debug("tick", "urls", len(args))
				}
			}
		},
	}
	monitorCmd.Flags().DurationVarP(&interval, "interval", "i", 5*time.Second, "polling interval")

	versionCmd := &cobra.Command{
		Use:   "version",
		Short: "Print build version metadata",
		Run: func(cmd *cobra.Command, _ []string) {
			fmt.Fprintf(cmd.OutOrStdout(), "urlmon %s (commit %s, built %s)\n", Version, Commit, Date)
		},
	}

	root.AddCommand(checkCmd, monitorCmd, versionCmd)
	return root
}

func Execute() error {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()
	return NewRootCmd().ExecuteContext(ctx)
}
```

## Related Chapters

- Previous: [330. URL Monitor CLI: Basic & Advanced Features with `flag`](330-url-monitor-flag-cli.md)
- Next: [332. Enhancing CLI Usability and Interactivity](332-cli-usability-and-interactivity.md)
- Foundations: [311. Cobra Basics](311-cobra-basics.md), [312. Cobra Advanced](312-cobra-advanced.md)
