---
title: '08. Events with CQRS Pattern'
---

# Events with CQRS Pattern

Manually marshalling JSON, constructing `message.Message`, and wiring topic names in every handler does not scale as a codebase grows. **CQRS (Command Query Responsibility Segregation)** separates write operations from read operations, and Watermill's `cqrs` package provides a strongly typed `EventBus` and `EventProcessor` on top of the low-level router.

## Learning & Build Goals

- **Learn:** CQRS fundamentals and how CQRS complements event-driven architecture; Watermill's high-level `EventBus` and `EventProcessor`; scaling event handlers with consumer groups.
- **Build:** Refactor low-level router handlers to Watermill's CQRS `EventBus` and `EventProcessor`; implement typed event handlers; configure per-handler consumer groups for horizontal scaling.

## Low-Level Router vs. CQRS Component

```text
Low-Level Router:
  func(msg *message.Message) error  --> manual json.Unmarshal + topic string wiring

CQRS EventProcessor:
  func(ctx context.Context, event *TicketBookingConfirmed) error
  --> automatic topic naming, marshalling, type routing, and consumer group creation
```

## Wiring an EventBus and EventProcessor

```go
package cqrsapp

import (
	"context"
	"fmt"

	"github.com/ThreeDotsLabs/watermill"
	"github.com/ThreeDotsLabs/watermill/components/cqrs"
	"github.com/ThreeDotsLabs/watermill/message"
)

type TicketBookingConfirmed struct {
	TicketID      string `json:"ticket_id"`
	CustomerEmail string `json:"customer_email"`
}

func SetupEventCQRS(
	router *message.Router,
	pub message.Publisher,
	newSubscriber func(consumerGroup string) (message.Subscriber, error),
	logger watermill.LoggerAdapter,
) (*cqrs.EventBus, error) {
	marshaler := cqrs.JSONMarshaler{
		GenerateName: cqrs.StructName,
	}

	eventBus, err := cqrs.NewEventBusWithConfig(pub, cqrs.EventBusConfig{
		GeneratePublishTopic: func(params cqrs.GenerateEventPublishTopicParams) (string, error) {
			return "events." + params.EventName, nil
		},
		Marshaler: marshaler,
		Logger:    logger,
	})
	if err != nil {
		return nil, fmt.Errorf("create event bus: %w", err)
	}

	eventProcessor, err := cqrs.NewEventProcessorWithConfig(router, cqrs.EventProcessorConfig{
		GenerateSubscribeTopic: func(params cqrs.EventProcessorGenerateSubscribeTopicParams) (string, error) {
			return "events." + params.EventName, nil
		},
		SubscriberConstructor: func(params cqrs.EventProcessorSubscriberConstructorParams) (message.Subscriber, error) {
			// Each handler gets its own consumer group so all handlers receive every event.
			return newSubscriber("handler." + params.HandlerName)
		},
		Marshaler: marshaler,
		Logger:    logger,
	})
	if err != nil {
		return nil, fmt.Errorf("create event processor: %w", err)
	}

	err = eventProcessor.AddHandlers(
		cqrs.NewEventHandler(
			"IssueReceiptOnTicketConfirmed",
			func(ctx context.Context, event *TicketBookingConfirmed) error {
				// Pure Go struct; no Watermill message boilerplate.
				return nil
			},
		),
	)
	if err != nil {
		return nil, fmt.Errorf("add event handlers: %w", err)
	}

	return eventBus, nil
}
```

## Common Pitfalls

- **Duplicate handler names:** Handler names determine consumer group names. Reusing a handler name across two different handlers causes them to compete for messages instead of both receiving every event.
- **Mismatched topic generators:** Ensure `EventBusConfig.GeneratePublishTopic` and `EventProcessorConfig.GenerateSubscribeTopic` produce identical topic names for the same event.

## Related Chapters

- Previous: [07. Component Testing](07-component-testing.md)
- Next: [09. At Least Once Delivery](09-at-least-once-delivery.md)
- Also see: [11. Commands in CQRS Pattern](11-commands-in-cqrs.md), [12. Read Models](12-read-models.md)
