# System Design: Search Autocomplete (Beginner-Friendly Guide)

---

## What is Search Autocomplete?

When you type "tw" in Google's search box and it immediately shows "twitter", "twitch", "twilight" — that's **search autocomplete** (also called typeahead or incremental search).

<img src="../../system-design-notes/13. Search Autocomplete/images/basic-search-suggestions.png" alt="Search Autocomplete Example" width="350">

**Real-world examples:**
- Google Search suggestions
- YouTube search bar
- Amazon product search
- Spotify song/artist search

> **Why is this hard to build?**  
> A billion users type every day. Each keystroke triggers a request. You need to return the top 5 most popular completions in **under 100ms** — consistently, at scale. A naive database query on every keystroke would be too slow.

---

## Step 1: Understand What We're Building

### Requirements

| Requirement | Detail |
|-------------|--------|
| Results shown | Top 5 suggestions |
| Ranking | By popularity (how often people searched it) |
| Characters | Lowercase English only |
| Response time | < 100ms |
| Scale | 10 million DAU, peak 48,000 QPS |
| Storage growth | ~0.4 GB of new query data per day |

> **Quick math:** 48,000 queries per second means if you're just 5ms late on a DB call, your response time SLA is already broken. You need data in memory, not on disk.

### Two Core Services

The whole system splits into two independent parts:

1. **Data Gathering Service** — Collects what people search and how often
2. **Query Service** — Answers "what should I autocomplete?" in real time

> **Question:** Why separate these two?  
> **Answer:** They have totally different needs. The query service needs to be blazing fast (< 100ms). The data gathering service can be slower — it's running batch jobs in the background. Mixing them would mean slow batch processing could slow down your real-time suggestions. Separate them!

---

## Step 2: The Core Data Structure — Trie (Prefix Tree)

Before diving into the full architecture, you need to understand the **trie** — it's the heart of every autocomplete system.

### What is a Trie?

A trie (rhymes with "try") is a **tree where each path from root to a leaf spells out a word**.

<img src="../../system-design-notes/13. Search Autocomplete/images/trie-structure.png" alt="Trie Structure" width="500">

**Example:** For words "tree", "true", "try", "toy", "wish", "win":

```
           root
          /    \
         t      w
        / \      \
       tr   to    wi
      / \    \   /  \
    tre  tru  toy wis  win
     |    |    |   |
   tree  true toy wish
```

> **Analogy:** Think of it like a file system directory. If you go to `/t/r/u/e/`, you've spelled "true." All words starting with "tr" live under the `/t/r/` directory.

### Why Not Just Use a Database with LIKE?

```sql
-- Bad approach
SELECT query FROM searches WHERE query LIKE 'tw%' ORDER BY frequency DESC LIMIT 5;
```

> **Problem:** At 48,000 QPS, a database scanning millions of rows on every keystroke would crash. A trie lookup is O(prefix_length) — traversing just the characters you've typed. That's 2-5 operations for a typical prefix, not millions.

### How the Trie Stores Frequency

Each complete word in the trie has a **frequency count** (how many times users searched it):

```
t → tr → try: 29
         tree: 10
         true: 35
      to → toy: 14
w → wi → wis → wish: 25
         win: 50
```

When you type "tr", you find the "tr" node and look at all words in its subtree. The top 5 by frequency are your suggestions.

---

## Step 3: Optimizing the Trie

### Problem: Finding Top-5 is Slow

Without optimization, getting the top 5 results for prefix "t" requires traversing the **entire subtree** under "t" — which could be millions of words. Too slow.

### Solution: Cache Top-K at Every Node

<img src="../../system-design-notes/13. Search Autocomplete/images/cached-trie.png" alt="Cached Trie" width="500">

Pre-compute and store the **top 5 results at each node**:

```
node "b"  → cache: [best:35, bet:29, bee:20, be:15, buy:14]
node "be" → cache: [best:35, bet:29, bee:20, be:15, beer:10]
node "bee"→ cache: [bee:20, beer:10]
```

