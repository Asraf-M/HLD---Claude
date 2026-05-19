# Indexes

## What Is It?

A database **index** is a separate data structure that lets the database find rows quickly **without scanning every row** in a table.

> **Analogy:** A book's index at the back. Instead of reading the entire book to find "photosynthesis", you look it up in the index → "page 147". The index is sorted alphabetically so lookup is O(log N), not O(N).

---

## Why Indexes Matter

Without index on `email` column (1 million users):
```sql
SELECT * FROM users WHERE email = 'alice@gmail.com';
-- DB scans all 1,000,000 rows → ~500ms
```

With index on `email`:
```sql
-- DB binary searches the index → finds row in ~0.1ms
-- 5000× faster
```

---

## The B-Tree Index (Default Index)

The most common index type. Stored as a **balanced tree** of sorted values.

```
              [M]
             /   \
          [E-K]  [S-Z]
         /  |  \   |  \
       [E] [G] [K] [S] [Z]
        ↓   ↓   ↓   ↓   ↓
       rows rows rows rows rows
```

**Properties:**
- Data is kept sorted
- All leaf nodes are at the same depth (balanced)
- Each node holds multiple keys (branching factor ~100-500)

**Supports:**
- **Exact match:** `WHERE email = 'alice@gmail.com'` → O(log N)
- **Range queries:** `WHERE age BETWEEN 20 AND 30` → O(log N + k) where k = matching rows
- **Prefix queries:** `WHERE name LIKE 'Ali%'` → O(log N)
- **ORDER BY** on indexed column → no sort needed, already sorted

