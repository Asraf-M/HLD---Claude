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

## How B-Tree Index Works Internally (Step by Step)

### The Setup

You have a `users` table with 7 rows and you create an index on the `age` column:

```
Table (unsorted, stored on disk in insertion order):
RowID | name    | age | email
  1   | Charlie |  35 | charlie@...
  2   | Alice   |  28 | alice@...
  3   | Bob     |  22 | bob@...
  4   | Diana   |  31 | diana@...
  5   | Eve     |  19 | eve@...
  6   | Frank   |  45 | frank@...
  7   | Grace   |  28 | grace@...
```

The B-Tree index on `age` is built separately:

```
Index on age (sorted):

                 [28]
               /      \
          [22]          [35]
         /    \        /    \
     [19]     [22]  [28]   [31, 35, 45]
      ↓         ↓    ↓ ↓    ↓    ↓   ↓
    RowID=5  RowID=3 2,7  RowID=4  1  6
```

Each **leaf node** stores: the index value + a pointer (RowID) back to the actual table row on disk.

---

### Query 1: Exact Match

```sql
SELECT * FROM users WHERE age = 28;
```

**Step-by-step:**
1. Start at root node `[28]`
2. `28 == 28` → go left subtree OR match found at this level
3. Follow pointer → leaf node for age=28 → RowIDs: 2, 7
4. Fetch rows 2 (Alice) and 7 (Grace) directly from disk using RowIDs

**Total:** 3 node reads + 2 row fetches. Compare to full scan: 7 row reads.
At 1 million rows: **3 node reads vs 1,000,000 row reads**.

```
O(log N) vs O(N)
log₂(1,000,000) ≈ 20 node reads
vs 1,000,000 row reads
```

---

### Query 2: Range Query

```sql
SELECT * FROM users WHERE age BETWEEN 25 AND 35;
```

**Step-by-step:**
1. B-tree search for `age = 25` → land on the first leaf ≥ 25 (age=28)
2. **Scan leaf nodes left to right** (leaf nodes are linked): 28 → 31 → 35
3. Stop when age > 35
4. Fetch RowIDs: 2, 7 (age=28), 4 (age=31), 1 (age=35)

```
Leaf level (doubly linked):
[19] ↔ [22] ↔ [28,28] ↔ [31] ↔ [35] ↔ [45]
              ↑ start here       ↑ stop here
```

This is why B-trees are great for range queries — leaf nodes are **linked in sorted order**.

---

### What Happens on INSERT

```sql
INSERT INTO users (name, age, email) VALUES ('Henry', 30, 'h@...');
```

1. Row is written to the table (RowID = 8)
2. Index must be updated:
   - Find where `age=30` belongs in the B-tree (between 28 and 31)
   - Insert new key `30 → RowID=8` into the correct leaf node
   - If the leaf node is full (overflow) → **split the node** into two, push middle key up
   - Tree rebalances if needed

This is why **every index adds write overhead** — inserts, updates, and deletes must update the tree.

---

### What Happens Without an Index (Full Table Scan)

```sql
SELECT * FROM users WHERE age = 28;
-- No index on age
```

DB reads **every single page** (block of rows) from disk sequentially:
```
Page 1: read Charlie(35), Alice(28) ← match!
Page 2: read Bob(22), Diana(31)
Page 3: read Eve(19), Frank(45)
Page 4: read Grace(28) ← match!
→ Read ALL pages before stopping
```

With 1 million rows across 10,000 pages — the DB reads all 10,000 pages even to find 2 matches.

---

### Internal Node Structure

Each B-tree node is one **disk page** (~4KB or 8KB). It stores:

```
| key1 | ptr1 | key2 | ptr2 | key3 | ptr3 | ... |
  ↑              ↑              ↑
  pointer to     pointer to     pointer to
  subtree with   subtree with   subtree with
  keys < key1    key1≤k<key2    keys ≥ key2
```

Branching factor = how many keys fit in one page ≈ 100–500 for typical key sizes.

With branching factor 200 and 1 million rows:
- Level 1 (root): 1 node
- Level 2: 200 nodes
- Level 3: 40,000 nodes (leaf level, covers 200×200 = 40,000 entries each)
- **Depth = 3** → only 3 disk reads to find any row among 1 million ✅

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

## Clustered vs Non-Clustered Index

This is one of the most important distinctions — and a very common interview topic.

### Clustered Index

The **table rows are physically stored in the order of the index key**. The leaf nodes of the B-tree ARE the actual table data.

```
Clustered index on id:

Leaf node (= actual table page):
| id=1 | Alice  | 28 | alice@...
| id=2 | Bob    | 22 | bob@...
| id=3 | Diana  | 31 | diana@...
   ↑
   The row data IS here — no extra lookup needed
```

- Each table can have **only one** clustered index (data can only be sorted one way)
- In MySQL InnoDB: the **primary key is always the clustered index**
- In PostgreSQL: no true clustered index (heap storage), but you can `CLUSTER` a table

**Benefit:** Range scans on the clustered key are ultra-fast — rows are physically adjacent on disk.

