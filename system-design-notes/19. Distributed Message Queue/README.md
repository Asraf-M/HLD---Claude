# Chapter 19: Distributed Message Queue

## Introduction

We'll be designing a **distributed message queue** in this chapter.

Benefits of message queues:
- **Decoupling**: Eliminates tight coupling between components. Let them update separately.
- **Improved scalability**: Producers and consumers can be scaled independently based on traffic.
- **Increased availability**: If one part of the system goes down, other parts continue interacting with the queue.
- **Better performance**: Producers can produce messages without waiting for consumer confirmation.

Some popular message queue implementations - Kafka, RabbitMQ, RocketMQ, Apache Pulsar, ActiveMQ, ZeroMQ.

Strictly speaking, Kafka and Pulsar are not message queues. They are event streaming platforms.
There is however a convergence of features which blurs the distinction between message queues and event streaming platforms.

In this chapter, we'll be building a message queue with support for more advanced features such as long data retention, repeated message consumption, etc.

---

## Step 1: Understand the Problem and Establish Design Scope

Message queues ought to support few basic features - producers produce messages and consumers consume them.
There are, however, different considerations with regards to performance, message delivery, data retention, etc.

Here's a set of potential questions between Candidate and Interviewer:
 * C: What's the format and average message size? Is it text only?
 * I: Messages are text-only and usually a few KBs
 * C: Can messages be repeatedly consumed?
 * I: Yes, messages can be repeatedly consumed by different consumers. This is an added requirement, which traditional message queues don't support.
 * C: Are messages consumed in the same order they were produced?
 * I: Yes, order guarantee should be preserved. This is an added requirement, traditional message queues don't support this.
 * C: What are the data retention requirements?
 * I: Messages need to have a retention of two weeks. This is an added requirement.
 * C: How many producers and consumers do we want to support?
 * I: The more, the better.
 * C: What data delivery semantic do we want to support? At-most-once, at-least-once, exactly-once?
 * I: We definitely want to support at-least-once. Ideally, we can support all and make them configurable.
 * C: What's the target throughput for end-to-end latency?
 * I: It should support high throughput for use cases like log aggregation and low throughput for more traditional use cases.

### **Functional requirements**

 * Producers send messages to a message queue
 * Consumers consume messages from the queue
 * Messages can be consumed once or repeatedly
 * Historical data can be truncated
 * Message size is in the KB range
 * Order of messages needs to be preserved
 * Data delivery semantics is configurable - at-most-once/at-least-once/exactly-once.

### **Non-functional requirements**

- **High throughput or low latency**: Configurable based on use-case
- **Scalable**: system should be distributed and support a sudden surge in message volume
- **Persistent and durable**: data should be persisted on disk and replicated among nodes

Traditional message queues typically don't support data retention and don't provide ordering guarantees. This greatly simplifies the design and we'll discuss it.

---

## Step 2: Propose High-Level Design and Get Buy-In

Key components of a message queue:

<div style="margin-left:3rem">
    <img src="./images/message-queue-components.png" alt="message-queue-components" width="500" />
</div>

 * Producer sends messages to a queue
 * Consumer subscribes to a queue and consumes the subscribed messages
 * Message queue is a service in the middle which decouples producers from consumers, letting them scale independently.
 * Producer and consumer are both clients, while the message queue is the server.

### **Messaging models**

The first type of messaging model is point-to-point and it's commonly found in traditional message queues:

<div style="margin-left:3rem">
    <img src="./images/point-to-point-model.png" alt="point-to-point-model" width="500" />
</div>

 * A message is sent to a queue and it's consumed by exactly one consumer.
 * There can be multiple consumers, but a message is consumed only once.
 * Once message is acknowledged as consumed, it is removed from the queue.
 * There is no data retention in the point-to-point model, but there is such in our design.

On the other hand, the publish-subscribe model is more common for event streaming platforms:

<div style="margin-left:3rem">
    <img src="./images/publish-subscribe-model.png" alt="publish-subscribe-model" width="500" />
</div>

 * In this model, messages are associated to a topic.
 * Consumers are subscribed to a topic and they receive all messages sent to this topic.

### **Topics, partitions and brokers**

What if the data volume for a topic is too large? One way to scale is by splitting a topic into partitions (aka sharding):

<div style="margin-left:3rem">
    <img src="./images/partitions.png" alt="partitions" width="500" />
</div>

 * Messages sent to a topic are evenly distributed across partitions
 * The servers that host partitions are called brokers
 * Each topic operates like a queue using FIFO for message processing. Message order is preserved within a partition.
 * The position of a message within the partition is called an **offset**.
 * Each message produced is sent to a specific partition. A partition key specifies which partition a message should land in. 
   * Eg a `user_id` can be used as a partition key to guarantee order of messages for the same user.
 * Each consumer subscribes to one or more partitions. When there are multiple consumers for the same messages, they form a consumer group.

