# System Design: Metrics Monitoring and Alerting System (Beginner-Friendly Guide)

---

## What Are We Building?

Think about the dashboard in a car — it shows your speed, fuel level, engine temperature. If something is wrong, a warning light flashes. You don''t have to open the hood to know the engine is overheating.

**A metrics monitoring system is that dashboard — but for software infrastructure.**

It answers questions like:
- Is our server CPU running hot right now?
- How many requests per second is our API handling?
- Did memory usage spike in the last 5 minutes?
- Is anything broken that we should page an engineer about at 3am?

**Real-world examples:** Datadog, Prometheus + Grafana, AWS CloudWatch, New Relic

**Our design scope:** Internal monitoring system for a large tech company — not a SaaS product.

---

## Step 1: Design Scope

**What we''re monitoring:**
- Operational metrics: CPU load, memory usage, disk space
- Application metrics: requests per second, error rates, queue depth
- NOT logs (that''s ELK stack) and NOT distributed tracing (that''s Jaeger/Zipkin)

**Scale:**
| Parameter | Value |
|-----------|-------|
| Daily Active Users of the platform being monitored | 100 million |
| Server pools | 1,000 |
| Machines per pool | 100 |
| Metrics per machine | ~100 |
| **Total metrics** | **~10 million** |
| Data retention | 1 year |

**Data retention policy (downsampling over time):**
```
Day 0–7:   raw data (every 10 seconds)
Day 7–30:  downsampled to 1-minute resolution
Day 30+:   downsampled to 1-hour resolution
```

> **Why downsample?** Storing 10-second resolution data for a year would be astronomically expensive. You don''t need to know the exact CPU spike at 14:32:40 from 8 months ago — the hourly average is enough for trend analysis.

**Alert channels:** Email, SMS, PagerDuty, webhooks

---

## Step 2: The Five Core Components

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/metrics-monitoring-core-components.png" alt="Core Components" width="450">

Every metrics monitoring system has these five building blocks:

| Component | Role |
|-----------|------|
| **Data Collection** | Gather metrics from servers, apps, databases |
| **Data Transmission** | Move metrics from sources to the monitoring system |
| **Data Storage** | Store time-series data efficiently |
| **Alerting** | Detect anomalies, trigger notifications |
| **Visualization** | Show graphs, charts, dashboards |

---

## Step 3: Understanding the Data Model — Time-Series

### What Is a Time-Series?

Every metric is a **time-series** — a sequence of values at different points in time.

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/metrics-example-1.png" alt="Metrics Example 1" width="450">

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/metrics-example-1-data.png" alt="Metrics Example 1 Data" width="450">

**Format (Line Protocol):**
```
{metric_name} {labels/tags} {timestamp} {value}

CPU.load host=webserver01,region=us-west 1613707265 50
CPU.load host=webserver01,region=us-west 1613707280 62
CPU.load host=webserver02,region=us-west 1613707265 43
```

Each data point has:
- **Metric name** — what you''re measuring (`CPU.load`)
- **Labels/Tags** — dimensions to filter by (`host=webserver01`, `region=us-west`)
- **Timestamp** — when it was measured (Unix epoch)
- **Value** — the measurement (50%)

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/time-series-data-example.png" alt="Time Series Data Example" width="450">

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/time-series-data-viz.png" alt="Time Series Visualization" width="450">

> **Why labels matter:** Without labels, you just know "CPU is at 62%." With `host=webserver01,region=us-west` you know *which* machine in *which* region — enabling filtering, grouping, and aggregation.

### Why Not Use a Regular Database?

| Storage Type | Problem for Metrics |
|-------------|---------------------|
| SQL (PostgreSQL) | Designed for random reads/writes; no time-range compression; no native aggregation |
| NoSQL (Cassandra) | Can work, but no native time-series query interface; hard to aggregate by labels |
| **Time-Series DB (InfluxDB)** | ✅ Built for this: compression, retention policies, label indexing, fast range queries |

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/influxdb-scale.png" alt="InfluxDB Scale" width="450">

