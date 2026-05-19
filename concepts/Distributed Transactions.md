# Distributed Transactions

## The Problem

In a monolith with a single database, a transaction is simple — ACID guarantees all-or-nothing:

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- debit Alice
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- credit Bob
COMMIT;  -- either both succeed, or neither
```

In a **microservices** architecture, each service has its own database:

```
Order Service   → Order DB
Payment Service → Payment DB
Inventory Service → Inventory DB
```

Now "create order + charge payment + reserve inventory" spans **three separate databases**. A single ACID transaction is impossible — each DB only controls its own data.

**Failure scenarios:**
```
1. Order created ✅
2. Payment charged ✅
3. Inventory reservation FAILS ❌

→ Customer was charged but no inventory reserved — inconsistent state!
```

---

## Solution 1: Two-Phase Commit (2PC)

The classic protocol to achieve atomicity across multiple participants.

### How It Works

**Two phases:**

**Phase 1 — Prepare:**
```
Coordinator → Participant A: "Can you commit?"
Coordinator → Participant B: "Can you commit?"
Coordinator → Participant C: "Can you commit?"

Each participant:
  - Acquires locks on the data
  - Writes changes to a WAL (undo log)
  - Replies YES or NO
```

**Phase 2 — Commit or Abort:**
```
If ALL said YES:
  Coordinator → All: "COMMIT"
  All participants commit → release locks

If ANY said NO (or timeout):
  Coordinator → All: "ROLLBACK"
  All participants rollback using undo log → release locks
```

### The Problem: Coordinator Failure

```
Phase 2 begins...
Participant A commits ✅
Coordinator CRASHES 💥
Participants B and C are stuck:
  - They said YES (locks acquired, changes prepared)
  - Don't know whether to commit or rollback
  - Can't release locks → entire system blocked
```

This is why 2PC is sometimes called "blocking commit protocol."

### When to Use 2PC

- Within a **single organization** where you control all participants
- **Same datacenter** — low latency, low failure probability
- **Short transactions** — locks held briefly
- Examples: distributed SQL databases (PostgreSQL with distributed extension like Citus), MySQL Group Replication

### 2PC Limitations

| Problem | Description |
|---------|-------------|
| **Blocking** | Coordinator failure leaves participants locked |
| **Latency** | 2 round trips minimum across all participants |
| **Scalability** | Locks held during network round trips → low throughput |
| **Not suitable for microservices** | Different teams own different services; can't coordinate at DB level |

---

## Solution 2: The Saga Pattern

Instead of one big transaction, break it into a **sequence of local transactions**, each with a **compensating transaction** to undo it if needed.

> **Analogy:** Booking a trip. You book flight, then hotel, then rental car separately. If the rental car is unavailable, you cancel the hotel (compensating action), then cancel the flight (compensating action). You didn't lock the airline and hotel systems for the duration.

### Structure

```
T1 → T2 → T3 → ... → Tn   (happy path)

If Ti fails:
  Run C(i-1) → C(i-2) → ... → C1  (compensating transactions in reverse)
```

**Example: E-commerce Order Saga**
```
T1: Create Order        → C1: Cancel Order
T2: Reserve Inventory   → C2: Release Inventory
T3: Charge Payment      → C3: Refund Payment
T4: Ship Order          → C4: Arrange Return

If T3 (payment) fails:
  Run C2 (release inventory) → C1 (cancel order)
  Customer gets order cancelled, inventory released, no charge
```

---

## Saga: Two Implementation Styles

### 1. Choreography (Event-Driven)

Each service listens to events and decides what to do next. No central coordinator.

```
OrderService:   CREATE ORDER → publishes "OrderCreated" event
InventoryService: listens → reserves stock → publishes "InventoryReserved"
PaymentService:   listens → charges card → publishes "PaymentCharged"
ShippingService:  listens → schedules shipment → publishes "OrderShipped"

On failure:
PaymentService fails → publishes "PaymentFailed"
InventoryService listens → releases stock → publishes "InventoryReleased"
OrderService listens → cancels order → publishes "OrderCancelled"
```

**Pros:**
- Simple, no single point of failure
- Services are loosely coupled

**Cons:**
- Hard to understand overall flow (distributed across many services)
- Difficult to debug (which step failed?)
- Cyclic dependencies possible

**Best for:** Simple sagas with 2-3 steps

### 2. Orchestration (Central Coordinator)

A central **saga orchestrator** tells each service what to do step-by-step.

```
SagaOrchestrator:
  1. → OrderService:    "Create order"      ← success
  2. → InventoryService: "Reserve inventory" ← success
  3. → PaymentService:  "Charge payment"    ← FAILURE
  
  (compensate in reverse)
  4. → InventoryService: "Release inventory" ← success
  5. → OrderService:    "Cancel order"      ← success
