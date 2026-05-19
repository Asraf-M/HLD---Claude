# Database Replication

## What Is It?

**Replication** copies data from one database server (the **leader/primary**) to one or more other servers (**followers/replicas/secondaries**).

> **Analogy:** A bank's central vault (leader) keeps the real records. Branch offices (replicas) have copies of customer data so they can serve customers locally — but all official updates go through the central vault first.

---

## Why Replicate?

| Goal | How Replication Helps |
|------|----------------------|
| **High availability** | If the leader crashes, promote a replica |
| **Read scaling** | Spread read queries across multiple replicas |
| **Disaster recovery** | Replica in another data center survives regional failure |
| **Analytics isolation** | Run heavy analytical queries on replica, not production leader |
| **Geo distribution** | Serve reads from a replica close to the user |

---

## Leader-Follower Replication (Most Common)

One **leader** handles all writes. One or more **followers** replicate the leader and handle reads.

```
                 ┌─────────────┐
Writes ─────────→│   LEADER    │
                 │  (Primary)  │──── replicates ──→ Follower 1 ──→ Read queries
                 └─────────────┘                 └──────────────→ Follower 2
```

### How replication works (under the hood)

Leader writes to its **WAL (Write-Ahead Log)** or **binlog** (MySQL). Followers connect and stream this log, replaying every change.

```
Leader WAL:
  [INSERT users (id=1, name='Alice')]
  [UPDATE orders SET status='shipped' WHERE id=42]
  [DELETE sessions WHERE expires < now()]

Follower replays same operations → identical data
```

### Synchronous vs Asynchronous Replication

**Synchronous:**
- Leader waits for follower to confirm receipt before acknowledging the write to the client
- **Pro:** Follower is guaranteed to have latest data
- **Con:** Write latency increases; if follower is slow, leader is blocked
- Used for: at most 1 synchronous follower (semi-sync)

**Asynchronous (default in most systems):**
- Leader writes and acknowledges immediately; follower catches up later
- **Pro:** Fast writes, follower lag doesn't affect leader
- **Con:** If leader crashes before follower catches up → **data loss of recent writes**
- **Replication lag** — follower may be seconds behind leader

---

## Replication Lag Problems

Async replication creates a window where follower data is stale.

### Read-Your-Own-Write Problem
```
1. User updates their profile photo (write → leader)
2. User immediately refreshes the page (read → replica)
3. Replica hasn't caught up yet → user sees OLD photo
4. User thinks their update was lost!
```

**Solution:** After a write, route that user's reads to the leader for a short window (e.g., 1 minute), or always read from leader for the user's own profile.

### Monotonic Reads Problem
```
1. First read → follower A (replication lag: 1s) → sees tweet from 2 seconds ago
2. Second read → follower B (replication lag: 5s) → doesn't see that tweet yet!
3. Data appears to "go backwards"
```

**Solution:** Route all reads for a given user to the same replica (sticky routing by user ID hash).

---

## Failover

When the leader crashes, a follower must be promoted to the new leader.

### Automatic Failover Steps
1. **Detect leader failure** — health check fails, heartbeat timeout
2. **Choose a new leader** — elect the follower with the most up-to-date data
3. **Reconfigure clients** — update connection strings to point to new leader
4. **Reconfigure old leader** — if it comes back, it becomes a follower

### Failover Problems
- **Split-brain:** Old leader comes back and both servers think they're the leader → two conflicting write streams
  - Solution: Force the old leader to step down (STONITH — Shoot The Other Node In The Head)
- **Data loss:** Async replication → new leader may be missing recent writes
- **Clients using stale connection info** — need service discovery (ZooKeeper, etcd, Consul)

---

## Multi-Leader Replication

Multiple nodes accept writes. Each replicates to all others.

```
Leader A ←──replicates──→ Leader B
    ↑                          ↑
 Writes                    Writes
```

**Use cases:**
- Multi-datacenter writes (users in US write to US leader, EU users write to EU leader)
- Offline clients (phone syncs when back online — phone is a "leader" while offline)

**Main problem: Write conflicts**

```
User 1 sets title = "Foo" on Leader A
User 2 sets title = "Bar" on Leader B (same millisecond)
Both replicate to each other → conflict!
```

