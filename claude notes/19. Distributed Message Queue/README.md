# System Design: Distributed Message Queue (Beginner-Friendly Guide)

---

## What Is a Message Queue?

Imagine a restaurant:
- The **waiter** (producer) takes your order and puts a ticket on the kitchen rail.
- The **kitchen** (consumer) processes tickets in order, one by one.
- The **ticket rail** is the message queue.

The waiter doesn''t wait for the kitchen to finish cooking. He takes the next order. The kitchen doesn''t get overwhelmed because they work through tickets at their own pace.

**That''s exactly what a message queue does in software.**

Real examples:
- When you place an Amazon order, a message goes into a queue. The payment service, warehouse service, and notification service each independently process that same event.
- When you upload a video to YouTube, a message is queued. Video transcoding, thumbnail generation, and CDN distribution all happen asynchronously from independent consumers.

**Popular implementations:** Kafka, RabbitMQ, RocketMQ, Apache Pulsar, AWS SQS

> **Note:** Kafka and Pulsar are technically *event streaming platforms*, but they''ve converged with message queues in features. In this chapter, we design something closer to Kafka — with **message retention**, **ordering guarantees**, and **repeated consumption**.

---

## Benefits of Using a Message Queue

<img src="../../system-design-notes/19. Distributed Message Queue/images/message-queue-components.png" alt="Message Queue Components" width="450">

| Benefit | What it means |
|---------|---------------|
| **Decoupling** | Producer and consumer don''t know about each other; change one without breaking the other |
| **Independent scaling** | Producer traffic spikes? Scale producers. Consumer is slow? Scale consumers. Independently. |
| **Fault tolerance** | If consumer crashes, messages stay in the queue; consumer resumes when it recovers |
| **Async performance** | Producer sends and moves on; no waiting for downstream processing |

---

## Step 1: Design Scope

| Requirement | Detail |
|-------------|--------|
| Message format | Text only, a few KB each |
| Repeated consumption? | **Yes** — multiple consumers can each read the same message |
| Ordering guarantee? | **Yes** — messages consumed in same order as produced |
| Data retention | 2 weeks |
| Throughput | Both high-throughput (log aggregation) and low-throughput (task queues) — configurable |
| Delivery semantics | At-least-once required; ideally all three are configurable |

> **Why is repeated consumption hard?** In traditional queues (like a ticket rail), once the kitchen grabs a ticket, it''s gone. Supporting "multiple kitchens each getting their own copy" — while maintaining order — requires a fundamentally different storage design.

---

## Step 2: Core Concepts Before We Design

### Messaging Models

#### Point-to-Point

<img src="../../system-design-notes/19. Distributed Message Queue/images/point-to-point-model.png" alt="Point to Point Model" width="450">

- One producer → one consumer
- Message consumed **once**, then **deleted**
- Multiple consumers compete for messages (one wins)
- Traditional message queues (RabbitMQ, SQS)

#### Publish-Subscribe (Pub/Sub)

<img src="../../system-design-notes/19. Distributed Message Queue/images/publish-subscribe-model.png" alt="Publish-Subscribe Model" width="450">

- Producer publishes to a **topic** (a named channel)
- Multiple consumers can subscribe to the same topic
- **Each subscriber gets every message** (no competition)
- Kafka, Pulsar, SNS

> **Why pub/sub for our design?** We need multiple independent services to each consume the same events. A single `order-placed` event should reach the billing service, the inventory service, and the notification service independently — without any of them knowing the others exist.

### Topics, Partitions, and Brokers

<img src="../../system-design-notes/19. Distributed Message Queue/images/partitions.png" alt="Partitions" width="450">

**The problem with one giant queue per topic:** If 10 million messages per second flow into one topic, one machine can''t handle it.

**Solution: Partition the topic** (shard it horizontally)

```
Topic: "orders"
  ├── Partition 0:  [msg_1] [msg_5] [msg_9]  → Broker A
  ├── Partition 1:  [msg_2] [msg_6] [msg_10] → Broker B
  └── Partition 2:  [msg_3] [msg_7] [msg_11] → Broker C
```

