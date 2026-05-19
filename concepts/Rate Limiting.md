# Rate Limiting

## What Is It?

Rate limiting controls how many requests a client can make to an API within a time window. Requests that exceed the limit are rejected with `429 Too Many Requests`.

> **Analogy:** A bouncer at a club lets 10 people in per minute. After that, everyone waits or is turned away.

---

## Why Rate Limit?

| Problem | Solution |
|---------|---------|
| DoS/DDoS attacks | Block flood of requests before they overload servers |
| API abuse | Prevent one user from consuming all capacity |
| Cost control | Stop excessive calls to expensive downstream APIs |
| Fairness | Ensure all users get reasonable share of resources |

---

## The 5 Algorithms

### 1. Token Bucket ⭐ (Most Common)

A bucket holds tokens. Tokens refill at a fixed rate. Each request consumes 1 token.

```
bucket_size = 10    (max burst)
refill_rate = 2/sec (sustained rate)

Request arrives:
  if tokens > 0 → tokens -= 1 → allow
  else → reject 429
```

**Allows bursts** up to `bucket_size`, then enforces `refill_rate` long-term.

**Best for:** Web APIs that need to tolerate short traffic spikes (e.g., mobile app reconnecting after network loss).

**Used by:** Stripe, most REST APIs.

---

### 2. Leaking Bucket

Requests enter a fixed-size FIFO queue. A worker drains requests at a constant rate.

```
requests → [queue max size N] → processed at fixed rate R
           if queue full → drop
```

**No bursts** — output is always smooth and constant.

**Best for:** Traffic shaping where downstream can't handle bursts (e.g., RTB bid requests to ad servers).

---

### 3. Fixed Window Counter

Divide time into fixed 1-minute windows. Each user has a counter per window that resets at window start.

```
window = floor(now / 60)
key = f"user:{id}:win:{window}"
count = INCR(key)
SET_EXPIRE(key, 60)  # only on first write
if count > limit → reject
```

**Problem — boundary spike bug:**

Limit = 5/min. User sends 5 requests at 1:00:59 ✅, then 5 more at 1:01:01 ✅.  
→ 10 requests in 2 seconds. 2× the intended rate!

---

### 4. Sliding Window Log

Store the exact **timestamp** of every request in a sorted set. On each request, remove old timestamps, count remaining.

```
ZREMRANGEBYSCORE key 0 (now - window)
count = ZCARD key
if count >= limit → reject
ZADD key now now
```

**Perfectly accurate** — no boundary spike.  
**Downside:** High memory — stores every request's timestamp. 1M users × 100 req/window = 100M entries.

---

### 5. Sliding Window Counter ⭐ (Best Balance)

Combine two fixed windows using a weighted formula:

```
rate = prev_count × (1 - elapsed/window) + curr_count

Example (limit = 7/min):
  prev_minute count = 5
  current minute: 30 seconds elapsed (50%)
  current count so far = 3
  
  rate = 5 × 0.5 + 3 = 5.5  → under limit, allow
```

**Memory-efficient** (only 2 counters), **nearly accurate** (within ~0.003% error empirically).

---

## Algorithm Comparison

| Algorithm | Memory | Accuracy | Burst OK? | Complexity |
|-----------|--------|----------|-----------|------------|
| Token Bucket | Low | Good | ✅ Yes | Low |
| Leaking Bucket | Low | Good | ❌ No | Low |
| Fixed Window | Low | Medium | Partial | Very Low |
| Sliding Window Log | **High** | **Perfect** | ❌ No | Medium |
| **Sliding Window Counter** | **Low** | **Good** | Partial | Medium |

---

## Architecture

```
Client → [Rate Limiter Middleware]
              ↓               ↓
          [Redis]       [Rules Cache]
              ↓               ↑
         [API Servers]  [Workers pull from Rules DB]
```

**Redis** stores counters: fast (`INCR` = O(1), ~0.1ms), atomic, distributed.

