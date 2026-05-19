# System Design: Proximity Service (Beginner-Friendly Guide)

---

## What is a Proximity Service?

A **proximity service** answers the question: *"What businesses are near me right now?"*

<img src="../../system-design-notes/16. Proximity Service/images/2d-search.png" alt="Nearby Businesses" width="280">

**Real-world examples:**
- Yelp — "Find restaurants within 1 km"
- Google Maps — "Gas stations near me"
- Uber — "Find drivers within 500m"
- Airbnb — "Homes near this location"

> **Why is this hard?**  
> You have 200 million businesses spread across the entire Earth. When a user asks "what's within 500 meters of me?", you need to answer in milliseconds. A naive database scan of 200 million rows on every request would be impossibly slow. The whole challenge is: **how do you search 2D space (latitude + longitude) efficiently?**

---

## Step 1: Understand What We're Building

### Functional Requirements

| Feature | Detail |
|---------|--------|
| Search nearby | Given (lat, long, radius) → return nearby businesses |
| Business CRUD | Owners can add/update/delete their businesses |
| Business details | Users can view full info about a specific business |

### Non-Functional Requirements

| Requirement | Why |
|-------------|-----|
| Low latency | Users expect instant results |
| High availability | Peak hours in dense city areas can spike traffic |
| Data privacy | Comply with GDPR and CCPA for location data |

### Scale

- **100 million DAU**
- **200 million businesses**
- **5,000 QPS** for nearby searches (100M users × 5 searches/day ÷ 86,400 sec)

> **5,000 searches per second** for location queries. Each query needs to scan and rank nearby businesses. This is the core scaling challenge.

---

## Step 2: The APIs

### Search Nearby
```
GET /v1/search/nearby?latitude=37.77&longitude=-122.41&radius=500
```
Returns: list of businesses sorted by distance

### Business Management
```
GET    /v1/businesses/{id}     → get business details
POST   /v1/businesses          → add a new business
PUT    /v1/businesses/{id}     → update business info
DELETE /v1/businesses/{id}     → remove a business
```

> **Question:** Why are location search and business management separate APIs?  
> **Answer:** They have totally different read/write patterns. Search is read-only, extremely high volume (5,000 QPS), and must be fast. Business management is low volume (owners rarely update) and writes are fine to be slightly delayed. Mixing them would mean slow business writes could affect fast search reads.

---

## Step 3: High-Level Architecture

The system splits into **two services**:

<img src="../../system-design-notes/16. Proximity Service/images/high-level-design.png" alt="High Level Design" width="450">

```
User
  |
[Load Balancer]
  |                    |
[LBS]            [Business Service]
(Location          - Add/update/delete businesses
 Based Service)    - Read business details
  |                    |
  |              [Database Cluster]
  |              (Primary → Replicas)
  |
[Read from Replicas]
```

### Location-Based Service (LBS)

- **Read-only** — no writes at all
- Answers "what businesses are near coordinates X,Y?"
- Stateless — easy to scale horizontally
- **High QPS** (5,000/sec) — needs very fast lookups

### Business Service

- Handles all CRUD for business data
- **Writes go to primary DB** (low volume — owners rarely update)
- **Reads go to replicas** (viewing business details)

### Database: Primary-Replica Architecture

One primary database accepts all writes. Multiple replica databases serve all reads. If the primary fails, a replica is promoted.

> **Question:** Is it a problem if the replicas are slightly behind the primary?  
> **Answer:** No — and this is an important insight. Business information (address, hours, phone number) doesn't change in real-time. A new restaurant added today appearing in search results 1–2 seconds later is completely acceptable. This eventual consistency is fine for this use case.

---

## Step 4: The Core Problem — How to Search 2D Space Efficiently

This is the hardest part of the whole design. Let's go through the approaches from bad to best.

### Approach 1: Naive SQL Query (Too Slow)

```sql
SELECT business_id, latitude, longitude
FROM business
WHERE latitude  BETWEEN (user_lat - radius) AND (user_lat + radius)
  AND longitude BETWEEN (user_long - radius) AND (user_long + radius);
```

> **Why is this slow?**  
> Database indexes work on **one dimension** at a time. Even with indexes on latitude AND longitude, the DB has to scan all rows matching the latitude range, then filter for longitude (or vice versa). At 200 million businesses with 5,000 QPS, this kills the database.

The core problem: **we need to convert 2D coordinates into something 1D that we can index efficiently.**

### The Solution: Geospatial Indexing

<img src="../../system-design-notes/16. Proximity Service/images/geospatial-index-types.png" alt="Geospatial Index Types" width="450">

There are two families of approaches:

