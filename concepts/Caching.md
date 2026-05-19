# Caching

## What Is It?

A **cache** is a fast storage layer that holds copies of frequently-accessed data so you don't have to re-fetch it from the slow original source (database, external API, disk) every time.

> **Analogy:** A chef keeps the most-used spices on the counter (cache) instead of walking to the pantry (database) every time. Faster cooking, same food.

---

## Why Cache?

| Without Cache | With Cache |
|--------------|-----------|
| Every request hits the DB | ~80-99% of requests served from memory |
| DB becomes bottleneck at scale | DB load drops dramatically |
| ~5-50ms DB read latency | ~0.1ms Redis read latency |
| Can't absorb traffic spikes | Cache absorbs burst reads |

**Cache hit rate** = (cache hits) / (total requests). Target: 80-99%.

---

## What to Cache

✅ **Good candidates:**
- Data that is read often but changes rarely (user profiles, product catalog)
- Expensive computations (recommendation scores, search results)
- Session data (user auth tokens)
- Rate limit counters (see Rate Limiting concept)
- DB query results that are the same for many users

❌ **Bad candidates:**
- Highly personalized data (unique per user per request)
- Financial balances (must always be accurate)
- Data that changes on every write

---

## Cache Strategies (Write Patterns)

### 1. Cache-Aside (Lazy Loading) ⭐ Most Common

Application manages the cache manually.

```
READ:
  data = cache.get(key)
  if data is None:                   # cache miss
    data = db.query(key)             # fetch from DB
    cache.set(key, data, ttl=3600)   # populate cache
  return data

WRITE:
  db.update(key, new_data)
  cache.delete(key)                  # invalidate cache entry
```

**Pros:**
- Only caches data that is actually requested (no wasted memory)
- Cache failure doesn't break the system (just slower)
- Works well for read-heavy workloads

**Cons:**
- First request after cache miss is slow (cache cold start)
- Potential for stale data between DB write and cache invalidation

**Used by:** Most web applications, Twitter, Facebook

---

### 2. Write-Through

Every write goes to cache AND database synchronously.

```
WRITE:
  cache.set(key, new_data)   # write to cache first
  db.update(key, new_data)   # then write to DB
  return success

READ:
  return cache.get(key)      # always in cache (no misses after first write)
```

**Pros:**
- Cache is always fresh — no stale reads
- No cache miss on reads after first write

**Cons:**
- Every write is slower (two writes: cache + DB)
- Cache stores data that may never be read (wasteful)

**Best for:** Systems where read-after-write consistency is critical (banking dashboards)

---

### 3. Write-Behind (Write-Back)

Write to cache immediately, flush to DB asynchronously later.

```
WRITE:
  cache.set(key, new_data)   # write to cache (fast, immediate)
  queue.push(write_job)      # async: flush to DB later

BACKGROUND:
  for job in queue:
    db.update(job.key, job.data)
```

**Pros:**
- Writes are extremely fast (cache only, no DB wait)
- Batches DB writes → reduces DB load

**Cons:**
- **Data loss risk** — if cache crashes before flush, writes are lost
- More complex to implement

**Best for:** High write-throughput scenarios where occasional loss is acceptable (analytics counters, view counts)

---

### 4. Read-Through

Cache sits in front of DB and handles fetching automatically (cache manages itself).

```
READ:
  return cache.get(key)  # cache fetches from DB automatically on miss
```

Similar to cache-aside but the **cache library/service** handles the miss logic, not the application code.

**Used by:** AWS ElastiCache with read-through plugins, some ORM caching layers.

---

## Cache Eviction Policies

When cache is full, which entry to remove?

### LRU — Least Recently Used ⭐ Most Common
Remove the item that hasn't been accessed for the longest time.

```
Access order: A → B → C → A → D (cache full, evict)
Evict: B (least recently used)
```

**Intuition:** If you haven't used it recently, you probably won't need it soon.

### LFU — Least Frequently Used
Remove the item that has been accessed the fewest times.

```
Counts: A=10, B=3, C=7, D=1
Evict: D (accessed only once)
```

**Intuition:** Rarely-used items are unlikely to be needed.

**Problem:** A new item starts with count=1 and gets evicted immediately even if it's about to be heavily used.

### FIFO — First In, First Out
Remove the oldest item regardless of access patterns. Simple but poor cache performance.

### TTL — Time To Live
Every item has an expiry time. Auto-evicted when TTL expires.

```
cache.set("user:123", data, ttl=3600)  # expires in 1 hour
```

Not strictly an eviction policy — more of an expiry mechanism. Usually combined with LRU.

---

## Cache Problems and Solutions

### 1. Cache Stampede (Thundering Herd)

**Problem:** A popular cache key expires. Simultaneously, 1,000 requests all get a cache miss and all go to the DB.

```
t=0:   cache["trending_posts"] expires
t=0.1: 1000 concurrent requests → all miss → all query DB
       DB gets 1000 simultaneous queries → overloaded
```

**Solutions:**

