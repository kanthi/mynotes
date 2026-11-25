---
title: '02. Message Broker'
---

# Message Broker

In-process goroutines and channels cannot survive process restarts, nor can they distribute work across multiple service replicas. A **message broker** decouples producers from consumers, persists messages across restarts, and enables horizontal scaling through consumer groups.

## Learning & Build Goals

- **Learn:** Role of message brokers in distributed systems; Publish/Subscribe (Pub/Sub) messaging; broker-level error handling and retries; consumer groups for scalability and fault tolerance.
- **Build:** Integrate Redis Streams, Kafka, and PostgreSQL as message brokers while keeping the broker an implementation detail; replace in-memory passing with a production broker; refactor handlers to leverage broker features.

## Publish/Subscribe and Consumer Groups

```text
                    +---> [ Consumer Group A: Email Service ]
                    |       ├── Instance 1 (msg 1, msg 3)
Producer -> Topic --+       └── Instance 2 (msg 2, msg 4)
                    |
                    +---> [ Consumer Group B: Analytics Service ]
                            └── Instance 1 (msg 1, msg 2, msg 3, msg 4)
```

- **Pub/Sub fan-out:** Each logical service subscribes with its own consumer group and receives its own copy of every message on the topic.
- **Competing consumers:** Multiple replicas within the *same* consumer group split the topic's messages among themselves for horizontal scaling and failover.
- **Acknowledgment (`Ack` / `Nack`):** A consumer acknowledges a message only after finishing its work. If the consumer crashes or sends `Nack`, the broker redelivers the message.

## Comparing Brokers Used in This Track

| Broker | Strengths | Trade-offs | Typical Fit |
|--------|-----------|------------|-------------|
| **Redis Streams** | Low latency, simple ops, built-in consumer groups (`XREADGROUP`) | Memory-bound durability unless carefully tuned | Fast queues, smaller to medium workloads |
| **Apache Kafka** | High throughput, partitioned append-only log, long retention & replay | Heavier operational footprint | High-volume event streams, data lakes |
| **PostgreSQL** | Zero extra infrastructure, shares ACID transactions with app tables | Lower peak throughput than Kafka; polling overhead | Transactional outbox, small/medium services |

## Keeping the Broker an Implementation Detail

Your business logic should depend on a minimal `Publisher` / `Subscriber` abstraction, not on Redis, Kafka, or SQL client types.

```go
package messaging

import "context"

type Message struct {
	UUID     string
	Metadata map[string]string
	Payload  []byte
}

type Publisher interface {
	Publish(ctx context.Context, topic string, messages ...*Message) error
	Close() error
}

type Subscriber interface {
	Subscribe(ctx context.Context, topic string) (<-chan *Message, error)
	Close() error
}
```

With this boundary in place, switching from an in-memory Go channel implementation in unit tests to Redis Streams in staging or Kafka in production requires changing only the dependency wiring in `main.go`.

## Common Pitfalls

- **Acking before processing:** Always `Ack` *after* durable work completes. Acking early causes silent data loss on crash.
- **Sharing one consumer group across unrelated services:** If Email and Billing share the same consumer group name, each message is delivered to only one of them—half your side effects silently disappear.
- **Leaking broker types into domain packages:** Keep broker serialization, topic naming, and client configuration in an adapter layer.

## Related Chapters

- Previous: [01. Basic Async](01-basic-async.md)
- Next: [03. Events](03-events.md)
- Book foundations: [Durable Distributed Message Queue](../18-distributed-infra/153b-distributed-message-queue.md)