Now when you type "be", you hit the "be" node and immediately return its cached top-5 — **zero traversal needed**. O(prefix_length) total.

> **Trade-off:** The cache uses more memory (each node stores 5 entries), but it makes read queries instant. Since reads far outnumber writes (nobody is updating the trie during the day), this is the right trade-off.

### How the Trie is Stored in a Database (KV Store)

<img src="../../system-design-notes/13. Search Autocomplete/images/trie-db.png" alt="Trie DB" width="500">

The trie is serialized into a key-value store:

| Key | Value (top-K cached) |
|-----|---------------------|
| `b` | [be:15, bee:20, beer:10, best:35] |
| `be` | [be:15, bee:20, beer:10, best:35] |
| `bee` | [bee:20, beer:10] |
| `bes` | [best:35] |
| `beer` | [beer:10] |
| `best` | [best:35] |

> **Why a KV store and not a document DB?**  
> The access pattern is purely: "give me the value for key `be`." That's a KV store's exact strength — O(1) lookup. No complex queries needed.

---

## Step 4: Data Gathering Service — How Do We Know What's Popular?

### Naive approach (too slow)

Update the trie in real-time every time someone searches.

> **Why this fails:**  
> Users make billions of queries per day. Rebuilding or updating the trie on every query would be constant, expensive writes to the trie — which is also being read by millions of users simultaneously. Writes and reads would conflict, causing locks and slowdowns.

### Better approach: Batch Processing Pipeline

<img src="../../system-design-notes/13. Search Autocomplete/images/data-gathering-flow.png" alt="Data Gathering Flow" width="500">

Instead of updating in real-time, run a batch job **weekly** (or daily for time-sensitive use cases like Twitter):

```
Step 1: Analytics Logs
   → Every search query is logged: {query: "twitter", timestamp: ..., user_id: ...}
   → Logs are append-only (fast writes, no index needed)

Step 2: Aggregators (Spark/MapReduce job)
   → Processes logs: count how many times each query was searched
   → Produces a frequency table:
```

<img src="../../system-design-notes/13. Search Autocomplete/images/data-gathering.png" alt="Data Gathering" width="500">

**Frequency table example:**

<img src="../../system-design-notes/13. Search Autocomplete/images/frequency-table.png" alt="Frequency Table" width="400">

```
Step 3: Workers
   → Read the frequency table
   → Build a brand new trie from scratch
   → Store it in Trie DB

Step 4: Trie DB → Trie Cache
   → A weekly snapshot of the DB is loaded into in-memory cache
   → This is what serves real-time queries
```

> **Question:** Isn't weekly too slow? What if something trends today?  
> **Answer:** For most search systems (Google, Amazon), weekly is fine — popular queries don't change drastically day-to-day. For Twitter/news, you'd aggregate every hour or use a real-time layer that blends trending queries with the weekly trie.

### How Frequency Accumulates Over Time

<img src="../../system-design-notes/13. Search Autocomplete/images/data-gathering.png" alt="Data Gathering Steps" width="500">

The log accumulates counts from all searches:
- After 1st search for "twitch": `{twitch: 1}`
- After searching "twitter": `{twitch: 1, twitter: 1}`
- After searching "twitter" again: `{twitch: 1, twitter: 2}`
- And so on...

---

## Step 5: Query Service — Serving Suggestions in Real Time

When a user types "tw":

```
① Browser: debounce for 100ms (don't send on every keystroke)
② Send: GET /api/v1/autocomplete?query=tw
③ Load Balancer routes to an Autocomplete Server
④ Autocomplete Server looks up "tw" in Trie Cache (in-memory)
⑤ Returns cached top-5 for "tw" → ["twitter", "twitch", "twilight", "twin peak", "twitch prime"]
⑥ Browser displays suggestions
```

### Query Flow

```
User types "tw"
      |
      v
[Browser] — debounce 100ms →
      |
      v
[Load Balancer]
      |
      v
[Autocomplete Server]
      |
 Check Trie Cache (Redis / in-memory)
      |
  Cache HIT → return top-5 instantly ✓
  Cache MISS → query Trie DB → cache result → return
```

