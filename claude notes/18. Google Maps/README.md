# System Design: Google Maps (Beginner-Friendly Guide)

---

## What Are We Building?

Google Maps lets you:
- **See a map** of anywhere in the world
- **Get directions** from point A to point B
- **See real-time traffic** and ETA
- **Share your location** as you move

**Scale:**
- 1 billion daily active users
- 99% world coverage
- 25 million real-world location updates per day
- 70+ petabytes of map tile data

> **Why is this hard?**  
> The Earth is a sphere, roads change constantly, traffic fluctuates by the second, and 1 billion phones need to render smooth maps, get accurate routes, and have their GPS positions processed — all simultaneously. Every piece of this needs a separate, carefully designed subsystem.

---

## Step 1: What We're Focusing On

Three core features:
1. **Location Updates** — Users' phones send GPS positions; this data improves traffic and ETAs
2. **Navigation** — Find the best route from A to B, give turn-by-turn directions, recalculate if you go off-route
3. **Map Rendering** — Display smooth, zoomable maps on any device

> **Non-functional priorities:**  
> - **Accuracy** over speed — wrong directions are unacceptable  
> - **Battery & data efficiency** — Google Maps runs on mobile; burning battery/data is a real UX problem  
> - High availability and scalability

---

## Step 2: Map 101 — Concepts You Need to Know First

Before designing the system, understand how maps actually work.

### Latitude & Longitude

<img src="../../system-design-notes/18. Google Maps/images/partitioning-system.png" alt="Partitioning System" width="450">

Every point on Earth is identified by two numbers:
- **Latitude** — how far north or south (-90° to +90°)
- **Longitude** — how far east or west (-180° to +180°)

Example: The Eiffel Tower is at latitude `48.8584°N`, longitude `2.2945°E`.

### From 3D Sphere to 2D Map — Map Projection

<img src="../../system-design-notes/18. Google Maps/images/map-projections.png" alt="Map Projections" width="450">

The Earth is a sphere. Your phone screen is flat. Converting 3D → 2D is called **map projection**. Every projection distorts something (area, shape, or distance).

Google Maps uses the **Web Mercator projection** — it preserves shapes near the equator but distorts sizes at the poles (Greenland looks bigger than Africa even though Africa is 14× larger).

> **Why does it matter?** The projection determines how you calculate which map tiles to show for a given viewport, and how you measure distances.

### Geocoding

**Geocoding** = converting an address into coordinates  
`"1600 Amphitheatre Pkwy, Mountain View, CA"` → `(37.4224°N, 122.0842°W)`

**Reverse geocoding** = converting coordinates into an address  
`(37.4224°N, 122.0842°W)` → `"1600 Amphitheatre Pkwy, Mountain View, CA"`

### Geohashing

<img src="../../system-design-notes/18. Google Maps/images/geohashing.png" alt="Geohashing" width="450">

Geohashing encodes a geographic area into a short alphanumeric string by recursively dividing the world into quadrants. (Same concept as Proximity Service chapter — longer prefix = more precise location.)

Used here to: identify which map tiles to fetch, and which routing tiles to load.

### Map Tiles — The "Mosaic" Approach

Instead of one gigantic image of the world, maps use **tiles** — small 256×256 pixel images stitched together like a mosaic.

```
Zoom level 0: World = 1 tile (256×256px)
Zoom level 1: World = 4 tiles (2×2 grid)
Zoom level 2: World = 16 tiles (4×4 grid)
...
Zoom level 21: Building-level detail = 4^21 tiles
```

<img src="../../system-design-notes/18. Google Maps/images/zoom-level-increases.png" alt="Zoom Level Increases" width="450">

When you zoom in, you download more tiles. When you pan, you download adjacent tiles. You only ever load the tiles you can actually see.

> **Why tiles?** A single image of the entire Earth at street-level detail would be hundreds of petabytes. Tiles let you load only what you need, when you need it.

### Roads as a Graph

<img src="../../system-design-notes/18. Google Maps/images/road-representation.png" alt="Road Representation" width="450">

For navigation, the road network is modeled as a **graph**:
- **Nodes** = intersections
- **Edges** = road segments between intersections (with weights: distance, speed limit, traffic)

