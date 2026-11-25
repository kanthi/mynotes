---
title: '01. Basic Async'
---

# Basic Async

Moving slow or failure-prone work off the synchronous HTTP request path is the first step toward resilient backend services. Before introducing a message broker, you can explore the trade-offs of asynchronous execution using Go's native concurrency primitives.

## Learning & Build Goals

- **Learn:** Synchronous vs. asynchronous communication patterns; benefits and challenges of async processing under high load; basic async execution with goroutines and retries.
- **Build:** Refactor synchronous API calls in the ticket aggregator to asynchronous background workers; handle transient errors with retries.

## Sync vs. Async Communication

In a synchronous request flow, the HTTP handler blocks until every downstream side effect (sending a confirmation email, calling a spreadsheet/analytics API, generating a receipt) finishes. One slow or flaky dependency degrades the entire API's latency and availability.

```text
Synchronous:
  Client -> HTTP Handler -> Save Ticket -> Send Email -> Update CRM -> 200 OK
                            (if Email times out, the whole request fails)

Asynchronous:
  Client -> HTTP Handler -> Enqueue Job -> 202 Accepted
                                |
                                v
                         Background Worker -> Send Email (with retries)
```

### Trade-offs

| Dimension | Synchronous | Asynchronous (In-Process) |
|-----------|-------------|---------------------------|
| **Client latency** | Sum of all downstream calls | Fast acknowledgement (`202 Accepted`) |
| **Fault isolation** | Downstream outage breaks caller | Caller succeeds; worker retries later |
| **Durability** | Caller knows immediately if it failed | **Risk:** in-memory queue is lost on process crash |
| **Complexity** | Simple linear control flow | Needs worker lifecycle, backpressure, and retry logic |

## Implementation Pattern: Bounded Worker with Retries

Avoid spawning an unbounded `go sendReceipt(ticket)` per request—under traffic spikes, unbounded goroutines exhaust memory and overwhelm downstream APIs. Use a buffered channel and a fixed worker loop with context cancellation.

```go
package worker

import (
	"context"
	"log/slog"
	"time"
)

type Job struct {
	TicketID string
	Email    string
}

type Worker struct {
	queue chan Job
	send  func(ctx context.Context, job Job) error
}

func New(buffer int, send func(context.Context, Job) error) *Worker {
	return &Worker{
		queue: make(chan Job, buffer),
		send:  send,
	}
}

func (w *Worker) Enqueue(job Job) bool {
	select {
	case w.queue <- job:
		return true
	default:
		return false // backpressure: queue is full
	}
}

func (w *Worker) Run(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			return
		case job := <-w.queue:
			w.processWithRetry(ctx, job)
		}
	}
}

func (w *Worker) processWithRetry(ctx context.Context, job Job) {
	backoff := 100 * time.Millisecond
	for attempt := 1; attempt <= 5; attempt++ {
		if err := w.send(ctx, job); err == nil {
			return
		} else {
			slog.Warn("job failed, retrying",
				"ticket_id", job.TicketID,
				"attempt", attempt,
				"err", err,
			)
		}

		select {
		case <-ctx.Done():
			return
		case <-time.After(backoff):
			backoff *= 2
		}
	}
	slog.Error("job exhausted retries", "ticket_id", job.TicketID)
}
```

## Common Pitfalls

- **Lost work on crash or deploy:** In-memory channels vanish when the process exits or is killed by OOM. This limitation motivates moving to a durable [Message Broker](02-message-broker.md).
- **Detached request contexts:** Never pass `r.Context()` directly into a background goroutine—it cancels the moment the HTTP response is written. Derive a worker context tied to the service's shutdown lifecycle.
- **Unbounded queues and goroutines:** Always cap buffer sizes and worker counts, and decide explicitly whether to block or shed load (`503 Service Unavailable`) when saturated.

## Related Chapters

- Next: [02. Message Broker](02-message-broker.md)
- Book foundations: [Worker Pools](../11-concurrency-parallelism/44b-worker-pools.md), [Queue Workers and Backpressure](../18-distributed-infra/153-queue-workers-backpressure.md)
