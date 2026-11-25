---
title: '19. Fault Tolerance'
---

# Fault Tolerance

In production, external APIs go down, databases fail over, and malformed messages slip through. A fault-tolerant event-driven service combines **retries**, **circuit breakers**, and **poison queues** so transient failures recover automatically, prolonged outages do not cause cascading collapse, and broken messages never block healthy traffic.

## Learning & Build Goals

- **Learn:** Availability and fault-tolerance strategies for async systems; how retries, circuit breakers, and poison queues complement each other.
- **Build:** Implement bounded retries for transient failures; add circuit breakers around unstable dependencies; route unrecoverable messages to a poison queue (dead-letter queue).

## Defense in Depth: Middleware Stack

```text
Incoming Message
  -> PoisonQueue Middleware (catches final failure after retries -> publishes to poison topic + Acks)
       -> Retry Middleware (exponential backoff for transient errors)
            -> Circuit Breaker (fails fast when downstream is unhealthy)
                 -> Handler -> External API / DB
```

### How the Three Mechanisms Work Together

| Mechanism | Problem Solved | Behavior |
|-----------|----------------|----------|
| **Retry (with backoff + jitter)** | Brief network blips, transient 503s | Re-attempts N times with increasing delay |
| **Circuit Breaker** | Prolonged downstream outage | Opens after consecutive failures; fails fast without hammering the struggling dependency; probes periodically in half-open state |
| **Poison Queue (DLQ)** | Malformed messages, persistent bugs, exhausted retries | Moves failing message + error metadata to a dedicated `poison_queue` topic and `Ack`s the original so the partition stays unblocked |

## Wiring Poison Queue and Circuit Breaker in Watermill

```go
package resilience

import (
	"fmt"
	"time"

	"github.com/ThreeDotsLabs/watermill/message"
	"github.com/ThreeDotsLabs/watermill/message/router/middleware"
	"github.com/sony/gobreaker"
)

func RegisterFaultTolerance(router *message.Router, pub message.Publisher) error {
	poisonQueue, err := middleware.PoisonQueue(pub, "poison_queue")
	if err != nil {
		return fmt.Errorf("create poison queue middleware: %w", err)
	}

	retry := middleware.Retry{
		MaxRetries:          5,
		InitialInterval:     200 * time.Millisecond,
		MaxInterval:         5 * time.Second,
		Multiplier:          2,
		RandomizationFactor: 0.2,
	}

	// Order matters: PoisonQueue must wrap Retry so it only fires after retries are exhausted.
	router.AddMiddleware(
		middleware.Recoverer,
		poisonQueue,
		retry.Middleware,
	)
	return nil
}

// NewExternalAPIBreaker protects an outbound HTTP client from cascading failures.
func NewExternalAPIBreaker(name string) *gobreaker.CircuitBreaker {
	return gobreaker.NewCircuitBreaker(gobreaker.Settings{
		Name:        name,
		MaxRequests: 3,                // allowed probes in half-open state
		Interval:    30 * time.Second, // window to clear counts while closed
		Timeout:     15 * time.Second, // stay open before testing half-open
		ReadyToTrip: func(counts gobreaker.Counts) bool {
			return counts.ConsecutiveFailures >= 5
		},
	})
}
```

## Operating a Poison Queue

A poison queue is only useful if you have an operational workflow for it:
1. **Alert** whenever a message enters `poison_queue`.
2. **Inspect** the attached metadata (`reason_poisoned`, `topic_poisoned`, `handler_poisoned`, `correlation_id`).
3. **Fix** the root cause (deploy bug fix or patch payload).
4. **Replay** poison messages back to their original topic using a controlled CLI or admin endpoint.

## Related Chapters

- Previous: [18. Tracing](18-tracing.md)
- Next: [20. Sagas and Process Managers](20-sagas-and-process-managers.md)
- Book foundations: [Retry and Circuit Breaker Patterns](../18-distributed-infra/152-retry-circuit-breaker-patterns.md)
