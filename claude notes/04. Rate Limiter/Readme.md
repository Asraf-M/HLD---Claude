# System Design: Rate Limiter (Beginner-Friendly Guide)

---

## What Are We Building?

A **rate limiter** is a bouncer at a club door. It lets people in — but only at a certain pace. If too many people rush at once, the bouncer turns them away.

In software terms: when a client sends too many API requests in a short time, the rate limiter blocks the extras and returns a `429 Too Many Requests` response.

<img src="../../system-design-notes/04. Rate Limiter/images/rate_limiter_architecture.png" alt="Rate Limiter Architecture" width="500">

**Why do we need rate limiting?**

| Problem | How Rate Limiting Helps |
|---------|------------------------|
| DoS/DDoS attacks | Block request floods before they overload servers |
| Expensive API abuse | A user sending 10,000 requests/sec costs real money |
| Fair usage | Prevent one user from hogging all capacity |
| Stability | Protect downstream services from traffic spikes |

**Real-world examples:**
- Twitter: 300 reads per 15 minutes per OAuth app
- GitHub: 5,000 requests per hour for authenticated users
- Google Maps API: daily quota + per-second burst limit
- Stripe: token bucket per API key, tiered by account type

---

## Step 1: Requirements

**Functional:**
- Limit the number of requests a client can make per time window
- Support different rules: per-user, per-IP, per-API-key, per-endpoint
- Inform the client when they''ve been rate limited

**Non-functional:**
- Low latency (can''t add 100ms to every request)
- Low memory (millions of users → can''t store huge data per user)
- Distributed (works across multiple servers)
- High fault tolerance (if rate limiter fails, system should degrade gracefully)

---

## Step 2: Where Should the Rate Limiter Live?

**Option 1: Client-side** — Bad idea. Clients can bypass or forge requests.

**Option 2: Server-side** — Good. The server controls the logic.

**Option 3: Middleware / API Gateway** — Best for microservices. One place enforces limits for all services. Products like Kong, AWS API Gateway, NGINX Plus have built-in rate limiting plugins.

> **Rule of thumb:** If you have microservices, put the rate limiter in the API gateway. If you have a monolith or small service, handle it in server-side middleware.

---

## Step 3: The 5 Rate Limiting Algorithms

### Algorithm 1: Token Bucket

<img src="../../system-design-notes/04. Rate Limiter/images/token-bucket.png" alt="Token Bucket Algorithm" width="480">

**Analogy:** Imagine a bucket that holds coins (tokens). A refiller drops coins in at a fixed rate. Each request costs 1 coin. If there are coins → request goes through. If bucket is empty → request is dropped.

**Parameters:**
- `bucket_size` — max tokens the bucket can hold (burst capacity)
- `refill_rate` — how many tokens added per second

**Example:** bucket_size=10, refill_rate=2/sec
- You can fire 10 requests instantly (burst)
- After that, only 2 requests/sec can go through (sustained rate)

```
New request arrives:
  if tokens > 0:
    tokens -= 1
    allow request
  else:
    reject with 429
```

**Pros:** Simple, memory-efficient (O(1) per user), allows bursts  
**Cons:** Need to tune two parameters carefully

**Best for:** Web APIs that want to allow occasional traffic spikes (e.g., mobile app syncing after reconnection)

---

### Algorithm 2: Leaking Bucket

<img src="../../system-design-notes/04. Rate Limiter/images/leaking-bucket.png" alt="Leaking Bucket Algorithm" width="480">

**Analogy:** Requests pour into a bucket with a small hole in the bottom. Water (requests) drains out at a fixed rate. If the bucket overflows → requests are dropped.

**Implementation:** A FIFO queue. Requests join the queue. A worker processes them at a fixed rate.

**Pros:** Smooth, constant output rate — great for traffic shaping  
**Cons:** A burst fills the queue; later requests are delayed even if the burst was long ago

**Best for:** Scenarios needing constant output rate (e.g., video streaming, RTB bid requests)

> **Token bucket vs Leaking bucket:** Token bucket allows bursts; leaking bucket smooths everything into a constant flow.

---

### Algorithm 3: Fixed Window Counter

<img src="../../system-design-notes/04. Rate Limiter/images/fixed-window-counter.png" alt="Fixed Window Counter" width="480">

**Analogy:** A bouncer resets their clicker every minute. You get 5 entries per minute. Simple.

**How it works:** Divide time into fixed 1-minute windows. Each user has a counter per window. Counter resets at the start of each window.

```
window = floor(current_time / window_size)
key = "user:{userId}:window:{window}"
count = INCR key
if count == 1: SET EXPIRE key window_size
if count > limit: reject
```

**Pros:** Very simple, memory-efficient  
**Cons:** Has a critical edge case bug ↓

**The boundary spike problem:**

<img src="../../system-design-notes/04. Rate Limiter/images/fixed-window-issue.png" alt="Fixed Window Issue" width="480">

If limit = 5 requests/minute:
- User sends 5 requests at 1:00:59 (end of minute 1) ✅ all pass
- User sends 5 requests at 1:01:01 (start of minute 2) ✅ all pass
- **In 2 seconds, 10 requests went through** — 2× the intended rate!