```

**Pros:**
- Clear, visible flow in one place
- Easy to debug and monitor
- Handles complex workflows

**Cons:**
- Central orchestrator = potential bottleneck/single point of failure (mitigated by making it stateless + using a persistent store for saga state)

**Best for:** Complex sagas with many steps, business-critical workflows

**Tools:** AWS Step Functions, Temporal, Conductor (Netflix), Apache Camel Saga

---

## Compensating Transactions

Compensating transactions are NOT the same as rollback. They are new operations that undo the business effect:

| Original Transaction | Compensating Transaction |
|---------------------|-------------------------|
| Create Order | Cancel Order |
| Charge $100 | Refund $100 |
| Reserve seat 12A | Release seat 12A |
| Send welcome email | ??? |

**Problem:** Some operations cannot be compensated (you can't un-send an email, un-notify a user).

**Solutions:**
1. Don't send the email until saga is confirmed complete
2. Send a follow-up "sorry, order cancelled" email
3. Accept that some notifications are best-effort

---

## 2PC vs Saga: Comparison

| | 2PC | Saga |
|--|-----|------|
| **Consistency** | Strong (ACID) | Eventual (BASE) |
| **Failure handling** | Atomic rollback | Compensating transactions |
| **Blocking** | Yes (locks held) | No |
| **Latency** | Higher (2 round trips + locks) | Lower |
| **Scalability** | Low | High |
| **Partial failure** | Never visible | Temporarily visible |
| **Use case** | Same org, same DC, DB-level control | Microservices, cross-service workflows |

---

## Outbox Pattern (Making Sagas Reliable)

A critical problem: how do you atomically update your DB AND publish an event?

**Naive approach (broken):**
```python
db.update_order(order_id, status="created")
kafka.publish("OrderCreated", order_id)  # what if this fails? DB updated, event never published
```

**Outbox pattern (correct):**
```python
with db.transaction():
    db.update_order(order_id, status="created")
    db.insert_outbox(event_type="OrderCreated", payload=order_id)
    # Both in ONE local transaction — atomically committed

# Separate process: tail the outbox table and publish to Kafka
# Use Debezium (CDC) to read DB WAL and publish events
```

The outbox table is in **the same database** as the order → local ACID transaction guarantees both succeed or both fail. Event publishing is decoupled and retried separately.

---

## Interview Q&A

**Q: Why can't you use a single database transaction across microservices?**  
A: Each microservice owns its own database. A database transaction only spans one database instance. To achieve atomicity across multiple databases, you'd need a distributed protocol like 2PC, but this requires all databases to participate in the same protocol — which isn't practical when different teams own different services with different databases.

**Q: Explain Two-Phase Commit (2PC).**  
A: 2PC has two phases. Prepare: the coordinator asks all participants "can you commit?" — each acquires locks and writes to WAL, then responds YES/NO. Commit: if all said YES, coordinator sends COMMIT and everyone commits. If any said NO or timed out, coordinator sends ROLLBACK and everyone rolls back. The problem: if the coordinator crashes after phase 1, participants are stuck holding locks indefinitely (blocking protocol).

**Q: What is the Saga pattern? When would you use it over 2PC?**  
A: Saga breaks a distributed transaction into a sequence of local transactions, each with a compensating transaction to undo it. If a step fails, the saga runs compensating transactions in reverse to undo previous steps. Use Saga for microservices (2PC isn't practical) and when eventual consistency is acceptable. Saga has higher availability and scalability than 2PC, but allows temporarily inconsistent state during execution.

**Q: What is the Outbox pattern and why is it needed?**  
A: It solves the dual-write problem: you can't atomically update a database AND publish an event to Kafka in a single operation. The outbox pattern writes the event to an outbox table in the same local database transaction as the business data update. A separate process (like Debezium CDC) reads the outbox table and publishes events to Kafka. This guarantees the event is always published exactly once if the transaction committed.

**Q: What are compensating transactions? Are they the same as rollback?**  
A: No. A database rollback undoes uncommitted changes atomically. A compensating transaction is a new business operation that reverses a previously committed transaction (e.g., refund for a charge, cancel for a create). Some operations can't be compensated (sending an email), so the saga design must either delay those operations until the saga succeeds, or accept they are best-effort.
