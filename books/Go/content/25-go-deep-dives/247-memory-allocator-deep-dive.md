---
title: Memory Allocator and TCMalloc Subsystem (mcache, mcentral, mheap, and Size-Specialized Malloc)
---

# Memory Allocator and TCMalloc Subsystem (`mcache`, `mcentral`, `mheap`, and Size-Specialized Malloc)

To achieve near-zero-latency heap allocations without a global heap lock, Go adapts Google's TCMalloc architecture (`src/runtime/malloc.go`, `mcache.go`, `mcentral.go`, `mheap.go`), binding thread-local caches directly to logical processors (`P`) so 99% of small allocations execute lock-free.

## Three-Tier Allocation Hierarchy

```text
               +---------------------------------------------+
               |                 Goroutine                   |
               +----------------------+----------------------+
                                      |
                         [1] Request Allocation (<32KB)
                                      v
               +---------------------------------------------+
               |        mcache (Per-Logical-Processor P)     |  <-- LOCK-FREE
               |  - 68 size classes x 2 (scan / noscan)      |
               |  - Tiny Allocator (<16B, no pointers)       |
               +----------------------+----------------------+
                                      |
                      [2] Local Span Exhausted (Miss)
                                      v
               +---------------------------------------------+
               |          mcentral (Per-Size-Class Span)     |  <-- FINE-GRAINED LOCK
               |  - partial span list (has free objects)     |
               |  - full span list (all allocated)           |
               +----------------------+----------------------+
                                      |
                      [3] Central Spans Exhausted
                                      v
               +---------------------------------------------+
               |             mheap (Global Page Heap)        |  <-- GLOBAL HEAP LOCK
               |  - Radix tree page allocator (8KB pages)    |
               |  - Virtual memory arenas (64MB chunks)      |
               |  - Large object allocations (>32KB)         |
               +----------------------+----------------------+
                                      |
                      [4] Virtual Memory Exhausted
                                      v
               +---------------------------------------------+
               |       Operating System Kernel (mmap)        |
               +---------------------------------------------+
```

## Object Categories and the Tiny Allocator

| Category | Size Range | Allocation Path | Locking |
|---|---|---|---|
| Tiny objects | `< 16 B` (pointer-free `noscan`) | `mcache.tiny` sub-allocator | Lock-free |
| Small objects | `16 B` – `32 KB` | `mcache.alloc[sizeclass]` | Lock-free |
| Large objects | `> 32 KB` | `mheap.alloc()` directly | Global lock |

- **Tiny allocator (`< 16 B`):** Packs multiple pointer-free values (`int32`, `bool`, short strings) into a single 16-byte block to prevent fragmentation. Objects containing pointers can never use the tiny allocator because GC must track exact pointer boundaries.
- **68 size classes & `mspan`:** Each `mspan` spans contiguous 8 KB pages split into equal-sized slots (`8 B` up to `32,768 B`) tracked by a 64-bit `allocCache` bitmap scanned in `O(1)` via hardware `CTZ` (Count Trailing Zeros).
- **`scan` vs. `noscan` spans:** Every size class has separate `scan` (pointer-bearing) and `noscan` (pointer-free) pools. Storing 100 MB in `[]byte` or pointer-free structs routes to `noscan` spans, costing **0 ms** of Green Tea GC mark scanning.
- **Virtual arenas & scavenging:** `mheap` maps 64 MB `heapArena` chunks via `mmap` and tracks 8 KB pages in a radix-tree bitmap (`pageAlloc.go`). Unused spans are returned to the OS via `madvise(MADV_DONTNEED)` rather than `munmap`.

## Modern Allocator Upgrades in Go 1.24–1.27

- **Size-specialized malloc (Go 1.27):** The compiler emits direct calls to size-specialized allocation routines for small objects (`< 80 B`), skipping generic `mallocgc` size-class dispatch and cutting small allocation cost by up to **30%** (opt out: `GOEXPERIMENT=nosizespecializedmalloc`).
- **Stack-allocated variable-size slices (Go 1.25–1.26):** Non-escaping `make([]T, n)` with runtime `n` uses bounded stack scratch space (`-compile=variablemake`), bypassing `mcache` and GC.
- **64-bit heap base randomization (Go 1.26+):** Randomizes `mheap` virtual base addresses at startup instead of predictable fixed offsets like `0x00c000000000` (`GOEXPERIMENT=norandomizedheapbase64` to opt out).
- **Linux anonymous VMA annotations (Go 1.25+):** Tags `mmap` regions as `[anon: Go: heap]` in `/proc/self/maps` (`GODEBUG=decoratemappings=1`).
- **Compact WebAssembly heap chunks (Go 1.24 & 1.26):** Cuts baseline `GOOS=wasm` RSS for heaps under 16 MiB.

## Benchmarking Tiny vs. Pointer Allocations

```go
package alloc_test

import "testing"

type TinyStruct struct{ A, B int8 }
type PointerStruct struct {
	A int8
	P *int8
}

func BenchmarkTinyAllocation(b *testing.B) {
	b.ReportAllocs()
	for b.Loop() {
		_ = &TinyStruct{A: 1, B: 2}
	}
}

func BenchmarkPointerAllocation(b *testing.B) {
	val := int8(42)
	b.ReportAllocs()
	for b.Loop() {
		_ = &PointerStruct{A: 1, P: &val}
	}
}
```

## Related Chapters

- [25d Size-Specialized Malloc](../03-memory/25d-size-specialized-malloc.md)
- [208 GC Internals and Pacer](208-gc-internals-pacer.md)
- [215 Stack Growth and Copying](215-stack-growth-copying.md)
