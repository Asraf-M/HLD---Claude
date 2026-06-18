# Optimistic Locking vs Pessimistic Locking

## What Problem Are We Solving?

When multiple users/services update the same row at the same time, we can get:
- Lost updates (one write overwrites another)
- Inconsistent business state
- Race conditions

Example:
- User A reads account balance = 100
- User B reads account balance = 100
- A withdraws 20 -> writes 80
- B withdraws 30 -> writes 70

Expected final balance = 50, but actual became 70 (A's update lost).

Locking strategies prevent this.

---

## Two Approaches

### 1. Optimistic Locking (Assume conflicts are rare)

Idea:
- Allow concurrent reads/writes without holding DB lock for long
- At update time, verify row did not change since read
- If changed, reject/retry

Most common implementation:
- Add `version` column (or use `updated_at`)
- Update with condition on current version

```sql
-- table
CREATE TABLE inventory (
  product_id BIGINT PRIMARY KEY,
  quantity INT NOT NULL,
  version INT NOT NULL
);

-- read
SELECT product_id, quantity, version
FROM inventory
WHERE product_id = 101;
-- returns: quantity=12, version=7

-- write attempt
UPDATE inventory
SET quantity = 11,
    version = version + 1
WHERE product_id = 101
  AND version = 7;

-- If rows_affected = 1 -> success
-- If rows_affected = 0 -> someone else updated first (conflict)
```

### 2. Pessimistic Locking (Assume conflicts are likely)

Idea:
- Lock row before modifying
- Other transactions wait (or fail by timeout)
- Prevent conflicting writes up front

Most common SQL pattern:
- `SELECT ... FOR UPDATE`

```sql
BEGIN;

SELECT quantity
FROM inventory
WHERE product_id = 101
FOR UPDATE;   -- row lock acquired

UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 101;

COMMIT;
```

While lock is held, another transaction trying to `FOR UPDATE` same row blocks.

---

## Visual Timeline (Same Row Update)

### Optimistic Locking

```text
T1: read row (version=7)
T2: read row (version=7)
T1: update ... WHERE version=7  -> success (version=8)
T2: update ... WHERE version=7  -> 0 rows (conflict), retry needed
```

### Pessimistic Locking

```text
T1: SELECT ... FOR UPDATE  -> lock acquired
T2: SELECT ... FOR UPDATE  -> waits
T1: UPDATE + COMMIT        -> lock released
T2: continues with latest row
```

---

## Real-World Examples

## Example A: E-commerce Inventory

### Optimistic version
Good when contention is low across many products.

```sql
UPDATE inventory
SET quantity = quantity - 1,
    version = version + 1
WHERE product_id = :pid
  AND quantity > 0
  AND version = :version_from_read;
```

If update fails (`rows_affected=0`), app retries by re-reading latest row.

### Pessimistic version
Good for flash sale on one hot product where collisions are frequent.

```sql
BEGIN;
SELECT quantity
FROM inventory
WHERE product_id = :pid
FOR UPDATE;

-- validate quantity > 0
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = :pid;
COMMIT;
```

Tradeoff:
- Correct and simple under heavy contention
- But many waiting transactions can increase latency

---

## Example B: Banking Transfer

Pessimistic locking is often preferred for critical money movement.

```sql
BEGIN;

SELECT balance FROM accounts WHERE id = :from_id FOR UPDATE;
SELECT balance FROM accounts WHERE id = :to_id   FOR UPDATE;

UPDATE accounts SET balance = balance - :amt WHERE id = :from_id;
UPDATE accounts SET balance = balance + :amt WHERE id = :to_id;

COMMIT;
```

Why:
- Strong correctness
- Avoids conflicting updates on same account

But must avoid deadlocks by locking rows in consistent order (for example, always lock smaller account id first).

---

## Detailed Tradeoffs

| Factor | Optimistic Locking | Pessimistic Locking |
|--------|--------------------|---------------------|
| Conflict assumption | Rare conflicts | Frequent conflicts |
| Concurrency | High (no long row lock) | Lower (waiting/blocking) |
| Latency under low contention | Usually lower | Usually higher |
| Latency under high contention | Can degrade (many retries) | Predictable but may queue |
| Throughput under low contention | High | Moderate |
| Throughput under hot key | Retry storms possible | Better correctness, lower parallelism |
| DB lock duration | Minimal | Can be long if transaction is long |
| Deadlock risk | Low | Higher (if lock order inconsistent) |
| App complexity | Retry + backoff logic required | Simpler conflict model, harder lock tuning |
| Best for | Most web CRUD, profiles, catalogs | Payments, inventory hot rows, critical counters |

---

## Failure Modes and Mitigations

### Optimistic Locking Failure Mode: Retry Storm

If 10K requests update one row, many fail version check and retry repeatedly.

Mitigations:
1. Exponential backoff with jitter
2. Limit max retries (for example, 3)
3. Queue hot-key updates through a single worker
4. Use sharded counters / event log aggregation

### Pessimistic Locking Failure Mode: Deadlocks and Long Waits

Transactions lock rows in different order and block each other.

Mitigations:
1. Global lock order rule (always lock by ascending id)
2. Keep transactions short
3. Use lock timeout and fail fast
4. Monitor deadlock metrics from DB

---

## Performance Intuition with Numbers

Assume 1,000 update requests/sec on one product row.

### Low conflict scenario (many products)
- Optimistic: ~95-99% first-try success, excellent throughput
- Pessimistic: extra lock overhead, no major correctness gain

### High conflict scenario (one hot product)
- Optimistic: first-try success may drop below 20%, retries inflate DB traffic
- Pessimistic: queue forms, but each commit is valid without retry storms

So choice depends more on contention pattern than absolute QPS.

---

## Choosing Strategy (Practical Rule)

Use **optimistic locking** when:
1. Same row is not updated frequently by many actors
2. You want high concurrency and low lock overhead
3. Retries are acceptable in business flow

Use **pessimistic locking** when:
1. Data correctness is critical (money, seats, stock)
2. Same row is highly contested
3. You need predictable conflict handling over retries

Hybrid pattern is common:
- Optimistic by default
- Switch to pessimistic for known hot rows or critical workflows

---

## ORM Examples

### JPA/Hibernate Optimistic

```java
@Entity
public class Product {
    @Id
    private Long id;

    private Integer quantity;

    @Version
    private Integer version;
}
```

Hibernate generates conditional update with version check automatically.

### JPA/Hibernate Pessimistic

```java
Product p = entityManager.find(
    Product.class,
    productId,
    LockModeType.PESSIMISTIC_WRITE
);
```

This translates to DB row locking (`FOR UPDATE` style behavior).

---

## Interview Q&A

**Q: What is the main difference between optimistic and pessimistic locking?**
A: Optimistic locking does not lock rows early; it detects conflict at commit/update using version checks. Pessimistic locking locks rows before update, forcing others to wait. Optimistic favors concurrency; pessimistic favors conflict prevention.

**Q: Why can optimistic locking fail under heavy contention?**
A: Many transactions read same version and fail update condition, causing repeated retries and extra load (retry storm).

**Q: Why can pessimistic locking reduce throughput?**
A: Waiting transactions queue behind locks. If lock-holding transaction is slow, tail latency rises and throughput drops.

**Q: How do you avoid deadlocks in pessimistic locking?**
A: Lock resources in a consistent global order, keep transactions short, and set lock timeout.

**Q: Which one should be used for payment systems?**
A: Usually pessimistic or carefully designed serializable workflows for high correctness paths, sometimes combined with idempotency and ledger-based event models.