### **Consumer groups**

Consumer groups are a set of consumers working together to consume messages from a topic:

<div style="margin-left:3rem">
    <img src="./images/consumer-groups.png" alt="consumer-groups" width="500" />
</div>

 * Messages are replicated per consumer group (not per consumer).
 * Each consumer group maintains its own offset.
 * Reading messages in parallel by a consumer group improves throughput but hampers the ordering guarantee.
 * This can be mitigated by only allowing one consumer from a group to be subscribed to a partition. 
 * This means that we can't have more consumers in a group than there are partitions.

### **High-level architecture**

<div style="margin-left:3rem">
    <img src="./images/high-level-architecture.png" alt="high-level-architecture" width="500" />
</div>

- **Clients**: producer and consumer. Producer pushes messages to a designated topic. Consumer group subscribes to messages from a topic.
- **Brokers**: hold multiple partitions. A partition holds a subset of messages for a topic.
- **Data storage**: stores messages in partitions.
- **State storage**: keeps the consumer states.
- **Metadata storage**: stores configuration and topic properties
- **Coordination service**: responsible for service discovery (which brokers are alive) and leader election (which broker is leader, responsible for assigning partitions).

---

## Step 3: Design Deep Dive

In order to achieve high throughput and preserve the high data retention requirement, we made some important design choices:
 * We chose an on-disk data structure which takes advantage of the properties of modern HDD and disk caching strategies of modern OS-es.
 * The message data structure is immutable to avoid extra copying, which we want to avoid in a high volume/high traffic system.
 * We designed our writes around batching as small I/O is an enemy of high throughput.

### **Data storage**

In order to find the best data store for messages, we must examine a message's properties:
 * Write-heavy, read-heavy
 * No update/delete operations. In traditional message queues, there is a "delete" operation as messages are not retained.
 * Predominantly sequential read/write access pattern.

What are our options:
- **Database**: not ideal as typical databases don't support well both write and read heavy systems.
- **Write-ahead log (WAL)**: a plain text file which only supports appending to it and is very HDD-friendly. 
  * We split partitions into segments to avoid maintaining a very large file.
  * Old segments are read-only. Writes are accepted by latest segment only.

<div style="margin-left:3rem">
    <img src="./images/wal-example.png" alt="wal-example" width="500" />
</div>

WAL files are extremely efficient when used with traditional HDDs. 

There is a misconception that HDD acces is slow, but that hugely depends on the access pattern.
When the access pattern is sequential (as in our case), HDDs can achieve several MB/s write/read speed which is sufficient for our needs.
We also piggyback on the fact that the OS caches disk data in memory aggressively.

### **Message data structure**

It is important that the message schema is compliant between producer, queue and consumer to avoid extra copying. This allows much more efficient processing.

Example message structure:

<div style="margin-left:3rem">
    <img src="./images/message-structure.png" alt="message-structure" width="500" />
</div>

The key of the message specifies which partition a message belongs to. An example mapping is `hash(key) % numPartitions`.
For more flexibility, the producer can override default keys in order to control which partitions messages are distributed to.

The message value is the payload of a message. It can be plaintext or a compressed binary block.

**Note:** Message keys, unlike traditional KV stores, need not be unique. It is acceptable to have duplicate keys and for it to even be missing.

Other message files:
- **Topic**: topic the message belongs to
- **Partition**: The ID of the partition a message belongs to
- **Offset**: The position of the message in a partition. A message can be located via `topic`, `partition`, `offset`.
- **Timestamp**: When the message is stored
- **Size**: the size of this message
- **CRC**: checksum to ensure message integrity

Additional features such as filtering can be supported by adding additional fields.

### **Batching**

Batching is critical for the performance of our system. We apply it in the producer, consumer and message queue.

It is critical because:
 * It allows the operating system to group messages together, amortizing the cost of expensive network round trips
 * Messages are written to the WAL in groups sequentially, which leads to a lot of sequential writes and disk caching.

There is a trade-off between latency and throughput:
 * High batching leads to high throughput and higher latency. 
 * Less batching leads to lower throughput and lower latency.

If we need to support lower latency since the system is deployed as a traditional message queue, the system could be tuned to use a smaller batch size.

If tuned for throughput, we might need more partitions per topic to compensate for the slower sequential disk write throughput.

### **Producer flow**

If a producer wants to send a message to a partition, which broker should it connect to?

One option is to introduce a routing layer, which route messages to the correct broker. If replication is enabled, the correct broker is the leader replica:

<div style="margin-left:3rem">
    <img src="./images/routing-layer.png" alt="routing-layer" width="500" />
