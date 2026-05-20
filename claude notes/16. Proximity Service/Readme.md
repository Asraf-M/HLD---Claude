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

Geohash converts any (latitude, longitude) on Earth into a **short string like `9q9hvt`**. Nearby locations produce similar strings. This lets you find nearby places using a simple string index — no complex 2D math needed.

---

### The Core Idea: Divide the World Like a Grid, Encode as a String

Think of it like this: you keep folding a map in half, and at each fold you write down which half you're on. After enough folds, the sequence of "left/right, top/bottom" choices uniquely identifies a tiny cell on Earth. That sequence is your geohash.

---

### Step 1: Split the World in Half Repeatedly

<img src="../../system-design-notes/16. Proximity Service/images/geohash.png" alt="Geohash Step 1" width="320">

**Round 1 — Longitude split (left/right):**
```
Entire Earth longitude range: -180 to +180
Split at 0:
  Left  (-180 to 0):  bit = 0  (Western hemisphere — Americas)
  Right (0 to +180):  bit = 1  (Eastern hemisphere — Europe, Asia)

San Francisco is at longitude -122.4  →  falls in Left half  →  bit = 0
```

**Round 2 — Latitude split (top/bottom):**
```
Entire Earth latitude range: -90 to +90
Split at 0:
  Bottom (-90 to 0):  bit = 0  (Southern hemisphere)
  Top    (0 to +90):  bit = 1  (Northern hemisphere)

San Francisco is at latitude 37.7  →  falls in Top half  →  bit = 1
```

After 2 splits, San Francisco is in the `01` quadrant (longitude=0, latitude=1).

**Round 3 — Longitude split again (now within the left half: -180 to 0):**
```
Left half splits at -90:
  Left  (-180 to -90):  bit = 0  (Far west — Hawaii, Alaska edge)
  Right (-90 to 0):     bit = 1  (Most of Americas)

San Francisco is at -122.4  →  falls in Left (-180 to -90)  →  bit = 0
```

**Round 4 — Latitude split (now within top half: 0 to 90):**
```
Top half splits at 45:
  Bottom (0 to 45):   bit = 0  (Tropics and mid-latitudes)
  Top    (45 to 90):  bit = 1  (Northern Europe, Canada, Russia)

San Francisco is at 37.7  →  falls in Bottom (0 to 45)  →  bit = 0
```

So far San Francisco has bits: `0  1  0  0` — and the cell is already narrowed from the whole Earth to a region of roughly ~2,500 km wide. Keep going for more precision.

---

### Step 2: Keep Subdividing — More Bits = More Precision

<img src="../../system-design-notes/16. Proximity Service/images/geohash-1.png" alt="Geohash Subdivision" width="300">

The pattern always alternates: **longitude bit, latitude bit, longitude bit, latitude bit...**

Each pair of bits cuts the current cell into 4 smaller cells. After 30 bits (15 pairs), you have a cell about 1 metre wide — precise enough for a front door.

```
Bits so far for San Francisco area:
  0  1  0  0  1  1  0  1  0  1  1  0  0  1  0  1  ...
  ^lon ^lat ^lon ^lat ...
  (continuing splits narrow down to street-level precision)
```

The key insight: **every additional bit halves the cell size in one direction**.

---

### Step 3: Convert Binary to Base32 (Readable String)

30 bits of binary is hard to read. Geohash groups the bits into **5-bit chunks** and maps each to one of 32 characters:

```
Base32 alphabet: 0123456789bcdefghjkmnpqrstuvwxyz
(note: a, i, l, o are excluded to avoid confusion with 0 and 1)

Binary:  01001  10101  10010  ...
         ↓      ↓      ↓
Base32:  9      q      9      ...  → "9q9hvt"
```

So `9q9hvt` is San Francisco's geohash at precision level 6 (6 characters = 30 bits = cell ~1.2km x 0.6km).

---

### Key Property: Shared Prefix = Nearby Locations

<img src="../../system-design-notes/16. Proximity Service/images/geohash-radius-mapping.png" alt="Geohash Radius Mapping" width="420">

