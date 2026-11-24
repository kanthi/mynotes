---
title: String, Slice, and UTF-8 Internals (nextslicecap and Copy Elision)
---

# String, Slice, and UTF-8 Internals (`nextslicecap` and Copy Elision)

Understanding how `runtime/slice.go`, `runtime/string.go`, and `cmd/compile` manage strings and slices eliminates silent 64 MB memory-pinning leaks, shared-capacity data races, and unnecessary heap copies.

## Memory Layouts: `string` (16 Bytes) vs. `[]T` (24 Bytes)

```text
string (16 Bytes on 64-bit)                     []T Slice (24 Bytes on 64-bit)
+---------------------------------------+       +---------------------------------------+
| str: unsafe.Pointer (8B) -> [ReadOnly]|       | array: unsafe.Pointer (8B) -> [Heap/  |
+---------------------------------------+       |                                Stack] |
| len: int            (8B)              |       +---------------------------------------+
+---------------------------------------+       | len:   int            (8B)            |
                                                +---------------------------------------+
                                                | cap:   int            (8B)            |
                                                +---------------------------------------+
```

String literals live in `.rodata` (not null-terminated; writing via `unsafe` triggers `SIGSEGV`). Passing a slice by value copies only the 24-byte header, sharing the backing array.

## How `runtime.growslice` and `nextslicecap` Grow Slices

Modern Go (`src/runtime/slice.go`) replaced the old abrupt `2x -> 1.25x` cliff at 1,024 with a smooth asymptotic curve starting at threshold `256`, followed by `roundupsize(capmem)` aligning the byte size to Go's 68 TCMalloc size classes:

```go
func nextslicecap(newLen, oldCap int) int {
	newcap := oldCap
	doublecap := newcap + newcap
	if newLen > doublecap {
		return newLen
	}
	const threshold = 256
	if oldCap < threshold {
		return doublecap
	}
	for {
		// Smoothly transitions from 2x down toward 1.25x as capacity grows.
		newcap += (newcap + 3*threshold) >> 2
		if uint(newcap) >= uint(newLen) {
			break
		}
	}
	return newcap
}
```

In Go 1.25–1.27, non-escaping `make([]T, n)` with a runtime variable `n` can also use a bounded stack scratch buffer (`-compile=variablemake`), avoiding heap allocation altogether.

## Five Compiler Copy-Elision Fast Paths (`0 B/op`)

`cmd/compile` elides the `[]byte <-> string` copy whenever the conversion is proven read-only and non-escaping:

| Go Code Pattern | What `cmd/compile` Emits | Heap Allocations |
|---|---|---|
| `val, ok := m[string(b)]` | Stack string header passed to SwissTable `mapaccess2_faststr` | **0 B/op** |
| `string(b) == "GET"` | `runtime.memequal` directly on `b`'s bytes vs. `.rodata` | **0 B/op** |
| `switch string(b) { case "GET": ... }` | Compares directly against case constants | **0 B/op** |
| `s := "<" + string(b) + ">"` | `runtime.concatstrings` reads `b` directly | **1 alloc** (final result only) |
| `for i, r := range []byte(s)` | Decodes UTF-8 runes directly from `s` | **0 B/op** |

> [!WARNING]
> Storing `k := string(b)` in a named variable before `m[k]` defeats copy elision if `k` can be observed elsewhere. Keep `string(b)` inline inside `m[string(b)]`.

## Inside `strings.Builder` and `abi.NoEscape`

`strings.Builder` stores a self-pointer `addr *Builder` via `abi.NoEscape` (so taking `&b` doesn't force heap escape) to panic if a non-empty `Builder` is copied by value, and finalizes `String()` in `O(1)` with zero copy via `unsafe.String(unsafe.SliceData(b.buf), len(b.buf))`.

## UTF-8 Fast Paths and Go 1.24–1.27 String Upgrades

- **Inlined ASCII fast path:** `for i, r := range s` tests `b < 0x80` inline and only calls `runtime.decoderune` on multi-byte lead bytes.
- **Go 1.27 single-allocation ASCII `ToLower` / `ToUpper`:** Scans first, returning the original string (`0 allocs`) when already converted, or allocating at most once.
- **Go 1.27 `strings.CutLast` / `bytes.CutLast`:** Splits cleanly on the final separator (`dir, file, ok := strings.CutLast(path, "/")`).
- **Go 1.24 `strings.Lines`, `SplitSeq`, `FieldsSeq`:** Zero-slice-allocation `iter.Seq[string]` iterators.

## Fixing Memory Pinning and Shared-Capacity Races

```go
package strslice

import (
	"slices"
	"strings"
)

// Release the parent 64 MB buffer before caching a 16-byte token.
func ExtractToken(hugePayload []byte) string {
	return strings.Clone(string(hugePayload[100:116]))
}

// Clip spare capacity before returning a sub-slice across API boundaries.
func SafeTrimAndClip(buf []byte) []byte {
	for len(buf) > 0 && buf[len(buf)-1] == '\n' {
		buf = buf[:len(buf)-1]
	}
	return slices.Clip(buf)
}

//noalloc:root
func BuildHeaderLine(key, val string) string {
	var sb strings.Builder
	sb.Grow(len(key) + 2 + len(val))
	sb.WriteString(key)
	sb.WriteString(": ")
	sb.WriteString(val)
	return sb.String()
}
```

Verify copy elision with `go test -bench=. -benchmem` (`for b.Loop()`), `go build -gcflags="-m"`, `noalloc`, and [lensm](https://github.com/loov/lensm); catch concurrent unclipped slice appends with [gohawk](https://gohawk.dev/) and `-race`.

## Related Chapters

- [205 Slice and Map Internals](205-slice-map-internals.md)
- [247 Memory Allocator Deep Dive](247-memory-allocator-deep-dive.md)
- [254 Race Detector Internals](254-race-detector-internals.md)
