---
title: Lock-Free Programming, Atomic Primitives, and Michael-Scott Queues
---

# Lock-Free Programming, Atomic Primitives, and Michael-Scott Queues

When dozens of goroutines contend on a single `sync.Mutex`, the runtime transitions from user-space spinning into kernel futex sleep (`gopark`), causing context switches, cache-line bouncing, and tail-latency spikes. Lock-free programming uses hardware atomic instructions (`LOCK CMPXCHG`) and Go's sequentially consistent `sync/atomic` primitives to eliminate mutex contention on hot paths.

## Hardware Cache Coherency and Synchronization Latency

```text
+--------------------------------+       +--------------------------------+
|            Core 0              |       |            Core 1              |
|  L1 Data Cache (32 KB)         |       |  L1 Data Cache (32 KB)         |
|  L2 Cache (512 KB - 1 MB)      |       |  L2 Cache (512 KB - 1 MB)      |
+-----------------+--------------+       +-----------------+--------------+
                  |                                        |
                  +-------------------+--------------------+
                                      |
                       Shared L3 Cache (16 - 64 MB) -> RAM
```

| Operation | Typical Latency |
|---|---|
| L1 cache hit | ~1 ns |
| L3 cache hit | ~10–15 ns |
| Cross-core cache-line invalidation (MESI) | ~40–60 ns |
| Atomic `LOCK CMPXCHG` | ~3–8 ns |
| Uncontended `sync.Mutex` | ~10–15 ns |
| Contended `sync.Mutex` (futex + `gopark`) | ~1,000–3,000 ns |

All `sync/atomic` operations in Go are **sequentially consistent**: an atomic store/swap/CAS *happens before* any subsequent atomic load that observes the stored value.

## Copy-On-Write with `atomic.Pointer[T]` and False-Sharing Defense

```go
package lockfree

import "sync/atomic"

type ServerConfig struct {
	MaxConns int
	Host     string
}

// PaddedCounter isolates each 8-byte atomic counter onto its own 64-byte cache line
// to prevent false sharing across CPU cores (15x–25x higher multi-core throughput).
type PaddedCounter struct {
	Count atomic.Uint64
	_     [56]byte
}

func UpdateConfigCOW(active *atomic.Pointer[ServerConfig], maxConns int) {
	for {
		cur := active.Load()
		next := *cur
		next.MaxConns = maxConns
		if active.CompareAndSwap(cur, &next) {
			return
		}
	}
}
```

## Lock-Free Michael-Scott FIFO Queue

Because Go's garbage collector never recycles a node's address while any goroutine holds a pointer to it, the classic C/C++ **ABA problem** is structurally impossible for pointer nodes in Go:

```go
package lockfree

import "sync/atomic"

type node[T any] struct {
	value T
	next  atomic.Pointer[node[T]]
}

type Queue[T any] struct {
	head atomic.Pointer[node[T]]
	tail atomic.Pointer[node[T]]
}

func NewQueue[T any]() *Queue[T] {
	sentinel := &node[T]{}
	q := &Queue[T]{}
	q.head.Store(sentinel)
	q.tail.Store(sentinel)
	return q
}

func (q *Queue[T]) Enqueue(val T) {
	newNode := &node[T]{value: val}
	for {
		tail := q.tail.Load()
		next := tail.next.Load()
		if tail == q.tail.Load() {
			if next == nil {
				if tail.next.CompareAndSwap(nil, newNode) {
					q.tail.CompareAndSwap(tail, newNode)
					return
				}
			} else {
				q.tail.CompareAndSwap(tail, next)
			}
		}
	}
}

func (q *Queue[T]) Dequeue() (T, bool) {
	var zero T
	for {
		head := q.head.Load()
		tail := q.tail.Load()
		next := head.next.Load()
		if head == q.head.Load() {
			if head == tail {
				if next == nil {
					return zero, false
				}
				q.tail.CompareAndSwap(tail, next)
			} else {
				val := next.value
				if q.head.CompareAndSwap(head, next) {
					return val, true
				}
			}
		}
	}
}
```

## Go 1.24–1.27 Atomic and Runtime Upgrades

- **Runtime `spinbitmutex` (Go 1.24+):** Encodes spinning thread state in low-order bits of internal runtime lock words to cut cross-core cache-line bouncing.
- **`sync.Map` `HashTrieMap` (Go 1.24+):** Lock-free 16-way atomic pointer traversal for reads, leaf-level locking for writes.
- **`go fix` `atomictypes` modernizer (Go 1.27):** Automatically rewrites error-prone `atomic.AddUint64(&x, 1)` calls into alignment-safe `atomic.Uint64`, `atomic.Int64`, `atomic.Bool`, and `atomic.Pointer[T]`.

## Related Chapters

- [46 Lock-Free Ring Buffers and MPMC Queues](46-lock-free-ring-buffers-mpmc.md)
- [202 Memory Model and Happens-Before](../25-go-deep-dives/202-memory-model-happens-before.md)
- [227 Mutex and Semaphore Runtime](../25-go-deep-dives/227-mutex-semaphore-runtime.md)