Because every character encodes a region subdivision, **locations that share a longer prefix are in the same (smaller) region**.

**Concrete example — San Francisco restaurants:**
```
You (standing at Union Square):   9q8yy1
Starbucks (100m away):            9q8yy3
In-N-Out Burger (300m away):      9q8yy9
Restaurant (2km away):            9q8yz4
Restaurant (10km away):           9q8zxx
Restaurant in Los Angeles:        9q5c2t
Restaurant in New York:           dr5reg

Shared prefix length:
  You vs Starbucks (100m):   9q8yy   → 5 chars shared → same ~5km cell
  You vs In-N-Out (300m):    9q8yy   → 5 chars shared → same ~5km cell
  You vs 2km away:           9q8y    → 4 chars shared → same ~40km cell
  You vs Los Angeles (600km):9q       → 2 chars shared → same ~1200km cell
  You vs New York (4000km):  (none)  → different macro-region
```

**Precision table:**
| Geohash Length | Cell Width | Cell Height | Use For |
|---|---|---|---|
| 1 char | 5,000 km | 5,000 km | Continent-level |
| 2 chars | 1,250 km | 625 km | Country-level |
| 3 chars | 156 km | 156 km | State/region |
| 4 chars | 39 km | 20 km | City-level |
| **5 chars** | **4.9 km** | **4.9 km** | **Neighbourhood (use for ~5km radius)** |
| **6 chars** | **1.2 km** | **0.6 km** | **Street-level (use for ~1km radius)** |
| 7 chars | 152 m | 152 m | Building-level |

For a "find restaurants within 1km" feature, use **geohash length 6**.

---

### How You Actually Use It — Full Example

**Scenario:** User is at lat=37.7749, lon=-122.4194 (San Francisco). Find all restaurants within 1 km.

**Step 1: Compute the user's geohash**
```
lat=37.7749, lon=-122.4194  →  geohash = "9q8yy1"  (length 6)
```

**Step 2: Find the 8 surrounding neighbour cells**
```
Every geohash cell has exactly 8 neighbours (N, NE, E, SE, S, SW, W, NW):

  +--------+--------+--------+
  | 9q8yxr | 9q8yy2 | 9q8yy3 |   ← NW,  N,  NE
  +--------+--------+--------+
  | 9q8yxp | 9q8yy1 | 9q8yy4 |   ← W,  YOU,  E
  +--------+--------+--------+
  | 9q8yxj | 9q8yy0 | 9q8yy5 |   ← SW,  S,  SE
  +--------+--------+--------+
```

**Step 3: Query the database for all 9 cells**
```sql
SELECT business_id, name, lat, lon
FROM businesses
WHERE geohash IN (
  '9q8yy1',   -- your cell
  '9q8yy2', '9q8yy4', '9q8yy0', '9q8yxp',  -- N, E, S, W
  '9q8yy3', '9q8yy5', '9q8yxj', '9q8yxr'   -- NE, SE, SW, NW
);
```

This returns all businesses in a ~3.6km x 1.8km area (3x3 grid of 1.2km cells).

**Step 4: Filter by exact distance in application code**
```java
List<Business> candidates = db.queryGeohashCells(nineGeohashes);

// Filter: keep only those within 1km using Haversine formula
return candidates.stream()
    .filter(b -> haversineDistance(userLat, userLon, b.getLat(), b.getLon()) <= 1000)
    .sorted(Comparator.comparingDouble(b -> haversineDistance(...)))
    .collect(toList());
```

The DB does a fast index scan. Your app does the precise distance check on the small candidate set (~50-200 businesses).

---

### The Boundary Problem (Important!)

<img src="../../system-design-notes/16. Proximity Service/images/boundary-issue.png" alt="Boundary Issue" width="320">

#### What is the Boundary Problem?

Every geohash cell has edges — imaginary lines drawn across the map. The boundary problem happens when **two locations are physically very close to each other, but the cell boundary line runs between them**. Because they are in different cells, their geohash strings look different even though they are nearly neighbours.

---

#### Example 1: Two Coffee Shops Separated by a Line

