# Types of Databases — System Design Guide

---

## The Big Picture

Every database makes trade-offs. There is no single "best" database — only the best database **for your specific access pattern**. Understanding those trade-offs is what system design interviews test.

```
The core questions to ask:
  1. Do I need strict consistency (ACID) or can I tolerate eventual consistency?
  2. Am I read-heavy, write-heavy, or balanced?
  3. Is my data structured (rows/columns) or flexible (documents, graphs)?
  4. Do I need to scale to billions of rows across many machines?
  5. Do I need complex queries (JOINs, aggregations) or simple key lookups?
```

---

## Category 1: Relational Databases (SQL)

**Examples:** PostgreSQL, MySQL, CockroachDB, Oracle, SQL Server

### What It Is

Data is stored in tables with rows and columns. Tables relate to each other via foreign keys. The database enforces a strict schema — every row must match the defined structure.

```
users table:               orders table:
id | name  | email         id  | user_id | amount | status
1  | Alice | a@gmail.com   101 | 1       | 50.00  | paid
2  | Bob   | b@gmail.com   102 | 1       | 30.00  | pending
3  | Carol | c@gmail.com   103 | 2       | 75.00  | paid
```

### Key Guarantees: ACID

| Property | Meaning | Example |
|---|---|---|
| **Atomicity** | All or nothing | Bank transfer: debit + credit both happen, or neither |
| **Consistency** | Data always valid | Can't create an order for a non-existent user |
| **Isolation** | Concurrent transactions don't interfere | Two people booking the last seat — only one wins |
| **Durability** | Committed data survives crashes | After "payment confirmed", data is safely on disk |

### When to Use SQL

- Financial systems — payments, banking, billing (need ACID absolutely)
- User accounts and authentication
- E-commerce: orders, inventory, products
- Any system where data has clear relationships and you need JOINs
- When you need complex reporting queries (GROUP BY, aggregations, subqueries)

### Weakness

- Scaling horizontally (adding more machines) is hard — SQL was designed for one machine
- Schema changes on large tables are painful (ALTER TABLE on 500M rows = hours of downtime)
- Not great for flexible or highly variable data shapes

---

## Category 2: Key-Value Stores

**Examples:** Redis, DynamoDB, Memcached

### What It Is

The simplest database model: a giant dictionary. Every value is stored under a unique key. No schema, no relations, no queries — just `get(key)` and `set(key, value)`.

```
"session:abc123"   →  { userId: 42, expires: 1748000000 }
"user:42:profile"  →  { name: "Alice", avatar: "url..." }
"rate:192.168.1.1" →  7   (number of requests in last minute)
"cart:user42"      →  ["item_101", "item_205", "item_309"]
```

### Access Pattern

- **O(1) lookup** — direct hash table access, no scanning
- Redis: 1,000,000+ reads/sec on a single node
- No joins, no complex queries — just fetch by key

### When to Use

| Use Case | Why Key-Value Wins |
|---|---|
| **Session storage** | Sessions accessed by session ID only — perfect key lookup |
| **Caching** | Store DB results by query key, serve from memory at microsecond speed |
| **Rate limiting** | `INCR user:42:requests` — atomic counter per user |
| **Shopping cart** | Cart stored under user ID, retrieved as one blob |
| **Leaderboards** | Redis Sorted Sets: score + rank in one data structure |

### Write vs Read

Redis is **both read-heavy and write-heavy** capable — it is in-memory, so both are fast. DynamoDB is optimised for **read-heavy** with DAX caching layer.

---

## Category 3: Document Stores

**Examples:** MongoDB, Firestore, CouchDB

### What It Is

Stores JSON-like documents. Each document can have a different structure. Related data is **embedded** in one document instead of spread across tables.

```json
// One document = one complete entity, no joins needed
{
  "_id": "post_456",
  "title": "System Design Tips",
  "author": { "id": 1, "name": "Alice" },
  "tags": ["tech", "backend"],
  "comments": [
    { "user": "Bob", "text": "Great post!", "likes": 5 },
    { "user": "Carol", "text": "Very helpful", "likes": 2 }
  ],
  "stats": { "views": 1502, "shares": 34 }
}
```

