# Sharding Strategies

## What Is Sharding?

**Sharding** (also called horizontal partitioning) splits a large database across multiple machines, where each machine holds a **subset of the data**.

> **Analogy:** A library with 1 million books. Instead of one giant building, you split by genre: building A has fiction, building B has non-fiction, building C has science. Each building is a shard.

**Why shard?**
- A single DB server can't hold 10 TB of data
- A single DB server can't handle 100,000 writes/second
- Sharding splits both storage and traffic across many machines

---

## Vertical vs Horizontal Scaling

**Vertical Scaling (Scale Up):** Add more CPU/RAM/disk to one machine.
- Simple, no code changes
- Has physical limits (~$50K for max server)
- Single point of failure

**Horizontal Scaling (Scale Out) = Sharding:** Add more machines.
- Theoretically unlimited
- More complex (routing, transactions, joins)
- No single point of failure

---

## Sharding vs Partitioning

| Term | Meaning |
|------|---------|
| **Partitioning** | Splitting data within a single DB instance (e.g., MySQL partition by date range) |
| **Sharding** | Splitting data across multiple separate DB instances/machines |

Sharding is distributed partitioning.

---

## The Shard Key

The **shard key** is the column(s) used to determine which shard a row goes to.

Choosing the right shard key is the most critical sharding decision. A bad shard key causes:
- **Hotspots** — one shard gets most traffic
- **Scatter-gather queries** — queries hit all shards
- **Cross-shard transactions** — complex, slow, or impossible

---

## Sharding Strategies

### 1. Range-Based Sharding

Split data by a value range of the shard key.

```
Shard 0: userId 1–1,000,000
Shard 1: userId 1,000,001–2,000,000
Shard 2: userId 2,000,001–3,000,000
```

**Pros:**
- Range queries are fast — scan one or a few shards
- Easy to add new shards for new ranges

**Cons:**
- **Hotspot risk** — recent data (new users, new orders) all goes to the last shard
- Uneven load distribution if range traffic isn't uniform

**Best for:** Time-series data with range queries (logs, events: "find all events in Jan 2026")

---

### 2. Hash-Based Sharding ⭐ (Most Common)

```
shardIndex = hash(shardKey) % numShards
```

```
User 42:  hash(42) % 4 = 2  → Shard 2
User 99:  hash(99) % 4 = 3  → Shard 3
User 103: hash(103) % 4 = 0 → Shard 0
```

**Pros:**
- Even distribution — no hotspots (assuming uniform key distribution)
- Simple to implement

**Cons:**
- Adding/removing shards remaps most keys (use consistent hashing to fix this)
- Range queries are slow — must hit all shards and merge

**Best for:** Even load distribution, key-based lookups (user by ID, session by token)

---

### 3. Consistent Hashing (Hash Sharding + No Remapping)

Use a hash ring instead of `hash % N`. Adding a shard only moves ~1/N of keys.

```
Ring: shard0 ... shard1 ... shard2 ... shard3
key → walk clockwise → first shard encountered
```

Used by: **Cassandra, DynamoDB, Redis Cluster**

**Best for:** Systems that frequently add/remove shards (cloud environments, elastic scaling)

---

### 4. Directory-Based Sharding

A **lookup service** maps each key to a shard.

```
Lookup table:
  userId 42 → Shard 2
  userId 99 → Shard 1
  userId 103 → Shard 3
```

**Pros:**
- Maximum flexibility — move any key to any shard
- Easy to handle special cases (VIP users on dedicated shards)

**Cons:**
- Lookup service is a single point of failure (must replicate it)
- Extra network hop for every request
- Lookup table must be updated when data moves

**Best for:** Complex partitioning needs, gradual data migration

---

### 5. Geographic Sharding

Route data based on user's location.

```
Shard US-East  → US East users
Shard EU-West  → European users
Shard APAC     → Asia-Pacific users
```

**Pros:**
- Low latency — data lives near the users who access it
- Regulatory compliance (GDPR: EU data stays in EU)

**Cons:**
- Uneven shard sizes if user distribution is uneven
- Cross-region queries (global analytics) are expensive

