# System Design: News Feed System (Beginner-Friendly Guide)

---

## What is a News Feed System?

When you open Instagram, Facebook, or Twitter — the list of posts you see is your **news feed**. It's a real-time, personalized stream of content from people you follow.

**Real-world examples:**
- Facebook's home timeline
- Instagram's photo/video feed
- Twitter/X's tweet timeline
- LinkedIn's post feed

> **This is one of the most common system design interview questions** because it touches on nearly every major concept: databases, caching, message queues, fanout, pagination, and scalability.

---

## Step 1: Understand What We're Building

### Requirements

| Requirement | Detail |
|-------------|--------|
| Platforms | Web and Mobile |
| Max friends per user | 5,000 |
| Daily Active Users (DAU) | 10 million |
| Content types | Text, images, videos |
| Feed sorting | Reverse chronological (newest first) |

**Two core flows:**
1. **Publishing:** User creates a post → it shows up in friends' feeds
2. **Reading:** User opens app → they see an up-to-date feed

> **Question:** Why is this hard? Can't we just query "all posts from my friends" when someone opens the app?  
> **Answer:** Yes — but that would be extremely slow at scale. If you have 5,000 friends, you'd be querying 5,000 users' posts, sorting them, and paginating — every single time someone opens the app. At 10M daily users, this would crush your database. We need a smarter approach.

---

## Step 2: The Two APIs

### API 1: Publish a Post

```
POST /v1/me/feed
Body: {
  content: "Hello world!",
  auth_token: "abc123"
}
```

### API 2: Read the Feed

```
GET /v1/me/feed
Params: auth_token=abc123
```

> **Why do we need `auth_token`?**  
> The server needs to know *who* is asking. Without authentication, anyone could read anyone's private feed. The token proves your identity without sending your password on every request.

---

## Step 3: High-Level Design — Feed Publishing

When a user posts something, here's what happens:

<img src="../../system-design-notes/11. News Feed System/images/feed-publishing.png" alt="Feed Publishing" width="400">

```
User clicks "Post"
      |
      v
[Load Balancer]         ← Distributes traffic across multiple servers
      |
      v
[Web Servers]           ← Authenticate the request, rate limit
      |
   ___|___________________________________________
  |               |                              |
[Post Service]  [Fanout Service]      [Notification Service]
      |               |                              |
[Post Cache]  [News Feed Cache]         "Notify friends"
      |
[Post DB]
```

### What does each component do?

**Load Balancer:**
> Routes each incoming request to one of many web servers. Like a traffic cop ensuring no single server gets overwhelmed.

**Web Servers:**
> First line of processing. They check: Is your `auth_token` valid? Are you posting too fast (rate limiting)? If yes to both, forward to the appropriate service.

**Post Service:**
> Saves the actual post content (text, media URL, timestamp, user_id) to the **Post DB** and **Post Cache**.

**Fanout Service:**
> The most interesting one. It takes the new post and "fans it out" — distributing it to the news feeds of all your friends.

**Notification Service:**
> Optionally sends push notifications to friends: "Alice posted something new."

---

## Step 4: High-Level Design — Reading the Feed

<img src="../../system-design-notes/11. News Feed System/images/news-feed-building.png" alt="News Feed Building" width="400">

```
User opens app → GET /v1/me/feed
      |
      v
[Load Balancer]
      |
      v
[Web Servers]
      |
      v
[News Feed Service]
      |
      v
[News Feed Cache]   ← Pre-computed list of post IDs for this user
      |
   (Fetch full post details for each ID)
      |
      v
[Post Cache / Post DB]
      |
      v
Return feed to user
```

> **Key insight:** The News Feed Cache stores a **pre-computed list of post IDs** for each user. When you open the app, we just look up your list (super fast!) and then fetch the post content for each ID.

> **Question:** Why store post IDs and not full post content in the feed cache?  
> **Answer:** Memory efficiency. A post can have 10KB of text + image URLs. If you have 5,000 friends each posting once a day, storing full content for every user's feed would require terabytes of cache memory. By storing only IDs (8 bytes each), you keep the cache lean. Fetch content separately as needed.

---

## Step 5: The Fanout Service — The Core Challenge

This is where the magic (and complexity) lives.

### What is "fanout"?

> When Alice posts something, we need to add that post to the feed of all of Alice's friends. This "spreading out" from one publisher to many recipients is called **fanout**.

> **Analogy:** Imagine you send a newsletter. The "fanout" is the process of printing a copy for each subscriber and putting it in their mailbox. One newsletter, thousands of mailboxes.

### Fanout Deep Dive

