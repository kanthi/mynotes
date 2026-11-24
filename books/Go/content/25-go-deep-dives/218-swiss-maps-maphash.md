---
title: Swiss Maps, Control Bytes, Extendible Hashing, and maphash (Go 1.24–1.27)
---

# Swiss Maps, Control Bytes, Extendible Hashing, and `maphash` (Go 1.24–1.27)

Starting in Go 1.24 and refined through Go 1.27, Go replaced its legacy `hmap`/`bmap` chained-bucket map (`runtime/map.go`) with a **SwissTable** design (`internal/runtime/maps`) paired with **extendible hashing** for bounded incremental growth, and rewrote `sync.Map` as a concurrent **`HashTrieMap`**.

## SwissTable Architecture and Extendible Hashing Directory

```text
                      Map Header (internal/runtime/maps.Map)
+-------------------------------------------------------------------------------+
|  used: uint64 (total live elements) | seed: uintptr (per-map hash seed)       |
|  dirPtr: unsafe.Pointer             | dirLen: int (directory size)            |
|  globalDepth: uint8                 | clearSeq: uint64 (iteration safety)     |
+-------------------------------------+-----------------------------------------+
                                      |
            +-------------------------+-------------------------+
            | (<= 8 elements: Inline Single Group)              | (> 8 elements: Directory of Tables)
            v                                                   v
 +---------------------+                             +---------------------+
 | Single Group (8 K/V)|                             | Directory [Table*]  |
 +---------------------+                             +----------+----------+
                                                                |
                                      +-------------------------+-------------------------+
                                      v                                                   v
                           +---------------------+                             +---------------------+
                           | Table 0 (<= 1024)   |                             | Table 1 (<= 1024)   |
                           | Groups of 8 slots   |                             | Groups of 8 slots   |
                           +---------------------+                             +---------------------+
```

Each `Table` is capped at **1,024 elements** (128 groups). When a table exceeds its `7/8` (87.5%) load factor, **only that single 1,024-element table splits into two tables**—eliminating the tail-latency spikes and giant contiguous allocations caused by legacy all-at-once table doubling.

## Group Layout and 64-Bit Control Word (`ctrlGroup`)

```text
                                 SwissTable Group (8 Slots)
+---------------------------------------------------------------------------------------+
| Control Word (ctrlGroup: uint64 — 8 control bytes: c0, c1, c2, c3, c4, c5, c6, c7)    |
+---------------------------------------------------------------------------------------+
| Slot 0 (Key 0, Val 0) | Slot 1 (Key 1, Val 1) | ... | Slot 7 (Key 7, Val 7)           |
+---------------------------------------------------------------------------------------+
```

| Control Byte Value | Binary | Meaning |
|---|---|---|
| `ctrlEmpty` | `0b10000000` (`0x80`) | Empty slot (probe stops if found) |
| `ctrlDeleted` | `0b11111110` (`0xFE`) | Tombstone (previously occupied, probe continues) |
| Full (`h2` fingerprint) | `0b0xxxxxxx` (`0x00`–`0x7F`) | Occupied slot; lower 7 bits store `h2(hash)` (eliminates 99.2% of key compares) |

## Parallel 8-Slot Probing via SWAR and SIMD

Instead of looping over 8 slots sequentially, Go loads all 8 control bytes into a single `uint64` register and matches all 8 slots in ~4 CPU instructions using SWAR (SIMD Within A Register) or hardware SIMD, then jumps to matching indices via hardware `TrailingZeros64`:

```go
const (
	bitsetLSB = 0x0101010101010101
	bitsetMSB = 0x8080808080808080
)

func matchH2(ctrl uint64, h2 uint8) uint64 {
	v := ctrl ^ (bitsetLSB * uint64(h2))
	return ((v - bitsetLSB) &^ v) & bitsetMSB
}
```

## Concurrent `sync.Map` (`HashTrieMap`) and Go 1.24–1.27 `hash/maphash`

- **`sync.Map` (`internal/sync/hashtriemap.go`, Go 1.24+):** Implemented as a concurrent 16-way Hash Array Mapped Trie (HAMT) with fine-grained leaf locking—disjoint key writes never contend on a global `dirty` lock, and there is zero warm-up ramp for concurrent reads.
- **`hash/maphash.Comparable` (Go 1.24+) and `maphash.Hasher[T]` (Go 1.27+):** Expose the same hardware-accelerated AES/wyhash functions to custom containers without reflection:

```go
package main

import (
	"fmt"
	"hash/maphash"
)

type UserKey struct {
	TenantID int
	Username string
}

func main() {
	seed := maphash.MakeSeed()
	h1 := maphash.Comparable(seed, UserKey{TenantID: 42, Username: "gopher"})
	fmt.Printf("maphash.Comparable: 0x%x\n", h1)

	var hasher maphash.ComparableHasher[UserKey]
	var h maphash.Hash
	h.SetSeed(seed)
	hasher.Hash(&h, UserKey{TenantID: 42, Username: "gopher"})
	fmt.Printf("Go 1.27 Hasher:     0x%x\n", h.Sum64())
}
```

> [!TIP]
> Pre-size bulk maps with `make(map[K]V, hint)` to avoid intermediate table splits, replace drastically shrunk maps (`m = make(map[K]V)`) to reclaim directory/tombstone memory, and prefer pointer-free keys/values (`map[uint64]uint64`) so SwissTable groups allocate in `noscan` spans (0 ms Green Tea GC scanning).

## Related Chapters

- [205 Slice and Map Internals](205-slice-map-internals.md)
- [999g maphash.Hasher](../07-stdlib/999g-maphash-hasher.md)
- [247 Memory Allocator Deep Dive](247-memory-allocator-deep-dive.md)