Imagine you are standing on a pavement. The geohash cell boundary (an invisible grid line) happens to run right through the middle of your street.

```
                  cell boundary
                       |
  Cell: 9q8yy1         |       Cell: 9q8yy4
                       |
  [YOU]  ----------20m-+--  [Coffee Shop B]
                       |
  [Coffee Shop A]      |
  (inside your cell)   |
```

- **Coffee Shop A** is in cell `9q8yy1` — same cell as you. Distance: 50m.
- **Coffee Shop B** is in cell `9q8yy4` — different cell. Distance: 20m. **Closer to you!**

Now your app searches only your cell `9q8yy1`:

```sql
SELECT * FROM businesses WHERE geohash = '9q8yy1';
-- Returns: Coffee Shop A (50m away)  ✅
-- MISSES:  Coffee Shop B (20m away)  ❌
```

Coffee Shop B is **closer** to you but completely invisible to the query. Your app would show the 50m option and miss the 20m option — wrong result.

---

#### Example 2: Corner Case — Four Cells Meet at One Point

The worst case is a **corner**, where four cells all meet at a single point. A location right in your corner is only 1 metre away, but it's in a completely different cell.

```
  Cell: 9q8yxr  |  Cell: 9q8yy2
  ───────────────+───────────────
  Cell: 9q8yxp  |  Cell: 9q8yy1
                         ↑
                        YOU are here (bottom-right corner of your cell)

  [Pharmacy] is 1 metre away — at the exact corner — in cell 9q8yxr
```

Geohashes of the four corners:
```
9q8yxr  ←  pharmacy is HERE (1m away, top-left cell)
9q8yy2  ←  top-right cell
9q8yxp  ←  bottom-left cell
9q8yy1  ←  YOU are here
```

`9q8yxr` and `9q8yy1` share only `9q8y` (4 chars) — they look far apart in the string. But physically they are 1 metre apart.

---

#### Why Does This Happen?

Think of it like ZIP codes. Two houses can be on opposite sides of a ZIP code boundary — one has ZIP 10001, the other has ZIP 10002 — even though they are next-door neighbours. Searching only ZIP 10001 misses the house next door.

Geohash cells have the same issue. The grid is fixed. The real world doesn't care where the lines are drawn.

---

#### The Fix: Always Search 9 Cells (Your Cell + 8 Neighbours)

Instead of querying only your cell, always query your cell **plus all 8 surrounding cells**. This creates a 3×3 grid of cells around you.

```
  +──────────+──────────+──────────+
  | 9q8yxr   | 9q8yy2   | 9q8yy3   |  ← NW, N, NE
  +──────────+──────────+──────────+
  | 9q8yxp   | 9q8yy1   | 9q8yy4   |  ← W, YOU, E
  +──────────+──────────+──────────+
  | 9q8yxj   | 9q8yy0   | 9q8yy5   |  ← SW, S, SE
  +──────────+──────────+──────────+
```

Now query all 9:

```sql
SELECT * FROM businesses
WHERE geohash IN (
  '9q8yy1',                              -- your cell
  '9q8yy2', '9q8yy4', '9q8yy0', '9q8yxp',  -- N, E, S, W
  '9q8yy3', '9q8yy5', '9q8yxj', '9q8yxr'   -- NE, SE, SW, NW
);
```

Now **no matter where the boundary falls**, any business within the 3×3 area is included:

```
  +──────────+──────────+──────────+
  |  ☕ A    |          |          |
  +──────────+──────────+──────────+
  |          |  [YOU]   | ☕ B     |  ← B is now found ✅
  +──────────+──────────+──────────+
  |          |    ☕ C  |          |
  +──────────+──────────+──────────+
```

All three coffee shops — A, B, and C — are returned regardless of which cell boundary they sit next to.

---

#### But Doesn't This Return Too Many Results?

Yes — the 3×3 grid is rectangular and larger than the circle you actually want. That is fine. You use the DB to quickly get a **candidate set**, then filter precisely in your application code using the **Haversine formula** (exact straight-line distance):

