# System Design: Ad Click Event Aggregation (Beginner-Friendly Guide)

---

## What Are We Building?

Every time you see an ad on Facebook, YouTube, or Google and click it, that click is an **event**. The advertiser pays for that click. So every click must be counted accurately — because money is on the line.

**Ad click event aggregation** answers questions like:
- How many times was ad #123 clicked in the last 5 minutes?
- What are the top 100 most clicked ads in the last minute?
- How many US users clicked this ad today?

**Why is this hard?**
- 1 billion ad clicks per day = ~10,000 clicks/second on average, 50,000 at peak
- Data must be **accurate** — 1% error could mean millions of dollars of discrepancy in advertiser bills
- Data must be **timely** — advertisers need near-real-time dashboards
- Late events, duplicates, and system crashes all threaten correctness

> **Real-Time Bidding (RTB)** is the other half of digital advertising: when you load a webpage, an auction for that ad slot runs in <100ms. Click aggregation data feeds RTB — your CTR (click-through rate) history informs how much advertisers bid for your attention.

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/digital-advertising-example.png" alt="Digital Advertising Example" width="450">

---

## Step 1: Design Scope

**Functional requirements:**
- Aggregate the number of clicks for an `ad_id` in the last Y minutes
- Return top 100 most clicked ads in the last M minutes (both configurable)
- Support filtering by `ip`, `user_id`, `country`

**Non-functional requirements:**
- **Correctness** — used for billing; small errors = large financial impact
- **Handle late/duplicate events** — network delays, client retries
- **Fault tolerance** — system must survive partial failures
- **Latency** — a few minutes of end-to-end latency is acceptable (this is reporting, not RTB)

**Scale estimation:**

| Metric | Value |
|--------|-------|
| Daily ad clicks | 1 billion |
| Average QPS | 10,000 clicks/sec |
| Peak QPS | 50,000 clicks/sec |
| Storage per click | 0.1 KB |
| Daily storage | ~100 GB |
| Monthly storage | ~3 TB |

---

## Step 2: API Design

Two endpoints cover all functional requirements:

**Get click count for a specific ad:**
```
GET /v1/ads/{ad_id}/aggregated_count
Query params:
  from   = start minute (default: now - 1min)
  to     = end minute (default: now)
  filter = filter ID (e.g., 001 = "non-US clicks only")

Response:
  { "ad_id": "ad001", "count": 2341 }
```

**Get top N most clicked ads:**
```
GET /v1/ads/popular_ads
Query params:
  count  = top N ads to return
  window = window size in minutes
  filter = filter ID

Response:
  { "ad_ids": ["ad999", "ad123", "ad456", ...] }
```

---

## Step 3: Data Model — Raw vs. Aggregated

### Raw Data (what arrives)

Each click event:
```
ad001, 2026-01-01 00:00:01, user1, 207.148.22.22, USA
```

Structured:
| ad_id | click_timestamp     | user_id | ip            | country |
|-------|---------------------|---------|---------------|---------|
| ad001 | 2026-01-01 00:00:01 | user1   | 207.148.22.22 | USA     |
| ad001 | 2026-01-01 00:00:02 | user1   | 207.148.22.22 | USA     |
| ad002 | 2026-01-01 00:00:02 | user2   | 209.153.56.11 | USA     |

### Aggregated Data (what we query)

Pre-aggregated by minute with filter support:
| ad_id | click_minute | filter_id | count |
|-------|--------------|-----------|-------|
| ad001 | 202601010000 | 0012      | 2     |
| ad001 | 202601010000 | 0023      | 3     |

Filter table (maps filter_id to criteria):
| filter_id | region | ip        | user_id |
|-----------|--------|-----------|---------|
| 0012      | US     | *         | *       |
| 0013      | *      | 123.1.2.3 | *       |

Top-N structure:
| window_size | update_time_minute | most_clicked_ads  |
|------------|-------------------|-------------------|
| 1          | 202601010001      | ["ad999", "ad123"] |

### Should We Store Raw or Aggregated? Both.

| Approach | Pros | Cons |
|----------|------|------|
| Raw only | Full precision; supports any future query | Huge storage; slow queries |
| Aggregated only | Fast queries; small storage | Data loss; can''t recalculate if aggregation had a bug |
| **Both (our choice)** | Debug from raw; query from aggregated | Extra storage cost |

