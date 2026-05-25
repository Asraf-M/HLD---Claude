# Frequently Asked System Design Questions

---

## Tier 1 — Asked Everywhere (Must Know)

| # | Question | Key Concepts |
|---|---|---|
| 1 | **Design URL Shortener** (Bitly) | Hashing, redirects, analytics |
| 2 | **Design Twitter/X Feed** (News Feed) | Fanout, timelines, ranking |
| 3 | **Design WhatsApp / Chat System** | WebSocket, message storage, delivery receipts |
| 4 | **Design YouTube / Netflix** | Video upload pipeline, CDN, adaptive bitrate |
| 5 | **Design Rate Limiter** | Token bucket, sliding window, distributed counters |
| 6 | **Design Key-Value Store** | Consistent hashing, replication, CAP theorem |
| 7 | **Design Notification System** | Push/SMS/email, fan-out, retry |
| 8 | **Design Unique ID Generator** | Snowflake ID, clock skew |

---

## Tier 2 — Common at FAANG

| # | Question | Key Concepts |
|---|---|---|
| 9 | **Design Google Drive / Dropbox** | File chunking, sync, conflict resolution |
| 10 | **Design Search Autocomplete / Typeahead** | Trie, Elasticsearch, ranking |
| 11 | **Design Uber / Lyft** | Geohash, matching, surge pricing |
| 12 | **Design Instagram / Photo Sharing** | Upload pipeline, CDN, feed |
| 13 | **Design Web Crawler** | BFS, politeness, deduplication |
| 14 | **Design Distributed Message Queue** (Kafka) | Partitions, consumer groups, offsets |
| 15 | **Design Nearby Friends / Proximity Service** | Geohash, Redis Pub/Sub, WebSocket |
| 16 | **Design Google Maps** | Routing graph, ETA, tile serving |
| 17 | **Design Ad Click Aggregation** | Stream processing, exactly-once, aggregation |
| 18 | **Design Google Docs** (collaborative editing) | OT/CRDT, real-time sync, WebSocket, conflict resolution |
| 19 | **Design a Leaderboard** | Redis sorted sets (ZADD/ZRANK), top-K, gaming companies |
| 20 | **Design Distributed Lock** | Redis Redlock, Zookeeper, fencing tokens, TTL |
| 21 | **Design Food Delivery** (Swiggy/DoorDash) | Geo + order matching, driver assignment, real-time tracking |
| 22 | **Design Stock Exchange / Trading System** | Order book, matching engine, exactly-once, low latency |
| 23 | **Design Video Conferencing** (Zoom) | WebRTC, SFU vs MCU, signalling server, adaptive bitrate |

---

## Tier 3 — Asked at Mid-Large Companies

| # | Question | Key Concepts |
|---|---|---|
| 18 | **Design Hotel / Ticket Reservation System** | Distributed locking, ACID, overbooking |
| 19 | **Design Distributed Email Service** (Gmail) | SMTP, threading, search |
| 20 | **Design S3-like Object Storage** | Consistent hashing, erasure coding, metadata |
| 21 | **Design Metrics Monitoring System** (Datadog/Prometheus) | Time-series DB, alerting |
| 22 | **Design Facebook Comments** | Nested threads, like count, eventual consistency |
| 23 | **Design Distributed Cache** (Redis cluster) | Eviction policies, replication, consistency |
| 24 | **Design Payment System** (Stripe) | Idempotency, double-spend prevention, ledger |
| 25 | **Design Live Streaming** (Twitch) | RTMP ingest, transcoding, HLS delivery |

---

## Company-Specific Patterns