```
DB query   →  returns ~200 businesses from 9 cells (fast, index scan)
     ↓
App code   →  for each business, calculate exact metres from user
     ↓
Filter     →  keep only those within 1000m
     ↓
Result     →  20 restaurants, correctly sorted by distance  ✅
```

The DB does the coarse filter. Your code does the precise filter. Two steps, both fast.

---

#### Summary

| Situation | Without 9-cell search | With 9-cell search |
|---|---|---|
| Business in same cell | Found ✅ | Found ✅ |
| Business just across boundary | **Missed ❌** | Found ✅ |
| Business at a corner (4 cells meet) | **Missed ❌** | Found ✅ |
| Business far away | Not returned ✅ | Not returned ✅ |

**Rule to remember:** Always query `your_cell + 8 neighbours`. Never query just your own cell.

---

### Geohash in the Database

Store the geohash as a plain string column and index it:

```sql
-- Schema
CREATE TABLE businesses (
    id          BIGINT PRIMARY KEY,
    name        VARCHAR(255),
    lat         DOUBLE,
    lon         DOUBLE,
    geohash     VARCHAR(12),   -- store at max precision, query at length 6
    ...
);

CREATE INDEX idx_geohash ON businesses(geohash);
```

The query uses a B-Tree index on a string column — it is as fast as looking up a username. No special spatial index needed. This is why geohash works on any standard SQL or NoSQL database.

---

### Why Geohash is Recommended for Interviews

| Property | Why it matters |
|---|---|
| **Simple concept** | "Divide the world in halves, encode as a string" — easy to explain in 2 minutes |
| **Standard SQL index** | `WHERE geohash IN (...)` — works on MySQL, PostgreSQL, DynamoDB, Cassandra |
| **Cacheable** | A geohash cell is a fixed region — cache "all restaurants in cell 9q8yy1" by key |
| **Scalable** | Partition your DB by geohash prefix — all businesses in a region go to one shard |
| **Predictable cell sizes** | You know exactly what precision level to use for a given radius |
| **No special infra** | Unlike Quadtree which needs in-memory tree servers, geohash just needs a DB column |

**The one limitation to mention:** geohash cells are rectangles, not circles. A radius search with geohash returns a rectangular region — you post-filter with Haversine to get exact results. Always mention this in interviews.

---

## Step 7: Approach 4 — Quadtree (Deep Dive)

<img src="../../system-design-notes/16. Proximity Service/images/building-quadtree.png" alt="Building Quadtree" width="480">

A **quadtree** is an in-memory tree that recursively divides a 2D map into four quadrants. Think of it like zooming into a map — each zoom level splits the view into 4 tiles, and each tile can be split again if it has too many points.

---

### Part 1: Building the Tree

**The rule:** If a region contains more than **100 businesses**, split it into 4 equal quadrants. Keep splitting until every region has ≤ 100 businesses.

Let's use a simplified city example with just 16 businesses spread unevenly — 12 in the downtown area, 4 in the suburbs:

```
The city map (each dot = a business):

  +──────────────────────────+
  |          |               |
  |  suburb  |  ● ● ●  CBD  |
  |    ●     |  ● ● ●       |
  |          |  ● ● ●       |
  |──────────|──────────────|
  |          |  ● ● ●  CBD  |
  |  suburb  |              |
  |    ●     |              |
  |    ●     |              |
  +──────────────────────────+
```

**Step 1: Root — entire city (16 businesses)**
16 > 100? No, this is a toy example. Let's say the threshold is **4** for illustration.
16 > 4 → **SPLIT into NW, NE, SW, SE**

```
Root: whole city (16 businesses)
  ├── NW quadrant: 1 business   ≤ 4 → LEAF (no more splitting)
  ├── NE quadrant: 9 businesses > 4 → SPLIT again
  ├── SW quadrant: 2 businesses ≤ 4 → LEAF
  └── SE quadrant: 4 businesses ≤ 4 → LEAF
```

**Step 2: Split the NE quadrant (9 businesses > 4)**

