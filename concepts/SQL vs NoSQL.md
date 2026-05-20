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

## Why NoSQL is Faster Than SQL

> Short answer: NoSQL isn't always faster. It's faster **for specific workloads** because it trades away features (joins, ACID, schema) that SQL pays a performance cost for.

### Reason 1: No Joins

**SQL — joining 3 tables:**

Imagine fetching a user's profile with their orders and shipping address:

```sql
SELECT u.name, o.amount, a.city
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN addresses a ON u.id = a.user_id
WHERE u.id = 123;
```

The DB must:
1. Scan/index `users` → find row for id=123
2. Scan/index `orders` → find all rows where user_id=123
3. Scan/index `addresses` → find rows where user_id=123
4. Merge (join) all three result sets in memory

Each JOIN is an extra operation. With millions of rows and complex queries, this gets expensive.

**NoSQL (Document DB) — one read:**

MongoDB stores the whole user as one document:
```json
{
  "_id": "123",
  "name": "Alice",
  "orders": [
    { "amount": 50.00, "date": "2024-01-01" },
    { "amount": 30.00, "date": "2024-01-10" }
  ],
  "address": { "city": "NYC" }
}
```

Fetching user 123 = **one disk read**. No joining. No merging. Done.

> This is called **denormalization** — duplicate data to avoid joins. You pay more storage, but save CPU and I/O at read time.

---

### Reason 2: No Schema Enforcement Overhead

SQL validates every write against the schema:
- Is this column the right type?
- Does this foreign key exist?
- Does this NOT NULL constraint pass?
- Does this UNIQUE constraint hold? (requires an index lookup)

```sql
INSERT INTO orders (user_id, amount) VALUES (999, 50.00);
-- DB checks: does user_id=999 exist in users table? (constraint check!)
-- If not → reject with FK violation
```

Every `INSERT` or `UPDATE` may trigger multiple constraint checks, which means extra index lookups.

**NoSQL skips all of this.** Write the document → it's written. No validation, no constraint checks. Faster writes.

---

### Reason 3: Simpler Data Model = Faster Lookups

**SQL key-value lookup:**
```sql
SELECT * FROM sessions WHERE session_id = 'abc123';
```
DB must: parse SQL → plan query → check indexes → B-tree traversal → return row.

**Redis key-value lookup:**
```java
redis.get("session:abc123");  // hash table lookup: O(1)
```
Redis stores data in memory as a hash table. Lookup is a direct memory address jump. No SQL parser, no query planner, no disk I/O.

**Speed difference:** Redis can do **1,000,000 reads/sec**. PostgreSQL might do **10,000–50,000 reads/sec** for the same query.

---

### Reason 4: Horizontal Scaling Without Cross-Shard Joins

**SQL scaled across multiple machines:**

```
Shard 1: users id 1–1M
Shard 2: users id 1M–2M

Query: SELECT * FROM users JOIN orders WHERE users.age > 25
→ Must query BOTH shards
→ Must join results from Shard 1 + Shard 2 in memory
→ Expensive scatter-gather on every query
```

**Cassandra / DynamoDB:**

Data is sharded by the primary key. If you design your access patterns correctly, every query hits **exactly one shard**:

```
partition_key = user_id
→ "give me all messages for user_id=123" → goes to exactly 1 node
→ No scatter, no gather, no merge
```

This is why Cassandra can do **millions of writes/sec** across a cluster while a SQL cluster struggles with cross-shard joins.

---

### Reason 5: Eventual Consistency = No Locking

**SQL with ACID** (strong consistency):

```
Thread 1: BEGIN TRANSACTION
Thread 1: UPDATE accounts SET balance = balance - 100 WHERE id = 1;
          ← row is LOCKED
Thread 2: tries to read/write same row → WAITS
Thread 1: COMMIT → row unlocked
Thread 2: now proceeds
```

Locks ensure correctness but kill throughput under high concurrency.

**Cassandra / DynamoDB** (eventual consistency):

```
Thread 1: write balance = 900
Thread 2: write balance = 950  ← no lock, writes concurrently
→ Both succeed immediately
→ Conflict resolved later by "last write wins" or version vectors
```