Raw data → cold storage (S3, cheap). Aggregated data → Cassandra (fast reads/writes).

> **Why Cassandra for both?** Raw data is write-heavy (10–50K writes/sec); few reads (only for debugging). Aggregated data is both read and write heavy (every dashboard query + every 1-minute aggregation write). Cassandra handles heavy writes natively via its LSM-tree storage. It also scales horizontally with consistent hashing — no manual resharding.

---

## Step 4: High-Level Design

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/high-level-design-1.png" alt="High Level Design 1" width="450">

The naive design: click → service → database. Problem: if the database is slow, everything backs up. If the aggregation service crashes mid-computation, we lose work.

**Solution: Decouple with Kafka**

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/high-level-design-2.png" alt="High Level Design 2" width="500">

```
[Ad Click Events]
       ↓
 [Kafka Topic 1]          ← raw click events
       ↓
 [Aggregation Service]    ← MapReduce DAG
       ↓
 [Kafka Topic 2]          ← aggregated results per minute
       ↓
  [Cassandra]             ← stores aggregated counts + raw backup
```

**Why two Kafka topics?**

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/atomic-commit.png" alt="Atomic Commit" width="450">

The second Kafka topic enables **atomic commit semantics**: the aggregation service writes its result to Kafka Topic 2 and commits its Kafka Topic 1 offset in the same transaction. If either fails, both roll back. This prevents partial writes where results land in the DB but the offset isn''t committed (or vice versa), which would cause duplicate processing.

---

## Step 5: Aggregation Service — MapReduce DAG

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/ad-count-map-reduce.png" alt="Ad Count MapReduce" width="450">

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/top-100-map-reduce.png" alt="Top 100 MapReduce" width="450">

The aggregation service is a DAG (Directed Acyclic Graph) of three node types:

### Map Node

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/map-node.png" alt="Map Node" width="450">

Reads events from Kafka Topic 1, filters/transforms them, then routes each event to the correct aggregation node based on `ad_id`:

```
event(ad001) → hash(ad001) % 3 = 0 → Aggregation Node 0
event(ad002) → hash(ad002) % 3 = 1 → Aggregation Node 1
event(ad003) → hash(ad003) % 3 = 2 → Aggregation Node 2
```

> **Why not just use Kafka partitions directly?** We might not control how events are produced — two events for the same `ad_id` might land in different Kafka partitions. The Map node gives us control to re-route events correctly. It also lets us sanitize/enrich data before aggregation.

### Aggregate Node

Counts ad click events by `ad_id` **in-memory** every minute. Each node handles a subset of `ad_id`s (determined by the Map node''s routing).

### Reduce Node

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/reduce-node.png" alt="Reduce Node" width="450">

Collects aggregated results from all Aggregate nodes → computes the final result → writes to Kafka Topic 2.

### Use Case 1: Count Clicks Per Ad

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/use-case-1.png" alt="Use Case 1" width="450">

Events partitioned by `ad_id % 3` → each partition''s events counted separately → results written out.

### Use Case 2: Top N Most Clicked Ads

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/use-case-2.png" alt="Use Case 2" width="450">

Each Aggregate node maintains a **min-heap** of top N ads for its partition. Reduce node merges all heaps to get the global top N.

### Use Case 3: Filtering (Star Schema)

To support "clicks from US only" without scanning everything, **pre-aggregate by dimension**:

| ad_id | click_minute | country | count |
|-------|--------------|---------|-------|
| ad001 | 202601010001 | USA     | 100   |
| ad001 | 202601010001 | GBR     | 200   |
| ad001 | 202601010001 | others  | 3000  |

This is the **star schema** — the filtering dimensions are pre-baked into the aggregation. Query for "US clicks for ad001" becomes an O(1) lookup instead of a full scan.

> **Trade-off:** More dimensions = more rows stored. With 10 filter dimensions × 2M ads × 1M minutes per year = a lot of rows. Keep dimensions bounded and meaningful.

---

## Step 6: Streaming vs. Batching

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/lambda-architecture.png" alt="Lambda Architecture" width="450">

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/kappa-architecture.png" alt="Kappa Architecture" width="450">

| Architecture | Description | Our Use? |
|-------------|-------------|---------|
| **Lambda** | Two parallel paths: streaming (fast, approximate) + batch (slow, accurate); results merged | Common but complex — two codebases |
| **Kappa** | Single stream processing path; historical reprocessing = replaying Kafka | ✅ Our design uses Kappa |

