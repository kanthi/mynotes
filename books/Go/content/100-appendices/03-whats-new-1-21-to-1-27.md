---
title: "What's New: Go 1.21 – Go 1.27.4"
---

# What's New: Go 1.21 – Go 1.27.4

This appendix is a concise release-by-release reference covering **Go 1.21 through Go 1.27.4** (latest patch release: November 10, 2026; [Go Release History](https://go.dev/doc/devel/release)). Every example compiles and runs on Go 1.27.4+.

## Go 1.21

| Feature | Package / Layer | What It Does |
|---|---|---|
| `min` / `max` / `clear` | Language builtins | Generic minimum/maximum builtins; `clear` zeroes slice elements or deletes all map keys. |
| `slices`, `maps`, `cmp` | Standard library | Generic collection helpers (`slices.SortFunc`, `cmp.Compare`, `cmp.Or`). |
| `log/slog` | `log/slog` | Structured leveled logging with JSON/Text handlers. |
| `sync.OnceFunc` / `OnceValue` | `sync` | Lazy one-time initialization helpers without boilerplate. |

## Go 1.22

| Feature | Package / Layer | What It Does |
|---|---|---|
| `for i := range n` | Language | Integer range loop (`0` to `n-1`). |
| Per-iteration loop variables | Language | Eliminates classic goroutine loop-variable capture bugs. |
| Enhanced `http.ServeMux` | `net/http` | Method + path wildcard patterns (`"GET /tickets/{id}"`) and `r.PathValue("id")`. |
| `math/rand/v2`, `reflect.TypeFor[T]` | Standard library | Modern PCG/ChaCha8 PRNGs and zero-alloc compile-time `reflect.Type` lookup. |

## Go 1.23

| Feature | Package / Layer | What It Does |
|---|---|---|
| Range-over-function & `iter` | Language / `iter` | `iter.Seq[V]`, `iter.Seq2[K,V]`, `iter.Pull` coroutines, `maps.Keys`, `slices.Chunk`, `slices.Backward`. |
| `unique` package | `unique` | `unique.Make[T]` value canonicalization (interning) with `O(1)` handle equality. |
| Synchronous unbuffered timers | `time` / runtime | `time.Timer`/`Ticker` channels become unbuffered and GC-eligible without `Stop()`. |

## Go 1.24

| Feature | Package / Layer | What It Does |
|---|---|---|
| SwissTable maps & `HashTrieMap` `sync.Map` | Runtime | Parallel 8-slot SWAR/SIMD map probing, bounded 1,024-element table splits, lock-free read `sync.Map`. |
| Generic type aliases | Language | Parameterized type aliases (`type Handler[T any] = func(T) error`). |
| `omitzero` struct tag | `encoding/json` | Omits zero-value structs (`time.Time`, `time.Duration`) cleanly. |
| `os.Root` (`os.OpenRoot`) | `os` | Kernel-enforced traversal-resistant directory scoping (`openat2` on Linux). |
| `weak.Pointer[T]` & `runtime.AddCleanup` | `weak` / `runtime` | Non-resurrecting, cycle-safe cleanups and zero-leak weak caches. |
| `testing.B.Loop()` & `T.Context()` | `testing` | Safer benchmark loop + test-scoped cancellation context. |
| `strings.SplitSeq`, `FieldsSeq`, `Lines` | `strings` / `bytes` | Zero-slice-allocation splitting iterators. |
| `crypto/mlkem`, `crypto/hkdf`, `crypto/sha3`, `crypto/fips140`, `NewGCMWithRandomNonce` | `crypto/*` | Post-quantum FIPS 203 KEM, native HKDF/SHA-3, built-in FIPS 140-3 module. |
| `go.mod` `tool` directives & `GOCACHEPROG` | Toolchain | First-class CLI tool tracking (`go get -tool`) and remote build cache protocol. |

## Go 1.25

| Feature | Package / Layer | What It Does |
|---|---|---|
| Container-aware `GOMAXPROCS` | Runtime | Reads Linux cgroup CPU quotas dynamically (`runtime.SetDefaultGOMAXPROCS`). |
| `sync.WaitGroup.Go(fn)` | `sync` | Launches and tracks a goroutine without `Add(1)` / `defer Done()` boilerplate. |
| `testing/synctest` (GA) | `testing/synctest` | Deterministic concurrency bubbles with virtualized clocks. |
| `runtime/trace.FlightRecorder` | `runtime/trace` | Always-on `<1%` CPU ring-buffer execution trace snapshots. |
| `reflect.TypeAssert[T]` | `reflect` | Zero-allocation generic extraction from `reflect.Value`. |
| `http.CrossOriginProtection` | `net/http` | Token-free CSRF defense using `Sec-Fetch-Site` / `Origin`. |
| Parallel `AddCleanup` & `unique` GC | Runtime | Concurrent cleanup workers and single-cycle `unique.Handle` reclamation; `GODEBUG=checkfinalizers=1`. |

## Go 1.26

| Feature | Package / Layer | What It Does |
|---|---|---|
| Green Tea GC (default) | Runtime | Span-local + SIMD pointer scanning cutting GC CPU overhead by 10%–40%. |
| `new(expr)` | Language | Built-in `new` accepts any value expression (`new(int32(5))`). |
| Self-referential type constraints | Language | Enables F-bounded polymorphism (`type Adder[A Adder[A]] interface { Add(A) A }`). |
| `errors.AsType[E]` | `errors` | Zero-reflection generic replacement for `errors.As`. |
| ~30% faster Cgo & 64-bit heap randomization | Runtime | Lower `cgocall` overhead and randomized 64-bit `mheap` base addresses. |
| Rewritten `go fix` modernizer | Toolchain | `go/analysis`-based modernizers and `//go:fix inline` deprecation inlining. |
| `crypto/hpke`, `testing/cryptotest`, `slog.NewMultiHandler`, `bytes.Buffer.Peek`, `Dialer.DialTCP`, `reflect.Type.Fields()` | Standard library | RFC 9180 HPKE, deterministic crypto tests, multi-handler logging, zero-alloc lookahead & typed dialing. |
| `runtime/secret` & `simd/archsimd` (experimental) | Runtime / `simd` | Cryptographic stack/register zeroing and hardware SIMD intrinsics. |

## Go 1.27 (Latest: Go 1.27.4 — Nov 10, 2026)

| Feature | Package / Layer | What It Does |
|---|---|---|
| Generic methods | Language | Methods on concrete types can declare their own type parameters (e.g., `(*rand.Rand).N[Int]`). |
| Promoted field keys in struct literals | Language | Flattened composite literals for unambiguously promoted embedded fields (`go fix -embedlit`). |
| Generalized function type inference | Language | Inference in composite literals, conversions, and channel sends. |
| Size-specialized malloc | Runtime | Up to 30% faster small (`< 80 B`) heap allocations. |
| `goroutineleak` pprof profile (GA) | `runtime/pprof` | GC-assisted `/debug/pprof/goroutineleak` profile; `pprof` labels printed in panic tracebacks by default. |
| `asynctimerchan` removed | Runtime / `time` | Timer channels are always synchronous and unbuffered. |
| `encoding/json/v2` & `jsontext` (GA) | `encoding/json/v2` | Strict-by-default v2 JSON engine (rejects duplicate keys & invalid UTF-8) + streaming token API. |
| `uuid` & `crypto/mldsa` | Standard library | First-party `uuid` package and NIST FIPS 204 ML-DSA post-quantum signatures + `tls.MLKEM1024`. |
| `strings.CutLast` / `bytes.CutLast`, `url.URL.Clone`, `synctest.Sleep`, `httptest.NewTestServer` | Standard library | Reverse cut, deep URL cloning, synctest virtual sleep, in-memory HTTP test server. |
| `driver.RowsColumnScanner` & `sql.ConvertAssign`, `maphash.Hasher[T]` | `database/sql` / `hash/maphash` | Zero-boxing SQL column scanning and generic hashers. |
| Portable `simd` (`GOEXPERIMENT=simd`) | `simd` | Width-agnostic portable vectors alongside `simd/archsimd` (amd64, arm64 NEON, Wasm). |
| Go 1.27.1–1.27.4 patch releases | Security & stability | Security fixes across `cgo`, `cmd/go`, compiler, linker, `crypto/x509`, `crypto/tls`, `html/template`, `net/http`, and `os`. |

## Complete Go 1.27.4 Showcase Program

```go
package main

import (
	"encoding/json/v2"
	"errors"
	"fmt"
	"strings"
	"uuid"
)

type ShiftEntry struct {
	ID         uuid.UUID `json:"id"`
	StaffID    string    `json:"staff_id"`
	HourlyRate *float64  `json:"hourly_rate,omitzero"`
}

func main() {
	shifts := []ShiftEntry{
		{ID: uuid.New(), StaffID: "emp-001", HourlyRate: new(22.50)},
		{ID: uuid.New(), StaffID: "emp-002"}, // HourlyRate omitted via omitzero.
	}
	data, err := json.Marshal(shifts)
	if err != nil {
		if syntaxErr, ok := errors.AsType[*json.SemanticError](err); ok {
			fmt.Println("semantic error:", syntaxErr)
		}
		panic(err)
	}
	fmt.Println(string(data))

	base, ext, _ := strings.CutLast("report.2026.csv", ".")
	fmt.Printf("base=%q ext=%q\n", base, ext)
}
```
