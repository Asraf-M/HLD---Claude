# System Design: Instagram / Facebook Comments (Beginner-Friendly Guide)

---

## What Are We Building?

The **comments system** lets users post text under any photo, video, or status update, reply to each other's comments, and like comments. You see it everywhere:

- Instagram: 3 lines of comments visible on feed, tap to expand all
- Facebook: threaded comments with replies
- YouTube: comment section under every video
- Twitter/X: replies thread

> **Why is this a common interview question?**  
> It looks simple on the surface — "just store comments in a database." But at Instagram/Facebook scale (1 billion DAU) you're handling **~100,000+ writes per second** and **millions of reads per second**. You need to think about sharding, caching, atomic counters, nested threading, and hot-post problems.

---

## Step 1: Understand What We're Building

### Functional Requirements

| Feature | Detail |
|---------|--------|
| Post a comment | On any post/photo/video |
| View comments | Paginated list, newest or top first |
| Reply to comment | One level of nesting (Instagram) or unlimited (Facebook/YouTube) |
| Like a comment | Increment like count |
| Delete / edit comment | Soft delete preferred |
| Comment count | Show "482 comments" on the post |
| Notifications | Notify post owner when commented; notify commenter when replied |

### Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| Scale | 1 billion DAU |
| Comment writes | ~100,000 writes/sec |
| Comment reads | ~1 million reads/sec (reads >> writes, 10:1) |
| Availability | High (99.99%) — users must always see comments |
| Consistency | Eventual is OK — slight delay in seeing new comments is fine |
| Latency | < 200ms for comment list load |

### Back-of-Envelope

```
DAU: 1 billion
Each user posts ~0.1 comments/day (not everyone comments)
→ 100 million comments/day
→ 100M / 86,400 ≈ 1,160 writes/sec (peak ~5×) ≈ 6,000 writes/sec

Reads: 10:1 ratio → 60,000 reads/sec
Each comment row: ~500 bytes (text + metadata)
Storage per day: 100M × 500 bytes = 50 GB/day
Storage per year: ~18 TB/year
```

---

## Step 2: The APIs

### 1. Post a Comment
```
POST /v1/posts/{post_id}/comments
Authorization: Bearer <token>
Body: {
  "text": "Love this photo! 😍",
  "parent_comment_id": null    // null = top-level comment
}

Response 201:
{
  "comment_id": "cmnt_7f3k9a",
  "post_id": "post_abc123",
  "user": { "id": 42, "username": "alice" },
  "text": "Love this photo! 😍",
  "created_at": "2024-05-19T10:30:00Z",
  "like_count": 0,
  "reply_count": 0
}
```

### 2. Reply to a Comment
```
POST /v1/posts/{post_id}/comments
Body: {
  "text": "@alice me too!",
  "parent_comment_id": "cmnt_7f3k9a"   // non-null = reply
}
```
> Replies use the same endpoint — a reply is just a comment with a `parent_comment_id`.

### 3. List Comments on a Post
```
GET /v1/posts/{post_id}/comments?limit=20&cursor=<cursor>&sort=top
```

```
Response 200:
{
  "comments": [
    {
      "comment_id": "cmnt_7f3k9a",
      "text": "Love this photo! 😍",
      "user": { "id": 42, "username": "alice" },
      "like_count": 847,
      "reply_count": 12,
      "created_at": "...",
      "top_replies": [...]   // 2 preview replies, inline
    },
    ...
  ],
  "next_cursor": "eyJpZCI6...",
  "total_count": 482
}
```

### 4. List Replies Under a Comment
```
GET /v1/comments/{comment_id}/replies?limit=10&cursor=<cursor>
```

### 5. Like / Unlike a Comment
```
POST   /v1/comments/{comment_id}/like    → like (idempotent)
DELETE /v1/comments/{comment_id}/like    → unlike
```

### 6. Delete a Comment
```
DELETE /v1/comments/{comment_id}
→ Soft delete (is_deleted = true), not physical removal
→ Comments with replies show "This comment was deleted"
```

---

## Step 3: High-Level Architecture

