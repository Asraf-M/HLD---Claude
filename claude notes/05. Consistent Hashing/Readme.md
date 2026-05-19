# System Design: Consistent Hashing (Beginner-Friendly Guide)

---

## What Are We Building?

Consistent hashing is not a service you build — it''s a **technique** you use inside distributed systems to decide: **which server should handle this data?**

It shows up everywhere:
- Apache Cassandra distributes database rows across nodes using it
- Amazon DynamoDB uses a similar ring approach
- Discord routes data to nodes with it
- Akamai CDN decides which edge server caches a URL using it
- Redis Cluster uses it to shard keys across shards

> **The core problem:** When you have multiple servers and want to spread data evenly across them — and when servers come and go — how do you do it without moving mountains of data?

---

## Step 1: The Naive Approach (And Why It Breaks)

The simplest idea: use modulo hashing.

```
serverIndex = hash(key) % N
```

Where N = number of servers.

<img src="../../system-design-notes/05. Consistent Hashing/images/server-hashing.png" alt="Server Hashing" width="450">

**Example with 4 servers (N=4):**
- `hash(key0) % 4 = 1` → server 1
- `hash(key1) % 4 = 0` → server 0
- `hash(key2) % 4 = 2` → server 2

This works great when N is fixed. But what happens when a server goes down?

### The Problem: One Server Leaves → Everything Reshuffles

<img src="../../system-design-notes/05. Consistent Hashing/images/server-hashing-miss.png" alt="Server Hashing Miss" width="450">

Now N = 3 (server 1 went down):
```
serverIndex = hash(key) % 3
```

- `hash(key0) % 3 = 0` → server 0 ← was on server 1, now moved!
- `hash(key1) % 3 = 1` → server 1 ← was on server 0, now moved!
- `hash(key2) % 3 = 2` → server 2 ← stayed
- `hash(key3) % 3 = 0` → server 0 ← was on server 0, now moved!

**Most of the keys moved to different servers!** In a distributed cache, this means a massive wave of cache misses — every client suddenly asks the wrong server, which goes to the database, causing a thundering herd problem.

> **Rule of thumb:** With modulo-N hashing, removing or adding 1 server remaps roughly **(N-1)/N** of all keys — almost all of them.

---

## Step 2: The Hash Ring — The Core Idea

Consistent hashing solves this with a clever visual metaphor: a **ring**.

### Building the Ring

<img src="../../system-design-notes/05. Consistent Hashing/images/hash-ring.png" alt="Hash Ring" width="400">

Imagine all possible hash values arranged in a line from 0 to 2^160-1 (SHA-1 produces 160-bit hashes). Now bend that line into a circle. The end connects back to the beginning.

This is the **hash ring**. Every possible hash value lives somewhere on this circle.

### Placing Servers on the Ring

<img src="../../system-design-notes/05. Consistent Hashing/images/server-ring.png" alt="Server Ring" width="450">

Hash each server''s IP address (or name) using the same hash function:
```
hash("server0_ip") → position on ring
hash("server1_ip") → position on ring
hash("server2_ip") → position on ring
hash("server3_ip") → position on ring
```

Each server lands at some position on the ring.

### Finding Which Server Owns a Key

<img src="../../system-design-notes/05. Consistent Hashing/images/server-lookup.png" alt="Server Lookup" width="450">

**Rule:** Hash the key → find its position on the ring → walk **clockwise** → the first server you hit is the owner.

```
hash("key0") → lands between s3 and s0 → clockwise → s0 owns it
hash("key1") → lands just before s1 → clockwise → s1 owns it
hash("key2") → lands just before s2 → clockwise → s2 owns it
hash("key3") → lands just before s3 → clockwise → s3 owns it
```

---

## Step 3: Adding a Server — Almost Nothing Changes

<img src="../../system-design-notes/05. Consistent Hashing/images/adding-server.png" alt="Adding Server" width="450">

When we add server 4 (s4) to the ring between s3 and s0:
- **Only the keys between s3 and s4** need to move from s0 to s4
- All other keys (key1 → s1, key2 → s2, key3 → s3) are **completely unaffected**