> InfluxDB can handle **250,000 writes per second** on an 8-core, 32GB RAM machine. A standard relational DB would melt under that load.

**Popular time-series databases:** InfluxDB, Prometheus (built-in TSDB), OpenTSDB, Amazon Timestream

> **Key warning on labels:** Keep label cardinality LOW. If you add `user_id` as a label on a metric — 50 services × 200 endpoints × 10M users = **1 trillion unique series**. That kills any TSDB instantly. Use labels for bounded dimensions only (region, hostname, status_code).

---

## Step 4: High-Level Design

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/high-level-design.png" alt="High Level Design" width="500">

```
[Metrics Sources]         ← servers, apps, DBs, queues
      ↓
[Metrics Collector]       ← gathers + transmits data
      ↓
[Kafka]                   ← buffer + decoupling layer
      ↓
[Stream Processor]        ← Flink/Spark (optional aggregation)
      ↓
[Time-Series Database]    ← InfluxDB / Prometheus
      ↓               ↓
[Query Service]    [Alerting System]
      ↓               ↓
[Visualization]   [Email/SMS/PagerDuty]
```

---

## Step 5: Metrics Collection — Pull vs. Push

This is one of the most important design decisions.

### Pull Model (Prometheus-style)

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/pull-model-example.png" alt="Pull Model" width="450">

The metrics collector **asks** each server for its metrics on a schedule:
```
Collector → GET http://server01:8080/metrics → gets metrics
Collector → GET http://server02:8080/metrics → gets metrics
```

**Service discovery** tells the collector which servers to poll:

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/service-discovery-example.png" alt="Service Discovery Example" width="450">

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/metrics-collection-flow.png" alt="Metrics Collection Flow" width="450">

Flow:
1. Collector fetches server list + config from service discovery (ZooKeeper/etcd)
2. Collector polls each server''s `/metrics` endpoint at the configured interval
3. Server responds with current metrics values
4. Collector writes to time-series DB

**Scaling:** Use a **consistent hash ring** to distribute servers across multiple collectors — no two collectors poll the same server:

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/consistent-hash-ring.png" alt="Consistent Hash Ring" width="450">

### Push Model (Graphite/StatsD-style)

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/push-model-example.png" alt="Push Model" width="450">

Each server proactively **sends** its metrics to the collector. A lightweight **agent** runs on each server:

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/metrics-collector-agent.png" alt="Metrics Collector Agent" width="450">

Agent collects local metrics → batches them → pushes to the central collector.

### Pull vs. Push Comparison

| Scenario | Pull | Push |
|----------|------|------|
| Debugging | ✅ Hit `/metrics` on any server directly from your laptop | ❌ Can''t inspect metrics without collector |
| Server health check | ✅ No response = server down | ❌ Silence could be network issue |
| Short-lived batch jobs | ❌ Job finishes before next scrape | ✅ Job pushes before it exits |
| Multi-datacenter setups | ❌ Collector must reach every server (firewall issues) | ✅ Services push outward through NAT/firewall |
| Transport | TCP (connection overhead) | UDP (lower latency, can drop packets) |
| Data authenticity | ✅ Only configured endpoints are scraped | ❌ Any client can push (need authentication) |

> **Verdict:** No clear winner. Large organizations support both. Prometheus = pull. CloudWatch, Graphite = push. Our design supports both by using Kafka as a universal buffer.

---

## Step 6: Scaling the Transmission Pipeline

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/metrics-transmission-pipeline.png" alt="Metrics Transmission Pipeline" width="450">

**Problem:** If the time-series DB is temporarily down, metric data is lost.

**Solution: Add Kafka as a buffer**

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/queuing-mechanism.png" alt="Queuing Mechanism" width="450">

```
[Metrics Collector] → [Kafka] → [Consumer/Stream Processor] → [Time-Series DB]
```

Benefits of Kafka here:
- **Buffer:** If TSDB is down, Kafka holds metrics until it recovers
- **Decoupling:** Collector doesn''t care about the downstream DB
- **Fan-out:** Same metrics stream can be consumed by multiple systems (TSDB + cold storage + anomaly detection)
- **Backpressure:** Consumers can process at their own rate

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/metrics-collection-kafka.png" alt="Metrics Collection Kafka" width="450">

