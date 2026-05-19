# Chapter 2: Back-of-the-Envelope Estimation

## Introduction
Back-of-the-envelope estimation is a crucial skill in system design interviews. It involves making quick, rough calculations to assess system capacity or performance. According to Jeff Dean, Google Senior Fellow, these estimates help evaluate whether designs meet requirements through thought experiments and common performance benchmarks.

This chapter covers key concepts, methodologies, and examples to build proficiency in scalability and estimation.

---

## Section 1: Key Concepts

### Power of Two
Understanding data volume in terms of powers of two is fundamental:

<img src="./images/power-of-two.png" alt="power-of-two" width="500" />

This knowledge helps in performing accurate storage and bandwidth calculations.

---

### Latency Numbers Every Programmer Should Know
Latency numbers represent the time taken for various operations in computing systems. These provide insights into relative performance:

| Operation                | Latency (2020) |
|--------------------------|----------------|
| L1 Cache Access          | 0.5 ns         |
| L2 Cache Access          | 7 ns           |
| Main Memory Access       | 100 ns         |
| SSD Random Read          | 150 µs         |
| HDD Random Seek          | 10 ms          |
| Round-Trip in Data Center| 500 µs         |
| Inter-Region Data Center | 150 ms         |

**Key Insights:**
- Memory is fast, disk is slow.
- Avoid disk seeks whenever possible.
- Compress data before transmitting over the internet to save bandwidth.


---

### Availability Numbers
High availability (HA) ensures minimal downtime. Availability is expressed in **nines**:
- **99% (Two Nines):** ~3.65 days/year of downtime
- **99.9% (Three Nines):** ~8.8 hours/year of downtime
- **99.99% (Four Nines):** ~52 minutes/year of downtime
- **99.999% (Five Nines):** ~5.3 minutes/year of downtime
- **99.9999% (Six Nines):** ~31.56 seconds/year of downtime


Cloud providers like Amazon, Google, and Microsoft aim for SLAs (Service Level Agreements) of **99.9% or higher**.

---

## Section 2: Example Estimation - Twitter QPS and Storage Requirements

### Assumptions
- **300 million monthly active users (MAU).**
- **50% daily active users (DAU).**
- **Average tweets/user/day:** 2.
- **10% of tweets contain media.**
- **Data retention:** 5 years.

### Estimations
1. **Query Per Second (QPS):**
   - DAU = \( 300M x 50\% = 150M \)
   - Tweets QPS = \( 150M x 2 tweets / 24 hour / 3600 seconds = ~3500 )
   - Peak QPS = \( 2 x 3500 = ~7000 \)

2. **Media Storage:**
   - **Tweet Size Components:**
     - `tweet_id`: 64 bytes
     - `text`: 140 bytes
     - `media`: 1 MB
   - **Daily Media Storage:** \( 150M x 2 x 10\% x 1MB = 30TB per day \)
   - **5-Year Storage:** \( 30TB x 365 x 5 = ~55PB \)

---

## Section 3: Tips for Effective Estimation

### 1. Rounding and Approximation
Precision is not critical; focus on the process. Simplify complex calculations using round numbers. For example:
- \( 99987 / 9.1 \) can be approximated as \( 100,000 / 10 = 10,000 \).

### 2. Write Down Assumptions
Document assumptions clearly for future reference.

### 3. Label Units
Avoid ambiguity by labeling units (e.g., `5 MB` instead of `5`).

### 4. Common Estimation Scenarios
- **QPS (Queries Per Second):** Measure traffic intensity.
- **Peak QPS:** Account for traffic spikes.
- **Storage Requirements:** Estimate total data needs.
- **Cache Requirements:** Evaluate memory requirements for caching.
- **Number of Servers:** Calculate hardware needs based on workload.

---

## Most Asked Interview Questions

**Q1. Estimate the QPS and storage requirements for a Twitter-like system with 300M DAU.**
> Assume each user makes ~10 requests/day → 300M × 10 / 86,400 ≈ 35,000 QPS (average); peak ~3–5× = ~100K–175K QPS. Each tweet ~280 chars ≈ 300 bytes. If 10% of users tweet once/day → 30M tweets/day × 300 bytes ≈ 9 GB/day → ~3.3 TB/year for text alone (media storage would be much larger).

**Q2. How would you estimate the bandwidth requirements for a video streaming service?**
> Assume 5 million concurrent viewers, each streaming at 5 Mbps average → 5M × 5 Mbps = 25 Tbps of outbound bandwidth. For storage, 500 hours of video uploaded per minute × 60 min = 30,000 hours/hour; at ~1.5 GB/hr per video → ~45 TB of new video per hour.

