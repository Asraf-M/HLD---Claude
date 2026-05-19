# Bloom Filters

## What Is It?

A **Bloom filter** is a space-efficient probabilistic data structure that answers one question:

> **"Is this element in the set?"**

With two possible answers:
- **"Definitely NOT in the set"** — 100% certain (no false negatives)
- **"Probably in the set"** — might be wrong (false positives possible)

> **Analogy:** Airport security has a list of banned passengers. They use a quick bloom filter check first. If it says "not banned" → definitely let them through. If it says "possibly banned" → do a full database lookup to confirm. The bloom filter saves 99% of expensive DB lookups.

---

## Why Not Just Use a Hash Set?

A hash set (`{element1, element2, ...}`) gives exact answers — but stores every element.

| | Hash Set | Bloom Filter |
|--|----------|-------------|
| Memory for 1 billion strings | ~50 GB | ~1.2 GB |
| False positives | No | Yes (tunable, e.g. 1%) |
| False negatives | No | **Never** |
| Deletion | Easy | Difficult (standard) |
| Lookup time | O(1) | O(k) — k hash functions |

When you have **billions of items** and exact membership isn't strictly required, a Bloom filter uses orders of magnitude less memory.

---

## How It Works

### Setup
A Bloom filter is just a **bit array** of size `m` (all bits start at 0) + **k hash functions**.

```
Bit array (m=10):
[0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
 0  1  2  3  4  5  6  7  8  9
```

### Adding an Element

To add `"apple"`:
1. Run through all k hash functions: `h1("apple")=1`, `h2("apple")=4`, `h3("apple")=7`
2. Set bits 1, 4, 7 to 1

```
[0, 1, 0, 0, 1, 0, 0, 1, 0, 0]
 0  1  2  3  4  5  6  7  8  9
```

To add `"banana"`:
1. `h1("banana")=2`, `h2("banana")=4`, `h3("banana")=9`
2. Set bits 2, 4, 9 to 1 (bit 4 was already 1)

```
[0, 1, 1, 0, 1, 0, 0, 1, 0, 1]
 0  1  2  3  4  5  6  7  8  9
```

### Checking an Element

To check `"apple"`:
1. Run through hash functions: bits 1, 4, 7
2. All bits are 1 → **"probably in set"** ✅

To check `"cherry"`:
1. `h1("cherry")=2`, `h2("cherry")=5`, `h3("cherry")=9`
2. Bit 5 is 0 → **"definitely NOT in set"** ❌

To check `"mango"`:
1. `h1("mango")=1`, `h2("mango")=2`, `h3("mango")=4`
2. All bits are 1 — but "mango" was never added!
3. **False positive** — all bits were set by other elements

---

## False Positive Rate

The probability of a false positive depends on:
- `m` — size of the bit array
- `n` — number of elements inserted
- `k` — number of hash functions

$$\text{False positive rate} \approx \left(1 - e^{-kn/m}\right)^k$$

**Practical rule:** To achieve ~1% false positive rate with n elements:
- Use ~10 bits per element: `m = 10n`
- Optimal k ≈ 7 hash functions

| Elements | 1% FP rate | Memory |
|----------|-----------|--------|
| 1 million | ~1.25 MB | — |
| 100 million | ~125 MB | — |
| 1 billion | ~1.25 GB | — |

Compare: storing 1 billion 50-byte strings = **50 GB**. Bloom filter = **1.25 GB** at 1% FP rate.

---

## Important Properties

### No False Negatives
If an element was added, the Bloom filter will **always** say "probably yes".

This is guaranteed because adding an element sets specific bits, and those bits never get unset (in standard Bloom filters).

### Deletion is Hard
You can't unset a bit — it might have been set by another element.

**Solution: Counting Bloom Filter** — use counters instead of bits. Increment on add, decrement on delete. Costs more memory (4 bits per slot instead of 1).

### Not Exact
The "probably in set" answer might be wrong. The error rate is tunable via m and k.

---

## Real-World Uses

### 1. Web Crawlers (Google)
**Problem:** Has this URL already been crawled?  
Storing billions of URLs in a hash set: ~50 GB  
Bloom filter: ~1.2 GB

If Bloom filter says "not seen" → crawl it.  
If Bloom filter says "possibly seen" → skip (acceptable false positive rate ~0.1% means occasional missed re-crawl).

### 2. Database Query Optimization (Cassandra, HBase)
**Problem:** An SSTable (disk file) might not have the requested key. Reading from disk is slow.