---

### Algorithm 4: Sliding Window Log

<img src="../../system-design-notes/04. Rate Limiter/images/sliding-window-log.png" alt="Sliding Window Log" width="480">

**Analogy:** Instead of a clicker that resets every minute, keep a running list of the exact timestamps of every request. Count only requests in the last 60 seconds.

**How it works:**
- Store all request timestamps in a sorted set (Redis sorted set, score = timestamp)
- On new request: remove timestamps older than `now - window`; count remaining; reject if over limit

```
ZREMRANGEBYSCORE key 0 (now - 60_seconds)
count = ZCARD key
if count >= limit: reject
ZADD key now now
```

**Example (limit = 2 per minute):**

| Time | Action | Log | Result |
|------|--------|-----|--------|
| 1:00:01 | Request | [1:00:01] | ✅ Allow (count=1) |
| 1:00:30 | Request | [1:00:01, 1:00:30] | ✅ Allow (count=2) |
| 1:00:50 | Request | [1:00:01, 1:00:30, 1:00:50] | ❌ Reject (count=3) |
| 1:01:40 | Request | [1:00:50, 1:01:40] (1:00:01 and 1:00:30 expired) | ✅ Allow (count=2) |

**Pros:** Perfectly accurate, no boundary spike problem  
**Cons:** High memory — stores every request timestamp. At 1M users × 100 requests/window = 100M entries

---

### Algorithm 5: Sliding Window Counter

<img src="../../system-design-notes/04. Rate Limiter/images/sliding-window-counter.png" alt="Sliding Window Counter" width="480">

**The best of both worlds** — memory-efficient like Fixed Window, accurate like Sliding Window Log.

**How it works:** Use two fixed windows (current + previous) and compute a weighted average:

```
requests_in_rolling_window = 
    (previous_window_count × overlap_with_previous) 
  + current_window_count

overlap = 1 - (seconds_into_current_window / window_size)
```

**Example (limit = 7 per minute):**
- Previous minute: 5 requests
- Current minute: 3 requests so far
- We''re 30% into the current minute (30 seconds elapsed)
- Estimated count = 5 × 70% + 3 = 3.5 + 3 = **6.5** → allow

**Pros:** Memory-efficient (only 2 counters per user), smooths out boundary spikes  
**Cons:** Approximate (assumes requests are uniformly distributed in the window) — not perfectly accurate but very close

---

## Algorithm Comparison

| Algorithm | Memory | Accuracy | Burst-friendly | Complexity |
|-----------|--------|----------|----------------|------------|
| Token Bucket | Low | Good | ✅ Yes | Low |
| Leaking Bucket | Low | Good | ❌ No (queued) | Low |
| Fixed Window Counter | Low | Medium | Partially | Very Low |
| Sliding Window Log | **High** | **Perfect** | ❌ No | Medium |
| Sliding Window Counter | Low | Good (approx) | Partially | Medium |

> **Interview tip:** Sliding Window Counter is the most commonly recommended for production — it balances all trade-offs.

---

## Step 4: High-Level Architecture

<img src="../../system-design-notes/04. Rate Limiter/images/architecture.png" alt="Architecture" width="500">

```
Client → Rate Limiter Middleware
              ↓ checks counter in
           [Redis]        [Cached Rules]
              ↓                ↑
         [API Servers]    [Workers fetch rules from Rules DB]
```

**Flow:**
1. Client sends request to the rate limiter middleware
2. Middleware reads the **rules** from local cache (rules define: which endpoint, which algorithm, what limit)
3. Middleware checks the **counter in Redis** for this user/endpoint
4. If under limit → forward to API servers; if over limit → return 429

**What goes in Redis?**
- For token bucket: `user:{id}:tokens` (current token count) + `user:{id}:last_refill` (timestamp)
- For fixed window: `user:{id}:window:{window_id}` (count with auto-expiry TTL)
- For sliding window log: `user:{id}` sorted set of timestamps

**What goes in the Rules config?**
```yaml
- key_type: user_id
  resource: /api/post
  algorithm: token_bucket
  limit: 5
  window: 1s

- key_type: ip
  resource: /api/login
  algorithm: sliding_window_counter
  limit: 10
  window: 60s
```

Workers periodically pull fresh rules from the rules database → update cache. The middleware always reads from cache (never hits the DB on the hot path).

---

## Step 5: Response Headers

When a client is rate limited, return:

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1748475600
Retry-After: 42
```

| Header | Meaning |
|--------|---------|
| `X-RateLimit-Limit` | Total allowed requests per window |
| `X-RateLimit-Remaining` | Requests left in current window |
| `X-RateLimit-Reset` | Unix timestamp when the window resets |
| `Retry-After` | Seconds until the client should retry |

> **Good practice:** Include `X-RateLimit-*` headers on **every** response (not just 429s). This lets clients proactively back off before hitting the limit.

---

## Step 6: Distributed Rate Limiting Challenges

### Problem: Race Conditions

Two API server instances both read counter = 4 (limit = 5) at the same time. Both think they''re within the limit. Both increment to 5. Both pass the request. But now 6 requests went through!

**Solution: Atomic Redis operations**

Use Lua scripts in Redis (executed atomically — no two scripts run at the same time):

```lua
local count = redis.call("INCR", KEYS[1])
if count == 1 then
  redis.call("EXPIRE", KEYS[1], ARGV[1])