Our design is **Kappa**: one aggregation service handles both real-time and historical reprocessing.

### Recalculation (Bug Recovery)

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/recalculation-example.png" alt="Recalculation Example" width="450">

If a bug is found in aggregation logic:
1. **Recalculation service** reads raw events from cold storage (S3/Cassandra)
2. Sends to a **separate aggregation service** (doesn''t disrupt real-time pipeline)
3. Results flow through Kafka Topic 2 → overwrite wrong results in Cassandra

> **This is why we keep raw data.** Without it, a bug means the wrong numbers are billed forever. With raw data, you can always recompute from ground truth.

---

## Step 7: Time — Event Time vs. Processing Time

Ad click events travel through: phone → cell tower → internet → load balancer → Kafka → aggregation service.

**Processing time** = when the aggregation service sees the event  
**Event time** = when the user actually clicked the ad (embedded in the event)

These can differ by seconds or minutes due to network delays and Kafka buffering.

| Approach | Pros | Cons |
|----------|------|------|
| Processing time | Simple; no clock skew issues | Inaccurate for late events (click at 11:59:58 counted in next minute''s bucket) |
| **Event time** | Accurate billing | Must handle late events; client clocks can be wrong/malicious |

**We use event time** — billing accuracy is non-negotiable.

### Watermarks — Handling Late Events

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/watermark-technique.png" alt="Watermark Technique" width="450">

Problem: if we close a 1-minute window exactly at minute mark, late events that arrive 5 seconds later are lost.

Solution: **extend the window with a watermark**

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/watermark-2.png" alt="Watermark 2" width="450">

```
Window: 00:00 → 01:00
Watermark extension: +30 seconds
Effective close time: 01:00:30

Events with event_time in [00:00, 01:00] that arrive before 01:00:30 → included ✅
Events arriving after 01:00:30 → too late, handled by end-of-day reconciliation
```

| Watermark size | Effect |
|---------------|--------|
| Short | Low latency, higher chance of missed late events |
| Long | Higher latency, fewer missed events |

> **How do you handle events that still miss the watermark?** End-of-day reconciliation — a batch job compares raw event totals against aggregated totals and patches discrepancies.

---

## Step 8: Aggregation Windows

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/tumbling-window.png" alt="Tumbling Window" width="450">

**Tumbling window** — fixed, non-overlapping buckets:
```
[00:00 → 01:00] [01:00 → 02:00] [02:00 → 03:00]
```
Used for: "how many clicks in minute 00:00?" — each event belongs to exactly one bucket.

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/sliding-window.png" alt="Sliding Window" width="450">

**Sliding window** — overlapping buckets:
```
[00:00 → 01:00]
      [00:30 → 01:30]
            [01:00 → 02:00]
```
Used for: "top 100 most clicked ads in the last M minutes" — a sliding window continuously updates the running total.

**Our design uses both:**
- Tumbling window → per-minute click count aggregation (for billing)
- Sliding window → top N ads over M minutes (for dashboards)

---

## Step 9: Delivery Guarantees & Deduplication

Billing requires **exactly-once** semantics. At-least-once is not enough — duplicate events = advertisers overbilled.

### Sources of Duplicates

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/data-duplication-example.png" alt="Data Duplication Example" width="450">

1. **Client-side:** User''s browser retries a failed HTTP request → same click sent twice
2. **Server-side:** Aggregation node processes event, crashes before acknowledging → upstream resends

### Naive Fix: Save Offset to S3 First

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/data-duplication-example-2.png" alt="Data Duplication Example 2" width="450">

Problem: save offset → crash before writing result downstream → offset is committed but result was never written. Next restart reads from wrong offset → **data lost**.

### Real Fix: Distributed Transaction (Atomic Commit)

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/data-duplication-example-3.png" alt="Data Duplication Example 3" width="450">

Commit Kafka offset AND write result to downstream **atomically**:
- Both succeed: ✅ normal operation
- Either fails: 🔄 roll back both; retry from last committed offset

> **Alternative:** If the downstream system handles duplicate writes idempotently (e.g., "write count=5 for ad001 at minute 14:00" is idempotent — same result if written twice), no distributed transaction is needed. Idempotent sinks are simpler and preferred when possible.

---

## Step 10: Scaling