Each SSTable has a Bloom filter. Before reading from disk:
- Check Bloom filter: "definitely not here" → skip this file entirely
- "Possibly here" → read from disk to confirm

Reduces disk I/O dramatically for key lookups.

### 3. Cache Penetration Prevention
**Problem:** Malicious requests for keys that don't exist bypass the cache and hammer the DB.

```
Cache miss → DB query → "not found" → cache not populated → next request also hits DB
```

**Solution:** Bloom filter of all valid keys. If key not in Bloom filter → return 404 immediately, don't touch DB.

### 4. Password Breach Checking (HaveIBeenPwned)
**Problem:** Check if a password hash is in a list of 500 million breached passwords.

Bloom filter of all breached hashes. Fast check before any DB lookup. False positives = tell user their password might be breached (conservative is safe here).

### 5. Duplicate Detection
- Email systems: "Has this email been sent to this recipient recently?"
- Ad systems: "Has this user seen this ad?"
- Spam filters: "Is this URL known-spam?"

---

## Bloom Filter vs Other Structures

| Structure | Exact? | Memory | Deletion | Use Case |
|-----------|--------|--------|---------|---------|
| Hash Set | ✅ Yes | High | Easy | Small-medium sets where exact membership needed |
| Bloom Filter | ❌ Probabilistic | **Very Low** | Hard | Billions of items, FP ok, FN not ok |
| Cuckoo Filter | ❌ Probabilistic | Low | ✅ Yes | When deletion needed (improved Bloom) |
| HyperLogLog | Count only | Very Low | N/A | Counting unique elements (not membership) |

---

## Implementation Sketch

```python
import hashlib
import math

class BloomFilter:
    def __init__(self, n_items, fp_rate=0.01):
        # Calculate optimal m and k
        m = math.ceil(-n_items * math.log(fp_rate) / (math.log(2)**2))
        k = math.ceil((m / n_items) * math.log(2))
        
        self.m = m
        self.k = k
        self.bits = bytearray(math.ceil(m / 8))
    
    def _hashes(self, item):
        positions = []
        for i in range(self.k):
            h = int(hashlib.md5(f"{i}:{item}".encode()).hexdigest(), 16)
            positions.append(h % self.m)
        return positions
    
    def add(self, item):
        for pos in self._hashes(item):
            self.bits[pos // 8] |= 1 << (pos % 8)
    
    def contains(self, item):
        return all(
            self.bits[pos // 8] & (1 << (pos % 8))
            for pos in self._hashes(item)
        )

bf = BloomFilter(n_items=1_000_000, fp_rate=0.01)
bf.add("apple")
print(bf.contains("apple"))   # True (definitely)
print(bf.contains("cherry"))  # False (definitely not)
print(bf.contains("mango"))   # Might be True (false positive, ~1%)
```

---

## Interview Q&A

**Q: What is a Bloom filter and what problem does it solve?**  
A: A space-efficient probabilistic data structure for set membership queries. Answers "definitely not in set" (always correct) or "probably in set" (may have false positives). Solves the problem of checking membership in very large sets (billions of items) where storing every element is too expensive. Uses ~10 bits per element for 1% false positive rate vs. ~400 bits per element for exact storage.

**Q: Can a Bloom filter have false negatives?**  
A: No — never. If an element was added, all its bits are set. Checking it will always find those bits set → always returns "probably in set". False negatives are impossible by design. Only false positives are possible (some bits were set by other elements).

**Q: How is a Bloom filter used in Cassandra?**  
A: Each SSTable (on-disk data file) has a Bloom filter containing the keys it stores. On a read, before reading from disk, Cassandra checks the Bloom filter. "Definitely not here" → skip this file entirely. "Possibly here" → read from disk. This eliminates most unnecessary disk I/O for key lookups, dramatically improving read performance.

**Q: How do you tune a Bloom filter's false positive rate?**  
A: Increase the bit array size m (more bits per element) or choose optimal k (number of hash functions). Rule of thumb: 10 bits per element + 7 hash functions → ~1% FP rate. 20 bits per element → ~0.1% FP rate. The trade-off is memory vs. accuracy. You can't get exactly 0% FP without exact data structures (hash sets).

**Q: What is the difference between a Bloom filter and a hash set?**  
A: Hash set stores exact data — 100% accurate, no false positives or negatives — but uses much more memory. Bloom filter stores only bit flags — much smaller (orders of magnitude) but can have false positives. Use a hash set when exactness is required and data volume is manageable. Use a Bloom filter when you have billions of items and can tolerate occasional false positives.