```
┌──────────────────────────────────────────────────────┐
│                     Clients                          │
│          (iOS / Android / Web Browser)               │
└───────────────────────┬──────────────────────────────┘
                        │ HTTPS
                        ▼
              ┌─────────────────┐
              │   API Gateway   │  ← Auth, Rate Limiting, Routing
              └────────┬────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
   ┌─────────┐  ┌──────────┐  ┌──────────┐
   │ Comment │  │  Like    │  │  Notif   │
   │ Service │  │ Service  │  │ Service  │
   └────┬────┘  └────┬─────┘  └────┬─────┘
        │             │              │
        ▼             ▼              ▼
   ┌─────────┐  ┌──────────┐  ┌──────────┐
   │ Comment │  │  Redis   │  │  Kafka   │
   │   DB    │  │ (Counts) │  │  Queue   │
   │(MySQL/  │  └──────────┘  └──────────┘
   │Postgres)│
   └────┬────┘
        │
   ┌────▼────┐
   │  Redis  │   ← Cache (comment list per post)
   │  Cache  │
   └─────────┘
```

---

## Step 4: Database Design

### Core Tables

#### `comments` Table
```sql
CREATE TABLE comments (
    comment_id    BIGINT       PRIMARY KEY,   -- Snowflake ID
    post_id       BIGINT       NOT NULL,
    user_id       BIGINT       NOT NULL,
    parent_id     BIGINT       DEFAULT NULL,  -- NULL = top-level comment
                                              -- non-NULL = reply
    content       VARCHAR(2200) NOT NULL,     -- Instagram's limit
    like_count    INT          DEFAULT 0,     -- denormalized
    reply_count   INT          DEFAULT 0,     -- denormalized
    is_deleted    BOOLEAN      DEFAULT FALSE,
    created_at    TIMESTAMP    NOT NULL,
    updated_at    TIMESTAMP    NOT NULL,

    INDEX idx_post_created  (post_id, created_at DESC),  -- list comments by post
    INDEX idx_parent_created (parent_id, created_at ASC) -- list replies
);
```

#### `comment_likes` Table
```sql
CREATE TABLE comment_likes (
    comment_id  BIGINT NOT NULL,
    user_id     BIGINT NOT NULL,
    created_at  TIMESTAMP NOT NULL,
    PRIMARY KEY (comment_id, user_id)  -- prevents duplicate likes
);
```

> **Why a separate `comment_likes` table instead of just a counter?**  
> The counter (like_count) tells you *how many* likes. The separate table tells you *who* liked it — needed for "did I already like this?" check and for anti-spam. The counter is a denormalized cache of the table's row count.

### Post Comment Count (Denormalized)
```sql
-- On the posts table:
ALTER TABLE posts ADD COLUMN comment_count INT DEFAULT 0;

-- Increment atomically when a comment is added:
UPDATE posts SET comment_count = comment_count + 1 WHERE post_id = ?
```

> This avoids `SELECT COUNT(*) FROM comments WHERE post_id = ?` on every post render — which would be catastrophically slow at scale.

---

## Step 5: Nested Comments (Threading)

Instagram and Facebook have different approaches:

### Instagram: 2-Level Nesting Only
```
Comment (level 1)
  └── Reply (level 2)
  └── Reply (level 2)
  └── Reply (level 2)
Comment (level 1)
  └── Reply (level 2)
```

This is simple — just check `parent_id IS NULL` for top-level, `parent_id IS NOT NULL` for replies. Maximum depth = 1.

### Facebook/YouTube: Unlimited Nesting

For unlimited nesting, use the **Adjacency List** model — each comment stores its `parent_id`.

```
comment_id | parent_id | text
-----------+-----------+------------------
100        | NULL      | "Great photo!"
101        | 100       | "I agree!"
102        | 101       | "Same here"       ← reply to a reply
103        | 100       | "Beautiful!"
```

**Fetching the full tree:**
```sql
-- Top-level comments for a post:
SELECT * FROM comments
WHERE post_id = 42 AND parent_id IS NULL
ORDER BY created_at DESC
LIMIT 20;

-- Replies for a specific comment:
SELECT * FROM comments
WHERE parent_id = 100
ORDER BY created_at ASC
LIMIT 10;
```

> **Why not fetch the full tree in one query?**  
> A recursive CTE could fetch it, but for deeply nested trees it's complex and slow at scale. Instead, lazy-load replies on demand (click "View replies" → separate API call). This is what Instagram, YouTube, and Facebook all do.

### Alternative: Materialized Path (Used by Reddit)
```
comment_id | path              | text
-----------+-------------------+------------------
100        | /100/             | "Great photo!"
101        | /100/101/         | "I agree!"
102        | /100/101/102/     | "Same here"
103        | /100/103/         | "Beautiful!"
```

Fetch all descendants of comment 100:
```sql
SELECT * FROM comments WHERE path LIKE '/100/%';
```