Finding the fastest route = finding the shortest weighted path through this graph (Dijkstra, A*).

> **Problem:** The US alone has ~50M intersections and ~120M road segments. Running Dijkstra on the entire graph for every navigation request would take minutes, not seconds.

### Routing Tiles — Hierarchical Graph

<img src="../../system-design-notes/18. Google Maps/images/routing-tiles.png" alt="Routing Tiles" width="450">

<img src="../../system-design-notes/18. Google Maps/images/map-routing-hierarchical.png" alt="Hierarchical Routing" width="450">

Just like map display tiles, the road graph is divided into **routing tiles** — geographic chunks of the road graph. The algorithm loads only the tiles needed for your specific route.

**Three levels of detail:**
- **Local tiles** — every street and alley (used for the first/last mile)
- **Regional tiles** — main roads and highways (used for mid-range routes)
- **Global tiles** — only major highways and motorways (used for cross-country routes)

When navigating from San Francisco to New York:
1. Use local tiles for first/last few miles
2. Use global tiles (interstate highways) for the cross-country portion
3. Stitch them together seamlessly

> **Analogy:** Like Google Maps zoom levels — when you're zoomed out, you see only highways; zoom in and you see every street. The routing algorithm does the same: use the "zoomed out" graph for long stretches, "zoom in" near origin and destination.

---

## Step 3: Scale Estimation

| Metric | Estimate |
|--------|---------|
| DAU | 1 billion |
| Navigation sessions (35 min/week average) | ~5 billion minutes/day |
| GPS update QPS (batched) | ~200,000/sec average, 1M/sec peak |
| Total map tile storage | ~70 PB (with compression) |

> **GPS update QPS = 200K/sec** is the core write throughput challenge. Every navigating phone sends a GPS ping every few seconds. This data must be ingested, processed for traffic, and used to improve ETAs — all in near real-time.

---

## Step 4: High-Level Architecture

<img src="../../system-design-notes/18. Google Maps/images/high-level-design.png" alt="High Level Design" width="500">

The system splits into three independent services:

```
Mobile App
    |               |               |
[Location       [Navigation     [Map Rendering]
 Service]        Service]
    |               |               |
[Cassandra]    [Routing         [CDN]
[Kafka]         Tiles (S3)]     (70PB of tiles)
```

---

## Step 5: Location Service — Collecting GPS Data

<img src="../../system-design-notes/18. Google Maps/images/location-service.png" alt="Location Service" width="450">

Every active phone sends GPS coordinates to the server periodically.

### Why is this data valuable?

1. **Traffic detection** — if 500 phones are moving slowly on I-280, there's traffic
2. **ETA improvement** — historical speed data at this time/day trains the ML model
3. **Road closure detection** — if all phones suddenly detour around a road, it might be closed
4. **ML training** — better predictions over time

### Optimizing: Batch Location Updates

<img src="../../system-design-notes/18. Google Maps/images/location-update-batches.png" alt="Location Update Batches" width="450">

Instead of sending one GPS ping at a time:
```
NAIVE:   [ping] [ping] [ping] [ping]   ← 4 HTTP requests, 4 connections
BATCHED: [ping, ping, ping, ping]      ← 1 HTTP request
```

The phone buffers locations locally and sends a batch every 15-30 seconds. This **dramatically reduces server load** and saves **mobile battery** (each HTTP request requires radio wakeup).

```
POST /v1/locations
Body: {
  locs: [
    { lat: 37.776, lng: -122.416, timestamp: 1746700000 },
    { lat: 37.777, lng: -122.415, timestamp: 1746700015 },
    ...
  ]
}
```

### Location Service Architecture

<img src="../../system-design-notes/18. Google Maps/images/location-service-diagram.png" alt="Location Service Diagram" width="450">

```
Phones → [Location Service] → [Cassandra] (durable storage)
                           → [Kafka]     (real-time stream processing)
```

**Why Cassandra?**
- Write-heavy (200K GPS pings/sec)
- Partitioned by `user_id` → fast lookups per user
- No complex queries needed (just time-range reads per user)

### Location Data Schema

<img src="../../system-design-notes/18. Google Maps/images/user-location-row-example.png" alt="User Location Row" width="450">