| Family | Methods |
|--------|---------|
| **Hash-based** | Even Grid, Geohash |
| **Tree-based** | Quadtree, Google S2, R-Tree |

The two most practical and commonly used in interviews are **Geohash** and **Quadtree**.

---

## Step 5: Approach 2 — Even Grid (Too Simple)

<img src="../../system-design-notes/16. Proximity Service/images/even-grid.png" alt="Even Grid" width="420">

Divide the entire world into a fixed grid of equal-sized cells. Each cell has an ID. Store each business with its cell ID. To search, find which cell you're in and query businesses in that cell + neighboring cells.

> **Problem:** This completely ignores population density. Manhattan has 70,000 businesses per square km; rural Wyoming has 1 per square km. Fixed-size cells mean Manhattan cells have millions of entries (slow to search) while Wyoming cells are nearly empty (wasted storage). You can't make the cells both small enough for cities and large enough to be useful in rural areas.

---

## Step 6: Approach 3 — Geohash (Recommended for Interviews)

Geohash is the most popular approach for proximity search. Here's how it works, step by step.

### Step 1: Split the World in Half Repeatedly

<img src="../../system-design-notes/16. Proximity Service/images/geohash.png" alt="Geohash Step 1" width="320">

Start with the entire Earth:
- Split vertically (left = longitude < 0, right = longitude ≥ 0)
- Left half gets bit `0`, right half gets bit `1`
- Split horizontally (bottom = latitude < 0, top = latitude ≥ 0)
- Bottom gets bit `0`, top gets bit `1`

So after 2 splits, the Earth has 4 quadrants with 2-bit codes:
- Top-left (Europe/Asia): `01`
- Top-right (N. America): `11`
- Bottom-left (Antarctica): `00`
- Bottom-right (S. America/Australia): `10`

### Step 2: Keep Subdividing

<img src="../../system-design-notes/16. Proximity Service/images/geohash-1.png" alt="Geohash Subdivision" width="300">

Keep splitting each cell into 4 smaller cells, alternating between longitude and latitude splits. After each round, add 2 more bits to the code.

After several rounds, cells are tiny (city blocks, then street-level). The **longer the geohash string, the smaller and more precise the cell.**

### Step 3: Convert Binary to Base32

The binary bit string is converted to a **short alphanumeric string** using base32 encoding:

```
Binary: 01 11 01 01 01 10 ...
Base32: 9q9hvt (example geohash for San Francisco)
```

### Key Property: Shared Prefix = Nearby Locations

<img src="../../system-design-notes/16. Proximity Service/images/geohash-radius-mapping.png" alt="Geohash Radius Mapping" width="420">

> The longer the shared prefix between two geohashes, the closer the two locations are.

| Geohash length | Cell size | Use for radius |
|----------------|-----------|----------------|
| 4 characters | ~156 km × 156 km | 20 km radius |
| 5 characters | ~4.9 km × 4.9 km | 1–5 km radius |
| 6 characters | ~1.2 km × 1.2 km | 0.5–1 km radius |

**Example:**
```
You are at:        9q9hvt  (San Francisco street)
Nearby restaurant: 9q9hvu  (1 block away)
Shared prefix:     9q9hv   (length 5 = both in same ~5km cell)

Far away:          dqcjq   (New York)
Shared prefix:     (none)
```

### Geohash Indexing in the Database

Instead of storing `(latitude, longitude)` and doing 2D queries, store the **geohash string** and index it:

```sql
CREATE INDEX idx_geohash ON businesses(geohash);

-- Fast query: all businesses in cell 9q9hvt and neighbors
SELECT * FROM businesses WHERE geohash IN ('9q9hvt', '9q9hvu', '9q9hvy', ...);
```

This is a single-column string index — extremely fast.

### The Boundary Problem

<img src="../../system-design-notes/16. Proximity Service/images/boundary-issue.png" alt="Boundary Issue" width="320">

> **Problem:** Two restaurants can be 10 meters apart but sit on opposite sides of a geohash boundary — they'd have completely different geohash strings (no shared prefix).

> **Solution:** Always search the target geohash **plus its 8 neighbors** (the surrounding cells). This guarantees you catch businesses near the edges of your cell.

```
Search these 9 geohashes:
[center_cell, north, south, east, west, NE, NW, SE, SW]
```

This adds only 8 extra DB lookups — still very fast with caching.

---

## Step 7: Approach 4 — Quadtree (Deep Dive)

<img src="../../system-design-notes/16. Proximity Service/images/building-quadtree.png" alt="Building Quadtree" width="480">

A **quadtree** is an in-memory tree data structure that recursively divides 2D space into four quadrants.

### How It's Built

