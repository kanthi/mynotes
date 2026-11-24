---
title: Command Trees, Zero-Alloc Flag Parsing, and Shell Completion Engines
---

# Command Trees, Zero-Alloc Flag Parsing, and Shell Completion Engines

The backbone of every developer tool (`kubectl`, `gh`, `docker`, `terraform`, `go`) is its command tree dispatcher, flag parser, and dynamic shell completion engine. A production-grade CLI must parse nested subcommands, validate strongly typed flags (`netip.Addr`, `slog.Level`, `url.URL`), inject dependencies cleanly for parallel unit testing, and respond to shell `<Tab>` completion queries (`__complete`) in under 10 milliseconds.

## Architectural Comparison: `flag.FlagSet` vs. Cobra vs. Kong

```text
1. Pointer Command Tree (spf13/cobra)              2. Struct-Tag Grammar Tree (alecthomas/kong)
+-----------------------------------------+        +-----------------------------------------+
|          *cobra.Command ("root")        |        | type CLI struct {                       |
|            /                 \          |        |   Cluster struct {                      |
|   *Command ("cluster")   *Command       |        |     List ListCmd `cmd:""`               |
|          |                              |        |   } `cmd:""`                            |
|   *Command ("list")                     |        | }                                       |
| Explicit node pointers & pflag.FlagSet  |        | Single struct hierarchy; dispatches     |
| Scales to 200+ commands & plugins       |        | directly to selected leaf .Run(deps)    |
+-----------------------------------------+        +-----------------------------------------+
```

