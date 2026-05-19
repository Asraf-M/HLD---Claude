# Message Queue (Kafka)

## What Is It?

A **message queue** is a system that lets services communicate **asynchronously** — the sender publishes a message and moves on; the receiver processes it whenever ready.

**Apache Kafka** is the most widely-used distributed message queue/streaming platform.

> **Analogy:** A restaurant order ticket system. The waiter (producer) drops order tickets onto a rail. The kitchen (consumer) picks up tickets and cooks. The waiter doesn't stand there waiting — they go take more orders. If the kitchen is slow, tickets pile up on the rail (queue) instead of blocking the waiter.

---

## Why Use a Message Queue?

| Problem (without queue) | Solution (with queue) |
|------------------------|----------------------|
| Service A calls Service B directly → B is slow, A is stuck | A publishes message → moves on; B processes when ready |
| Traffic spike overwhelms downstream | Queue absorbs the spike; consumers process at their pace |
| Service B crashes → A loses the request | Message stays in queue; B processes after recovery |
| One event needs to go to multiple services | Multiple consumers subscribe to same topic |

**Core benefit:** **Decoupling** — producers and consumers are independent.

---

## Kafka Core Concepts

### Topic
A named stream of messages. Like a category or feed.

```
topic: "user-signups"
topic: "order-placed"
topic: "payment-processed"
```

### Partition
Each topic is split into **partitions** — ordered, append-only logs on disk.

```
Topic: "order-placed"
  Partition 0: [msg0, msg1, msg4, msg7, ...]
  Partition 1: [msg2, msg5, msg8, ...]
  Partition 2: [msg3, msg6, msg9, ...]
```

Why partition?
- **Parallelism** — different consumers read different partitions simultaneously
- **Scalability** — spread partitions across multiple brokers

### Offset
Each message in a partition has a sequential integer **offset** (position in the log).

```
Partition 0:
  offset 0: { orderId: 1, userId: 42, amount: 50 }
  offset 1: { orderId: 2, userId: 17, amount: 30 }
  offset 2: { orderId: 3, userId: 99, amount: 75 }
```

Consumers track which offset they've read up to → allows replay, resume after crash.

### Broker
A Kafka server node. A Kafka **cluster** = multiple brokers.

```
Kafka Cluster:
  Broker 1 → stores Partition 0 of topic A, Partition 2 of topic B
  Broker 2 → stores Partition 1 of topic A, Partition 0 of topic B
  Broker 3 → stores Partition 2 of topic A, Partition 1 of topic B
```

### Producer
Publishes messages to a topic. Chooses which partition via:
- **Round-robin** (for even distribution)
- **Key-based** — `hash(key) % num_partitions` — same key always goes to same partition

```
producer.send("order-placed", key=userId, value=orderData)
# Same userId always → same partition → ordering preserved per user
```

### Consumer Group
A group of consumers that **collectively** read a topic. Each partition is assigned to exactly one consumer in the group at a time.

```
Topic: "order-placed" (3 partitions)
Consumer Group "payment-service":
  Consumer A → reads Partition 0
  Consumer B → reads Partition 1
  Consumer C → reads Partition 2
```