Start with the entire world as the root. If a region has more than **100 businesses**, split it into 4 quadrants. Keep splitting recursively until every leaf node has ≤ 100 businesses.

```
World (200M businesses) → too many, split
  ├── NW quadrant (50M) → too many, split
  │     ├── NW-NW (12M) → split...
  │     ├── NW-NE (4M)  → split...
  │     ├── NW-SW (5M)  → split...
  │     └── NW-SE (9M)  → split...
  ├── NE quadrant (30M) → too many, split...
  ├── SW quadrant (70M) → too many, split...
  └── SE quadrant (60M) → too many, split...
```

<img src="../../system-design-notes/16. Proximity Service/images/quadtree.png" alt="Quadtree" width="450">

**Result:** Dense cities like NYC get subdivided into tiny cells (many levels deep). Rural areas stop splitting early because each region already has ≤ 100 businesses. The tree **automatically adapts to population density**.

### How Searches Work

To find businesses near (lat, long):
1. Traverse the tree from root to the leaf containing your coordinates
2. Return the businesses in that leaf
3. If fewer than k results, expand to parent and include sibling leaves

### Real-World Result

<img src="../../system-design-notes/16. Proximity Service/images/realworld-quadtree.png" alt="Real World Quadtree" width="420">

Notice how city centers (Denver downtown) have tiny cells (many businesses) while suburbs have large cells (fewer businesses). This is the quadtree adapting automatically.

### Quadtree Operational Notes

- The quadtree is **built in memory** on each LBS server at startup — not in a database
- 200 million businesses → tree builds in a few minutes (O(N log N))
- **During build, the server can't serve traffic** → roll out new builds gradually (not all servers at once)
- When a business is added/deleted: easiest to **rebuild the tree incrementally** (or full rebuild nightly)

> **Question:** Why build it in memory instead of storing it in a database?  
> **Answer:** The whole point is speed. Traversing a tree in memory takes microseconds. Querying a tree structure stored in a database requires multiple disk reads — too slow for 5,000 QPS. The tree fits comfortably in a few GB of RAM on modern servers.

---

## Step 8: Geohash vs. Quadtree — Which to Use?

| Criteria | Geohash | Quadtree |
|----------|---------|---------|
| Implementation | Simple — just string operations | Harder — need to build/maintain a tree |
| Fixed radius search | Excellent | Good |
| K-nearest search | Harder | Excellent (traverse up until k found) |
| Density-aware | No (fixed cell size) | Yes (auto-splits dense areas) |
| Updating index | Easy — just update a string column | Complex — may need to rebuild tree |
| Storage | In DB with string index | In-memory on LBS servers |
| **Best for** | "Restaurants within 1km" | "Find the 10 nearest gas stations" |

> **Interview tip:** For most proximity search problems (Yelp, Google Maps), **Geohash is the recommended answer** — simpler, well-understood, easy to implement, and works great with Redis caching. Mention Quadtree as the alternative when density adaptation matters.

---

## Step 9: Google S2 — Honorable Mention

<img src="../../system-design-notes/16. Proximity Service/images/hilbert-curve.png" alt="Hilbert Curve" width="320">

Google S2 maps the sphere to a 1D index using a **Hilbert curve** — a space-filling curve that preserves locality (nearby points on the curve are nearby on the Earth).

<img src="../../system-design-notes/16. Proximity Service/images/geofence.png" alt="Geofence" width="380">

**Best for: Geofencing** — defining arbitrary polygon shapes on a map (not just circles) and checking if a point is inside.

> **Example:** "Alert me when a delivery driver enters this irregular polygon around my neighborhood" — S2 handles this perfectly. Geohash is limited to rectangular cells.

---

## Step 10: Caching Strategy

### What to Cache

| Cache Key | Cache Value | Why? |
|-----------|------------|------|
| `geohash` | List of `business_id`s in that cell | Avoid DB hit for every prefix search |
| `business_id` | Full business details (name, address, rating) | Avoid DB hit for every business lookup |

**Cache Tier:** Two Redis clusters:
1. **Geohash Redis** — maps geohash → list of business IDs
2. **Business Info Redis** — maps business_id → full business details

> **Question:** Why not use latitude/longitude as the cache key?  
> **Answer:** GPS coordinates are slightly imprecise and change as the user moves. Two users 1 meter apart have slightly different coordinates → cache misses. A geohash is a stable string for an entire geographic cell — any location in that cell hits the same cache entry.

---

## Step 11: Final Architecture

<img src="../../system-design-notes/16. Proximity Service/images/final-design.png" alt="Final Design" width="500">

