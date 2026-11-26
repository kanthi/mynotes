---
title: 'Project: Text and Log Tools (textkit)'
---

# Project: Text and Log Tools (`textkit`)

Build a composable Unix-style streaming CLI (`textkit`) that reads from `stdin` or files using `bufio.Scanner` with an enlarged token buffer, filters structured JSON logs by level/field, and emits summary counts without loading whole files into memory.

## Subcommands & Pipeline Usage

```bash
# Filter ERROR/WARN lines from a JSON log stream and count by service
cat app.log | textkit filter --level=error | textkit count --field=service
```

## Streaming JSON Log Filter & Aggregator

```go
package main

import (
	"bufio"
	"encoding/json"
	"flag"
	"fmt"
	"io"
	"os"
	"sort"
	"strings"
)

func filterLogs(r io.Reader, w io.Writer, minLevel string) error {
	sc := bufio.NewScanner(r)
	sc.Buffer(make([]byte, 64*1024), 4*1024*1024) // handle up to 4 MiB lines

	want := strings.ToLower(minLevel)
	for sc.Scan() {
		line := sc.Bytes()
		var entry map[string]any
		if err := json.Unmarshal(line, &entry); err != nil {
			continue // skip non-JSON lines cleanly
		}
		lvl, _ := entry["level"].(string)
		if want == "" || strings.EqualFold(lvl, want) {
			_, _ = w.Write(line)
			_, _ = w.Write([]byte("\n"))
		}
	}
	return sc.Err()
}

func countByField(r io.Reader, w io.Writer, field string) error {
	sc := bufio.NewScanner(r)
	sc.Buffer(make([]byte, 64*1024), 4*1024*1024)
	counts := make(map[string]int)

	for sc.Scan() {
		var entry map[string]any
		if err := json.Unmarshal(sc.Bytes(), &entry); err != nil {
			continue
		}
		key := fmt.Sprint(entry[field])
		counts[key]++
	}
	if err := sc.Err(); err != nil {
		return err
	}

	keys := make([]string, 0, len(counts))
	for k := range counts {
		keys = append(keys, k)
	}
	sort.Strings(keys)
	for _, k := range keys {
		fmt.Fprintf(w, "%6d  %s\n", counts[k], k)
	}
	return nil
}

func main() {
	if len(os.Args) < 2 {
		fmt.Fprintln(os.Stderr, "usage: textkit <filter|count> [flags]")
		os.Exit(2)
	}
	switch os.Args[1] {
	case "filter":
		fs := flag.NewFlagSet("filter", flag.ExitOnError)
		level := fs.String("level", "error", "log level to match")
		_ = fs.Parse(os.Args[2:])
		if err := filterLogs(os.Stdin, os.Stdout, *level); err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
	case "count":
		fs := flag.NewFlagSet("count", flag.ExitOnError)
		field := fs.String("field", "service", "JSON field to group by")
		_ = fs.Parse(os.Args[2:])
		if err := countByField(os.Stdin, os.Stdout, *field); err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
	default:
		fmt.Fprintf(os.Stderr, "unknown subcommand %q\n", os.Args[1])
		os.Exit(2)
	}
}
```

## Verification

- Feed malformed lines mixed with valid JSON into `filterLogs` and confirm it never panics and exits `0`.
- Test with a 1 MiB single JSON line to verify `sc.Buffer` prevents `bufio.ErrTooLong`.
