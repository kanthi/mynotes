---
title: Debugging by Eye & Reading Panic Traces
---

# Debugging by Eye & Reading Panic Traces

Debugging is the discipline of forming a hypothesis about why observed behavior diverges from expected behavior, narrowing the search space, and proving the root cause with evidence. Reaching for an interactive debugger or profiler is powerful, but **debugging by eye** and **reading panic traces** resolve most everyday bugs in seconds.

## The Debugging Escalation Ladder

```text
1. Debugging by Eye      -> Read code, types, and error messages; spot common Go traps
2. Structured Logging    -> Trace control flow & state with slog (local or ELK/Kibana)
3. Interactive Debugger  -> Pause execution with Delve (VSCode, GoLand, Docker)
4. Production Signals    -> Metrics (Prometheus), Distributed Tracing (OTel), Profiling (pprof / gotraceui)
```

Start at the lightest level that can answer your question.

## Common Go Bug Patterns to Spot by Eye

### 1. Variable Shadowing (`:=` Inside Inner Scopes)

```go
var err error
if needsConfig {
	cfg, err := loadConfig() // Shadows outer err!
	if err != nil {
		return err
	}
	use(cfg)
}
// Outer err is still nil here.
```

### 2. Nil Pointer & Nil Interface Traps

- Dereferencing a pointer returned alongside an ignored or early-checked error.
- Returning a typed `nil` pointer (`var p *MyErr = nil`) as an `error` interface—the interface value itself is **non-nil** (`err != nil` is `true`).

### 3. Nil Map Writes & Slice Aliasing

- Reading from a `nil` map returns the zero value safely, but writing to a `nil` map panics (`assignment to entry in nil map`).
- Slicing `sub := base[:2]` shares `base`'s backing array; `append(sub, x)` overwrites `base[2]` unless you use a full slice expression `base[:2:2]`.

### 4. Unclosed Resources & Deferred Error Loss

- Forgetting `defer resp.Body.Close()` after `http.Get` leaks connections and file descriptors.
- Calling `defer f.Close()` on a writable file without checking the close error can silently drop buffered write failures.

## Pair Programming & Rubber-Duck Debugging

When staring at your own code stops yielding new hypotheses:
- **Rubber-duck debugging:** Explain the function line by line out loud—stating *what the code actually does*, not what you *intended* it to do.
- **Pair programming:** A second pair of eyes notices assumptions you treat as invisible (wrong variable name, inverted boolean condition, stale build/config).

## Guided Exercise: Spot the Three Bugs by Eye

Before running the snippet below, find all three bugs:

```go
package main

import "fmt"

type User struct {
	Name string
}

func findUser(id int) *User {
	if id == 1 {
		return &User{Name: "Ada"}
	}
	return nil
}

func main() {
	var counts map[string]int
	u := findUser(2)
	fmt.Println(u.Name)        // Bug 1
	counts["visits"]++         // Bug 2

	s := []int{1, 2, 3}
	sub := s[:2]
	sub = append(sub, 99)      // Bug 3
	fmt.Println("s:", s)
}
```

### Solution

1. **Bug 1:** `findUser(2)` returns `nil`; `u.Name` panics with `invalid memory address or nil pointer dereference`. Guard with `if u == nil`.
2. **Bug 2:** `counts` is a `nil` map; `counts["visits"]++` panics. Initialize with `counts := make(map[string]int)`.
3. **Bug 3:** `sub := s[:2]` has capacity 3, so `append(sub, 99)` mutates `s[2]` to `99` in place. Use `s[:2:2]` if you want a separate backing array on append.

## Reading Panic Stack Traces

When a Go program panics, the runtime prints a stack trace from the innermost frame outward:

```text
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x8 pc=0x49a120]

goroutine 1 [running]:
main.processOrder(0x0)
        /app/orders.go:42 +0x20
main.main()
        /app/main.go:18 +0x54
```

How to read it fast:
1. **Top line (`panic: ...`):** Tells you the failure kind (`nil pointer`, `index out of range`, `assignment to entry in nil map`).
2. **Goroutine header (`goroutine 1 [running]:`):** Identifies which goroutine panicked (and `created by ...` at the bottom shows which line spawned it).
3. **First user-code frame (`/app/orders.go:42`):** Start here, not inside `runtime.*` frames. Notice the argument `0x0` passed to `processOrder(0x0)`—it immediately confirms a `nil` pointer was passed from `main.go:18`.

> [!TIP]
> Set `GOTRACEBACK=all` (or `crash` for core dumps) when you need stack traces for **all** live goroutines during a crash, not just the panicking one.

## Related Chapters

- [35 Panic and Recover](../05-errors/35-panic-recover.md)
- [10 Interactive Debugging: Delve, VSCode, GoLand & Docker](10-interactive-debugging-delve-ide-docker.md)
- [250 Stacktraces & Binary Metadata](../25-go-deep-dives/250-stacktraces-runtime-metadata.md)