| Company | Favourite Topics |
|---|---|
| **Google** | Maps, Search, Bigtable-style storage, distributed consensus |
| **Meta/Facebook** | News Feed, Chat, Social Graph, Notifications |
| **Amazon** | S3, DynamoDB-style KV store, Reservations, Payment |
| **Microsoft** | Teams (Chat + Video), OneDrive, Azure services |
| **Uber/Lyft** | Geo systems, Matching, Surge, Trip tracking |
| **LinkedIn** | Search, Feed, Connection graph, Notifications |
| **Twitter/X** | Feed fanout (celebrity problem), Trending, Search |
| **Airbnb** | Search (Elasticsearch), Reservations, Pricing |
| **Netflix** | Video pipeline, CDN, Recommendation, A/B testing |
| **Stripe/PayPal** | Payment, Idempotency, Ledger, Fraud detection |

---

## Most Common Follow-Up Questions (Know These Deep)

Every system design interview will probe at least 3-4 of these regardless of the question asked:

| Follow-Up | What They Want to Hear |
|---|---|
| **How do you handle hotspots?** | Random suffix, dedicated shard, caching in front of hot key |
| **How do you scale from 1M to 100M users?** | Horizontal scaling, sharding, read replicas, CDN, caching layer |
| **How do you ensure exactly-once delivery?** | Idempotency keys, deduplication at consumer, outbox pattern |
| **SQL vs NoSQL — justify your choice** | Access pattern, scale, consistency requirements, schema flexibility |
| **How do you handle failure / retry?** | Exponential backoff, dead letter queue, idempotent operations |
| **Where do you add a cache?** | In front of DB for read-heavy, Redis with TTL, cache-aside pattern |
| **What eviction policy?** | LRU for general, LFU for skewed access, TTL for session/rate limit data |
| **How do you monitor the system?** | Metrics (Prometheus), logs (ELK), traces (Jaeger), alerting thresholds |
| **How do you handle the celebrity problem?** | Pre-computed fanout, pull model for celebrities, random read replica |
| **How do you prevent data loss?** | WAL, replication factor ≥ 3, async vs sync replication trade-off |

---

## Interview Framework — How to Structure Your 45 Minutes

Interviewers penalise candidates who jump straight to drawing boxes. Always follow this structure:

### Step 1: Clarify Requirements (5 min)
- **Functional requirements** — what the system must do
  - "Users can post tweets, follow others, see a feed" 
- **Non-functional requirements** — how well it must do it
  - Scale, latency, availability, consistency (see NFR section below)
- **Out of scope** — explicitly state what you are NOT designing
  - "I will not design the ads system or recommendation algorithm"
- **Ask before assuming:** DAU, read/write ratio, data retention, global vs single-region

### Step 2: Back-of-Envelope Estimation (5 min)
- DAU → QPS (reads and writes separately)
- Storage per day / per year
- Bandwidth (read throughput)
- Decide if you need sharding, caching, CDN based on numbers

```
Example: Twitter
  300M DAU, each views 100 tweets/day
  Read QPS  = 300M × 100 / 86400 ≈ 350,000 reads/sec
  Write QPS = 300M × 1 tweet / 86400 ≈ 3,500 writes/sec  (100:1 read/write ratio)
  Tweet size = 280 chars + metadata ≈ 1KB
  Storage/day = 3,500 × 1KB × 86400 ≈ 300GB/day
```

### Step 3: High-Level Design (10 min)
- Draw the core components: clients, load balancer, app servers, DB, cache, CDN, queue
- Show the happy path — one read request and one write request end-to-end
- Label which DB you are using and why

### Step 4: Deep Dive (20 min)
- Interviewer will pick 1-2 components to go deep on
- Common deep dives: DB schema, fanout mechanism, caching strategy, failure handling
- Proactively say: "The tricky part here is X, let me explain how I'd handle it"

### Step 5: Wrap Up (5 min)
- Bottlenecks in your design and how you'd fix them
- What you'd monitor (metrics, alerts)
- What you'd do differently at 10× the scale
- Single points of failure and how to eliminate them

---

## Non-Functional Requirements — Always Define These First

Before drawing anything, define these. Interviewers expect you to drive this.