**Kafka partitioning strategy:** One partition per metric name → consumers can aggregate data per metric name efficiently. Further partition by tags/labels for scale.

> **Alternative:** Facebook''s Gorilla in-memory metrics system skips Kafka entirely and writes directly to an in-memory store. For very large scales, a purpose-built ingestion system may outperform a general-purpose Kafka cluster.

### Where to Aggregate?

Metrics can be aggregated at three stages:

| Stage | How | Trade-off |
|-------|-----|-----------|
| **Collection agent** | Agent sums counters locally for 1min, sends one value | Simple; loses precision |
| **Ingestion pipeline** (Flink/Spark) | Stream processor aggregates before writing to DB | Reduces write volume; loses raw data |
| **Query time** | Aggregate when you run a query | Full precision; slow queries on large data |

Most systems use a combination: light aggregation at the agent, raw data written to TSDB, heavier aggregation via pre-computed "recording rules."

---

## Step 7: Query Service

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/cache-layer-query-service.png" alt="Cache Layer Query Service" width="450">

A dedicated query service sits between the TSDB and visualization/alerting systems:

```
[Visualization (Grafana)] ──→ [Query Service] ──→ [Cache] ──→ [Time-Series DB]
[Alerting System]         ──→ [Query Service]
```

**Why a separate query service?**
- Decouples the DB from clients — swap databases without touching Grafana config
- Adds a caching layer to reduce repeated expensive queries
- Can enforce access control (who can query which metrics)

**Why not SQL for metrics queries?**

This SQL query computes a 15-minute exponential moving average:
```sql
SELECT id, temp,
  AVG(temp) OVER (PARTITION BY group_nr ORDER BY time_read) AS rolling_avg
FROM (
  SELECT id, temp, time_read, interval_group,
    id - ROW_NUMBER() OVER (PARTITION BY interval_group ORDER BY time_read) AS group_nr
  FROM (
    SELECT id, time_read,
      "epoch"::timestamp + "900 seconds"::interval * (
        EXTRACT(EPOCH FROM time_read)::int4 / 900
      ) AS interval_group, temp
    FROM readings
  ) t1
) t2 ORDER BY time_read;
```

The same query in **Flux** (InfluxDB''s query language):
```
from(db:"telegraf")
  |> range(start:-1h)
  |> filter(fn: (r) => r._measurement == "foo")
  |> exponentialMovingAverage(size:-10s)
```

> Time-series query languages are designed for exactly these patterns. SQL is the wrong tool here.

---

## Step 8: Storage — Keeping 10 Million Metrics Efficiently

### Compression: Double-Delta Encoding

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/double-delta-encoding.png" alt="Double Delta Encoding" width="450">

Instead of storing full timestamps for every data point:
```
Raw:     1613707200, 1613707210, 1613707220, 1613707230  (4 × 8 bytes = 32 bytes)
Delta:   1613707200, +10, +10, +10                       (smaller)
Double delta: 1613707200, +10, 0, 0                      (even smaller — deltas of deltas)
```

Good time-series databases (InfluxDB, Facebook''s Gorilla) achieve **12× compression** on float64 metric values using this technique. Huge storage savings at 10M metrics.

### Downsampling

Raw 10-second data for one year per metric = ~3.1 million data points. At 10M metrics = 31 trillion data points. Impossible to store raw.

**Solution: Downsample old data**

```
10-second resolution data:
| timestamp            | host   | cpu_pct |
|----------------------|--------|---------|
| 2026-05-08 19:00:00  | host-a | 10      |
| 2026-05-08 19:00:10  | host-a | 16      |
| 2026-05-08 19:00:20  | host-a | 20      |
| 2026-05-08 19:00:30  | host-a | 30      |
| 2026-05-08 19:00:40  | host-a | 20      |
| 2026-05-08 19:00:50  | host-a | 30      |

Downsampled to 30-second resolution (avg):
| timestamp            | host   | cpu_pct |
|----------------------|--------|---------|
| 2026-05-08 19:00:00  | host-a | 15.3    |
| 2026-05-08 19:00:30  | host-a | 26.7    |
```

