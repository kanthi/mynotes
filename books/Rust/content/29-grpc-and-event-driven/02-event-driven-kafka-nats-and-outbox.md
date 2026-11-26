---
title: Event-Driven Systems: Kafka, NATS, and the Transactional Outbox
---

# Event-Driven Systems: Kafka, NATS, and the Transactional Outbox

When Service A updates its PostgreSQL database and then calls `broker.publish(event)`, a crash between those two lines leaves your system permanently inconsistent: the database committed, but downstream billing or search services never received the event. The **transactional outbox** pattern paired with **idempotent consumers** eliminates dual-write bugs completely.

## Choosing Between Kafka (`rdkafka`) and NATS JetStream (`async-nats`)

| Dimension | Apache Kafka (`rdkafka`) | NATS JetStream (`async-nats`) |
| :--- | :--- | :--- |
| **Rust integration** | Wraps C `librdkafka` (battle-tested, C build dep) | 100% pure Rust async client |
| **Ordering & scale** | Partition-ordered durable append log; massive throughput | Subject-based streams, wildcards, built-in KV & object store |
| **Operational weight** | Heavy cluster footprint (KRaft brokers) | Single static binary, sub-millisecond latency |
| **Best fit** | High-volume event streaming, CDC, analytics pipelines | Microservice RPC, edge fleets, work queues, real-time fan-out |

## Idempotent Consumer Pattern

Because every durable broker guarantees **at-least-once** delivery (a worker can crash after processing a message but before sending `ACK`), every consumer must be safe to run twice for the same `event_id`:

```rust
use std::collections::HashSet;

#[derive(Debug, Clone)]
pub struct Event {
    pub event_id: String,
    pub order_id: String,
    pub amount_cents: u64,
}

pub struct IdempotentLedger {
    seen_events: HashSet<String>,
    total_cents: u64,
}

impl IdempotentLedger {
    pub fn new() -> Self {
        Self { seen_events: HashSet::new(), total_cents: 0 }
    }

    pub fn apply(&mut self, evt: &Event) -> bool {
        // In production: INSERT INTO processed_events(event_id) ON CONFLICT DO NOTHING
        if !self.seen_events.insert(evt.event_id.clone()) {
            return false; // Duplicate redelivery safely ignored
        }
        self.total_cents += evt.amount_cents;
        true
    }
}
```

## Transactional Outbox Table Schema

Write both the domain entity and its outgoing event inside one atomic PostgreSQL transaction:

```sql
CREATE TABLE outbox_events (
  id          BIGSERIAL PRIMARY KEY,
  event_id    UUID NOT NULL UNIQUE,
  topic       TEXT NOT NULL,
  payload     JSONB NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  published_at TIMESTAMPTZ
);

CREATE INDEX idx_outbox_unpublished
  ON outbox_events (id) WHERE published_at IS NULL;
```

A relay worker claims batches using `SELECT ... FROM outbox_events WHERE published_at IS NULL ORDER BY id LIMIT 100 FOR UPDATE SKIP LOCKED`, publishes to NATS/Kafka, and marks `published_at = now()`.

> [!TIP]
> Always route messages that fail deserialization or business invariants after N retries to a **Dead-Letter Queue (DLQ)** topic with the original payload and error reason—never let one poison-pill message block an entire Kafka partition forever.

## Hands-On Lab: Verify Duplicate Event Redelivery Safety

```rust
fn main() {
    let mut ledger = IdempotentLedger::new();
    let e1 = Event { event_id: "evt-001".into(), order_id: "ord-9".into(), amount_cents: 2500 };

    assert!(ledger.apply(&e1));  // First delivery applied
    assert!(!ledger.apply(&e1)); // Redelivery deduplicated
    assert_eq!(ledger.total_cents, 2500);
    println!("Idempotent ledger total: {} cents (duplicates ignored)", ledger.total_cents);
}
```
