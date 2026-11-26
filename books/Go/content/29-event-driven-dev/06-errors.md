---
title: '06. Errors'
---

# Errors

Not every failure in an event-driven system should be retried blindly. Retrying a malformed JSON payload 100 times wastes CPU and blocks consumer throughput, whereas dropping a temporary network timeout loses customer data. Effective error handling classifies failures by recoverability.

## Learning & Build Goals

- **Learn:** Handling different error categories; centralized error logging and monitoring via middleware; distinguishing temporary errors, malformed messages, and code bugs.
- **Build:** Implement an error-logging middleware and a retry middleware; handle malformed messages and code bugs gracefully without stalling queues.

## Error Taxonomy

| Error Category | Examples | Strategy |
|----------------|----------|----------|
| **Temporary (Transient)** | Downstream HTTP 503, DB connection reset, rate-limit 429 | Retry with exponential backoff + jitter |
| **Malformed Message** | Invalid JSON, missing required fields, unknown schema | Do **not** retry; log with payload context and route to a poison queue or Ack+drop |
| **Code Bug / Panic** | Nil pointer dereference, index out of range | Recover panic via middleware, alert immediately, route to poison queue after bounded retries |
| **Domain / Business Rule** | Ticket already canceled, invalid state transition | Handle explicitly (usually Ack without retrying) |

## Retry and Error-Classification Middleware

Watermill includes `middleware.Retry` and `middleware.Recoverer`. You can combine them with explicit permanent-error wrapping so non-retryable failures fail fast.

```go
package middleware

import (
	"errors"
	"log/slog"
	"time"

	"github.com/ThreeDotsLabs/watermill/message"
	wmid "github.com/ThreeDotsLabs/watermill/message/router/middleware"
)

// PermanentError marks a failure that will never succeed on retry.
type PermanentError struct {
	Err error
}

func (e PermanentError) Error() string { return e.Err.Error() }
func (e PermanentError) Unwrap() error { return e.Err }

func Permanent(err error) error {
	if err == nil {
		return nil
	}
	return PermanentError{Err: err}
}

func NewRetryMiddleware(logger *slog.Logger) message.HandlerMiddleware {
	retry := wmid.Retry{
		MaxRetries:          5,
		InitialInterval:     200 * time.Millisecond,
		MaxInterval:         10 * time.Second,
		Multiplier:          2,
		RandomizationFactor: 0.2,
		OnRetryHook: func(retryNum int, delay time.Duration) {
			logger.Warn("retrying handler", "retry", retryNum, "delay", delay)
		},
	}

	return func(next message.HandlerFunc) message.HandlerFunc {
		retried := retry.Middleware(next)
		return func(msg *message.Message) ([]*message.Message, error) {
			produced, err := retried(msg)
			var perm PermanentError
			if errors.As(err, &perm) {
				logger.Error("permanent error, skipping retry",
					"message_uuid", msg.UUID,
					"err", perm.Err,
				)
				return nil, nil // or forward to a poison queue before returning nil
			}
			return produced, err
		}
	}
}
```

> [!WARNING]
> Always include `wmid.Recoverer` in your router middlewares. An unrecovered panic inside a handler crashes the entire service process and drops all other in-flight handlers.

## Common Pitfalls

- **Infinite Nack loops:** Returning an error forever without a max-retry cap or poison queue creates a "poison pill" that wedges a consumer partition.
- **Swallowing errors silently:** If you return `nil` for a malformed message, emit a metric and structured log (or send to a poison queue) so broken producers are detected.

## Related Chapters

- Previous: [05. Middlewares](05-middlewares.md)
- Next: [07. Component Testing](07-component-testing.md)
- Also see: [19. Fault Tolerance](19-fault-tolerance.md), [Error Wrapping](../05-errors/34-error-wrapping.md)