| NFR | Typical Question to Ask | Example Numbers |
|---|---|---|
| **Scale** | How many DAU? Read/write ratio? | 10M DAU, 100:1 read/write |
| **Availability** | 99.9% (8.7h downtime/yr) or 99.99% (52min/yr)? | Most systems: 99.99% |
| **Latency** | p99 latency for reads? Writes? | Feed load < 200ms, chat < 100ms |
| **Consistency** | Strong or eventual? Can users see stale data? | Feed: eventual OK, payments: strong |
| **Durability** | Can we lose any data? How much? | Payments: zero loss. Logs: small loss OK |
| **Throughput** | Peak QPS? Sustained QPS? | 10K writes/sec sustained, 50K peak |

**Key trade-offs to state explicitly:**
- Availability vs Consistency (CAP theorem — you cannot have both during a partition)
- Latency vs Durability (sync replication = durable but slow, async = fast but risk losing data)
- Cost vs Performance (more replicas = faster reads but expensive)

---

## Numbers Every Engineer Should Know

Memorize these. Back-of-envelope estimation depends on them.

### Latency Numbers

| Operation | Latency | Intuition |
|---|---|---|
| L1 cache read | 1 ns | |
| L2 cache read | 4 ns | |
| RAM read | 100 ns | 100× slower than L1 |
| SSD random read | 100 µs | 1,000× slower than RAM |
| HDD random read | 10 ms | 100,000× slower than RAM |
| Same datacenter network | 0.5 ms | |
| Cross-region (US → EU) | 150 ms | |
| Redis GET | ~0.5 ms | In-memory, same DC |
| PostgreSQL query (indexed) | 1–5 ms | Disk I/O + query plan |

**Rule of thumb:** RAM = fast, SSD = OK, Disk = slow, Network = depends on distance

### Throughput Numbers

| System | Throughput |
|---|---|
| Single Redis server | ~1,000,000 ops/sec |
| Single Kafka partition | ~100,000 messages/sec |
| Single PostgreSQL (writes) | ~5,000–10,000 writes/sec |
| Single Cassandra node (writes) | ~50,000 writes/sec |
| Single app server (HTTP) | ~10,000 requests/sec |
| 1 Gbps network link | ~125 MB/sec |

### Storage Units

```
1 KB  = 1,000 bytes      → a short text message
1 MB  = 1,000 KB         → a photo thumbnail
1 GB  = 1,000 MB         → a movie
1 TB  = 1,000 GB         → ~1M photos
1 PB  = 1,000 TB         → large company's annual data

Useful estimates:
  1 char    = 1 byte
  Tweet     = ~280 bytes (text) + metadata ≈ 1 KB total
  Photo     = ~300 KB (compressed)
  HD video  = ~1–2 GB per hour
  1M users posting 1 photo/day = 300 KB × 1M = 300 GB/day
```

### Time Units for QPS Estimation

```
1 day  = 86,400 seconds  ≈ 100,000 sec  (use this to simplify)
1 year = 365 × 86,400   ≈ 31.5M seconds

QPS formula:  QPS = (DAU × actions_per_user) / 86,400

Example:
  100M DAU, each sends 10 messages/day
  Write QPS = (100M × 10) / 100,000 = 10,000 writes/sec
```

---

## Quick Topic Checklist — Before Any Interview

- [ ] CAP Theorem — and which trade-off your design makes
- [ ] Consistent Hashing — adding/removing nodes, virtual nodes
- [ ] SQL vs NoSQL — when to use which
- [ ] Caching — cache-aside, write-through, TTL, eviction
- [ ] Message Queue — Kafka partitions, consumer groups, at-least-once vs exactly-once
- [ ] Rate Limiting — token bucket vs sliding window log
- [ ] WebSocket vs Long Polling — when each makes sense
- [ ] Geohash / Quadtree — for any location-based system
- [ ] Database Replication — leader-follower, replication lag
- [ ] Sharding — hash vs range, hotspot prevention
- [ ] Bloom Filter — for deduplication (web crawler, username check)
- [ ] Idempotency — payment, message processing, API design
