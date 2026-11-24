---
title: Zero-Allocation Iterators, Coroutine iter.Pull Internals, and Generic Stenciling
---

# Zero-Allocation Iterators, Coroutine `iter.Pull` Internals, and Generic Stenciling

Go 1.23–1.27's `iter.Seq` / `iter.Pull` iterators and generics (including Go 1.27 generic methods) can run at bare-metal loop speed—or trigger hidden heap escapes and `.dict` indirect calls—depending on how `cmd/compile` lowers them.

## Push Iterators (`iter.Seq`) vs. Pull Coroutines (`iter.Pull`)

- **Push iterators (`cmd/compile/internal/rangefunc`):** The compiler rewrites `for x := range seq` into a synthesized `yield` closure. When both the iterator and `yield` body fit the 80-point inlining budget (or PGO hot-edge budget), the pipeline compiles into a raw `JMP` loop (`0 B/op`, identical to a `for` slice loop).
- **Pull iterators (`src/runtime/coro.go`):** `iter.Pull(seq)` uses switch-on-demand runtime coroutines (`runtime.coroswitch`) that swap `G_caller` and `G_coro` **directly on the same OS thread `M` and processor `P`** without touching `p.runq` or `runtime.schedule()` (~12–18 ns vs. ~168 ns for a channel handoff).

```text
Caller Goroutine (G_caller on M0 / P0)            Pull Coroutine (G_coro on SAME M0 / P0)
+-------------------------------------------+     +-------------------------------------------+
| Calls next()                              |     |                                           |
|   -> runtime.coroswitch(c) ===============+===> | Awakens immediately on M0 / P0            |
|   (G_caller -> _Gwaiting, no scheduler)   |     | Runs seq until yield(v)                   |
|                                           | <===+=== Calls runtime.coroswitch(c)            |
| Awakens immediately on M0 / P0 with 'v'   |     |   (G_coro -> _Gwaiting, stays on M0 / P0) |
+-------------------------------------------+     +-------------------------------------------+
```

```go
package iterators

import "iter"

// Zip combines two push iterators in lockstep via iter.Pull.
func Zip[A, B any](seqA iter.Seq[A], seqB iter.Seq[B]) iter.Seq2[A, B] {
	return func(yield func(A, B) bool) {
		nextB, stopB := iter.Pull(seqB)
		defer stopB() // Mandatory: unwinds suspended coroutine on early break/error!
		for a := range seqA {
			b, ok := nextB()
			if !ok || !yield(a, b) {
				return
			}
		}
	}
}
```

## Generic GCShape Stenciling + `.dict` Runtime Dictionaries

Go groups generic type instantiations by **GCShape** (size, alignment, and pointer bitmap):

| Instantiation Pattern | Compiled Representation | Speed vs. Concrete Code |
|---|---|---|
| Value types (`int64`, `float64`, `Point`) | Dedicated stencil (`go.shape.int64`) | **100% identical** |
| Slices of values (`[]int64`) | Dedicated stencil, constant stride | **100% identical** |
| Pointer types calling interface methods (`*User`) | Shared `go.shape.*uint8` + hidden `.dict` parameter | **10%–25% slower** unless inlined / PGO |

> [!TIP]
> Default to push iterators (`iter.Seq`) for single-stream pipelines, reserve `iter.Pull` for multi-stream lockstep zip/merge (always paired with `defer stop()`), enforce zero-alloc loops with `noalloc` (`//noalloc:root`), inspect `go.shape.*` stencils in [lensm](https://github.com/loov/lensm), and verify `stop()` cleanup on every branch with [gohawk](https://gohawk.dev/).

## Related Chapters

- [229 Iterators and range-over-func](../25-go-deep-dives/229-iterators-range-over-func.md)
- [207 Generics Implementation](../25-go-deep-dives/207-generics-implementation.md)
- [253 Compiler IR, Escape Analysis, and SSA Backend](../25-go-deep-dives/253-compiler-ir-ssa-backend.md)