Fetching a post with all its comments = **one read**. In SQL you'd need a JOIN across 3 tables.

### When to Use

- Content management systems (articles, posts, pages)
- Product catalogs where each product has different attributes
- User profiles where fields vary per user
- Event logs and activity feeds
- Rapid development where schema changes frequently

### Write vs Read

Mostly **read-heavy**. Document stores are optimised for fetching entire documents. Writes that update nested arrays (appending a comment) can be expensive at scale.

---

## Category 4: Wide-Column Stores

**Examples:** Apache Cassandra, Google Bigtable, HBase, ScyllaDB

### What It Is

Data is stored in rows, but each row can have **different columns**. Think of it as a table where columns are not fixed — rows define their own columns. Optimised for writing and reading by a specific key (partition key).

```
Row key: "user123_2026-05"
Columns: { "2026-05-01 10:00": "login", "2026-05-01 10:05": "purchase", ... }

Row key: "user456_2026-05"
Columns: { "2026-05-03 08:30": "login", "2026-05-03 08:31": "search" }
```

### Why Cassandra — Deep Dive

Cassandra is the go-to database when you need:

**1. Extreme write throughput**

Cassandra uses an **LSM-tree** (Log-Structured Merge-tree). Every write goes to:
- An in-memory buffer (memtable) — extremely fast
- An append-only commit log on disk — sequential write = fast

No random disk seeks. No row locking. No B-tree rebalancing. Just append.

```
Result: Cassandra handles 1,000,000+ writes/sec across a cluster
Compare: PostgreSQL handles ~10,000-50,000 writes/sec on a single node
```

**2. Linear horizontal scaling**

Add a node → it automatically takes over a slice of the data. Remove a node → its data redistributes. No manual sharding, no DBA needed.

```
10 nodes  →  500K writes/sec
20 nodes  →  1M writes/sec   (just add nodes)
100 nodes →  5M writes/sec
```

**3. No single point of failure**

Cassandra is peer-to-peer — there is no master node. Every node is equal. Lose 2 nodes out of 10 (with replication factor 3) → system keeps running with zero downtime.

**4. Tunable consistency**

You choose per-query: do you want strong consistency or eventual consistency?
```
QUORUM read   → reads from majority of replicas → strong consistency, slower
ONE read      → reads from 1 replica → eventual consistency, faster
ALL read      → reads from all replicas → strongest, slowest
```

### When to Use Cassandra

| Use Case | Why |
|---|---|
| **Netflix viewing history** | Billions of watch events/day, always writing, read by user ID |
| **IoT sensor data** | Millions of devices writing temperature/GPS every second |
| **Messaging (WhatsApp)** | Billions of messages/day, read by conversation ID |
| **Activity feeds (Instagram)** | Write every like/comment, read by user timeline |
| **Time-series metrics** | Append-only writes, query by time range |

### Cassandra's Weakness

- **No joins** — data must be denormalised (duplicated) for different query patterns
- **No ACID transactions** across multiple partitions
- **Reads are slower than writes** — LSM-tree requires merging multiple levels
- **Schema design is hard** — you must design tables around your queries, not your data model

---

## Category 5: CockroachDB — Why It Exists

**Type:** Distributed SQL (NewSQL)

### The Problem It Solves

PostgreSQL is great but doesn't scale horizontally. Cassandra scales but loses ACID. CockroachDB gives you **both**:

```
PostgreSQL:  ACID ✅  |  Horizontal scale ❌
Cassandra:   ACID ❌  |  Horizontal scale ✅
CockroachDB: ACID ✅  |  Horizontal scale ✅
```

### How It Works

CockroachDB is built on top of RocksDB (LSM-tree storage) but adds a distributed SQL layer on top. It uses the **Raft consensus algorithm** to keep replicas in sync — every write is committed to a majority of replicas before being acknowledged.

