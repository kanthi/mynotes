---
title: Timer Heap Architecture, 4-ary Min-Heaps, and Synchronous Timer Channels
---

# Timer Heap Architecture, 4-ary Min-Heaps, and Synchronous Timer Channels

Every network microservice creates, resets, and cancels thousands of timers per second via `context.WithTimeout`, `http.Client.Timeout`, `conn.SetDeadline`, `time.After`, and `time.NewTicker`. The Go runtime implements this in `src/runtime/time.go`, tightly integrated with the GMP scheduler, the channel engine (`hchan.timer`), and the network poller (`netpollBreak`).

## Four Generations of Go Timers

```text
Gen 1 (Go 1.0–1.9):   Single global mutex + 1 'timerproc' goroutine
                      -> Severe lock contention beyond 8 CPU cores

Gen 2 (Go 1.10–1.13): 64 sharded timer buckets + 64 'timerproc' goroutines
                      -> Lower lock contention, costly goroutine wakeups

Gen 3 (Go 1.14–1.22): Per-P 4-ary min-heaps (p.timers) + netpoll timeout integration
                      -> Zero 'timerproc' goroutines; lazy state bitmask transitions

Gen 4 (Go 1.23–1.27): Synchronous unbuffered timer channels (hchan.timer) +
                      lazy heap insertion (maybeAdd) + immediate GC of unstopped timers
                      (Legacy GODEBUG=asynctimerchan permanently removed in Go 1.27)
```

## Why Go Uses a Cache-Line-Aligned 4-ary Min-Heap (`timerWhen`)

Each logical processor `P` embeds a `timers` 4-ary min-heap storing 16-byte `timerWhen` structs inline (`{timer *timer; when int64}`):

```text
                 4-ary Min-Heap Layout in Contiguous Slice ([]timerWhen)
                       Index 0 (Root: Earliest Deadline 'when')
                    /              |                 |              \
               Index 1          Index 2           Index 3          Index 4
          [ timer, when ]   [ timer, when ]   [ timer, when ]   [ timer, when ]
          |<------------ Exactly 64 Bytes (1 Full CPU L1 Cache Line) -------->|
```

1. **Exact 64-byte L1 cache-line fit:** All four children of node `i` sit at `4*i + 1` through `4*i + 4`. Four 16-byte `timerWhen` entries occupy one exact 64-byte cache line.
2. **Zero pointer chasing during `siftDownTimer`:** Comparing all four sibling `when` timestamps reads inline `int64` values inside that single cache line without dereferencing `*timer` pointers.
3. **Halved tree depth:** 65,536 timers have depth `log4(65536) = 8` instead of `16`.

## Lazy State Bitmasks: Lock-Free `Stop()` and `Reset()`

Because 99.9% of network timeouts are canceled or reset before firing—often from a different `P`—`timer.Stop()` and `timer.Modify(newWhen)` do **not** lock the foreign `p.timers` heap immediately. Instead:

- `Stop()` sets `t.when = 0`, marks `timerModified`, and increments `ts.zombies`.
- `Modify(newWhen)` updates `t.when` under the timer's small per-instance lock `t.mu`. Only if `newWhen` is earlier than `ts.minWhenModified` does it call `netpollBreak()` to wake a sleeping `epoll_wait` thread early.
- Actual heap sifting (`cleanHead` / `updateHeap`) is amortized when the owning `P` runs `checkTimers()` in `runtime.schedule()` or when `zombies` exceeds 25% of heap length.

## Go 1.23–1.27 Synchronous Timer Channels (`hchan.timer` and `maybeAdd`)

```text
          Go 1.27 Synchronous Timer Channel (hchan.timer)
+-------------------------------------------------------------------------------+
| 1. t := time.NewTimer(5 * time.Second)                                        |
|    - Allocates unbuffered hchan (dataqsiz = 0) with c.timer = t               |
|    - Timer is NOT inserted into p.timers heap yet (zero heap overhead)        |
+---------------------------------------+---------------------------------------+
                                        |
      +---------------------------------+---------------------------------+
      | (Case A: Request finishes in 2ms; t dropped)                      | (Case B: G blocks on <-t.C)
      v                                                                   v
+-----------------------------------------+   +-----------------------------------------+
| - Timer NEVER entered p.timers heap     |   | - chanrecv / select calls t.maybeAdd()  |
| - Green Tea GC collects t immediately   |   | - Inserts t into P's 4-ary timer heap   |
|   even if t.Stop() was NEVER called     |   | - When t fires, sendChan wakes G        |
+-----------------------------------------+   +-----------------------------------------+
```