**Mutex/Lock:** First request acquires a lock, fetches from DB, repopulates cache. Other requests wait.
```python
if not cache.get(key):
    with lock(key):
        if not cache.get(key):  # double-check after acquiring lock
            data = db.query(key)
            cache.set(key, data)
```

**Probabilistic Early Expiration:** Randomly expire cache slightly before actual TTL for high-traffic keys — smooths out the stampede.

**Stale-While-Revalidate:** Serve stale data while refreshing in the background.

---

### 2. Cache Penetration

**Problem:** Requests for keys that **don't exist in DB** bypass cache every time (cache can't store "not found").

```
Attacker sends 1M requests for user_id = -1  (doesn't exist)
→ All miss cache (nothing to cache)
→ All hit DB
→ DB overwhelmed
```

**Solutions:**

**Cache null values:** Store `cache.set(key, NULL, ttl=60)` for missing keys.

**Bloom filter:** Before checking cache, check bloom filter of all valid keys. If key not in bloom filter → return 404 immediately.

---

### 3. Cache Avalanche

**Problem:** Many cache keys expire at the **same time** → massive simultaneous DB load.

```
At midnight: 10,000 cached items all expire (same TTL set at midnight yesterday)
→ 10,000 DB queries simultaneously
```

**Solutions:**

**Add random jitter to TTL:**
```python
ttl = 3600 + random.randint(-300, 300)  # 1 hour ± 5 minutes
```

**Pre-warm cache:** Load critical data before it expires via background jobs.

---

### 4. Stale Data

**Problem:** DB updated but cache still has old value.

**Solutions:**
- Short TTL (auto-expires, brief staleness window)
- Cache invalidation on write (`cache.delete(key)`)
- Event-driven invalidation via message queue

---

## Redis vs Memcached

| | Redis | Memcached |
|--|-------|----------|
| Data types | Strings, Lists, Sets, Hashes, Sorted Sets, Streams | Strings only |
| Persistence | ✅ RDB snapshots + AOF log | ❌ Memory only |
| Replication | ✅ Leader-follower | Limited |
| Clustering | ✅ Redis Cluster | ✅ Client-side hashing |
| Pub/Sub | ✅ Yes | ❌ No |
| Lua scripting | ✅ Yes | ❌ No |
| Use case | Rich data structures, persistence needed | Simple key-value, max throughput |

**Choose Redis** in almost all modern systems.  
**Choose Memcached** only when you need maximum raw throughput for simple string values and nothing else.

---

## Cache Tiers in a Typical Architecture

```
User
  ↓
Browser Cache (HTTP Cache-Control headers) ← fastest, closest to user
  ↓ miss
CDN Cache (Cloudflare, CloudFront) ← global edge cache
  ↓ miss
Application Cache (Redis, Memcached) ← shared in-memory cache
  ↓ miss
Database ← slowest, source of truth
```

---

## Cache in System Design Interviews

**Design Twitter feed:** Cache the last 20 tweets for each user in Redis. On timeline load → Redis hit. Background job updates cache when new tweets arrive.

**Design rate limiter:** Store counters in Redis with TTL.

**Design a leaderboard:** Redis Sorted Set — `ZADD leaderboard score userId`. `ZRANGE` for top-K.

**Design autocomplete:** Redis sorted set with prefix-based scores, or cache popular prefix → suggestions mapping.

---

## Interview Q&A

**Q: What is the difference between cache-aside and write-through?**  
A: Cache-aside: application fetches from DB on miss and manually populates cache; writes invalidate cache. Cache is only populated with data that is actually requested. Write-through: every write goes to cache AND DB synchronously, keeping cache always fresh. Cache-aside is more memory-efficient (only hot data cached); write-through ensures no stale reads but caches everything written.

**Q: What is a cache stampede and how do you prevent it?**  
A: When a popular cache key expires, thousands of simultaneous requests all get a miss and hit the DB together. Prevent with: mutex lock (first requester fetches, others wait), probabilistic early expiration (refresh before expiry for hot keys), or stale-while-revalidate (serve stale data while background refresh runs). Adding jitter to TTL prevents mass simultaneous expiry.

**Q: What is cache penetration?**  
A: Requests for keys that don't exist in the DB — every request misses cache because there's nothing to cache. Attackers exploit this to hammer the DB with non-existent keys. Prevent by caching null responses (with short TTL) or using a Bloom filter to reject invalid keys before they reach cache or DB.

**Q: What eviction policy would you use for a general-purpose cache?**  
A: LRU (Least Recently Used) is the most common choice — items not accessed recently are least likely to be needed. Redis uses LRU by default (`maxmemory-policy allkeys-lru`). Use LFU for workloads where access frequency matters more than recency (e.g., some items are accessed in bursts then never again).

**Q: When would you NOT cache data?**  
A: When data must always be perfectly accurate (bank balances, inventory counts for purchases). When data is unique per request and never reused. When the write rate is so high that the cache is invalidated constantly (low hit rate). When data is sensitive and caching on shared infrastructure creates security risks.