### Message Queue Scaling

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/scale-consumers.png" alt="Scale Consumers" width="450">

- **Producers:** No limit — add more producers freely
- **Consumers:** Scale by adding consumer group members (up to partition count)
- **Partitions:** Pre-create enough partitions; adding partitions triggers consumer rebalance — do it during off-peak hours
- **Geography:** Partition topic by region (`topic_na`, `topic_eu`) to reduce cross-region data transfer

### Aggregation Service Scaling

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/aggregation-service-scaling.png" alt="Aggregation Service Scaling" width="450">

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/multi-threading-example.png" alt="Multi-Threading Example" width="450">

- Add more Map/Aggregate/Reduce nodes (horizontal scaling)
- Within each node: multi-threading to process multiple partitions concurrently
- Use resource managers (Apache YARN) to allocate CPU/memory dynamically across nodes

### Database Scaling

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/cassandra-scalability.png" alt="Cassandra Scalability" width="450">

Cassandra scales horizontally with **consistent hashing**:
- Add a new node → data automatically rebalances across all nodes via virtual nodes
- No manual resharding required
- Write throughput scales linearly with number of nodes

### Hotspot Issue — One Ad Going Viral

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/hotspot-issue.png" alt="Hotspot Issue" width="450">

Problem: `ad_id=superBowlAd2026` gets 1 million clicks per second during the Super Bowl. One aggregation node gets overwhelmed.

Solution: **dynamic resource allocation**
1. Overwhelmed node requests more resources from the resource manager
2. Resource manager allocates 3 additional aggregation nodes
3. Original node splits events into 3 sub-groups, each handled by a new node
4. Sub-results flow back to the original node for final merging

More advanced solutions: **Global-Local Aggregation** or **Split Distinct Aggregation** for complex aggregations.

---

## Step 11: Fault Tolerance

Aggregation nodes keep intermediate state (running click counts) **in memory**. If a node crashes, that in-memory state is lost.

### Recovery Strategy: Snapshots + Kafka Offsets

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/fault-tolerance-example.png" alt="Fault Tolerance Example" width="450">

Every minute, each aggregation node saves a **snapshot** of its current state to durable storage (S3/HDFS):
```
Snapshot at minute 14:
  node_0: { ad001: 342, ad004: 156, ... }
  node_1: { ad002: 891, ad005: 203, ... }
```

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/fault-tolerance-recovery-example.png" alt="Fault Tolerance Recovery Example" width="450">

When a node crashes and a replacement starts:
1. Load latest snapshot (e.g., minute 14''s state)
2. Read Kafka consumer offset for that snapshot
3. Re-consume Kafka events from that offset forward
4. Continue aggregation without restarting from zero

> **Trade-off:** Snapshot interval = recovery time. Snapshot every 1 minute → re-process at most 1 minute of events on crash. Snapshot every 10 minutes → re-process up to 10 minutes.

---

## Step 12: Data Correctness — Reconciliation

Since click data is used for billing, we need a correctness safety net beyond just "the pipeline seems to be working."

### Monitoring Metrics

| Metric | Why Monitor |
|--------|------------|
| Kafka consumer lag | Lag spike = aggregation service is falling behind |
| Aggregation output rate | Sudden drop = pipeline may be broken |
| System resources on nodes | CPU/memory spike = hotspot or memory leak |
| E2E latency (event time → DB write time) | Validates SLA |

### End-of-Day Reconciliation

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/reconciliation-flow.png" alt="Reconciliation Flow" width="450">

A nightly batch job:
1. Reads all raw click events for the day from cold storage
2. Computes accurate aggregated counts independently
3. Compares against what the streaming pipeline stored in Cassandra
4. Patches any discrepancies → generates reports for advertisers

```
Streaming pipeline says: ad001 → 1,234,567 clicks
Batch recomputation says: ad001 → 1,234,589 clicks
Discrepancy: 22 clicks (likely late-arriving events that missed watermark)
Action: update Cassandra with the accurate count; flag for investigation
```

---

## Step 13: Alternative Design (Off-the-Shelf Tooling)

<img src="../../system-design-notes/21. Ad Click Event Aggregation/images/alternative-design.png" alt="Alternative Design" width="450">

Instead of building a custom MapReduce aggregation service, use purpose-built tools:

