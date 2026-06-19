# Elasticsearch: Examples and Tradeoffs

## The Scenario

You are building search for a large e-commerce app:
- 100 million products in catalog
- 2 million search requests/minute at peak
- users expect typo-tolerant search + filters + relevance ranking

Typical user query:
- "wireless sony headphones"
- filters: in stock, price <= 300, rating >= 4

What happens if we do this directly on primary SQL with LIKE and many joins?

---

## Without Elasticsearch (What Breaks)

### Scenario 1: SQL LIKE + Wildcards at Scale

```sql
SELECT *
FROM products
WHERE title ILIKE '%wireless%'
  AND title ILIKE '%sony%'
  AND title ILIKE '%headphones%'
  AND in_stock = true
  AND price <= 300
  AND rating >= 4.0
ORDER BY popularity DESC
LIMIT 20;
```

Outcome:
- high CPU due to text scans
- poor relevance quality
- expensive under heavy concurrent traffic
- primary DB gets overloaded and impacts core transactions

### Scenario 2: Search on Source-of-Truth DB Only

Outcome:
- checkout/order writes compete with search reads
- latency spikes for both search and transactions
- hard to add features like facets/autocomplete/fuzzy matching

---

## Correct Approach: Specialized Search Layer

Use Elasticsearch as read-optimized search index.
Keep SQL/NoSQL database as durable source of truth.

### Index Example

```json
{
  "product_id": "p123",
  "title": "Sony WH-1000XM5 Wireless Headphones",
  "brand": "Sony",
  "price": 299.99,
  "rating": 4.7,
  "in_stock": true,
  "category": "audio"
}
```

### Query Example

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "wireless sony headphones" } }
      ],
      "filter": [
        { "term": { "in_stock": true } },
        { "range": { "price": { "lte": 300 } } },
        { "range": { "rating": { "gte": 4.0 } } }
      ]
    }
  },
  "aggs": {
    "brands": { "terms": { "field": "brand.keyword" } },
    "categories": { "terms": { "field": "category.keyword" } }
  }
}
```

---

## End-to-End Flow (Start to End)

```
1. Product is created/updated in primary DB
2. Change event emitted to Kafka
3. Indexer service consumes event
4. Indexer transforms document and upserts into Elasticsearch
5. Elasticsearch refresh makes document searchable (near real-time)
6. User search API queries Elasticsearch
7. Search results returned with relevance + filters + facets
8. Product detail/checkout re-validate critical fields from primary DB
```

Why step 8 matters:
- Elasticsearch is eventually consistent.
- Final money/stock correctness must come from source-of-truth DB.

---

## Architecture Under Load

```
Users
  -> API Gateway
    -> Search API
      -> Elasticsearch Cluster (search reads)

Product/Admin updates
  -> Primary DB
  -> Kafka events
  -> Indexer workers
  -> Elasticsearch updates
```

Cluster components:
- primary shards for data partitioning
- replicas for high availability and read scale
- coordinator nodes for query fanout + merge

---

## Throughput and Latency (Practical)

### Read Throughput (Search)

- Simple query on indexed field: 10-50 ms per query
- Complex bool with multiple filters and aggregations: 50-500 ms
- Leading wildcard query (for example *wireless*): very slow, can block threads

How throughput scales:
- Each shard executes query independently
- Coordinator merges results from all shards
- More shards = more parallelism, but also more merge cost

Example:
```
10 shards, 5 nodes
Query hits all shards in parallel
Each shard returns top 10 results
Coordinator merges 100 results, returns top 10

If 1 shard is slow (hot or imbalanced), total query waits for it
```

### Write Throughput (Indexing)

- Single document index: 5-20 ms
- Bulk indexing (1000 docs/batch): much higher total throughput
- Refresh interval (default 1 second): controls when new docs become searchable

Example:
```
Without bulk:
  1000 individual writes = 1000 * 15ms = 15 seconds

