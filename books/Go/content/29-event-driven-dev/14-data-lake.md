---
title: '14. Data Lake'
---

# Data Lake

Message brokers are optimized for delivery, not indefinite historical storage or ad-hoc querying. A **Data Lake** (or central event store) archives every domain event in append-only form so you can audit history, run analytics, debug production incidents, and rebuild read models from scratch.

## Learning & Build Goals

- **Learn:** Role of a data lake in event-driven architectures; storing every domain event for future processing and business insights; event storage schemas and retrieval patterns.
- **Build:** Store all events centrally in a data lake table/store; forward events from the message broker into the data lake; retrieve historical events to feed read models and analytics.

## Broker-to-Data-Lake Architecture

```text
Producers ---> Message Broker (Redis Streams / Kafka)
                     ├──> Normal Service Consumers
                     └──> [ Data Lake Consumer ] ---> Append-Only Events Table / Object Store
                                                             |
                                                             +--> Analytics / BI Queries
                                                             +--> Read Model Rebuild / Replay
```

## Append-Only Event Storage Schema (PostgreSQL)

For small-to-medium systems, a dedicated append-only PostgreSQL table (or partitioned table by `event_date`) works well before graduating to Parquet files on S3/GCS or ClickHouse.

```sql
CREATE TABLE IF NOT EXISTS events (
    event_id      UUID PRIMARY KEY,
    published_at  TIMESTAMPTZ NOT NULL,
    event_name    TEXT NOT NULL,
    event_payload JSONB NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_events_published_at ON events (published_at, event_id);
```

## Forwarding Events into the Data Lake

```go
package datalake

import (
	"context"
	"database/sql"
	"fmt"
	"time"
)

type EventRecord struct {
	EventID      string
	PublishedAt  time.Time
	EventName    string
	EventPayload []byte
}

type Store struct {
	db *sql.DB
}

func (s *Store) Save(ctx context.Context, rec EventRecord) error {
	const query = `
		INSERT INTO events (event_id, published_at, event_name, event_payload)
		VALUES ($1, $2, $3, $4)
		ON CONFLICT (event_id) DO NOTHING
	`
	if _, err := s.db.ExecContext(ctx, query, rec.EventID, rec.PublishedAt, rec.EventName, rec.EventPayload); err != nil {
		return fmt.Errorf("save event %s to data lake: %w", rec.EventID, err)
	}
	return nil
}

func (s *Store) StreamSince(ctx context.Context, since time.Time, batchSize int, handle func(EventRecord) error) error {
	rows, err := s.db.QueryContext(ctx, `
		SELECT event_id, published_at, event_name, event_payload
		FROM events
		WHERE published_at >= $1
		ORDER BY published_at ASC, event_id ASC
		LIMIT $2
	`, since, batchSize)
	if err != nil {
		return fmt.Errorf("query data lake: %w", err)
	}
	defer rows.Close()

	for rows.Next() {
		var rec EventRecord
		if err := rows.Scan(&rec.EventID, &rec.PublishedAt, &rec.EventName, &rec.EventPayload); err != nil {
			return fmt.Errorf("scan event: %w", err)
		}
		if err := handle(rec); err != nil {
			return err
		}
	}
	return rows.Err()
}
```

## Common Pitfalls

- **Mutating or deleting historical events:** The data lake is an immutable audit log. Never update past event payloads in place; handle schema evolution with [Internal and Versioned Events](15-internal-and-versioned-events.md).
- **Storing unencrypted PII without a retention plan:** If events contain customer emails or personal data, plan for GDPR/right-to-erasure (for example, crypto-shredding per customer key or separating PII into a lookup table).
- **Using the data lake as a hot transactional OLTP store:** Keep operational queries on dedicated [Read Models](12-read-models.md), not raw event scans.

## Related Chapters

- Previous: [13. Message Ordering](13-message-ordering.md)
- Next: [15. Internal and Versioned Events](15-internal-and-versioned-events.md)
- Also see: [16. Migrating Read Models](16-migrating-read-models.md)