| Dimension | Standard Library `flag.FlagSet` | [Kong](https://github.com/alecthomas/kong) | [Cobra](https://github.com/spf13/cobra) + `pflag` |
|---|---|---|---|
| Flag syntax | Plan 9 style (`-flag=x`, stops at first positional arg) | Full POSIX/GNU (`-v`, `--verbose`, `-abc`, env vars) | Full POSIX/GNU via `spf13/pflag` (`--flag`, `-f`, interleaved args) |
| Tree definition | Manual `switch args[0]` routing per `FlagSet` | Declarative Go struct tags (`cmd:""`, `arg:""`, `enum:"a,b"`) | Explicit `*cobra.Command` tree construction |
| Reflection cost | Near-zero (`flag.Value` interface dispatch) | Reflects over a single root struct once at startup (~50 µs) | Zero struct reflection; scales to massive plugin trees |
| Best use case | Micro-tools, zero-dependency daemons, `-vettool` binaries | Medium-to-large strongly typed CLIs wanting minimal boilerplate | Large multi-team CLIs needing dynamic `__complete` and man pages |

## Standard Library `flag` Internals: `flag.TextVar` and the Positional Stop Rule

Inside `src/flag/flag.go`, `FlagSet.Parse(arguments []string)` has two subtle behaviors:

1. **Parsing stops at the first non-flag argument:** Running `mycli -v deploy --force` parses `-v`, sees the positional word `deploy`, and immediately stops parsing—leaving `--force` unparsed inside `fs.Args()`. POSIX parsers (`pflag` in Cobra and `kong`) interleave flags and positional arguments unless `--` is encountered.
2. **Zero-boilerplate typed flags via `flag.TextVar`:** Any type implementing `encoding.TextUnmarshaler` and `encoding.TextMarshaler`—including `netip.Addr`, `netip.AddrPort`, `time.Time`, `slog.Level`, and `big.Int`—binds directly without writing a custom `flag.Value` wrapper:

```go
package main

import (
	"flag"
	"fmt"
	"io"
	"log/slog"
	"net/netip"
)

type DaemonConfig struct {
	BindAddr netip.AddrPort
	LogLevel slog.Level
}

// ParseDaemonFlags uses an isolated FlagSet (never global flag.CommandLine)
// and binds rich standard-library types directly via flag.TextVar.
func ParseDaemonFlags(args []string, errOut io.Writer) (DaemonConfig, error) {
	var cfg DaemonConfig

	fs := flag.NewFlagSet("daemon", flag.ContinueOnError)
	fs.SetOutput(errOut)

	defaultAddr := netip.MustParseAddrPort("127.0.0.1:8080")
	fs.TextVar(&cfg.BindAddr, "listen", defaultAddr, "TCP listen address:port")
	fs.TextVar(&cfg.LogLevel, "log-level", slog.LevelInfo, "Minimum log level (DEBUG, INFO, WARN, ERROR)")

	if err := fs.Parse(args); err != nil {
		return DaemonConfig{}, err
	}
	return cfg, nil
}

func main() {
	cfg, _ := ParseDaemonFlags([]string{"-listen=0.0.0.0:9090", "-log-level=DEBUG"}, io.Discard)
	fmt.Printf("Addr=%s Level=%s\n", cfg.BindAddr, cfg.LogLevel)
}
```

## Eliminating Two Production Cobra Traps

### Trap 1: Global `func init()` Command Scaffolding

Placing `var rootCmd = &cobra.Command{...}` in package globals and wiring subcommands in `func init()` causes three problems:

- **Cold-start penalty:** Every subcommand struct and flag set allocates during `runtime.main` `init()`, even for `mycli version`.
- **Untestable global state:** Parallel tests (`t.Parallel()`) mutate the same global flags and trigger data races.
- **No dependency injection:** Subcommands cannot accept injected `io.Reader`, `io.Writer`, `*sql.DB`, or mock HTTP clients cleanly.

**Fix (Constructor Pattern):** Define every command as a constructor (`NewDeployCmd(deps *Dependencies) *cobra.Command`) returning a fresh struct-scoped command tree.

### Trap 2: The `PersistentPostRunE` Resource Leak

Look at how `(*cobra.Command).ExecuteC()` runs lifecycle hooks:

```text
1. PersistentPreRunE(cmd, args)   <-- Opens DB pool / telemetry trace file
2. PreRunE(cmd, args)
3. RunE(cmd, args)                <-- Returns an error (validation failed)
4. PostRunE(cmd, args)            <-- SKIPPED ON ERROR
5. PersistentPostRunE(cmd, args)  <-- SKIPPED ON ERROR (Resource LEAKED)
```

> [!CAUTION]
> If `RunE` (or `PreRunE`) returns a non-nil error, Cobra **never calls** `PostRunE` or `PersistentPostRunE`. Any file, gRPC connection, or database handle opened in `PersistentPreRunE` and closed in `PersistentPostRunE` leaks on every failed command invocation. Attach cleanups to a `defer` around `rootCmd.ExecuteContext(ctx)` instead.

## Dynamic Shell Completions (`__complete` Protocol)

When a user presses `<Tab>` after `mycli deploy --region us-`, the shell completion shim invokes your binary's hidden `__complete` subcommand:

```text
User Terminal (Zsh / Bash / Fish)                   Your Go Binary (mycli)
+-----------------------------------------+         +-----------------------------------------+
| User types:                             |         |                                         |
| $ mycli deploy --region us-<TAB>        |         |                                         |
|                                         |         |                                         |
| Shell completion shim intercepts <TAB>  |         |                                         |
| and executes hidden command:            |         |                                         |
|   mycli __complete deploy --region "us-"| ======> | 1. Walks command tree to 'deploy'       |
|                                         |         | 2. Sees incomplete flag '--region'      |
|                                         |         | 3. Invokes FlagCompletionFunc("us-")    |
|                                         | <====== | 4. Writes candidates to stdout:         |
| Parses candidates & directive ":4"      |         |    us-east-1\tVirginia                  |
| (4 = ShellCompDirectiveNoFileComp)      |         |    us-west-2\tOregon                    |
| Renders interactive tab menu            |         |    :4                                   |
+-----------------------------------------+         +-----------------------------------------+
```

| Directive Constant | Bit Value | Meaning to the Shell (Bash / Zsh / Fish) |
|---|---|---|
| `ShellCompDirectiveDefault` | `:0` | Fall back to standard filename completion if no candidates returned. |
| `ShellCompDirectiveError` | `:1` | An error occurred during completion; ignore output. |
| `ShellCompDirectiveNoSpace` | `:2` | Do not append a trailing space after inserting (useful for `s3://bucket/` prefixes). |
| `ShellCompDirectiveNoFileComp` | `:4` | Never fall back to local disk filenames (critical for virtual IDs like pod names or regions). |

## Complete Zero-Global CLI Tree and `__complete` Engine

```go
package clitree

import (
	"context"
	"flag"
	"fmt"
	"io"
	"strings"
)

// CompDirective controls shell completion behavior after '__complete' returns.
type CompDirective uint8

const (
	CompDefault    CompDirective = 0
	CompError      CompDirective = 1 << 0
	CompNoSpace    CompDirective = 1 << 1
	CompNoFileComp CompDirective = 1 << 2
)

// IOStreams decouples stdin/stdout/stderr for parallel testing without os.Stdout globals.
type IOStreams struct {
	In     io.Reader
	Out    io.Writer
	ErrOut io.Writer
}

// Command represents a node in a zero-global, dependency-injected CLI tree.
type Command struct {
	Name         string
	Summary      string
	SetupFlags   func(fs *flag.FlagSet)
	CompleteFunc func(ctx context.Context, toComplete string) ([]string, CompDirective)
	Run          func(ctx context.Context, streams IOStreams, args []string) error
	Subcommands  map[string]*Command
}

// Execute dispatches either normal execution or the hidden '__complete' protocol.
func (c *Command) Execute(ctx context.Context, streams IOStreams, args []string) error {
	if len(args) > 0 && args[0] == "__complete" {
		return c.executeCompletion(ctx, streams.Out, args[1:])
	}
	return c.executeNode(ctx, streams, args)
}

func (c *Command) executeNode(ctx context.Context, streams IOStreams, args []string) error {
	if len(args) > 0 && c.Subcommands != nil {
		if sub, ok := c.Subcommands[args[0]]; ok {
			return sub.executeNode(ctx, streams, args[1:])
		}
	}

	fs := flag.NewFlagSet(c.Name, flag.ContinueOnError)
	fs.SetOutput(streams.ErrOut)
	if c.SetupFlags != nil {
		c.SetupFlags(fs)
	}
	if err := fs.Parse(args); err != nil {
		return err
	}

	if c.Run == nil {
		return fmt.Errorf("%s: subcommand required", c.Name)
	}
	return c.Run(ctx, streams, fs.Args())
}

func (c *Command) executeCompletion(ctx context.Context, out io.Writer, args []string) error {
	if len(args) == 0 {
		fmt.Fprintf(out, ":%d\n", CompNoFileComp)
		return nil
	}

	// Final element is always the partial word currently under the user's cursor.
	toComplete := args[len(args)-1]
	pathArgs := args[:len(args)-1]

	curr := c
	for _, token := range pathArgs {
		if curr.Subcommands != nil {
			if sub, ok := curr.Subcommands[token]; ok {
				curr = sub
			}
		}
	}

	if len(curr.Subcommands) > 0 {
		for name, sub := range curr.Subcommands {
			if strings.HasPrefix(name, toComplete) {
				fmt.Fprintf(out, "%s\t%s\n", name, sub.Summary)
			}
		}
		fmt.Fprintf(out, ":%d\n", CompNoFileComp)
		return nil
	}

	if curr.CompleteFunc != nil {
		candidates, dir := curr.CompleteFunc(ctx, toComplete)
		for _, cand := range candidates {
			if strings.HasPrefix(cand, toComplete) {
				fmt.Fprintln(out, cand)
			}
		}
		fmt.Fprintf(out, ":%d\n", dir)
		return nil
	}

	fmt.Fprintf(out, ":%d\n", CompDefault)
	return nil
}
```

## Production Takeaways and Tooling

- **Enforce a 50ms timeout on dynamic `__complete` network calls:** Wrap completion contexts in `context.WithTimeout(ctx, 50*time.Millisecond)` so an unreachable cluster never freezes the user's `<Tab>` key.
- **Reserve `os.Stdout` strictly for data and `__complete` candidates:** Route all diagnostic logs, spinners, and warnings to `os.Stderr` so `mycli list --json | jq .` and shell completion scripts never break.
- **Static leak verification:** Run [gohawk](https://gohawk.dev/) (`go tool gohawk ./...`) to catch `PersistentPostRunE` resource leaks on error return paths, and consider [Carapace](https://github.com/carapace-sh/carapace) when you need rich completions across 10+ shells (including Nushell and Elvish).

## Related Chapters

- [303 Subcommands with FlagSet](303-subcommands-flagset.md)
- [311 Cobra Basics](311-cobra-basics.md)
- [312 Cobra Advanced](312-cobra-advanced.md)
- [314 Other CLI Frameworks](314-other-cli-frameworks.md)
