---
title: '07. Project: Unified OpenTelemetry Stack, Cardinality Guardrails & SLO Burn Alerts'
---

# Project: Unified OpenTelemetry Stack, Cardinality Guardrails & SLO Burn Alerts

Instrument a multi-service Go + Python application with all four observability signals—**logs, metrics, traces, and continuous eBPF profiling (Pyroscope / Parca)**—routed through an **OpenTelemetry Collector** configured with **cardinality/cost guardrails** (dropping high-cardinality attributes before Prometheus/Mimir, Loki, and Tempo) and **multi-window SLO burn-rate alerts** (Sloth / Pyrra) linked to Markdown runbooks.

## Collector Cardinality Guardrail (`otel-collector.yaml`)

```yaml
processors:
  batch:
    timeout: 2s
    send_batch_size: 2048
  # Drop unbounded user/request IDs from metric attributes to prevent Prometheus OOM.
  transform/drop_high_cardinality:
    metric_statements:
      - context: datapoint
        statements:
          - delete_key(attributes, "user_id")
          - delete_key(attributes, "request_id")
          - delete_key(attributes, "http.target")

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [transform/drop_high_cardinality, batch]
      exporters: [prometheusremotewrite]
```

## Multi-Window SLO Burn-Rate Alerting

Instead of noisy static CPU/latency thresholds, alert on **error budget burn rate**:
- **Page (Fast Burn):** 2% of 30-day error budget consumed in 1 hour (`burn_rate > 14.4` over 5m & 1h windows).
- **Ticket (Slow Burn):** 5% of 30-day error budget consumed in 6 hours (`burn_rate > 6` over 30m & 6h windows).

## Acceptance Drills

- Emit synthetic metrics with 50,000 unique `user_id` labels and verify the OTel Collector drops `user_id` before storage so active Prometheus time series remain flat.
- Inject 2% HTTP 500 errors for 10 minutes and confirm the fast-burn SLO alert fires with a linked runbook URL and correlated Tempo trace + Pyroscope CPU profile.