</div>

 * Routing layer reads the replication plan from the metadata store and caches it locally.
 * Producer sends a message to the routing layer.
 * Message is forwarded to broker 1 who is the leader of the given partition
 * Follower replicas pull the new message from the leader. Once enough confirmations are received, the leader commits the data and responds to the producer.

The reason for having replicas is to enable fault tolerance.

This approach works but has some drawbacks:
 * Additional network hops due to the extra component
 * The design doesn't enable batching messages

To mitigate these issues, we can embed the routing layer into the producer:

<div style="margin-left:3rem">
    <img src="./images/routing-layer-producer.png" alt="routing-layer-producer" width="500" />
</div>

 * Fewer network hops lead to lower latency
 * Producers can control which partition a message is routed to
 * The buffer allows us to batch messages in-memory and send out larger batches in a single request, which increases throughput.

The batch size choice is a classical trade-off between throughput and latency. 

<div style="margin-left:3rem">
    <img src="./images/batch-size-throughput-vs-latency.png" alt="batch-size-throughput-vs-latency" width="500" />
</div>

 * Larger batch size leads to longer wait time before batch is committed. 
 * Smaller batch size leads to request being sent sooner and having lower latency but lower throughput.

### **Consumer flow**

The consumer specifies its offset in a partition and receives a chunk of messages, beginning from that offset:

<div style="margin-left:3rem">
    <img src="./images/consumer-example.png" alt="consumer-example" width="500" />
</div>

One important consideration when designing the consumer is whether to use a push or a pull model:
- **Push model**: leads to lower latency as broker pushes messages to consumer as it receives them.
  * However, if rate of consumption falls behind the rate of production, the consumer can be overwhelmed.
  * It is challenging to deal with consumers with varying processing power as the broker controls the rate of consumption.
- **Pull model**: leads to the consumer controlling the consumption rate. 
  * If rate of consumption is slow, consumer will not be overwhelmed and we can scale it to catch up.
  * The pull model is more suitable for batch processing, because with the push model, the broker can't know how many messages a consumer can handle. 
  * With the pull model, on the other hand, consumers can aggressively fetch large message batches.
  * The down side is the higher latency and extra network calls when there are no new messages. Latter issue can be mitigated using long polling.

Hence, most message queues (and us) choose the pull model.

<div style="margin-left:3rem">
    <img src="./images/consumer-flow.png" alt="consumer-flow" width="500" />
</div>

 * A new consumer subscribes to topic A and joins group 1.
 * The correct broker node is found by hashing the group name. This way, all consumers in a group connect to the same broker.
 * Note that this consumer group coordinator is different from the coordination service (ZooKeeper).
 * Coordinator confirms that the consumer has joined the group and assigns partition 2 to that consumer.
 * There are different partition assignment strategies - round-robin, range, etc.
 * Consumer fetches latest messages from the last offset. The state storage keeps the consumer offsets.
 * Consumer processes messages and commits the offset to the broker. The order of those operations affects the message delivery semantics.

### **Consumer rebalancing**

Consumer rebalancing is responsible for deciding which consumers are responsible for which partition.

This process occurs when a consumer joins/leaves or a partition is added/removed.

The broker, acting as a coordinator plays a huge role in orchestrating the rebalancing workflow.

<div style="margin-left:3rem">
    <img src="./images/consumer-rebalancing.png" alt="consumer-rebalancing" width="500" />
</div>

 * All consumers from the same group are connected to the same coordinator. The coordinator is found by hashing the group name.
 * When the consumer list changes, the coordinator chooses a new leader of the group.
 * The leader of the group calculates a new partition dispatch plan and reports it back to the coordinator, which broadcasts it to the other consumers.

When the coordinator stops receiving heartbeats from the consumers in a group, a rebalancing is triggered:

<div style="margin-left:3rem">
    <img src="./images/consumer-rebalance-example.png" alt="consumer-rebalance-example" width="500" />
</div>

Let's explore what happens when a consumer joins a group:

<div style="margin-left:3rem">
    <img src="./images/consumer-join-group-usecase.png" alt="consumer-join-group-usecase" width="500" />
</div>

 * Initially, only consumer A is in the group and it consumes all partitions.
 * Consumer B sends a request to join the group.
 * The coordinator notifies all group members that it's time to rebalance passively - as a response to the heartbeat.
 * Once all consumers rejoin the group, the coordinator chooses a leader and notifies the rest about the election result.
 * The leader generates the partition dispatch plan and sends it to the coordinator. Others wait for the dispatch plan.
 * Consumers start consuming from the newly assigned partitions.

Here's what happens when a consumer leaves the group:

<div style="margin-left:3rem">
    <img src="./images/consumer-leaves-group-usecase.png" alt="consumer-leaves-group-usecase" width="500" />
