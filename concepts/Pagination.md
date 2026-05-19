# Pagination

## What Is It?

Pagination is the technique of splitting large datasets into smaller chunks ("pages") so that:
- The API doesn't return millions of rows in one response
- The client can fetch data incrementally
- Database queries stay fast

> **Analogy:** A book has chapters and page numbers. You don't read (or carry) the whole book at once — you open to a specific page. Pagination in APIs works the same way.

---

## Why Pagination Matters

```sql
-- Without pagination:
SELECT * FROM posts ORDER BY created_at DESC;
-- Returns 50 million rows → 4GB response → client dies, DB overloaded

-- With pagination:
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 0;
-- Returns 20 rows → fast, manageable
```

---

## Approach 1: Offset-Based Pagination

The simplest approach — use `LIMIT` and `OFFSET`.

### API Design
```
GET /api/posts?page=1&size=20      → rows 1-20
GET /api/posts?page=2&size=20      → rows 21-40
GET /api/posts?page=3&size=20      → rows 41-60
```

Or with raw offset:
```
GET /api/posts?limit=20&offset=0   → rows 1-20
GET /api/posts?limit=20&offset=20  → rows 21-40
```

### SQL
```sql
SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 20 OFFSET 200;   -- page 11
```

### Problems

**1. Performance degrades at high offsets:**
```sql
SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 20 OFFSET 1000000;
-- DB must scan and discard 1,000,000 rows before returning 20
-- Gets slower as page number increases
```

**2. Data inconsistency during inserts:**
```
User fetches page 1 (rows 1-20).
A new post is inserted at the top.
User fetches page 2 (offset 20):
  Row 20 from page 1 now appears at offset 21 → shown again (duplicate).
  Or row 21 from original set is now at offset 22 → skipped (missing).
```

**3. Cannot be used with deep pagination** (offset > 100,000 is typically a problem).

### When to Use
- Small datasets (< 10,000 rows)
- Admin dashboards with numbered pages
- When users don't scroll to page 500+ (most users stay on early pages)
- Search results (Google — who goes to page 50?)

---

## Approach 2: Cursor-Based Pagination

Use an **opaque cursor** (pointer to a specific row) instead of a page number.

> **Analogy:** A bookmark. Instead of saying "page 11", you say "show me the next 20 posts after this specific post (ID 12345)."

### API Design
```
First request:
GET /api/posts?limit=20

Response:
{
  "data": [...20 posts...],
  "next_cursor": "eyJpZCI6MTIzNDV9"  // base64 of {"id": 12345}
}

Next request:
GET /api/posts?limit=20&cursor=eyJpZCI6MTIzNDV9

Response:
{
  "data": [...next 20 posts...],
  "next_cursor": "eyJpZCI6MTIzNjV9"
}
```

### SQL (using ID as cursor)
```sql
-- First page:
SELECT * FROM posts ORDER BY id DESC LIMIT 20;
-- Last id returned: 12345

-- Next page (cursor = 12345):
SELECT * FROM posts WHERE id < 12345 ORDER BY id DESC LIMIT 20;
-- Directly finds the position using the indexed id → O(log N) lookup
```

### Why It's Faster

Offset requires: scan from start → skip N rows → return next 20.  
Cursor requires: index seek to cursor position → scan next 20.

```sql
-- Offset at page 50,000:
OFFSET 1000000  → scan and discard 1,000,000 rows → slow

-- Cursor at same position:
WHERE id < 12345 → index seek → immediate → fast
```

Both are O(1) with index on the cursor column (assuming B-tree index on `id` or `created_at`).

### Cursor Stability

New inserts don't affect your cursor position:
```
User fetches with cursor=12345 (posts with id < 12345).
New post id=99999 is inserted.
User fetches next page with cursor=12345:
  Still returns id < 12345 → no duplicates, no skips ✅
```

### Encoding the Cursor

The cursor should be **opaque** — clients shouldn't know or manipulate it.

```python
import base64, json

def encode_cursor(last_id, last_created_at):
    data = json.dumps({"id": last_id, "created_at": str(last_created_at)})
    return base64.b64encode(data.encode()).decode()

def decode_cursor(cursor):
    data = base64.b64decode(cursor.encode()).decode()
    return json.loads(data)
```

### Limitations of Cursor-Based
- **No random access**: you can't jump to page 500 (no "go to page N" UI)
- **No total count** easily available (can estimate, not paginate by total)
- **Cursor invalidation**: if the cursor row is deleted, cursor may be stale (handle with fallback)
- **Bi-directional scroll** is more complex (need both `before` and `after` cursors)