```
user_id (partition key) | timestamp (clustering key) | lat     | lng
------------------------|---------------------------|---------|----------
user_A                  | 2026-05-08 14:00:00       | 37.776  | -122.416
user_A                  | 2026-05-08 14:00:15       | 37.777  | -122.415
```

- `user_id` as partition key → all of one user's data on the same node
- `timestamp` as clustering key → data stored sorted by time for range queries

### Kafka for Real-Time Streaming

<img src="../../system-design-notes/18. Google Maps/images/location-update-streaming.png" alt="Location Update Streaming" width="450">

Location updates published to Kafka are consumed by multiple downstream services:
- **Traffic Service** — computes real-time road speeds from aggregated pings
- **ML Training Pipeline** — trains ETA prediction models
- **Map Update Service** — detects road changes, new construction

> **Why Kafka and not just Cassandra for this?** Cassandra stores data for later query. Kafka enables multiple services to process the same location stream in real-time, each at their own pace. One insert to Kafka → multiple consumers react independently.

---

## Step 6: Map Rendering — Serving 70 Petabytes of Tiles

<img src="../../system-design-notes/18. Google Maps/images/static-map-tiles.png" alt="Static Map Tiles" width="450">

### The Key Insight: Tiles Are Static

Map tiles almost never change. A tile showing downtown Paris at zoom level 15 looks the same today as it did last month. This is **perfect for CDN caching**.

```
Tile key: /{zoom}/{x}/{y}.png
Example:  /15/16383/10837.png
```

The client computes the tile keys for its current viewport → fetches from CDN → CDN serves from nearest edge node.

> **Question:** Can't we generate tiles dynamically per request?  
> **Answer:** Generating a tile requires rendering roads, buildings, labels, terrain — computationally expensive. At 1 billion users, this would require absurd server capacity. Pre-generating all tiles offline and storing in CDN costs compute once and serves billions of requests for free.

### CDN vs. No CDN

<img src="../../system-design-notes/18. Google Maps/images/cdn-vs-no-cdn.png" alt="CDN vs No CDN" width="450">

Without CDN: user in Tokyo fetches tiles from servers in Virginia → ~150ms latency → map feels sluggish.

With CDN: user in Tokyo fetches from a CDN edge node in Osaka → ~5ms → map renders instantly.

> At 70 PB of tile data served to 1B users globally, a CDN isn't optional — it's the entire delivery infrastructure.

### Client-Side Tile URL Calculation

<img src="../../system-design-notes/18. Google Maps/images/map-tile-url-calculation.png" alt="Map Tile URL Calculation" width="450">

Two options for determining which tile URLs to fetch:

| Option | How | Pros | Cons |
|--------|-----|------|------|
| Client calculates | Client converts geohash → tile URL math directly | Zero extra API call | Hard to change formula later (requires app updates) |
| Server calculates | Client sends viewport → server returns tile URLs | Easy to change server-side | Extra API roundtrip |

Most production systems use client-side calculation for performance.

### Vector Tiles — Better Than Images

Instead of PNG images, modern maps use **vector tiles**: raw geometric data (roads as lines, buildings as polygons) encoded in a compact binary format (protobuf).

| | Raster Tiles (PNG) | Vector Tiles |
|--|-------------------|--------------|
| Size | ~200 KB/tile | ~20 KB/tile |
| Scaling | Blurry when zoomed | Always crisp |
| Dark mode | Separate tile set | Free (re-render client-side) |
| Language | Fixed in tile | Change without new tiles |

The client's GPU renders vector data → smoother zoom animations, smaller downloads, dynamic styling.

### Precomputed Map Tile Storage

<img src="../../system-design-notes/18. Google Maps/images/precomputed-map-tile-image.png" alt="Precomputed Map Tile" width="450">

The entire world's tiles are pre-generated by an offline pipeline and stored in object storage (S3). The CDN sits in front of S3 — it pulls tiles from S3 on cache miss, then serves subsequent requests from cache.

---

## Step 7: Navigation Service — Getting from A to B

<img src="../../system-design-notes/18. Google Maps/images/navigation-service.png" alt="Navigation Service" width="450">

```
GET /v1/nav?origin=Market+St,SF&destination=Disneyland
```

The navigation service is broken into specialized sub-services:

### 1. Geocoding Service

Converts addresses into coordinates:
```
"Disneyland, Anaheim, CA" → (33.8121°N, 117.9190°W)
```

Uses a Redis-backed geocoding database: `{address string → (lat, lng)}`. Redis is chosen because geocoding is **read-heavy** (millions of address lookups/sec) and **infrequently updated** (addresses rarely change). Sub-millisecond lookups.

### 2. Shortest Path Service (A* on Routing Tiles)

<img src="../../system-design-notes/18. Google Maps/images/shortest-path-service.png" alt="Shortest Path Service" width="450">

This is the core routing algorithm:

```
Input: origin (lat, lng), destination (lat, lng)
Output: ordered list of road segments forming the optimal path

Steps:
① Convert origin + destination to geohashes
② Look up the routing tiles containing those geohashes (from S3)
③ Run A* algorithm:
   - Start at origin node
   - Explore neighbors, prioritizing nodes closer to destination
   - Load adjacent routing tiles as needed
   - Stop when destination is reached
④ Return the path as a sequence of waypoints
```

> **Why not Dijkstra?** Dijkstra explores nodes in all directions uniformly. A* uses the straight-line distance to destination as a heuristic — it focuses the search toward the destination, making it ~10x faster for geographic routing.

> **Why routing tiles and not one big graph?** The US road graph has 120M edges. Loading all of it for a 2-mile trip in Brooklyn would be absurd. Routing tiles load only the portion of the graph relevant to your route.

### 3. ETA Service

Uses ML models trained on historical GPS probe data:
```
ETA = Σ (segment_length / predicted_speed_at_this_time)
```

Factors in:
- Historical speeds (same hour, same day of week)
- Current real-time speeds from GPS probes
- Weather conditions
- Known events (stadium game ending, parade)

Accuracy improves continuously as more GPS data trains better models.

### 4. Ranker Service

After the shortest path is found, the ranker applies user filters:
- Avoid toll roads
- Avoid highways
- Prefer scenic routes
- Prefer fastest vs. shortest

Returns the top-ranked routes.

---

## Step 8: Adaptive ETA and Rerouting

What happens when traffic suddenly gets worse on your route mid-drive?

### How the System Detects You're Affected

<img src="../../system-design-notes/18. Google Maps/images/adaptive-eta-data-storage.png" alt="Adaptive ETA Data Storage" width="450">

For every active navigation session, store which routing tiles the user will pass through:

```
user_1: [tile_r1, tile_r2, tile_r3, ... tile_rk]
user_2: [tile_r4, tile_r6, tile_r9, ... tile_rn]
```

When a traffic incident occurs on **tile_r2**:
- Query: "Which users have tile_r2 in their remaining route?"
- All affected users receive a reroute calculation
- Only users who haven't passed tile_r2 yet are affected

**Storage optimization:** Instead of storing every tile, store the origin tile plus tiles at progressively coarser resolutions until destination is covered. Checking if the incident tile is within this sparse set is fast.

### Rerouting Flow

```
Traffic incident detected on tile_r2 (Kafka event)
           ↓
Find all users whose route passes through tile_r2
           ↓
For each affected user:
  → Run new shortest path from user's current position
  → Compare new route ETA vs. current route ETA
  → If new route is >2 minutes faster → push reroute to user's phone
           ↓
User's phone receives new route via WebSocket/SSE
```

### Delivery Protocol: WebSocket for Real-Time Rerouting

> **Why not HTTP?** Rerouting needs the server to **push** to the client — "there's a faster route!" HTTP requires the client to ask first. WebSocket allows the server to push at any time.

> **Why not SMS/push notifications?** SMS has limited payload. Push notifications have delays and don't work in web apps. WebSocket (or SSE) is the right real-time protocol for navigation.

---

## Step 9: Data Storage Summary

| Data Type | Storage | Why |
|-----------|---------|-----|
| Map tiles (images/vectors) | S3 + CDN | Static, large, globally served |
| Routing tiles (road graph) | S3 (binary files) | Read-heavy, no DB features needed, aggressively cached |
| User GPS history | Cassandra | Write-heavy (200K/sec), time-range queries per user |
| GPS stream (real-time) | Kafka | Fan-out to multiple consumers (traffic, ML, analytics) |
| Geocoding (address ↔ lat/lng) | Redis | Read-heavy, infrequent updates, sub-millisecond needed |
| Active navigation sessions | Redis/in-memory | Fast lookup of "which tile is user on?" |