```
Write "UPDATE accounts SET balance=900 WHERE id=42":

CockroachDB node 1  ─────┐
CockroachDB node 2  ──── Raft consensus: majority must agree before commit
CockroachDB node 3  ─────┘

Once majority (2/3) agree → write confirmed → ACID guaranteed
Even if node 1 crashes → nodes 2 and 3 have the data
```

### Geo-distribution

You can tell CockroachDB: "keep US users' data in US datacentres, EU users' data in EU datacentres." This is called **geo-partitioning** — data residency compliance built in.

```
user_id 1–1M  →  stored in US-East
user_id 1M–2M →  stored in EU-West
user_id 2M–3M →  stored in Asia-Pacific
```

### When to Use CockroachDB

| Use Case | Why |
|---|---|
| **Global fintech apps** | Need ACID for money + need to span multiple regions |
| **Multi-region SaaS** | Data residency laws require data to stay in specific countries |
| **High-availability OLTP** | Can't afford downtime, need SQL, need to scale out |
| **Replacing sharded MySQL** | When manual sharding becomes too complex to manage |

### CockroachDB vs Cassandra

| | CockroachDB | Cassandra |
|---|---|---|
| Transactions | Full ACID ✅ | Limited (lightweight transactions only) |
| Write speed | High (but slower than Cassandra) | Extreme (LSM, no locking) |
| Query flexibility | Full SQL, any column | Must query by partition key |
| Consistency | Strong (Raft) | Tunable (eventual by default) |
| Best for | Financial, compliance, relational data at scale | Time-series, IoT, messaging, feeds |

---

## Category 6: Graph Databases

**Examples:** Neo4j, Amazon Neptune, JanusGraph

### What It Is

Data is stored as **nodes** (entities) and **edges** (relationships between entities). Each node and edge can have properties.

```
(Alice) --[FOLLOWS]--> (Bob)
(Alice) --[LIKES]----> (Post: "System Design Tips")
(Bob)   --[LIKES]----> (Post: "System Design Tips")
(Bob)   --[FRIENDS_WITH]--> (Carol)
(Carol) --[WORKS_AT]----> (Company: "Google")
```

### Why a Separate DB for Graphs?

**In SQL**, finding "friends of friends of Alice who work at Google" requires:

```sql
SELECT u3.name
FROM users u1
JOIN friendships f1 ON u1.id = f1.user_id
JOIN users u2 ON f1.friend_id = u2.id
JOIN friendships f2 ON u2.id = f2.user_id
JOIN users u3 ON f2.friend_id = u3.id
JOIN employments e ON u3.id = e.user_id
JOIN companies c ON e.company_id = c.id
WHERE u1.name = 'Alice'
  AND c.name = 'Google';
```

At Facebook scale (1B users), this multi-level JOIN is **impossibly slow**. Every additional level of relationship multiplies the work.

**In Neo4j**, the same query:

```cypher
MATCH (alice:User {name: "Alice"})-[:FRIENDS_WITH*2]->(person)-[:WORKS_AT]->(c:Company {name: "Google"})
RETURN person.name
```

Graph DBs use **index-free adjacency** — each node directly holds pointers to its neighbour nodes in memory. Traversing one hop = following a pointer. No join, no index scan. Speed is constant regardless of total graph size.

### When to Use Graph DB

| Use Case | Why Graph Wins |
|---|---|
| **Social networks** | "Friends of friends", "people you may know" — multi-hop traversal |
| **Fraud detection** | "Find accounts connected to known fraudulent accounts within 3 hops" |
| **Recommendation engines** | "Users similar to you also liked..." — collaborative filtering |
| **Knowledge graphs** | Google's knowledge graph — entities and relationships at scale |
| **Access control (IAM)** | "Does user X have permission Y through any role chain?" |
| **Route planning** | Shortest path, cheapest path between nodes |

### Graph DB Weakness

- Not suitable for simple key-value or tabular data
- Scaling horizontally is harder than Cassandra
- Not great for bulk analytics (count all nodes, sum all weights)

---

## Write-Heavy vs Read-Heavy: Choosing the Right DB

### Write-Heavy Systems

**Definition:** Far more writes than reads. The bottleneck is ingesting data fast enough.