end
if count > tonumber(ARGV[2]) then
  return 0  -- reject
end
return 1  -- allow
```

Or use Redis `MULTI/EXEC` (transaction block).

### Problem: Synchronization Across Nodes

If each API server has its own in-process counter, a user can send N requests to each of K servers → K×N total requests bypass the limit.

**Solution:** Use Redis as a **shared centralized counter store** — all API servers read/write the same Redis key.

### Performance Optimization

- Redis INCR is ~0.1ms → acceptable on the hot path
- Cache rules locally → never hit the rules DB per request
- Redis Cluster for horizontal scaling of the counter store
- Multi-datacenter: replicate Redis across regions with eventual consistency

---

## Step 7: What Happens When Rate Limiting Fails?

If Redis goes down or the rate limiter crashes:

**Fail-open:** Allow all requests through (prioritize availability)
- Risk: Abuse could spike during the outage window
- Good for: User-facing APIs where availability matters more

**Fail-closed:** Reject all requests
- Risk: Full service outage for legitimate users
- Good for: Financial APIs or critical security endpoints

> Most systems choose **fail-open** with alerting — the rare Redis outage is less damaging than a full API blackout. Add circuit breakers (Resilience4j, Hystrix) to detect the failure quickly.

---

## Summary: The "Why" Behind Each Decision

| Decision | Why? |
|----------|------|
| Redis for counters (not DB) | Redis INCR is O(1), sub-millisecond; SQL DB would add 10–50ms per request |
| Lua scripts for atomicity | Prevents race conditions between read-check-increment steps |
| Cache rules locally | Rules rarely change; reading from DB on every request would be a bottleneck |
| Middleware / API Gateway | Centralizes enforcement; individual services don''t need to implement it |
| 429 + Retry-After header | Tells clients exactly when to retry; prevents unnecessary immediate retries |
| Sliding Window Counter | Best balance of memory efficiency and accuracy for most production use cases |
| Token Bucket for burst-tolerant APIs | Allows legitimate spikes (mobile app reconnecting) while enforcing long-term rate |
| Leaking Bucket for smooth output | Shapes traffic into constant rate for upstream services that can''t handle bursts |
| Fail-open on Redis failure | Availability > strict rate enforcement during rare infrastructure failures |

---

## Quick Interview Q&A Cheat Sheet

**Q: What are the main rate limiting algorithms and their trade-offs?**  
A: (1) **Token bucket** — allows bursts up to bucket size, then steady rate; memory-efficient. (2) **Leaking bucket** — constant output rate via FIFO queue; no bursts. (3) **Fixed window counter** — simple but has boundary spike bug (2× rate possible at window edges). (4) **Sliding window log** — perfectly accurate but stores all timestamps (high memory). (5) **Sliding window counter** — weighted approximation using two fixed windows; best balance of accuracy + memory.

**Q: How do you handle race conditions in a distributed rate limiter?**  
A: Use atomic Redis operations — specifically Lua scripts that execute as one unit, or Redis MULTI/EXEC transactions. This prevents the read-check-increment race where two concurrent requests both read the same counter, both pass the check, and both increment — allowing more requests than the limit.

**Q: What HTTP status code and headers do you return when rate limiting?**  
A: `429 Too Many Requests`. Include: `X-RateLimit-Limit` (total allowed), `X-RateLimit-Remaining` (left in window), `X-RateLimit-Reset` (when window resets), `Retry-After` (seconds to wait). Return these headers on every response (not just 429) so clients can proactively back off.

**Q: Why use Redis instead of an in-memory counter on each server?**  
A: In-memory counters are per-server — a user could send N requests to each of K servers for K×N total, bypassing the intended limit N. Redis is a centralized shared store — all servers read/write the same counter, ensuring consistent enforcement regardless of which server handles a request.

**Q: What happens if the rate limiter or Redis goes down?**  
A: Two options: **fail-open** (allow all requests through — keeps the API available but risks abuse during the outage), or **fail-closed** (reject all requests — protects the backend but causes downtime). Most production systems use fail-open with circuit breakers and immediate alerting.

**Q: Where should you place the rate limiter?**  
A: For microservices: API gateway middleware (Kong, AWS API Gateway, NGINX) — centralizes enforcement across all services without each service needing to implement it. For monoliths: server-side middleware. Never client-side — clients can bypass it.

**Q: How do you support different limits for different user tiers (free vs premium)?**  
A: Store rate limit config per tier (e.g., free = 100/min, premium = 10,000/min). The rate limiter middleware looks up the user''s tier (cached by user ID with a short TTL) and applies the corresponding limit. The counting algorithm is the same — only the limit value differs.
