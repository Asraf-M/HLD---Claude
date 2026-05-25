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

**Pub/Sub = Publish/Subscribe.** It is a messaging pattern with two roles:
- A **publisher** sends a message to a **channel** (a named topic). It does not know or care who is listening.
- A **subscriber** listens on a channel and receives every message the moment it is published.

> **Analogy:** Think of a WhatsApp group. When Alice posts a message to the group, every member instantly sees it — Alice does not send it to each person individually. Redis Pub/Sub works exactly like that. The "group" is the channel, Alice is the publisher, and the group members are the subscribers.

---

### What Does a Redis Pub/Sub Channel Look Like?

A channel is just a **named string**. In this system, every user gets their own personal channel:

```
Channel name format:  user:{user_id}:location

Alice's channel:   user:alice:location
Bob's channel:     user:bob:location
Charlie's channel: user:charlie:location
```

Whenever Alice moves, her location update is **published** to `user:alice:location`.  
Whoever is subscribed to that channel — Alice's friends' servers — instantly receive it.

---

### The Three Redis Pub/Sub Commands

```
SUBSCRIBE   user:alice:location       ← "I want to receive Alice's updates"
PUBLISH     user:alice:location  <msg> ← "Alice moved, tell everyone"
UNSUBSCRIBE user:alice:location       ← "Stop sending me Alice's updates"
```

That is the entire API. Simple.

---

### Full Worked Example: Alice Moves, Bob and Charlie Are Her Friends

**Setup:**
- Alice, Bob, and Charlie are all friends
- All three have the app open
- Bob is on WebSocket Server 1 (WS1)
- Charlie is on WebSocket Server 2 (WS2)
- Alice is on WebSocket Server 3 (WS3)

**When Bob opened the app (initialization — covered in Step 6):**
```
WS1 (Bob's server) subscribes to Alice's and Charlie's channels:
  SUBSCRIBE user:alice:location
  SUBSCRIBE user:charlie:location

Redis notes: "WS1 wants Alice's and Charlie's updates"
```

**When Charlie opened the app:**
```
WS2 (Charlie's server) subscribes to Alice's and Bob's channels:
  SUBSCRIBE user:alice:location
  SUBSCRIBE user:bob:location

Redis notes: "WS2 also wants Alice's updates"
```

**Now Alice moves to a new location:**

```
Alice's phone  →  GPS: (lat=37.778, lon=-122.415)
     ↓
WebSocket Server 3 (WS3) receives it via WebSocket connection
     ↓
WS3 runs three things in parallel:
  1. Save to Cassandra (location history)
  2. Update Redis cache: SET user:alice:location {lat, lon, time}
  3. PUBLISH user:alice:location "{lat:37.778, lon:-122.415}"
     ↓
Redis broadcasts to ALL subscribers of user:alice:location:
  → WS1 receives it  (Bob's server)
  → WS2 receives it  (Charlie's server)
     ↓
WS1 checks: is Alice within 5 miles of Bob?
  Alice: (37.778, -122.415)
  Bob:   (37.780, -122.410)  ← Bob's position is in WS1's memory
  Distance: 0.3 miles → YES, send update
  → WS1 pushes "Alice is 0.3 miles away" to Bob's phone ✅

WS2 checks: is Alice within 5 miles of Charlie?
  Alice:   (37.778, -122.415)
  Charlie: (37.500, -122.100)  ← Charlie is far away
  Distance: 22 miles → NO, drop the update ❌
  Charlie's phone sees nothing (Alice is not "nearby")
```

The whole flow from Alice's GPS ping to Bob's phone update takes **under 100ms**.

---

### Visual: The Full Pub/Sub Flow

