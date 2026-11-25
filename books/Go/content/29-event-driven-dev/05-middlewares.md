---
title: '05. Middlewares'
---

# Middlewares

Cross-cutting concerns—structured logging, correlation ID propagation, panic recovery, retries, metrics, and tracing—should never be duplicated inside every message handler. Watermill middlewares wrap handlers in a composable chain, just like `net/http` middleware.

## Learning & Build Goals

- **Learn:** Middleware functions for cross-cutting concerns and code reuse; structured logging and correlation ID propagation across async boundaries; passing dependencies and configuration into middlewares.
- **Build:** Implement a logging middleware and a correlation ID middleware; refactor middlewares to accept explicit dependencies and configuration.

## Middleware Pipeline

```text
Incoming Message
  -> CorrelationID Middleware (extracts ID into ctx & logs)
    -> Logging Middleware (logs start, duration, outcome)
      -> Recoverer Middleware (turns panics into errors)
        -> Retry Middleware (retries transient failures)
          -> Domain Handler
```

> [!TIP]
> Middleware registration order matters: outer middlewares wrap inner ones. Place correlation ID and logging on the outside so retry attempts and recovered panics automatically carry the correlation ID.

## Correlation ID and Logging Middlewares

Propagate the correlation ID from the originating HTTP request into outgoing message metadata, then extract it back into `context.Context` inside the consumer middleware.

```go
package middleware

import (
	"context"
	"log/slog"
	"time"

	"github.com/ThreeDotsLabs/watermill/message"
	"github.com/google/uuid"
)

const CorrelationIDHeader = "correlation_id"

type ctxKey struct{}

func WithCorrelationID(ctx context.Context, id string) context.Context {
	return context.WithValue(ctx, ctxKey{}, id)
}

func CorrelationIDFromContext(ctx context.Context) string {
	if id, ok := ctx.Value(ctxKey{}).(string); ok && id != "" {
		return id
	}
	return ""
}

func CorrelationID(next message.HandlerFunc) message.HandlerFunc {
	return func(msg *message.Message) ([]*message.Message, error) {
		corrID := msg.Metadata.Get(CorrelationIDHeader)
		if corrID == "" {
			corrID = uuid.NewString()
			msg.Metadata.Set(CorrelationIDHeader, corrID)
		}

		ctx := WithCorrelationID(msg.Context(), corrID)
		msg.SetContext(ctx)

		produced, err := next(msg)
		for _, out := range produced {
			if out.Metadata.Get(CorrelationIDHeader) == "" {
				out.Metadata.Set(CorrelationIDHeader, corrID)
			}
		}
		return produced, err
	}
}

func Logging(logger *slog.Logger) message.HandlerMiddleware {
	return func(next message.HandlerFunc) message.HandlerFunc {
		return func(msg *message.Message) ([]*message.Message, error) {
			start := time.Now()
			corrID := CorrelationIDFromContext(msg.Context())
			handlerName := message.HandlerNameFromCtx(msg.Context())

			log := logger.With(
				"message_uuid", msg.UUID,
				"correlation_id", corrID,
				"handler", handlerName,
			)
			log.Info("handling message")

			produced, err := next(msg)
			if err != nil {
				log.Error("message handling failed",
					"err", err,
					"duration", time.Since(start),
				)
				return produced, err
			}

			log.Info("message handled",
				"produced", len(produced),
				"duration", time.Since(start),
			)
			return produced, err
		}
	}
}
```

## Publisher Decorators

Middlewares wrap *incoming* handlers. To ensure *outgoing* messages published outside a handler (for example from an HTTP endpoint) also receive a correlation ID, wrap `message.Publisher` with a decorator that reads `CorrelationIDFromContext(ctx)` and sets `msg.Metadata.Set(CorrelationIDHeader, id)`.

## Common Pitfalls

- **Global logger or global config inside middleware:** Always inject `*slog.Logger` and config structs via constructor functions (`Logging(logger)`) so tests can capture logs and isolate state.
- **Dropping correlation IDs on fan-out:** Always copy the correlation ID onto every message produced by a handler so end-to-end flows stay linked.

## Related Chapters

- Previous: [04. Watermill Basics](04-watermill-basics.md)
- Next: [06. Errors](06-errors.md)
- Book foundations: [Structured Logging and Correlation](../19-observability-sre/161-structured-logging-correlation.md), [HTTP Middleware Patterns](../15-web-development-in-go/280-middleware-patterns.md)