**Examples:** IoT sensors, clickstream logging, metrics collection, messaging

| Database | Why It's Write-Heavy Friendly |
|---|---|
| **Cassandra** | LSM-tree: writes go to RAM first, flushed sequentially — no random I/O, no locking |
| **Kafka** | Append-only log: sequential disk writes are 100x faster than random writes |
| **InfluxDB** | Purpose-built for time-series: compresses repeated timestamps, batch writes |
| **Redis** | In-memory: writes are microsecond-fast (but limited by RAM size) |

**Real example — Uber GPS tracking:**
```
10M active drivers, each sending GPS ping every 4 seconds
= 2,500,000 writes/second

Need: Cassandra (LSM-tree, distributed, write-optimised)
Avoid: PostgreSQL (B-tree, row locking, single node)
```

---

### Read-Heavy Systems

**Definition:** Far more reads than writes. The bottleneck is serving queries fast enough.

**Examples:** Product catalog, user profiles, news feeds, search results

| Database | Why It's Read-Heavy Friendly |
|---|---|
| **Redis** | In-memory: 1M reads/sec, sub-millisecond latency |
| **PostgreSQL** | B-tree indexes: fast range queries, complex JOINs, query planner optimises reads |
| **Elasticsearch** | Inverted index: full-text search across billions of documents in milliseconds |
| **DynamoDB + DAX** | DAX is an in-memory cache layer: reads served from RAM |
| **CDN** | Not a DB, but caches read-heavy static assets at edge locations |

**Real example — Amazon product page:**
```
1 product page load = 20+ read queries (product info, reviews, recommendations, inventory...)
Writes: only when someone submits a review or makes a purchase

Need: PostgreSQL (product data) + Redis (caching) + Elasticsearch (search)
```

---

### Even Distribution: Why It Matters and Which DBs Handle It

**The problem:** If data is not evenly spread across nodes, some nodes get overloaded while others sit idle. This is called a **hotspot**.

**Example of bad distribution:**
```
Shard by first letter of user name:
  Shard A: users starting with A-F → 40% of all users (overloaded)
  Shard G: users starting with G-M → 35% of all users
  Shard N: users starting with N-Z → 25% of all users (underloaded)
```

**Databases that handle even distribution automatically:**

| Database | How It Distributes Evenly |
|---|---|
| **Cassandra** | Consistent hashing on partition key → each node owns a ring slice; virtual nodes ensure ~equal load |
| **DynamoDB** | Automatic partitioning: AWS splits hot partitions automatically |
| **CockroachDB** | Range-based splitting: automatically splits ranges that get too large or too hot |
| **Redis Cluster** | 16,384 hash slots distributed across nodes; rebalances when nodes added/removed |

**Real example — WhatsApp message storage in Cassandra:**
```
Bad shard key: user_name (A-Z unevenly distributed)
Good shard key: user_id (hash of UUID → uniformly random → even distribution)

Cassandra hashes user_id → places it on the ring
1B users → each of 100 nodes owns ~10M users each
No hotspots, no overloaded nodes
```

---

## Category 7: Search Engines (Inverted Index)

**Examples:** Elasticsearch, Apache Solr, OpenSearch, Meilisearch

### What It Is

A search engine database is built entirely around an **inverted index** — a map of every word to every document containing it. It is purpose-built for one thing SQL can't do efficiently: searching inside text across billions of records in milliseconds.

```
Documents:
  Doc 1: "Cassandra is a distributed wide-column database"
  Doc 2: "Redis is an in-memory key-value store"
  Doc 3: "Cassandra handles write-heavy workloads"

Inverted Index:
  "cassandra"    → [Doc1, Doc3]
  "distributed"  → [Doc1]
  "redis"        → [Doc2]
  "write-heavy"  → [Doc3]
  "in-memory"    → [Doc2]
```

Search "cassandra write" → intersect [Doc1, Doc3] ∩ [Doc3] → **Doc3** returned in milliseconds.

### Why Not Just Use SQL LIKE?

