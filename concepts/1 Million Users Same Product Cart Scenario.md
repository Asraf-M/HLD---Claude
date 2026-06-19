# 1 Million Users Adding Same Product to Cart at Same Time

## The Scenario

It's a flash sale: a limited-stock item (10,000 units) drops at 12:00 PM.

Within 10 seconds:
- 1,000,000 concurrent users try to add it to their cart
- Each user requests: `POST /cart/add-item { product_id: 12345, quantity: 1 }`

What happens?

---

## Without Any Optimization (Everything Breaks)

### What Database Sees

```
1,000,000 simultaneous writes to:
  - inventory table (product_id = 12345)
  - cart table (1M user rows)
```

### Scenario 1: Pessimistic Locking (FOR UPDATE)

```sql
BEGIN;
SELECT stock FROM inventory WHERE product_id = 12345 FOR UPDATE;
-- Only ONE transaction can acquire lock

-- Other 999,999 transactions QUEUE behind the lock
-- Average wait = (1M requests * avg_transaction_time) / 1 = disaster

-- By the time 10K approvals complete, waiting queue is so long that
-- many hit timeout (30-60 seconds) -> users give up -> cart operations fail
-- After timeout, they are not refunded from queue -> lost sales
```

Outcome:
- Throughput bottleneck at the inventory row lock
- p99 latency = potentially minutes
- Only first ~10,000 users succeed after long waits; rest timeout
- Lost revenue (users never get confirmation)

### Scenario 2: Optimistic Locking (Version Column)

```sql
UPDATE inventory
SET quantity = quantity - 1,
    version = version + 1
WHERE product_id = 12345
  AND version = 7;

-- Without coordination, many transactions read version=7
-- All 1M attempt update at same time
-- Only 1 succeeds (updates to version=8)
-- Remaining 999,999 fail version check -> retry

-- Retry loop:
-- T1: read -> version=8
-- T1: update -> version=9
-- Remaining 999,998 retry...
-- ... this continues
```

Outcome:
- Retry storm: database CPU spikes to 100%
- Query storm: 1M reads + 1M failed writes + retries = 10-100M database operations/sec
- P99 latency = seconds to minutes
- Only 10K users get item; rest timeout waiting for retry success
- System is thrashing

### Scenario 3: No Locking / Race Condition

```java
// API reads stock, checks it > 0, then decrements
int stock = db.query("SELECT stock FROM inventory WHERE product_id = ?").get(0);
if (stock > 0) {
    db.execute("UPDATE inventory SET stock = stock - 1 WHERE product_id = ?");
}

// But 1M requests read stock=10000 before ANY decrement
// All 1M think "yes, stock available"
// All 1M write decrement
// Final stock = ??? (could be -999990 if all write)
```

Outcome:
- Overselling: system sells 1M items when only 10K available
- Massive refund chaos
- Inventory becomes negative (corrupted state)
- Reputation damage

---

## Correct Approach: Multi-Layer Strategy

### Layer 1: Distributed Cache (Redis)

Instead of hitting inventory DB for every request, use cache as gate.

```
POST /cart/add-item
  1. Check Redis cache: available_stock
  2. If available, decrement in Redis (atomic)
  3. Later, sync to database asynchronously
```

Redis operation:
```
DECR inventory:product:12345:available
```

Single Redis node can handle 100K+ ops/sec. 10M operations across a distributed Redis cluster spread across multiple nodes.

First 10,000 requests:
```
Redis: available = 10000
Request 1:     DECR -> 9999, add to cart [OK]
Request 2:     DECR -> 9998, add to cart [OK]
...
Request 10000: DECR -> 0,    add to cart [OK]
Request 10001: DECR -> -1,   REJECT "Out of stock"
```

Problem: But what about consistency with database? And what if Redis crashes?

### Layer 2: Request Queue / Rate Limiting

Instead of serving all 1M simultaneously, gate first then queue.

```
Request arrives
  -> Check cache stock
  -> If available: quick add to cart
  -> If unavailable: reject immediately with 429 (out of stock)

  -> Enqueue accepted event to Kafka for async processing
  -> Return immediately to user
```

This prevents thundering herd:
- Inventory cache handles 100K+ checks/sec
- Only successful adds go to persistent queue
- Database gets steady 10K writes (not 1M)

### Layer 3: Async Inventory Sync

Background worker processes successful additions and updates DB.

```
Kafka topic: cart-add-events
Consumer batch:
  1000 successful adds per batch
  Batch write to DB: 1 query, 1000 rows affected

Performance:
  1000 additions / 1 query = much better than 1000 queries
```

### Layer 4: Inventory Hold / Timeout

Added to cart does not equal purchased.

```
inventory table:
  product_id  | total_stock | reserved | available
  12345       | 10000       | 9500     | 500

When user adds:    reserved += 1
When user abandons cart after 10 min: reserved -= 1
```

This prevents hoarding (if user adds 100 units, other users can still add).

---

## Architecture Under Load

```
1M concurrent users
        |
[Load Balancer] (distribute across 10 API nodes)
        |
[API Servers] (10 nodes)
        +-- Read from Redis cache (in-memory, <1ms)
        +-- Check available stock
        +-- If yes: queue to Kafka
        +-- Return 200 or 429 immediately

[Redis Cluster] (5 nodes, replicas)
  - DECR inventory:product:12345
  - Cache consistency maintained

[Kafka Topic: cart-add-events]
  - Throughput: easily handles 100K msgs/sec

[Inventory Consumer] (3 workers)
  - Batch 1000 events every 100ms
  - Bulk insert to database

[Primary Database]
  - Handles ~10K inserts/sec (batch style)
  - No lock contention (1 batch at a time, not 1M concurrent)
```

