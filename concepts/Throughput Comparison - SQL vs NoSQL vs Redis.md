# Throughput Comparison: SQL vs NoSQL vs Redis (Operations Per Second)

## What Is Throughput?

Throughput is how many operations (reads/writes) a system can handle per second.

For a single node/instance, throughput is the primary metric for deciding:
- Can my database handle current traffic?
- Do I need horizontal scaling?
- Which database is fast enough?

---

## Typical Throughput Ranges

### Redis (In-Memory)

**Single node throughput:**
- Simple GET/SET: 100,000 to 1,000,000+ ops/sec
- INCR/DECR (atomic): 100,000 to 1,000,000+ ops/sec
- More complex ops (sorted set, list operations): 50,000 to 500,000 ops/sec

**Why so fast:**
- Everything is in RAM (no disk I/O)
- Single-threaded event loop (simple serialization)
- No complex query optimization

**Typical real-world:** 200K-500K ops/sec per node under normal load.

---

### SQL Databases (PostgreSQL, MySQL, MariaDB)

**Single node throughput (depends heavily on query complexity):**

**Simple queries (indexed lookup):**
- SELECT by primary key: 10,000 to 100,000 ops/sec
- INSERT (single row, no foreign keys): 5,000 to 50,000 ops/sec
- UPDATE (indexed, no joins): 5,000 to 50,000 ops/sec

**Moderate queries (indexed range scan, simple joins):**
- SELECT with range scan: 1,000 to 10,000 ops/sec
- JOIN on 2 tables: 500 to 5,000 ops/sec

**Complex queries (multiple joins, aggregations):**
- Multi-table JOIN + GROUP BY: 100 to 1,000 ops/sec

**Typical real-world (mixed OLTP):** 10K-50K ops/sec per node.

**Why slower than Redis:**
- Disk I/O for reads/writes (except fully cached working set)
- Query optimization overhead
- ACID transaction coordination
- Lock contention under high concurrency

---

### NoSQL Databases (MongoDB, Cassandra, DynamoDB, etc.)

Throughput varies widely by database design and consistency model.

**MongoDB (single node):**
- Simple document INSERT/find by _id: 10,000 to 100,000 ops/sec
- Queries with index: 5,000 to 50,000 ops/sec
- Queries without index: 100 to 1,000 ops/sec

**Cassandra (distributed by design):**
- Write optimized: 50,000 to 500,000+ writes/sec per node (partition-aware)
- Read optimized: 10,000 to 100,000 reads/sec per node
- Scales horizontally very well

**DynamoDB (AWS managed):**
- Provisioned: user specifies ops/sec (e.g., 100 WCU = 400 writes/sec)
- On-demand: auto-scales, typically 40K-100K+ ops/sec per partition

**Typical real-world (distributed NoSQL):** 50K-500K writes/sec (total cluster), 10K-100K reads/sec.

**Why variable:**
- Some NoSQL databases (Cassandra) optimize for writes
- Others optimize for reads (Memcached, Redis)
- Partition key distribution matters (hot partition can bottleneck)
- Replication/consistency model affects throughput

---

## Practical Numbers: Small Example

Assume 1 million reads/sec total traffic across your system.

**Using Redis alone:**
- Need ~2-5 Redis nodes (at 200K-500K ops/sec each)
- Cost: moderate (RAM is expensive)

**Using SQL (PostgreSQL):**
- Need ~20-100 SQL nodes (at 10K-50K ops/sec each)
- Cost: very high (DB licensing, management complexity)
- Better approach: cache layer (Redis) in front of SQL

**Using NoSQL (Cassandra):**
- Need ~2-20 Cassandra nodes (at 50K-500K ops/sec each)
- Cost: moderate to high (depends on cluster size)
- Good for write-heavy workloads

**Recommended hybrid:**
- Redis for hot reads (200K ops/sec per node) → 5 nodes
- SQL or NoSQL for durable storage
- Total: 5 Redis + 3 DB nodes = manageable scale

---

## Throughput vs Latency

Important distinction:
- **Throughput:** total ops/sec the system can handle
- **Latency:** time for one operation

Both matter:
- High throughput but high latency: system is overloaded
- High throughput and low latency: ideal