Key concepts:
- **Partition** = an ordered, append-only log (like a WAL)
- **Broker** = the server hosting one or more partitions
- **Offset** = the position of a message within a partition (message #0, #1, #2...)
- **Partition key** = determines which partition a message goes to (`hash(key) % numPartitions`)

> **Why partition key matters:** If you use `user_id` as the partition key, all messages from the same user go to the same partition → guaranteed order for that user. If you need global order, you''d have to use a single partition — but that kills parallelism.

### Consumer Groups

<img src="../../system-design-notes/19. Distributed Message Queue/images/consumer-groups.png" alt="Consumer Groups" width="450">

A **consumer group** is a set of consumers that work together to consume a topic:
- Each partition is assigned to **exactly one consumer** in the group
- Different groups are **completely independent** — they each maintain their own offset
- Adding a consumer group for a new service costs nothing; existing consumers are unaffected

```
Topic "orders" with 3 partitions:

Consumer Group "billing":
  consumer_1 → Partition 0
  consumer_2 → Partition 1
  consumer_3 → Partition 2

Consumer Group "inventory":    ← Independent! Same messages, own offsets
  consumer_A → Partition 0
  consumer_B → Partition 1 + 2
```

> **Why can''t you have more consumers than partitions?** A partition can only be assigned to one consumer per group. If you have 5 consumers but 3 partitions, 2 consumers sit idle.

---

## Step 3: High-Level Architecture

<img src="../../system-design-notes/19. Distributed Message Queue/images/high-level-architecture.png" alt="High Level Architecture" width="500">

```
[Producers]
     ↓
[Routing Layer (embedded in producer)]
     ↓
[Brokers] ← hold partitions, replicas
     |
  ┌──────────────────────┐
  ↓                      ↓
[Data Storage]     [Coordination Service]
(WAL files on disk) (ZooKeeper)
                         |
                    ┌────┴───────────┐
                    ↓                ↓
             [State Storage]  [Metadata Storage]
             (consumer offsets) (topic configs)
                    ↑
             [Consumers / Consumer Groups]
```

| Component | Responsibility |
|-----------|---------------|
| **Broker** | Stores partitions; handles reads/writes from producers and consumers |
| **Data Storage** | Append-only WAL files on disk, split into segments |
| **State Storage** | Consumer offsets (where each group is up to) |
| **Metadata Storage** | Topic configs, partition assignments, replica plans |
| **Coordination Service (ZooKeeper)** | Service discovery, leader election, metadata + state storage |

---

## Step 4: Data Storage — The Write-Ahead Log (WAL)

### Why Not Use a Database?

| Storage Type | Problem |
|-------------|---------|
| SQL Database | Designed for random read/write; struggles with sequential append at high velocity |
| NoSQL | Better, but adds overhead for features we don''t need (indexes, queries) |

Our access pattern is:
- **Append-only writes** (no updates, no deletes)
- **Sequential reads** (read messages from offset X forward)

**Perfect match: Write-Ahead Log (WAL)** — a plain file you can only append to.

<img src="../../system-design-notes/19. Distributed Message Queue/images/wal-example.png" alt="WAL Example" width="450">

```
Partition 0 WAL:
[msg_0][msg_1][msg_2]...[msg_999] ← Segment 1 (read-only, full)
[msg_1000][msg_1001]...           ← Segment 2 (active, accepting writes)
```

- **Segments** prevent one giant file; old segments are read-only
- Sequential disk access is **fast**: HDDs reach ~200 MB/s sequential vs ~2 MB/s random
- The OS aggressively caches hot segments in RAM — recent messages served from memory

> **Common misconception:** "Disk is slow." Only for *random* access. Sequential appends (WAL) are so fast that Kafka commonly saturates network bandwidth before disk becomes a bottleneck.

### Message Structure

<img src="../../system-design-notes/19. Distributed Message Queue/images/message-structure.png" alt="Message Structure" width="450">

| Field | Purpose |
|-------|---------|
| `key` | Determines partition (`hash(key) % N`); can be null (round-robin) |
| `value` | The actual payload (text, JSON, binary, compressed) |
| `topic` | Which topic this message belongs to |
| `partition` | Which partition |
| `offset` | Position within partition |
| `timestamp` | When stored |
| `size` | Byte count |
| `CRC` | Checksum — detect data corruption |

> **Keys don''t have to be unique!** Multiple messages with the same key (e.g., `user_id=42`) just all land in the same partition in order.

---

## Step 5: Producer Flow — Sending Messages

### Routing Layer (Embedded in Producer)

<img src="../../system-design-notes/19. Distributed Message Queue/images/routing-layer.png" alt="Routing Layer" width="400">

Early design: separate routing service between producer and broker.

**Problem:** Extra network hop, can''t batch messages efficiently.

**Better design: embed routing in the producer:**

<img src="../../system-design-notes/19. Distributed Message Queue/images/routing-layer-producer.png" alt="Routing Layer in Producer" width="450">

- Producer fetches partition metadata from ZooKeeper (cached locally)
- Producer calculates which partition the message belongs to
- Producer **buffers messages in memory** and sends in batches
- One network request carries many messages → huge throughput improvement

### Batching Trade-off

<img src="../../system-design-notes/19. Distributed Message Queue/images/batch-size-throughput-vs-latency.png" alt="Batch Size vs Throughput vs Latency" width="450">

| Batch Size | Throughput | Latency | Use Case |
|-----------|-----------|---------|---------|
| Large | High | High (wait to fill batch) | Log aggregation, analytics |
| Small | Lower | Low (send immediately) | Payment events, alerts |

This is configurable per producer — same message queue infrastructure works for both.

---

## Step 6: Consumer Flow — Reading Messages

### Pull vs. Push

> **Question:** Should the broker *push* messages to consumers, or should consumers *pull* from the broker?

**Push model:**
- ✅ Low latency (message arrives instantly)
- ❌ If consumer is slow, broker overwhelms it with more messages than it can handle
- ❌ Broker can''t know how many messages each consumer can handle

**Pull model:**
- ✅ Consumer controls its own rate
- ✅ Consumer can fetch large batches when ready
- ✅ Slow consumers just fall behind — they don''t crash
- ❌ Slight latency (poll interval), extra requests when no new messages (mitigated by long-polling)

**We use pull model** (like Kafka).

### How a Consumer Reads

<img src="../../system-design-notes/19. Distributed Message Queue/images/consumer-example.png" alt="Consumer Example" width="450">

Consumer says: "Give me messages from Partition 0, starting at offset 42."  
Broker returns a batch starting from offset 42.  
Consumer processes them, then asks for the next batch starting at offset 42 + batch_size.

<img src="../../system-design-notes/19. Distributed Message Queue/images/consumer-flow.png" alt="Consumer Flow" width="450">

**Detailed consumer join flow:**
1. New consumer wants to join Consumer Group "billing"
2. Hash("billing") → finds the correct broker acting as Group Coordinator
3. Consumer sends `JoinGroup` request to coordinator
4. Coordinator assigns partitions (round-robin or range strategy)
5. Consumer fetches from its assigned partition starting from last committed offset
6. Consumer processes messages, then commits offset

> **Why hash the group name?** All consumers in the same group must talk to the same coordinator broker to coordinate. Hashing the group name deterministically picks one broker as the coordinator.

### State Storage — Tracking Offsets

<img src="../../system-design-notes/19. Distributed Message Queue/images/state-storage.png" alt="State Storage" width="450">

```
Consumer Group "billing":
  Partition 0: offset 6 ← consumed messages 0-5
  Partition 1: offset 12
  Partition 2: offset 8
```

Stored in ZooKeeper (or Kafka''s own `__consumer_offsets` internal topic).

> **Why this matters:** If consumer crashes and restarts, it resumes from its last committed offset. No data lost, no duplicate processing (as long as you commit at the right time).

---

## Step 7: Consumer Rebalancing

Consumer rebalancing decides **which consumer gets which partition** — and re-runs whenever the group membership changes.

### When Rebalancing Triggers

<img src="../../system-design-notes/19. Distributed Message Queue/images/consumer-rebalancing.png" alt="Consumer Rebalancing" width="450">

- A consumer joins the group
- A consumer leaves the group
- A consumer stops sending heartbeats (assumed dead)
- A partition is added or removed

<img src="../../system-design-notes/19. Distributed Message Queue/images/consumer-rebalance-example.png" alt="Consumer Rebalance Example" width="450">

### Consumer Joins Group

<img src="../../system-design-notes/19. Distributed Message Queue/images/consumer-join-group-usecase.png" alt="Consumer Joins Group" width="450">

1. Consumer A is alone, consuming all partitions
2. Consumer B sends `JoinGroup` request
3. Coordinator notifies all group members to rebalance (sent back with heartbeat response)
4. All consumers re-join; coordinator elects a **group leader**
5. Group leader calculates new partition assignments
6. Leader sends plan to coordinator; coordinator broadcasts to all consumers
7. Each consumer starts consuming its newly assigned partitions

### Consumer Leaves Group

<img src="../../system-design-notes/19. Distributed Message Queue/images/consumer-leaves-group-usecase.png" alt="Consumer Leaves Group" width="450">

Consumer B gracefully sends `LeaveGroup` → coordinator triggers rebalance → Consumer A takes over B''s partitions.

### Consumer Crashes (No Heartbeat)

<img src="../../system-design-notes/19. Distributed Message Queue/images/consumer-no-heartbeat-usecase.png" alt="Consumer No Heartbeat" width="450">

If coordinator doesn''t receive a heartbeat from a consumer within `session.timeout.ms` → assumes dead → triggers rebalance without waiting.

> **The cost of rebalancing:** During a rebalance, **all consumers in the group pause processing** (stop-the-world). This is why minimizing unnecessary rebalances matters at scale.

---

## Step 8: ZooKeeper — The Coordination Brain

<img src="../../system-design-notes/19. Distributed Message Queue/images/zookeeper.png" alt="ZooKeeper" width="450">

ZooKeeper handles three critical responsibilities:

| Responsibility | Details |
|----------------|---------|
| **Metadata storage** | Topic configs, partition counts, retention policies |
| **State storage** | Consumer group offsets |
| **Leader election** | When a broker replica becomes the leader for a partition |
| **Service discovery** | Which brokers are alive? Which is the leader for partition X? |

With ZooKeeper, brokers only need to worry about storing and serving messages — all coordination is offloaded.

> **Modern note:** Newer Kafka versions replace ZooKeeper with **KRaft** (Kafka Raft) — Kafka manages its own metadata using a Raft consensus log. Same concepts, simpler operations.

---

## Step 9: Replication — Surviving Broker Failures

A single broker storing all partitions = single point of failure. If it goes down, data is lost.

**Solution: Replicate each partition across multiple brokers.**

<img src="../../system-design-notes/19. Distributed Message Queue/images/replication-example.png" alt="Replication Example" width="450">

```
Partition 0:
  Broker A ← LEADER (producer writes here)
  Broker B ← follower (pulls from leader)
  Broker C ← follower (pulls from leader)
```

- Producers always write to the **leader** replica
- Followers **pull** from leader (not pushed to them)
- Once enough followers confirm they have the message → leader sends ACK to producer

### In-Sync Replicas (ISR)

<img src="../../system-design-notes/19. Distributed Message Queue/images/in-sync-replicas-example.png" alt="In-Sync Replicas" width="450">

ISR = the set of replicas that are **caught up with the leader** (within configured lag tolerance).

```
Leader offset: 15
Replica 2: offset 15 ✅ (in ISR)
Replica 3: offset 15 ✅ (in ISR)
Replica 4: offset 11 ❌ (lagging too far → removed from ISR)
```

A message is **committed** only once all ISR replicas have written it. The committed offset is what consumers can safely read.

> **Trade-off:** Strict ISR = high durability, slow writes (wait for slow replicas). Loose ISR = fast writes, risk of data loss if a non-ISR replica becomes leader.

### Acknowledgment Modes

**ACK=all — Highest Durability**

<img src="../../system-design-notes/19. Distributed Message Queue/images/ack-all.png" alt="ACK All" width="400">

All ISR replicas confirm the write → ACK sent to producer.
- Slowest, but **no data loss** even if multiple brokers fail simultaneously

**ACK=1 — Balanced**

<img src="../../system-design-notes/19. Distributed Message Queue/images/ack-1.png" alt="ACK 1" width="400">

Leader confirms it received the write → ACK sent.
- Fast, but if leader crashes before followers pull the message → **message lost**

**ACK=0 — Lowest Durability (Fire and Forget)**

<img src="../../system-design-notes/19. Distributed Message Queue/images/ack-0.png" alt="ACK 0" width="400">

Producer sends and moves on without waiting for any ACK.
- Fastest, but messages can silently be lost
- Acceptable for: metrics, logs where occasional loss is tolerable

---

## Step 10: Broker Scalability

### Broker Failure Recovery

<img src="../../system-design-notes/19. Distributed Message Queue/images/broker-failure-recovery.png" alt="Broker Failure Recovery" width="450">

1. Broker A fails
2. ZooKeeper/KRaft detects failure (no heartbeat)
3. New leader elected from ISR for each affected partition
4. Coordinator redistributes partitions to remaining brokers
5. Those brokers act as followers until they catch up → then rejoin ISR

> **Best practice:** Spread replicas across different brokers (and ideally different racks/data centers) so a single hardware failure can''t take down all replicas of a partition.

### Adding a New Broker

<img src="../../system-design-notes/19. Distributed Message Queue/images/broker-replica-redistribution.png" alt="Broker Replica Redistribution" width="450">

1. New broker joins the cluster
2. Temporarily allow extra replicas while it catches up
3. Once new broker is in ISR, remove the old excess replica
4. Traffic gradually shifts to new broker

### Adding Partitions

<img src="../../system-design-notes/19. Distributed Message Queue/images/partition-exmaple.png" alt="Partition Example" width="450">

New partition is created; producer metadata updates; new messages route to new partition; consumers rebalance.

Old messages stay in old partitions — no reshuffling needed.

### Decreasing Partitions

<img src="../../system-design-notes/19. Distributed Message Queue/images/partition-decrease.png" alt="Partition Decrease" width="450">

Harder than adding:
1. Mark partition as decommissioned; stop accepting new messages
2. Producers route to remaining active partitions
3. Consumers still read from decommissioned partition until its messages expire
4. After retention period passes, truncate data; partition is gone
5. Rebalance consumers

---

## Step 11: Data Delivery Semantics

The order in which you **fetch messages** and **commit offsets** determines your delivery guarantee.

### At-Most-Once — "Maybe"

<img src="../../system-design-notes/19. Distributed Message Queue/images/at-most-once.png" alt="At Most Once" width="450">

```
Consumer: fetch message → immediately commit offset → process message
```

If consumer crashes after committing but before processing → **message is lost forever** (offset moved past it).

Use when: metrics logging, analytics where occasional loss is acceptable. Never for payments.

### At-Least-Once — "Definitely, Maybe Twice"

<img src="../../system-design-notes/19. Distributed Message Queue/images/at-least-once.png" alt="At Least Once" width="450">

```
Consumer: fetch message → process message → commit offset
Producer: retry until ACK received
```

If consumer crashes after processing but before committing → message is reprocessed on restart.

**Result: every message processed at least once, possibly more.**

> **Requirement:** Consumer logic must be **idempotent** — processing the same message twice should have the same result as processing it once. (E.g., "set user status to SHIPPED" is idempotent. "Add $10 to balance" is not — unless you deduplicate by message ID.)

### Exactly-Once — "Exactly Once, Guaranteed"

<img src="../../system-design-notes/19. Distributed Message Queue/images/exactly-once.png" alt="Exactly Once" width="450">

Requires coordination between producer, broker, and consumer:
- **Idempotent producer**: broker deduplicates retries using `ProducerID + SequenceNumber`
- **Transactional writes**: producer atomically commits batches across multiple partitions
- **Consumer `read_committed` isolation**: only reads committed messages

**Result: every message processed exactly once — no loss, no duplicates.**

Most expensive to implement. Use for: financial transactions, inventory updates, anything where duplicates cause real-world problems.

---

## Step 12: Advanced Features

### Message Filtering

<img src="../../system-design-notes/19. Distributed Message Queue/images/message-filtering.png" alt="Message Filtering" width="450">

Some consumers only want certain types of messages within a partition.

Options:
- **Separate topics per type** — clean but leads to topic explosion
- **Tags on messages** — consumer subscribes to specific tags; broker filters server-side
- **Consumer-side filtering** — wastes bandwidth (consumer downloads all, discards most)

Tag-based filtering is the most practical: producers attach tags; consumers declare tag subscriptions; broker only sends matching messages.

### Delayed Messages

<img src="../../system-design-notes/19. Distributed Message Queue/images/delayed-message-implementation.png" alt="Delayed Message Implementation" width="450">

Use case: "Send a payment verification check 30 minutes from now."

Design:
1. Producer sends message with delivery timestamp to a **delay topic** (temporary storage)
2. A timer function scans for messages whose delivery time has arrived
3. Timer moves the message to the real topic partition
4. Consumer receives it at the scheduled time

Timer can be implemented with:
- **Dedicated delay queues** (one per delay bucket: 1m, 5m, 30m, 1h, 24h)
- **Hierarchical time wheel** — a circular buffer of time slots, similar to a clock

> **Note:** Kafka doesn''t natively support delayed messages — this is built on top of it.

---

## Summary: The "Why" Behind Each Decision

| Decision | Why? |
|----------|------|
| Pub/Sub model | Multiple independent services need the same events without coupling |
| Partitioning by key | Horizontal scaling + ordering guarantee within a partition |
| WAL (append-only file) | Sequential disk I/O is fast; no update/delete needed; OS caches hot segments |
| Batching in producer | Amortizes network cost; dramatically improves throughput |
| Pull model for consumers | Consumer controls pace; no risk of overwhelm; natural back-pressure |
| ISR + ACK modes | Configurable trade-off between write speed and durability |
| Consumer groups | Multiple services independently read the same data; isolation by offset |
| Consumer rebalancing | Automatic failover when consumers join/leave/crash; fault-tolerant |
| ZooKeeper for coordination | Metadata + state storage, leader election out of the box |
| Offset-based delivery semantics | Commit offset before/after processing controls exactly/at-least/at-most-once |

---

## Quick Interview Q&A Cheat Sheet

**Q: What''s the difference between a message queue and an event streaming platform?**  
A: Message queue (RabbitMQ, SQS): message deleted after consumption; one consumer gets it. Event streaming (Kafka): messages retained on disk; multiple independent consumer groups each read the full stream. Our design is Kafka-style with retention and repeated consumption.

**Q: What are the three delivery guarantees?**  
A: At-most-once: commit before processing → possible loss. At-least-once: process then commit → possible duplicates (most common). Exactly-once: idempotent producer + transactional commit → most expensive, no loss or duplicates.

**Q: How does Kafka achieve high throughput?**  
A: Sequential append to WAL (sequential disk I/O is fast), producer batching (fewer network round trips), zero-copy file transfer (`sendfile()` syscall), and partition parallelism across brokers.

**Q: How does message ordering work?**  
A: Order is guaranteed *within* a partition. Use a consistent partition key (e.g., `user_id`) so all messages for the same entity go to the same partition. Cross-partition ordering is not guaranteed. For global ordering, use a single partition — but that eliminates parallelism.

**Q: What is an ISR and why does it matter?**  
A: In-Sync Replica = a follower that is caught up with the leader. `ACK=all` waits for all ISRs to confirm a write — maximum durability. A lagging follower is removed from ISR so it doesn''t block writes. If a broker fails, a new leader is elected from ISRs → no data loss.

**Q: How do consumer groups enable fan-out?**  
A: Each consumer group maintains independent offsets. Adding a new consumer group (new service) doesn''t affect existing groups. One `order-placed` event is independently consumed by billing, inventory, and notifications — each at their own pace.

**Q: What happens during a consumer rebalance?**  
A: ALL consumers in the group pause processing. Coordinator elects a group leader → leader computes new partition assignments → broadcasts to all consumers → consumers resume from last committed offset. Triggered by join/leave/crash/timeout.

**Q: Push vs. pull for consumers?**  
A: Pull wins because consumers control their own rate. No risk of being overwhelmed. Natural back-pressure (slow consumer just falls behind). Broker can''t know processing capacity of each consumer, so push would require complex throttling logic.

**Q: How do delayed messages work?**  
A: Message with future timestamp goes to a delay topic (temporary storage). Timer service scans for due messages and moves them to the real topic. Consumer gets the message at scheduled time. Kafka doesn''t support this natively — built on top.

**Q: How do you scale a topic that receives too much traffic?**  
A: Increase the number of partitions → traffic is distributed across more brokers in parallel. Add more consumers (up to partition count) to match throughput. Each partition can handle ~10K–100K msg/sec depending on message size.