> **Why is this good?** With 4 servers and 1 new server added, only ~1/5 of keys move. With modulo hashing, ~4/5 would move. That''s a 4x reduction in data reshuffling.

---

## Step 4: Removing a Server — Same Story

<img src="../../system-design-notes/05. Consistent Hashing/images/removing-server.png" alt="Removing Server" width="450">

When server 1 (s1) goes down:
- **Only the keys between s0 and s1** need to move — they now walk clockwise to s2
- All other keys are completely unaffected

> **In a cache:** If s1 crashes, only s1''s keys cause cache misses. Everything else continues hitting the right server. No thundering herd.

---

## Step 5: The Problem With Basic Consistent Hashing

The basic approach has two issues:

### Issue 1: Uneven Partition Sizes

Servers land at random positions on the ring. If s1 lands very close to s0, s1 owns a tiny slice of the ring. If s2 lands far from s1, s2 owns a huge slice. Some servers get way more data than others.

### Issue 2: Non-Uniform Key Distribution

Even if server positions look balanced, certain hash ranges might have many more keys than others (especially with non-uniformly distributed keys).

**Both problems are solved with Virtual Nodes.**

---

## Step 6: Virtual Nodes — The Real Production Solution

<img src="../../system-design-notes/05. Consistent Hashing/images/virtual-nodes.png" alt="Virtual Nodes" width="450">

Instead of placing each server once on the ring, place it **multiple times** at different positions. Each placement is called a **virtual node (vnode)**.

**Example with 2 servers, 3 vnodes each:**
```
Server 0 → positions: s0_0, s0_1, s0_2 (spread around the ring)
Server 1 → positions: s1_0, s1_1, s1_2 (spread around the ring)
```

Now the ring alternates: s0, s1, s0, s1, s0, s1 — evenly interleaved!

**In production:** 100–200 virtual nodes per physical server is typical. With 3 servers × 150 vnodes = 450 ring positions, the standard deviation of load per server drops to ~10% of average — very uniform.

### Lookup is the same — just binary search

Store all vnode positions in a **sorted array**. For a key:
1. Compute `hash(key)` → get position
2. Binary search the sorted array for the first position ≥ key position
3. That vnode''s server is the owner

Time complexity: **O(log N)** where N = total vnodes

### Virtual Nodes for Heterogeneous Servers

Have a beefy server with 2× the RAM? Give it 2× the vnodes. It''ll own ~2× the key space — proportional to its capacity.

---

## Step 7: Which Keys Are Affected When Servers Change?

### Adding a Server

<img src="../../system-design-notes/05. Consistent Hashing/images/server-addition.png" alt="Server Addition" width="450">

When s4 is added between s3 and s0 on the ring:
- Move **anticlockwise** from s4 until you find the previous server (s3)
- Keys between s3 and s4 must migrate from s0 → s4
- Everything else: untouched

### Removing a Server

<img src="../../system-design-notes/05. Consistent Hashing/images/server-removed.png" alt="Server Removed" width="450">

When s1 is removed:
- Move **anticlockwise** from s1''s position until you find s0
- Keys between s0 and s1 must migrate from s1 → s2 (next clockwise server)
- Everything else: untouched

---

## Step 8: Replication With Consistent Hashing

For fault tolerance (e.g., replication factor R=3), each key is stored on the next **R distinct physical servers** clockwise from its position.

```
key position → walk clockwise:
  - 1st server → primary (handles writes, coordinates reads)
  - 2nd server → replica 1
  - 3rd server → replica 2
  (skip if same physical server as previous, even if different vnode)
```

This is exactly how **Apache Cassandra** works. With R=3, the system tolerates 2 simultaneous server failures.

---

## Visual Summary: Naive vs Consistent