**Conflict resolution strategies:**
- **Last Write Wins (LWW):** Keep whichever write has the latest timestamp (risk of data loss)
- **Application-level merge:** Let the application define how to merge conflicts (e.g., CRDTs in collaborative editors)
- **Avoid conflicts:** Route all writes for a given record to the same leader

---

## Leaderless Replication (Dynamo-style)

No single leader. Any node accepts writes. Data is written to N nodes simultaneously.

```
Write: send to all 3 nodes, wait for quorum (2 of 3) to confirm
Read: send to all 3 nodes, take the most recent value
```

**Quorum condition:**
- `w` = write quorum (min nodes that must confirm write)
- `r` = read quorum (min nodes that must respond to read)
- As long as `w + r > n`, at least one read will see the latest write

Example: n=3, w=2, r=2 → w+r=4 > 3 ✅ → always consistent

**Used by:** Apache Cassandra, Amazon Dynamo, Riak

**Trade-off:** Higher availability (can lose up to n-w nodes on writes), tunable consistency.

---

## Read Replicas in Practice

```sql
-- Application code: route reads vs writes
def get_user(user_id):
    return read_replica_db.query("SELECT * FROM users WHERE id = ?", user_id)

def update_user(user_id, data):
    return leader_db.execute("UPDATE users SET ... WHERE id = ?", user_id, data)
```

**Connection pooling libraries** (PgBouncer, ProxySQL) handle routing automatically.

**Rule of thumb:** Most web apps are 90% reads, 10% writes. Read replicas let you scale reads by adding more replicas.

---

## Replication in Popular Databases

| Database | Default Mode | Notes |
|----------|-------------|-------|
| **PostgreSQL** | Async leader-follower | WAL streaming; synchronous mode available |
| **MySQL** | Async leader-follower | Binlog-based; semi-sync available |
| **MongoDB** | Replica set (1 primary, N secondaries) | Automatic failover via Raft |
| **Cassandra** | Leaderless, tunable quorum | RF=3 most common |
| **Redis** | Async leader-follower | Redis Sentinel for failover; Redis Cluster for sharding |

---

## Replication vs Sharding

| | Replication | Sharding |
|--|-------------|---------|
| **Purpose** | High availability + read scaling | Storage + write scaling |
| **Each node has** | Full copy of data | Subset of data |
| **Writes handled by** | Leader only (or multi-leader) | Multiple shards in parallel |
| **Use together?** | ✅ Yes — shard + replicate each shard |  |

Real systems use both: data is sharded across N shards, each shard has 3 replicas.

---

## Interview Q&A

**Q: What is the difference between leader-follower and leaderless replication?**  
A: Leader-follower: one node accepts all writes, others replicate and serve reads. Simple to reason about, easy conflict avoidance. Leaderless: any node accepts writes, quorum-based consistency. Higher availability (no leader bottleneck) but requires conflict resolution strategies. Cassandra uses leaderless; PostgreSQL/MySQL use leader-follower.

**Q: What is replication lag and what problems does it cause?**  
A: With async replication, followers may be seconds behind the leader. Problems: (1) Read-your-own-write — user writes something then immediately reads from a stale replica and doesn't see their change. (2) Monotonic reads — two reads hit different replicas with different lag, data appears to go backwards. Solutions: route reads to leader for recent writes, or use consistent routing to the same replica per user.

**Q: What is split-brain and how do you prevent it?**  
A: Split-brain occurs when a network partition makes both the old leader and the newly-promoted leader think they're primary — both accept writes, diverging data. Prevention: fencing (STONITH — revoke old leader's write access), or require quorum majority to elect a new leader (so old leader in minority partition can't accept writes).

**Q: What is the quorum condition in leaderless replication?**  
A: With n replicas, write quorum w, read quorum r: as long as w + r > n, at least one node in any read set overlaps with the write set — guaranteeing the latest write is always visible. Example: n=3, w=2, r=2. If you need high availability for writes (survive 1 node down), use w=2. If you need strong read consistency, use r=2. Trade-off: lowering w or r improves availability but can allow stale reads.

**Q: When would you use read replicas vs a cache?**  
A: Read replicas: for complex SQL queries that can't be cached easily, when data freshness must be within seconds (cache might be stale for minutes), or when you need full query flexibility. Cache (Redis): for identical repeated reads, session data, computed results — sub-millisecond latency but requires explicit invalidation logic. Many systems use both: cache for hot data, read replicas for complex queries.