With bulk API:
  1 bulk request, 1000 docs = 200ms total
  Result: 75x better throughput
```

### Oversharding Problem

More shards seems better but is actually harmful beyond a point.

Why:
- Every query goes to every shard (scatter and gather)
- Coordinator must merge results from all of them
- Too many shards = high coordination overhead even for simple queries

Practical rule:
- Keep each shard 10-50 GB
- For 100 GB index: 3-5 shards is fine
- For 1 TB index: 20-50 shards

Example of oversharding impact:
```
Scenario: 100 GB data split into 500 shards (2 GB each)
Effect:   Every query hits 500 shards, coordinator merges 5000 results
Latency:  Goes from 20ms to 300ms for same query
Fix:      Reduce to 20 shards, latency drops back to 25ms
```

### Hot Shard Problem

If many documents have the same routing key (for example same user_id), they all land on one shard, creating a hotspot.

```
Normal: 10 shards, traffic evenly distributed
Hot:    1 shard receives 80% of queries due to bad routing

Result:
- Hot shard CPU: 90%
- Other shards: 10%
- Overall cluster seems healthy but one shard bottlenecks all queries
```

Fix: use custom routing or increase shard count with better key distribution.

### Refresh Interval Tradeoff

Lower refresh interval = faster search visibility but higher write overhead.

```
Default: refresh_interval = 1s
Result: docs visible after ~1 second, moderate write throughput

Set: refresh_interval = 30s
Result: higher write throughput, docs delayed 30s visibility

Set: refresh_interval = -1 (disabled during bulk load)
Result: maximum ingest throughput, use for initial data load
```

### Summary: Throughput Tradeoffs

| Factor | Pros | Tradeoffs |
|--------|------|-----------|
| More shards | Better read parallelism | Higher coordination cost, merge overhead |
| Fewer large shards | Lower merge cost | Less parallelism, slower individual queries at scale |
| Low refresh interval | Near-realtime visibility | Higher write cost, more segment merges |
| High refresh interval | Higher write throughput | Delayed search visibility |
| Bulk API for writes | Much higher indexing throughput | Adds batching logic in app layer |
| Replicas | Better read throughput and availability | More storage, higher indexing overhead |

---

## Detailed Tradeoff Analysis

| Option | Search Quality | Transaction Safety | Latency at Scale | Complexity |
|--------|----------------|--------------------|------------------|------------|
| SQL only | Medium | Strong | High under load | Low |
| Elasticsearch only as source of truth | High | Weak for transactions | Good for search | Medium |
| SQL + Elasticsearch (recommended) | High | Strong | Good | High |

---

## Common Doubt: Why Not Write Directly to Elasticsearch Only?

**Question:**
If Elasticsearch search is fast, why not store everything there and skip SQL?

**Answer:**
Because core business flows need stronger transactional guarantees than Elasticsearch is designed for.

Use pattern:
1. DB is source of truth for writes and critical correctness
2. Elasticsearch is derived index for search reads

---

## Common Pitfalls

1. Using analyzed text field for exact filters (use keyword fields)
2. Too many tiny shards (high coordination overhead)
3. No index versioning/reindex plan for mapping changes
4. Running heavy leading wildcard queries frequently
5. Treating stale search result as final checkout truth

---

## Key Insights

1. Elasticsearch solves search relevance and scale, not transactional correctness.
2. Event-driven indexing decouples write path from search path.
3. Eventual consistency is acceptable for search, not for payment/order finalization.
4. Good mappings and shard strategy are as important as hardware.

---

## Interview Answer (Concise)

Use Elasticsearch as a specialized search index, not primary source of truth. Write product updates to SQL/NoSQL first, stream changes via Kafka to indexer workers, and serve search from Elasticsearch for fast relevance and filtering. Tradeoff is eventual consistency and operational overhead (mappings, shards, reindexing), but this architecture gives both fast search UX and strong transactional correctness.
