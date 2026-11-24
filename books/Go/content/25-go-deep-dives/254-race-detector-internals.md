---
title: Race Detector (-race) and ThreadSanitizer Shadow Memory Internals
---

# Race Detector (`-race`) and ThreadSanitizer Shadow Memory Internals

`go test -race` and `go build -race` integrate ThreadSanitizer (TSan) directly into `cmd/compile` and `src/runtime/race.go`. Unlike heuristic linters, the race detector guarantees **zero false positives**: every `WARNING: DATA RACE` proves an unsynchronized concurrent memory access where at least one operation is a write.

## Compiler SSA Instrumentation

```text
Normal Go Code                                  Compiler -race Instrumented Output
+-----------------------------------------+     +-------------------------------------------------+
| func increment(p *int64) {              |     | func increment(p *int64) {                      |
|     *p++                                |     |     runtime.racefuncenter(callerpc)             |
| }                                       |     |     runtime.raceread(uintptr(p))                |
|                                         |     |     v := *p                                     |
|                                         |     |     runtime.racewrite(uintptr(p))               |
|                                         |     |     *p = v + 1                                  |
|                                         |     |     runtime.racefuncexit()                      |
|                                         |     | }                                               |
+-----------------------------------------+     +-------------------------------------------------+
```

`runtime.raceread` and `runtime.racewrite` (`src/runtime/race_amd64.s`) extract `g.racectx`, switch `RSP` from the 2 KB goroutine stack onto the OS thread's `g0` stack (without entering `_Gsyscall`), and call `__tsan_read` / `__tsan_write`.

## Direct-Mapped Shadow Memory (1 Application Word -> 4 Shadow Words)

```text
             Application Memory Word (8 Bytes at Address 'A')
             +-----+-----+-----+-----+-----+-----+-----+-----+
             | B0  | B1  | B2  | B3  | B4  | B5  | B6  | B7  |
             +-----+-----+-----+-----+-----+-----+-----+-----+
                                     |
       Direct Bitwise Mapping: ShadowAddr = ((A & ~7) * 4) + ShadowBase
       (Zero hash tables, zero locks — 3-instruction bitwise math)
                                     v
             4 Contiguous 64-Bit Shadow Cells (32 Bytes Total)
+-------------------------------------------------------------------------+
| Shadow Cell 0 (64-bit): [ tid: 14b | epoch: 42b | off: 3b | sz: 2b | W ] |
| Shadow Cell 1 (64-bit): [ tid: 14b | epoch: 42b | off: 3b | sz: 2b | R ] |
| Shadow Cell 2 (64-bit): [ Empty (0x0)                                  ] |
| Shadow Cell 3 (64-bit): [ Empty (0x0)                                  ] |
+-------------------------------------------------------------------------+
```

Each 64-bit shadow cell encodes:
- **`tid` (14 bits):** Goroutine race context ID.
- **`epoch` (42 bits):** Monotonic logical clock tick of that goroutine at access time.
- **`off` & `sz` (byte mask):** Exact byte range (`0..7`) touched within the 8-byte word—proving in 1 bitwise `AND` that concurrent writes to adjacent fields `s.A` (bytes `0..3`) and `s.B` (bytes `4..7`) are **not** a race.
- **`isWrite` & `isAtomic`:** Distinguishes safe `R/R` from `W/R` and `W/W` conflicts.

## Vector Clocks and Happens-Before Verification

```text
Goroutine G1 Vector Clock:  VC(G1) = [ G1: 45, G2: 10 ]
(G1 writes x = 42 at G1 tick 45)
       |
       |  G1 calls mu.Unlock() -> runtime.RaceReleaseMerge(&mu)
       v  Sync Object VC(&mu) = max(VC(&mu), VC(G1)) = [ G1: 45, G2: 10 ]
+-----------------------------------------------------------------------+
       |
       |  G2 calls mu.Lock()   -> runtime.RaceAcquire(&mu)
       v  G2 merges VC(&mu) into VC(G2): VC(G2) = [ G1: 45, G2: 22 ]
Goroutine G2 reads x:
  - Shadow Cell shows G1 wrote x at epoch = 45.
  - Checks VC(G2)[G1] >= 45 (45 >= 45 -> TRUE).
  - Result: Happens-Before edge exists -> SAFE (No Data Race).
```

When all 4 shadow cells for an 8-byte word are occupied, TSan evicts 1 random cell to keep memory bounded at 4x–8x.

## Catching the Shared Backing-Array `append` Race

```go
package racetraps

import "sync"

// BUG: If cap(base) > len(base), both goroutines write to backing array index len(base).
func AppendSharedCapacityBug(base []int) {
	var wg sync.WaitGroup
	wg.Go(func() { _ = append(base, 100) })
	wg.Go(func() { _ = append(base, 200) }) // DATA RACE on base[len(base)]
	wg.Wait()
}

// FIX: Three-index slicing clips cap == len, forcing independent backing arrays.
func AppendClippedCapacitySafe(base []int) {
	clipped := base[:len(base):len(base)]
	var wg sync.WaitGroup
	wg.Go(func() { _ = append(clipped, 100) })
	wg.Go(func() { _ = append(clipped, 200) })
	wg.Wait()
}
```

## Dynamic `-race` vs. Static SSA Verification (`gohawk`)

| Dimension | Dynamic Race Detector (`go test -race`) | Static SSA Analyzer ([gohawk](https://gohawk.dev/)) |
|---|---|---|
| Mechanism | Runtime Shadow Memory + Vector Clocks (TSan) | Compile-time interprocedural SSA dataflow & call graph |
| Unexercised branches | Blind if tests do not hit the concurrent interleaving | 100% path coverage across all error branches |
| Deadlocks / lock order | Does not detect AB-BA lock inversions | `lockorder` catches lock inversions at compile time |
| Orphaned goroutines | Does not flag unjoined goroutines | `goroutineownership` flags unjoined goroutines |
| Overhead | 5x–10x RAM, 2x–20x CPU | Zero runtime overhead (runs in CI) |

> [!TIP]
> **Production canary strategy:** Pair `go tool gohawk ./...` in CI with `go test -race ./...` (including `testing/synctest` bubbles), and consider running one staging canary pod built with `go build -race` (`GORACE="log_path=/var/log/race/report halt_on_error=0"`). There are no benign data races in Go—SSA passes assume race-free code and can hoist loads or tear 16-byte strings/interfaces.

## Related Chapters

- [202 Memory Model and Happens-Before](202-memory-model-happens-before.md)
- [223 testing/synctest Bubbles](223-testing-synctest-bubbles.md)
- [262 Race Conditions vs. Data Races](../10-concurrency-ground-up/262-race-conditions-vs-data-races.md)