```sql
SELECT * FROM articles WHERE body LIKE '%cassandra%';
-- Full table scan on every row, every character — O(N × M)
-- At 100M articles: takes minutes, not milliseconds
```

Elasticsearch pre-builds the inverted index at write time. At query time it does two hash lookups and an intersection — O(1) per word, regardless of dataset size.

### What Elasticsearch Can Do That SQL Cannot

- **Full-text search with relevance scoring** — results ranked by how well they match
- **Fuzzy search** — `"cassndra"` still finds `"cassandra"` (typo tolerance)
- **Faceted search** — "show me hotels, filtered by price + stars + location simultaneously"
- **Autocomplete / prefix search** — `"sys"` → suggests `"system design"`, `"syscall"`
- **Geo-distance search** — "restaurants within 2km", combined with text filters
- **Log aggregation + analytics** — the ELK stack (Elasticsearch + Logstash + Kibana)

### When to Use Elasticsearch

| Use Case | Why |
|---|---|
| **Airbnb / booking search** | Filter by location + dates + price + amenities simultaneously |
| **LinkedIn search** | Search people by name + skills + company + location |
| **Log analysis (ELK stack)** | Index millions of log lines/sec, search by error message, trace ID |
| **E-commerce search** | "red nike shoes size 10" → ranked results with facets |
| **Search autocomplete** | Prefix queries on billions of terms, sub-50ms |
| **GitHub code search** | Full-text across millions of repositories |

### Elasticsearch Weakness

- **Not a primary database** — it is eventually consistent (near-realtime, ~1 second delay after write before searchable)
- **No ACID transactions**
- **Expensive** — storing inverted indexes takes 2-3× more disk than raw data
- **Pattern:** Always write to PostgreSQL/Cassandra first (source of truth), then sync to Elasticsearch for search

```
Write path:  App → PostgreSQL (source of truth)
                 → Elasticsearch (sync via CDC or Kafka)

Read path:   Search query → Elasticsearch (fast)
             Get full record → PostgreSQL (by ID)
```

---

## Category 8: Time-Series Databases

**Examples:** InfluxDB, Prometheus, TimescaleDB, VictoriaMetrics, OpenTSDB

### What It Is

A time-series database is optimised for data that is **always appended with a timestamp** and queried by time range. Think metrics, sensor readings, financial tick data, server CPU usage.

```
Typical time-series data:
  timestamp           | metric          | value | tags
  2026-05-25 10:00:00 | cpu.usage       | 72.3  | host=web-01, dc=us-east
  2026-05-25 10:00:10 | cpu.usage       | 74.1  | host=web-01, dc=us-east
  2026-05-25 10:00:20 | cpu.usage       | 69.8  | host=web-01, dc=us-east
  2026-05-25 10:00:00 | memory.used_gb  | 6.2   | host=web-01, dc=us-east
```

### Why Not Cassandra or PostgreSQL?

You could use Cassandra for time-series (and many do). But dedicated time-series DBs offer:

**1. Time-aware compression**
Timestamps that differ by 10 seconds can be stored as a delta (just store `+10`, `+10`, `+10` instead of full timestamps). Values that change slowly compress similarly. InfluxDB achieves **10–100× compression** over raw storage.

```
Raw:     1748170800, 72.3
         1748170810, 74.1     → 10 bytes each × millions of rows = gigabytes
         1748170820, 69.8

Compressed: base=1748170800, deltas=[+10, +10], values=[72.3, +1.8, -4.3]
          → 70% smaller
```

**2. Built-in time functions**
```sql
-- TimescaleDB (PostgreSQL extension)
SELECT time_bucket('5 minutes', time) AS bucket,
       AVG(value) AS avg_cpu
FROM metrics
WHERE metric = 'cpu.usage'
  AND time > NOW() - INTERVAL '1 hour'
GROUP BY bucket
ORDER BY bucket;
```
`time_bucket`, `first()`, `last()`, `rate()` — all time-aware functions built in.

**3. Automatic data retention (TTL per time range)**
Keep raw data for 7 days, hourly aggregates for 90 days, daily aggregates for 5 years — automatically. No manual cleanup jobs.