> [!TIP]
> **Delete legacy channel-draining boilerplate:** Because `cap(t.C) == 0` and `GODEBUG=asynctimerchan` is removed in Go 1.27, calling `t.Stop()` or `t.Reset(d)` immediately invalidates any prior tick—`if !t.Stop() { <-t.C }` is no longer needed. Still prefer reusing a single `time.NewTimer` + `Reset` inside tight loops to avoid per-iteration heap allocations (`0 B/op`).

## Runnable Cache-Aligned 4-ary Min-Heap

```go
package quadheap

// Entry mirrors runtime.timerWhen (16 bytes: 4 entries = 64B CPU cache line).
type Entry[T any] struct {
	When int64
	Task *T
}

type QuadHeap[T any] struct {
	nodes []Entry[T]
}

func NewQuadHeap[T any](capHint int) *QuadHeap[T] {
	return &QuadHeap[T]{nodes: make([]Entry[T], 0, capHint)}
}

func (h *QuadHeap[T]) Len() int { return len(h.nodes) }

func (h *QuadHeap[T]) Push(when int64, task *T) {
	i := len(h.nodes)
	h.nodes = append(h.nodes, Entry[T]{When: when, Task: task})
	h.siftUp(i)
}

func (h *QuadHeap[T]) PopMin() (Entry[T], bool) {
	n := len(h.nodes)
	if n == 0 {
		return Entry[T]{}, false
	}
	minEntry := h.nodes[0]
	last := h.nodes[n-1]
	h.nodes[n-1] = Entry[T]{} // Clear pointer for Green Tea GC.
	h.nodes = h.nodes[:n-1]

	if n > 1 {
		h.nodes[0] = last
		h.siftDown(0)
	}
	return minEntry, true
}

func (h *QuadHeap[T]) siftUp(i int) {
	nodes := h.nodes
	tmp := nodes[i]
	when := tmp.When
	for i > 0 {
		parent := int(uint(i-1) >> 2) // Unsigned shift proves parent < i for BCE.
		if when >= nodes[parent].When {
			break
		}
		nodes[i] = nodes[parent]
		i = parent
	}
	nodes[i] = tmp
}

func (h *QuadHeap[T]) siftDown(i int) {
	nodes := h.nodes
	n := len(nodes)
	tmp := nodes[i]
	when := tmp.When
	for {
		firstChild := (i << 2) + 1
		if firstChild >= n {
			break
		}
		minChild := firstChild
		minWhen := nodes[firstChild].When
		if firstChild+3 < n {
			_ = nodes[firstChild+3] // Front-load bounds check for all 4 siblings.
			if w1 := nodes[firstChild+1].When; w1 < minWhen {
				minWhen, minChild = w1, firstChild+1
			}
			if w2 := nodes[firstChild+2].When; w2 < minWhen {
				minWhen, minChild = w2, firstChild+2
			}
			if w3 := nodes[firstChild+3].When; w3 < minWhen {
				minWhen, minChild = w3, firstChild+3
			}
		} else {
			for c := firstChild + 1; c < n; c++ {
				if w := nodes[c].When; w < minWhen {
					minWhen, minChild = w, c
				}
			}
		}
		if minWhen >= when {
			break
		}
		nodes[i] = nodes[minChild]
		i = minChild
	}
	nodes[i] = tmp
}
```

## Verification with `testing/synctest`, `lensm`, and `gohawk`

- **`testing/synctest` bubble timers:** Inside `synctest.Test(t, fn)`, timers route to `bubble.timers` instead of `p.timers`; once all goroutines durably block, `synctest.Wait()` and Go 1.27's `synctest.Sleep()` advance the fake clock instantaneously.
- **Bounds-check verification:** Inspect `siftDown` with `go build -gcflags="-d=ssa/check_bce"` or [lensm](https://github.com/loov/lensm) (`lensm -filter=siftDown`).
- **`time.AfterFunc` concurrency hazards:** Unlike `NewTimer`, `time.AfterFunc(d, fn)` runs `fn` on a new goroutine; if `t.Stop()` returns `false`, `fn` may already be executing concurrently. Run [gohawk](https://gohawk.dev/) (`go tool gohawk ./...`) to catch `concurrentcapture` and `lockorder` races.

## Related Chapters

- [203 Channels and Select Internals](203-channels-select-internals.md)
- [223 testing/synctest Bubbles](223-testing-synctest-bubbles.md)
- [213 Netpoller Network I/O](213-netpoller-network-io.md)
