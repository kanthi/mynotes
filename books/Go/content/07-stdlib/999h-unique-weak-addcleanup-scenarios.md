---
title: Modern Identity and Lifecycle Scenarios — uuid, unique, weak, and runtime.AddCleanup
---

# Modern Identity and Lifecycle Scenarios: `uuid`, `unique`, `weak`, and `runtime.AddCleanup`

Between Go 1.23 and Go 1.27, the standard library added four cohesive primitives for identity, value deduplication, and memory-pressure-aware resource lifecycles:

- **`uuid` (Go 1.27):** Stack-allocated `[16]byte` UUID generation (backed by `crypto/rand` / Linux vDSO `getrandom`) and parsing with `TextAppender`/`BinaryAppender`.
- **`unique.Handle[T]` (Go 1.23 / parallel GC in Go 1.25):** Interns any comparable value into an 8-byte handle with `O(1)` pointer equality.
- **`weak.Pointer[T]` (Go 1.24):** References heap objects via an 8-byte runtime indirection word without preventing GC.
- **`runtime.AddCleanup` (Go 1.24 / parallel execution in Go 1.25):** Non-resurrecting, cycle-safe replacement for `runtime.SetFinalizer`.

## Scenario 1: Request Tracing and Idempotency Keys (`uuid`)

```go
package main

import (
	"fmt"
	"uuid"
)

func AssignRequestID(incomingHeader string) (string, error) {
	if incomingHeader != "" {
		parsed, err := uuid.Parse(incomingHeader)
		if err != nil {
			return "", fmt.Errorf("invalid X-Request-ID %q: %w", incomingHeader, err)
		}
		return parsed.String(), nil
	}
	return uuid.New().String(), nil
}
```

## Scenario 2: Telemetry Label Deduplication (`unique.Handle[T]`)

```go
package main

import (
	"fmt"
	"unique"
)

type MetricScope struct {
	Region, ServiceName, Environment string
}

type DataPoint struct {
	Scope     unique.Handle[MetricScope]
	LatencyMs float64
}

func main() {
	s1 := unique.Make(MetricScope{"ap-south-1", "payment-engine", "prod"})
	s2 := unique.Make(MetricScope{"ap-south-1", "payment-engine", "prod"})
	fmt.Println("O(1) handle equality:", s1 == s2, "service:", s1.Value().ServiceName)
}
```

## Scenario 3: Weak WebSocket Subscriber Registry (Zero Disconnected Leaks)

```go
package main

import (
	"sync"
	"weak"
)

type Session struct {
	ClientID string
	Inbox    chan string
}

type Broadcaster struct {
	mu   sync.Mutex
	subs []weak.Pointer[Session]
}

func (b *Broadcaster) Subscribe(s *Session) {
	b.mu.Lock()
	defer b.mu.Unlock()
	b.subs = append(b.subs, weak.Make(s))
}

func (b *Broadcaster) Broadcast(msg string) {
	b.mu.Lock()
	defer b.mu.Unlock()
	active := b.subs[:0]
	for _, wp := range b.subs {
		if s := wp.Value(); s != nil {
			select {
			case s.Inbox <- msg:
			default:
			}
			active = append(active, wp)
		}
	}
	b.subs = active
}
```

> [!TIP]
> Audit `runtime.AddCleanup` registrations in staging/CI with `GODEBUG=checkfinalizers=1` (Go 1.25+) to catch any cleanup callback that accidentally retains a strong pointer to the tracked object.

## Related Chapters

- [999f uuid](999f-uuid.md)
- [217 weak and unique Packages](../25-go-deep-dives/217-weak-unique-packages.md)
- [189 String Interning, Weak Pointers, and Zero-Leak Caches](../24-performance-engineering/189-string-interning-weak-caches.md)