---

## Step 10: Final Architecture

<img src="../../system-design-notes/18. Google Maps/images/final-design.png" alt="Final Design" width="500">

```
Mobile App
    │ (WebSocket / HTTP)
    ↓
[Load Balancer]
    │                   │                    │
[Location          [Navigation           [Map Tile
 Service]           Service]              API]
    │                   │                    │
[Kafka] → [Traffic  [Geocoding   [Routing  [CDN]
           Service]  Service     Tiles       ↑
[Cassandra          (Redis)]    (S3)]     [S3]
 GPS history]
    │
[ML Pipeline]
(ETA models)
    │
[Shortest Path
 Service (A*)]
    │
[ETA Service]
    │
[Ranker Service]
    ↓
[Navigation Response]
 + WebSocket pushes
 (rerouting)
```

---

## Summary: The "Why" Behind Each Decision

| Decision | Why? |
|----------|------|
| Batch GPS updates from client | Reduce server requests; save mobile battery (radio wakeup cost) |
| Cassandra for GPS history | Write-heavy 200K/sec; partition by user_id for range queries |
| Kafka for GPS stream | One stream, multiple consumers (traffic, ML, analytics) independently |
| Pre-generated tiles stored in S3 + CDN | Tiles are static; generate once, serve billions of times from edge nodes |
| Vector tiles over raster (PNG) | 10× smaller, client-rendered, supports dark mode, smooth zoom |
| Routing tiles (not one global graph) | Can't load 120M edges for a 2-mile trip; load only relevant subgraph |
| Hierarchical routing tiles (3 levels) | Cross-country route uses highway-only graph; local uses full detail |
| A* over Dijkstra | A* focuses search toward destination using heuristic; ~10x faster |
| Redis for geocoding DB | Read-heavy (millions/sec), rarely updated, needs sub-ms latency |
| WebSocket for rerouting push | Server needs to push new route to driver unprompted; HTTP can't do that |
| ETA via ML on GPS probe data | Real traffic speeds from actual phones are far more accurate than speed limits |

---

## Quick Interview Q&A Cheat Sheet

**Q: How are map tiles served?**  
A: Pre-rendered and stored in S3 object storage, keyed by `{zoom}/{x}/{y}`. A CDN caches tiles at edge nodes near users. Client computes tile keys from its viewport geohash and fetches directly from CDN. Tiles are static — no dynamic rendering on the critical path.

**Q: How does turn-by-turn routing work?**  
A: Road network = graph (intersections = nodes, roads = edges with weights). A* algorithm finds shortest weighted path on routing tiles loaded from S3. Route = sequence of road segments → decomposed into turn instructions ("In 500m, turn left").

**Q: How does Google Maps know there's traffic?**  
A: Anonymized GPS pings from all active phones are ingested into Kafka → stream processor aggregates speeds per road segment → segments get updated travel times → routing engine uses updated edge weights for ETA and rerouting.

**Q: How does rerouting work in real time?**  
A: Active navigation sessions store which routing tiles the user will pass through. When an incident occurs on a tile, all affected users are found and a new A* route is computed from their current position. If >2 minutes faster, the new route is pushed to the phone via WebSocket.

**Q: Why not use one big SQL database for the road graph?**  
A: The US road graph is 120M edges. SQL doesn't support efficient geographic graph traversal. Routing tiles in S3 (binary adjacency lists) are loaded on-demand and cached in memory — perfect for the random-access, geography-bounded access pattern of routing.

**Q: How does ETA become more accurate over time?**  
A: GPS probe data (real speeds from real phones) trains ML models: "on this segment at 5pm on a Friday in December, expect 15 mph." Models improve with more data. Real-time speeds from current probe data update predictions for the current moment, on top of historical baselines.

**Q: What protocol is used for real-time navigation updates?**  
A: WebSocket. Server needs to proactively push rerouting suggestions and ETA updates to the driver. HTTP requires client to poll. WebSocket maintains a persistent connection so server can push at any time with minimal overhead.
