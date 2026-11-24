---
title: Arena Allocators, Bump-Pointer Slabs, Index Handles, and Off-Heap Mmap
---

# Arena Allocators, Bump-Pointer Slabs, Index Handles, and Off-Heap `mmap`

Even with Go 1.27's size-specialized allocator and the Green Tea GC, allocating millions of short-lived AST/query-plan nodes per request or holding 50 million `*Node` pointers in an in-memory graph creates two structural bottlenecks: per-object GC sweep overhead and `O(N)` GC pointer-scanning cost.

## Standard Heap vs. Bump-Pointer Slab Arena

```text
Standard Per-Object Heap Allocation           Region-Based Bump-Pointer Arena
+-------------------------------------+       +-------------------------------------+
| n1 := &Node{} -> mallocgc #1        |       | [ Node 0 | Node 1 | Node 2 | ... ]  |
| n2 := &Node{} -> mallocgc #2        |       |                 ^                   |
| Scattered across mspan slots        |       |            cursor++ (~0.5 ns)       |
| Freed individually by GC sweep      |       | Contiguous L1/L2 cache locality     |
+-------------------------------------+       | Bulk O(1) Reset() via clear()       |
                                              +-------------------------------------+
```

### Generic Chunked `SlabArena[T]`

```go
package arena

const defaultChunkSize = 1024

type SlabArena[T any] struct {
	chunks    [][]T
	chunkIdx  int
	itemIdx   int
	chunkSize int
}

func NewSlabArena[T any](chunkSize int) *SlabArena[T] {
	if chunkSize <= 0 {
		chunkSize = defaultChunkSize
	}
	return &SlabArena[T]{chunks: [][]T{make([]T, chunkSize)}, chunkSize: chunkSize}
}

func (a *SlabArena[T]) Alloc() *T {
	if a.itemIdx >= a.chunkSize {
		a.chunkIdx++
		if a.chunkIdx >= len(a.chunks) {
			a.chunks = append(a.chunks, make([]T, a.chunkSize))
		}
		a.itemIdx = 0
	}
	ptr := &a.chunks[a.chunkIdx][a.itemIdx]
	a.itemIdx++
	return ptr
}

func (a *SlabArena[T]) Reset() {
	for i := 0; i <= a.chunkIdx && i < len(a.chunks); i++ {
		limit := a.chunkSize
		if i == a.chunkIdx {
			limit = a.itemIdx
		}
		clear(a.chunks[i][:limit]) // Bulk memclr prevents data bleed & releases child pointers.
	}
	a.chunkIdx, a.itemIdx = 0, 0
}
```

## Eliminating GC Scanning Completely: Index Handles (`noscan` Bypass)

Replacing 64-bit `*Node` pointers (`Left *Node`, `Right *Node` -> 24 B, `scan` mspan) with 32-bit slice indices (`Left int32`, `Right int32`, `-1 = nil` -> 16 B, `noscan` mspan) packs 4 nodes per 64-byte cache line and reduces Green Tea GC mark time to **0 ms** regardless of tree size.

## Off-Heap `syscall.Mmap` Slabs with `runtime.AddCleanup`

```go
//go:build unix

package offheap

import (
	"fmt"
	"runtime"
	"syscall"
	"unsafe"
)

type MmapSlab struct{ data []byte }

func NewMmapSlab(size int) (*MmapSlab, error) {
	if size <= 0 {
		return nil, fmt.Errorf("offheap: invalid size %d", size)
	}
	raw, err := syscall.Mmap(-1, 0, size, syscall.PROT_READ|syscall.PROT_WRITE, syscall.MAP_ANON|syscall.MAP_PRIVATE)
	if err != nil {
		return nil, err
	}
	slab := &MmapSlab{data: raw}
	runtime.AddCleanup(slab, func(b []byte) { _ = syscall.Munmap(b) }, raw)
	return slab, nil
}

// CastSlice reinterprets off-heap memory as []T. T MUST NOT contain Go heap pointers.
func CastSlice[T any](s *MmapSlab) []T {
	var zero T
	return unsafe.Slice((*T)(unsafe.Pointer(unsafe.SliceData(s.data))), len(s.data)/int(unsafe.Sizeof(zero)))
}
```

> [!CAUTION]
> Never store pointers to Go-heap objects inside off-heap `mmap` memory: the GC does not scan outside `mheap` and will prematurely free Go objects only referenced from `MmapSlab`.

## Related Chapters

- [182 Memory Allocations and GC](182-memory-allocations-gc.md)
- [189 String Interning, Weak Pointers, and Zero-Leak Caches](189-string-interning-weak-caches.md)
- [247 Memory Allocator Deep Dive](../25-go-deep-dives/247-memory-allocator-deep-dive.md)