<img src="../../system-design-notes/11. News Feed System/images/fanout-service.png" alt="Fanout Service" width="400">

Here's the step-by-step when Alice publishes a post:

**Step 1 — Get friend IDs:**
> Query the **Graph DB** to get Alice's friend list. A graph database is perfect for relationship data (who follows whom) because it's optimized for traversing connections.

**Step 2 — Get friends' notification settings:**
> Check the **User Cache** for each friend. Has any friend muted Alice? Is Alice's post visible to all friends? Filter the list accordingly.

**Step 3 — Push to Message Queue:**
> Send `{post_id: 123, recipient_user_ids: [friend1, friend2, ...]}` to a **Message Queue** (like Kafka). 

**Step 4 — Fanout Workers process the queue:**
> Workers pull from the queue and for each friend, add `post_id:123` to their **News Feed Cache** entry.

**Step 5 — News Feed Cache is updated:**
> Each friend's pre-computed feed now includes Alice's post. Next time they open the app, it's already there — instant!

```
Alice posts → Fanout Service
                  |
           ① Get friend IDs from Graph DB
           ② Filter by user settings (muted, private, etc.)
           ③ Push to Message Queue
                  |
           ④ Fanout Workers consume queue
                  |
           ⑤ Write post_id to each friend's feed cache
```

> **Question:** Why use a Message Queue instead of directly updating caches?  
> **Answer:** Speed and reliability. If Alice has 3,000 friends, updating 3,000 cache entries synchronously would take too long — Alice would see a spinner waiting for her post to "publish." By pushing to a queue and returning immediately, Alice sees "Post published!" in milliseconds. Workers update caches asynchronously in the background.

---

## Step 6: Fanout on Write vs. Fanout on Read

This is the **most important tradeoff** in news feed design and a guaranteed interview question.

### Fanout on Write (Push Model)

> When a post is created, immediately push it to all followers' feed caches.

```
Write time:  Alice posts → update 3,000 friend caches  ← Work happens HERE
Read time:   Open app → read from cache                ← Instant!
```

**Pros:**
- Feed reads are instant (cache is pre-built)
- Real-time updates

**Cons:**
- Slow/expensive writes for users with many friends
- Wasted work if friends never open the app

### Fanout on Read (Pull Model)

> Don't pre-compute anything. When a user opens the app, compute their feed on the fly.

```
Write time:  Alice posts → save to DB only             ← Cheap!
Read time:   Open app → query all friends' posts       ← Work happens HERE
```

**Pros:**
- Cheap writes
- No wasted computation for inactive users

**Cons:**
- Slow feed loads (query + aggregate + sort on every open)
- Expensive at high read volumes

### The Hybrid Approach (What Real Systems Use)

> **Question:** So which one should we use?  
> **Answer:** Both — for different user types.

| User type | Strategy | Why? |
|-----------|----------|------|
| Regular users (< 10K followers) | Fanout on Write | Fast reads, writes are cheap enough |
| Celebrities (> 10K followers) | Fanout on Read | Writing to 1M caches per post is too expensive |

**How it works at read time for a user following both regular users and celebrities:**
```
Fetch feed = [pre-built cache (regular users' posts)]
           + [real-time query for celebrity posts]
           → Merge and sort
           → Return to user
```

> **Example:** You follow 200 regular people and Taylor Swift. When you open Instagram:
> - Your pre-built cache has the 200 regular posts ready
> - The system separately fetches Taylor's last 5 posts in real time
> - They're merged, sorted by time, and returned as your unified feed

---

## Step 7: Cache Architecture — 5 Layers

When reading a feed, many types of data are needed. The cache is split into 5 layers for efficiency:

<img src="../../system-design-notes/11. News Feed System/images/cache-architecture.png" alt="Cache Architecture" width="400">

| Layer | What it stores | Example |
|-------|---------------|---------|
| **News Feed Cache** | Pre-computed list of post IDs per user | `user:101 → [post:55, post:42, post:31, ...]` |
| **Content Cache** | Full post content (hot = popular posts) | `post:55 → {text: "...", img_url: "...", author: ...}` |
| **Social Graph Cache** | Who follows whom | `user:101:friends → [201, 305, 407, ...]` |
| **Action Cache** | Likes, replies, shares per post | `post:55:actions → {liked: true, replied: false}` |
| **Counter Cache** | Like count, reply count, follower count | `post:55:likes → 1,247` |

> **Question:** Why not just cache everything in one big blob?  
> **Answer:** Different data has different update frequencies and access patterns. Like counts change every second; post content rarely changes. Separating them lets us set appropriate TTLs (Time-To-Live) and invalidation strategies for each type.

