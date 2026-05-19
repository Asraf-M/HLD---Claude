# System Design: Nearby Friends (Beginner-Friendly Guide)

---

## What is Nearby Friends?

It's the feature in apps like Facebook, Snapchat, or Zenly that shows you **which of your friends are physically close to you right now** — with a distance and a timestamp.

**Real-world examples:**
- Facebook Nearby Friends
- Snapchat's Snap Map
- Find My Friends (Apple)
- Zenly (RIP 2023)

> **How is this different from the Proximity Service (Yelp/Google Maps)?**  
> The Proximity Service finds **static locations** — restaurants and businesses don't move. Nearby Friends handles **dynamic locations** — users move constantly and their positions update every 30 seconds. This single difference changes the entire architecture: you can't cache aggressively because positions go stale in seconds, and you need a real-time streaming pipeline to ingest hundreds of thousands of updates per second.

---

## Step 1: Understand What We're Building

### Requirements

| Feature | Detail |
|---------|--------|
| See nearby friends | Show friends within 5 miles with distance + last-seen timestamp |
| Real-time updates | Nearby friends list refreshes every few seconds |
| Inactive detection | Friends inactive for 10+ minutes disappear from the map |
| Location history | Store history for analytics/ML (not shown to users) |

### Non-Functional Requirements

| Requirement | Detail |
|-------------|--------|
| Low latency | Location updates must propagate to friends fast |
| High availability | Occasional update loss is OK; system must stay up |
| Eventual consistency | Few-second delay in replicas is acceptable |

### Scale

- **1 billion total users**, 10% use Nearby Friends = **100 million DAU**
- **10% are concurrent** at any time = **10 million simultaneous users**
- Each user has **400 friends** using the feature
- Location updated every **30 seconds**
- **Location Update QPS = 10M ÷ 30 = ~334,000 updates/second**

> **334,000 location updates per second** is the core challenge. Each update must be fanned out to all online friends who are within 5 miles. At 400 friends with 10% online, that's ~40 fan-outs per update = **~14 million pushes per second** through the system.

---

## Step 2: The Core Design Challenge

### How do we get User A's location to all of User A's online friends?

<img src="../../system-design-notes/17. Nearby Friends/images/fan-out-backend.png" alt="Fan-out Backend" width="450">

The naive idea: User A sends their location → Backend → delivers to Friends B, C, D.

The backend needs to:
1. Receive location updates from ALL active users (334K/sec)
2. For each update, find which friends are active and within range
3. Push the update to those friends

> **Question:** Why not use a peer-to-peer approach (users talk directly to each other)?  
> **Answer:** Mobile apps have flaky connections, NAT firewalls, and tight battery/power constraints. A centralized backend is far more reliable — it handles reconnections, buffering, and routing transparently.

### Why WebSocket?

> **Question:** Why not HTTP polling? Can't each user just ask "any updates from my friends?" every 5 seconds?  
> **Answer:** At 10M active users polling every 5 seconds = **2 billion HTTP requests per minute** — most of them empty responses wasting server resources. WebSocket maintains a **persistent, bidirectional connection** — the server *pushes* updates to the client only when something actually changed. Zero wasted requests.

---

## Step 3: High-Level Design

<img src="../../system-design-notes/17. Nearby Friends/images/simple-high-level-design.png" alt="Simple High Level Design" width="500">

```
Mobile Users
      |          |
   [WebSocket]  [HTTP]
      |          |
  [Load Balancer]
      |                    |
[WebSocket Servers]   [API Servers]
- Real-time location  - User management
  send/receive        - Friendship CRUD
      |               - Auth, profile
   ___↓____________________
  |         |         |    |
[Redis   [Location  [Location  [User
 Pub/Sub]  Cache]   History    DB]
           (Redis)   DB]
                  (Cassandra)
```

### What Each Component Does

**WebSocket Servers:**
> Maintain persistent connections with every active mobile user. When User A's phone sends a new location, it arrives here. When a friend's location update needs to be pushed to User A's phone, it goes through here.

**API Servers (HTTP):**
> Handle all non-real-time operations — login, add/remove friends, profile updates. These are stateless and easy to scale.