A background job runs periodically: every old segment beyond the retention threshold gets averaged into coarser buckets.

> **~85% of queries** (from Facebook research) access data from the **past 26 hours**. So keeping recent data in fast hot storage (SSD) and old data in slower cold storage (HDD/S3) makes a massive performance difference.

### Cold Storage

Data older than the operational window can be archived in cheap object storage (S3, GCS). Cost is 10–50× lower than SSD. Queries against archived data are slow, but that''s acceptable — no one needs sub-second latency for a query about CPU usage from 11 months ago.

---

## Step 9: Alerting System

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/alerting-system.png" alt="Alerting System" width="450">

### How Alerts Are Defined

Alert rules are stored as YAML configuration:

```yaml
- name: instance_down
  rules:
  - alert: instance_down
    expr: up == 0          ← PromQL expression
    for: 5m                ← must be true for 5 min (avoids transient spikes)
    labels:
      severity: page
```

### Alert Manager Flow

```
[Alert Config Cache]
      ↓
[Alert Manager] ← queries [Query Service] every interval
      ↓
 Rule met? → create alert event
      ↓
[Alert Store (Cassandra)]  ← durable state, ensures at-least-once delivery
      ↓
[Kafka alert topic]
      ↓
[Alert Consumers]
      ↓
[Email] [SMS] [PagerDuty] [Webhooks]
```

**Alert Manager responsibilities:**

| Responsibility | Why |
|----------------|-----|
| **Filtering** | Avoid noisy low-value alerts reaching on-call |
| **Deduplication** | If same instance fires 10 times in 1 minute → 1 alert event |
| **Access control** | Only authorized people can modify alert rules |
| **Retry/at-least-once** | Critical alerts must not be silently dropped |

> **Kafka for alerts:** Publishing alerts to Kafka enables multiple consumer channels (email + PagerDuty + Slack webhook) to all receive the same alert independently, at their own pace.

---

## Step 10: Visualization System

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/grafana-dashboard.png" alt="Grafana Dashboard" width="500">

Grafana is the industry-standard visualization tool. It:
- Connects to any TSDB via data source plugins (Prometheus, InfluxDB, CloudWatch...)
- Lets you build dashboards with graphs, gauges, tables, heatmaps
- Stores dashboards as JSON (version-controllable in Git)
- Has its own alerting engine built in

> **Don''t build your own visualization system.** Grafana is open source, battle-tested, and has thousands of community dashboards. The engineering effort to build something comparable is enormous.

---

## Step 11: Final Architecture

<img src="../../system-design-notes/20. Metrics Monitoring and Alerting System/images/final-design.png" alt="Final Design" width="500">

```
[Servers / Apps / DBs / Queues]
          ↓ (pull: /metrics scrape or push: agent)
[Metrics Collector Fleet]  ←  [Service Discovery (ZooKeeper/etcd)]
   (consistent hash ring distributes targets across collectors)
          ↓
       [Kafka]
          ↓
[Stream Processor (Flink/Spark)]  ← optional aggregation
          ↓
[Time-Series DB (InfluxDB)]
   Hot: 7 days SSD
   Warm: 30 days HDD (1min resolution)
   Cold: 1 year S3 (1hr resolution)
          ↓                    ↓
   [Query Service]        [Alert Manager]
   [+ Cache Layer]              ↓
          ↓            [Alert Store (Cassandra)]
   [Grafana]                    ↓
   (visualization)           [Kafka]
                                ↓
                    [Alert Consumers]
                    → Email / SMS / PagerDuty / Webhooks
```

---

## Summary: The "Why" Behind Each Decision

