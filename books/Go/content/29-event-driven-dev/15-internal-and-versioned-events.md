---
title: '15. Internal and Versioned Events'
---

# Internal and Versioned Events

Event schemas are long-lived public contracts—especially once events are archived in a [Data Lake](14-data-lake.md) or consumed by multiple services. Changing a field name or type in place breaks running consumers and prevents replaying historical events.

## Learning & Build Goals

- **Learn:** Distinction between internal and public/versioned events; evolving event schemas without breaking backward compatibility; using versioning and internal events to decouple services.
- **Build:** Add explicit event versioning to the ticket aggregator; separate internal events from public contracts; implement compatibility handling across versions.

## Internal vs. Public (Versioned) Events

| Type | Scope | Evolution Rules |
|------|-------|-----------------|
| **Internal Event** | Private to a single service boundary (e.g., `internal-events.svc-tickets.*`) | Can evolve faster, owned by one team, never subscribed to by external services |
| **Public / Versioned Event** | Shared across services and stored in the data lake (e.g., `TicketBookingConfirmed_v1`) | Immutable contract; breaking changes require a new version (`_v2`) |

```text
Service A (Internal)                 Public Contract Topics
  TicketConfirmedInternal  -------->  events.TicketBookingConfirmed_v1  (existing consumers)
                           -------->  events.TicketBookingConfirmed_v2  (new consumers)
```

## Naming and Handling Versioned Events in Go

Include the version suffix explicitly in the Go struct name so Watermill's `cqrs.StructName` marshaler routes `v1` and `v2` cleanly without ambiguity.

```go
package events

type TicketBookingConfirmed_v1 struct {
	Header        Header `json:"header"`
	TicketID      string `json:"ticket_id"`
	CustomerEmail string `json:"customer_email"`
	PriceAmount   string `json:"price_amount"`
	PriceCurrency string `json:"price_currency"`
}

// TicketBookingConfirmed_v2 splits price into a structured Money object and adds ShowID.
type Money struct {
	Amount   string `json:"amount"`
	Currency string `json:"currency"`
}

type TicketBookingConfirmed_v2 struct {
	Header        Header `json:"header"`
	TicketID      string `json:"ticket_id"`
	ShowID        string `json:"show_id"`
	CustomerEmail string `json:"customer_email"`
	Price         Money  `json:"price"`
}
```

### Safe Evolution Rules

1. **Additive changes (non-breaking):** Adding a new optional JSON field does not require a new version, provided consumers handle the zero value gracefully.
2. **Breaking changes (new version required):** Renaming a field, changing a field's type, changing semantics, or removing a required field requires a new event version (`_v2`).
3. **Dual-publish / upcast during migration:** Publish both `v1` and `v2` (or translate `v1 -> v2` in an upcaster adapter when reading from the data lake) until all consumers migrate, then retire `v1`.

## Common Pitfalls

- **Breaking replay of historical data:** Even if all live services upgrade to `v2`, your [Data Lake](14-data-lake.md) still contains `v1` records. Your replay/migration pipeline must continue to understand `v1` (or upcast `v1` to `v2`).
- **Exposing internal domain refactors publicly:** Keep internal workflow events on private topics so internal refactoring does not ripple across teams.

## Related Chapters

- Previous: [14. Data Lake](14-data-lake.md)
- Next: [16. Migrating Read Models](16-migrating-read-models.md)
- Also see: [03. Events](03-events.md)
