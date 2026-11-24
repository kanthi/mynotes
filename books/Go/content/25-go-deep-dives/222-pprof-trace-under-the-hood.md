---
title: Execution Tracer v2, FlightRecorder, and SIGPROF Profiler Internals
---

# Execution Tracer v2, `FlightRecorder`, and `SIGPROF` Profiler Internals

Recording nanosecond execution traces and 100 Hz CPU samples without locking the GMP scheduler or deadlocking signal handlers relies on three specialized runtime subsystems: the **Generation-Based Execution Tracer v2** (`src/runtime/trace.go`), the lock-free **`profBuf`** ring buffer (`src/runtime/cpuprof.go`, `profbuf.go`), and **512 KB Poisson heap sampling** (`mcache.nextSample`).

## Execution Tracer v2: Per-M Double Buffering and Generations

```text
Legacy Tracer v1 (Go <= 1.21)                   Execution Tracer v2 (Go 1.22 – Go 1.27)
+-----------------------------------------+     +-------------------------------------------------+
| - Global trace.lock on buffer flushes   |     | - Partitioned into self-contained Generations   |
| - Post-processing loads ENTIRE trace    |     | - Per-M double buffers: m.trace.buf[gen % 2]    |
|   into RAM to sort events & tables      |     | - Zero global lock on event writes              |
| - Cannot support FlightRecorder         |     | - Enables O(1) FlightRecorder ring-buffer drop  |
+-----------------------------------------+     +-------------------------------------------------+
```

1. **Zero-lock event writes:** Each OS thread `M` writes varint-encoded events (delta timestamp, `goid`, `stackID`, `stringID`) directly into its local `m.trace.buf[gen % 2]`.
2. **Wait-free generation flips (`traceAdvance`):** Roughly once per second, `trace.gen` increments (`N -> N+1`). Active writers immediately switch to `buf[(gen+1)%2]`, making generation `N` immutable for `traceReader` to drain with its own stack and string tables.

## How `trace.FlightRecorder` (Go 1.25–1.27) Runs Always-On at <1% CPU

```text
                   FlightRecorder Sliding Generation Ring
+-------------------------------------------------------------------------------+
|  [Gen 40: Dropped]   [Gen 41: 1.8s old]   [Gen 42: 0.9s old]   [Gen 43: Live] |
|                      |<----- Retained inside FlightRecorder.ring []rawGen --->|
+-------------------------------------------------------------------------------+
                                                      |
                        SLO Breach! fr.WriteTo(out) --+
                        1. Writes single GoTrace02 header
                        2. Copies raw bytes of Gen 41 + Gen 42 + Gen 43 in O(N)
```

Because each Tracer v2 generation is self-contained, evicting expired generations drops a single `[]byte` slice without parsing events or touching active threads.

## `SIGPROF` CPU Sampling and Lock-Free `profBuf`

`pprof.StartCPUProfile` arms a 100 Hz OS interval timer (`ITIMER_PROF` / `timer_create`). When `SIGPROF` interrupts `M` onto its 32 KB `m.gsignal` stack, `runtime.sigprof` cannot lock any mutex or allocate heap memory:

```text
Signal Context (m.gsignal stack)                 Background Reader (cpuProfile Goroutine)
+----------------------------------------+       +----------------------------------------+
| runtime.sigprof(pc, sp, lr, gp, mp)    |       | profBuf.read(profBufBlocking)          |
| 1. Unwinds gp's stack (up to 64 PCs)   |       | 1. Reads committed header + PCs        |
| 2. Reads gp.labels (*labelMap pointer) |       | 2. Resolves PCs via .gopclntab         |
| 3. Writes [count, time, tag, pc0..pcN] | ====> | 3. Compresses & writes pprof protobuf  |
|    into lock-free profBuf ring buffer  |       +----------------------------------------+
+----------------------------------------+
```

If `profBuf` fills up because the `io.Writer` stalls, `sigprof` atomically increments `buf.overflow` and returns immediately rather than blocking the signal handler.

## Heap Profiling: 512 KB Poisson Sampling (`mcache.nextSample`)

Each `P`'s `mcache.nextSample` holds a randomly sampled byte countdown with mean `runtime.MemProfileRate = 512 * 1024` (512 KB). Every allocation subtracts `size` from `nextSample`: 99.9% of allocations take a single subtraction with zero profiling cost; only when `nextSample <= 0` does the runtime unwind the stack into `mbuckets`.

## Complete Production Diagnostics Hub in Go 1.27

```go
package diagnostics

import (
	"context"
	"fmt"
	"net/http"
	"os"
	"runtime"
	"runtime/pprof"
	"runtime/trace"
	"sync/atomic"
	"time"
)

type Engine struct {
	fr           *trace.FlightRecorder
	lastSnapshot atomic.Int64
}

func Start() (*Engine, error) {
	runtime.SetMutexProfileFraction(5)
	runtime.SetBlockProfileRate(10_000) // Sample waits > 10 µs.

	fr := trace.NewFlightRecorder(trace.FlightRecorderConfig{
		MinAge:   5 * time.Second,
		MaxBytes: 4 << 20, // 4 MiB sliding ring buffer.
	})
	if err := fr.Start(); err != nil {
		return nil, fmt.Errorf("start flight recorder: %w", err)
	}
	return &Engine{fr: fr}, nil
}

func (e *Engine) Stop() error { return e.fr.Stop() }

func (e *Engine) InstrumentHandler(route string, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		tenant := r.Header.Get("X-Tenant-ID")
		if tenant == "" {
			tenant = "anonymous"
		}
		// Go 1.27: pprof labels appear in CPU profiles AND panic tracebacks.
		pprof.Do(r.Context(), pprof.Labels("route", route, "tenant", tenant), func(ctx context.Context) {
			ctx, task := trace.NewTask(ctx, "HTTP:"+route)
			defer task.End()

			start := time.Now()
			next.ServeHTTP(w, r.WithContext(ctx))

			if elapsed := time.Since(start); elapsed > 250*time.Millisecond {
				e.captureOncePerMinute()
			}
		})
	})
}

func (e *Engine) captureOncePerMinute() {
	now := time.Now().UnixNano()
	prev := e.lastSnapshot.Load()
	if now-prev < int64(time.Minute) || !e.lastSnapshot.CompareAndSwap(prev, now) {
		return
	}
	f, err := os.Create(fmt.Sprintf("flight-%d.trace", now))
	if err != nil {
		return
	}
	defer f.Close()
	_, _ = e.fr.WriteTo(f)
}
```

## Modern Go 1.26–1.27 Profiling Defaults

- **`go tool pprof -http=:8080`:** Opens directly to the interactive Flame Graph view (Go 1.26+).
- **`go tool trace -http=:6060`:** Binds safely to `localhost` by default (Go 1.27+).
- **`/debug/pprof/goroutineleak`:** Generally available in Go 1.27—uses a Green Tea GC mark pass to report goroutines parked on unreachable channels or locks.
- **Avoid `MemProfileRate = 1` or `SetBlockProfileRate(1)` in production:** Both add severe throughput overhead; keep `FlightRecorder` always-on with bounded `MaxBytes` and verify snapshot file cleanup with [gohawk](https://gohawk.dev/).

## Related Chapters

- [166b Flight Recorder](../19-observability-sre/166b-flight-recorder.md)
- [269 goroutineleak pprof Profile](269-goroutineleak-pprof.md)
- [258 OS Signals and Runtime](258-os-signals-runtime.md)