```
NE quadrant (9 businesses)
  ├── NE-NW: 2 businesses ≤ 4 → LEAF
  ├── NE-NE: 3 businesses ≤ 4 → LEAF
  ├── NE-SW: 3 businesses ≤ 4 → LEAF
  └── NE-SE: 1 business  ≤ 4 → LEAF
```

**Final tree structure:**

```
                  ROOT (whole city)
                 /   |    |    \
              NW    NE    SW    SE
            (LEAF) (node)(LEAF)(LEAF)
              1biz  /|\ \  2biz  4biz
                   / | \ \
               NE-NW NE-NE NE-SW NE-SE
               (LEAF)(LEAF)(LEAF)(LEAF)
               2biz  3biz  3biz  1biz
```

Key observation: **the dense downtown area (NE) got subdivided into smaller cells. The sparse suburbs (NW, SW) stayed as large cells.** This is what makes the quadtree density-aware.

---

### Part 2: What Each Node Stores

Every node in the tree stores:

```
Internal Node (has children):
  - bounding box:  top-left corner, bottom-right corner
  - 4 child pointers: NW, NE, SW, SE

Leaf Node (no children):
  - bounding box:  top-left corner, bottom-right corner
  - business list: [ business_id_1, business_id_2, ... ]  (max 100)
```

Example leaf node for the NE-NW quadrant:
```
Leaf {
  boundingBox: { topLeft: (lat=40.75, lon=-74.00), bottomRight: (lat=40.73, lon=-73.98) }
  businesses: [
    { id: 101, name: "Joe's Pizza",   lat: 40.748, lon: -73.995 },
    { id: 102, name: "NYC Pharmacy",  lat: 40.744, lon: -73.991 },
  ]
}
```

---

### Part 3: User Search — Root to Leaf Traversal

**Scenario:** You are standing at **lat=40.748, lon=-73.996** (Midtown Manhattan). You want nearby businesses.

The quadtree is loaded in memory on the server. Here is exactly what happens:

---

**Step 1: Start at the ROOT node**

```
ROOT bounding box: covers the entire city
  top-left:     (lat=40.80, lon=-74.10)
  bottom-right: (lat=40.60, lon=-73.80)

Question: which quadrant does (40.748, -73.996) fall into?

  Midpoint lat = (40.80 + 40.60) / 2 = 40.70
  Midpoint lon = (-74.10 + -73.80) / 2 = -73.95

  User lat 40.748 > midpoint 40.70  →  NORTH half
  User lon -73.996 < midpoint -73.95 →  WEST half

  →  Go to NE quadrant  (north + west = NE in this coordinate system)
     Wait... let me re-clarify:
     North (lat > midpoint) + West (lon < midpoint) = NW quadrant

  → Go to NW child
```

Actually let's simplify with clear directional labelling:

```
ROOT splits at centre point (40.70, -73.95):

  lat > 40.70 AND lon < -73.95  →  NW  (north-west)
  lat > 40.70 AND lon > -73.95  →  NE  (north-east)
  lat < 40.70 AND lon < -73.95  →  SW  (south-west)
  lat < 40.70 AND lon > -73.95  →  SE  (south-east)

User is at (40.748, -73.996):
  40.748 > 40.70  ✅ north
  -73.996 < -73.95 ✅ west
  → Go to NW child  ↓
```

---

**Step 2: Arrive at NW node — is it a leaf?**

```
NW node bounding box: (40.70 to 40.80) latitude, (-74.10 to -73.95) longitude
Children: NW-NW, NW-NE, NW-SW, NW-SE  ← has children, NOT a leaf

NW centre point: (40.75, -74.025)

User (40.748, -73.996):
  40.748 < 40.75  →  south
  -73.996 > -74.025 →  east
  → Go to NW-SE child  ↓
```

---

**Step 3: Arrive at NW-SE node — is it a leaf?**

```
NW-SE bounding box: (40.70 to 40.75) latitude, (-74.025 to -73.95) longitude
No children → THIS IS A LEAF ✅

Businesses stored here:
  [
    { id: 101, name: "Joe's Pizza",       lat: 40.748, lon: -73.995 },
    { id: 102, name: "NYC Pharmacy",      lat: 40.744, lon: -73.991 },
    { id: 103, name: "Central Park Cafe", lat: 40.742, lon: -74.001 },
  ]
```

