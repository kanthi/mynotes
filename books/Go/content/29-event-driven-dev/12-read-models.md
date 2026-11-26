---
title: '12. Read Models'
---

# Read Models

Normalized write tables are optimized for transactional integrity, not for complex reporting, dashboards, or cross-service queries. **Read models** (also called projections) are purpose-built views populated asynchronously from domain events so read endpoints stay fast and decoupled from write schemas.

## Learning & Build Goals

- **Learn:** Role of read models in CQRS; building and updating read models incrementally from incoming events; tailoring read models to specific query patterns.
- **Build:** Design and implement read models for the ticket aggregator (such as VIP/operations views); update projections from domain events; expose read models via HTTP API endpoints.

## Write Side vs. Read Side

```text
Write Side:
  Command -> Booking Service -> PostgreSQL (bookings) -> Outbox -> Events

Read Side (asynchronous projection):
  Events ---> Read Model Updater ---> PostgreSQL / Redis / Search Index (ops_read_model)
                                                   ^
                                                   |
                                            GET /ops/bookings (fast, zero-join read)
```

## Event-Driven Read Model Updater

A read model handler consumes multiple event types and incrementally merges them into a denormalized row (for example, a PostgreSQL `JSONB` document or flat reporting table).

```go
package readmodel

import (
	"context"
	"database/sql"
	"encoding/json"
	"fmt"
	"time"
)

type OpsBooking struct {
	BookingID     string                 `json:"booking_id"`
	BookedAt      time.Time              `json:"booked_at"`
	Tickets       map[string]OpsTicket   `json:"tickets"`
	LastUpdate    time.Time              `json:"last_update"`
}

type OpsTicket struct {
	TicketID      string `json:"ticket_id"`
	CustomerEmail string `json:"customer_email"`
	Confirmed     bool   `json:"confirmed"`
	Refunded      bool   `json:"refunded"`
}

type Store struct {
	db *sql.DB
}

func (s *Store) UpdateBooking(
	ctx context.Context,
	bookingID string,
	updateFn func(*OpsBooking) error,
) error {
	tx, err := s.db.BeginTx(ctx, nil)
	if err != nil {
		return fmt.Errorf("begin tx: %w", err)
	}
	defer tx.Rollback()

	var raw []byte
	err = tx.QueryRowContext(
		ctx,
		`SELECT payload FROM read_model_ops_bookings WHERE booking_id = $1 FOR UPDATE`,
		bookingID,
	).Scan(&raw)

	var model OpsBooking
	switch {
	case err == sql.ErrNoRows:
		model = OpsBooking{
			BookingID: bookingID,
			Tickets:   make(map[string]OpsTicket),
		}
	case err != nil:
		return fmt.Errorf("select read model: %w", err)
	default:
		if err := json.Unmarshal(raw, &model); err != nil {
			return fmt.Errorf("unmarshal read model: %w", err)
		}
	}

	if err := updateFn(&model); err != nil {
		return err
	}
	model.LastUpdate = time.Now().UTC()

	encoded, err := json.Marshal(model)
	if err != nil {
		return fmt.Errorf("marshal read model: %w", err)
	}

	const upsert = `
		INSERT INTO read_model_ops_bookings (booking_id, payload)
		VALUES ($1, $2)
		ON CONFLICT (booking_id) DO UPDATE SET payload = EXCLUDED.payload
	`
	if _, err := tx.ExecContext(ctx, upsert, bookingID, encoded); err != nil {
		return fmt.Errorf("upsert read model: %w", err)
	}

	return tx.Commit()
}
```

## Common Pitfalls

- **Assuming strong consistency on reads:** Read models are **eventually consistent**. Design UIs and APIs to tolerate brief projection lag (e.g., returning the created ID immediately and polling or showing optimistic state).
- **Skipping `SELECT ... FOR UPDATE` on concurrent updates:** If two events for the same booking arrive concurrently on different workers, a plain read-modify-write loses one of the updates. Lock the row or use [Message Ordering](13-message-ordering.md) strategies.
- **Using a read model as the source of truth for write invariants:** Business rules belong on the authoritative write side, not on eventually consistent projections.

## Related Chapters

- Previous: [11. Commands in CQRS Pattern](11-commands-in-cqrs.md)
- Next: [13. Message Ordering](13-message-ordering.md)
- Also see: [16. Migrating Read Models](16-migrating-read-models.md)
