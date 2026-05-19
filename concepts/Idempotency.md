# Idempotency

## What Is It?

An operation is **idempotent** if performing it multiple times produces the **same result** as performing it once.

> **Analogy:** Pressing an elevator button. Pressing it once calls the elevator. Pressing it 5 more times doesn't call 5 elevators — same result. The operation is idempotent.

In distributed systems: networks fail, timeouts happen, clients retry. Idempotency ensures **retries are safe** — the same request processed twice doesn't cause double-charges, duplicate orders, or corrupt state.

---

## Why It Matters

```
Client → POST /charge-card → Server
         ... timeout! ...
         Did the server process it? Unknown.

Without idempotency:
  Client retries → POST /charge-card again
  Server processes AGAIN → customer charged TWICE 💸

With idempotency:
  Client retries with same idempotency key
  Server detects duplicate → returns same success response
  Customer charged exactly ONCE ✅
```

---

## HTTP Methods and Idempotency

| Method | Idempotent? | Safe? | Notes |
|--------|------------|-------|-------|
| **GET** | ✅ Yes | ✅ Yes | Read-only, no side effects |
| **PUT** | ✅ Yes | No | "Set resource to this value" — same result every time |
| **DELETE** | ✅ Yes | No | First delete removes it; second returns 404 but no new harm |
| **HEAD** | ✅ Yes | ✅ Yes | Like GET but headers only |
| **POST** | ❌ No | No | Creates a new resource each time — inherently non-idempotent |
| **PATCH** | ❌ No | No | Relative updates like `+1` are not idempotent |

**PUT vs PATCH:**
```http
PUT  /cart     {"items": ["apple", "banana"]}  → idempotent (sets cart to this)
PATCH /cart    {"add": "banana"}                → NOT idempotent (+1 banana each call)
```

---

## Idempotency Keys

The standard solution for making non-idempotent operations (like POST) safe to retry.

**Client generates a unique key** (UUID) per logical operation and sends it as a header.

```http
POST /v1/payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{ "amount": 100, "currency": "USD", "card": "tok_visa" }
```

**Server logic:**
```python
def charge_card(request):
    key = request.headers["Idempotency-Key"]
    
    # Check if already processed
    cached = redis.get(f"idempotency:{key}")
    if cached:
        return cached  # return same response as before
    
    # Process the payment (first time)
    result = payment_processor.charge(request.body)
    
    # Store result with TTL (e.g., 24 hours)
    redis.set(f"idempotency:{key}", result, ttl=86400)
    
    return result
```

**Retry behavior:**
```
First request:  key=abc123 → charge $100 → store result → return 200
Client timeout → retry
Second request: key=abc123 → found in Redis → return same 200 (no charge)
```

**Used by:** Stripe, Braintree, PayPal, Twilio — all payment/messaging APIs.

---

## Implementing Idempotency in a Database

Instead of Redis, use a DB table to persist idempotency records (survives cache restarts):

```sql
CREATE TABLE idempotency_keys (
    key         UUID PRIMARY KEY,
    request_hash TEXT,    -- hash of request body (detect changed payloads)
    response    JSONB,    -- stored response to replay
    status      TEXT,     -- 'processing' | 'completed' | 'failed'
    created_at  TIMESTAMP DEFAULT now(),
    expires_at  TIMESTAMP -- auto-cleanup
);
```

```python
def handle_request(key, body):
    with transaction():
        record = db.query("SELECT * FROM idempotency_keys WHERE key = ?", key)
        
        if record:
            if record.status == 'processing':
                raise ConflictError("Request in progress")  # concurrent retry
            if record.request_hash != hash(body):
                raise BadRequestError("Same key, different body")
            return record.response  # replay stored response
        
        # Insert with 'processing' status to lock concurrent requests
        db.execute("INSERT INTO idempotency_keys (key, status) VALUES (?, 'processing')", key)
    
    # Do the actual work outside the lock
    result = do_actual_work(body)
    
    # Update to 'completed'
    db.execute("UPDATE idempotency_keys SET status='completed', response=? WHERE key=?",
               result, key)
    return result
```