**Redis Location Cache:**
> Stores the **current location** of every active user: `{user_id → (lat, lng, timestamp)}`. Each entry has a **TTL (Time-To-Live)** of ~10 minutes. When a user goes inactive and stops sending updates, their entry auto-expires and they disappear from everyone's map. No cleanup job needed.

**Location History Database (Cassandra):**
> Stores every location event permanently for analytics and ML. Cassandra is ideal because it's optimized for **write-heavy** workloads (millions of inserts per second) and time-range queries (`give me user A's path from 2pm-4pm`).

**User Database:**
> Stores user profiles and the friendship graph (who is friends with whom).

**Redis Pub/Sub:**
> The message bus that delivers location updates from one user to their friends. More on this below.

---

## Step 4: Redis Pub/Sub — The Heart of the System

<img src="../../system-design-notes/17. Nearby Friends/images/redis-pubsub-usage.png" alt="Redis Pub/Sub Usage" width="500">

### What is Pub/Sub?

**Pub/Sub = Publish/Subscribe.** It's a messaging pattern where:
- A **publisher** sends a message to a **channel** (without knowing who's listening)
- All **subscribers** of that channel receive the message instantly

> **Analogy:** Pub/Sub is like a radio station. The DJ (publisher) broadcasts to a frequency (channel). Anyone who has their radio tuned to that frequency (subscriber) receives it. The DJ doesn't need to know who's listening or how many people there are.

### How It Works for Nearby Friends

**Every user has their own Pub/Sub channel:** `user:A:location`

When User A moves:
1. WebSocket Server publishes to `user:A:location`: "A is at (37.77, -122.41)"
2. All of A's friends who are subscribed to `user:A:location` receive this message
3. Their WebSocket servers push the update to their phones

When User B comes online:
- B's WebSocket server **subscribes** to channels for each of B's friends: `user:A:location`, `user:C:location`, etc.
- Now whenever any friend moves, B's server gets notified automatically

When User B goes offline:
- B's WebSocket server **unsubscribes** from all friend channels

> **Question:** Why not one shared channel for everyone?  
> **Answer:** Privacy. If all 10M users shared one channel, you'd receive location updates for strangers. With per-user channels, you only receive updates from people you subscribe to (your friends).

---

## Step 5: Periodic Location Update Flow — Step by Step

<img src="../../system-design-notes/17. Nearby Friends/images/periodic-location-update.png" alt="Periodic Location Update" width="500">

Here's exactly what happens every 30 seconds when your phone sends a location update:

```
① Phone sends location (37.77, -122.41) via WebSocket
② Load Balancer → your persistent WebSocket Server
③ WebSocket Server saves to Location History DB (Cassandra)
   → permanent record
④ WebSocket Server updates Location Cache (Redis)
   → overwrites old position, refreshes TTL
⑤ WebSocket Server publishes to Redis Pub/Sub
   → "user:A:location" channel
⑥ Redis Pub/Sub broadcasts to all WebSocket Servers
   that have friends subscribed to "user:A:location"
⑦ Each subscribed WebSocket Server checks:
   - Is this friend within 5 miles of User A?
   - Yes → push update to friend's phone via WebSocket
   - No  → drop the update (too far away)
