---
title: String Interning, Weak Pointers, and Zero-Leak Memory Caches
---

# String Interning, Weak Pointers, and Zero-Leak Memory Caches

In telemetry collectors, compilers, and API gateways, up to 70% of heap RAM is often duplicate immutable values (metric labels, header names, tenant IDs). Go 1.23–1.27 solves value deduplication without unbounded map leaks through three integrated primitives: `unique.Handle[T]`, `weak.Pointer[T]`, and `runtime.AddCleanup`.

## Value Canonicalization (`unique.Handle[T]`)

- `unique.Make(val)` stores a single canonical copy in a per-type concurrent `HashTrieMap[T, weak.Pointer[T]]` and clones substrings internally (`strings.Clone`) so a 10-byte key never pins a 64 MB parent payload.
- Comparing two handles (`h1 == h2`) is a single 64-bit pointer compare (`~0.3 ns`) instead of byte-by-byte string comparison.
- In Go 1.25–1.27, unreferenced handles are reclaimed in parallel within a single GC cycle.

## `weak.Pointer[T]` and `runtime.AddCleanup` Internals

```text
weak.Pointer[T] (copy 1)       weak.Pointer[T] (copy 2)
          \                             /
           v                           v
    +-----------------------------------------+
    | 8-Byte Runtime Weak Indirection Word    | <-- Atomically zeroed by GC when T dies
    +--------------------+--------------------+
                         v (Not traced as strong root)
    +-----------------------------------------+
    | Target Heap Object (*T)                 |
    +-----------------------------------------+
```

| Feature | Legacy `runtime.SetFinalizer` | Modern `runtime.AddCleanup` (Go 1.24–1.27) |
|---|---|---|
| Object resurrection | Resurrects for >= 1 extra GC cycle | Never resurrects; freed immediately |
| Reference cycles | Leaks forever | Collects cycles normally |
| Multiple callbacks / interior ptrs | 1 finalizer at offset 0 only | Unlimited cleanups, interior pointers supported |
| Execution | Single-goroutine bottleneck | Concurrent parallel workers (Go 1.25+) |

## Self-Evicting Zero-Leak Weak Cache in Go 1.27

```go
package weakcache

import (
	"runtime"
	"sync"
	"weak"
)

type Cache[K comparable, V any] struct {
	store sync.Map // K -> weak.Pointer[V] (HashTrieMap in Go 1.24+)
}

type cleanupState[K comparable, V any] struct {
	store *sync.Map
	key   K
	wp    weak.Pointer[V]
}

func (c *Cache[K, V]) Get(key K) (*V, bool) {
	raw, ok := c.store.Load(key)
	if !ok {
		return nil, false
	}
	if val := raw.(weak.Pointer[V]).Value(); val != nil {
		return val, true
	}
	return nil, false
}

func (c *Cache[K, V]) GetOrCompute(key K, compute func() *V) *V {
	if val, ok := c.Get(key); ok {
		return val
	}
	newVal := compute()
	wp := weak.Make(newVal)

	for {
		actual, loaded := c.store.LoadOrStore(key, wp)
		if !loaded {
			runtime.AddCleanup(newVal, func(s cleanupState[K, V]) {
				// CompareAndDelete avoids deleting a newer entry inserted before cleanup ran.
				s.store.CompareAndDelete(s.key, s.wp)
			}, cleanupState[K, V]{store: &c.store, key: key, wp: wp})
			return newVal
		}
		existingWP := actual.(weak.Pointer[V])
		if existingVal := existingWP.Value(); existingVal != nil {
			return existingVal
		}
		if c.store.CompareAndSwap(key, existingWP, wp) {
			runtime.AddCleanup(newVal, func(s cleanupState[K, V]) {
				s.store.CompareAndDelete(s.key, s.wp)
			}, cleanupState[K, V]{store: &c.store, key: key, wp: wp})
			return newVal
		}
	}
}
```

> [!WARNING]
> Never pass `ptr` inside the `arg` of `runtime.AddCleanup(ptr, fn, arg)`—holding a strong reference back to `ptr` prevents GC forever. Run `GODEBUG=checkfinalizers=1` (Go 1.25+) in CI to catch accidental retention.

## Related Chapters

- [217 weak and unique Packages](../25-go-deep-dives/217-weak-unique-packages.md)
- [999h uuid, unique, weak, and AddCleanup Scenarios](../07-stdlib/999h-unique-weak-addcleanup-scenarios.md)
- [190 High-Performance Caching, W-TinyLFU, and Zero-GC Byte Slabs](190-high-perf-caching-w-tinylfu-byte-slabs.md)