---

## At-Least-Once + Idempotent = Effectively Exactly-Once

In distributed systems, exactly-once delivery is hard (and expensive). The practical alternative:

```
Message Queue: at-least-once delivery (may deliver duplicate messages)
+
Consumer: idempotent processing (deduplicates by message ID)
=
Effectively exactly-once behavior
```

**Pattern: Deduplication table**
```python
def process_message(msg):
    # Check if already processed
    if db.exists("SELECT 1 FROM processed_messages WHERE msg_id = ?", msg.id):
        return  # skip duplicate
    
    # Process message
    do_work(msg)
    
    # Mark as processed (atomic with do_work in a transaction)
    db.insert("INSERT INTO processed_messages (msg_id) VALUES (?)", msg.id)
```

---

## Common Idempotency Patterns

### 1. Conditional Updates (Optimistic Locking)
```sql
UPDATE accounts
SET balance = balance - 100, version = version + 1
WHERE id = 42 AND version = 7;  -- only update if version matches
-- Returns 0 rows updated if version changed → retry or fail
```

### 2. Upsert (INSERT OR UPDATE)
```sql
INSERT INTO user_settings (user_id, theme)
VALUES (42, 'dark')
ON CONFLICT (user_id) DO UPDATE SET theme = EXCLUDED.theme;
-- Idempotent: always results in theme='dark' for user 42
```

### 3. Set Operations (Not Increment)
```
Not idempotent:  SET likes = likes + 1     (retry → adds twice)
Idempotent:      INSERT INTO user_likes (user_id, post_id) ON CONFLICT DO NOTHING
                 COUNT(*) FROM user_likes WHERE post_id = X
```

---

## Real-World Examples

### Stripe Payments
```http
POST /v1/charges
Idempotency-Key: <client-generated UUID>
```
Stripe stores the result for 24 hours. Any retry with the same key returns the original response without re-charging.

### Email Sending
Bad: `sendEmail(to, subject, body)` called twice → two emails sent.  
Good: `sendEmail(to, subject, body, idempotencyKey)` → server checks if this key was already sent.

### Database Migrations
Each migration script has a version number. Migration runner checks if version already applied → skips.

### Kafka Consumer Processing
Every Kafka message has an offset. Consumer tracks last processed offset → skips messages already processed.

---

## Interview Q&A

**Q: What is idempotency and why does it matter in distributed systems?**  
A: An operation is idempotent if running it multiple times produces the same result as running it once. It matters because network failures and timeouts are common — clients retry failed requests. Without idempotency, retries cause duplicate effects (double-charged payments, duplicate emails, duplicate orders). With idempotency, retries are safe.

**Q: How do you implement an idempotent payment API?**  
A: Client generates a UUID (idempotency key) per payment attempt and sends it as a header. Server checks if the key already exists in a store (Redis or DB). If found: return the stored response without re-processing. If not found: process payment, store the result keyed by the idempotency key (with TTL), return result. Any retry with the same key gets the cached response.

**Q: Which HTTP methods are idempotent?**  
A: GET, HEAD, PUT, DELETE are idempotent. POST is not (each call creates a new resource). PATCH usually isn't (relative updates like increment are not idempotent). To make POST idempotent, use idempotency keys.

**Q: What is the difference between idempotent and safe?**  
A: Safe means no side effects (read-only). Idempotent means multiple identical calls have the same effect as one call. GET is both safe and idempotent. DELETE is idempotent but not safe (first call has a side effect — deletes the resource; subsequent calls return 404 but no new harm). PUT is idempotent but not safe (modifies the resource).

**Q: How do you achieve exactly-once processing with a message queue?**  
A: Use at-least-once delivery (Kafka guarantees this) + idempotent consumers. Each message has a unique ID. Consumer checks a deduplication table before processing. If ID already present: skip. If not: process + insert ID atomically (in a transaction). This gives effectively exactly-once semantics without expensive distributed transaction protocols.
