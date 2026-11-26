---
title: '17. Metrics and Alerting'
---

# Metrics and Alerting

Asynchronous systems fail differently from synchronous APIs: an HTTP endpoint can return `202 Accepted` with 0% error rate while a background consumer is silently stuck, lagging hours behind, or dropping messages into a poison queue. **Metrics and alerts** make asynchronous health visible.

## Learning & Build Goals

- **Learn:** Importance of observability in event-driven backends; instrumenting message handlers and brokers with Prometheus metrics; designing actionable alerts.
- **Build:** Add Prometheus metrics middleware to the ticket aggregator; expose `/metrics`; configure alerts for handler failures, latency, and consumer lag.

## Essential Event-Driven Metrics

| Metric | Type | Labels | What It Detects |
|--------|------|--------|-----------------|
| `messages_processed_total` | Counter | `handler`, `topic`, `status` (`ok`/`error`) | Error spikes, silent stalls (rate drops to 0) |
| `message_handling_duration_seconds` | Histogram | `handler`, `topic` | Slow handlers, downstream DB/API degradation |
| `messages_published_total` | Counter | `topic`, `status` | Producer publish errors, traffic surges |
| `consumer_lag` / `oldest_pending_age_seconds` | Gauge | `consumer_group`, `topic` | Consumers falling behind producers |
| `poison_queue_messages_total` | Counter | `handler` | Unrecoverable messages requiring intervention |

> [!WARNING]
> Keep metric label cardinality strictly bounded. Never put `ticket_id`, `booking_id`, `customer_email`, or `message_uuid` in Prometheus labels—they will explode memory usage and crash your metrics server.

## Prometheus Metrics Middleware for Watermill

```go
package metrics

import (
	"time"

	"github.com/ThreeDotsLabs/watermill/message"
	"github.com/prometheus/client_golang/prometheus"
)

type HandlerMetrics struct {
	processed *prometheus.CounterVec
	duration  *prometheus.HistogramVec
}

func NewHandlerMetrics(reg prometheus.Registerer) *HandlerMetrics {
	m := &HandlerMetrics{
		processed: prometheus.NewCounterVec(prometheus.CounterOpts{
			Name: "messages_processed_total",
			Help: "Total number of messages processed by handler and status.",
		}, []string{"handler", "status"}),
		duration: prometheus.NewHistogramVec(prometheus.HistogramOpts{
			Name:    "message_handling_duration_seconds",
			Help:    "Duration of message handling in seconds.",
			Buckets: prometheus.DefBuckets,
		}, []string{"handler"}),
	}
	reg.MustRegister(m.processed, m.duration)
	return m
}

func (m *HandlerMetrics) Middleware(next message.HandlerFunc) message.HandlerFunc {
	return func(msg *message.Message) ([]*message.Message, error) {
		start := time.Now()
		handler := message.HandlerNameFromCtx(msg.Context())

		produced, err := next(msg)
		status := "ok"
		if err != nil {
			status = "error"
		}

		m.processed.WithLabelValues(handler, status).Inc()
		m.duration.WithLabelValues(handler).Observe(time.Since(start).Seconds())
		return produced, err
	}
}
```

## Actionable Alerting Rules

Alert on **symptoms that threaten user experience or data freshness**, not noisy low-level fluctuations:

- **High handler error rate:** Error ratio > 5% for 5 minutes on any handler.
- **Poison queue growth:** `increase(poison_queue_messages_total[5m]) > 0` (immediate warning).
- **Consumer lag / oldest message age:** Oldest unprocessed message > SLA threshold (e.g., 5 minutes).
- **Outbox backlog:** Unforwarded rows in the outbox table growing steadily over 5 minutes.

## Related Chapters

- Previous: [16. Migrating Read Models](16-migrating-read-models.md)
- Next: [18. Tracing](18-tracing.md)
- Book foundations: [Prometheus Metrics Design](../19-observability-sre/162-metrics-prometheus-design.md), [Alerting and Runbooks](../19-observability-sre/166-alerting-and-runbooks.md)