```
Alice's phone
    |  GPS update every 30s
    ↓
[WS Server 3]  ──PUBLISH──►  Redis Pub/Sub
                              Channel: user:alice:location
                              Message: {lat:37.778, lon:-122.415}
                                   |
                    ┌──────────────┴──────────────┐
                    ↓                             ↓
             [WS Server 1]                 [WS Server 2]
              (Bob's server)              (Charlie's server)
              subscribed ✅                subscribed ✅
                    |                             |
              Distance check               Distance check
              0.3 miles ✅                 22 miles ❌
                    |                             |
              Push to Bob's phone          Drop update
              "Alice: 0.3mi away"
```

---

### What Happens When a New Friend Comes Online?

Say Dave is Alice's friend. Dave was offline and just opened the app.

```
Dave opens app → connects to WS4 (Dave's WebSocket server)

WS4 does on startup:
  1. Fetch Dave's friend list from User DB → [Alice, Bob, Charlie]
  2. SUBSCRIBE user:alice:location    ← start watching Alice
     SUBSCRIBE user:bob:location      ← start watching Bob
     SUBSCRIBE user:charlie:location  ← start watching Charlie
  3. Fetch current positions from Redis cache for all 3 friends
  4. Filter: who is within 5 miles of Dave right now?
  5. Send Dave the initial list → "Alice is 1.2mi away, Bob is 3mi away"

From now on: WS4 receives all future updates via Pub/Sub automatically
```

---

### What Happens When a Friend Goes Offline?

Say Bob closes the app or loses connection.

```
Bob closes app → WebSocket connection to WS1 breaks

WS1 detects disconnection:
  UNSUBSCRIBE user:alice:location   ← stop receiving Alice's updates
  UNSUBSCRIBE user:charlie:location ← stop receiving Charlie's updates

Redis removes WS1 from the subscriber list for both channels.

Result: Alice's and Charlie's updates are no longer sent to WS1.
Nobody wastes bandwidth delivering updates nobody will use.
```

Meanwhile, Bob's Redis cache entry (`user:bob:location`) expires after 10 minutes TTL → Bob automatically disappears from everyone's map.

---

### Why One Channel Per User (Not One Global Channel)?

A natural question: why not have a single `all:locations` channel that everyone publishes to?

```
BAD — single global channel:
  Every user publishes to: all:locations
  Every server subscribes to: all:locations

  If 10M users each update every 30s:
  = 333,000 messages/sec through ONE channel
  = Every server receives ALL 333,000 msg/sec
  = 99.99% of messages are for users you don't care about
  = Massive wasted processing and bandwidth
```

```
GOOD — per-user channel:
  Alice publishes to: user:alice:location
  Only servers with Alice's friends subscribed receive it
  
  Alice has 200 friends, 50 are online
  = 50 servers receive Alice's update
  = 0 irrelevant servers receive it
  = No wasted work
```

Per-user channels give you **targeted fan-out** — you only get updates you actually need.

---

### One More Thing: What If Two Friends Are on the Same WebSocket Server?

If Bob and Dave are both connected to WS1, and both are friends with Alice:

```
WS1 subscribes to user:alice:location   (once — not twice)

Redis sends Alice's update once to WS1.

WS1:
  Check Bob's distance from Alice → within 5 miles → push to Bob ✅
  Check Dave's distance from Alice → 18 miles → drop ❌
```

Redis sends **one message** to WS1 regardless of how many of Alice's friends are on that server. WS1 handles the fan-out to its own users locally. This is efficient — you never get duplicate messages from Redis.

---

### Summary: Why Redis Pub/Sub Fits This Problem Perfectly

| Requirement | How Pub/Sub Solves It |
|---|---|
| Real-time delivery | Message delivered in milliseconds, no polling |
| Only relevant updates | Per-user channels → servers only receive their users' friends' updates |
| User goes offline | UNSUBSCRIBE cleans up immediately |
| User comes online | SUBSCRIBE and fetch current positions in one startup sequence |
| Scales with users | Each channel is independent — adding users adds channels, not load to existing ones |
| Simple to reason about | Publish one message → all subscribers get it — no complex routing logic |

