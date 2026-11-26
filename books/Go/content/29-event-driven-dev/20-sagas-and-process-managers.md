---
title: '20. Sagas and Process Managers'
---

# Sagas and Process Managers

Single-service operations can rely on local ACID database transactions. Multi-step business workflows that span multiple services or external APIs—such as booking a VIP bundle (reserving seats, charging payment, booking a taxi, issuing receipts, and rolling back if any step fails)—cannot use distributed two-phase commit (2PC). Instead, they use **Sagas** and **Process Managers**.

## Learning & Build Goals

- **Learn:** Sagas and process managers for long-running distributed processes; orchestration vs. choreography trade-offs; implementing complex multi-step workflows and compensating actions.
- **Build:** Design and implement a saga/process manager in the ticket aggregator; implement compensating rollbacks on partial failure; maintain eventual consistency across services.

## Choreography vs. Orchestration

```text
Choreography (decentralized):
  BookingCreated -> [ Payment Svc ] -> PaymentSucceeded -> [ Taxi Svc ] -> TaxiBooked
  (Simple for 2–3 steps; hard to trace or modify when workflows grow complex)

Orchestration (Process Manager / Saga):
                     +---> Command: BookTickets  ---> [ Ticket Svc ] ---> TicketsBooked
  [ Process Manager ]+---> Command: ChargeCard   ---> [ Payment Svc ] --> PaymentFailed
                     +---> Command: CancelTickets --> [ Ticket Svc ] (compensating action)
```

| Dimension | Choreography | Orchestration (Process Manager) |
|-----------|--------------|---------------------------------|
| **Control flow** | Implicit across event subscribers | Explicit in one stateful coordinator |
| **Coupling** | Low between services | Orchestrator knows workflow steps; workers stay decoupled |
| **Visibility** | Must stitch logs/traces across services | Single state machine shows exact workflow progress |
| **Best fit** | Simple linear reactions (2–3 steps) | Multi-step workflows with branching, timeouts, and rollbacks |

## Stateful Process Manager Pattern in Go

A Process Manager persists its state in PostgreSQL, reacts to incoming domain events, advances its state machine, and emits the next CQRS command (or a compensating command on failure).

```go
package saga

import (
	"context"
	"fmt"
)

type VIPBundleStatus string

const (
	StatusPending        VIPBundleStatus = "pending"
	StatusTicketsBooked  VIPBundleStatus = "tickets_booked"
	StatusCompleted      VIPBundleStatus = "completed"
	StatusCompensating   VIPBundleStatus = "compensating"
	StatusFailed         VIPBundleStatus = "failed"
)

type VIPBundleProcess struct {
	ID            string
	BookingID     string
	TaxiBookingID string
	Status        VIPBundleStatus
}

type CommandBus interface {
	Send(ctx context.Context, cmd any) error
}

type Repo interface {
	Update(ctx context.Context, id string, fn func(*VIPBundleProcess) error) error
}

type ProcessManager struct {
	repo Repo
	bus  CommandBus
}

type BookTaxi struct{ BundleID string }
type RefundTickets struct{ BookingID string }

func (pm *ProcessManager) OnTicketsBooked(ctx context.Context, bundleID, bookingID string) error {
	return pm.repo.Update(ctx, bundleID, func(p *VIPBundleProcess) error {
		if p.Status != StatusPending {
			return nil // idempotent guard against redelivery
		}
		p.BookingID = bookingID
		p.Status = StatusTicketsBooked
		return pm.bus.Send(ctx, BookTaxi{BundleID: bundleID})
	})
}

func (pm *ProcessManager) OnTaxiBookingFailed(ctx context.Context, bundleID string) error {
	return pm.repo.Update(ctx, bundleID, func(p *VIPBundleProcess) error {
		if p.Status == StatusCompensating || p.Status == StatusFailed {
			return nil
		}
		p.Status = StatusCompensating
		// Compensating action: roll back the already-booked tickets.
		if err := pm.bus.Send(ctx, RefundTickets{BookingID: p.BookingID}); err != nil {
			return fmt.Errorf("send compensating refund: %w", err)
		}
		return nil
	})
}
```

> [!IMPORTANT]
> Just like any other state transition, persist the Process Manager's updated state and emit its outgoing command atomically using the [Transactional Outbox](10-outbox.md) pattern. Otherwise a crash between saving state and sending the next command stalls the workflow permanently.

## Common Pitfalls

- **Assuming compensating actions always succeed on the first try:** Compensations (`RefundTickets`, `CancelTaxi`) are regular async commands that can fail transiently. They must be retried, idempotent, and backed by a [Poison Queue](19-fault-tolerance.md) + alert if they exhaust retries.
- **Forgetting timeout handling:** If a downstream service never responds with either `Succeeded` or `Failed`, the saga stays `pending` forever. Schedule a timeout check (or delayed command) to detect stuck workflows and trigger compensation.
- **God orchestrators:** Keep business logic inside each domain service; the Process Manager should only coordinate state transitions and commands.

## Related Chapters

- Previous: [19. Fault Tolerance](19-fault-tolerance.md)
- Curriculum overview: [EventDrivenDev Intro](index.md)
- Also see: [10. Outbox](10-outbox.md), [11. Commands in CQRS Pattern](11-commands-in-cqrs.md)
