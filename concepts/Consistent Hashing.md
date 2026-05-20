# Consistent Hashing

## What Is It?

A technique to distribute data across servers so that **adding or removing a server only moves a small fraction of data**, not everything.

---

## The Problem With Naive Hashing

```
serverIndex = hash(key) % N
```

Works great when N is fixed. When a server goes down (N → N-1), almost **all keys remap** to different servers → cache miss storm → database overload.

With modulo-N: removing 1 of 4 servers remaps ~75% of keys.  
With consistent hashing: removing 1 of 4 servers remaps ~25% of keys.

---

## The Hash Ring

1. Imagine all hash values (0 → 2^160) bent into a **circle** (ring)
2. Each server is hashed to a **position** on the ring
3. Each key is hashed to a position; it belongs to the **first server clockwise** from it

```
Ring: ... s3 ... k0 ... s0 ... k1 ... s1 ... k2 ... s2 ... k3 ...
k0 → walks clockwise → hits s0 → s0 owns k0
k1 → walks clockwise → hits s1 → s1 owns k1
```

**Adding a server:** only keys between the new server and its predecessor move.  
**Removing a server:** only that server's keys move to the next clockwise server.

---

## Virtual Nodes (The Real Solution)

Basic consistent hashing: servers land at random spots → uneven load.

**Virtual nodes:** each physical server gets **100–200 positions** on the ring.

```
Server 0 → positions: s0_0, s0_1, s0_2, ... (spread evenly around ring)
Server 1 → positions: s1_0, s1_1, s1_2, ... (interleaved with s0)
```

Result: keys distribute evenly across all servers.

**Heterogeneous servers:** give a 2× capacity server 2× the virtual nodes → it owns 2× the key space.

---

## Implementation

```java
import java.security.MessageDigest;
import java.math.BigInteger;
import java.util.Map;
import java.util.TreeMap;

public class ConsistentHashRing {
    private final TreeMap<Long, String> ring = new TreeMap<>();
    private final int vnodes;

    public ConsistentHashRing(int vnodes) {
        this.vnodes = vnodes;
    }

    public void addServer(String serverId) {
        for (int i = 0; i < vnodes; i++) {
            long pos = hash(serverId + "#" + i);
            ring.put(pos, serverId);
        }
    }

    public String getServer(String dataKey) {
        long pos = hash(dataKey);
        Map.Entry<Long, String> entry = ring.ceilingEntry(pos);
        if (entry == null) entry = ring.firstEntry();  // wrap around
        return entry.getValue();
    }

    private long hash(String key) {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] digest = md.digest(key.getBytes());
            return new BigInteger(1, digest).longValue() & 0xFFFFFFFFL;
        } catch (Exception e) { throw new RuntimeException(e); }
    }
}
```

Lookup: **O(log N)** via binary search on sorted ring positions.

---

## Replication

For replication factor R=3, each key is stored on the **next R distinct physical servers** clockwise:

```
key position → server A (primary) → server B (replica 1) → server C (replica 2)
```

Skip vnodes of the same physical server when counting. This is how **Apache Cassandra** works.

---

## Where It's Used

| System | Usage |
|--------|-------|
| Apache Cassandra | Token ring for row partitioning |
| Amazon DynamoDB | Virtual partitions |
| Redis Cluster | 16,384 hash slots assigned to nodes |
| Akamai CDN | Which edge server caches a URL |
| Discord | Routes user data to nodes |

---

## Quick Reference

| | Modulo-N | Consistent Hashing |
|--|----------|-------------------|
| Keys remapped on 1 change | ~(N-1)/N ≈ all | ~1/N ≈ few |
| Hotspot risk | Yes | Mitigated by vnodes |
| Lookup time | O(1) | O(log N) |
| Add/remove server | Massive disruption | Minimal disruption |

---

## Interview Q&A

**Q: What problem does consistent hashing solve?**  
A: Modulo-N remaps ~all keys when N changes. Consistent hashing only remaps ~K/N keys (K=total keys). Critical for distributed caches where mass remapping causes thundering herd.

**Q: What is a virtual node?**  
A: Multiple ring positions for one physical server. Prevents uneven load when servers land at random positions. 100–200 vnodes per server is typical in production.

**Q: How does adding a server work?**  
A: New server placed on ring. Keys between the new server and its predecessor (anticlockwise) migrate from the old owner. Everything else unaffected.