> **Question:** Why not use Kafka instead of Redis Pub/Sub here?  
> **Answer:** Kafka is durable — it stores messages on disk and lets consumers replay them. For location updates, you do not need durability. If you missed a location update from 2 seconds ago, the next one arrives in 30 seconds anyway. Redis Pub/Sub is fire-and-forget (no storage, no replay) — which is exactly what you want here. Lower latency, simpler setup, no disk I/O.

---

### Who Plays Which Role — Quick Clarification

A common confusion is who exactly is the publisher, the channel, and the subscriber. Here it is clearly:

```
Alice's phone
    │  sends GPS update via WebSocket
    ▼
[WS Server 3]  ← PUBLISHER  (server publishes to Redis)
    │
    │  PUBLISH user:alice:location "{lat, lon}"
    ▼
[Redis Pub/Sub]  ← CHANNEL lives here  (just a message bus, stores nothing)
    │
    ├──────────────────────┐
    ▼                      ▼
[WS Server 1]         [WS Server 2]
  SUBSCRIBER            SUBSCRIBER
(Bob's server)        (Charlie's server)
    │                      │
    ▼                      ▼
Bob's phone          Charlie's phone
```

| Role | Who plays it |
|---|---|
| **Publisher** | A WebSocket Server — the one Alice is connected to |
| **Channel** | Lives inside Redis (`user:alice:location`) |
| **Subscriber** | Other WebSocket Servers — the ones Bob's and Charlie's phones connect to |

The phones **never talk to Redis directly**. Redis is purely a **server-to-server message bus** — servers publish to it, servers subscribe to it. The phones only ever talk to their own WebSocket server.

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

---

### Channel vs Redis Server — What is the Difference?

These are two completely different things:

| | Redis Server | Channel |
|---|---|---|
| **What it is** | Physical machine/process | A logical pub/sub topic (just a string name) |
| **Where it lives** | Data center rack | *Inside* a Redis server |
| **Example** | R1, R2, ... R140 | `"location:alice"`, `"location:bob"` |
| **Scale** | 140 servers | Millions of channels |

One Redis server hosts thousands of channels. Think of it like:

```
Redis Server 1 (R1)              Redis Server 2 (R2)
┌──────────────────────┐         ┌──────────────────────┐
│  channel:alice       │         │  channel:bob         │
│  channel:charlie     │         │  channel:david       │
│  channel:eve         │         │  channel:frank       │
└──────────────────────┘         └──────────────────────┘
```

**Consistent hashing decides: which Redis SERVER hosts a given channel.**

```
hash("location:alice") = 342  →  lands on R1  →  channel lives on R1
hash("location:bob")   = 891  →  lands on R2  →  channel lives on R2
```

So when WS Server 1 wants to publish Alice's location:
1. Hash `"location:alice"` → get `342` → go to **R1**
2. On R1, publish to **channel:alice**

When WS Server 2 wants to subscribe for Bob (who is friends with Alice):
1. Hash `"location:alice"` → get `342` → go to **R1** (same result, always)
2. On R1, subscribe to **channel:alice** → receives Alice's updates

**Alice and Bob are on different WS servers — but they reach the same Redis channel on R1 because the hash is deterministic.**

---

### Who Sends What Channel ID to Redis?

Both WS servers use **Alice's channel ID** as input — not Bob's. The channel belongs to the person being tracked (Alice), not the person watching (Bob).

```
Alice moves
  → Phone → WS Server 1 → PUBLISH  "location:alice" + coordinates → Redis R1
                                            ↑
                                     same channel name
                                            ↓
Bob (friend of Alice)
  → Phone ← WS Server 2 ← SUBSCRIBE "location:alice"              → Redis R1
```