**Q3. What are the typical latency numbers every engineer should know?**
> L1 cache reference: ~0.5 ns; L2 cache: ~7 ns; RAM read: ~100 ns; SSD random read: ~100 μs; Network round trip in same datacenter: ~500 μs; HDD seek: ~10 ms; Cross-continent network round trip: ~150 ms. These inform when caching, local processing, or async are appropriate.

**Q4. Estimate the number of servers needed to handle 10M requests per day.**
> 10M / 86,400 ≈ 116 RPS average. A modern server can handle ~1,000 RPS for lightweight API requests. So ~1 server suffices for average load. For peak (3–5×) = ~350–600 RPS, still ~1 server. However, you add more for fault tolerance, so a minimum of 2–3 servers in practice.

**Q5. How would you estimate the cache memory needed for a news feed system?**
> Apply the 80/20 rule: 20% of content drives 80% of reads. If you have 10M DAU and each feed fetches 20 posts averaging 1 KB each → 200 KB per user × 10M = 2 TB of total data; 20% of that = 400 GB of hot cache needed.

**Q6. What is the typical read-to-write ratio for social media platforms?**
> Social media platforms are typically heavily read-dominant. Twitter is approximately 100:1 (read:write); Instagram feed reads vastly outnumber photo uploads. This justifies heavy investment in read optimization — caching, fanout on write, read replicas — rather than write optimization.

**Q7. Estimate the storage required for 10 years of Tweets (140 chars, 100M tweets/day).**
> 140 chars × 2 bytes + 30 bytes metadata ≈ 310 bytes/tweet. 100M tweets/day × 310 bytes = 31 GB/day × 365 days × 10 years ≈ 113 TB for text alone. Add indexes (~2×): ~226 TB. Media (photos, videos) would increase this by 100× or more.

**Q8. How do you derive QPS from DAU and average requests per user per day?**
> QPS = (DAU × avg_requests_per_user) / 86,400. For peak QPS, multiply by a peak factor (typically 2–5×). Example: 10M DAU, 30 requests/user/day → 10M × 30 / 86,400 ≈ 3,472 avg QPS; peak ≈ 10,000–17,000 QPS.

**Q9. What is the difference between throughput and latency?**
> Latency is the time for a single operation to complete (e.g., 50ms per request). Throughput is how many operations the system completes per second (e.g., 10,000 RPS). By Little's Law: Throughput = Concurrency / Latency. Optimizing one often hurts the other; batch processing increases throughput but increases per-item latency.

**Q10. Walk through a back-of-the-envelope estimation for designing a URL shortener.**
> 100M URLs generated/day → ~1,157 write QPS average, ~5,000 peak. Read:write = 10:1 → 50,000 read QPS peak. URL record: 100 bytes × 100M/day × 365 × 10 years = 36.5 TB. Cache: 20% of URLs drive 80% reads → cache top 20M URLs × 100 bytes = 2 GB, very manageable. Need ~5–10 servers for peak read QPS.

**Q11. How many servers does Google need to handle its search traffic?**
> Google handles ~8.5B searches/day ≈ ~100,000 QPS. A single powerful search server might handle ~100 QPS due to complex processing → ~1,000 servers minimum for search alone. In reality, Google runs millions of servers across all services, with massive redundancy, caching layers, and geographic distribution.

**Q12. How do you estimate storage for a photo-sharing app like Instagram?**
> 100M DAU, 10% upload 1 photo/day = 10M photos/day. Average photo = 5 MB → 50 TB/day raw. After compression/resizing to 3 formats: ~15 TB/day effective. Over 10 years = ~54 PB. CDN cache for popular photos would serve ~80% of reads from cache.

**Q13. What are the 2 most important metrics to estimate in a system design interview?**
> QPS (Queries Per Second) — determines how many servers, load balancers, and caches you need. Storage — determines database sizing, sharding strategy, and replication cost. Everything else (bandwidth, memory) can be derived from these two.

**Q14. How do you estimate the memory needed for a cache that serves a given QPS?**
> Identify the working set: the data accessed in 80% of requests. If QPS = 50,000 and average object = 1 KB, with 20% of objects serving 80% traffic → cache the hot 20%. If total objects = 10M × 1 KB = 10 GB, cache 20% = 2 GB. A single Redis instance handles this comfortably.

**Q15. How do you account for data growth in your estimations?**
> Assume linear growth based on user acquisition projections. Size your system for 1–2 years of projected growth to avoid frequent re-architecting. Apply a safety factor of 2× on storage estimates for indexes, replication, and unexpected growth. Re-estimate as traffic patterns become clearer.