```sql
SELECT * FROM orders WHERE id BETWEEN 1000 AND 2000;
-- Clustered on id → rows 1000-2000 are on consecutive disk pages → sequential I/O
```

---

### Non-Clustered Index (Secondary Index)

A **separate structure** from the table. Leaf nodes store the index key + a pointer back to the row.

```
Non-clustered index on email:

Leaf node:
| email='alice@...' | → pointer to actual row (RowID or primary key)
| email='bob@...'   | → pointer to actual row
| email='diana@...' | → pointer to actual row
         ↑
         Index is sorted by email
         Rows are NOT stored here — need a second lookup
```

**Two-step lookup (Double Read):**
1. Search index → get RowID or PK value
2. Use RowID/PK to fetch actual row from the table

```
Query: WHERE email = 'alice@gmail.com'
Step 1: B-tree on email → finds leaf → pointer says PK=2
Step 2: B-tree on PK (clustered) → fetch full row for PK=2
→ 2 B-tree lookups total
```

**This second lookup is called a "heap fetch" or "bookmark lookup"** — it's why non-clustered indexes are slightly slower than clustered for full row reads.

| | Clustered | Non-Clustered |
|---|---|---|
| Where data lives | In the index itself | Separate from index |
| Number per table | **1 only** | Many (up to 999 in SQL Server) |
| Row lookup | Direct — data is the leaf | Extra step — pointer to row |
| Range scans | Very fast (sequential I/O) | Slower (random I/O per row) |
| Insert order | Rows inserted in key order | No impact on row order |

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

**Why it's so fast:**
```
Without covering index:
  Step 1: B-tree on user_id → RowID=101, 205, 309
  Step 2: fetch row 101 from disk (random I/O)
  Step 3: fetch row 205 from disk (random I/O)  ← expensive
  Step 4: fetch row 309 from disk (random I/O)

With covering index (user_id, status, amount):
  Step 1: B-tree → leaf node has user_id=42, status='paid', amount=100
          All data is RIGHT HERE in the index leaf — no row fetch needed ✅
```

---

## Bitmap Index

Used for **low-cardinality columns** (columns with very few distinct values: gender, status, country).

Instead of pointing to individual rows, it stores a **bit array** — one bit per row.

```
Table:
RowID | status
  1   | active
  2   | inactive
  3   | active
  4   | pending
  5   | active

Bitmap index on status:
            Row1 Row2 Row3 Row4 Row5
active    =  1    0    1    0    1
inactive  =  0    1    0    0    0
pending   =  0    0    0    1    0
```

**Query: WHERE status = 'active'**
→ Read the 'active' bitmap: `10101`
→ Rows 1, 3, 5 match. Done.

**Multi-condition query: WHERE status = 'active' AND country = 'US'**
→ active bitmap:  `10101`
→ US bitmap:      `11010`
→ AND (bitwise):  `10000`  → only Row 1 matches both ✅

Bitwise AND on bitmaps is extremely fast (CPU processes 64 bits at once).

**Used by:** Oracle, PostgreSQL (for internal operations), data warehouses (Redshift, Vertica).

**Not for write-heavy tables** — every insert/update requires updating multiple bitmaps.

---

## Full-Text Index (Inverted Index)

For searching **inside text** (not just exact match). Used by search engines.

A B-tree index on `body` can't help with:
```sql
SELECT * FROM articles WHERE body LIKE '%distributed systems%';
-- Full scan — no index usable
```

A **full-text / inverted index** builds a map: **word → list of documents containing it**

```
Document 1: "distributed systems are complex"
Document 2: "systems design interview"
Document 3: "distributed databases scale well"

Inverted Index:
"distributed" → [Doc1, Doc3]
"systems"     → [Doc1, Doc2]
"design"      → [Doc2]
"databases"   → [Doc3]
```

**Query: search for "distributed systems"**
→ Lookup "distributed" → {Doc1, Doc3}
→ Lookup "systems"     → {Doc1, Doc2}
→ Intersect → Doc1 ✅

```sql
-- PostgreSQL full-text search
CREATE INDEX idx_articles_fts ON articles USING GIN(to_tsvector('english', body));

SELECT * FROM articles
WHERE to_tsvector('english', body) @@ to_tsquery('distributed & systems');
```

**Used by:** Elasticsearch (entire engine), PostgreSQL GIN index, MySQL FULLTEXT index.

---

## GiST Index (Spatial / Geometric)

For **geospatial data** — finding nearby points, shapes, overlapping rectangles.

```sql
-- PostGIS spatial index
CREATE INDEX idx_places_geo ON places USING GIST(location);

-- Find all restaurants within 5km of a point
SELECT * FROM places
WHERE ST_DWithin(location, ST_MakePoint(40.7128, -74.0060), 5000);
```

GiST uses **bounding boxes** arranged in a tree — not sorted by a single number, but by spatial region containment. Used by mapping apps (Uber, Google Maps), proximity services.

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