- **WS Server 1** (Alice's server) sends: `PUBLISH location:alice <new_coordinates>`
- **WS Server 2** (Bob's server) sends: `SUBSCRIBE location:alice`

Bob's WS server subscribes to Alice's channel — not its own. Because Bob wants to *receive* Alice's location, not broadcast his own.

**The rule:**
- Your own WS server **publishes** to → **your channel**
- Your friends' WS servers **subscribe** to → **your channel**

So if Alice has 500 friends scattered across 50 different WS servers, all 50 of those servers subscribe to `location:alice` on Redis R1. When Alice's WS server publishes **once**, Redis fans it out to all 50 servers simultaneously — and each server pushes the update to their connected friends.

---

### Distributing Channels Across Redis Servers

With 140 servers, how does a WebSocket server know which Redis server holds `user:A:location`?

---

#### First, Understand the Problem

Every channel (`user:alice:location`) lives on **exactly one** Redis server. When a WebSocket server wants to PUBLISH or SUBSCRIBE to a channel, it must connect to **the correct Redis server** — the one that owns that channel.

If the publisher sends to Redis-7 but the subscriber is connected to Redis-3, they are on different servers and will **never see each other's messages**.

```
WRONG — publisher and subscriber on different Redis servers:

WS3 → PUBLISH user:alice:location → Redis-7
WS1 → SUBSCRIBE user:alice:location → Redis-3   ← different server!

Result: WS1 never receives Alice's update. Bob never gets notified. ❌
```

```
CORRECT — both on same Redis server:

WS3 → PUBLISH user:alice:location → Redis-7
WS1 → SUBSCRIBE user:alice:location → Redis-7   ← same server!

Result: WS1 receives Alice's update instantly. Bob gets notified. ✅
```

So the rule is: **for a given channel, all publishers AND all subscribers must go to the same Redis server.** The routing logic must be deterministic and consistent.

---

#### Naive Approach: Modulo Hashing (and why it breaks)

The simplest idea: hash the channel name, mod by number of servers.

```
Redis server index = hash("user:alice:location") % 140

hash("user:alice:location") = 4,892,041
4,892,041 % 140 = 41  → always goes to Redis-41
```

Every WebSocket server runs the same formula → always gets `41` → always connects to Redis-41 for Alice's channel. Publisher and subscriber agree. ✅

**The problem: you add or remove a server.**

```
Before: 140 servers
  hash("user:alice:location") % 140 = 41  → Redis-41

You add 1 server: now 141 servers
  hash("user:alice:location") % 141 = 103  → Redis-103  ← DIFFERENT!

You remove 1 server: now 139 servers
  hash("user:alice:location") % 139 = 78   → Redis-78   ← DIFFERENT AGAIN!
```

Every time you add or remove even one Redis server, **almost every channel gets remapped to a different server**. All WebSocket servers suddenly have wrong subscriptions. You get a massive storm of unsubscribe + resubscribe requests. Missed messages everywhere. System goes down.

---

#### The Real Solution: Consistent Hashing

<img src="../../system-design-notes/17. Nearby Friends/images/channel-distribution-data.png" alt="Channel Distribution" width="450">

<img src="../../system-design-notes/17. Nearby Friends/images/consistent-hashing.png" alt="Consistent Hashing" width="450">

Consistent hashing imagines all possible hash values arranged in a **circle (ring)** from 0 to 2³²-1. Redis servers are placed at positions on the ring. A channel maps to the **first Redis server clockwise from its hash position**.

**Step 1: Place Redis servers on the ring**

```
Hash ring (0 to 4,294,967,295):

         0
         │
    R2 ──┤ (hash=350M)
         │
    R5 ──┤ (hash=900M)
         │
    R1 ──┤ (hash=1.4B)
         │
    R4 ──┤ (hash=2.1B)
         │
    R3 ──┤ (hash=3.2B)
         │
         4,294,967,295  (wraps back to 0)
```

With 140 Redis servers, they are spread roughly evenly around the ring (using virtual nodes for balance — more on that below).

**Step 2: Hash a channel name to find its server**

```
Channel: "user:alice:location"
hash("user:alice:location") = 1,100,000,000

Walk clockwise from 1.1B on the ring:
  R2 at 350M  → already passed (counter-clockwise)
  R5 at 900M  → already passed
  R1 at 1.4B  → FIRST server clockwise from 1.1B ✅

→ Alice's channel lives on R1
```

```
Channel: "user:bob:location"
hash("user:bob:location") = 620,000,000

Walk clockwise from 620M:
  R2 at 350M  → already passed
  R5 at 900M  → FIRST server clockwise from 620M ✅

→ Bob's channel lives on R5
```

```
Channel: "user:charlie:location"
hash("user:charlie:location") = 3,800,000,000

Walk clockwise from 3.8B:
  R3 at 3.2B  → already passed
  (wrap around to 0...)
  R2 at 350M  → FIRST server clockwise from 3.8B (after wrap) ✅

→ Charlie's channel lives on R2
```

---

#### Step 3: Both Publisher and Subscriber Run the Same Hash

Every WebSocket server has the ring configuration in memory (synced from Zookeeper/etcd). They all run the exact same hash function.

```
WS3 (Alice's server) wants to PUBLISH "user:alice:location":
  hash("user:alice:location") → 1.1B → clockwise → R1
  → connects to R1, publishes ✅

WS1 (Bob's server) wants to SUBSCRIBE "user:alice:location":
  hash("user:alice:location") → 1.1B → clockwise → R1
  → connects to R1, subscribes ✅

Both independently computed the same answer: R1.
Publisher and subscriber are on the same Redis server.
Message flows correctly.
```

---

#### Step 4: What Happens When You Add a New Redis Server

Say you add **R6** at hash position `1,200,000,000` (between R1 at 1.4B and R5 at 900M on the ring).

```
Ring before adding R6:
  ...R5 (900M) → R1 (1.4B)...

Ring after adding R6:
  ...R5 (900M) → R6 (1.2B) → R1 (1.4B)...
```

Now recheck Alice's channel:
```
hash("user:alice:location") = 1,100,000,000

Before: clockwise from 1.1B → R1 (first at 1.4B)
After:  clockwise from 1.1B → R6 (first at 1.2B) ← NEW SERVER
```

Only channels whose hash falls **between 900M and 1.2B** are affected — they move from R1 to R6. All other channels stay exactly where they were.

```
With modulo hashing:  ~99% of channels remapped  ❌
With consistent hashing: ~1/141 of channels remapped  ✅ (only the slice R6 takes over)
```

WebSocket servers detect the ring change (via Zookeeper watch), re-subscribe only the affected channels on R6, and unsubscribe them from R1. A small, contained migration.

---

#### Virtual Nodes: Fixing Uneven Distribution

With only 140 real servers, they might cluster unevenly on the ring — one server gets 30% of channels, another gets 2%.

**Fix:** Each Redis server gets **150 virtual nodes** — 150 positions on the ring, all pointing to the same physical server.

```
R1 appears at positions: 1.4B, 0.2B, 2.8B, 3.6B, ...  (150 spots)
R2 appears at positions: 0.35B, 1.9B, 3.1B, 0.7B, ... (150 spots)
...
```

With 140 × 150 = 21,000 points on the ring, channels distribute **almost perfectly evenly** across all 140 Redis servers.

---

#### Summary

| Question | Answer |
|---|---|
| How does WS know which Redis? | Hash the channel name → walk ring clockwise → first Redis server |
| What if publisher and subscriber hash differently? | They can't — same channel name + same hash function = same result always |
| What if a Redis server is added? | Only ~1/N channels remapped, not all of them |
| What if a Redis server crashes? | Its channels shift to the next server clockwise — only those channels are affected |
| Where is the ring config stored? | Zookeeper/etcd — all WebSocket servers watch it and cache it in memory |

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