**Does NOT support:**
- `WHERE name LIKE '%alice%'` (leading wildcard — can't use sorted order)
- Functions: `WHERE YEAR(created_at) = 2024` (breaks index usage)

---

## The LSM-Tree Index (Write-Optimized)

Used by: **Cassandra, RocksDB, LevelDB, HBase**

Traditional B-trees require random disk writes (updating a node anywhere in the tree) → slow on spinning disks.

**LSM-Tree (Log-Structured Merge-Tree)** makes writes fast by:
1. Write to in-memory buffer (memtable) — extremely fast
2. When buffer full → flush to disk as a sorted immutable file (SSTable)
3. Background compaction merges SSTables → keeps reads manageable

```
Write → Memtable (RAM)
          ↓ (when full)
        SSTable L0 (disk)
          ↓ (compaction)
        SSTable L1 (larger, sorted)
          ↓ (compaction)
        SSTable L2 (even larger)
```

**Reads** must check memtable + multiple SSTable levels → slower than B-tree reads.

**Bloom filter per SSTable** — quickly determine if a key is definitely NOT in an SSTable → skip it.

| | B-Tree | LSM-Tree |
|--|--------|---------|
| Write speed | Medium (random writes) | **Fast** (sequential appends) |
| Read speed | **Fast** (single lookup) | Slower (check multiple levels) |
| Space overhead | Low | Higher (before compaction) |
| Used by | PostgreSQL, MySQL | Cassandra, RocksDB |

---

## Hash Index

Simply: a hash map in memory. `hash(key) → disk offset of row`.

**Supports:** Only exact match — `WHERE id = 42` → O(1)  
**Does NOT support:** Range queries (hash completely scrambles order)

Used internally by PostgreSQL for specific operations, and by many in-memory key-value stores (Redis).

---

## Composite Index (Multi-Column Index)

An index on **multiple columns**.

```sql
CREATE INDEX idx_user_date ON orders (user_id, created_at);
```

The index is sorted by `user_id` first, then `created_at` within each `user_id`.

**Supports (uses index):**
```sql
WHERE user_id = 42                        -- leftmost prefix ✅
WHERE user_id = 42 AND created_at > '2024' -- both columns ✅
WHERE user_id = 42 ORDER BY created_at    -- sort without extra sort ✅
```

**Does NOT use index:**
```sql
WHERE created_at > '2024'  -- skips leftmost column ❌
```

**Left-prefix rule:** A composite index on (A, B, C) can be used for queries on A, (A,B), or (A,B,C) — but NOT B alone or C alone.

---

## Covering Index

An index that **contains all columns needed** by a query — so the DB never has to look up the actual row.

```sql
CREATE INDEX idx_covering ON orders (user_id, status, amount);

SELECT status, amount FROM orders WHERE user_id = 42;
-- All needed columns (status, amount) are IN the index
-- DB doesn't touch the actual table → "index-only scan" → very fast
```

---

## Partial Index

Index only a **subset of rows** matching a condition.

```sql
CREATE INDEX idx_active_users ON users (email) WHERE active = true;
```

Only active users are indexed. Smaller index → faster lookups + less memory.

**Use when:** You always filter on the same condition (active users, unread messages, pending orders).

---

## When Indexes Hurt

**Write performance:** Every insert/update/delete must update all indexes on the table.

```
INSERT into orders:
  1. Write row to table
  2. Update index on (user_id)
  3. Update index on (created_at)
  4. Update index on (status)
  5. Update composite index on (user_id, created_at)
→ 5 writes instead of 1
```

**Over-indexing** on write-heavy tables (e.g., a logs table with 100K inserts/sec) → index maintenance overhead dominates.

**Rule of thumb:**
- Read-heavy tables → index generously
- Write-heavy tables → index only the most critical queries

---

## How to Know if Index is Being Used

### PostgreSQL: EXPLAIN ANALYZE
```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'alice@gmail.com';

-- Good output:
Index Scan using idx_email on users (cost=0.43..8.45 rows=1)
  Index Cond: (email = 'alice@gmail.com')
  Actual time: 0.024ms

-- Bad output (no index used):
Seq Scan on users (cost=0.00..15420.00 rows=1000000)
  Filter: (email = 'alice@gmail.com')
  Actual time: 482ms
```

**Seq Scan** = full table scan = index not used.  
**Index Scan** = index used ✅.

---

## Index Patterns in System Design

### User lookup by email
```sql
CREATE UNIQUE INDEX idx_users_email ON users (email);
-- Also enforces uniqueness
```

### Feed: get user's posts sorted by date
```sql
CREATE INDEX idx_posts_user_date ON posts (user_id, created_at DESC);
-- Composite index: all posts for user_id sorted newest first
```

### Search: find unread messages in a conversation
```sql
CREATE INDEX idx_messages_conv_unread ON messages (conversation_id, is_read)
  WHERE is_read = false;
-- Partial index: only unread messages indexed
```

### Geospatial: find nearby locations
```sql
-- PostGIS
CREATE INDEX idx_places_location ON places USING GIST (location);
-- GiST index supports spatial operators (ST_DWithin, ST_Contains)
```

---

## Interview Q&A

**Q: What is a database index and how does it work?**  
A: An index is a separate data structure (usually a B-tree) that stores a sorted copy of one or more columns alongside pointers to the actual rows. Instead of scanning every row (O(N)), the DB binary-searches the index (O(log N)) to find matching rows. The trade-off: faster reads but slower writes (index must be updated on every insert/update/delete).

**Q: What is the left-prefix rule for composite indexes?**  
A: A composite index on (A, B, C) can be used for queries filtering on A, A+B, or A+B+C — but not B alone or C alone. The index is sorted first by A, then by B within A, then by C within B. Skipping A means the sort order is irrelevant and the index can't be used for efficient lookup.

**Q: What is the difference between B-tree and LSM-tree indexes?**  
A: B-tree: balanced sorted tree, fast reads (O(log N)), moderate write speed (random disk writes to maintain balance). Used by PostgreSQL/MySQL. LSM-tree: write-optimized, all writes are sequential appends to memory then disk. Reads check multiple levels (slower). Used by Cassandra, RocksDB. Choose B-tree for read-heavy OLTP; choose LSM for write-heavy (time-series, event logs).

**Q: When should you NOT add an index?**  
A: On write-heavy tables where index maintenance overhead outweighs read benefit (e.g., a high-throughput events/logs table). On low-cardinality columns (e.g., `gender` with 2 values — the index isn't selective enough to help). On small tables where a full scan is faster than index lookup. When the column is almost never used in WHERE clauses.

**Q: What is a covering index?**  
A: An index that contains all columns needed to answer a query, so the DB never reads the actual table rows (index-only scan). Example: index on (user_id, status, amount) for a query that selects status and amount filtered by user_id. All data is in the index. Much faster than fetching from the table — avoids random I/O.