### When to Use Time-Series DB

| Use Case | Why |
|---|---|
| **Server metrics monitoring** (Prometheus) | CPU/memory/latency per host, alert on threshold breach |
| **IoT sensor data** (InfluxDB) | Millions of devices sending readings every second |
| **Application performance monitoring (APM)** | Request rates, error rates, p99 latency over time |
| **Financial tick data** | Stock prices every millisecond — 10TB/day per exchange |
| **Ad click aggregation** | Count clicks per ad per minute, aggregate over time |

### Prometheus vs InfluxDB

| | Prometheus | InfluxDB |
|---|---|---|
| Pull vs Push | **Pull** — scrapes metrics endpoints every 15s | **Push** — services push metrics to it |
| Storage | Local only (no native clustering) | Distributed, cloud-native |
| Query language | PromQL | Flux / InfluxQL |
| Best for | Kubernetes monitoring, alerting | General time-series, IoT |
| Alert manager | Built-in (Alertmanager) | External (Grafana) |

---

## Category 9: Data Warehouses (OLAP)

**Examples:** BigQuery, Amazon Redshift, Snowflake, ClickHouse, Apache Hive

### OLTP vs OLAP — The Key Distinction

| | OLTP (Operational) | OLAP (Analytical) |
|---|---|---|
| **Purpose** | Day-to-day transactions | Historical analysis, reporting |
| **Query pattern** | Many small queries (insert/update/select 1 row) | Few huge queries (scan millions of rows, aggregate) |
| **Optimised for** | Write throughput, low latency | Read throughput, complex aggregations |
| **Examples** | PostgreSQL, Cassandra, MySQL | BigQuery, Redshift, Snowflake |
| **Row vs Column** | Row-oriented storage | **Column-oriented storage** |

### Why Column-Oriented Storage?

**Row storage (PostgreSQL):**
```
Row 1: [id=1, name="Alice", age=28, country="US", revenue=500]
Row 2: [id=2, name="Bob",   age=35, country="UK", revenue=750]
Row 3: [id=3, name="Carol", age=22, country="US", revenue=300]
```

For `SELECT country, SUM(revenue) GROUP BY country`, you read ALL columns of ALL rows — even `name` and `age` which you don't need.

**Column storage (BigQuery/Redshift):**
```
column "country": ["US", "UK", "US", ...]
column "revenue": [500, 750, 300, ...]
column "name":    ["Alice", "Bob", "Carol", ...]  ← not read at all
column "age":     [28, 35, 22, ...]               ← not read at all
```

Only the `country` and `revenue` columns are read off disk. 10× less I/O. 10× faster aggregation. Also compresses beautifully (repeated "US" values → store once with a bitmap).

### Real Example — Uber Analytics

```
OLTP (operational):
  PostgreSQL → store each trip (driver, rider, fare, GPS route)
  Cassandra  → store real-time location pings

OLAP (analytical):
  Redshift / BigQuery → answer questions like:
    "What is average fare per city per hour for the last 6 months?"
    "Which drivers have the most cancellations on Friday nights?"
    "Which routes are most profitable in rainy weather?"

These queries scan billions of rows — totally wrong for PostgreSQL.
```

### Data Pipeline Pattern

Data does NOT go directly from app to data warehouse. The standard pipeline:

```
App → PostgreSQL/Cassandra (OLTP, source of truth)
          ↓  (ETL / CDC via Kafka)
      Data Warehouse (OLAP, analytics)
          ↓
      BI tools (Tableau, Looker, Grafana)
```

### When to Use Data Warehouse

| Use Case | Why |
|---|---|
| **Business intelligence / dashboards** | "Revenue by region last quarter" — scans 500M rows |
| **A/B test analysis** | Compare metrics across millions of user sessions |
| **Fraud pattern analysis** | Historical pattern matching across years of transactions |
| **Recommendation model training** | Export user behaviour data for ML |
| **Compliance reporting** | Aggregate all transactions for regulators |

---

## Category 10: Object / Blob Storage

**Examples:** Amazon S3, Google Cloud Storage, Azure Blob, MinIO

