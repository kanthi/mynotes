---
title: '333. Testing CLI Applications'
---

# Testing CLI Applications

CLIs that construct fresh Cobra command trees per test and inject `*http.Client` and `io.Writer` dependencies can be tested in milliseconds—no live internet calls and no brittle subprocess execution required.

## Module Syllabus (6 Lessons)

1. Introduction to Testing CLI Applications
2. Writing Unit Tests
3. Mocking HTTP Calls (`httptest.NewServer` / custom `http.RoundTripper`)
4. Using `ExecuteC` to test Cobra Commands
5. Exercise: Dynamic input testing for command parsing
6. Exercise Solution

## Mocking HTTP Calls & Testing Cobra with `ExecuteC`

`cmd.ExecuteC()` returns both the executed subcommand and any error, letting table-driven tests verify flag parsing, subcommand routing, captured `stdout`/`stderr`, and mocked HTTP responses.

```go
package cmd_test

import (
	"bytes"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"

	"github.com/spf13/cobra"
)

func TestCheckCommandWithExecuteC(t *testing.T) {
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		switch r.URL.Path {
		case "/ok":
			w.WriteHeader(http.StatusOK)
		default:
			w.WriteHeader(http.StatusServiceUnavailable)
		}
	}))
	defer srv.Close()

	tests := []struct {
		name        string
		args        []string
		wantSubcmd  string
		wantOut     string
		wantErr     bool
	}{
		{
			name:       "healthy endpoint",
			args:       []string{"check", srv.URL + "/ok"},
			wantSubcmd: "check",
			wantOut:    "200",
		},
		{
			name:       "missing url argument",
			args:       []string{"check"},
			wantSubcmd: "check",
			wantErr:    true,
		},
	}

	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			var stdout, stderr bytes.Buffer
			root := newTestRootCmd(srv.Client())
			root.SetOut(&stdout)
			root.SetErr(&stderr)
			root.SetArgs(tc.args)

			executed, err := root.ExecuteC()
			if (err != nil) != tc.wantErr {
				t.Fatalf("ExecuteC() err = %v, wantErr = %v", err, tc.wantErr)
			}
			if executed != nil && executed.Name() != tc.wantSubcmd {
				t.Fatalf("executed = %q, want %q", executed.Name(), tc.wantSubcmd)
			}
			if tc.wantOut != "" && !strings.Contains(stdout.String(), tc.wantOut) {
				t.Fatalf("stdout = %q, want substring %q", stdout.String(), tc.wantOut)
			}
		})
	}
}

func newTestRootCmd(client *http.Client) *cobra.Command {
	root := &cobra.Command{Use: "urlmon", SilenceUsage: true, SilenceErrors: true}
	root.AddCommand(&cobra.Command{
		Use:  "check [url]",
		Args: cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			resp, err := client.Get(args[0])
			if err != nil {
				return err
			}
			defer resp.Body.Close()
			cmd.Printf("%s -> %d\n", args[0], resp.StatusCode)
			return nil
		},
	})
	return root
}
```

## Common Pitfalls

- **Reusing global Cobra `rootCmd` across tests:** Persistent flag values leak between test cases! Always instantiate a fresh command via a constructor like `NewRootCmd(...)` inside each subtest.
- **Writing directly to `os.Stdout` inside `RunE`:** Always write to `cmd.OutOrStdout()` and `cmd.ErrOrStderr()` so `SetOut` / `SetErr` capture everything cleanly.

## Related Chapters

- Previous: [332. Enhancing CLI Usability and Interactivity](332-cli-usability-and-interactivity.md)
- Next: [334. Error Handling & Logging in CLIs](334-cli-error-handling-and-log-rotation.md)
- Foundations: [309. Testing CLIs](309-testing-cli.md), [41. `httptest` and API Testing](../06-testing/41-httptest-and-api-testing.md)