**Total traversal: 3 comparisons (root → NW → NW-SE). Done in microseconds since it's all in memory.**

---

**Step 4: Return the businesses from this leaf**

The 3 businesses in this leaf are your initial candidates. The server then calculates the exact distance from you to each one:

```
You are at (40.748, -73.996)

Joe's Pizza       (40.748, -73.995) → distance = 80m   ✅ within 500m
NYC Pharmacy      (40.744, -73.991) → distance = 450m  ✅ within 500m
Central Park Cafe (40.742, -74.001) → distance = 680m  ❌ too far
```

Result returned: Joe's Pizza and NYC Pharmacy.

---

### Part 4: What If the Leaf Has Too Few Results?

Sometimes you land on a leaf with only 2 businesses but the user wants 20 results. In that case, **walk back up to the parent** and include the sibling leaves.

**Example:** You want `k=10` nearest businesses but your leaf only has 3.

```
Step 1: Land on leaf NW-SE → 3 businesses. Need 10. Not enough.
Step 2: Go UP to parent node NW.
        NW has 4 children: NW-NW, NW-NE, NW-SW, NW-SE
        Collect businesses from ALL 4 children:
          NW-NW: 2 businesses
          NW-NE: 4 businesses
          NW-SW: 1 business
          NW-SE: 3 businesses  ← you were here
        Total: 10 businesses ✅ enough!

Step 3: Calculate exact distance to all 10, sort, return top 10.
```

If still not enough, go up one more level to the root's NW-NE grandchildren, and so on. Keep expanding until you have k results.

```
Not enough results? → Go to parent → collect all sibling leaves
Still not enough?   → Go to grandparent → collect all aunt/uncle leaves
Keep going until k results are found
```

---

### Part 5: The Real World — NYC vs Rural Wyoming

This is where the quadtree shines over a fixed grid.

```
Manhattan (dense):
  Root → NE → NE-NW → NE-NW-SE → NE-NW-SE-NE → ... (many levels deep)
  Each leaf covers about 1 city block
  Each leaf has ~80-100 businesses

Rural Wyoming (sparse):
  Root → SW  ← THIS IS ALREADY A LEAF
  The entire state is one leaf with 40 businesses
  No further splitting needed
```

With geohash, Wyoming would have the same cell precision as Manhattan (wasteful). With quadtree, Wyoming stays as one big cell because there is no reason to split it further.

---

### Part 6: Full Traversal Diagram

```
You: lat=40.748, lon=-73.996 (Midtown Manhattan)

ROOT (whole city)
 ├── Is (40.748, -73.996) in this node's bounds? YES
 ├── Has children? YES
 ├── Which child? → NW (lat > mid, lon < mid)
 │
 └──► NW node
       ├── Is (40.748, -73.996) in this node's bounds? YES
       ├── Has children? YES
       ├── Which child? → NW-SE (lat < mid, lon > mid)
       │
       └──► NW-SE node  ← LEAF
             ├── Has children? NO — stop here
             └── Return businesses:
                   Joe's Pizza      80m  ✅
                   NYC Pharmacy     450m ✅
                   Central Park Cafe 680m ❌ (filtered out)
```

**Total: 2 pointer hops, 2 comparisons, all in RAM — takes ~1 microsecond.**

---

### Quadtree Operational Notes

- Built **entirely in memory** on the LBS (Location-Based Service) server at startup
- 200 million businesses → builds in a few minutes, fits in ~a few GB of RAM
- **During build, the server cannot serve traffic** → roll out new builds gradually (one server at a time)
- Business added/deleted → easiest to rebuild the tree incrementally or do a full nightly rebuild

> **Why in memory and not in a DB?**  
> Traversing the tree takes 2–3 pointer hops in RAM = microseconds. If the tree were in a DB, each hop would be a disk read = milliseconds. At 5,000 QPS that difference matters enormously.

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