| Decision | Why? |
|----------|------|
| Time-series DB over SQL | Write-heavy append pattern; compression; label indexing; domain-specific query language |
| Kafka between collector and DB | Decouples collection from storage; absorbs spikes; enables fan-out; prevents data loss on DB outage |
| Consistent hash ring for collectors | Prevents duplicate scraping when multiple collectors run |
| Downsampling old data | Can''t store 10-second resolution for 1 year — costs too much; old data accessed rarely |
| Double-delta encoding | 12× compression by storing timestamp/value deltas instead of absolute values |
| Query service (separate from DB) | Decouples DB from clients; adds caching layer; enables DB swaps without touching clients |
| Alert rules with `for: 5m` | Prevents false positives from transient spikes; alert must be sustained to matter |
| Alert deduplication in Alert Manager | 10 instances of the same alert → 1 PagerDuty page (avoid on-call engineer burnout) |
| Cassandra for alert store | Durable KV store; ensures at-least-once alert delivery if Alert Manager restarts |
| Grafana for visualization | Too expensive to build; open-source; thousands of community dashboards; plugin ecosystem |

---

## Quick Interview Q&A Cheat Sheet

**Q: What is a time-series database and why not use PostgreSQL?**  
A: Metrics are `{metric, labels, timestamp, value}` tuples — billions of rows written sequentially, queried by time range. TSDBs use columnar storage, delta encoding (12× compression), built-in downsampling, and label indexes. PostgreSQL lacks native compression for float time-series, has no auto-aggregation, and struggles past ~10M rows/day. InfluxDB handles 250K writes/sec on a single 8-core machine.

**Q: Pull vs. push model — which is better?**  
A: No clear winner. Pull (Prometheus): easier debugging, natural health checks, server controls scrape rate — but struggles with firewall rules and short-lived jobs. Push (Graphite/CloudWatch): works anywhere including batch jobs and behind NATs — but any client can send data (needs auth). Large orgs support both. Our design accepts both via Kafka.

**Q: Why put Kafka between the collector and the TSDB?**  
A: Three reasons: (1) Buffer — if TSDB is down, Kafka retains data until recovery; (2) Decoupling — swap TSDB without changing collectors; (3) Fan-out — same stream consumed by TSDB, anomaly detection, and cold archive simultaneously.

**Q: How do you handle a year of data for 10 million metrics without spending a fortune?**  
A: Downsampling + tiered storage. Raw 10s data kept 7 days on SSD. After 7 days → downsample to 1-minute averages (6× reduction). After 30 days → downsample to 1-hour averages (60× reduction from 1-minute). Old data moved to cheap object storage (S3). Additionally, delta encoding gives ~12× compression on raw values.

**Q: What is cardinality and why is it dangerous?**  
A: Cardinality = unique time series = product of unique values across all labels. Adding `user_id` as a metric label: 50 services × 200 endpoints × 10M users = 1 trillion series — kills any TSDB. Never use unbounded values (user_id, request_id, IP) as labels. Use distributed tracing for per-request granularity.

**Q: How does the alerting system avoid waking up engineers for false alarms?**  
A: Three mechanisms: (1) `for: 5m` — alert must be sustained for 5 minutes before firing (ignores transient spikes); (2) Deduplication — same alert firing 10 times → 1 PagerDuty page; (3) Severity tiers — P1 pages on-call, P2 creates a ticket, P3 is just a log entry. Alert on SLO breaches (user-visible impact), not individual component metrics.

**Q: What''s the difference between metrics, logs, and traces?**  
A: Metrics: numerical time-series aggregates (CPU = 82%) — cheap, great for dashboards/alerts, no per-request detail. Logs: text emitted by code (ERROR user_id=123 failed) — expensive at scale, great for debugging specific errors. Traces: distributed request path across microservices (Jaeger) — correlates with logs via trace_id. All three together = full observability.

**Q: How do you prevent the monitoring system itself from being a single point of failure?**  
A: Run 2+ Prometheus instances scraping the same targets. Run Alertmanager as a 3-node cluster with gossip deduplication. Use an external uptime checker (Pingdom) to monitor the monitoring system itself. Keep alert delivery on external SaaS (PagerDuty) which has its own 99.99% SLA.
