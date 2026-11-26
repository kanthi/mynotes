---
title: '16. Migrating Read Models'
---

# Migrating Read Models

Because [Read Models](12-read-models.md) are derived views rather than the source of truth, you can evolve their schema—or recover from projection bugs and data corruption—by **replaying historical events** from your [Data Lake](14-data-lake.md).

## Learning & Build Goals

- **Learn:** Migrating read models when query requirements or schemas change; rebuilding read models from a data lake after corruption or data loss; keeping live traffic consistent during migration.
- **Build:** Implement a read-model migration and rebuild workflow from data-lake events; ensure zero-downtime consistency while catching up with live events.

## Zero-Downtime Blue/Green Read Model Rebuild

Never truncate a live read-model table in production while users are querying it. Build the new projection alongside the old one and switch atomically once caught up.

```text
1. Live traffic reads v1 table:   API ---> read_model_v1  <--- Live Updater v1
2. Start v2 updater on live + replay historical events from Data Lake:
                                  Data Lake (historical) --+
                                                           +--> Updater v2 ---> read_model_v2
                                  Broker (live stream)   --+
3. Verify row counts & spot-check invariants between v1 and v2
4. Cut over API reads to read_model_v2; retire Updater v1 and drop read_model_v1
```

## Replaying Events Safely

Because historical events from the data lake and live events from the broker may overlap during catch-up, the projection updater must be **idempotent** and **order-safe** (using the techniques from [At Least Once Delivery](09-at-least-once-delivery.md) and [Message Ordering](13-message-ordering.md)).

```go
package migrate

import (
	"context"
	"encoding/json"
	"fmt"
	"time"
)

type ArchivedEvent struct {
	EventID      string
	PublishedAt  time.Time
	EventName    string
	EventPayload []byte
}

type EventReplayer struct {
	handlers map[string]func(ctx context.Context, payload []byte) error
}

func (r *EventReplayer) Register(eventName string, fn func(context.Context, []byte) error) {
	if r.handlers == nil {
		r.handlers = make(map[string]func(context.Context, []byte) error)
	}
	r.handlers[eventName] = fn
}

func (r *EventReplayer) ReplayBatch(ctx context.Context, batch []ArchivedEvent) error {
	for _, evt := range batch {
		fn, ok := r.handlers[evt.EventName]
		if !ok {
			continue // event type not relevant to this read model
		}
		if err := fn(ctx, evt.EventPayload); err != nil {
			return fmt.Errorf("replay %s (%s): %w", evt.EventName, evt.EventID, err)
		}
	}
	return nil
}

// Helper for typed registration during replay.
func On[T any](r *EventReplayer, name string, fn func(context.Context, T) error) {
	r.Register(name, func(ctx context.Context, raw []byte) error {
		var v T
		if err := json.Unmarshal(raw, &v); err != nil {
			return err
		}
		return fn(ctx, v)
	})
}
```

## Common Pitfalls

- **Triggering side effects during replay:** Replaying historical events must **only** update the target read model. Never replay into handlers that send emails, issue receipts, or charge payment providers!
- **Loading the entire data lake into memory:** Always stream in keyset-paginated batches (`WHERE (published_at, event_id) > ($1, $2) ORDER BY published_at, event_id LIMIT 500`).

## Related Chapters

- Previous: [15. Internal and Versioned Events](15-internal-and-versioned-events.md)
- Next: [17. Metrics and Alerting](17-metrics-and-alerting.md)
- Also see: [12. Read Models](12-read-models.md), [14. Data Lake](14-data-lake.md)