</div>

 * Consumer A and B are in the same group
 * Consumer B asks to leave the group
 * When coordinator receives A's heartbeat, it informs them that it's time to rebalance.
 * The rest of the steps are the same.

The process is similar when a consumer doesn't send a heartbeat for a long time:

<div style="margin-left:3rem">
    <img src="./images/consumer-no-heartbeat-usecase.png" alt="consumer-no-heartbeat-usecase" width="500" />
</div>

### **State storage**

The state storage stores mapping between partitions and consumers, as well as the last consumed offsets for a partition.

<div style="margin-left:3rem">
    <img src="./images/state-storage.png" alt="state-storage" width="500" />
</div>

Group 1's offset is at 6, meaning all previous messages are consumed. If a consumer crashes, the new consumer will continue from that message on wards.
 
Data access patterns for consumer states:
 * Frequent read/write operations, but low volume
 * Data is updated frequently, but rarely deleted
 * Random read/write
 * Data consistency is important

Given these requirements, a fast KV storage like Zookeeper is ideal.

### **Metadata storage**

The metadata storage stores configuration and topic properties - partition number, retention period, replica distribution.

Metadata doesn't change often and volume is small, but there is a high consistency requirement.
Zookeeper is a good choice for this storage.

### **ZooKeeper**

Zookeeper is essential for building distributed message queues.

It is a hierarchical key-value store, commonly used for a distributed configuration, synchronization service and naming registry (ie service discovery).

<div style="margin-left:3rem">
    <img src="./images/zookeeper.png" alt="zookeeper" width="500" />
</div>

With this change, the broker only needs to maintain data for the messages. Metadata and state storage is in Zookeeper.

Zookeeper also helps with leader election of the broker replicas.

### **Replication**

In distributed systems, hardware issues are inevitable. We can tackle this via replication to achieve high availability.

<div style="margin-left:3rem">
    <img src="./images/replication-example.png" alt="replication-example" width="500" />
</div>

 * Each partition is replicated across multiple brokers, but there is only one leader replica.
 * Producers send messages to leader replicas
 * Followers pull the replicated messages from the leader
 * Once enough replicas are synchronized, the leader returns acknowledgment to the producer
 * Distribution of replicas for each partition is called the replica distribution plan.
 * The leader for a given partition creates the replica distribution plan and saves it in Zookeeper

### **In-sync replicas**

One problem we need to tackle is keeping messages in-sync between the leader and the followers for a given partition.

In-sync replicas (ISR) are replicas for a partition that stay in-sync with the leader.

The `replica.lag.max.messages` defines how many messages can a replica be lagging behind the leader to be considered in-sync.

<div style="margin-left:3rem">
    <img src="./images/in-sync-replicas-example.png" alt="in-sync-replicas-example" width="500" />
</div>

 * Committed offset is 13
 * Two new messages are written to the leader, but not committed yet.
 * A message is committed once all replicas in the ISR have synchronized that message
 * Replica 2 and 3 have fully caught up with leader, hence, they are in ISR
 * Replica 4 has lagged behind, hence, is removed from ISR for now

ISR reflects a trade-off between performance and durability.
 * In order for producers not to lose messages, all replicas should be in sync before sending an acknowledgment
 * But a slow replica will cause the whole partition to become unavailable

Acknowledgment handling is configurable.

`ACK=all` means that all replicas in ISR have to sync a message. Message sending is slow, but message durability is highest.

<div style="margin-left:3rem">
    <img src="./images/ack-all.png" alt="ack-all" width="500" />
</div>

`ACK=1` means that producer receives acknowledgment once leader receives the message. Message sending is fast, but message durability is low.

<div style="margin-left:3rem">
    <img src="./images/ack-1.png" alt="ack-1" width="500" />
</div>

`ACK=0` means that producer sends messages without waiting for any acknowledgment from leader. Message sending is fastest, message durability is lowest.

<div style="margin-left:3rem">
    <img src="./images/ack-0.png" alt="ack-0" width="500" />
</div>

On the consumer side, we can connect all consumers to the leader for a partition and let them read messages from it:
 * This makes for the simplest design and easiest operation
 * Messages in a partition are sent to only one consumer in a group, which limits the connections to the leader replica
 * The number of connections to leader replica is typically not high as long as the topic is not super hot
 * We can scale a hot topic by increasing the number of partitions and consumers
 * In certain scenarios, it might make sense to let a consumer lead from an ISR, eg if they're located in a separate DC

The ISR list is maintained by the leader who tracks the lag between itself and each replica.

### **Scalability**

Let's evaluate how we can scale different parts of the system.

#### Producer

The producer is much smaller than the consumer. Its scalability can easily be achieved by adding/removing new producer instances.