Example:
```
Redis:
  Throughput: 500K ops/sec
  Latency: 1ms p99

SQL:
  Throughput: 10K ops/sec
  Latency: 10ms p99

Under 10K ops/sec load:
  Redis: stays at 1ms latency (healthy)
  SQL: stays at 10ms latency (acceptable)

Under 100K ops/sec load:
  Redis: still ~1-2ms latency (handles it)
  SQL: queues build, latency spikes to 100ms+ (degraded)
```

---

## Throughput Limiting Factors

### For SQL:
1. Disk I/O (WAL fsync, dirty page flushes)
2. Lock contention (same row updates)
3. Query complexity (joins, aggregations)
4. Connection pool size
5. Memory for caching working set

### For NoSQL (e.g., Cassandra):
1. Network replication factor (consistency level)
2. Partition hotspots (uneven distribution)
3. Compaction background tasks (GC pauses)
4. Rebalancing during node failures

### For Redis:
1. Network I/O (though very fast)
2. Memory limits (eviction policy kicks in)
3. Persistence settings (RDB/AOF can slow writes)
4. Lua script complexity (blocks event loop)
5. Single-threaded writes (for one key)

---

## Real-World Scaling Scenarios

### Scenario 1: 100K reads/sec, read-heavy

**Option A: SQL + Redis**
- Redis: 3 nodes for cache
- SQL: 1 primary + 2 read replicas (20K reads/sec each)
- Cost: low to moderate
- Works well (reads mostly from cache)

**Option B: NoSQL alone**
- Cassandra: 3 nodes (100K reads/sec total)
- Cost: low to moderate
- Works, but overkill for read-heavy

### Scenario 2: 100K writes/sec, write-heavy

**Option A: SQL**
- Needs ~10-20 nodes just for throughput
- Cost: very high
- Usually avoids for this scale

**Option B: NoSQL (Cassandra)**
- Cassandra: 3-5 nodes (100K+ writes/sec)
- Cost: moderate
- Scales well

**Option C: NoSQL + Redis**
- Redis: 2 nodes for write buffering
- Cassandra: 3 nodes for durable storage
- Cost: moderate
- Very fast write acceptance in Redis, eventual durability in Cassandra

### Scenario 3: 1M ops/sec (mixed)

**Only viable approach:**
- Redis cluster (10 nodes, 100K ops/sec each)
- Cassandra cluster (10 nodes, 100K writes/sec each)
- Both layers, not one alone

---

## Throughput Comparison Table

| Database | Single Node Throughput | Typical Use | Scales To | Cost |
|----------|------------------------|-------------|-----------|------|
| Redis | 200K-1M ops/sec | Cache, rate limit, session | 10M+ (cluster) | High (RAM) |
| PostgreSQL | 10K-50K ops/sec | OLTP, transactional | 100K (replicated) | Moderate |
| MySQL | 5K-50K ops/sec | General OLTP | 100K (replicated) | Low |
| Cassandra | 50K-500K ops/sec | Time-series, logs, high writes | 1M+ (cluster) | Moderate |
| MongoDB | 10K-100K ops/sec | Document store, flexible schema | 500K (cluster) | Moderate |
| DynamoDB (AWS) | 40K-100K ops/sec (per partition) | Managed, variable | 1M+ (auto) | High (per req) |

---

## Interview Q&A

**Q: How many operations per second can a single PostgreSQL node handle?**
A: Typically 10K-50K ops/sec, depending on query complexity and hardware. Simple indexed lookups can reach 50K+; complex joins drop to 1K-5K ops/sec. This is why caching (Redis) is critical for high-scale systems.

**Q: If I need 1 million reads per second, how many database nodes do I need?**
A: Depends on the database:
- SQL: 20-100 nodes (very expensive)
- Redis: 2-5 nodes (moderate cost)
- Cassandra: 2-20 nodes (moderate cost)

The answer is usually "use Redis in front of your primary database" rather than scaling the database itself.

**Q: Why is Redis so much faster than SQL?**
A: In-memory operations are orders of magnitude faster than disk I/O. Redis also avoids query optimization overhead and lock contention. But Redis data must fit in RAM, which is expensive.

**Q: Does adding read replicas increase my write throughput?**
A: No. Replicas help with read throughput and availability, but writes still go to the primary. Write throughput is bottlenecked by the primary's disk I/O and lock contention, not replica count.

**Q: What is the difference between throughput and latency?**
A: Throughput is total ops/sec the system handles. Latency is time per operation. High throughput doesn't guarantee low latency; an overloaded system can have high throughput with high latency due to queuing.