> **Question:** What is "debounce"?  
> **Answer:** If the user types "t", "w", "i", "t", "t", "e", "r" rapidly, you don't want to fire 7 separate API requests. Debouncing waits until the user **stops typing for 100ms**, then fires one request. This cuts server load dramatically.

---

## Step 6: Filtering — Removing Bad Suggestions

<img src="../../system-design-notes/13. Search Autocomplete/images/delete-kv.png" alt="Delete KV Filter" width="450">

What if a harmful phrase becomes very popular and gets into the trie?

**Two-step approach:**
1. **Filter Layer:** Sits in front of the Trie Cache. Before returning results to the user, the filter removes any blocked terms from the suggestions.
2. **Async Cleanup:** A background job physically removes the blocked term from the Trie DB. The next weekly rebuild won't include it either.

```
Trie Cache → [Filter Layer] → API Servers → User

Filter Layer checks suggestions against a blocklist:
If suggestion matches blocklist → drop it before returning
```

> **Why not delete from the cache immediately?**  
> Deleting a term from the trie means finding every node path that leads to it and cleaning up the cached top-K at each ancestor node. This is complex and risky to do live. The filter layer gives instant blocking while the proper cleanup happens safely in the background.

---

## Step 7: Scalability — Sharding the Trie

At 48,000 QPS, one server can't handle all requests. You need to split the trie across multiple servers.

### Sharding by Prefix Range

<img src="../../system-design-notes/13. Search Autocomplete/images/sharding.png" alt="Sharding" width="400">

Split the alphabet across servers:
- Shard 1: queries starting with `a–m`
- Shard 2: queries starting with `n–z`

A **Shard Map Manager** tracks which shard handles which prefix range:

```
User types "tw" →
      |
      v
Web Server asks Shard Map Manager: "What shard handles 'tw'?"
Shard Map Manager: "Shard 2 (n-z)"
      |
      v
Web Server queries Shard 2 → returns top-5
```

> **Problem:** The letter distribution isn't even. "S", "C", "A" have many more searches than "X", "Q", "Z". A naive alphabetical split leaves some shards overloaded.

> **Solution:** Analyze the data first, then split by traffic, not alphabet. If "s-z" has 40% of traffic, put it on 2 servers while "a-c" gets 1 server.

---

## Step 8: Full Architecture

```
                    ┌─────────────────────────────┐
                    │     OFFLINE (Weekly Batch)   │
                    │                              │
  Analytics Logs →  │  Aggregators (Spark)         │
  (every search     │       ↓                      │
   is logged)       │  Frequency Table             │
                    │       ↓                      │
                    │  Workers (build new Trie)     │
                    │       ↓                      │
                    │  Trie DB (MongoDB / KV Store) │
                    │       ↓                      │
                    │  Trie Cache (in-memory Redis) │
                    └─────────┬───────────────────┘
                              │ weekly snapshot
                              ↓
    ┌─────────────────────────────────────────────┐
    │            ONLINE (Real-Time Queries)        │
    │                                              │
    │  User types "tw"                             │
    │       ↓                                      │
    │  Browser (debounce 100ms)                    │
    │       ↓                                      │
    │  Load Balancer                               │
    │       ↓                                      │
    │  Autocomplete Servers (read from Trie Cache) │
    │       ↓                                      │
    │  [Filter Layer] ← blocklist                  │
    │       ↓                                      │
    │  Return top-5 to user                        │
    └─────────────────────────────────────────────┘
```

### Additional Optimizations

**Browser caching:**
```
GET /autocomplete?query=tw
Response headers: Cache-Control: max-age=3600
```
If the user types "tw" again within an hour, the browser uses its local cache — zero server request.

**CDN caching:**
Single-character prefixes ("t", "s", "a") are searched millions of times per minute. Cache these at CDN edge nodes globally. Most users get their suggestion results from the CDN without ever hitting your servers.

---