> **Question:** What's a "hot cache"?  
> **Answer:** Viral posts (a celebrity's post with millions of views) are read millions of times per minute. "Hot cache" keeps these in memory with the highest priority, possibly replicated across more Redis nodes, to serve that demand without hitting the database.

---

## Step 8: Full Architecture — Publishing a Post

<img src="../../system-design-notes/11. News Feed System/images/feed-publishing-deep-dive.png" alt="Feed Publishing Deep Dive" width="400">

Here's the complete write path with all components:

```
User → POST /v1/me/feed (content + auth_token)
              |
       [Load Balancer]
              |
       [Web Servers]
       - Validate auth_token
       - Rate limit (prevent spam)
              |
    __________|__________________________
   |          |                         |
[Post      [Fanout               [Notification
 Service]   Service]              Service]
   |             |                     |
[Post Cache] ① Get friend IDs      Send push
   |            from Graph DB      notifications
[Post DB]    ② Check User Cache
             (filter muted/blocked)
             ③ → Message Queue
                      |
                 ④ Fanout Workers
                      |
                 ⑤ News Feed Cache
                 (update each friend's feed)
```

---

## Step 9: Handling Scale

### Math Check

- 10M DAU, assume 1 feed read/day per user = ~116 reads/sec average, ~580/sec peak
- If 1% of users post per day = 100K posts/day
- Average 200 followers = 20M cache writes/day from fanout = ~231/sec

> This is very manageable with Redis + a small fanout worker pool. Horizontal scaling (add more workers and Redis nodes) handles 10x growth.

### The Celebrity Problem (Thundering Herd)

> **Scenario:** Elon Musk has 100 million followers. He posts a tweet. Without the hybrid model, the fanout service would need to write to 100 million feed caches — all at once. This would:
> - Overwhelm the message queue
> - Spike the fanout workers to thousands of instances
> - Take minutes to complete

**Solutions:**
1. **Hybrid fanout** (as described above) — skip fanout for celebrities entirely
2. **Rate-limit the fanout worker** — spread writes over 30–60 seconds instead of instantly
3. **Cache the celebrity post object** with high replication — millions of reads for the same post should all hit a heavily cached object

### Database Scaling

| Data | Storage choice | Why? |
|------|---------------|------|
| Posts | PostgreSQL or DynamoDB | Structured data, can shard by `user_id` |
| News Feed (pre-computed) | Redis sorted sets | O(log N) range queries by timestamp score |
| Social graph (follows) | Graph DB or `(follower_id, followee_id)` table | Optimized for relationship traversal |
| Media (images, video) | S3 + CDN | Object storage for large files, CDN for edge delivery |

> **Why Redis sorted sets for feed cache?**  
> A sorted set stores items with a score. Use `timestamp` as the score and `post_id` as the value. Fetching "last 20 posts" = `ZREVRANGE user:101:feed 0 19` — a single O(log N) command. This is orders of magnitude faster than a SQL query with ORDER BY + LIMIT.

---

## Step 10: Pagination — Infinite Scroll

> **Question:** How does "infinite scroll" work technically?

### Bad approach: Offset Pagination
```sql
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 200
```
This gets slower and slower as `OFFSET` grows (database has to skip 200 rows every time).

### Good approach: Cursor-Based Pagination
```
First load:    GET /v1/me/feed?limit=20
               → Returns 20 posts + cursor="post_id:55"

Next scroll:   GET /v1/me/feed?limit=20&cursor=post_id:55
               → Returns 20 posts BEFORE post_id:55 + new cursor
```

The cursor is the ID of the last post the user saw. The server uses it as a starting point for the next query — no offset, no slowdown.

> **Analogy:** Instead of "go to page 47 of this encyclopedia," you're saying "show me everything after the bookmark I left."

---

## Step 11: Additional Features

### Handling Deleted Posts

> **Problem:** You fan out post_id:123 to 5,000 friend caches. Then the author deletes the post. Now 5,000 caches have a reference to a deleted post.

**Solution — Lazy deletion:**
1. Soft-delete the post in the DB (`deleted_at` timestamp, not actually removed)
2. Leave the cache references alone
3. When rendering the feed, if fetching post_id:123 returns "deleted," skip it silently
4. Asynchronously, a cleanup job removes dead post IDs from feed caches

> **Why not immediately clean up all caches?** Because finding and removing one `post_id` from 5,000 users' Redis entries is expensive. Lazy deletion is simpler and has the same user experience.

### Privacy-Sensitive Posts (Friends Only)

Each post has a visibility level: `public`, `friends`, or `group`.

- On fanout: only push to users who match the visibility rule
- On render: re-check visibility (defense in depth — in case privacy changed after fanout)

> **Question:** Why re-check at render time if we already filtered at fanout time?  
> **Answer:** Race condition. If Alice posts as "friends-only" and then 10 seconds later changes it to "only me," the fanout may have already distributed it. Re-checking at render time catches these post-fanout privacy changes.

### New User Cold Start Problem

> **Problem:** A brand new user has zero follows. Their feed is empty. What do you show them?

**Solutions:**
1. **Onboarding flow:** Ask interests → suggest popular accounts to follow
2. **Trending feed:** Show globally popular posts until the user's social graph builds up
3. **Import contacts:** Find friends already on the platform via phone/email

---

## Step 12: Complete Architecture Summary

```
                        User (Web / Mobile)
                               |
                          [CDN] ← Static assets, media
                               |
                       [Load Balancer]
                               |
                  ┌────────────────────────┐
                  │      Web Servers        │
                  │  (Auth + Rate Limiting) │
                  └──────────┬─────────────┘
                             |
             ________________|___________________
            |                |                   |
      [Post Service]  [Fanout Service]   [Notification Service]
            |                |
      [Post Cache]    ① Graph DB (friend IDs)
      [Post DB]       ② User Cache (settings/filters)
                      ③ Message Queue (Kafka)
                             |
                      ④ Fanout Workers
                             |
                      ⑤ News Feed Cache (Redis)
                             |
                     (read path: fetch post IDs)
                             |
                      Post Cache → Post DB

    ─────────────────────────────────────────────────
    Cache Layers (Redis):
    ┌──────────────┬─────────────────────────────────┐
    │ News Feed    │ [post_id_1, post_id_2, ...]      │
    │ Content      │ hot posts | normal posts          │
    │ Social Graph │ follower list | following list    │
    │ Action       │ liked | replied | shared          │
    │ Counters     │ like count | reply count          │
    └──────────────┴─────────────────────────────────┘
```

---

## Summary: The "Why" Behind Each Decision

| Decision | Why? |
|----------|------|
| Pre-compute feed in cache | Feed reads are ~100x more frequent than writes; pay cost at write time |
| Store post IDs in feed cache (not full content) | Memory efficiency; content cached separately by post_id |
| Message Queue between fanout service and workers | Absorbs traffic spikes; decouples writer from cache updaters |
| Hybrid fanout (push for regular, pull for celebs) | Avoid thundering herd from celebrity posts |
| Graph DB for social relationships | Optimized for "who follows whom" traversal |
| Cursor-based pagination | Consistent performance regardless of page depth |
| Redis sorted sets for feed | O(log N) range queries by timestamp; fast pagination |
| Soft-delete posts | Eager cache cleanup across thousands of feeds is too expensive |
| 5-layer cache architecture | Different data types have different access patterns and TTLs |
| CDN for media | Serve images/video from edge nodes near users; reduce origin load |

---

## Quick Interview Q&A Cheat Sheet

**Q: How do you design a news feed for 10M DAU?**  
A: Two flows — write (post → post service → fanout service → message queue → fanout workers → feed cache) and read (app open → news feed service → feed cache → post cache → return feed). The fanout pre-computes feeds so reads are instant.

**Q: What is fanout on write vs. fanout on read?**  
A: Write = push post to all friends' caches when posting (fast reads, expensive writes). Read = compute feed from all friends' posts when loading (cheap writes, slow reads). Use hybrid: write for regular users, read for celebrities.

**Q: What database for the social graph?**  
A: Graph DB (Neo4j) or a `follows` table `(follower_id, followee_id)` indexed on both columns. Graph DBs excel at "friends of friends" traversals.

**Q: How does infinite scroll work?**  
A: Cursor-based pagination. Client sends the ID of the last seen post with each scroll request. Server returns the next batch starting from that cursor. O(log N) per query, consistent speed regardless of scroll depth.

**Q: How do you handle a deleted post that was already fanned out?**  
A: Soft-delete (set `deleted_at`). Leave feed cache entries alone. At render time, skip any post that returns deleted from the content cache. Async cleanup job removes dead IDs eventually.

**Q: What's the thundering herd problem?**  
A: A celebrity posts → naive fanout writes to millions of caches at once → spikes the message queue and workers. Solution: skip fanout for celebrities entirely; merge their posts at read time from a separate query (hybrid model).

**Q: Why use Redis sorted sets for the feed cache?**  
A: `ZREVRANGE user:101:feed 0 19` fetches the 20 newest post IDs in O(log N). Timestamp is the score, post_id is the value. Far faster than any SQL ORDER BY + LIMIT on a large table.