```
Click events → Kafka → Flink (streaming aggregation) → ClickHouse/Druid (OLAP)
                    → S3 (raw data lake with Parquet/ORC format)
                    → Spark (batch reprocessing for recalculation)
```

| Component | Tool | Role |
|-----------|------|------|
| Stream processing | Apache Flink | Stateful windowed aggregation |
| OLAP database | ClickHouse / Apache Druid | Fast analytical queries |
| Raw storage | S3 + Parquet | Cheap, columnar, reprocessable |
| Batch processing | Apache Spark | Nightly reconciliation |
| Dashboard | Grafana / custom | Visualization |

> **In a system design interview:** You don''t need to know Flink internals. But knowing that Flink exists for stateful stream processing, Druid/ClickHouse for fast OLAP, and how they fit together shows strong practical knowledge.

---

## Summary: The "Why" Behind Each Decision

| Decision | Why? |
|----------|------|
| Kafka between click events and aggregation | Decouples producers from consumers; absorbs peak 50K/sec spikes; enables replay |
| Two Kafka topics | Enables atomic commit — offset commit and result write are transactional |
| MapReduce DAG (Map → Aggregate → Reduce) | Distributes load; each node does one focused job; horizontally scalable |
| Store raw data in cold storage | Bug recovery — recompute accurate aggregates from ground truth |
| Event time over processing time | Billing accuracy — a click at 11:59:58 belongs in the right minute''s bucket |
| Watermarks | Extend window to catch late events; trade latency for completeness |
| Tumbling window for counts | Fixed 1-minute buckets — each event belongs to exactly one bucket |
| Sliding window for top-N | Continuously updated "last M minutes" ranking |
| Exactly-once via atomic commit | Billing can''t tolerate duplicates; 1% error = millions of dollars |
| Snapshots + Kafka offsets | Fault tolerance without replaying all history — fast recovery |
| Reconciliation batch job | Catch residual discrepancies; provide auditable, accurate final numbers |
| Cassandra for storage | Write-heavy (50K/sec); scales horizontally with consistent hashing |

---

## Quick Interview Q&A Cheat Sheet

**Q: What is the high-level flow for ad click aggregation?**  
A: Click event → Kafka Topic 1 → Aggregation Service (MapReduce DAG) → Kafka Topic 2 → Cassandra. Raw events also written to cold storage (S3) for recalculation. Two Kafka topics enable atomic commit to prevent duplicates.

**Q: How do you handle late-arriving click events?**  
A: Use **event time** (not processing time) for aggregation. Add a **watermark** — extend each window by N seconds after it nominally closes. Events arriving within the watermark are included. Events arriving after the watermark are handled by end-of-day batch reconciliation.

**Q: Why exactly-once instead of at-least-once?**  
A: Click data is used for billing. Duplicate events = advertisers pay more than they should. Even 1% duplicates at $10 billion/year ad spend = $100 million overcharge. Exactly-once is required for financial accuracy.

**Q: How does the MapReduce aggregation work?**  
A: Map node routes events to Aggregate nodes by `hash(ad_id) % N`. Each Aggregate node counts clicks for its subset of ads in memory. Reduce node merges results for final output. For top-N: each Aggregate node maintains a min-heap; Reduce merges all heaps.

**Q: How do you recover when an aggregation node crashes?**  
A: Nodes snapshot their in-memory state (running counts) every minute to durable storage. On recovery: load latest snapshot + read Kafka from the offset corresponding to that snapshot + replay only the missed events. Recovery = at most 1 minute of data replayed.

**Q: What is the star schema and why use it for filtering?**  
A: Pre-aggregate by filter dimensions (country, region, user_id range) and store each dimension combination as a separate row. Query for "US clicks on ad001" becomes O(1) lookup instead of scanning all events. Trade-off: more rows stored, but dramatically faster queries.

**Q: Kappa vs. Lambda architecture?**  
A: Lambda: two codebases — streaming (fast, approximate) + batch (slow, accurate), merged at query time. Complex to maintain. Kappa: one streaming codebase handles both real-time and historical reprocessing (by replaying Kafka). Our design uses Kappa. Lambda is only better when batch produces significantly more accurate results than the stream.

**Q: How do you handle one ad going viral (hotspot)?**  
A: The overwhelmed aggregation node requests extra resources. Resource manager spins up sub-nodes. The hot ad''s events are split across sub-nodes; sub-results merged back. More advanced: global-local aggregation patterns where partial counts are distributed.
