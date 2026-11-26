---
title: '18. Tracing'
---

# Tracing

Metrics tell you *that* something is slow or failing; **distributed tracing** tells you *where* and *why* across asynchronous hops. By propagating OpenTelemetry trace context inside broker message metadata, a single trace connects the initial HTTP request, the outbox write, the forwarder, and every downstream event handler.

## Learning & Build Goals

- **Learn:** Distributed tracing fundamentals (traces, spans, context propagation); instrumenting async publishers and subscribers to uncover bottlenecks and errors; end-to-end visibility across service boundaries.
- **Build:** Instrument the ticket aggregator with OpenTelemetry; propagate W3C `traceparent` context through message metadata; inspect traces to troubleshoot async flows.

## Trace Propagation Across a Message Broker

```text
[ HTTP POST /bookings (Span 1) ]
   ├── [ SQL INSERT + Outbox (Span 2) ]
   └── inject traceparent into msg.Metadata
             |
       (Message Broker)
             |
   extract traceparent from msg.Metadata
   ├── [ Event Handler: IssueReceipt (Span 3, parent=Span 1) ]
   │     └── [ HTTP POST /receipts (Span 4) ]
   └── [ Event Handler: UpdateReadModel (Span 5, parent=Span 1) ]
```

## Propagating OpenTelemetry Context in Watermill

Implement OpenTelemetry's `propagation.TextMapCarrier` over `message.Metadata` so standard W3C Trace Context headers flow automatically through Redis Streams, Kafka, or PostgreSQL.

```go
package tracing

import (
	"github.com/ThreeDotsLabs/watermill/message"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/codes"
	"go.opentelemetry.io/otel/trace"
)

type metadataCarrier message.Metadata

func (c metadataCarrier) Get(key string) string { return message.Metadata(c).Get(key) }
func (c metadataCarrier) Set(key, val string)   { message.Metadata(c).Set(key, val) }
func (c metadataCarrier) Keys() []string {
	keys := make([]string, 0, len(c))
	for k := range c {
		keys = append(keys, k)
	}
	return keys
}

// PublisherDecorator injects the active span context into outgoing message metadata.
type PublisherDecorator struct {
	message.Publisher
}

func (p PublisherDecorator) Publish(topic string, messages ...*message.Message) error {
	for _, msg := range messages {
		otel.GetTextMapPropagator().Inject(msg.Context(), metadataCarrier(msg.Metadata))
	}
	return p.Publisher.Publish(topic, messages...)
}

// Middleware extracts trace context from incoming metadata and starts a consumer span.
func Middleware(tracer trace.Tracer) message.HandlerMiddleware {
	return func(next message.HandlerFunc) message.HandlerFunc {
		return func(msg *message.Message) ([]*message.Message, error) {
			ctx := otel.GetTextMapPropagator().Extract(msg.Context(), metadataCarrier(msg.Metadata))
			handlerName := message.HandlerNameFromCtx(ctx)

			ctx, span := tracer.Start(ctx, "handler."+handlerName,
				trace.WithSpanKind(trace.SpanKindConsumer),
				trace.WithAttributes(attribute.String("messaging.message.id", msg.UUID)),
			)
			defer span.End()

			msg.SetContext(ctx)
			produced, err := next(msg)
			if err != nil {
				span.RecordError(err)
				span.SetStatus(codes.Error, err.Error())
			}
			return produced, err
		}
	}
}
```

## Common Pitfalls

- **Forgetting to set `otel.SetTextMapPropagator`:** By default, OpenTelemetry uses a no-op propagator! Register `propagation.NewCompositeTextMapPropagator(propagation.TraceContext{}, propagation.Baggage{})` at startup or traces will break at every broker hop.
- **Losing trace context in the Outbox forwarder:** Ensure your outbox table preserves `message.Metadata` verbatim so the `traceparent` header survives the hop from PostgreSQL to Redis/Kafka.

## Related Chapters

- Previous: [17. Metrics and Alerting](17-metrics-and-alerting.md)
- Next: [19. Fault Tolerance](19-fault-tolerance.md)
- Book foundations: [Tracing and Context Propagation](../19-observability-sre/163-tracing-context-propagation.md)
