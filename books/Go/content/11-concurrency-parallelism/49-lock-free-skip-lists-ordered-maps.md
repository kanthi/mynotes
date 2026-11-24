---
title: Lock-Free Skip Lists and Concurrent Ordered Maps (LSM MemTables)
---

# Lock-Free Skip Lists and Concurrent Ordered Maps (LSM MemTables)

Storage engines like CockroachDB's **Pebble** and Dgraph's **BadgerDB** need an in-memory MemTable that handles high-concurrency writes *and* ordered range scans (`Seek`, `ScanRange`). Hash maps (`map[K]V`, `sync.Map`) cannot range-scan, while balanced trees (RB-trees, B+ trees) require global rebalancing locks. **Lock-free skip lists** solve both using probabilistic multi-level towers and local atomic CAS splicing.

```text
Level 3: Head ------------------------------------------------------> [Node 25] -------------> Tail
Level 2: Head -----------------------------> [Node 10] -------------> [Node 25] -------------> Tail
Level 1: Head -------------> [Node 5] -----> [Node 10] -------------> [Node 25] -> [Node 30] -> Tail
Level 0: Head -> [Node 2] -> [Node 5] -----> [Node 10] -> [Node 18] -> [Node 25] -> [Node 30] -> Tail
```

| Structure | Lookup | Insert | Range Scan | Concurrency Model |
|---|---|---|---|---|
| Go `map[K]V` (SwissTable) | `O(1)` | `O(1)` | Unsupported | Unsafe (panics on concurrent write) |
| `sync.Map` (`HashTrieMap`) | `O(1)` | `O(1)` | Unsupported | Lock-free reads, leaf locks |
| B+ Tree | `O(log N)` | `O(log N)` | Fast | Page latching / RWMutex |
| Lock-Free Skip List | `O(log N)` | `O(log N)` | Fast (Level 0) | 100% lock-free reads & writes |

## Complete Lock-Free Skip List with `iter.Seq2` Range Scans

```go
package skiplist

import (
	"cmp"
	"iter"
	"math/rand/v2"
	"sync/atomic"
)

const (
	MaxHeight = 16
	Branching = 4
)

type node[K cmp.Ordered, V any] struct {
	key     K
	val     atomic.Pointer[V]
	deleted atomic.Bool
	forward []atomic.Pointer[node[K, V]]
}

func newNode[K cmp.Ordered, V any](key K, val V, height int) *node[K, V] {
	n := &node[K, V]{key: key, forward: make([]atomic.Pointer[node[K, V]], height)}
	n.val.Store(&val)
	return n
}

type ConcurrentSkipList[K cmp.Ordered, V any] struct {
	head *node[K, V]
}

func New[K cmp.Ordered, V any]() *ConcurrentSkipList[K, V] {
	var zk K
	var zv V
	return &ConcurrentSkipList[K, V]{head: newNode(zk, zv, MaxHeight)}
}

func (s *ConcurrentSkipList[K, V]) randomHeight() int {
	h := 1
	for h < MaxHeight && rand.IntN(Branching) == 0 {
		h++
	}
	return h
}

func (s *ConcurrentSkipList[K, V]) Get(key K) (V, bool) {
	var zero V
	curr := s.head
	for level := MaxHeight - 1; level >= 0; level-- {
		for {
			next := curr.forward[level].Load()
			if next == nil || cmp.Compare(next.key, key) >= 0 {
				break
			}
			curr = next
		}
	}
	target := curr.forward[0].Load()
	if target != nil && cmp.Compare(target.key, key) == 0 && !target.deleted.Load() {
		if vp := target.val.Load(); vp != nil {
			return *vp, true
		}
	}
	return zero, false
}

func (s *ConcurrentSkipList[K, V]) Put(key K, val V) {
	var preds, succs [MaxHeight]*node[K, V]
	for {
		curr := s.head
		for level := MaxHeight - 1; level >= 0; level-- {
			for {
				next := curr.forward[level].Load()
				if next == nil || cmp.Compare(next.key, key) >= 0 {
					break
				}
				curr = next
			}
			preds[level] = curr
			succs[level] = curr.forward[level].Load()
		}
		if target := succs[0]; target != nil && cmp.Compare(target.key, key) == 0 {
			target.val.Store(&val)
			target.deleted.Store(false)
			return
		}
		height := s.randomHeight()
		newN := newNode(key, val, height)
		for i := range height {
			newN.forward[i].Store(succs[i])
		}
		// Bottom-up CAS: Level 0 makes the node atomically visible.
		if preds[0].forward[0].CompareAndSwap(succs[0], newN) {
			for level := 1; level < height; level++ {
				for {
					newN.forward[level].Store(succs[level])
					if preds[level].forward[level].CompareAndSwap(succs[level], newN) {
						break
					}
					curr = s.head
					for {
						n := curr.forward[level].Load()
						if n == nil || cmp.Compare(n.key, key) >= 0 {
							break
						}
						curr = n
					}
					preds[level] = curr
					succs[level] = curr.forward[level].Load()
				}
			}
			return
		}
	}
}

// ScanRange yields keys in [from, to) via Go 1.23+ range-over-func iterators.
func (s *ConcurrentSkipList[K, V]) ScanRange(from, to K) iter.Seq2[K, V] {
	return func(yield func(K, V) bool) {
		curr := s.head
		for level := MaxHeight - 1; level >= 0; level-- {
			for {
				next := curr.forward[level].Load()
				if next == nil || cmp.Compare(next.key, from) >= 0 {
					break
				}
				curr = next
			}
		}
		for curr = curr.forward[0].Load(); curr != nil && cmp.Compare(curr.key, to) < 0; curr = curr.forward[0].Load() {
			if !curr.deleted.Load() {
				if vp := curr.val.Load(); vp != nil && !yield(curr.key, *vp) {
					return
				}
			}
		}
	}
}
```

## Related Chapters

- [45 Lock-Free Atomics and Queues](45-lock-free-atomics-and-queues.md)
- [218 Swiss Maps and maphash](../25-go-deep-dives/218-swiss-maps-maphash.md)
- [229 Iterators and range-over-func](../25-go-deep-dives/229-iterators-range-over-func.md)