---

## Approach 3: Keyset Pagination

Cursor-based using **actual column values** (not encoded/opaque) as the cursor.

```
GET /api/posts?after_id=12345&limit=20
```

```sql
SELECT * FROM posts
WHERE id < 12345   -- "keyset" condition
ORDER BY id DESC
LIMIT 20;
```

This is essentially cursor-based pagination with the cursor being the key value itself. Used by many APIs that want simpler, transparent pagination.

---

## Approach 4: Time-Based Cursor (for feeds/timelines)

Useful for time-series data (social feeds, chat messages, logs).

```
GET /api/messages?before=2024-01-15T10:30:00Z&limit=50
```

```sql
SELECT * FROM messages
WHERE created_at < '2024-01-15T10:30:00Z'
ORDER BY created_at DESC
LIMIT 50;
```

**Problem:** Multiple events can have the same timestamp. Use `(created_at, id)` as a composite cursor:

```sql
WHERE (created_at, id) < ('2024-01-15T10:30:00Z', 12345)
ORDER BY created_at DESC, id DESC
LIMIT 50;
```

---

## API Response Design

### Standard REST Pagination Response
```json
{
  "data": [
    { "id": 12345, "title": "Post title" },
    ...
  ],
  "pagination": {
    "next_cursor": "eyJpZCI6MTIzNDV9",
    "prev_cursor": "eyJpZCI6MTI0MDB9",
    "has_more": true,
    "total_count": 50000   // optional, expensive to compute
  }
}
```

### GraphQL (Relay-Style Connection)
```graphql
{
  posts(first: 20, after: "cursor123") {
    edges {
      node { id title }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

---

## Page Size Considerations

| Size | Trade-off |
|------|-----------|
| Too small (5) | Too many API calls |
| Too large (1000) | Slow response, heavy memory usage |
| **20-100** | **Sweet spot for most use cases** |

Allow clients to specify page size but enforce a maximum:
```python
size = min(request.params.get("size", 20), 100)  # default 20, max 100
```

---

## Infinite Scroll vs Numbered Pages

| UI Pattern | Best Pagination | Why |
|-----------|----------------|-----|
| **Infinite scroll** (Twitter, Instagram) | Cursor-based | No random access needed, stable under inserts |
| **Numbered pages** (Google, Amazon) | Offset-based | Users expect "page 5 of 200" |
| **Load more button** | Cursor-based | Same as infinite scroll |
| **Search results** | Offset-based | Random access to page N needed |

---

## Interview Q&A

**Q: What are the two main pagination approaches? What are the trade-offs?**  
A: Offset-based: simple, supports random page access (page 42), works well for small datasets. Problem: O(N) at high offsets (DB scans all skipped rows), and concurrent inserts cause duplicates/skips. Cursor-based: uses a pointer (ID or timestamp) to the last row. O(log N) with an index regardless of position, stable under concurrent inserts. Problem: no random access (can't jump to page 500), no easy total count.

**Q: Why is OFFSET 1000000 slow?**  
A: The database must read and discard 1,000,000 rows before returning the next page. Even though the final result is 20 rows, the DB has done 1,000,001 rows of work. With cursor-based pagination (`WHERE id < 12345`), the B-tree index lets the DB jump directly to the cursor position in O(log N) time.

**Q: How do you handle concurrent inserts with offset pagination?**  
A: You can't fully fix it with offset pagination. The standard solutions are: (1) switch to cursor-based pagination which is stable under inserts; (2) take a "snapshot" (e.g., read from a replica at a point in time); (3) accept slight inconsistency and note it in docs. For feeds (Twitter, Instagram) cursor-based is standard for this reason.

**Q: How would you paginate a social media feed efficiently?**  
A: Use cursor-based pagination with a composite cursor on (created_at, id). First page: no cursor, return first 20 posts sorted by created_at DESC, id DESC. Subsequent pages: use cursor = (last_created_at, last_id), query `WHERE (created_at, id) < (cursor_time, cursor_id)`. Encode cursor as opaque base64 so clients can't manipulate it. Index on (created_at, id) makes each page lookup O(log N).

**Q: What is the difference between cursor-based and keyset pagination?**  
A: They're essentially the same concept. Keyset pagination uses the actual column values directly as query parameters (`WHERE id < 12345`). Cursor-based pagination encodes those values into an opaque token (base64/JWT) that clients pass back. Cursor-based is more flexible (can encode multiple columns, versioning) and prevents clients from crafting arbitrary queries by knowing the internal structure.