> **Materialized path** is great for deep trees (Reddit-style). **Adjacency list** is simpler for 2-3 levels (Instagram-style). Use adjacency list unless you need truly unlimited depth.

---

## Step 6: Comment ID — Snowflake IDs

> **Why not auto-increment?**  
> When you shard across multiple database nodes, each node would assign the same IDs (1, 2, 3...) causing collisions.

Use a **Snowflake ID** (like Twitter's):
```
64-bit integer:
| 41 bits timestamp (ms) | 10 bits machine ID | 12 bits sequence |

Example: 7302145832847360001
→ Timestamp: 2024-05-19 10:30:00.123
→ Machine:   server-5
→ Sequence:  1 (first ID in that millisecond on that machine)
```

**Benefits:**
- Globally unique across all shards
- Time-sortable (higher ID = newer comment)
- No coordination needed between machines
- Fits in a BIGINT

---

## Step 7: Sharding Strategy

At 100M+ comments/day, a single database won't handle writes. We shard.

### Option A: Shard by `post_id`
All comments for a post live on the same shard.

```
post_id % 100 → shard number

post_id 1234 → shard 34  (all comments for this post)
post_id 5678 → shard 78  (all comments for this post)
```

**Pros:** All comments for a post in one query — no scatter-gather.  
**Cons:** Viral posts (millions of comments) create a **hotspot** — one shard gets overloaded.

### Option B: Shard by `comment_id`
Comments distributed evenly across shards.

**Pros:** Even distribution, no hotspots.  
**Cons:** Reading all comments for a post requires querying all shards and merging results (scatter-gather) — complex and slow.

### Best Approach: Shard by `post_id` + Cache Hot Posts

Shard by `post_id` for the happy path (most posts are not viral). For viral posts that become hotspots: **cache the comments in Redis** so the DB shard barely gets hit.

```
Request → Redis cache hit? → Return cached comments  (DB not touched)
                ↓ cache miss
        → Query DB shard for post_id  → populate cache
```

> Most posts get <1000 comments. The DB shard handles this easily. Only viral posts (millions of comments) need caching. This is the same principle Instagram uses.

---

## Step 8: Caching Strategy

### What to Cache

| Cache Key | Value | TTL |
|-----------|-------|-----|
| `comments:post:{post_id}:page1` | First 20 comments (JSON) | 5 min |
| `comment_count:post:{post_id}` | Integer count | 1 min |
| `likes:comment:{comment_id}` | Like count | 5 min |
| `user_liked:user:{user_id}:cmnt:{comment_id}` | Boolean | 24h |

### Cache-Aside Pattern
```python
def get_comments(post_id, cursor=None):
    if cursor is None:   # only cache first page
        cache_key = f"comments:post:{post_id}:page1"
        cached = redis.get(cache_key)
        if cached:
            return json.loads(cached)
    
    # Cache miss or non-first page → query DB
    comments = db.query(
        "SELECT * FROM comments WHERE post_id = ? AND parent_id IS NULL "
        "ORDER BY created_at DESC LIMIT 20",
        post_id
    )
    
    if cursor is None:
        redis.setex(cache_key, 300, json.dumps(comments))  # 5 min TTL
    
    return comments
```

### Invalidation on New Comment
```python
def post_comment(post_id, user_id, text):
    comment = db.insert_comment(post_id, user_id, text)
    
    # Invalidate cache so next read fetches fresh
    redis.delete(f"comments:post:{post_id}:page1")
    
    # Increment comment count atomically in Redis
    redis.incr(f"comment_count:post:{post_id}")
    
    # Publish event for notifications
    kafka.publish("comment.created", {
        "comment_id": comment.id,
        "post_id": post_id,
        "author_id": user_id
    })
```

> **Why invalidate instead of update?**  
> Updating the cached list requires fetching the new comment, inserting it at the top, trimming to 20 items — more complex and error-prone. Invalidating is simpler: next request just refetches from DB. At Instagram scale, this is fine — the DB is read once, then serves the next 1000 requests from cache.

---

## Step 9: Like Count — Atomic Counters

> **Problem:** Like button gets clicked thousands of times per second on a viral comment. Updating `like_count` in SQL for every click causes lock contention.

**Solution: Redis atomic counter + periodic DB sync**

```
User likes comment → Redis INCR like:comment:123
                   → Background job every 30s flushes to DB:
                     UPDATE comments SET like_count = ? WHERE comment_id = 123
```

```python
def like_comment(user_id, comment_id):
    # 1. Prevent duplicate likes (check/set in Redis or DB)
    already_liked = redis.sismember(f"liked_by:comment:{comment_id}", user_id)
    if already_liked:
        return  # idempotent — no-op

    # 2. Record the like (for "did I like this?" queries)
    db.execute(
        "INSERT IGNORE INTO comment_likes (comment_id, user_id) VALUES (?, ?)",
        comment_id, user_id
    )
    
    # 3. Increment counter in Redis (fast, atomic)
    redis.incr(f"like_count:comment:{comment_id}")
    redis.sadd(f"liked_by:comment:{comment_id}", user_id)
    
    # 4. Background job periodically: flush Redis counter → DB like_count
```

> **Why not update the DB like_count directly on every like?**  
> A comment can receive 10,000 likes/second in a viral spike. At that rate, `UPDATE comments SET like_count = like_count + 1` causes heavy lock contention and could saturate the DB. Redis `INCR` is O(1) and lock-free — can handle millions of operations per second.

---

## Step 10: Notifications

When someone comments on your post, or replies to your comment, you get notified.

```
Comment Service → Kafka → Notification Service → Push/Email/In-App

Kafka event payload:
{
  "event": "comment.created",
  "post_id": "post_abc",
  "post_owner_id": 99,          // who owns the post → notify them
  "comment_author_id": 42,
  "comment_id": "cmnt_7f3k",
  "parent_comment_id": null     // null = top-level, post owner notified
                                // non-null = reply, parent commenter notified
}
```

**Notification Service logic:**
```python
def handle_comment_event(event):
    if event.parent_comment_id is None:
        # New comment on a post → notify post owner
        notify(user_id=event.post_owner_id,
               msg=f"{commenter_name} commented on your post")
    else:
        # Reply to a comment → notify original commenter
        parent = db.get_comment(event.parent_comment_id)
        notify(user_id=parent.user_id,
               msg=f"{commenter_name} replied to your comment")
    
    # Don't notify if commenter IS the post/comment owner
    # Don't notify if user has muted the post
```

> **Why Kafka and not direct call?**  
> Notifications are non-critical for comment submission — the user doesn't wait for the notification to be sent before they see their comment appear. Decoupling via Kafka means Comment Service always responds fast. If Notification Service is down, events queue up in Kafka and are processed when it recovers — no notifications lost.

---

## Step 11: Handling Hot / Viral Posts

A post with millions of comments is a **hot key problem** — all traffic hits one shard.

### Problem
```
Kylie Jenner posts a photo → 3 million comments in 24 hours
→ All on one DB shard (shard = post_id % 100)
→ That shard: 100,000 reads/sec just for this post
→ Shard overloaded → slow reads for ALL posts on that shard
```

### Solutions

**1. Read from replica (most common)**  
All comment reads go to read replicas. The primary only handles writes. One primary, N replicas.

**2. Cache aggressive TTL**  
Cache the first page of comments for viral posts with a longer TTL (10 min instead of 1 min). Fewer DB reads.

**3. Hot post detection + dedicated cache**  
```python
# Detect hot posts
def get_comments(post_id, ...):
    if is_hot_post(post_id):  # e.g., >100K comments or >1M views/hr
        return get_from_hot_cache(post_id)  # separate Redis cluster
    return normal_get_comments(post_id)

def is_hot_post(post_id):
    # Simple: check request rate for this post_id
    return redis.incr(f"req_rate:{post_id}") > THRESHOLD
```

**4. Shard splitting**  
For permanently hot posts, move them to their own dedicated shard.

---

## Step 12: Full Request Flow (End to End)

### Posting a Comment
```
1. User types "Love this!" and taps Post
2. Client → POST /v1/posts/abc/comments (with auth token)
3. API Gateway validates token, rate-limits (max 10 comments/min/user)
4. Comment Service:
   a. Generates Snowflake ID for new comment
   b. Writes to DB: INSERT INTO comments (...)
   c. Increments comment_count in Redis: INCR comment_count:post:abc
   d. Invalidates comment list cache: DEL comments:post:abc:page1
   e. Publishes Kafka event: "comment.created"
5. Response 201 with comment object → displayed immediately on client
6. Kafka consumer (Notification Service) → notifies post owner
```

### Loading Comments
```
1. User opens a post
2. Client → GET /v1/posts/abc/comments?limit=20
3. Comment Service:
   a. Check Redis: GET comments:post:abc:page1
   b. Cache hit → return immediately (< 5ms)
   c. Cache miss → query DB:
      SELECT * FROM comments WHERE post_id=abc AND parent_id IS NULL
      ORDER BY created_at DESC LIMIT 20
   d. Populate cache with 5min TTL
   e. Return comments with like counts, reply counts, 2 preview replies each
4. Client renders comments list
```

---

## Summary Table

| Component | Choice | Why |
|-----------|--------|-----|
| **Comment ID** | Snowflake (64-bit) | Globally unique across shards, time-sortable |
| **Primary DB** | MySQL / PostgreSQL | Strong consistency for comment writes |
| **Sharding key** | `post_id` | All comments for a post on one shard |
| **Nested comments** | Adjacency list (parent_id) | Simple, works for 1-2 levels |
| **Cache** | Redis (comment list, counts) | Absorbs massive read traffic |
| **Cache strategy** | Cache-aside, invalidate on write | Simple, consistent |
| **Like counts** | Redis INCR + async DB flush | Lock-free, handles viral spikes |
| **Comment count** | Denormalized in posts table | Avoid COUNT(*) on every render |
| **Notifications** | Kafka + Notification Service | Decoupled, async, no lost events |
| **Hot posts** | Read replicas + aggressive cache TTL | Prevents shard overload |
| **Pagination** | Cursor-based | Stable under concurrent inserts |
| **Delete** | Soft delete (is_deleted flag) | Preserve replies, audit trail |

---

## Q&A Cheat Sheet

**Q: How would you design a comments system for Instagram at scale?**  
A: Store comments in a sharded MySQL/PostgreSQL DB with shard key = post_id (so all comments for a post are colocated). Use Snowflake IDs for globally unique comment IDs. Cache the first page of comments per post in Redis with a short TTL. Denormalize comment counts in the posts table to avoid expensive COUNT queries. Use Redis INCR for like counts and flush periodically to DB. Publish events to Kafka for async notification delivery.

**Q: How do you handle nested/threaded comments?**  
A: Use an adjacency list — each comment has a `parent_comment_id` column. NULL means top-level, non-NULL means reply. For Instagram (max 2 levels), fetch top-level comments in one query, then fetch replies on demand when the user taps "View replies." For unlimited depth (Facebook/YouTube), use the same adjacency list but lazy-load each level. Reddit uses a materialized path for deep trees.

**Q: How do you prevent a viral post from overloading one database shard?**  
A: Shard by post_id so all comments for a post are on one shard (efficient for reads). Protect that shard with: (1) read replicas — all reads go to replicas, writes to primary; (2) Redis cache with aggressive TTL for hot posts' comment lists; (3) hot post detection — if a post exceeds X requests/sec, serve from a dedicated cache cluster. This way the DB shard only gets hit on cache misses.

**Q: How do you keep comment counts accurate without running COUNT(*) every time?**  
A: Denormalize the count. Maintain a `comment_count` column in the posts table. On every INSERT to comments, atomically increment: `UPDATE posts SET comment_count = comment_count + 1 WHERE id = ?`. For in-flight accuracy, also cache the count in Redis with `INCR`. The DB counter is the source of truth; Redis is the fast-path. Accept that counts may lag by a few seconds — eventual consistency is fine for comment counts.

**Q: Why use Kafka for notifications instead of calling the notification service directly?**  
A: Synchronous calls couple Comment Service to Notification Service. If Notification Service is slow or down, comment posting becomes slow or fails. With Kafka: Comment Service writes to DB, publishes event, returns 201 immediately — fast and reliable. Notification Service consumes from Kafka asynchronously. If it goes down, events queue up and are processed on recovery. No notifications are lost, and comment posting is never blocked by notification delivery.

**Q: How do you make the "like" button idempotent?**  
A: Use a composite primary key on `comment_likes (comment_id, user_id)`. An INSERT with `ON CONFLICT DO NOTHING` (Postgres) or `INSERT IGNORE` (MySQL) ensures double-tapping doesn't create duplicate like records. Also check in Redis (`SISMEMBER liked_by:comment:X`) before hitting the DB. This is the standard idempotency key pattern — the (comment_id, user_id) pair is the natural idempotency key.

**Q: Cursor vs offset pagination for comments — which do you use?**  
A: Cursor-based. Comments are actively written to (new comments appear at top or bottom). With offset pagination, a new comment at position 0 shifts everything — page 2 now contains a duplicate of the last item on page 1. Cursor-based (`WHERE created_at < :last_seen_time`) is stable under concurrent inserts. Encode the cursor as an opaque base64 token so clients can't manipulate it.