| | Modulo-N Hashing | Consistent Hashing |
|--|-----------------|-------------------|
| Server added | ~(N-1)/N keys move | ~K/N keys move (just 1/N fraction) |
| Server removed | ~(N-1)/N keys move | Only that server''s keys move |
| Hotspots | Possible with modulo | Mitigated by virtual nodes |
| Implementation | `key % N` | Sorted ring + binary search |
| Lookup time | O(1) | O(log N) |
| Flexibility | N must be fixed | N can change freely |

---

## Real-World Applications

| System | How it uses consistent hashing |
|--------|-------------------------------|
| **Apache Cassandra** | Partitions rows across nodes using a token ring; vnodes for even distribution |
| **Amazon DynamoDB** | Virtual partitions on a ring; auto-rebalancing |
| **Discord** | Routes user data to Elixir nodes |
| **Akamai CDN** | Decides which edge server caches a given URL |
| **Redis Cluster** | Divides key space into 16,384 hash slots assigned to nodes |
| **Google Maglev** | Consistent hashing with bounded loads for network load balancing |

---

## Summary: The "Why" Behind Each Decision

| Decision | Why? |
|----------|------|
| Hash ring instead of modulo-N | Modulo-N remaps ~all keys when N changes; ring remaps only ~1/N keys |
| Clockwise lookup rule | Any deterministic traversal direction works; clockwise is the convention |
| Virtual nodes | Evens out key distribution; prevents single servers from owning huge ring arcs |
| 100–200 vnodes per server | Empirically good balance between uniformity and memory overhead |
| Binary search on sorted ring | O(log N) lookup — fast enough; avoids the O(N) of a linear scan |
| Assign more vnodes to bigger servers | Capacity-proportional load distribution in heterogeneous clusters |
| Replicate to next R distinct servers clockwise | Simple, deterministic replication; easy to determine replicas without coordination |

---

## Quick Interview Q&A Cheat Sheet

**Q: What problem does consistent hashing solve?**  
A: With modulo-N hashing (`hash(key) % N`), adding or removing one server remaps ~(N-1)/N of all keys — almost everything. Consistent hashing remaps only K/N keys (K = total keys, N = servers). This is critical for distributed caches where mass remapping causes thundering herd from cache misses, and for databases where mass data movement is expensive.

**Q: How does the hash ring work?**  
A: All hash values (0 to 2^160-1 for SHA-1) form a circle. Each server is hashed to a position on this ring. A key is hashed to a position, then you walk clockwise to find the first server — that server owns the key. Adding a server only takes keys from its clockwise neighbor; removing only gives keys to the next clockwise server.

**Q: What is a virtual node and why do we need it?**  
A: Each physical server is placed at multiple positions on the ring (virtual nodes / vnodes). Without vnodes, servers land at random positions — one server might own a huge arc and get most of the keys (hotspot). With 100–200 vnodes per server, the ring alternates evenly between servers, distributing keys uniformly. More powerful servers can be assigned more vnodes proportionally.

**Q: How do you look up which server owns a key with virtual nodes?**  
A: Store all vnode positions in a sorted array. Compute `hash(key)` → binary search for the first position ≥ that hash value (wrap around to the smallest if none found) → that entry tells you which physical server owns the key. O(log N) where N = total vnode positions.

**Q: How does adding a server work in consistent hashing?**  
A: Place the new server''s vnode positions on the ring. For each new vnode position: identify the predecessor (walk anticlockwise to the previous server) → migrate keys between predecessor and new vnode position from the current owner to the new server. Only those keys move. Everything else is unaffected.

**Q: How is consistent hashing used for replication?**  
A: For replication factor R, each key is assigned to the next R distinct physical servers clockwise from its position on the ring. The first is the primary (handles writes); the rest are replicas. This is Apache Cassandra''s approach. With R=3, the cluster tolerates 2 simultaneous server failures with no data loss.

**Q: Which hash function should you use for consistent hashing?**  
A: MurmurHash or xxHash — both are fast and have uniform output distribution. Cryptographic functions (SHA-256, MD5) work but are slower than needed; consistent hashing doesn''t require collision resistance, just uniform distribution. Cassandra uses Murmur3 by default.