```
User (lat=37.77, long=-122.41, radius=500m)
              |
       [Load Balancer]
         /          \
     [LBS]      [Business Service]
       |              |         \
    ① Calculate geohash     [Write]  [Read]
      length for 500m → 6    |          |
    ② Get my geohash + 8   [Primary] [Replicas]
       neighbors
    ③ Query Geohash Redis   ←←←  [Sync from DB]
       for each geohash
    ④ Get business IDs
    ⑤ Query Business Info Redis
       for each business ID
    ⑥ Sort by distance from user
    ⑦ Return ranked list
```

### Step-by-Step Request Flow

**User searches: restaurants within 500m at (37.776720, -122.416730)**

1. Request hits Load Balancer → routed to LBS
2. LBS looks up the geohash length table: 500m → length **6**
3. LBS computes the geohash for user's location: `9q8zn9`
4. LBS computes 8 neighboring geohashes: `[9q8zn6, 9q8znd, 9q8znf, 9q8zn3, 9q8znc, 9q8zn2, 9q8znb, 9q8zn8]`
5. LBS queries **Geohash Redis** for each of the 9 geohashes **in parallel** → gets lists of business IDs
6. LBS queries **Business Info Redis** for each business ID → gets full details
7. LBS calculates exact distance from user for each business, sorts by distance
8. Returns top N results to user

> **Why query 9 geohashes (center + 8 neighbors)?**  
> Because a restaurant 400m away might be in an adjacent geohash cell. Searching only your own cell would miss it. Searching 9 cells guarantees you find all businesses within the radius regardless of which side of a cell boundary they're on.

---

## Step 12: Scaling

### LBS Scaling

- Stateless → add more servers behind the load balancer as QPS grows
- Deploy in multiple **regions** (US, EU, Asia) to reduce latency globally
- Quadtree (if used) is built on each server independently at startup

### Database Scaling

- **Business table:** Shard by `business_id` for even distribution
- **Geohash index table:** Fits on one server (200M rows × ~50 bytes ≈ ~10 GB) → no sharding needed, just add read replicas
- Add replicas as read load grows

### Cache Scaling

- Redis cluster with slots distributed across nodes
- Geohash cache hit rate is high — same city areas queried constantly
- Business info cache TTL: ~10 minutes (business details rarely change)

---

## Summary: The "Why" Behind Each Decision

| Decision | Why? |
|----------|------|
| Separate LBS and Business Service | Different read/write patterns; don't let low-volume writes affect high-QPS reads |
| Geospatial indexing (Geohash/Quadtree) | SQL 2D range queries are too slow at 200M businesses and 5K QPS |
| Geohash preferred over even grid | Even grid can't adapt to density; cities vs. rural areas need different cell sizes |
| Search center geohash + 8 neighbors | Prevent missing businesses near cell boundaries |
| Geohash length matched to radius | Longer geohash = smaller cell; match cell size to search radius |
| Redis for geohash and business cache | Sub-millisecond lookups; geohash is stable key (unlike GPS coordinates) |
| Quadtree in memory (not DB) | Tree traversal in RAM is microseconds; DB tree traversal requires disk reads |
| Read replicas for DB | LBS is read-heavy (5K QPS); replicas serve reads, primary handles rare writes |
| Eventual consistency acceptable | Business data (address, hours) doesn't change in real-time; 1-2 sec delay is fine |

---

## Quick Interview Q&A Cheat Sheet

**Q: What data structure would you use for proximity search?**  
A: Geohash for most cases. Encode (lat, long) into a short alphanumeric string. Longer shared prefix = closer locations. Index the geohash string in a DB or Redis. Search your cell + 8 neighbors to handle boundary cases.

**Q: Why not just use a SQL query with lat/long range?**  
A: DB indexes are 1D. A 2D range query still requires scanning all rows matching one dimension, then filtering the other. Too slow at 200M businesses and 5K QPS.

**Q: How do you pick geohash length?**  
A: Use the radius-to-length mapping table. 500m → length 6 (~1.2km cells), 5km → length 4 (~156km cells). Smaller radius = longer geohash (more precise cell).

**Q: What is the boundary problem in Geohash and how do you fix it?**  
A: Two locations 10m apart can have completely different geohashes if they're on opposite sides of a cell boundary. Fix: always search the target geohash plus its 8 neighboring geohashes.

**Q: Geohash vs. Quadtree — when to use which?**  
A: Geohash: simpler, great for fixed-radius search (Yelp-style). Quadtree: harder to implement, better for k-nearest search and adapts to population density automatically. Geohash is the right default for most interview questions.

**Q: How does the final query flow work?**  
A: User sends (lat, long, radius) → LBS calculates geohash length from radius → computes center geohash + 8 neighbors → queries Geohash Redis for business IDs (parallel) → queries Business Info Redis for details → calculates exact distances → sorts → returns results.