## Step 9: Handling Special Cases

### Trending Queries (Breaking News)

The weekly trie is stale for real-time trends. Solution — a **two-layer blending approach**:

```
Result = top-K from (weekly Trie)
       + top-K from (real-time trending layer in Redis sorted set)
       → merge and re-rank
       → return top-5
```

The real-time layer tracks query counts in a 1-hour sliding window in Redis. If "earthquake" suddenly spikes, it surfaces in autocomplete within minutes.

### Multi-Language Support

Build **separate trie instances** per language. Detect language from:
- Browser's `Accept-Language` header
- User's profile setting
- Script detection (Latin vs. Cyrillic vs. Chinese characters)

Route to the appropriate trie at query time.

### Trie Rebuild Without Downtime

> **Question:** If the trie is being rebuilt weekly, do users get an outage during the swap?  
> **Answer:** No. Old and new trie exist simultaneously. The old trie keeps serving traffic. Once the new one is built and validated, the routing layer atomically swaps to the new trie (pointer swap). If the new trie fails validation, the swap is aborted and old trie keeps running. Zero downtime.

---

## Summary: The "Why" Behind Each Decision

| Decision | Why? |
|----------|------|
| Trie instead of SQL LIKE query | O(prefix_length) lookup vs. O(table_size) scan; must be < 100ms at 48K QPS |
| Cache top-K at every node | Avoids subtree traversal; makes every lookup O(prefix_length) |
| Weekly batch rebuild (not real-time updates) | Billions of queries/day makes real-time trie updates impractical; top queries are stable week-to-week |
| Separate data gathering and query service | Different scaling needs; batch jobs shouldn't affect real-time latency |
| KV store for trie serialization | Prefix lookup = exact key lookup; O(1), perfect KV use case |
| In-memory Trie Cache | Disk is ~1000x slower than RAM; 100ms SLA requires in-memory |
| Debounce on client | Reduces server requests by ~5x; users type faster than we need to query |
| Filter layer in front of cache | Instant blocking of harmful content without risky live trie modification |
| Shard by prefix range | One trie is too large for one server; split by prefix across nodes |
| CDN for common prefixes | Single-char prefixes hit millions of times/day; CDN absorbs most load |

---

## Quick Interview Q&A Cheat Sheet

**Q: What data structure powers autocomplete?**  
A: A **trie** (prefix tree). Each node represents a character; traversing root→node spells a prefix. Top-K suggestions are cached at each node for O(prefix_length) lookup.

**Q: How do you rank suggestions?**  
A: By search frequency. Batch-process historical query logs weekly (via Spark/MapReduce), count occurrences per query, store the top-K by frequency at each trie node.

**Q: How do you handle 48,000 QPS in under 100ms?**  
A: (1) Trie in-memory (Redis / application memory); (2) CDN caches common prefixes; (3) Browser caches results with TTL; (4) Client debounces to reduce requests. No disk I/O on the hot path.

**Q: How is the trie updated without downtime?**  
A: Build a new trie in the background from fresh frequency data. Once complete and validated, atomically swap the pointer to the new trie. Old trie stays live until the swap. Zero downtime.

**Q: How do you handle a harmful query that needs to be removed immediately?**  
A: A filter layer sits in front of the trie cache and blocks results matching a real-time blocklist. This takes effect instantly. Async background job removes the term from the trie DB permanently; next rebuild also excludes it.

**Q: Why not store the trie in a SQL database?**  
A: SQL `LIKE 'tw%'` requires a full index scan at scale. Trie lookup is O(prefix_length) — just 2 character traversals for "tw". At 48K QPS, the SQL approach would require enormous DB clusters while the trie approach runs fine on a handful of in-memory servers.

**Q: What is the difference between the data gathering and query services?**  
A: Data gathering is the offline pipeline — collects logs, aggregates frequencies weekly, rebuilds the trie, stores it in DB/cache. Query service is the online path — takes user input, looks up the trie cache, returns top-5 in < 100ms. They are completely decoupled so batch processing never affects live query latency.
