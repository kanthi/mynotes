---
title: Lock-Free Ring Buffers, Bounded MPMC Queues, and the Disruptor Pattern
---

# Lock-Free Ring Buffers, Bounded MPMC Queues, and the Disruptor Pattern

In trading engines, packet processors, and high-rate telemetry pipelines handling tens of millions of events/sec, standard Go channels (`chan T`) hit a ceiling because every send/receive acquires `hchan.lock`. **Dmitry Vyukov's bounded MPMC ring buffer** eliminates the shared mutex using per-slot sequence counters and 64-byte cache-line isolation, achieving ~13x higher throughput (`~14 ns/op` vs. `~185 ns/op`) with zero heap allocations.

## Channel Mutex vs. Sequence-Indexed Ring Buffer

```text
             Go Channel (chan T)               Bounded Lock-Free Ring Buffer
    +-----------------------------------+   +-----------------------------------+
    | Producers -> [ hchan.lock ]       |   | Producers -> Atomic CAS Pos       |
    |              [ Circular Buffer  ] |   |              (No Mutex)           |
    | Consumers <- [ hchan.lock ]       |   | Consumers <- Per-Slot Sequence    |
    +-----------------------------------+   +-----------------------------------+
      Contention on shared mutex               Independent slot sequence numbers
      Causes futex sleep / gopark              Zero kernel transitions
```

- **Power-of-two capacity (`mask = cap - 1`):** Replaces 20-cycle modulo division with a 1-cycle bitwise `pos & mask`.
- **Per-slot `sequence atomic.Uint64`:** Initialized to slot index `i`. Producers claim a slot when `seq - pos == 0` and publish `seq = pos + 1`; consumers claim when `seq - (pos + 1) == 0` and release `seq = pos + mask + 1`.
- **Zero popped references:** Clearing `s.value = zero` lets the Green Tea GC reclaim pointer payloads immediately.

## Complete Bounded MPMC Ring Buffer in Go 1.27

```go
package ringbuffer

import (
	"math/bits"
	"runtime"
	"sync/atomic"
)

type slot[T any] struct {
	sequence atomic.Uint64
	value    T
}

// MPMCQueue is a bounded, lock-free multi-producer multi-consumer ring buffer.
type MPMCQueue[T any] struct {
	_          [56]byte
	enqueuePos atomic.Uint64
	_          [56]byte
	dequeuePos atomic.Uint64
	_          [56]byte
	mask       uint64
	buffer     []slot[T]
}

func NewMPMCQueue[T any](capacity uint64) *MPMCQueue[T] {
	if capacity < 2 {
		capacity = 2
	}
	if capacity&(capacity-1) != 0 {
		capacity = 1 << (64 - bits.LeadingZeros64(capacity))
	}
	q := &MPMCQueue[T]{
		mask:   capacity - 1,
		buffer: make([]slot[T], capacity),
	}
	for i := range capacity {
		q.buffer[i].sequence.Store(i)
	}
	return q
}

func (q *MPMCQueue[T]) Push(item T) bool {
	var backoff int
	for {
		pos := q.enqueuePos.Load()
		s := &q.buffer[pos&q.mask]
		diff := int64(s.sequence.Load()) - int64(pos)
		if diff == 0 {
			if q.enqueuePos.CompareAndSwap(pos, pos+1) {
				s.value = item
				s.sequence.Store(pos + 1)
				return true
			}
		} else if diff < 0 {
			return false // Full.
		} else if backoff++; backoff > 16 {
			runtime.Gosched()
			backoff = 0
		}
	}
}

func (q *MPMCQueue[T]) Pop() (T, bool) {
	var zero T
	var backoff int
	for {
		pos := q.dequeuePos.Load()
		s := &q.buffer[pos&q.mask]
		diff := int64(s.sequence.Load()) - int64(pos+1)
		if diff == 0 {
			if q.dequeuePos.CompareAndSwap(pos, pos+1) {
				val := s.value
				s.value = zero // Release pointer for GC.
				s.sequence.Store(pos + q.mask + 1)
				return val, true
			}
		} else if diff < 0 {
			return zero, false // Empty.
		} else if backoff++; backoff > 16 {
			runtime.Gosched()
			backoff = 0
		}
	}
}
```

## The LMAX Disruptor Pattern in Go

```text
                                LMAX Disruptor Ring
               +---------------------------------------------------+
               | Pre-Allocated Mutable Event Slots (Zero-Alloc GC) |
               +-------------------------+-------------------------+
                                         |
                  +----------------------+----------------------+
                  v                                             v
      [ Consumer 1: Journaler ]                     [ Consumer 2: Replicator ]
        Cursor: Seq 42                                Cursor: Seq 42
                  \                                             /
                   +---------------------+---------------------+
                                         v
                            [ Consumer 3: Business Logic ]
                              (Gated on min(C1, C2) >= Seq)
```

Pre-allocate all event structs once at startup (`make([]OrderEvent, 65536)`), mutate slots in place, and let multiple consumers read the same slot with independent atomic sequence cursors.

## Related Chapters

- [45 Lock-Free Atomics and Queues](45-lock-free-atomics-and-queues.md)
- [203 Channels and Select Internals](../25-go-deep-dives/203-channels-select-internals.md)