```

<img src="../../system-design-notes/17. Nearby Friends/images/detailed-periodic-location-update.png" alt="Detailed Periodic Location Update" width="500">

> **Question:** Why check distance at step ⑦ instead of earlier?  
> **Answer:** We don't check at publish time (step ⑤) because the publishing server doesn't know where each friend is. Each subscribing WebSocket server knows its own users' positions (cached in memory) — so it checks the distance after receiving the update and decides whether to push it. This distributes the computation across many servers.

### WebSocket API Routines

| Routine | Direction | Purpose |
|---------|-----------|---------|
| Periodic location update | Client → Server | Phone sends new GPS coordinates |
| Friend location update | Server → Client | Server pushes a friend's new position |
| Initialization | Client → Server | On app open, client sends current location; server sends back all nearby friends |
| Subscribe to new friend | Server → Client | Tells client to start tracking a friend who just came online |
| Unsubscribe from friend | Server → Client | Tells client to stop tracking a friend who went offline |

---

## Step 6: What Happens When You First Open the App?

When User A opens the Nearby Friends feature, their WebSocket server must:

```
① Fetch User A's full friend list from User DB
② Subscribe to each friend's Pub/Sub channel (user:B:location, user:C:location, ...)
③ Fetch each friend's current location from Location Cache (Redis)
④ Filter: keep only friends within 5 miles
⑤ Send the initial nearby friends list to User A's phone
⑥ Now any future updates arrive via Pub/Sub automatically
```

After initialization, updates are purely push-based — no more polling.

---

## Step 7: Data Model

### Location Cache (Redis)

```
Key:   user:{user_id}
Value: { lat: 37.77, lng: -122.41, updated_at: 1746700000 }
TTL:   600 seconds (10 minutes)
```

> If a user stops sending updates, their Redis entry auto-expires after 10 minutes → they disappear from maps automatically. No cleanup job needed. **This is a beautiful use of Redis TTL.**

### Location History Table (Cassandra)

```
user_id    | timestamp           | latitude | longitude
-----------|---------------------|----------|----------
user_A     | 2026-05-08 14:00:00 | 37.776   | -122.416
user_A     | 2026-05-08 14:00:30 | 37.777   | -122.415
user_A     | 2026-05-08 14:01:00 | 37.778   | -122.414
```

**Why Cassandra?**
- Write-heavy (334K inserts/sec)
- Partitioned by `user_id` → fast range queries per user
- Horizontally scalable without complex sharding logic

### User/Friendship Database

```
users:       user_id | name | profile_pic | location_sharing_enabled
friendships: user_id | friend_id | created_at
```

---

## Step 8: Scaling — The Hard Part

### Scaling WebSocket Servers

WebSocket connections are **stateful** — User A is tied to a specific server. When scaling:

- New servers → **graceful shutdown** of old servers ("draining mode")
  - Mark old server as "draining" in load balancer → no new connections go to it
  - Existing connections stay until clients naturally reconnect to new servers
- Never abruptly kill a WebSocket server — connections would drop for all users on it

### Scaling Redis Pub/Sub — The Bottleneck

The biggest challenge: **14 million Pub/Sub pushes per second**.

A single Redis server handles ~100K pushes/sec → you need **~140 Redis servers**.

But 200 channels (one per user) × 10 bytes each ≈ only **~200 GB memory** → only 2 servers for memory.

**The bottleneck is CPU, not memory.** You need 140 servers for compute, not storage.

### Distributing Channels Across Redis Servers

With 140 servers, how does a WebSocket server know which Redis server holds `user:A:location`?

**Answer: Consistent Hashing**

<img src="../../system-design-notes/17. Nearby Friends/images/channel-distribution-data.png" alt="Channel Distribution" width="450">

<img src="../../system-design-notes/17. Nearby Friends/images/consistent-hashing.png" alt="Consistent Hashing" width="450">

Each channel's name is hashed → the hash value determines which Redis server is responsible. WebSocket servers use a **Zookeeper/etcd** service registry to look up this routing information (cached in-memory on each WebSocket server for efficiency).

> **Why consistent hashing?** When you add or remove Redis servers, consistent hashing minimizes the number of channels that need to be remapped. Normal hashing would reshuffle nearly all channels — consistent hashing reshuffles only a small fraction.

**When scaling up/down:**
- Do it at **lowest-traffic hours** (e.g., 3am)
- Expect a brief period of re-subscription requests from WebSocket servers
- Expect some missed location updates during remapping (acceptable)

---

## Step 9: The "Nearby Random Person" Feature

What if you want to show not just friends, but any random nearby person who opted in?

<img src="../../system-design-notes/17. Nearby Friends/images/geohash-pubsub.png" alt="Geohash Pub/Sub" width="500">

Instead of per-user channels, create **per-geohash channels**:

<img src="../../system-design-notes/17. Nearby Friends/images/location-updates-geohash.png" alt="Location Updates by Geohash" width="500">

```
Channel: geohash:9q8znd  (covers a ~1km cell in San Francisco)

Everyone physically in geohash 9q8znd:
  → subscribes to channel "geohash:9q8znd"
  → publishes their location to "geohash:9q8znd"
  → receives location updates from anyone else in the cell
