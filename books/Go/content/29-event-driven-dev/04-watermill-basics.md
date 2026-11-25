---
title: '04. Watermill Basics'
---

# Watermill Basics

Writing manual subscription loops, `Ack`/`Nack` plumbing, goroutine coordination, and shutdown handling for every topic quickly turns into repetitive, bug-prone infrastructure code. **Watermill's Router** abstracts that plumbing so handlers stay focused on business logic.

> [!NOTE]
> In the Three Dots Labs course brochure, this module is also referred to as **Router**.

## Learning & Build Goals

- **Learn:** Implementing handlers without infrastructure boilerplate; business-logic-first handler design; standard patterns for handler registration and error handling.
- **Build:** Replace hand-rolled consumer loops with Watermill's `message.Router`; implement graceful shutdown; add health check endpoints tied to broker connectivity.

## How the Watermill Router Works

```text
Broker Topic A ---> [ Subscriber ] ---> [ Middleware Chain ] ---> [ Handler A ] ---> (optional output messages) ---> [ Publisher ] ---> Topic B
                                                                        |
                                                                nil error -> Ack
                                                                non-nil   -> Nack
```

The router owns the subscribe loop, dispatches each message through middlewares to your handler, automatically publishes any returned messages, and calls `Ack()` on `nil` error or `Nack()` on error.

## Setting Up a Router with Graceful Shutdown

```go
package app

import (
	"context"
	"fmt"
	"time"

	"github.com/ThreeDotsLabs/watermill"
	"github.com/ThreeDotsLabs/watermill/message"
	"github.com/ThreeDotsLabs/watermill/message/router/plugin"
)

func NewRouter(
	sub message.Subscriber,
	pub message.Publisher,
	logger watermill.LoggerAdapter,
) (*message.Router, error) {
	router, err := message.NewRouter(message.RouterConfig{
		CloseTimeout: 30 * time.Second,
	}, logger)
	if err != nil {
		return nil, fmt.Errorf("create router: %w", err)
	}

	// Automatically closes the router on SIGINT / SIGTERM.
	router.AddPlugin(plugin.SignalsHandler)

	// Handler that consumes from one topic and publishes to another.
	router.AddHandler(
		"ticket_confirmed_to_receipt",
		"tickets.confirmed",
		sub,
		"receipts.issue",
		pub,
		func(msg *message.Message) ([]*message.Message, error) {
			// Pure business logic; router handles Ack/Nack and publishing.
			out := message.NewMessage(watermill.NewUUID(), msg.Payload)
			return []*message.Message{out}, nil
		},
	)

	// Consumer-only handler (no output topic).
	router.AddConsumerHandler(
		"ticket_confirmed_email",
		"tickets.confirmed",
		sub,
		func(msg *message.Message) error {
			return nil
		},
	)

	return router, nil
}
```

## Readiness and Health Checks

A service that has started its HTTP server before the router is connected to the broker—or after the router has crashed—should not receive traffic.

```go
// Wait until the router is running before marking the service ready.
select {
case <-router.Running():
	// Ready to serve HTTP traffic
case <-ctx.Done():
	return ctx.Err()
}
```

For `/healthz` or `/readyz`, verify both `router.IsRunning()` and a lightweight broker ping (e.g., Redis `PING` or database `PingContext`) with a short timeout.

## Common Pitfalls

- **Starting HTTP before the router is ready (or forgetting to close both):** Coordinate the HTTP server and Watermill router with `errgroup.WithContext` so a fatal failure in either triggers a clean shutdown of the other.
- **Short `CloseTimeout`:** If `CloseTimeout` is shorter than your slowest in-flight handler, shutdown will interrupt active work and force redeliveries.
- **Ignoring `router.Running()`:** In tests and deployments, publishing messages before `<-router.Running()` completes can race with subscription startup.

## Related Chapters

- Previous: [03. Events](03-events.md)
- Next: [05. Middlewares](05-middlewares.md)
- Book foundations: [Health and Readiness Probes](../19-observability-sre/165-health-readiness-probes.md), [Graceful Shutdown](../15-web-development-in-go/281-graceful-shutdown-web.md)