**Q16. Why is it important to distinguish between read QPS and write QPS in system design?**
> Read-heavy systems (high read QPS) are optimized with caching, read replicas, and CDNs. Write-heavy systems (high write QPS) require robust write paths, write-optimized storage (LSM trees), and sharding. Mixing them without distinction leads to choosing wrong databases or inadequate caching strategies.

**Q17. How do you estimate peak QPS from average QPS?**
> Peak QPS is typically 2–5× average QPS for most consumer applications, and up to 10× for seasonal or event-driven systems (e.g., Black Friday, World Cup). Always design for peak. Use average × 3 as a common rule of thumb when you don't have traffic data.

**Q18. What is the significance of 2^10, 2^20, 2^30 in back-of-the-envelope calculations?**
> 2^10 ≈ 1,000 (Kilo), 2^20 ≈ 1,000,000 (Mega), 2^30 ≈ 1,000,000,000 (Giga). These allow rapid mental math. For example, 1 billion items at 1 byte each = 1 GB; 1 million items at 1 KB each = 1 GB. Mastering these powers speeds up all storage and bandwidth calculations.

**Q19. How much data can a single Kafka partition handle per second?**
> A single Kafka partition can typically handle ~10–50 MB/s for writes and similar for reads. For messages at 1 KB average, that's 10,000–50,000 messages/second per partition. This informs how many partitions to create based on your target throughput.

**Q20. Estimate the number of hard drives required to store 1 PB of data.**
> At 10 TB per HDD (modern drives), you need 1 PB / 10 TB = 100 drives for raw storage. With 3× replication for durability: 300 drives. Add overhead for RAID parity (~20%): ~360 drives. This maps to roughly 6–8 standard server racks.

**Q21. How do you estimate the network bandwidth between services in a system?**
> (Message size in bytes) × (messages per second) = bytes per second. Convert to bits (×8) for bandwidth in bps. Example: 10,000 RPS × 10 KB average response = 100 MB/s = 800 Mbps. A standard 1 Gbps NIC can handle this; for 10× peak, use a 10 Gbps NIC.

**Q22. What simplifications are acceptable in back-of-the-envelope calculations?**
> Round liberally (e.g., 86,400 ≈ 100,000 for a day). Ignore constants unless they are order-of-magnitude different. Use 80/20 rule freely. Don't worry about exact numbers — the goal is to determine if something requires 1 server, 10 servers, or 10,000 servers. Communicating your assumptions clearly is more important than precision.

**Q23. Estimate the storage for a chat application like WhatsApp with 2 billion users.**
> Assume 50M messages/day average. Message = 100 bytes text + 30 bytes metadata = 130 bytes. 50M × 130 bytes = 6.5 GB/day text. Media (assumed 10% of messages, 100 KB avg): 5M × 100 KB = 500 GB/day. Over 10 years: ~1.8 PB text + 1.8 EB media (media dominates). Most systems keep only recent messages in hot storage and archive the rest.

**Q24. How do you estimate the number of machines needed for a distributed cache?**
> Total data to cache / per-machine memory capacity. E.g., 100 GB hot working set, each cache node has 32 GB RAM → 100/32 ≈ 4 nodes minimum. Add 30-40% headroom for fragmentation and overhead → 6 nodes. Add at least 1 replica per node for availability → 12 total nodes.

**Q25. How would you estimate the CDN cost for a video streaming service?**
> Assume 1M concurrent viewers × 5 Mbps = 5 Tbps CDN bandwidth. CDN typically charges ~$0.01–$0.05 per GB. 5 Tbps = 5,000 Gbps; over one hour = 5,000 GB × 3,600s / 8 = 2.25 PB/hour bandwidth out → roughly $22,500–$112,500/hour at those rates. This demonstrates why CDN cost optimization (compression, bitrate adaptation) is critical.

**Q26. How do you estimate the size of a database index?**
> For a B-tree index on an integer column (8 bytes) over 100M rows: each B-tree node holds ~100 entries → tree height ≈ log₁₀₀(100M) ≈ 3–4 levels. Storage: 100M × (8 bytes key + 8 bytes pointer) = 1.6 GB. Index is much smaller than the table but still significant — avoid indexing every column.

**Q27. What reliability (number of 9s) corresponds to different downtime durations?**
> 99.9% (three 9s) = 8.76 hours/year downtime; 99.99% (four 9s) = 52.6 min/year; 99.999% (five 9s) = 5.26 min/year; 99.9999% (six 9s) = 31.5 sec/year. Most consumer services target 99.99%. Achieving each additional 9 requires disproportionately more engineering effort and cost.

