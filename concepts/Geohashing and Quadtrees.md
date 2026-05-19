# Geohashing and Quadtrees

## The Core Problem

> "Find all restaurants within 2 km of my location."

This is a **proximity search** — given a lat/lng coordinate, find nearby entities efficiently.

Naively scanning every restaurant in a DB and calculating distance is O(N) — unusable at scale.

We need a way to **spatially index** locations so we can quickly narrow down candidates.

Two main approaches: **Geohashing** and **Quadtrees**.

---

## Geohashing

### What Is It?

Geohashing converts a **2D coordinate (lat, lng)** into a **1D string** such that nearby locations share a common prefix.

```
Eiffel Tower (48.8584, 2.2945) → geohash: "u09tvw0"
Louvre Museum (48.8606, 2.3376) → geohash: "u09tvy4"
Tokyo (35.6762, 139.6503) → geohash: "xn76urwe"
```

Paris locations both start with "u09tv" → same geographic area.  
Tokyo starts with "xn" → different region entirely.

### How It Works (Conceptually)

The Earth is divided recursively by bisecting lat/lng ranges, encoding each decision as a bit.

```
Step 1: Is longitude in [-180, 0] or [0, 180]? → 0 or 1
Step 2: Is latitude in [-90, 0] or [0, 90]?    → 0 or 1
Step 3: Bisect again...
```

After 30+ bits, the resulting binary string is Base-32 encoded into a short string (the geohash).

### Geohash Precision

| Length | Cell Size | Use Case |
|--------|-----------|---------|
| 1 | 5,000 km × 5,000 km | Country level |
| 3 | 156 km × 156 km | Region |
| 5 | 4.9 km × 4.9 km | City district |
| 6 | 1.2 km × 0.6 km | Neighborhood |
| **7** | **152 m × 152 m** | **Street level (most common for proximity)** |
| 8 | 38 m × 19 m | Building |
| 9 | 4.8 m × 4.8 m | Room |

**For "find nearby restaurants within 500m"** → use geohash prefix of length 6 (1.2km cell).

### The Boundary Problem

Two points very close to each other can have completely different geohashes if they're on opposite sides of a grid boundary.

```
Point A: 48.8580, 2.2940 → "u09tvu"
Point B: 48.8580, 2.2941 → "u09tvv"  ← different last char!
```

These points are 1 meter apart but share only 5-character prefix.

**Solution:** Also check the **8 neighboring cells** around the user's cell.

```
[NW] [N ] [NE]
[W ] [ME] [E ]   ← always search these 9 cells
[SW] [S ] [SE]
```

```python
import geohash2
neighbors = geohash2.neighbors("u09tv")
# Returns 8 surrounding geohash cells
search_cells = [current_geohash] + neighbors
```

### Database Query with Geohash

```sql
-- Find all restaurants in user's cell and 8 neighbors
SELECT * FROM restaurants
WHERE geohash LIKE 'u09tv%'  -- prefix query on indexed column
  AND geohash IN ('u09tv', 'u09tw', 'u09ty', ...)

-- Then filter by actual distance in application code
```

Index on `geohash` column → prefix scan is fast (B-tree index supports LIKE 'prefix%').

### When to Use Geohash

✅ Simple to implement (single string column, standard SQL)  
✅ Efficient range queries (prefix LIKE on B-tree index)  
✅ Easy to cache at cell level ("all restaurants in cell u09tv")  
✅ Works well with Redis (store geohash as sorted set key)

❌ Boundary problem requires checking 9 cells  
❌ Fixed grid cells — can't adjust density dynamically  
❌ Cells are rectangles, not circles — approximation  

---

## Quadtrees

### What Is It?

A **quadtree** recursively divides a 2D space into four quadrants until each cell contains at most a threshold number of points (e.g., 100).

```
World
├── NW quadrant
│   ├── NW-NW (sparse → leaf)
│   ├── NW-NE (sparse → leaf)
│   ├── NW-SW (dense → subdivide)
│   │   ├── NW-SW-NW (leaf)
│   │   ├── NW-SW-NE (leaf)
│   │   └── ...
│   └── NW-SE (sparse → leaf)
├── NE quadrant (NYC, London — dense → deeply subdivided)
├── SW quadrant
└── SE quadrant
```

### Building a Quadtree

```
Start with entire world as root node (capacity = 100 points)
For each business/point:
  Insert into correct leaf node
  If node exceeds capacity → split into 4 children, redistribute points
```

Result: dense areas (NYC, Tokyo) get many tiny cells; sparse areas (oceans, deserts) keep large cells.

### Search in a Quadtree

```
User at (48.858, 2.294) with radius 1km:

1. Start at root
2. Which quadrant contains the point? → descend
3. Does this node's bounding box intersect the search circle? → keep
4. Is it a leaf? → return its points
5. Repeat for all intersecting children
```

Typical depth: 10–15 levels for city-level precision.  
Search time: O(log N) to find the node + O(k) to return k results.

### When to Use Quadtrees