```

**Handling cell boundaries:**

<img src="../../system-design-notes/17. Nearby Friends/images/geohash-borders.png" alt="Geohash Borders" width="500">

Subscribe to your geohash **plus neighboring geohashes** (same solution as Proximity Service). Someone 50m away in the adjacent cell still shows up because you're subscribed to their geohash channel too.

> **Privacy note:** Only users who explicitly opt in to "Nearby Random People" appear in these channels. Never mix this with the regular friends feature. Show approximate location (rounded to 500m), not exact GPS.

---

## Step 10: Full Architecture Summary

```
Mobile User (Phone GPS updates every 30s)
        |                   |
    [WebSocket]           [HTTP]
        |                   |
    [Load Balancer]
        |                        |
[WebSocket Servers]         [API Servers]
  ① Receive location          - Login/Auth
  ② Save to history DB        - Add/remove friends
  ③ Update location cache     - Profile management
  ④ Publish to Pub/Sub
  ⑥ Receive friend updates
  ⑦ Check distance
  ⑦ Push to friend phones
        |
   _____|_________________________________
  |               |           |           |
[Redis         [Redis      [Cassandra] [MySQL/
 Pub/Sub        Cache]     Location    Postgres
 ~140 servers]  (current   History     User DB +
                location,  (permanent  Friendship]
                TTL 10min)  record)
```

---

## Summary: The "Why" Behind Each Decision

| Decision | Why? |
|----------|------|
| WebSocket (not HTTP polling) | 10M users × polling = billions of wasted requests; WebSocket pushes only when something changes |
| Per-user Pub/Sub channel | Privacy — only friends subscribe; one shared channel would expose everyone's location to everyone |
| Redis for current location | Sub-millisecond reads; TTL auto-expires inactive users |
| Redis TTL for inactivity | Inactive users vanish automatically without cleanup jobs |
| Cassandra for location history | Write-heavy (334K/sec inserts); time-range queries per user |
| Consistent hashing for Redis cluster | Adding/removing servers reshuffles minimum channels; minimizes disruption |
| Filter by distance at WebSocket server | Publishing server doesn't know friend positions; subscribing server does (in memory) |
| Geohash channels for random nearby people | Natural geographic grouping; easy to handle boundaries by subscribing to 9 cells |
| Graceful WebSocket server shutdown (draining) | Abrupt shutdown would drop all connections on that server simultaneously |

---

## Quick Interview Q&A Cheat Sheet

**Q: How is Nearby Friends different from Proximity Service?**  
A: Proximity Service finds static things (businesses don't move). Nearby Friends tracks dynamic locations — users update every 30 seconds. No aggressive caching possible; need real-time streaming pipeline for 334K updates/sec.

**Q: Why WebSocket over HTTP for this feature?**  
A: WebSocket is bidirectional and persistent. Server can push friend location updates the moment they arrive, without the client polling. 10M users polling every 5 seconds = 2 billion HTTP requests/minute — mostly empty responses. WebSocket cuts this to zero.

**Q: How does a location update reach a user's friends?**  
A: User A's WebSocket server publishes to `user:A:location` Redis Pub/Sub channel. All WebSocket servers with A's friends connected are subscribed to that channel. They receive the update, check if the friend is within 5 miles, and push to the friend's phone via WebSocket.

**Q: How does the system detect that a user has gone inactive?**  
A: Every location update refreshes the Redis TTL (10 minutes). When a user stops sending updates (phone put down, app closed), their Redis entry expires after 10 minutes → they automatically disappear from all friends' maps. No active detection needed.

**Q: How do you scale Redis Pub/Sub to 14M pushes/second?**  
A: Use a cluster of ~140 Redis servers. Distribute channels across servers using consistent hashing (hash the channel name → assign to a server). WebSocket servers cache the hash ring in memory for O(1) routing. Use Zookeeper/etcd to track which servers are alive.

**Q: What happens on app startup (initialization)?**  
A: WebSocket server fetches user's friend list → subscribes to each friend's Pub/Sub channel → fetches all friends' current locations from Redis → filters to those within 5 miles → sends initial list to phone. From then on, all updates are push-based with zero polling.

**Q: Why Cassandra for location history?**  
A: Cassandra is optimized for write-heavy workloads (334K writes/sec is trivial for it). Partition by `user_id`, cluster by `timestamp` → fast range scans for "show me user A's path from 2pm-4pm". SQL databases would need heavy sharding for this write volume.
