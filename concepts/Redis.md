# Redis

## What Is It?

**Redis** is an in-memory data store used as a cache, key-value store, pub/sub broker, and lightweight data structure engine.

It is popular because it is:
- Very fast (microseconds to low milliseconds)
- Simple for common patterns (cache, counters, rate limiting, session store)
- Rich in data structures (strings, hashes, lists, sets, sorted sets, streams)

> **Simple view:** Redis keeps hot data in RAM so your app avoids slow database trips for every request.

---

## Why Is Redis So Important?

### 1. It Reduces Database Load Dramatically
If your app keeps hitting DB for the same data, DB becomes the bottleneck.

With Redis cache:
- First request: DB read + cache write
- Next requests: served from Redis

This can reduce DB QPS by 10x to 100x in read-heavy systems.

### 2. It Improves User Latency
Typical DB query can take 10-100 ms (or worse under load).
Redis lookup is often <1 ms.

Even a small cache hit rate can noticeably improve p95 latency.

### 3. It Handles Bursty Traffic Better
Redis absorbs sudden spikes (product launch, live event, sale) by serving hot keys quickly.

### 4. It Enables Common System Design Building Blocks
Redis is often used for:
- Session storage
- API rate limiting (token bucket/sliding window)
- Distributed locks (careful usage)
- Leaderboards (sorted sets)
- Counters and quotas
- Idempotency key store

---

## Quick Comparison: Without vs With Redis

| | Without Redis | With Redis |
|--|---------------|------------|
| Read path | App -> DB | App -> Redis (hit) -> DB (miss) |
| Avg latency | Higher | Lower |
| DB pressure | High | Lower |
| Spike handling | Fragile | Better |
| Cost profile | DB-heavy | Memory-heavy |

---

## Typical Cache Flow

### Read-Through Pattern (Cache-Aside)
1. App checks Redis for `key`
2. If hit -> return value
3. If miss -> query DB
4. Store result in Redis with TTL
5. Return value

```text
Client -> API -> Redis
              | hit -> return fast
              | miss -> DB -> write Redis (TTL) -> return
```

### Example Numbers

```text
Traffic: 200,000 read requests/sec
Cache hit rate: 95%

DB reads without cache: 200,000 rps
DB reads with cache:     10,000 rps

Result: DB read load reduced by 20x
```

---

## Common Redis Data Types and Use Cases

- **String**: cache object, token, feature flag
- **Hash**: user profile fields (`user:123 -> name, tier, locale`)
- **Set**: unique memberships (followers, tags)
- **Sorted Set (ZSET)**: leaderboards, ranking by score/time
- **List**: simple queues (legacy), recent items
- **Stream**: event log / consumer groups
- **Bitmap / HyperLogLog**: compact analytics patterns

---

## Why It Works So Well in Distributed Systems

1. **In-memory speed** keeps tail latency low
2. **TTL support** controls staleness and memory growth
3. **Atomic operations** (`INCR`, `SETNX`, Lua scripts) simplify concurrency
4. **Replication + Sentinel/Cluster** improves availability and scale

---

## Tradeoffs and Risks (Important)

### 1. Memory Is Expensive
Redis is RAM-first. Large datasets can get costly quickly.

Tradeoff:
- Faster reads
- Higher infrastructure cost than disk-based stores

### 2. Cache Invalidation Is Hard
Stale data can be served if invalidation is wrong.

Tradeoff:
- Better latency
- Additional consistency complexity

### 3. Evictions Can Hurt Predictability
When memory is full, Redis evicts keys based on policy (`allkeys-lru`, etc.).

Tradeoff:
- Service keeps running
- Important keys may disappear if policy/key design is poor

### 4. Persistence Is Optional and Not Free
Redis can persist via RDB/AOF, but durability settings affect performance.

Tradeoff:
- Better durability with AOF/fsync
- More disk I/O and potentially higher write latency

### 5. Single Hot Key Problem
A very popular key can overload one shard/node.

Tradeoff:
- Great average performance
- Need hot-key mitigation (replication fanout, local caching, key splitting)