✅ **Adaptive density** — dense areas get finer cells automatically  
✅ Better than geohash for very uneven point distributions (cities vs. oceans)  
✅ Range queries are natural (bounding box intersection)  
✅ Efficient for "all restaurants in this bounding box"  

❌ More complex to implement (tree structure, not a simple string)  
❌ Usually built in-memory on servers (not directly in SQL)  
❌ Tree must be rebuilt or rebalanced when points change frequently  

---

## Geohash vs Quadtree

| | Geohash | Quadtree |
|--|---------|---------|
| **Data structure** | String prefix in DB | In-memory tree |
| **Cell shape** | Fixed rectangles (same size at same precision) | Variable (adaptive to density) |
| **Density adaption** | ❌ Fixed grid | ✅ Subdivides dense areas |
| **Implementation** | Simple (SQL + index) | Complex (tree operations) |
| **Boundary problem** | Yes (check 9 neighbors) | Handled naturally |
| **Cache-friendly** | ✅ Cache by cell prefix | ❌ Harder |
| **Used by** | Uber, PostGIS, Redis GEO | Google Maps, game engines |

---

## Redis GEO Commands (Geohash Under the Hood)

Redis has built-in geospatial commands using geohash internally:

```bash
# Add locations
GEOADD restaurants 2.2945 48.8584 "eiffel_cafe"
GEOADD restaurants 2.3376 48.8606 "louvre_bistro"

# Find within radius
GEORADIUS restaurants 2.2945 48.8584 1 km ASC
# Returns: ["eiffel_cafe", "louvre_bistro"]

# Get geohash of a location
GEOHASH restaurants "eiffel_cafe"
# Returns: "u09tvw0"
```

All stored in a **sorted set** with geohash as the score → O(log N) inserts, O(log N + k) radius queries.

---

## System Design: Proximity Service (Yelp / Uber)

### Write Path
```
Business registers location
  → Hash lat/lng to geohash (e.g., precision=6)
  → Store in DB: { business_id, lat, lng, geohash }
  → Update Redis sorted set (geohash as score)
```

### Read Path
```
User requests "restaurants near me"
  → Hash user's lat/lng to geohash
  → Query 9 cells (user's cell + 8 neighbors) from Redis
  → Filter results by actual Haversine distance
  → Return sorted by distance
```

### Haversine Distance Formula

Great-circle distance between two points on Earth:

$$d = 2r \arcsin\left(\sqrt{\sin^2\left(\frac{\Delta\phi}{2}\right) + \cos(\phi_1)\cos(\phi_2)\sin^2\left(\frac{\Delta\lambda}{2}\right)}\right)$$

In practice: use a DB function (`ST_Distance` in PostGIS) or a library. The geohash/quadtree narrows candidates → Haversine filters the exact set.

---

## Real-World Usage

| Product | Approach | Details |
|---------|----------|---------|
| **Uber** (driver matching) | Geohash | Drivers indexed by geohash; matched to riders in same cell |
| **Yelp** (nearby search) | Quadtree | Adaptive to city density |
| **Google Maps** | Quadtree + S2 geometry | S2 library for spherical geometry cells |
| **Twitter** (geo-tweets) | Geohash | Geohash prefix stored, LIKE prefix query |
| **PostGIS / PostgreSQL** | R-tree index (GiST) | `ST_DWithin(point, center, radius)` |
| **Redis** | Geohash (sorted set) | `GEORADIUS` command |

---

## Interview Q&A

**Q: How would you find all restaurants within 2km of a user?**  
A: Use geohashing. Convert user's lat/lng to a geohash at precision 6 (~1.2km cells). Query the restaurant DB for all entries whose geohash starts with the user's cell prefix plus the 8 neighboring cell prefixes. The geohash column has a B-tree index so prefix queries are fast. Then apply exact Haversine distance filter in application code to handle the circular radius.

**Q: What is the boundary problem in geohashing and how do you solve it?**  
A: Two points very close to each other may have completely different geohash strings if they're near a cell boundary. Solved by always searching the user's cell plus its 8 neighboring cells (3×3 grid). This guarantees no nearby point in an adjacent cell is missed. Every proximity query searches 9 geohash prefixes.

**Q: What is a quadtree and when would you use it over geohashing?**  
A: A quadtree recursively divides 2D space into 4 quadrants, subdividing further where there are many points. Unlike geohash's fixed-size grid, quadtree cells adapt to data density — tiny cells in NYC, large cells over the ocean. Use quadtrees when you have very uneven point distributions and want adaptive precision. Geohashing is simpler to implement and works well when a fixed grid is acceptable.

**Q: How does Uber use geohashing for driver-rider matching?**  
A: Each active driver's location is stored in a geohash cell (updated every few seconds via WebSocket). When a rider requests a pickup, the system looks up drivers in the rider's geohash cell and 8 neighboring cells. This gives a small candidate set of nearby drivers. Further filtering by actual distance and availability produces the match. The geohash lookup is O(1) in Redis, handling millions of location updates per second.
