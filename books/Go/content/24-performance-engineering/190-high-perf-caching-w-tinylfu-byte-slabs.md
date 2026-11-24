---
title: High-Performance In-Memory Caching, W-TinyLFU, and Zero-GC Byte Slabs
---

# High-Performance In-Memory Caching, W-TinyLFU, and Zero-GC Byte Slabs

A classic `map[string]*Item` guarded by a single `sync.RWMutex` and doubly linked LRU list collapses under multi-core load: every `Get` mutates the LRU list (forcing an exclusive `Lock()`), one-hit-wonder scans flush hot items, and 20 million `*Item` pointers tax GC mark phases. Modern Go caches (Ristretto, Otter, FastCache, BigCache) solve all three with **sharded SwissTables**, **4-bit Count-Min Sketch W-TinyLFU admission**, and **zero-GC `noscan` byte slabs**.

## 4-Bit Count-Min Sketch Admission Filter

```text
                4-Bit Count-Min Sketch (2 Counters per Byte, 0..15 Cap)
      Col 0   Col 1   Col 2   Col 3   Col 4   Col 5   Col 6   Col 7
Row 0 [ 0 ]   [ 3 ]   [12 ]   [ 0 ]   [ 1 ]   [ 0 ]   [ 7 ]   [ 0 ]  <-- h1(key) -> 12
Row 1 [ 1 ]   [ 0 ]   [ 0 ]   [ 9 ]   [ 0 ]   [ 2 ]   [ 0 ]   [ 0 ]  <-- h2(key) -> 9
Row 2 [ 0 ]   [ 0 ]   [ 0 ]   [ 0 ]   [ 0 ]   [14 ]   [ 0 ]   [ 1 ]  <-- h3(key) -> 14
Row 3 [ 2 ]   [ 9 ]   [ 0 ]   [ 0 ]   [ 0 ]   [ 0 ]   [ 0 ]   [ 0 ]  <-- h4(key) -> 9

Estimated Frequency of 'key' = min(12, 9, 14, 9) = 9
```

Every `W` additions, all 4-bit counters decay by 50% in parallel (16 counters per CPU instruction) via `word = (word >> 1) & 0x7777777777777777`. When full, a candidate `C` replaces victim `V` only if `Estimate(C) > Estimate(V)`.

## Zero-GC Sharded Byte-Slab Cache (`map[uint64]uint32` + `[]byte`)

```text
             Zero-GC Sharded Byte-Slab Cache Layout
+-----------------------------------------------------------------------+
| Index: map[uint64]uint32 (Key Hash -> 32-bit Ring Buffer Offset)      |
| -> Both K and V are pointer-free integers -> 'noscan' SwissTable      |
+-----------------------------------+-----------------------------------+
                                    | Offset: 1048
                                    v
+-----------------------------------------------------------------------+
| Data Slab: []byte (Contiguous 'noscan' Byte Ring Buffer)              |
| ... | [KeyLen: 2B][ValLen: 4B]["user:42"][Raw Protobuf/JSON Bytes] |  |
+-----------------------------------------------------------------------+
```

Because neither `map[uint64]uint32` nor `[]byte` contains pointers, a 50-million-entry cache costs **0.0 ms** of Green Tea GC mark scanning:

```go
package slabcache

import (
	"encoding/binary"
	"hash/maphash"
	"sync"
)

const numShards = 64

type shard struct {
	mu     sync.RWMutex
	index  map[uint64]uint32 // Pointer-free -> noscan SwissTable.
	slab   []byte            // Pointer-free -> noscan byte ring.
	cursor uint32
	_      [48]byte          // Pad to 64B cache-line boundary.
}

type ZeroGCCache struct {
	shards [numShards]shard
	seed   maphash.Seed
}

func NewZeroGCCache(slabBytesPerShard uint32, expectedKeysPerShard int) *ZeroGCCache {
	c := &ZeroGCCache{seed: maphash.MakeSeed()}
	for i := range c.shards {
		c.shards[i] = shard{
			index:  make(map[uint64]uint32, expectedKeysPerShard),
			slab:   make([]byte, slabBytesPerShard),
			cursor: 1, // 0 = empty sentinel.
		}
	}
	return c
}

func (c *ZeroGCCache) Set(key string, val []byte) {
	h := maphash.String(c.seed, key)
	s := &c.shards[h&(numShards-1)]
	entrySize := uint32(2 + 4 + len(key) + len(val))

	s.mu.Lock()
	defer s.mu.Unlock()
	if entrySize > uint32(len(s.slab))-1 {
		return
	}
	if s.cursor+entrySize > uint32(len(s.slab)) {
		s.cursor = 1
	}
	offset := s.cursor
	buf := s.slab[offset : offset+entrySize]
	binary.LittleEndian.PutUint16(buf[0:2], uint16(len(key)))
	binary.LittleEndian.PutUint32(buf[2:6], uint32(len(val)))
	copy(buf[6:], key)
	copy(buf[6+len(key):], val)
	s.index[h] = offset
	s.cursor += entrySize
}

func (c *ZeroGCCache) Get(key string, dst []byte) ([]byte, bool) {
	h := maphash.String(c.seed, key)
	s := &c.shards[h&(numShards-1)]

	s.mu.RLock()
	defer s.mu.RUnlock()
	offset, ok := s.index[h]
	if !ok || offset == 0 {
		return dst, false
	}
	buf := s.slab[offset:]
	keyLen := int(binary.LittleEndian.Uint16(buf[0:2]))
	valLen := int(binary.LittleEndian.Uint32(buf[2:6]))
	if string(buf[6:6+keyLen]) != key {
		return dst, false // Hash collision or ring wrap overwrite.
	}
	return append(dst, buf[6+keyLen:6+keyLen+valLen]...), true
}
```

When locking multiple shards in batch operations, always sort and deduplicate shard indices first to prevent AB-BA deadlocks, and verify with [gohawk](https://gohawk.dev/) (`lockorder`), [lensm](https://github.com/loov/lensm), and [benchstat](https://pkg.go.dev/golang.org/x/perf/cmd/benchstat).

## Related Chapters

- [188 Arena Allocators, Bump-Pointer Slabs, and Off-Heap Mmap](188-arena-allocators-bump-slabs-mmap.md)
- [189 String Interning, Weak Pointers, and Zero-Leak Caches](189-string-interning-weak-caches.md)
- [218 Swiss Maps and maphash](../25-go-deep-dives/218-swiss-maps-maphash.md)