### 6. Distributed Locking Nuance
Redis locks are useful but easy to misuse under partitions/timeouts.

Tradeoff:
- Easy coordination primitive
- Must design carefully for correctness-critical workflows

---

## Practical Best Practices

1. Use TTL for almost all cache keys
2. Add jitter to TTL to avoid synchronized expiry
3. Use cache-aside with negative caching for not-found keys
4. Protect DB on misses (request coalescing / cache lock)
5. Monitor hit rate, evictions, memory fragmentation, p99 latency
6. Define key naming conventions and size limits
7. Keep Redis for hot/operational data, not as primary source of truth

---

## When Redis Is a Great Fit

- Read-heavy workloads with repeated access patterns
- Low-latency APIs
- Session/token/idempotency storage
- Counters, quotas, and leaderboards
- Short-lived state in high-scale systems

## When Redis Is Not Enough Alone

- Long-term durable system-of-record data
- Large analytics scans over huge historical data
- Strict relational constraints and complex joins

Use Redis with a primary DB (PostgreSQL/MySQL/NoSQL), not usually as replacement.

---

## Redis vs SQL vs NoSQL: Read/Write Speed Comparison

### Typical Latency Ranges (Practical)

| Store Type | Read Latency | Write Latency | Why It Behaves This Way |
|------------|--------------|---------------|--------------------------|
| Redis (in-memory) | ~0.1 to 1 ms | ~0.2 to 2 ms | Data in RAM, simple key access, minimal disk path on normal operations |
| SQL (PostgreSQL/MySQL) | ~1 to 20 ms | ~2 to 50 ms | Disk + index traversal + transaction constraints + WAL/fsync overhead |
| NoSQL (MongoDB/Cassandra/Dynamo-style) | ~1 to 15 ms | ~1 to 30 ms | Optimized for partitioned scale; latency depends on replication and consistency level |

These are directional ranges, not guarantees. Actual numbers vary by indexing, query pattern, hardware, and network.

### Why Redis Is Usually Fastest

1. RAM-first design avoids disk seek on hot path
2. Simpler access patterns (key lookup vs relational joins)
3. Lower transactional overhead than full relational engines

### Tradeoffs by Option

**Redis**
- Pros: Lowest latency, excellent for cache/session/rate limiting/counters
- Tradeoffs: RAM cost, limited query flexibility, cache invalidation complexity, durability tuning tradeoffs

**SQL**
- Pros: Strong ACID transactions, joins, constraints, mature ecosystem
- Tradeoffs: Higher latency under heavy scale, harder horizontal write scaling, careful schema/index tuning required

**NoSQL**
- Pros: Horizontal scale, high throughput, flexible models (document/wide-column)
- Tradeoffs: Weaker joins, consistency model complexity (often eventual), partition-key design is critical

### Throughput Intuition

- Redis can handle very high simple key operations per node.
- SQL usually has lower per-node throughput for mixed read/write transactional workloads.
- NoSQL can reach high throughput when partitioning and access patterns are designed well.

### Practical Architecture Rule

In real systems, use them together:
- Redis for hot-path speed
- SQL or NoSQL as durable source of truth

This gives both low latency and reliable persistence.

---

## Interview Q&A

**Q: Why is Redis commonly added in system design interviews?**
A: Because it addresses two hard constraints at once: latency and scale. Redis serves hot data from memory in sub-ms time and offloads databases significantly, which improves both user experience and system stability during traffic spikes.

**Q: What is the biggest tradeoff of using Redis?**
A: Consistency and cost. You gain speed but add cache invalidation complexity and memory cost. If invalidation is wrong, users may see stale data.

**Q: How do you prevent cache stampede in Redis?**
A: Use TTL jitter, request coalescing (cache lock), stale-while-revalidate patterns, and pre-warm important keys.

**Q: Is Redis durable like a database?**
A: It can persist (RDB/AOF), but it is primarily designed for in-memory speed. For critical source-of-truth data, keep a durable primary database.

**Q: What metrics matter most for Redis health?**
A: Cache hit rate, evictions, used memory, latency (especially p99), replication lag, and command stats for hot keys.
