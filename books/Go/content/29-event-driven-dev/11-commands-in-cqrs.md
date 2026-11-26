---
title: '11. Commands in CQRS Pattern'
---

# Commands in CQRS Pattern

Events broadcast facts to zero or many interested consumers. When you want **one specific handler** to perform an action asynchronously—such as refunding a ticket or booking seats—use a **Command** instead of overloading an event.

## Learning & Build Goals

- **Learn:** Difference between commands (user/system intent) and events (recorded state change); using a CQRS `CommandBus` for asynchronous execution; implementing command handlers and processors.
- **Build:** Refactor imperative async flows to commands; implement command handlers and processors with Watermill CQRS; isolate commands and events onto separate broker topics.

## Commands vs. Events

| Characteristic | Command | Event |
|----------------|---------|-------|
| **Intent** | Imperative request to *do* something (`RefundTicket`) | Past-tense fact that *happened* (`TicketRefunded`) |
| **Receivers** | **Exactly one** handler owns the command | **Zero, one, or many** independent subscribers |
| **Can be rejected?** | Yes (validation, business invariant) | No—facts have already occurred |
| **Topic prefix** | `commands.<CommandName>` | `events.<EventName>` |

```text
HTTP API ---> CommandBus.Send(RefundTicket) ---> Topic: commands.RefundTicket
                                                        |
                                                        v
                                              [ RefundTicket Handler ]
                                                 ├── Call payment gateway
                                                 └── EventBus.Publish(TicketRefunded)
```

## Wiring a CommandBus and CommandProcessor

```go
package cqrsapp

import (
	"context"
	"fmt"

	"github.com/ThreeDotsLabs/watermill"
	"github.com/ThreeDotsLabs/watermill/components/cqrs"
	"github.com/ThreeDotsLabs/watermill/message"
)

type RefundTicket struct {
	TicketID string `json:"ticket_id"`
}

func SetupCommandCQRS(
	router *message.Router,
	pub message.Publisher,
	newSubscriber func(consumerGroup string) (message.Subscriber, error),
	logger watermill.LoggerAdapter,
) (*cqrs.CommandBus, error) {
	marshaler := cqrs.JSONMarshaler{GenerateName: cqrs.StructName}

	commandBus, err := cqrs.NewCommandBusWithConfig(pub, cqrs.CommandBusConfig{
		GeneratePublishTopic: func(params cqrs.CommandBusGeneratePublishTopicParams) (string, error) {
			return "commands." + params.CommandName, nil
		},
		Marshaler: marshaler,
		Logger:    logger,
	})
	if err != nil {
		return nil, fmt.Errorf("create command bus: %w", err)
	}

	commandProcessor, err := cqrs.NewCommandProcessorWithConfig(router, cqrs.CommandProcessorConfig{
		GenerateSubscribeTopic: func(params cqrs.CommandProcessorGenerateSubscribeTopicParams) (string, error) {
			return "commands." + params.CommandName, nil
		},
		SubscriberConstructor: func(params cqrs.CommandProcessorSubscriberConstructorParams) (message.Subscriber, error) {
			return newSubscriber("command." + params.HandlerName)
		},
		Marshaler: marshaler,
		Logger:    logger,
	})
	if err != nil {
		return nil, fmt.Errorf("create command processor: %w", err)
	}

	err = commandProcessor.AddHandlers(
		cqrs.NewCommandHandler(
			"RefundTicket",
			func(ctx context.Context, cmd *RefundTicket) error {
				// Execute refund and emit TicketRefunded event.
				return nil
			},
		),
	)
	if err != nil {
		return nil, fmt.Errorf("add command handlers: %w", err)
	}

	return commandBus, nil
}
```

## Common Pitfalls

- **Registering multiple handlers for one command:** Unlike events, a command has a single authoritative owner. If two actions must react to a single trigger, emit an event from the command handler instead.
- **Mixing command and event topics:** Keeping `commands.*` and `events.*` on separate topics makes routing, permissions, retention policies, and monitoring unambiguous.

## Related Chapters

- Previous: [10. Outbox](10-outbox.md)
- Next: [12. Read Models](12-read-models.md)
- Also see: [08. Events with CQRS Pattern](08-events-with-cqrs.md), [20. Sagas and Process Managers](20-sagas-and-process-managers.md)
