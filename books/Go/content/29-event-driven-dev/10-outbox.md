---
title: '10. Outbox'
---

# Outbox

Writing to a database and publishing to an external message broker are two separate network operations—there is no distributed transaction spanning both. If the database commits and the broker publish fails (or vice versa), your system enters a split-brain state. The **Transactional Outbox pattern** solves this dual-write problem.

## Learning & Build Goals

- **Learn:** The dual-write problem and how the outbox pattern guarantees data consistency; publishing events inside the same SQL transaction as business state; forwarding outbox rows to the message broker.
- **Build:** Implement transactional outbox publishing in PostgreSQL; run an outbox forwarder to relay events to Redis Streams / Kafka; refactor the ticket aggregator to use the outbox.

## How the Outbox Pattern Works

```text
                 Single PostgreSQL ACID Transaction
                 +--------------------------------+
HTTP Request --->| 1. INSERT INTO bookings (...)  |
                 | 2. INSERT INTO outbox_events   |
                 +--------------------------------+
                                  |
                         (async background relay)
                                  v
                        [ Outbox Forwarder ]
                                  |
                                  v
                       [ Redis Streams / Kafka ] ---> Downstream Consumers
```

Both the domain write and the outgoing event are committed atomically in PostgreSQL. A separate **forwarder** reads committed rows from the outbox table, publishes them to the main broker, and marks them as processed.

## Publishing Inside a SQL Transaction

With Watermill's `watermill-sql` publisher and `forwarder` component, your domain code publishes through the exact same `cqrs.EventBus` interface, backed by the active `*sql.Tx`.

```go
package booking

import (
	"context"
	"database/sql"
	"fmt"

	"github.com/ThreeDotsLabs/watermill"
	wsql "github.com/ThreeDotsLabs/watermill-sql/v3/pkg/sql"
	"github.com/ThreeDotsLabs/watermill/components/cqrs"
	"github.com/ThreeDotsLabs/watermill/components/forwarder"
	"github.com/ThreeDotsLabs/watermill/message"
)

func CreateBookingWithOutbox(
	ctx context.Context,
	db *sql.DB,
	bookingID string,
	event any,
	marshaler cqrs.CommandEventMarshaler,
	logger watermill.LoggerAdapter,
) error {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return fmt.Errorf("begin tx: %w", err)
	}
	defer tx.Rollback()

	// 1. Write domain state inside tx.
	if _, err := tx.ExecContext(ctx, `INSERT INTO bookings (booking_id) VALUES ($1)`, bookingID); err != nil {
		return fmt.Errorf("insert booking: %w", err)
	}

	// 2. Wrap tx in a SQL publisher + forwarder decorator so the event lands in the outbox table.
	var pub message.Publisher
	pub, err = wsql.NewPublisher(tx, wsql.PublisherConfig{
		SchemaAdapter: wsql.DefaultPostgreSQLSchema{},
	}, logger)
	if err != nil {
		return fmt.Errorf("new sql publisher: %w", err)
	}
	pub = forwarder.NewPublisher(pub, forwarder.PublisherConfig{
		ForwarderTopic: "events_to_forward",
	})

	eventBus, err := cqrs.NewEventBusWithConfig(pub, cqrs.EventBusConfig{
		GeneratePublishTopic: func(p cqrs.GenerateEventPublishTopicParams) (string, error) {
			return "events." + p.EventName, nil
		},
		Marshaler: marshaler,
		Logger:    logger,
	})
	if err != nil {
		return fmt.Errorf("new event bus: %w", err)
	}

	if err := eventBus.Publish(ctx, event); err != nil {
		return fmt.Errorf("publish outbox event: %w", err)
	}

	return tx.Commit()
}
```

## Common Pitfalls

- **Forgetting idempotency downstream:** The forwarder itself provides *at-least-once* delivery (if it crashes after publishing to Kafka but before marking the outbox row acked, it will forward the row again). Pair Outbox with [At Least Once Delivery](09-at-least-once-delivery.md).
- **Unbounded outbox table growth:** Configure periodic cleanup or retention of acknowledged outbox rows so table bloat does not degrade PostgreSQL query performance.
- **Publishing outside the transaction:** Double-check that the SQL publisher receives `*sql.Tx`, not `*sql.DB`.

## Related Chapters

- Previous: [09. At Least Once Delivery](09-at-least-once-delivery.md)
- Next: [11. Commands in CQRS Pattern](11-commands-in-cqrs.md)
- Book foundations: [Transactional Outbox Pattern](../18-distributed-infra/157-outbox-reliable-side-effects.md)
