---
title: '07. Component Testing'
---

# Component Testing

Unit tests verify isolated functions, but event-driven bugs almost always live at the seams: JSON marshalling, topic routing, middleware ordering, database transactions, and consumer group wiring. **Component tests** boot your entire service in-process against real local infrastructure (Redis, Postgres) while mocking only external third-party HTTP APIs.

## Learning & Build Goals

- **Learn:** Why component testing is critical for event-driven services; designing clean mocks for external dependencies; running the full service inside `go test`.
- **Build:** Refactor the ticket aggregator service so it can be constructed and run programmatically in tests; write thread-safe mocks for external services; implement end-to-end component tests.

## Component Test Architecture

```text
go test
  └── Service.Run(ctx)
        ├── HTTP API (Echo / net/http)  <--- Test sends real HTTP POST /tickets
        ├── Watermill Router            <--- Real Redis Streams / Postgres
        ├── SQL Repository              <--- Real Postgres DB
        └── External Clients            <--- In-memory thread-safe Fakes/Mocks
                                             (Spreadsheet, Receipt, Dead Nation API)
```

## Making the Service Runnable in Tests

Extract service assembly out of `main()` into a `Service` struct whose `Run(ctx context.Context) error` method starts both the HTTP server and the Watermill router, and stops cleanly when `ctx` is canceled.

```go
package service_test

import (
	"context"
	"net/http"
	"sync"
	"testing"
	"time"
)

type FakeReceiptsClient struct {
	mu     sync.Mutex
	Issued []string
}

func (f *FakeReceiptsClient) IssueReceipt(_ context.Context, ticketID string) error {
	f.mu.Lock()
	defer f.mu.Unlock()
	f.Issued = append(f.Issued, ticketID)
	return nil
}

func (f *FakeReceiptsClient) HasIssued(ticketID string) bool {
	f.mu.Lock()
	defer f.mu.Unlock()
	for _, id := range f.Issued {
		if id == ticketID {
			return true
		}
	}
	return false
}

// assertEventually polls an async condition without fragile time.Sleep calls.
func assertEventually(t *testing.T, timeout time.Duration, check func() bool) {
	t.Helper()
	deadline := time.Now().Add(timeout)
	for time.Now().Before(deadline) {
		if check() {
			return
		}
		time.Sleep(25 * time.Millisecond)
	}
	t.Fatalf("condition not met within %s", timeout)
}

func TestTicketBookingFlow(t *testing.T) {
	ctx, cancel := context.WithCancel(t.Context())
	defer cancel()

	receipts := &FakeReceiptsClient{}
	// Start service in background goroutine with real broker/DB and fake external clients.
	// Wait for HTTP /healthz to return 200 OK before sending requests.
	_ = http.StatusOK

	// Send POST /tickets-status, then assert async side effects:
	assertEventually(t, 5*time.Second, func() bool {
		return receipts.HasIssued("ticket-123")
	})
}
```

## Common Pitfalls

- **Fixed `time.Sleep` in tests:** Makes tests slow on fast machines and flaky on busy CI runners. Always poll with `assertEventually` (or `require.Eventually` from `testify`).
- **Data races in fakes:** Event handlers run concurrently in background goroutines while the test goroutine reads fake state. Always guard mock slices/maps with `sync.Mutex` and run `go test -race ./...`.
- **Shared state across tests:** Generate unique IDs (UUIDs) per test case so parallel or repeated runs never collide on existing database rows or Redis streams.

## Related Chapters

- Previous: [06. Errors](06-errors.md)
- Next: [08. Events with CQRS Pattern](08-events-with-cqrs.md)
- Book foundations: [Integration Testing](../06-testing/39-integration-testing.md), [Testing Concurrent Code](../10-concurrency-ground-up/260-testing-concurrent-code.md)
