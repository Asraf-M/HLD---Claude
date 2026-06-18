# Elasticsearch

## What Is It?

**Elasticsearch** is a distributed search and analytics engine built on top of Apache Lucene.

It is designed for:
- Full-text search (Google-like search over large text)
- Fast filtering and aggregations
- Near real-time indexing and querying
- Horizontal scaling across many nodes

> **Simple view:** If a relational DB is great for exact lookups (`id = 123`), Elasticsearch is great for search-like queries such as:  
> "find all products related to wireless noise-cancelling headphones under $300 sorted by relevance."

---

## Why Not Just Use SQL `LIKE`?

You *can* search text with SQL, but it gets expensive and limited at scale.

| | SQL (LIKE / ILIKE) | Elasticsearch |
|--|---------------------|---------------|
| Full-text relevance ranking | Limited | ✅ Built-in (BM25 scoring) |
| Typo tolerance | Hard | ✅ Fuzzy queries |
| Autocomplete | Complex | ✅ Easy with analyzers/suggester |
| Faceted filters (brand, price, rating) | Expensive joins/group by | ✅ Fast aggregations |
| Scale to billions of docs | Hard | ✅ Native sharding |
| Real-time analytics | Not ideal | ✅ Optimized |

Use SQL for transactions and source-of-truth data. Use Elasticsearch as a search layer.

---

## Core Concepts

### 1. Index, Document, Field
- **Index**: Similar to a database/table namespace
- **Document**: A JSON object
- **Field**: Key inside a document

Example document:
```json
{
  "product_id": "p123",
  "title": "Sony WH-1000XM5 Wireless Headphones",
  "brand": "Sony",
  "price": 299.99,
  "rating": 4.7,
  "in_stock": true
}
```

### 2. Inverted Index (The Main Idea)
Instead of storing document-by-document for search, Elasticsearch stores term-to-doc mappings.

```
Term        -> Document IDs
"wireless" -> [1, 5, 8, 10]
"sony"     -> [1, 6, 20]
"headphone"-> [1, 3, 7, 15]
```

When user searches "sony wireless headphone", ES intersects/scorers these postings efficiently.

### 3. Analyzer Pipeline
On indexing and query time, text is processed:
1. Character filters (cleanup)
2. Tokenizer (split into terms)
3. Token filters (lowercase, stemming, stop-word removal)

Example:
```
Input: "Running Shoes for Men"
Tokens: ["running", "shoes", "for", "men"]
After filters: ["run", "shoe", "men"]
```

---

## How Data Flows

### Write Path (Indexing)
1. Client sends JSON document to index API
2. ES routes doc to a primary shard (hash on `_id`)
3. Primary shard indexes doc into Lucene segment
4. Changes replicated to replica shards
5. After refresh (default ~1s), doc becomes searchable

### Read Path (Searching)
1. Coordinator node receives query
2. Broadcasts to relevant shards
3. Each shard executes locally, returns top-k hits
4. Coordinator merges/sorts results and returns final response

---

## Query DSL Basics

### Match Query (full text)
```json
GET /products/_search
{
  "query": {
    "match": {
      "title": "wireless sony headphones"
    }
  }
}
```

### Bool Query (must/filter)
```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "wireless headphones" } }
      ],
      "filter": [
        { "term": { "brand.keyword": "Sony" } },
        { "range": { "price": { "lte": 300 } } },
        { "term": { "in_stock": true } }
      ]
    }
  }
}
```

### Aggregation (facets)
```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "brands": { "terms": { "field": "brand.keyword" } },
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [{"to":100}, {"from":100, "to":300}, {"from":300}]
      }
    }
  }
}
```

---

## Shards and Replicas

### Primary Shards
- Split index data for horizontal scale
- Example: 100GB index with 5 primaries -> ~20GB per shard

### Replica Shards
- Copies of primary shards
- Improve read throughput and availability

If a node with primary shard fails, replica is promoted to primary.

---

## Near Real-Time (NRT) Behavior

Elasticsearch is near real-time, not strictly real-time:
- Indexed doc is not immediately searchable
- Search visibility after refresh interval (default ~1 second)

Important operations:
- **refresh**: Makes recent changes searchable
- **flush**: Commits segments and clears transaction log
- **merge**: Combines smaller segments for performance

---

## Elasticsearch in System Design

### Typical Architecture Pattern
```
App writes -> Primary DB (PostgreSQL/MySQL)
           -> Event stream (Kafka)
           -> Indexer service updates Elasticsearch

Search reads -> Elasticsearch
Critical reads/writes -> Primary DB
```

Why this pattern?
- DB remains source of truth
- Elasticsearch optimized for search workloads
- Event-driven sync keeps index eventually consistent

---

## Common Pitfalls

1. **Using analyzed field for exact filters**  
Use `brand.keyword` for exact match, not `brand` analyzed text.

2. **Too many shards**  
Oversharding wastes memory and slows cluster operations.

3. **Treating ES as source of truth**  
ES is not ideal for strict transactional guarantees.

4. **Heavy wildcard leading queries (`*term`)**  
Can be very expensive and slow.

5. **Ignoring mapping design**  
Bad mappings lead to poor relevance and high storage costs.

---

## Elasticsearch vs Other Options

| Tool | Best For | Not Best For |
|------|----------|--------------|
| Elasticsearch | Full-text search + aggregations | Strict OLTP transactions |
| PostgreSQL FTS | Moderate search in existing DB | Large-scale, complex search |
| OpenSearch | Similar to Elasticsearch (fork) | Same limitations as ES |
| Redis Search | Fast in-memory search | Very large datasets (memory-heavy) |

---

## Quick Setup Sketch

```bash
# Create index with mapping
PUT /products
{
  "mappings": {
    "properties": {
      "title": { "type": "text" },
      "brand": { "type": "keyword" },
      "price": { "type": "float" },
      "rating": { "type": "float" },
      "in_stock": { "type": "boolean" }
    }
  }
}

# Index document
POST /products/_doc/p123
{
  "title": "Sony WH-1000XM5 Wireless Headphones",
  "brand": "Sony",
  "price": 299.99,
  "rating": 4.7,
  "in_stock": true
}
```

---

## Interview Q&A

**Q: Why use Elasticsearch instead of querying SQL directly?**  
A: SQL is great for exact lookups and transactions but weak for advanced full-text relevance, fuzzy search, and faceted analytics at scale. Elasticsearch uses inverted indexes and BM25 scoring, which makes search much faster and more relevant for text-heavy use cases.

**Q: What is the difference between `text` and `keyword` field types?**  
A: `text` fields are analyzed/tokenized for full-text search (e.g., title, description). `keyword` fields are stored as-is for exact match, sorting, and aggregations (e.g., brand, status, IDs).

**Q: How does Elasticsearch scale horizontally?**  
A: By sharding indexes across nodes. Each query runs in parallel on shards, and the coordinator merges results. Replicas provide failover and improve read throughput.

**Q: Is Elasticsearch strongly consistent?**  
A: Not in the same way as transactional RDBMS. It's near real-time and typically eventually consistent for search visibility due to refresh intervals and distributed replication. For strict correctness, keep primary DB as source of truth.

**Q: How do you keep Elasticsearch in sync with primary DB?**  
A: Use CDC/events (Kafka, Debezium) and an indexer service. On insert/update/delete in DB, emit events and apply corresponding index updates. Use retry + dead-letter queue for resiliency.

**Q: What are the main performance levers in Elasticsearch?**  
A: Good mapping design, right shard count, using filters instead of scoring when possible, avoiding expensive wildcard queries, proper refresh interval tuning, and controlling field cardinality for aggregations.