**Rules** (stored in config/DB, cached locally):
```yaml
- endpoint: /api/login
  key: ip
  algorithm: sliding_window_counter
  limit: 10
  window: 60s

- endpoint: /api/post
  key: user_id
  algorithm: token_bucket
  limit: 5
  window: 1s
```

---

## Distributed Race Condition

**Problem:** Two servers both read counter = 4 (limit = 5), both decide to allow, both increment to 5. Two requests passed but combined count is 6.

**Solution: Lua scripts in Redis** (atomic execution — no interleaving):

```lua
local count = redis.call("INCR", KEYS[1])
if count == 1 then
  redis.call("EXPIRE", KEYS[1], ARGV[1])
end
if count > tonumber(ARGV[2]) then
  return 0  -- reject
end
return 1    -- allow
```

The entire script runs atomically on the Redis server — no race condition possible.

---

## Response Headers

Return these on **every** response (not just 429):

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1748475600
Retry-After: 42
```

| Header | Meaning |
|--------|---------|
| `X-RateLimit-Limit` | Total allowed per window |
| `X-RateLimit-Remaining` | Left in current window |
| `X-RateLimit-Reset` | Unix timestamp when window resets |
| `Retry-After` | Seconds until client can retry |

Clients can use `X-RateLimit-Remaining` to **proactively back off** before hitting the limit.

---

## Rate Limit Keys (What to Rate Limit By)

| Key Type | Redis Key | Use Case |
|----------|-----------|---------|
| Per user | `user:{userId}:count` | Authenticated API limits |
| Per IP | `ip:{ip}:count` | Login, registration, unauthenticated endpoints |
| Per API key | `key:{apiKey}:count` | B2B APIs |
| Per endpoint | `user:{id}:endpoint:/login:count` | Different limits per route |
| Global | `global:api:count` | Total API capacity protection |

Combine multiple: one request can consume from both per-user and per-IP buckets simultaneously.

---

## Tiered Rate Limiting

```
Free tier:     100 req/min
Pro tier:    1,000 req/min
Enterprise: 10,000 req/min
```

Look up user tier from cache (short TTL) → apply matching limit. Same algorithm, different limit value.

---

## Fail Behavior

If Redis goes down:
- **Fail-open:** Allow all requests through — API stays up, no rate limiting temporarily
- **Fail-closed:** Reject all requests — rate limiting enforced but service is degraded

Most production systems: **fail-open** + alerting. A short window of unlimited traffic is preferable to a full API outage.

---

## Where Rate Limiters Live

| Location | Pros | Cons |
|----------|------|------|
| **API Gateway** (Kong, AWS API GW) | Centralized, no per-service code | Less flexible |
| **Server middleware** | Fine-grained control | Must implement in every service |
| **Client-side** | ❌ Never — clients can bypass | |

---

## Interview Q&A

**Q: What is a rate limiter and why do we need it?**  
A: Controls how many requests a client can make per time window. Rejects excess with 429. Needed to prevent DoS attacks, API abuse, control costs, and ensure fair usage across all clients.

**Q: Which algorithm would you use in production?**  
A: Sliding window counter for most cases — low memory (2 counters), close to accurate, handles boundary spikes. Token bucket when bursts need to be allowed (mobile apps, batch jobs). Leaking bucket when downstream requires constant smooth rate.

**Q: How do you prevent race conditions in a distributed rate limiter?**  
A: Use Redis Lua scripts. The entire read-check-increment sequence executes atomically on the Redis server. No two scripts execute concurrently, so it's impossible for two requests to both read the same count and both decide to allow.

**Q: What happens if the Redis rate limiter cluster goes down?**  
A: Fail-open (allow all traffic) is preferred for most APIs — brief unprotected window is better than full outage. Fail-closed (reject all) for security-critical endpoints (login, payment). Add circuit breakers and immediate alerting either way.

**Q: How would you support different rate limits for different user tiers?**  
A: Fetch the user's tier (cached in Redis or local memory, short TTL). Apply the limit from a config/rules service corresponding to that tier. The counting algorithm is identical — only the limit threshold differs.