**Scaling:** add more consumers to a group → Kafka rebalances partitions.  
**Max parallelism:** limited by number of partitions (can't have more active consumers than partitions).

Multiple consumer groups can read the same topic independently:
```
Consumer Group "payment-service" ← topic "order-placed" → Consumer Group "notification-service"
```

---

## Message Flow

```
Producer → Kafka Broker (Leader Partition)
                ↓ replicated to
           Follower Brokers
                ↓ consumer pulls
Consumer Group (reads at its own pace)
```

### Producer Acknowledgment (ACK)

| `acks` setting | Meaning | Risk |
|----------------|---------|------|
| `acks=0` | Fire and forget, no wait | Message may be lost |
| `acks=1` | Wait for leader to write | Safe unless leader crashes before replication |
| `acks=all` | Wait for all replicas to write | Slowest but no data loss |

---

## Replication & Durability

Each partition has:
- 1 **Leader** — handles all reads/writes
- N-1 **Followers** — replicate from leader

**ISR (In-Sync Replicas):** followers that are caught up with the leader.

If the leader crashes, Kafka elects a new leader from the ISR set. Data is safe as long as at least 1 ISR exists.

```
Replication factor = 3:
  Leader (Broker 1) → Follower (Broker 2) + Follower (Broker 3)
  Broker 1 crashes → Broker 2 becomes new leader
  Consumers continue reading without interruption
```

---

## Retention Policy

Kafka keeps messages for a configured time (default: 7 days), regardless of whether they've been consumed.

```
log.retention.hours=168  (7 days)
log.retention.bytes=1073741824  (1 GB per partition)
```

This enables:
- **Consumer replay** — reprocess old messages
- **New services** — subscribe and process all historical messages
- **Debugging** — inspect what happened in the past

---

## Pull vs Push

Kafka uses **pull** — consumers ask the broker for messages.

| | Pull (Kafka) | Push |
|--|-------------|------|
| Consumer pace | Consumer controls | Broker controls |
| Back pressure | Natural — slow consumer just polls less | Can overwhelm slow consumers |
| Complexity | Slightly more code | Broker must track each consumer |

---

## Delivery Semantics

| Semantic | Guarantee | How |
|----------|-----------|-----|
| **At-most-once** | Message delivered 0 or 1 time — may be lost | Commit offset before processing |
| **At-least-once** | Message delivered ≥1 time — may be duplicated | Commit offset after processing |
| **Exactly-once** | Message delivered exactly once | Kafka transactions + idempotent producers |

**Most common in production:** at-least-once + idempotent consumers (deduplicate by message ID).

---

## Consumer Rebalancing

When a consumer joins or leaves a consumer group, Kafka **rebalances** partitions:

```
Before: Consumer A (P0, P1), Consumer B (P2)
Consumer C joins →
After:  Consumer A (P0), Consumer B (P1), Consumer C (P2)
```

During rebalancing: consumption pauses briefly. Use sticky assignors to minimize moves.

---

## Kafka vs Traditional Message Queues (RabbitMQ)

| | Kafka | RabbitMQ |
|--|-------|---------|
| Model | Log-based (consumers read from offset) | Queue-based (message deleted after delivery) |
| Retention | Days/weeks | Until consumed |
| Replay | ✅ Yes | ❌ No |
| Throughput | Very high (millions/sec) | High (thousands/sec) |
| Ordering | Per-partition | Per-queue |
| Use case | Event streaming, analytics | Task queues, RPC |

---

## When to Use Kafka

- **Event-driven architecture** — microservices reacting to events
- **Activity tracking** — user clicks, page views, audit logs
- **Log aggregation** — collect logs from many services
- **Stream processing** — real-time analytics pipelines
- **Change Data Capture (CDC)** — replicate DB changes downstream
- **Metrics pipeline** — time-series metrics to monitoring systems

---

## Common System Design Patterns

### Fan-out
```
"user-signup" topic
  → Consumer Group "email-service"    (send welcome email)
  → Consumer Group "analytics-service" (track signup event)
  → Consumer Group "billing-service"  (create free tier account)
```

One event triggers multiple independent workflows.

### Work Queue (Load Distribution)
```
"video-encoding" topic (1000 jobs)
  Consumer Group "encoders" (10 workers)
    → each worker gets ~100 jobs automatically
```

Add more workers → more parallelism automatically.

### Event Sourcing
Store all state changes as events. Rebuild state by replaying events from offset 0.

---

## Interview Q&A

**Q: What is a message queue and why do we use it?**  
A: Asynchronous communication between services. Producer publishes a message and moves on; consumer processes it independently. Benefits: decoupling (services don't need to be up simultaneously), load leveling (absorbs traffic spikes), fault tolerance (messages persist if consumer crashes), fan-out (one event to many consumers).

**Q: What is a Kafka partition and why does it matter?**  
A: A partition is an ordered, append-only log. A topic is split into N partitions spread across brokers. Partitions enable parallelism (different consumers read different partitions simultaneously) and scalability (spread load across machines). Messages with the same key always go to the same partition, preserving per-key ordering.

**Q: What is a consumer group?**  
A: A set of consumers that collectively consume a topic. Each partition is assigned to exactly one consumer in the group at a time. Kafka auto-rebalances partitions when consumers join or leave. Multiple consumer groups can independently consume the same topic at different offsets.

**Q: What is the difference between at-least-once and exactly-once delivery?**  
A: At-least-once: offset committed after processing — if consumer crashes before committing, message is reprocessed → possible duplicates. Exactly-once: uses Kafka transactions and idempotent producers to ensure each message is processed exactly once — more complex and slightly slower. In practice, at-least-once with idempotent consumers (deduplicate by message ID) is the most common approach.

**Q: Why use Kafka over a database for async processing?**  
A: Kafka is optimized for high-throughput sequential writes and reads. It handles millions of messages/second, supports fan-out to multiple consumers, retains messages for replay, and decouples producers from consumers. A DB table as a queue suffers from polling overhead, locking contention, and no native fan-out.
