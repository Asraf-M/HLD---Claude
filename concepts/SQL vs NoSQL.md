# SQL vs NoSQL

## The Core Question

> "Should I use a relational database (SQL) or a non-relational database (NoSQL)?"

This is one of the most common decisions in system design. The answer depends on your data shape, access patterns, and scale requirements.

---

## SQL (Relational Databases)

**Examples:** PostgreSQL, MySQL, Oracle, SQL Server

### What it looks like

Data is stored in **tables** with rows and columns. Tables relate to each other via foreign keys.

```sql
users:          orders:
id | name       id | user_id | amount
1  | Alice      101 | 1      | 50.00
2  | Bob        102 | 1      | 30.00
                103 | 2      | 75.00
```

### Key Properties

- **Schema enforced** — every row must follow the defined structure
- **ACID transactions** — Atomicity, Consistency, Isolation, Durability
- **Joins** — query across related tables in one statement
- **Strong consistency** — after a write, all reads see the new value

### ACID Explained Simply

| Property | Meaning | Analogy |
|----------|---------|---------|
| **Atomicity** | All or nothing — transaction either fully completes or fully rolls back | Bank transfer: debit + credit both happen, or neither |
| **Consistency** | Data always moves from one valid state to another | Can't create an order for a non-existent user |
| **Isolation** | Concurrent transactions don't see each other's partial work | Two people booking last seat — only one wins |
| **Durability** | Committed data survives crashes | After "payment confirmed", data is on disk |

### When to Use SQL

- Financial systems (payments, banking) — need ACID
- User accounts, profiles — structured, relational
- E-commerce (orders, products, inventory) — transactions
- Reporting/analytics with complex queries (JOINs, GROUP BY)
- Data that fits well into a table structure

---

## NoSQL (Non-Relational Databases)

**Not one thing** — four major types:

### 1. Key-Value Stores
**Examples:** Redis, DynamoDB, Memcached

```
"user:123" → { name: "Alice", age: 30 }
"session:abc" → { userId: 123, expires: 1748000000 }
```

- Simplest model: key maps to a blob of data
- Extremely fast (O(1) lookup)
- **Use for:** caching, sessions, rate limiting counters, shopping carts

### 2. Document Stores
**Examples:** MongoDB, CouchDB, Firestore

```json
{
  "_id": "post_456",
  "title": "Hello World",
  "author": { "id": 1, "name": "Alice" },
  "tags": ["tech", "beginner"],
  "comments": [
    { "user": "Bob", "text": "Great post!" }
  ]
}
```

- Stores JSON-like documents; schema is flexible per document
- Good for hierarchical or nested data
- **Use for:** CMS, catalogs, user profiles with varying fields, event logs

### 3. Wide-Column Stores
**Examples:** Apache Cassandra, Google Bigtable, HBase

```
Row key: "user123"
Columns: { "email": "alice@gmail.com", "age": "30", "city": "NYC" }

Row key: "user456"
Columns: { "email": "bob@gmail.com" }   ← different columns per row OK
```

- Each row can have different columns
- Optimized for time-series, write-heavy workloads
- **Use for:** IoT sensor data, activity feeds, messaging, metrics

### 4. Graph Databases
**Examples:** Neo4j, Amazon Neptune

```
(Alice) --[FOLLOWS]--> (Bob)
(Alice) --[LIKES]--> (Post)
(Bob)   --[LIKES]--> (Post)
```

- Stores entities (nodes) and relationships (edges)
- Extremely fast for traversal queries
- **Use for:** Social networks, recommendation engines, fraud detection

---

## Head-to-Head Comparison

| | SQL | NoSQL |
|--|-----|-------|
| **Schema** | Fixed (enforced) | Flexible (schema-less) |
| **Scaling** | Vertical (bigger machine) | Horizontal (more machines) |
| **Transactions** | Full ACID | Usually BASE (eventually consistent) |
| **Joins** | Yes — powerful | Limited/no joins |
| **Query language** | SQL (standard) | Varies by DB |
| **Consistency** | Strong by default | Tunable (eventual → strong) |
| **Best for** | Structured, relational data | Unstructured, high-volume, varied |

### BASE (NoSQL consistency model)

- **B**asically **A**vailable — system is usually available
- **S**oft state — data may change over time without input
- **E**ventually consistent — after some time, all replicas converge

> **Analogy:** A bank (SQL) processes your transaction and gives you an exact balance immediately. A social media like count (NoSQL) might show 999 likes for 200ms before converging to 1,000.

---

## Scaling Differences

### SQL: Vertical Scaling (Scale Up)
Add CPU/RAM to one machine. Has limits. Expensive.

For horizontal: read replicas + sharding (hard to do manually, needs middleware).

### NoSQL: Horizontal Scaling (Scale Out)
Add more machines. Designed for it. Consistent hashing distributes data automatically.

```
Cassandra ring: 
node1 → owns keys 0-25%
node2 → owns keys 25-50%
node3 → owns keys 50-75%
node4 → owns keys 75-100%
Add node5 → each node gives up a slice automatically
```

---

## Decision Guide

```
Does your data have complex relationships (foreign keys, joins)?
  YES → SQL

Does your data need ACID transactions?
  YES → SQL (PostgreSQL, MySQL)

Do you need to scale to billions of rows across many machines?
  YES → NoSQL (Cassandra, DynamoDB)

Is your schema changing rapidly / data structure varies per record?
  YES → NoSQL (MongoDB)

Do you need very fast reads/writes with simple key-based access?
  YES → Key-Value (Redis, DynamoDB)

Are you storing time-series or append-heavy data?
  YES → Wide-column (Cassandra, InfluxDB)

Do you need to traverse relationships between entities?
  YES → Graph DB (Neo4j)
```

---

## Real-World Usage Examples

| Company | SQL | NoSQL |
|---------|-----|-------|
| **Airbnb** | PostgreSQL (listings, users, bookings) | — |
| **Twitter** | MySQL (tweets, users) | Redis (timelines, caching) |
| **Netflix** | MySQL (billing) | Cassandra (viewing history) |
| **Instagram** | PostgreSQL (user data) | Cassandra (activity feeds) |
| **Uber** | MySQL (trips, payments) | Schemaless (location data) |

Most large systems use **both** — SQL for transactional data, NoSQL for high-scale read/write paths.

---

## Interview Q&A

**Q: When would you choose SQL over NoSQL?**  
A: When you need ACID transactions (financial operations, inventory), complex queries with JOINs, or well-structured relational data that won't change shape much. Also when the team is more familiar with SQL and scale requirements are moderate.

**Q: When would you choose NoSQL?**  
A: When you need massive horizontal scale (billions of records), flexible schema (different attributes per entity), write-heavy workloads (time-series, events), or simple key-based access patterns. Also when eventual consistency is acceptable.

**Q: Can you use both in the same system?**  
A: Yes, and large companies typically do. PostgreSQL for user accounts + payments (ACID needed). Cassandra for activity feeds, messages, metrics (scale needed). Redis for caching + sessions (speed needed).

**Q: What is eventual consistency?**  
A: After a write, replicas may temporarily return the old value, but within a short time window (milliseconds to seconds), all replicas converge to the new value. Acceptable for likes/views counts. Not acceptable for bank balances.