### What It Is

Not a traditional database at all — it is a file system for the internet. Stores unstructured binary data (images, videos, PDFs, backups, ML model weights) as objects identified by a key.

```
Key:   "profile-images/user_42/avatar.jpg"
Value: [binary JPEG data — 150KB]

Key:   "videos/upload_789/raw.mp4"
Value: [binary MP4 data — 2GB]
```

### Why Not Store Files in PostgreSQL?

```
PostgreSQL BLOB column:
  Stores binary in rows → row size explodes
  Backup size = DB size + all files
  Can't serve files directly to browsers (no CDN integration)
  Replication copies file data = expensive

S3:
  Designed for files — stores petabytes cheaply ($0.023/GB/month)
  Direct URL serving → integrate with CloudFront CDN
  Multipart upload for large files
  Lifecycle policies: auto-move old files to cheaper storage tiers
  99.999999999% (11 nines) durability
```

### Access Pattern

```
Upload: App → S3 (direct PUT via presigned URL, bypasses your server)
Download: Browser → CloudFront CDN → S3 (served at edge, not from your app)
```

### When to Use Object Storage

| Use Case | Why |
|---|---|
| **User profile photos, post images** | Cheap storage, CDN delivery, not in your DB |
| **Video storage (YouTube/Netflix)** | Multi-GB files — only S3/GCS handles this at scale |
| **Database backups** | Automated snapshots stored cheaply in S3 |
| **ML model weights** | Large binary files shared between training and serving |
| **Static website assets** | JS, CSS, images served via CloudFront |
| **Data lake** (raw event logs) | Dump raw data cheaply, query with Athena/BigQuery |

---

## Updated Quick Reference: Which DB for Which System

| System to Design | Primary DB | Why |
|---|---|---|
| **Twitter/Instagram feed** | Cassandra | Write-heavy, read by user timeline |
| **WhatsApp/Slack messages** | Cassandra | Billions of messages/day, read by conversation |
| **Uber driver locations** | Cassandra + Redis | Cassandra for history, Redis for current position |
| **Netflix viewing history** | Cassandra | Write every viewing event, read by user ID |
| **Google Maps routes** | Neo4j / custom graph | Road network = graph, shortest path traversal |
| **Amazon product catalog** | DynamoDB or PostgreSQL | Read-heavy, structured, fast lookup by ID |
| **Banking / payments** | PostgreSQL / CockroachDB | ACID critical |
| **Search (Airbnb, LinkedIn)** | Elasticsearch | Full-text search, filters, facets |
| **Rate limiting** | Redis | Atomic counter per user, TTL auto-expiry |
| **Session storage** | Redis | Fast lookup by session ID |
| **IoT sensor data** | Cassandra / InfluxDB | Time-series, massive write throughput |
| **Server metrics / APM** | Prometheus / InfluxDB | Time-series, alerting, retention policies |
| **Business analytics / BI** | BigQuery / Redshift | OLAP, column-oriented, scan billions of rows |
| **User images / videos** | S3 + CloudFront | Cheap blob storage + CDN delivery |
| **Log analysis** | Elasticsearch (ELK) | Full-text search across millions of log lines |
| **Fraud detection** | Neo4j | Relationship traversal — connected to fraudsters? |
| **Autocomplete / typeahead** | Elasticsearch / Redis Trie | Prefix search at sub-50ms |
| **Caching layer** | Redis | In-memory, 1M ops/sec, universal |

---

## Summary: One-Line Decision Guide

```
Need ACID + complex queries + moderate scale?          → PostgreSQL
Need ACID + horizontal scale + multi-region?           → CockroachDB
Need extreme write throughput + time-series data?      → Cassandra
Need microsecond reads + caching + pub/sub?            → Redis
Need flexible schema + nested documents?               → MongoDB
Need full-text search + relevance ranking?             → Elasticsearch
Need relationship traversal + graph queries?           → Neo4j
Need simple key-value at AWS scale?                    → DynamoDB
Need metrics + monitoring + alerting?                  → Prometheus / InfluxDB
Need analytics on billions of rows (OLAP)?             → BigQuery / Redshift / Snowflake
Need to store images, videos, large files?             → S3 / Object Storage
```

