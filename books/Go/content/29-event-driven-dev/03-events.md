---
title: '03. Events'
---

# Events

Raw byte messages work for transport, but maintainable systems require well-defined **events**—immutable records of facts that have already happened in the domain.

## Learning & Build Goals

- **Learn:** Difference between events and plain messages; designing and structuring events for maintainability and evolution; marshalling/unmarshalling; using event headers for metadata and observability.
- **Build:** Refactor the ticket aggregator to publish and consume structured events instead of plain messages; design clean event payloads; attach standardized event headers.

## Events vs. Messages

| Concept | Meaning | Naming Convention | Coupling |
|---------|---------|-------------------|----------|
| **Message** | Low-level transport envelope (bytes + metadata) | Any topic/key | Transport-focused |
| **Event** | Immutable business fact that already occurred | Past tense (`TicketBookingConfirmed`, `TicketCanceled`) | Producer does not know or care who reacts |

An event describes **what happened**, not **what another service should do**. Avoid imperative names like `SendTicketEmail` for events—those are [Commands](11-commands-in-cqrs.md).

## Designing Event Payloads and Headers

Separate **domain payload** from **operational metadata** by placing cross-cutting fields in an `EventHeader` struct (or message metadata headers).

```go
package events

import (
	"time"

	"github.com/google/uuid"
)

type Header struct {
	ID             string    `json:"id"`
	PublishedAt    time.Time `json:"published_at"`
	IdempotencyKey string    `json:"idempotency_key,omitempty"`
}

func NewHeader() Header {
	return Header{
		ID:          uuid.NewString(),
		PublishedAt: time.Now().UTC(),
	}
}

type TicketBookingConfirmed struct {
	Header        Header `json:"header"`
	TicketID      string `json:"ticket_id"`
	CustomerEmail string `json:"customer_email"`
	PriceAmount   string `json:"price_amount"`
	PriceCurrency string `json:"price_currency"`
}
```

### Design Guidelines

1. **Keep payloads self-contained but lean:** Include the identifiers and facts consumers actually need so they do not have to call back into the producer's API on every event.
2. **Use explicit types for money and time:** Represent monetary amounts as decimal strings or integer minor units with an explicit currency code—never `float64`. Always serialize timestamps in UTC (RFC 3339).
3. **Design for additive evolution:** Adding optional fields is backward-compatible; renaming or removing fields breaks existing consumers. (Covered in depth in [Internal and Versioned Events](15-internal-and-versioned-events.md).)
4. **Separate transport metadata from domain headers:** Correlation IDs, trace contexts, and content-type headers belong in broker message metadata so middlewares can inspect them without unmarshalling the JSON payload.

## Common Pitfalls

- **Publishing database ORM structs directly:** Coupling event schemas to internal SQL models turns every schema refactor into a breaking contract change across services.
- **Thin "notification-only" events everywhere:** Sending only `{"ticket_id": "..."}` forces every consumer to query the producer over HTTP, recreating synchronous coupling and N+1 load spikes.
- **Missing event IDs:** Without a unique `Header.ID` or idempotency key, downstream consumers cannot safely deduplicate redeliveries.

## Related Chapters

- Previous: [02. Message Broker](02-message-broker.md)
- Next: [04. Watermill Basics](04-watermill-basics.md)
- Also see: [11. Commands in CQRS Pattern](11-commands-in-cqrs.md), [15. Internal and Versioned Events](15-internal-and-versioned-events.md)
