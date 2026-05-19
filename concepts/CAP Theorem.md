# CAP Theorem

## What Is It?

The **CAP Theorem** (also called Brewer's Theorem) states:

> A distributed system can guarantee **at most 2 of these 3 properties** at any given time:
> - **C**onsistency
> - **A**vailability  
> - **P**artition Tolerance

---

## The Three Properties

### C — Consistency
Every read receives the **most recent write** (or an error).

> **Analogy:** You transfer $100 from account A to account B. Any person who checks their balance afterward sees the updated amount — not the old one.

In technical terms: all nodes see the same data at the same time. After a write completes, all subsequent reads return that value.

### A — Availability
Every request receives a **response** (not an error), but it may not be the most recent data.

> **Analogy:** The bank always answers your balance inquiry — even if it might show a slightly stale number.

In technical terms: every non-failing node returns a valid response within a reasonable time. No request hangs or errors out.

### P — Partition Tolerance
The system continues operating even when **some nodes can't talk to each other** (network partition).

> **Analogy:** The bank's New York and London branches lose their connection. Both keep serving customers rather than shutting down.

In technical terms: the system keeps functioning even if arbitrary messages are dropped or delayed between nodes.

---

## Why You Can't Have All Three

Imagine two database nodes (Node A and Node B) with a network partition between them — they can't communicate.

A write comes in to Node A: "Set X = 2"

Now a read comes in to Node B: "Get X"

You must choose:
- **Be Consistent (CP):** Node B refuses to answer (or returns an error) until it can sync with Node A → sacrifices **Availability**
- **Be Available (AP):** Node B answers with its stale value X = 1 → sacrifices **Consistency**

There is no third option. **Partition tolerance is not optional** in any real distributed system — network failures happen. So the real choice is always: **CP or AP**.

---

## The Real Choice: CP vs AP

```
                 P (Partition Tolerance)
                 ↑
        CP ──────┼────── AP
                 │
```

Since P is mandatory in practice, you choose between:

### CP Systems (Consistent + Partition Tolerant)
Prioritize correctness. During a partition, reject requests rather than serve stale data.

**Examples:** HBase, Zookeeper, etcd, Google Spanner, MongoDB (default config)

**Use when:** Financial transactions, inventory management, reservation systems — where wrong data is worse than no data.

### AP Systems (Available + Partition Tolerant)
Prioritize uptime. During a partition, serve possibly stale data rather than error out.

**Examples:** Cassandra, CouchDB, DynamoDB (default config), Amazon S3

**Use when:** Social media likes, shopping carts, DNS, CDN caches — where stale data is acceptable for a short time.

---

## Visualizing CAP

```
              Consistency
                  /\
                 /  \
                /    \
               / RDBMS \
              /  MySQL  \
             /____________\
            /      |       \
           /  HBase |  DNS  \
          / Zookpr  | Dynamo \
         /  MongoDB | Cassnd. \
        /___________x__________\
    Availability         Partition
                         Tolerance
```

> Note: Traditional single-node RDBMS ignores P (no network partition if it's one machine) and gives you C + A.

---

## PACELC — The Extension to CAP

CAP only describes behavior during a partition. **PACELC** asks: even when there's **no partition**, what do you trade off?

```
If Partition → trade off A vs C
Else (normal operation) → trade off L(atency) vs C(onsistency)
```

| System | Partition behavior | Normal behavior |
|--------|-------------------|-----------------|
| Cassandra | AP | EL (high availability, low latency, eventual consistency) |
| HBase | CP | PC (consistent, higher latency) |
| MySQL | CP | PC |
| DynamoDB | AP (configurable) | EL (default) |

---

## Consistency Levels (Tunable)

Modern NoSQL systems (Cassandra, DynamoDB) let you **tune** consistency per operation:

### Cassandra Consistency Levels

| Level | Meaning | Use Case |
|-------|---------|---------|
| `ONE` | 1 replica responds | Highest availability, lowest consistency |
| `QUORUM` | Majority of replicas respond | Balance of both (most common) |
| `ALL` | All replicas respond | Highest consistency, lowest availability |
| `LOCAL_QUORUM` | Quorum within local datacenter | Multi-DC with local preference |

**Quorum rule:** With N replicas, quorum = ⌊N/2⌋ + 1

For N=3: quorum = 2. If write quorum = 2 AND read quorum = 2, then read always sees the latest write (because at least 1 node overlaps).

---

## Real-World Decision Examples

| System | Choice | Why |
|--------|--------|-----|
| Bank transfers | **CP** | Wrong balance = fraud. Unavailability is better than wrong data. |
| Hotel booking | **CP** | Double-booking is worse than "try again later" |
| Social media likes | **AP** | Showing 999 vs 1000 likes briefly is fine |
| Shopping cart | **AP** | Better to show slightly stale cart than error out |
| DNS | **AP** | Serves stale IP → eventually propagates correct one |
| Distributed lock | **CP** | Must be correct or two processes think they hold the lock |

---

## Strong vs Eventual Consistency

### Strong Consistency
After a write completes, all reads immediately return the new value.

```
Time → 
Write X=2 ─────────────────────
                ↓
All reads after: X=2 (immediately)
```

**Cost:** Higher latency (must coordinate across replicas before returning)

### Eventual Consistency
After a write, replicas converge over time. Reads may temporarily return old value.

```
Time →
Write X=2 ──────────────────────
Node A: X=2 immediately
Node B: X=1 → X=2 (after sync)
Node C: X=1 → X=1 → X=2 (after sync)
```

**Cost:** Readers may see stale data for milliseconds to seconds

---

## Interview Q&A

**Q: What does the CAP theorem say?**  
A: A distributed system can guarantee at most 2 of: Consistency (every read gets latest write), Availability (every request gets a response), and Partition Tolerance (works despite network failures). Since network partitions always happen in real systems, you actually choose between CP (sacrifice availability during partitions) or AP (sacrifice consistency during partitions).

**Q: Is partition tolerance optional?**  
A: No. In any distributed system with multiple machines connected by a network, network partitions can and do occur. You can't eliminate partitions — you can only decide how to react to them. So the real choice is always CP vs AP.

**Q: Why would you choose an AP system?**  
A: When availability matters more than perfect accuracy. Examples: social media counters (likes, views), shopping carts, DNS resolution, CDN caches. A brief period of stale data is acceptable and returning an error would be worse for user experience.

**Q: Why would you choose a CP system?**  
A: When correctness is non-negotiable. Examples: bank transfers, inventory (can't sell the same last item twice), distributed locks, reservation systems. Wrong data causes real harm; brief unavailability is preferable.

**Q: What is eventual consistency?**  
A: After a write, different replicas may temporarily return different values. Over time (milliseconds to seconds), all replicas converge to the same value. Cassandra, DynamoDB (default), and DNS use this model. The system remains available during partitions but may serve slightly stale data.