---

## Interview Q&A

**Q: Why would you choose Cassandra over PostgreSQL for a messaging app?**
A: A messaging app like WhatsApp handles billions of messages per day — that is millions of writes per second. PostgreSQL uses a B-tree index which requires random disk writes and row-level locking for ACID. Under millions of concurrent writes, lock contention becomes a bottleneck. Cassandra's LSM-tree writes to memory first and flushes sequentially to disk — no locking, no random I/O. It also scales linearly by adding nodes. The trade-off: no JOINs and no multi-partition ACID, which is acceptable for messaging where each query is just "get all messages for conversation X."

**Q: When would you pick CockroachDB over Cassandra?**
A: When you need both horizontal scale AND ACID transactions. Cassandra gives you scale but sacrifices strong consistency. CockroachDB uses Raft consensus to guarantee ACID across distributed nodes. Choose CockroachDB for fintech, payments, or any system where data correctness is non-negotiable but you've outgrown a single PostgreSQL instance. The trade-off: CockroachDB's write throughput is lower than Cassandra because Raft consensus adds latency.

**Q: Why does Netflix use Cassandra for viewing history?**
A: Netflix has 200M+ users, each generating continuous viewing events (pause, resume, seek, episode end). That is hundreds of millions of writes per day. The access pattern is always "get all viewing history for user X" — which maps perfectly to Cassandra's partition key model. Cassandra's write throughput handles the volume, the data model fits the access pattern, and it scales horizontally as Netflix grows.

**Q: What is a hotspot and how do you prevent it?**
A: A hotspot is when one node receives disproportionately more traffic than others. Caused by a bad shard key (e.g., shard by date → today's shard gets all writes) or a popular entity (celebrity user). Prevention: choose a high-cardinality, uniformly distributed shard key (like a UUID hash). Use consistent hashing with virtual nodes so load spreads evenly. For celebrity hotspots: add a random suffix to split the hot key across sub-shards.

**Q: Why use Redis as a cache in front of PostgreSQL?**
A: PostgreSQL reads require disk I/O and query planning — typically 1–10ms. Redis stores data in RAM — reads take 0.1–0.5ms at 1M ops/sec. Redis caches repeated query results with a TTL. A 99% cache hit rate means 99% of queries never touch PostgreSQL. Dramatically reduces DB load and latency.

**Q: Why use Elasticsearch alongside PostgreSQL instead of just PostgreSQL?**
A: PostgreSQL's `LIKE '%keyword%'` does a full table scan — O(N) for every search. At 100M records this takes minutes. Elasticsearch pre-builds an inverted index: each word maps to documents containing it. Searching "cassandra write-heavy" does two hash lookups and an intersection — milliseconds regardless of dataset size. The pattern: PostgreSQL is the source of truth, Elasticsearch is synced via Kafka/CDC for the search use case only.

**Q: What is the difference between OLTP and OLAP databases?**
A: OLTP (Online Transaction Processing) databases like PostgreSQL and Cassandra are optimised for many small, fast read/write operations — inserting an order, updating a balance, fetching a user profile. OLAP (Online Analytical Processing) databases like BigQuery and Redshift are optimised for few, massive read queries — scanning billions of rows and computing aggregations for business intelligence. OLAP uses column-oriented storage: for `SELECT country, SUM(revenue)`, only the country and revenue columns are read off disk, not all columns. This makes aggregation queries 10–100× faster than row-oriented storage.

**Q: Why do we store images/videos in S3 instead of the database?**
A: Storing binary blobs in a database bloats row sizes, explodes backup sizes, makes replication expensive (binary data copies with every replica sync), and the DB cannot serve files via CDN. S3 is purpose-built for large binary objects — it costs $0.023/GB/month, integrates natively with CloudFront CDN so files are served from edge nodes near users, supports multipart upload for large files, and provides 11 nines of durability. The DB stores only the S3 URL (a small string), not the file itself.