#### Consumer

Consumer groups are isolated from each other. It is easy to add/remove consumer groups at will.

Rebalancing help handle the case when consumers are added/removed from a group gracefully.

Consumer groups are rebalancing help us achieve scalability and fault tolerance.

#### Broker

How do brokers handle failure?

<div style="margin-left:3rem">
    <img src="./images/broker-failure-recovery.png" alt="broker-failure-recovery" width="500" />
</div>

 * Once a broker fails, there are still enough replicas to avoid partition data loss
 * A new leader is elected and the broker coordinator redistributes partitions which were at the failed broker to existing replicas
 * Existing replicas pick up the new partitions and act as followers until they're caught up with the leader and become ISR

Additional considerations to make the broker fault-tolerant:
 * The minimum number of ISRs balances latency and safety. You can fine-tune it to meet your needs.
 * If all replicas of a partition are in the same node, then it's a waste of resources. Replicas should be across different brokers.
 * If all replicas of a partition crash, then the data is lost forever. Spreading replicas across data centers can help, but it adds up a lot of latency. One option is to use [data mirroring](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=27846330) as a work around.

How do we handle redistribution of replicas when a new broker is added?

<div style="margin-left:3rem">
    <img src="./images/broker-replica-redistribution.png" alt="broker-replica-redistribution" width="500" />
</div>

 * We can temporarily allow more replicas than configured, until new broker catches up
 * Once it does, we can remove the partition replica which is no longer needed

#### Partition

Whenever a new partition is added, the producer is notified and consumer rebalancing is triggered.

In terms of data storage, we can only store new messages to the new partition vs. trying to copy all old ones:

<div style="margin-left:3rem">
    <img src="./images/partition-exmaple.png" alt="partition-example" width="500" />
</div>

Decreasing the number of partitions is more involved:

<div style="margin-left:3rem">
    <img src="./images/partition-decrease.png" alt="partition-decrease" width="500" />
</div>

 * Once a partition is decommissioned, new messages are only received by remaining partitions
 * The decommissioned partition isn't removed immediately as messages can still be consumed from it
 * Once a pre-configured retention period passes, do we truncate the data and storage space is freed up
 * During the transitional period, producers only send messages to active partitions, but consumers read from all
 * Once retention period expires, consumers are rebalanced

### **Data delivery semantics**

Let's discuss different delivery semantics.

#### At-most once

With this guarantee, messages are delivered not more than once and could not be delivered at all.

<div style="margin-left:3rem">
    <img src="./images/at-most-once.png" alt="at-most-once" width="500" />
</div>

 * Producer sends a message asynchronously to a topic. If message delivery fails, there is no retry.
 * Consumer fetches message and immediately commits offset. If consumer crashes before processing the message, the message will not be processed.

#### At-least once

A message can be sent more than once and no message should be left unprocessed.

<div style="margin-left:3rem">
    <img src="./images/at-least-once.png" alt="at-least-once" width="500" />
</div>

 * Producer sends message with `ack=1` or `ack=all`. If there is any issue, it will keep retrying.
 * Consumer fetches the message and consumes the offset only after it's done processing it.
 * It is possible for a message to be delivered more than once if eg consumer crashes before committing offset but after processing it.
 * This is why, this is good for use-cases where data duplication is acceptable or deduplication is possible.

#### Exactly once

Extremely costly to implement for the system, albeit it's the friendliest guarantee to users:

<div style="margin-left:3rem">
    <img src="./images/exactly-once.png" alt="exactly-once" width="500" />
</div>

### **Advanced features**

Let's discuss some advanced features, we might discuss in the interview.

#### Message filtering

Some consumers might want to only consume messages of a certain type within a partition.

This can be achieved by building separate topics for each subset of messages, but this can be costly if systems have too many differing use-cases.
 * It is a waste of resources to store the same message on different topics
 * Producer is now tightly coupled to consumers as it changes with each new consumer requirement

We can resolve this using message filtering.
 * A naive approach would be to do the filtering on the consumer-side, but that introduces unnecessary consumer traffic
 * Alternatively, messages can have tags attached to them and consumers can specify which tags they're subscribed to
 * Filtering could also be done via the message payloads but that can be challenging and unsafe for encrypted/serialized messages
 * For more complex mathematical formulaes, the broker could implement a grammar parser or script executor, but that can be heavyweight for the message queue

<div style="margin-left:3rem">
    <img src="./images/message-filtering.png" alt="message-filtering" width="500" />
</div>

#### Delayed messages & scheduled messages

For some use-cases, we might want to delay or schedule message delivery. 
For example, we might submit a payment verification check for 30m from now, which triggers the consumer to see if a payment was successful.

This can be achieved by sending messages to temporary storage in the broker and moving the message to the partition at the right time:

<div style="margin-left:3rem">
    <img src="./images/delayed-message-implementation.png" alt="delayed-message-implementation" width="500" />
</div>

 * The temporary storage can be one or more special message topics
 * The timing function can be achieved using dedicated delay queues or a [hierarchical time wheel](http://www.cs.columbia.edu/~nahum/w6998/papers/sosp87-timing-wheels.pdf)

---

## Step 4: Wrap Up

Additional talking points:
- **Protocol of communication**: Important considerations - support all use-cases and high data volume, as well as verify message integrity. Popular protocols - AMQP and Kafka protocol.
- **Retry consumption**: if we can't process a message immediately, we could send it to a dedicated retry topic to be attempted later.
- **Historical data archive**: old messages can be backed up in high-capacity storages such as HDFS or object storage (eg S3).

---

## Most Asked Interview Questions

**Q1. What is the difference between a message queue and an event streaming platform?**
> Message queue (RabbitMQ, SQS): messages are consumed once and deleted; focus is on task distribution and work queues; consumers compete for messages. Event streaming (Kafka): messages are retained on disk for a configurable period; multiple independent consumer groups can each read the full stream; focus is on event log and replay. Kafka is preferred for audit trails, event sourcing, and fan-out to many consumers.

**Q2. What delivery guarantees can a message queue provide?**
> At-most-once: fire-and-forget; message may be lost if broker crashes before ack — acceptable for non-critical metrics. At-least-once: message is re-sent if ack not received; consumer may receive duplicates — most common; consumers must be idempotent. Exactly-once: producer uses idempotent writes + transactions; consumer commits offset transactionally with business logic — highest complexity, used for payments/financial events. Kafka supports all three via producer acks + idempotent producers + transactions.

**Q3. How does Kafka achieve high throughput and durability?**
> Sequential disk I/O: Kafka appends to partition log files sequentially — sequential writes on HDD reach ~600 MB/s vs 100 MB/s random. Zero-copy: `sendfile()` syscall copies data from disk to NIC without copying to user space. Batching: producer batches multiple records → one network round trip. Replication: `min.insync.replicas=2` + `acks=all` ensures data is on 2 brokers before ack → survives single broker failure. Partition parallelism enables linear horizontal scaling.

**Q4. What is a consumer group and how does it enable parallel processing?**
> A consumer group is a set of consumers that collectively consume all partitions of a topic. Each partition is assigned to exactly one consumer in the group. Parallelism = number of partitions (adding more consumers than partitions yields no benefit). Different consumer groups independently read the same topic from their own offsets — a billing service and an analytics service can both consume the same events without interfering. Groups coordinate via Kafka's Group Coordinator (embedded in a broker).

**Q5. How do you ensure message ordering in a distributed message queue?**
> Kafka guarantees ordering within a partition. Use a consistent partitioning key (e.g., user_id or order_id) so all messages for the same entity go to the same partition → preserved order. Cross-partition ordering is not guaranteed. If global ordering is required: use a single partition (limits parallelism to 1 consumer). Alternatively, use sequence numbers in messages and sort at the consumer — but this adds complexity and latency.

**Q6. How does message retention work in Kafka vs. traditional queues?**
> Traditional queue (RabbitMQ/SQS): message is deleted once consumed — "destructive read". Kafka: messages are written to a durable partition log and retained regardless of consumption (configurable: by time `retention.ms=604800000` = 7 days, or by size `retention.bytes`). Retention enables consumer replays (re-process past events), multiple independent consumers, and historical audit. Old segments are deleted by a background compaction/cleanup process.

**Q7. How does Kafka partitioning affect parallelism and performance?**
> Topics are split into N partitions. Producers write to partitions in parallel (round-robin or keyed hash). Consumers in a group each own 1+ partitions and process in parallel. More partitions → more parallelism but: more open file handles on brokers, bigger consumer group rebalance time, more ZooKeeper/KRaft metadata overhead. Rule of thumb: target 1 partition per consumer needed at peak throughput, with headroom. A typical production topic has 10–100 partitions.

**Q8. How do you handle messages that consistently fail to process (Dead Letter Queue)?**
> Consumer catches processing exception → after N retries (e.g., 3), moves the message to a dedicated Dead Letter Topic (DLT) alongside error metadata (exception, timestamp, original topic/offset). DLT is monitored by alerting. Operations team investigates the DLT messages — fix the bug, replay the DLT after fix. DLT prevents a poison pill message from blocking an entire partition indefinitely. DLT processing logic mirrors the original consumer but with relaxed retry policy.

**Q9. What is the difference between push and pull consumption models?**
> Push: broker pushes messages to consumers as they arrive — low latency, but consumer can be overwhelmed if slow. RabbitMQ uses push with prefetch count to limit in-flight messages. Pull: consumer polls the broker for new messages — consumer controls pace, eliminates overwhelm; slight latency penalty (poll interval). Kafka uses pull: consumers call `poll()` which returns up to `max.poll.records` messages. Pull also makes offset management consumer-controlled.

**Q10. How do you ensure messages are not lost during a broker failure?**
> Producer: `acks=all` — leader waits for ISR (in-sync replicas) to acknowledge write before responding to producer. Replication factor ≥ 2 (`replication.factor=3` for critical topics). `min.insync.replicas=2` — if only 1 replica is alive, writes are rejected (prefer unavailability over data loss). Consumer: commit offsets only after successful processing (at-least-once). With these settings, a single broker failure loses no committed messages.

**Q11. What is the In-Sync Replica (ISR) mechanism in Kafka?**
> ISR is the set of replicas that are fully caught up with the partition leader (within `replica.lag.time.max.ms` = 10 seconds by default). Leader sends write to all ISR members. Follower that falls behind is removed from ISR. `acks=all` means the ack is sent after all ISR members confirm the write. If an ISR member crashes, it's removed from ISR and the leader continues with remaining ISR. When the crashed broker recovers, it re-joins ISR after catching up.

**Q12. What is log compaction in Kafka and when do you use it?**
> Log compaction retains only the most recent value per message key — older entries for the same key are garbage-collected. Use case: materializing the latest state (e.g., user preferences — only need the last value, not the full history). Normal topics use time/size-based retention (keep last 7 days of events). Compacted topics are like a changelog of a database table. Enabled with `cleanup.policy=compact`. Compaction runs asynchronously in the background.

**Q13. How does Kafka handle back-pressure when consumers are slow?**
> Pull model inherently applies back-pressure: consumer only fetches as fast as it can process. If consumer group falls behind, the lag metric (`consumer_lag`) increases. Operational responses: (1) Scale out consumers (add partitions + consumers); (2) Fix slow processing logic; (3) Increase retention to prevent data loss while lag is high. No explicit back-pressure protocol needed — the gap between committed offset and latest offset IS the back-pressure signal.

**Q14. What is Kafka KRaft mode and why is it replacing ZooKeeper?**
> Legacy Kafka used Apache ZooKeeper to store cluster metadata (topic configs, partition leaders, consumer group offsets). ZooKeeper is a separate cluster to operate, limits cluster size (~200K partitions), and adds operational complexity. KRaft (Kafka Raft Metadata mode, KIP-500): metadata is stored in a dedicated Kafka partition using Raft consensus. Benefits: simpler operations (no ZooKeeper), supports millions of partitions, faster controller failover (<1s), fully integrated into Kafka.

**Q15. How do you implement exactly-once semantics in Kafka?**
> Producer side: enable idempotent producer (`enable.idempotence=true`) — each message has a ProducerID + sequence number; broker deduplicates retries. Transactional producer: `producer.initTransactions()` → `beginTransaction()` → produce messages → `commitTransaction()` — atomically publishes a batch across multiple partitions. Consumer side: use `read_committed` isolation level to not see uncommitted messages. Full exactly-once: wrap consumer+produce logic in a transaction (consume-transform-produce pattern).

**Q16. How do you design a topic and partition schema for an order processing system?**
> Topic: `orders` — all order events (created, paid, shipped, cancelled). Partition key: `order_id` — all events for the same order go to the same partition → in-order processing per order. Number of partitions: estimated peak write rate / per-partition throughput (target ~10K msg/sec per partition) = e.g., 20 partitions for 200K msg/sec peak. Separate topics for different event types vs one topic with type field — tradeoff: separate topics enable per-topic retention but more topic management.

**Q17. How would you monitor the health of a Kafka cluster?**
> Key metrics: (1) Consumer lag (max offset - committed offset) per group — lag >10K messages = alert; (2) Under-replicated partitions (URP) — URP>0 = broker health issue; (3) Request latency (99th percentile); (4) Disk utilization per broker; (5) Network throughput (bytes in/out per sec); (6) Leader election rate — high rate = broker instability. Tools: Kafka JMX metrics → Prometheus → Grafana. External: Burrow (LinkedIn's consumer lag monitor), Cruise Control for auto-rebalancing.

**Q18. What is the role of consumer offsets, and where are they stored?**
> An offset is the position of the last processed message in a partition. The consumer commits its offset after processing. Kafka stores committed offsets in the internal `__consumer_offsets` topic (compacted, replicated). On restart or rebalance, consumers resume from their last committed offset. Manual offset management: commit only after successfully processing (at-least-once). Auto-commit (`enable.auto.commit=true`, default 5s) can lead to data loss if app crashes between auto-commit and processing.

**Q19. How do you handle a schema change in messages (schema evolution)?**
> Use a schema registry (Confluent Schema Registry): producers register Avro/Protobuf/JSON Schema before publishing; consumers validate against registered schema. Schema evolution rules: backward compatible (old consumers can read new messages — add optional fields, never remove required fields). Schema ID embedded in message header. When schema changes: publish new schema version → update producers → update consumers (rolling deploy) → old versions still work due to backward compatibility.

**Q20. What is the difference between Kafka and AWS SQS/SNS?**
> Kafka: self-hosted (or Confluent Cloud), stateful consumer offsets, retained messages, partitions for ordering, designed for high-throughput streaming (millions msg/sec). SQS (queue): managed AWS service, message deleted after consume, no replay, FIFO queue available, max retention 14 days, max 3K msg/sec standard. SNS (pub/sub fan-out): push to subscribers (Lambda, SQS, HTTP). SQS/SNS: fully managed (simpler ops). Kafka: more power and flexibility. Use SQS for simple task queues; Kafka for event streaming at scale.

**Q21. How do you handle message replay in Kafka (reprocessing past events)?**
> Replay: reset consumer group offset to an earlier position via `kafka-consumer-groups.sh --reset-offsets --to-datetime 2024-01-01T00:00:00Z`. All messages from that timestamp forward are reprocessed. Use cases: fixing a bug in consumer logic, backfilling a new data store, testing. Requires: (1) messages still in retention window; (2) consumer logic to be idempotent (re-processing same message doesn't cause double-charge). Kafka's durable log is what enables this — traditional queues can't do replay.

**Q22. How does a message queue support fan-out (one message → many consumers)?**
> Kafka: multiple independent consumer groups each subscribe to the same topic and receive all messages. Adding a new consumer group doesn't affect existing groups. Example: an `order-placed` event is consumed by the inventory service, billing service, notification service, and analytics service — each has its own consumer group and offset. No configuration change needed on the producer. This is Kafka's publish-subscribe model.

**Q23. How do you ensure fair load distribution across brokers in a Kafka cluster?**
> Partition leader distribution: ideally each broker is leader for an equal share of partitions. Kafka auto-assigns leaders but can become unbalanced after broker failures. Tools: `kafka-preferred-replica-election` (reassigns leadership to preferred replicas), Cruise Control (auto-rebalances partition replicas and leaders across brokers). Monitor broker disk/network utilization — if one broker is hot, move some partition leaders to less-loaded brokers.

**Q24. What are the trade-offs between larger vs. smaller message batches in Kafka?**
> Larger batches: higher throughput (more messages per network round trip, better compression), higher latency (messages wait in batch buffer). Smaller batches: lower latency, more overhead per message. Configuration: `linger.ms` (how long producer waits to fill a batch, default 0ms = no wait), `batch.size` (max batch size in bytes, default 16KB). For log ingestion (throughput-sensitive): `linger.ms=5–10ms`, `batch.size=64KB`. For payment events (latency-sensitive): `linger.ms=0`, small batches.

**Q25. What happens during a Kafka consumer group rebalance?**
> Rebalance is triggered when: a consumer joins/leaves the group, or a consumer fails to heartbeat within `session.timeout.ms` (default 10s). During rebalance: ALL consumers in the group pause processing (stop-the-world). Group Coordinator reassigns partitions to remaining consumers. After rebalance, consumers resume from last committed offset. Minimize rebalances: tune `max.poll.interval.ms` (processing SLA), use static group membership (consumer IDs persist across restarts to avoid unnecessary rebalances), use cooperative rebalancing protocol (incremental instead of stop-the-world).

**Q26. How would you design the message queue to support delayed messages?**
> Use case: "send email 24 hours after order placed". Approach: (1) Store the message in a delay store (Redis sorted set where score = delivery_timestamp, DynamoDB TTL); (2) A timer service scans for messages due now and publishes them to the real Kafka topic; (3) Consumers receive the message on schedule. Alternatively: RabbitMQ supports native message TTL + Dead Letter Exchange for delay routing. SQS supports delay queues (up to 15 minutes). Kafka has no native delay support.

**Q27. How do you estimate the infrastructure needs for a message queue handling 1 million messages per second?**
> At 1M msg/sec with average 1KB message size: 1 GB/s network bandwidth on the broker. Disk throughput: 1 GB/s write (sequential → NVMe SSD handles 3–5 GB/s). With replication factor 3: 3 GB/s write across cluster. Number of brokers: each broker handles ~200 MB/s safely → 15 brokers for 3GB/s. Partitions: 100 partitions × 10K msg/sec/partition = 1M msg/sec. Consumer groups: each group needs throughput matching 1M msg/sec → scale consumers = # partitions = 100 consumers per group.
