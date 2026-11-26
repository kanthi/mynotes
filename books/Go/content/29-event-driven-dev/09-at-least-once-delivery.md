---
title: '09. At Least Once Delivery'
---

# At Least Once Delivery

Distributed message brokers cannot guarantee true end-to-end "exactly-once" delivery across network partitions and process crashes. Production systems default to **at-least-once delivery** and push deduplication into **idempotent handlers and repositories**.

## Learning & Build Goals

- **Learn:** Why at-least-once delivery is the practical standard; how redelivery happens; designing idempotent handlers and repositories.
- **Build:** Persist event-driven ticket state in PostgreSQL; implement idempotent handlers and SQL repositories; ensure consistency when the same event is delivered multiple times.

## Why Redelivery Happens

```text
Broker ---> Consumer: TicketBookingConfirmed (ID: evt-42)
              ├── 1. INSERT INTO tickets ... (COMMIT succeeds)
              ├── 2. Process crashes before Ack reaches Broker!
              └── Broker times out and redelivers evt-42 to another replica
```

Because step 1 committed before the crash, the redelivered message must not fail with a duplicate-key error or trigger a second charge/email.

## Idempotent SQL Repositories

Push idempotency into the database using primary keys, unique constraints, and `ON CONFLICT` clauses.

```go
package repository

import (
	"context"
	"database/sql"
	"fmt"
)

type Ticket struct {
	ID            string
	PriceAmount   string
	PriceCurrency string
	CustomerEmail string
}

type TicketsRepo struct {
	db *sql.DB
}

func (r *TicketsRepo) Add(ctx context.Context, t Ticket) error {
	const query = `
		INSERT INTO tickets (ticket_id, price_amount, price_currency, customer_email)
		VALUES ($1, $2, $3, $4)
		ON CONFLICT (ticket_id) DO NOTHING
	`
	if _, err := r.db.ExecContext(ctx, query, t.ID, t.PriceAmount, t.PriceCurrency, t.CustomerEmail); err != nil {
		return fmt.Errorf("insert ticket %s: %w", t.ID, err)
	}
	return nil
}

func (r *TicketsRepo) Delete(ctx context.Context, ticketID string) error {
	// DELETE is naturally idempotent: deleting a non-existent row affects 0 rows with no error.
	if _, err := r.db.ExecContext(ctx, `DELETE FROM tickets WHERE ticket_id = $1`, ticketID); err != nil {
		return fmt.Errorf("delete ticket %s: %w", ticketID, err)
	}
	return nil
}
```

### Idempotency for External API Calls

When calling an external HTTP service (such as issuing a receipt or charging a card), pass a deterministic **idempotency key** derived from the domain identifier (e.g., `event.Header.IdempotencyKey` or `ticketID`), never a freshly generated `uuid.New()` inside the handler—otherwise every retry generates a new key and creates duplicate side effects.

## Common Pitfalls

- **Generating idempotency keys inside the consumer:** A retry will generate a different UUID and defeat deduplication. Generate the key once at the source (HTTP client or originating command).
- **Check-then-act without constraints:** Running `SELECT` followed by `INSERT` has a race window under concurrent redelivery. Always back idempotency with a database `PRIMARY KEY` or `UNIQUE` constraint.

## Related Chapters

- Previous: [08. Events with CQRS Pattern](08-events-with-cqrs.md)
- Next: [10. Outbox](10-outbox.md)
- Book foundations: [Idempotency Keys](../18-distributed-infra/155-idempotency-keys.md)