**Q: What is a database index and how does it work internally?**  
A: An index is a separate B-tree data structure where leaf nodes store the indexed column values sorted, each paired with a pointer (RowID or primary key) to the actual table row. To find a row, the DB starts at the root node and at each level compares the search value to keys to decide left/right, until reaching the correct leaf node — O(log N) node reads instead of O(N) row reads. With a branching factor of 200 and 1 million rows, only ~3 node reads are needed.

**Q: What is the difference between a clustered and non-clustered index?**  
A: A clustered index physically stores the table rows in the order of the index key — the leaf nodes ARE the row data. Each table can have only one. A non-clustered index is a separate structure where leaf nodes hold the key + a pointer back to the row. Reading a full row via a non-clustered index requires two lookups: one to find the pointer, one to fetch the row ("heap fetch"). In MySQL InnoDB, the primary key is always the clustered index.

**Q: What is the left-prefix rule for composite indexes?**  
A: A composite index on (A, B, C) can be used for queries filtering on A, A+B, or A+B+C — but not B alone or C alone. The index is sorted first by A, then by B within A, then by C within B. Skipping A means the sort order is irrelevant and the index can't be used for efficient lookup.

**Q: What is the difference between B-tree and LSM-tree indexes?**  
A: B-tree: balanced sorted tree, fast reads (O(log N)), moderate write speed (random disk writes to maintain balance). Used by PostgreSQL/MySQL. LSM-tree: write-optimized, all writes are sequential appends to memory then disk. Reads check multiple levels (slower). Used by Cassandra, RocksDB. Choose B-tree for read-heavy OLTP; choose LSM for write-heavy (time-series, event logs).

**Q: When should you NOT add an index?**  
A: On write-heavy tables where index maintenance overhead outweighs read benefit (e.g., a high-throughput events/logs table). On low-cardinality columns (e.g., `gender` with 2 values — the index isn't selective enough to help). On small tables where a full scan is faster than index lookup. When the column is almost never used in WHERE clauses.

**Q: What is a covering index?**  
A: An index that contains all columns needed to answer a query, so the DB never reads the actual table rows (index-only scan). Example: index on (user_id, status, amount) for a query that selects status and amount filtered by user_id. All data is in the index. Much faster than fetching from the table — avoids random I/O.

**Q: Why does a query with a B-tree index only need 3 disk reads for 1 million rows?**  
A: Because B-tree nodes are large (one disk page ~8KB) and can hold hundreds of keys (branching factor ~200). With branching factor 200: Level 1 covers 200 entries, Level 2 covers 40,000 entries, Level 3 (leaf) covers 8,000,000 entries. So 1 million rows fits in just 3 levels — meaning 3 disk reads to find any row. This is why indexes are so effective at scale.

**Q: What is a bitmap index? When would you use it?**  
A: A bitmap index stores one bit per row per distinct value — e.g., for a `status` column with values active/inactive/pending, it stores three bit arrays. Queries like `WHERE status='active' AND country='US'` become a bitwise AND of two bitmaps — extremely fast. Best for low-cardinality columns in read-heavy/analytical workloads (data warehouses). Not suitable for write-heavy tables because every write must update multiple bitmaps.

**Q: What is an inverted index? How does Elasticsearch use it?**  
A: An inverted index maps each unique word → list of documents containing that word. For a query "distributed systems", it looks up both words and intersects their document lists. Elasticsearch is built entirely on inverted indexes (via Apache Lucene). This is what makes full-text search fast — instead of scanning every document for the word, you do two hash/tree lookups and intersect.

**Q: Why can't a B-tree index be used for `WHERE name LIKE '%alice%'`?**  
A: A B-tree index works because values are sorted. `LIKE 'alice%'` (prefix) works because all matching values are in a contiguous sorted range. `LIKE '%alice%'` (substring) has no defined position in sort order — 'zzalice' and 'aalice' both match but are at opposite ends of the tree. The DB must scan all index entries to find matches, making the index useless — it falls back to a full scan. Use a full-text (inverted) index for substring searches.

**Q: What happens to indexes during a high-volume INSERT workload?**  
A: Each INSERT must update every index on the table. If a table has 5 indexes, an insert causes 6 writes (1 for the row + 5 index updates). Inserting in non-clustered-index order causes random I/O — the index B-tree node to update can be anywhere on disk. This is why write-heavy tables should have few indexes, and why LSM-tree databases (Cassandra) buffer writes in memory and flush sequentially — avoiding random I/O entirely.

**Q: A query is slow even though the column has an index. What could be wrong?**  
A: Several reasons: (1) Function on the column — `WHERE YEAR(created_at) = 2024` wraps the column in a function, breaking index usage; rewrite as range query instead. (2) Implicit type cast — `WHERE user_id = '123'` where user_id is INT causes a cast, index not used. (3) Low cardinality — if 90% of rows have `status='active'`, the DB may choose a full scan over the index (not selective enough). (4) Wrong prefix — composite index on (A,B) but query only filters on B. (5) Stale statistics — run `ANALYZE` to update query planner stats.
