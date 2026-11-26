---
title: 'Project: Optimize a Hot HTTP Endpoint'
---

# Project: Optimize a Hot HTTP Endpoint

Follow an evidence-based performance workflow: establish a baseline benchmark, capture CPU and allocation profiles with `pprof`, eliminate avoidable heap allocations, and prove the improvement quantitatively with `benchstat`.

## The Slow Baseline Handler

This handler compiles a regex on every request, allocates intermediate strings with `+`, and uses `json.Marshal` into a fresh `[]byte` slice:

```go
package hotpath

import (
	"encoding/json"
	"net/http"
	"regexp"
)

func SlowHandler(w http.ResponseWriter, r *http.Request) {
	// Bug 1: recompiles regexp on every single request.
	re := regexp.MustCompile(`^[a-z0-9-]+$`)
	id := r.URL.Query().Get("id")
	if !re.MatchString(id) {
		http.Error(w, "invalid id", http.StatusBadRequest)
		return
	}

	// Bug 2: allocates a map and intermediate buffer per request.
	payload := map[string]any{
		"id":     id,
		"status": "ok",
		"tag":    "item-" + id,
	}
	b, _ := json.Marshal(payload)
	w.Header().Set("Content-Type", "application/json")
	_, _ = w.Write(b)
}
```

## The Optimized Handler

Hoist the regex to package init (or replace with a zero-alloc ASCII loop), use a typed struct instead of `map[string]any`, and reuse a pooled `bytes.Buffer`:

```go
package hotpath

import (
	"bytes"
	"encoding/json"
	"net/http"
	"sync"
)

type okResponse struct {
	ID     string `json:"id"`
	Status string `json:"status"`
	Tag    string `json:"tag"`
}

var bufPool = sync.Pool{
	New: func() any { return bytes.NewBuffer(make([]byte, 0, 128)) },
}

func isValidID(s string) bool {
	if len(s) == 0 {
		return false
	}
	for i := 0; i < len(s); i++ {
		c := s[i]
		if !((c >= 'a' && c <= 'z') || (c >= '0' && c <= '9') || c == '-') {
			return false
		}
	}
	return true
}

func FastHandler(w http.ResponseWriter, r *http.Request) {
	id := r.URL.Query().Get("id")
	if !isValidID(id) {
		http.Error(w, "invalid id", http.StatusBadRequest)
		return
	}

	buf := bufPool.Get().(*bytes.Buffer)
	buf.Reset()
	defer bufPool.Put(buf)

	_ = json.NewEncoder(buf).Encode(okResponse{
		ID:     id,
		Status: "ok",
		Tag:    "item-" + id,
	})

	w.Header().Set("Content-Type", "application/json")
	_, _ = w.Write(buf.Bytes())
}
```

## Measuring and Proving the Gain

```bash
# 1. Capture baseline
go test -bench=BenchmarkHandler -benchmem -count=10 -cpuprofile=cpu.out -memprofile=mem.out > old.txt

# 2. Inspect top allocators
go tool pprof -top -alloc_space mem.out

# 3. Capture optimized run and compare with benchstat
go test -bench=BenchmarkHandler -benchmem -count=10 > new.txt
benchstat old.txt new.txt
```

## Acceptance Criteria

- Allocation count (`allocs/op`) drops by > 70% between `SlowHandler` and `FastHandler`.
- `go test -race ./...` passes with zero races on `bufPool` reuse.
- `benchstat` confirms statistically significant latency reduction (`p < 0.05`).
