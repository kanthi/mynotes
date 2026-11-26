---
title: Intro
---

# Intro

This part is a structured Go curriculum and engineering reference for **Event-Driven Architecture**, organized around the 20-module syllabus of **Go Event-Driven v2.0** (Three Dots Labs, [threedots.tech/event-driven](https://threedots.tech/event-driven/)).

The running project across all 20 modules is a **ticket aggregator startup** that starts as a synchronous HTTP service and evolves step by step into a scalable, resilient, observable event-driven system. Although the reference stack uses **Watermill**, **Redis Streams**, **Kafka**, and **PostgreSQL**, the patterns are broker-agnostic and transfer directly to NATS, RabbitMQ, AWS SNS/SQS, or Google Pub/Sub.

> [!NOTE]
> These chapters provide original Go explanations, architecture sketches, and idiomatic implementation patterns mapped to each module's learning and build objectives. They complement the hands-on exercises run locally via the `tdl` CLI.

## Course Overview & Metadata

| Attribute | Details |
|-----------|---------|
| **Curriculum** | Go Event-Driven (v2.0) — *Learn building Extremely Scalable & Resilient Go backends* |
| **Creators** | Miłosz Smółka & Robert Laszczak (Three Dots Labs, authors of Watermill) |
| **Target Level** | Intermediate to advanced Go developers |
| **Format** | Hands-on, local IDE workflow (`tdl training run`) with automated test verification; usable solo or with an AI coding agent |
| **Scale** | 20 modules, 94 exercises, ~20+ coding hours |
| **Running Project** | Ticket aggregator startup grown from synchronous MVP to production event-driven architecture |
| **Brokers Covered** | Redis Streams, Apache Kafka, PostgreSQL (kept as an implementation detail behind clean interfaces) |
| **Router Library** | Watermill (used as the message router/plumbing; patterns remain portable across libraries and languages) |

## Architecture Evolution

```text
Modules 01–04: Foundations
  Sync HTTP -> Goroutine workers -> Message Broker (Redis/Kafka/PG) -> Typed Events -> Watermill Router

Modules 05–08: Production Plumbing & CQRS
  Middlewares (logging, correlation) -> Error classification & retries -> Component tests -> CQRS Event Bus

Modules 09–12: Consistency & Projections
  At-least-once & idempotency -> Transactional Outbox -> CQRS Command Bus -> Read Models

Modules 13–16: Ordering, Evolution & Recovery
  Partitioning & versioning -> Central Data Lake -> Internal & Versioned Events -> Rebuilding Read Models

Modules 17–20: Operability & Workflows
  Metrics & Alerting -> Distributed Tracing -> Circuit Breakers & Poison Queues -> Sagas & Process Managers
```

## Module Syllabus & Chapter Map

| # | Module | Core Focus | Chapter |
|---|--------|------------|---------|
| 01 | **Basic Async** | Sync vs. async patterns, goroutine workers, retries | [Basic Async](01-basic-async.md) |
| 02 | **Message Broker** | Pub/Sub, Redis Streams, Kafka, PostgreSQL, consumer groups | [Message Broker](02-message-broker.md) |
| 03 | **Events** | Events vs. messages, payload design, marshalling, headers | [Events](03-events.md) |
| 04 | **Watermill Basics** | Router, business-logic-first handlers, graceful shutdown, health checks | [Watermill Basics](04-watermill-basics.md) |
| 05 | **Middlewares** | Cross-cutting concerns, logging, correlation IDs, dependency injection | [Middlewares](05-middlewares.md) |
| 06 | **Errors** | Transient errors, malformed payloads, code bugs, retry middleware | [Errors](06-errors.md) |
| 07 | **Component Testing** | Running services in tests, external mocks, end-to-end verification | [Component Testing](07-component-testing.md) |
| 08 | **Events with CQRS Pattern** | CQRS fundamentals, Watermill EventBus, event processors, scaling | [Events with CQRS Pattern](08-events-with-cqrs.md) |
| 09 | **At Least Once Delivery** | Redelivery guarantees, idempotent handlers, idempotent repositories | [At Least Once Delivery](09-at-least-once-delivery.md) |
| 10 | **Outbox** | Transactional outbox, atomic DB + event writes, outbox forwarder | [Outbox](10-outbox.md) |
| 11 | **Commands in CQRS Pattern** | Commands vs. events, CommandBus, command handlers, topic separation | [Commands in CQRS Pattern](11-commands-in-cqrs.md) |
| 12 | **Read Models** | Projections from events, query-optimized schemas, read APIs | [Read Models](12-read-models.md) |
| 13 | **Message Ordering** | Race conditions, partitioning, entity versioning, out-of-order handling | [Message Ordering](13-message-ordering.md) |
| 14 | **Data Lake** | Central event archive, broker-to-lake forwarding, replay & analytics | [Data Lake](14-data-lake.md) |
| 15 | **Internal and Versioned Events** | Schema evolution, backward compatibility, service decoupling | [Internal and Versioned Events](15-internal-and-versioned-events.md) |
| 16 | **Migrating Read Models** | Schema migrations, rebuilding projections from the data lake | [Migrating Read Models](16-migrating-read-models.md) |
| 17 | **Metrics and Alerting** | Prometheus instrumentation, queue/handler metrics, actionable alerts | [Metrics and Alerting](17-metrics-and-alerting.md) |
| 18 | **Tracing** | OpenTelemetry distributed tracing, context propagation across brokers | [Tracing](18-tracing.md) |
| 19 | **Fault Tolerance** | Retries, circuit breakers, poison queues (dead-letter queues) | [Fault Tolerance](19-fault-tolerance.md) |
| 20 | **Sagas and Process Managers** | Long-running workflows, orchestration vs. choreography, compensations | [Sagas and Process Managers](20-sagas-and-process-managers.md) |

## Key Design Takeaways (FAQ Highlights)

- **Not Watermill-locked:** Watermill is used as the router and CQRS helper so you focus on architecture instead of broker boilerplate. Every pattern—outbox, idempotency, read models, poison queues, sagas—applies identically to raw Kafka/NATS/SQS clients or other languages.
- **Broker as an implementation detail:** Domain handlers should never import Redis, Kafka, or SQL driver specifics. Swapping brokers becomes a wiring change at startup.
- **Real-world progression:** Start simple (in-process async) and introduce brokers, outboxes, data lakes, and sagas only when the corresponding failure mode or scaling bottleneck appears.

## Related Parts in This Book

- [Distributed Infrastructure](../18-distributed-infra/150-distributed-infra-index.md) — deep dives on [retry & circuit breakers](../18-distributed-infra/152-retry-circuit-breaker-patterns.md), [idempotency keys](../18-distributed-infra/155-idempotency-keys.md), [transactional outbox](../18-distributed-infra/157-outbox-reliable-side-effects.md), and [durable message queues](../18-distributed-infra/153b-distributed-message-queue.md).
- [Observability and SRE](../19-observability-sre/160-observability-sre-index.md) — [structured logging & correlation](../19-observability-sre/161-structured-logging-correlation.md), [Prometheus metrics](../19-observability-sre/162-metrics-prometheus-design.md), [OpenTelemetry tracing](../19-observability-sre/163-tracing-context-propagation.md), and [health/readiness probes](../19-observability-sre/165-health-readiness-probes.md).
- [Concurrency Ground Up](../10-concurrency-ground-up/250-concurrency-ground-up-index.md) — goroutines, channels, `context.Context`, and backpressure fundamentals.