**Best for:** Global applications with data residency requirements (WhatsApp, Twitter)

---

## Hotspot Problem

Even with good sharding, some keys get disproportionate traffic:
- Celebrity user with 100M followers — all activity on one shard
- Viral post being liked by millions — all writes to one shard

**Solutions:**
1. **Add a random suffix** — split the hot key across multiple sub-shards
   ```
   hot_key_0, hot_key_1, hot_key_2  → 3 shards
   Read: query all 3, merge results
   ```
2. **Cache** — put Redis in front of the hot data
3. **Dedicated shard** — move the hot entity to its own shard

---

## Cross-Shard Operations

### Cross-Shard Queries
Query involves data on multiple shards.

```sql
SELECT * FROM users WHERE age > 25
```

With hash sharding: users with age > 25 are on all shards.

**Solution:** Scatter-gather
1. Send query to all shards in parallel
2. Each shard returns its matching rows
3. Coordinator merges and returns final result

**Cost:** Latency × number of shards. Minimize by designing queries to be shard-local.

### Cross-Shard Transactions
Transferring $100 from user on Shard A to user on Shard B.

**Options:**
1. **2-Phase Commit (2PC)** — atomic but complex, high latency, can deadlock
2. **Saga pattern** — series of local transactions with compensating transactions on failure
3. **Redesign** — avoid cross-shard transactions by collocating related data on the same shard

---

## Shard Key Selection Guide

| Use Case | Good Shard Key | Why |
|----------|---------------|-----|
| Social app (tweets, posts) | `user_id` | User's data on one shard; most queries are per-user |
| E-commerce (orders) | `user_id` | Order history per-user is shard-local |
| Multi-tenant SaaS | `tenant_id` | All tenant data on one shard; isolation |
| Hotels/reservations | `hotel_id` | All inventory for one hotel on one shard |
| Chat messages | `channel_id` | All messages in a conversation on one shard |
| Time-series logs | `timestamp` (range) | Range queries efficient; watch for hotspot on recent shard |

---

## Resharding (Adding More Shards)

When a shard grows too large or gets too much traffic:

1. **Split a shard** — divide its data range into 2 new shards
2. **Migrate data** — background job copies data to new shards
3. **Update routing** — once migration is complete, route new writes to new shards
4. **Decommission old shard**

**Key challenge:** Zero-downtime resharding requires dual-writes (write to old and new shard), then cut over.

**Consistent hashing** minimizes resharding cost by only moving ~K/N keys when adding 1 shard of N.

---

## Interview Q&A

**Q: What is sharding and when do you use it?**  
A: Splitting a database horizontally across multiple machines where each machine holds a subset of rows. Use it when a single machine can't handle storage or traffic: typically at hundreds of GBs or thousands of writes/sec. Before sharding, try vertical scaling, query optimization, read replicas, and caching.

**Q: What is a shard key and why does it matter?**  
A: The column used to route rows to shards. Bad shard key → hotspots (one shard overloaded) or scatter-gather (every query hits all shards). Good shard key: high cardinality, even access distribution, aligns with most common query pattern. Often `user_id` or `tenant_id`.

**Q: What is the difference between range-based and hash-based sharding?**  
A: Range: routes by value ranges (users 1–1M → shard 0). Fast range queries but risks hotspots on recent data. Hash: routes by `hash(key) % N`. Even distribution, no hotspots, but range queries hit all shards. Most production systems use hash-based or consistent hashing.

**Q: How do you handle a hotspot shard?**  
A: Add a random suffix to the hot key to spread it across multiple sub-shards (reads merge results). Put Redis cache in front of hot data. Or move the hot entity to a dedicated shard. The root cause is usually a celebrity/viral entity — design for it upfront if your domain has such patterns.

**Q: How do you handle cross-shard transactions?**  
A: Avoid them by design — collocate related data on the same shard. When unavoidable: use Saga pattern (chain of local transactions with compensating rollbacks on failure) or 2-Phase Commit (atomic but complex and has availability trade-offs). Most large-scale systems avoid strict cross-shard ACID and rely on eventual consistency instead.
