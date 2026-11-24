---
title: 'Advanced Profiling: pprof list, Always-On pprof & gotraceui'
---

# Advanced Profiling: `pprof list`, Always-On `pprof` & `gotraceui`

Once you capture a `pprof` profile or an execution trace, knowing how to pinpoint exact source lines—and choosing the right visualization tool—turns raw samples into concrete speedups.

## Line-by-Line Attribution with `pprof list`

Flame graphs show *which* function is hot; `pprof`'s `list` command shows **which exact line** inside that function is burning CPU or allocating memory.

```bash
go tool pprof cpu.out
(pprof) top 10
(pprof) list processBatch
```

Example output:

```text
ROUTINE ======================== main.processBatch
     1.82s      4.95s (flat, cum) 62.10% of Total
         .          .     24: func processBatch(items []Item) []byte {
         .      2.10s     25:     buf := make([]byte, 0)
     1.82s      2.85s     26:     for _, it := range items {
         .          .     27:         buf = append(buf, encode(it)...)
         .          .     28:     }
         .          .     29:     return buf
         .          .     30: }
```

- **Flat** time/bytes: spent directly on that line.
- **Cum (cumulative)** time/bytes: spent on that line plus functions it called.
- Line 25 + 27 immediately reveal repeated slice growth—pre-allocating `make([]byte, 0, estimatedSize)` eliminates the bottleneck.

Use `weblist processBatch` (or `go tool pprof -http=:8081`) to inspect the same line-by-line view alongside assembly instructions in a browser.

## Should I Leave `pprof` Running Always?

Yes—**with strict network isolation**:
- **Negligible idle overhead:** Importing `net/http/pprof` registers handlers that do nothing until requested. CPU profiling only samples when `/debug/pprof/profile` is actively hit (~1–2% overhead during a 30s capture).
- **Avoid `http.DefaultServeMux` on public ports:** Never expose `/debug/pprof` on your public API router. Bind it to a separate loopback/admin listener (`127.0.0.1:6060`) reachable only via SSH tunnel or `kubectl port-forward`.
- **Watch `SetMutexProfileFraction` and `SetBlockProfileRate`:** Keep these at `0` (off) or use conservative sampling rates in production, as `rate=1` on high-contention locks adds measurable overhead.

## Case Study: 10x API Throughput with `pprof`

A common production pattern when an HTTP API stalls at low CPU:
1. **Symptom:** API throughput plateaued at ~300 req/s with 40% CPU usage.
2. **Capture:** `go tool pprof http://127.0.0.1:6060/debug/pprof/heap` and `/debug/pprof/mutex`.
3. **Finding (`list`):** Every request compiled a fresh `regexp.MustCompile(...)` inside the handler and serialized logs under a global mutex.
4. **Fix:** Hoisted `regexp.MustCompile` to package initialization and replaced the global lock with lock-free structured `slog` output.
5. **Result:** Heap allocations per request dropped by >85%, lock contention vanished, and throughput jumped >10x on the same hardware.

## Visualizing Execution Traces with `gotraceui`

While `go tool trace` renders traces in a browser, **[`gotraceui`](https://github.com/dominikh/gotraceui)** is a dedicated native desktop GUI built specifically for Go execution traces (including `runtime/trace.FlightRecorder` snapshots):

```bash
go install honnef.co/go/gotraceui/cmd/gotraceui@latest
gotraceui snapshot.trace
```

Why engineers reach for `gotraceui`:
- **Smooth 60fps zooming & panning** across multi-second traces with thousands of goroutines without browser tab freezes.
- **Goroutine state timelines:** Instantly spot whether a goroutine was *running*, *runnable* (waiting for a `P`), *blocked on sync/channel*, or *waiting on syscall/netpoll*.
- **Direct CPU-sample overlay & span histograms:** Inspect latency distributions across spans and jump straight to the exact goroutine wakeup chain that caused a p99 tail spike.

## Related Chapters

- [01 Advanced Profiling (pprof/trace)](01-pprof-trace.md)
- [166b Flight Recorder](../19-observability-sre/166b-flight-recorder.md)
- [222 pprof & Tracer Internals](../25-go-deep-dives/222-pprof-trace-under-the-hood.md)
- [181 Benchmarking & Profiling Workflow](../24-performance-engineering/181-benchmarking-profiling-workflow.md)