---

## Admission Tradeoff: Load Balancer First vs Queue First

When 1M requests arrive, there are two patterns:

### Option A (Recommended for most flash sales)

```
Load Balancer -> API Nodes -> Redis atomic gate -> Kafka (only accepted events)
```

How it works:
1. Requests are distributed across API nodes
2. API does atomic stock check/decrement in Redis
3. If stock exists, accept and publish event to Kafka for async durable DB update
4. If stock is 0, reject immediately (429 out of stock)

Important clarification for point 3:
- Redis gives fast accept/reject on hot path, but DB remains source of truth.
- Kafka consumer persists reservation/cart/inventory events to DB asynchronously.
- Consumers must be idempotent to handle retries/duplicates safely.

Pros:
- Very fast user response (accept/reject in milliseconds)
- Kafka is not filled with guaranteed-reject requests
- Lower backlog and lower infra cost

Tradeoffs:
- Strict global first-come fairness is harder to prove
- API layer must handle idempotency/retry carefully

### Option B (Queue First)

```
Load Balancer -> API Nodes -> Kafka (all requests) -> Consumers -> Redis/DB decision
```

How it works:
1. All requests are queued first
2. Consumers process in queue order and decide success/reject

Pros:
- Better auditability and ordering/fairness story
- Natural smoothing via queue backpressure

Tradeoffs:
- User response is delayed by queue wait time
- Large backlog includes many requests that were always going to fail
- Higher operational pressure during peak spikes

### Practical Decision Rule

- Use Option A when fast UX is the top priority
- Use Option B when strict fairness/lottery semantics are mandatory

In most e-commerce flash sales, Option A is preferred.

---

## Expected Outcome with Optimization

```
Timeline: t=0 to t=10 (seconds)

t=0:    1M requests arrive
t=0-1:  Redis gets 1M DECR operations, serves ~100K/sec
        First 10K accepted (stock goes from 10K to 0 atomically) [OK]

t=1-2:  Next batch of requests hit "out of stock" in cache
        Immediate rejection, no DB query [OK]

t=2-10: Remaining requests get 429 immediately

t=5-15: Background Kafka consumer writes 10K successful adds to DB
        Database: ~1000 writes/sec (batched) [OK]

Final state:
- 10,000 users have product in cart [OK]
- 990,000 users rejected with "out of stock" (in <100ms) [OK]
- Database consistency maintained [OK]
- System stayed healthy (CPU, memory, throughput all normal) [OK]
```

---

## Detailed Tradeoff Analysis

| Approach | Correctness | Latency | Throughput | Complexity |
|----------|-------------|---------|-----------|------------|
| Pessimistic lock | Strong | Minutes | Low | Simple |
| Optimistic lock | Strong | Seconds (retries) | Degrades under contention | Simple |
| No lock / race | Overselling | Fast | High | Simple |
| Redis + Queue + Async | Eventual strong | <100ms | Very high | Complex |

---

## Common Doubt: One Redis Per API Node or Shared Redis?

Question:
If 1 million requests are distributed to 20 API servers, should each server have its own Redis, or should all 20 use one common Redis layer?

Answer:
Use one shared Redis deployment for all API nodes (with replicas for HA), not separate per-server Redis for stock decisions.

Why:
1. Inventory is global state; all API nodes must see the same stock number
2. Atomic decrement only works correctly if everyone updates the same key space
3. Per-server Redis can create split state and overselling risk

| Design | Pros | Tradeoffs |
|--------|------|-----------|
| Shared Redis (recommended) | Single source for hot inventory counters, consistent stock decisions | Hot-key pressure on one shard/primary; requires HA setup |
| Per-server Redis (avoid for inventory) | Local low-latency cache reads per node | No global consistency, difficult reconciliation, high oversell risk |

Important note:
- Redis replicas improve availability/read scale.
- Replicas do not increase write capacity for one hot key; that key still writes to one primary owner.

---

## Key Insights

1. Cache is essential for hot keys. Redis can handle 1M+ operations on same key in seconds. Database cannot.
2. Async processing decouples load. Instead of 1M concurrent DB writes, get 10K batched writes.
3. Early rejection prevents waste. Reject "out of stock" in cache tier instead of database. Saves 10,000x on latency and DB load for rejections.
4. Idempotency matters. If user retries, need to detect duplicate add.
5. System degradation is better than system failure. Some requests succeed, others immediately rejected.

---

## Real-World Example: Black Friday Sale

Amazon/Walmart/Flipkart handle this every year:

1. Pre-warming: Heavy items are pre-cached in CDN + Redis
2. Rate limiting: Queue-based admission control
3. Inventory sync: Periodic batch writes, not per-request
4. Circuit breaker: If DB queue gets too long, reject new requests gracefully
5. Monitoring: Watch QPS, p99 latency, cache hit rate, DB write backlog

Result: They sell items correctly without system collapse.

---

## Interview Answer (Concise)

How would you handle 1M users adding the same limited-stock product to cart simultaneously?

1. Redis cache for inventory count - can handle 1M ops/sec in parallel, not database
2. Async queue (Kafka) - decouple request handling from DB writes
3. Batch writes - instead of 1M DB queries, batch into bulk inserts
4. Early rejection in cache - if stock is 0 in Redis, reject immediately without DB hit
5. Idempotency keys - prevent duplicate adds on user retry
6. Eventual consistency - database is source of truth but eventually updated asynchronously

Result: Users get <100ms response, system stays stable, inventory is correct.