No waiting = higher throughput. The trade-off: briefly inconsistent state.

> **Real example:** When you like a tweet, the like count might show 999 for 100ms then jump to 1,000. Nobody cares. But if a bank showed your balance as $500 for 100ms when it's actually $400 — that's a bug. SQL is correct there. NoSQL is fast here.

---

### When SQL is FASTER Than NoSQL

NoSQL is not universally faster:

| Scenario | SQL wins | Why |
|---|---|---|
| Complex reporting (GROUP BY, aggregations) | ✅ SQL | Optimized query planner, columnar stores |
| Ad-hoc queries on unknown fields | ✅ SQL | Flexible `WHERE` on any column |
| Transactions across multiple entities | ✅ SQL | NoSQL multi-document transactions are slow |
| Small dataset, single machine | ✅ SQL | NoSQL overhead isn't worth it |
| Strong consistency required | ✅ SQL | NoSQL pays extra cost to achieve strong consistency |

---

### Summary: Why NoSQL is faster (for its use cases)

| SQL Cost | NoSQL Avoids It By |
|---|---|
| JOINs across tables | Denormalizing data into one document |
| Schema + constraint validation on writes | No schema enforcement |
| Query parsing + planning | Direct key lookup (hash table / B-tree on one value) |
| Row locking for ACID | Eventual consistency (no locks) |
| Cross-shard scatter-gather | Partition key routes to one node |
| Disk-based storage | In-memory (Redis) |

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

**Q: Why is NoSQL faster than SQL for high-scale reads?**  
A: Several reasons: (1) No joins — document DBs store all related data together so a single read returns everything. (2) No schema validation overhead on writes. (3) Key-value stores like Redis use in-memory hash tables with O(1) lookup vs SQL's B-tree index traversal + query planner. (4) No row locking — eventual consistency means no waiting for locks under concurrent writes.

**Q: Is NoSQL always faster than SQL?**  
A: No. SQL can be faster for complex analytics (GROUP BY, aggregations) because its query planner optimizes execution. For small datasets on a single machine, SQL's overhead is negligible. NoSQL is faster specifically for high-throughput simple access patterns: key lookups, time-series writes, and horizontally sharded reads where each query hits exactly one partition.

**Q: Why can Cassandra handle millions of writes per second but PostgreSQL can't?**  
A: Cassandra uses an LSM-tree (Log-Structured Merge-tree) — all writes go to an in-memory buffer (memtable) first, then are flushed sequentially to disk. Sequential I/O is 100x faster than random I/O. PostgreSQL uses a B-tree which requires random disk writes and locks for ACID. Also, Cassandra is designed to scale horizontally — 100 nodes each doing 10k writes/sec = 1M writes/sec total. PostgreSQL is primarily vertical.

**Q: What is denormalization and why does it make NoSQL faster?**  
A: Denormalization means storing duplicate/redundant data to avoid joins. In SQL you'd have users + orders in separate tables and JOIN them. In MongoDB you embed orders inside the user document. Reading user + orders = one disk read instead of two reads + a join. The downside: if a user's name changes, you must update it everywhere it's duplicated. The trade-off is write complexity for read speed.

**Q: Why is Redis so much faster than any disk-based database?**  
A: Redis stores everything in RAM. Memory access is ~100 nanoseconds; disk access is ~10 milliseconds — that's 100,000x slower. Redis also uses a single-threaded event loop, avoiding lock contention. For simple get/set operations, Redis achieves 1M+ ops/sec on a single machine. No SQL database can match this for key-value workloads because they're disk-bound by design.

**Q: Why does Cassandra scale horizontally but PostgreSQL doesn't naturally?**  
A: Cassandra was built from the ground up with a distributed, peer-to-peer architecture. Every write goes to a partition determined by a consistent hash of the key — no central coordinator, no cross-node joins required. PostgreSQL assumes a single-node model; horizontal scaling requires external sharding middleware (Citus, Vitess) which adds complexity and limits cross-shard queries. The fundamental difference: NoSQL sacrifices joins and ACID to make horizontal scaling seamless.
