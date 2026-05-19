# System Design: S3-like Object Storage (Beginner-Friendly Guide)

---

## What Are We Building?

**Amazon S3** (Simple Storage Service) — or something that works just like it.

S3 is where you put files that need to live forever in the cloud:
- A photo you upload to Instagram
- A video you upload to YouTube (before processing)
- A backup of your database
- The source code bundle your CI/CD system deploys
- Static assets (images, CSS, JS) for a website

> **Analogy:** Object storage is like a giant warehouse with infinite shelves. You label each box (object) with a unique name, throw it on a shelf, and can retrieve it any time using that label. No folders, no hierarchy — just labels and boxes.

**Scale we need to handle:**
- 100 petabytes of data
- 99.9999% data durability (six nines — lose 1 file per million per year)
- 99.99% availability (four nines — ~52 minutes of downtime per year)

---

## Step 1: Three Types of Storage — Which Is Which?

<img src="../../system-design-notes/24. S3-like Object Storage/images/storage-comparison.png" alt="Storage Comparison" width="500">

| | Block Storage | File Storage | Object Storage |
|--|--------------|--------------|----------------|
| **What it is** | Raw disk blocks (HDD/SSD) | Folders + files (like your laptop) | Flat key → file mapping |
| **Mutable?** | Yes — overwrite anytime | Yes — edit in place | No — immutable (versioned instead) |
| **Cost** | High | Medium–High | Low |
| **Performance** | Very high | Medium–High | Low–Medium |
| **Access** | SAS/iSCSI (block protocol) | NFS/SMB (network file share) | HTTP REST API |
| **Best for** | VMs, databases | Shared file access | Backups, media, data lakes |

> **Q: Why is object storage the cheapest?**  
> A: Because it sacrifices random-write performance. It''s write-once, read-many — no in-place edits, no directory traversal overhead. Simpler internals = cheaper at massive scale.

---

## Step 2: Key Terminology

| Term | Meaning |
|------|---------|
| **Bucket** | A logical container for objects. Name is globally unique (like a domain name). |
| **Object** | One piece of data stored in a bucket: file contents + metadata (content-type, size, custom tags). |
| **Versioning** | Keep multiple versions of an object in the same bucket (each PUT creates a new version). |
| **URI** | Every resource has a unique URL: `https://bucket.s3.example.com/path/to/file.jpg` |
| **SLA** | Contract guaranteeing durability/availability levels. S3 Standard: 99.999999999% durability. |

---

## Step 3: Design Philosophy — The UNIX Inode Analogy

<img src="../../system-design-notes/24. S3-like Object Storage/images/object-store-vs-unix.png" alt="Object Store vs UNIX" width="500">

On a UNIX system, when you save a file:
1. The OS creates an **inode** — a metadata structure storing the filename, size, permissions, and **pointers to where the actual data blocks live on disk**
2. The actual file contents are stored separately, at those disk locations

Object storage works the same way:
- **Metadata store** = inode (stores object name, size, content-type, which data node holds it)
- **Data store** = disk blocks (stores the actual bytes)

> **Why separate them?**  
> You can scale each independently. A hot bucket with tiny files needs more metadata capacity. A bucket storing huge videos needs more data capacity. Separating them lets you right-size each.

<img src="../../system-design-notes/24. S3-like Object Storage/images/bucket-and-object.png" alt="Bucket and Object Structure" width="500">

---

## Step 4: High-Level Architecture

<img src="../../system-design-notes/24. S3-like Object Storage/images/high-level-design.png" alt="High Level Design" width="500">

| Component | Responsibility |
|-----------|---------------|
| **Load Balancer** | Distributes API requests across stateless API servers |
| **API Service** | Stateless orchestrator — validates requests, calls IAM, routes to metadata/data stores |
| **IAM Service** | Identity & Access Management — authentication, authorization, access control |
| **Metadata Store** | Stores object metadata: name, size, bucket, data node UUID, version |
| **Data Store** | Stores the actual raw bytes of every object |

**Key object properties:**
- **Immutable** — objects are never updated in place; new versions are created
- **Key-value** — the object''s URI is its key; HTTP GET returns the value (file bytes)
- **Write once, read many** — LinkedIn research shows ~95% of operations are reads

---

## Step 5: Uploading an Object

<img src="../../system-design-notes/24. S3-like Object Storage/images/uploading-object.png" alt="Uploading Object" width="500">

**Example request:**
```http
PUT /bucket-to-share/script.txt HTTP/1.1
Host: foo.s3example.org
Content-Type: text/plain
Content-Length: 4567

[4567 bytes of file content]
```

**Step by step:**
1. User sends `PUT /bucket-to-share` → API service creates the bucket
2. IAM verifies user has write permission to this bucket
3. Metadata store records the new bucket → success response returned
4. User sends `PUT /bucket-to-share/script.txt` with file contents
5. IAM verifies write permission
6. API service streams object data to **Data Store** → Data Store persists it, returns a **UUID**
7. API service saves metadata to **Metadata Store**: `{object_name: "script.txt", bucket: "bucket-to-share", object_id: UUID}`
8. Success response returned to client

> **Q: Why does the data store return a UUID, not the original filename?**  
> A: The data store is a low-level blob engine — it doesn''t know about bucket names or object keys. Keeping it dumb makes it simpler and faster. The metadata store is the one that maps "script.txt in bucket-to-share" → UUID. This separation makes both layers independently scalable.

---

## Step 6: Downloading an Object

<img src="../../system-design-notes/24. S3-like Object Storage/images/download-object.png" alt="Download Object" width="500">

**Example request:**
```http
GET /bucket-to-share/script.txt HTTP/1.1
Host: foo.s3example.org
```

**Step by step:**
1. Client sends `GET /bucket-to-share/script.txt` to Load Balancer
2. API service asks IAM: "Does this user have read permission on this bucket?"
3. API service queries Metadata Store: "What is the UUID for script.txt in bucket-to-share?"
4. API service retrieves the object bytes from Data Store using the UUID → streams to client

---

## Step 7: Data Store Deep Dive

### Components of the Data Store

<img src="../../system-design-notes/24. S3-like Object Storage/images/data-store-main-components.png" alt="Data Store Main Components" width="500">

| Component | Role |
|-----------|------|
| **Data Routing Service** | RESTful/gRPC API layer; routes reads/writes to correct data nodes; stateless |
| **Placement Service** | Decides which data node should store a new object; maintains virtual cluster map |
| **Data Nodes** | Physical servers with HDDs/SSDs storing the actual object bytes |

<img src="../../system-design-notes/24. S3-like Object Storage/images/data-store-interactions.png" alt="Data Store Interactions" width="500">

### Virtual Cluster Map

<img src="../../system-design-notes/24. S3-like Object Storage/images/virtual-cluster-map.png" alt="Virtual Cluster Map" width="500">

The Placement Service maintains a **virtual cluster map** — a live picture of all data nodes, which drives they manage, and how much space is left on each.

Data nodes send **heartbeats** to the Placement Service:
- "I have 4 drives managing 40TB total"
- "Drive 2 is at 87% capacity"
- "I''m still alive" (no heartbeat for 30s → node marked dead, data re-replicated)

> **Why does the Placement Service need to be highly available?**  
> Without it, no writes can proceed (we don''t know where to store objects). So it runs as a 5–7 node cluster using Raft/Paxos consensus. A 7-node cluster tolerates 3 nodes failing simultaneously.

### Data Persistence Flow

<img src="../../system-design-notes/24. S3-like Object Storage/images/data-persistence-flow.png" alt="Data Persistence Flow" width="500">

1. API Service sends object data to Data Routing Service
2. Data Routing Service asks Placement Service: "Which node is the primary for this object?"
3. Data Routing Service streams data to the **Primary Data Node**
4. Primary saves locally + replicates to 2 **Secondary Data Nodes** (synchronously)
5. After all 3 confirm: UUID returned to API Service

> **Q: Why wait for all 3 replicas before returning success?**  
> A: Strong consistency. If we returned after 1 write and then the node crashed, the data would be lost. Waiting for 3 guarantees durability before the client thinks the upload succeeded. The cost is slightly higher latency — but for storage, durability > speed.

<img src="../../system-design-notes/24. S3-like Object Storage/images/consistency-vs-latency.png" alt="Consistency vs Latency" width="500">

### How Objects Are Stored on Disk

**Naive approach:** One file per object.

**Problem:** With millions of small objects (< 1MB):
- HDD block size is 4KB → a 1KB file wastes 3KB of disk space
- Too many inodes → OS slows down; there''s a hard inode limit
- Millions of tiny files = catastrophic fragmentation

**Solution: Merge many small files into large WAL files**

<img src="../../system-design-notes/24. S3-like Object Storage/images/wal-optimization.png" alt="WAL Optimization" width="500">

```
[Object A bytes] [Object B bytes] [Object C bytes] [Object D bytes] → /data/volume_001
                                                                      (grows to ~1GB, then sealed)
[Object E bytes] [Object F bytes] ...                               → /data/volume_002
```

Objects are **appended sequentially** to the current active volume file. Sequential writes are dramatically faster than random writes on spinning disks (100–150x faster).

**Object lookup:** To find where an object is within a volume file, each data node maintains a local SQLite database:

```
object_mapping
  object_id    UUID    ← the object''s UUID
  filename     TEXT    ← which volume file (e.g., "/data/volume_001")
  file_offset  INT     ← byte position within that file
  object_size  INT     ← how many bytes to read
```

> **Q: Why SQLite on the data node instead of a shared cluster database?**  
> A: Data nodes only care about their own objects. A shared cluster database would add network round-trips on every read/write. SQLite is embedded — zero network latency. Since the data node owns the data, it should own the index too.

### Updated Data Persistence Flow

<img src="../../system-design-notes/24. S3-like Object Storage/images/updated-data-persistence-flow.png" alt="Updated Data Persistence Flow" width="500">

1. API Service sends data to Data Routing Service
2. Data node appends object bytes to current volume file (e.g., `/data/volume_003`)
3. Data node inserts record into local SQLite `object_mapping` table
4. UUID returned up the chain

---

## Step 8: Durability — Surviving Failures

### Failure Domain Isolation

<img src="../../system-design-notes/24. S3-like Object Storage/images/failure-domain-isolation.png" alt="Failure Domain Isolation" width="500">

A power outage can take out an entire rack. A network switch failure can isolate an entire cabinet. If all 3 replicas are on the same rack, one event destroys all copies.

**Solution:** Replicate across failure domains:
- Replica 1 → Rack A
- Replica 2 → Rack B
- Replica 3 → different Data Center

With HDD annual failure rate of 0.81%, 3 cross-rack replicas achieves **6 nines of durability**.

### Erasure Coding — More Durable, Less Storage

<img src="../../system-design-notes/24. S3-like Object Storage/images/erasure-coding.png" alt="Erasure Coding" width="500">

Replication is simple but wastes space: 3 replicas = **200% storage overhead**.

**Erasure coding** (like RAID, but for distributed systems) uses math to rebuild lost data from parity:

```
Object → split into 8 data shards + 4 parity shards (8+4 erasure coding)
         → store all 12 shards on different nodes
         → can reconstruct the full object from ANY 8 of the 12 shards
```

<img src="../../system-design-notes/24. S3-like Object Storage/images/erasure-coding-across-failure-domains.png" alt="Erasure Coding Across Failure Domains" width="500">

With 8+4 across different failure domains:

<img src="../../system-design-notes/24. S3-like Object Storage/images/erasure-coding-vs-replication.png" alt="Erasure Coding vs Replication" width="500">

| | Replication (3×) | Erasure Coding (8+4) |
|--|-----------------|---------------------|
| Storage overhead | 200% | 50% |
| Durability | 6 nines | **11 nines** |
| Read latency | Low (1 node) | Higher (fetch from 8 nodes) |
| CPU cost | Low | Higher (encode/decode) |
| Complexity | Simple | Hard to implement |

> **Bottom line:** Use replication for latency-sensitive hot data. Use erasure coding for cold/archival data where storage cost matters more than speed. S3 uses both depending on the storage tier.

### Checksums — Detecting Silent Corruption

<img src="../../system-design-notes/24. S3-like Object Storage/images/checksums-for-correctness.png" alt="Checksums for Correctness" width="500">

A disk can silently corrupt data — returning wrong bytes without any error flag. This is called **bit rot**.

**Solution:** Store a checksum (MD5 or SHA256) alongside every object and every shard:
- On write: compute `checksum = SHA256(bytes)`, store both data and checksum
- On read: recompute `SHA256(bytes)` and compare to stored checksum
- Mismatch → corruption detected → fetch from a healthy replica

With erasure coding (8+4): verify all 8 data shard checksums, reconstruct corrupted shards from parity.

---

## Step 9: Metadata Store Design

<img src="../../system-design-notes/24. S3-like Object Storage/images/metadata-data-model.png" alt="Metadata Data Model" width="500">

**Tables:**

```sql
-- Small table: users create at most a few hundred buckets
buckets (bucket_id, owner_id, bucket_name, region, created_at)

-- Large table: billions of objects
objects (bucket_id, object_name, object_version, object_id, is_delete_marker, ...)
```

**Queries to support:**
- Find an object by name → `WHERE bucket_id=? AND object_name=?`
- Insert/delete an object by name
- List objects in a bucket sharing a prefix (simulates folder navigation)

### Sharding Strategy

The `buckets` table is small (users create few buckets) → fits on one server, replicated for read throughput.

The `objects` table has billions of rows → must be sharded.

| Sharding Strategy | Problem |
|-------------------|---------|
| `hash(bucket_id)` | All objects in a huge bucket on one shard → **hotspot** |
| `hash(object_name)` | Can''t do bucket-scoped queries efficiently |
| **`hash(bucket_name + object_name)`** ✅ | Objects distributed evenly; most queries use both bucket+name |

### Listing Objects (The Hard Part)

When you do `list objects in bucket-X with prefix "images/"`, the query needs to find all matching objects — but they''re spread across many shards.

**Solution 1:** Scatter-gather — run the query on every shard, merge results in memory. Slow but works.

**Solution 2:** Create a **denormalized listing table** sharded by `bucket_id`:
```
bucket_objects_index (bucket_id, object_name)
```
All objects in one bucket stay on the same shard → fast listing. Write to this table in addition to the main objects table (eventual consistency OK for listing).

> Object storage is not optimized for listing. S3''s official docs say: "Don''t rely on LIST for real-time inventory at scale — use S3 Inventory instead."

---

## Step 10: Object Versioning

<img src="../../system-design-notes/24. S3-like Object Storage/images/object-versioning.png" alt="Object Versioning" width="500">

With versioning enabled, every `PUT` to the same key creates a **new version** (new `object_id`) rather than overwriting:

```
PUT /bucket/photo.jpg → version_id=v1, object_id=UUID_A
PUT /bucket/photo.jpg → version_id=v2, object_id=UUID_B  (v1 still exists)
PUT /bucket/photo.jpg → version_id=v3, object_id=UUID_C  (v1, v2 still exist)

GET /bucket/photo.jpg           → returns v3 (latest)
GET /bucket/photo.jpg?ver=v1    → returns v1 (specific version)
```

`version_id` is a **TIMEUUID** — embeds a timestamp → natural chronological sort order.

### Deleting a Versioned Object

<img src="../../system-design-notes/24. S3-like Object Storage/images/deleting-versioned-object.png" alt="Deleting Versioned Object" width="500">

`DELETE /bucket/photo.jpg` doesn''t actually delete anything. It creates a special **delete marker** entry with a new version:

```
version_id=v4, is_delete_marker=true
```

`GET /bucket/photo.jpg` → sees delete marker → returns **404 Not Found** (but old versions still exist).

To truly delete: `DELETE /bucket/photo.jpg?versionId=v4` (removes the delete marker) or `DELETE /bucket/photo.jpg?versionId=v1` (removes a specific old version permanently).

> **Why delete markers instead of actual deletes?**  
> Protection against accidental deletion. Also allows "undelete" by removing the delete marker. Important for compliance: regulated industries often require immutable audit trails.

---

## Step 11: Multipart Upload (Large Files)

<img src="../../system-design-notes/24. S3-like Object Storage/images/multipart-upload.png" alt="Multipart Upload" width="500">

Uploading a 50GB video file as one HTTP request is risky: any network glitch → restart from zero.

**Multipart upload splits it into chunks:**

```
1. POST /initiate-multipart?key=video.mp4
   → Server returns: upload_id = "abc123"

2. PUT /upload-part?upload_id=abc123&part=1  (bytes 0–5MB)
   → Server returns: ETag = "checksum_of_part1"

   PUT /upload-part?upload_id=abc123&part=2  (bytes 5MB–10MB)   ← can run in parallel!
   → Server returns: ETag = "checksum_of_part2"

   ... (up to 10,000 parts)

3. POST /complete-multipart
   Body: { upload_id: "abc123", parts: [{1, ETag1}, {2, ETag2}, ...] }
   → Server reassembles parts → returns final object URI
```

**Benefits:**
- Resume from failed part (only retry the failed chunk, not the whole file)
- Parallel part uploads → much faster for large files
- Required for objects > 5GB in S3

**Minimum part size:** 5MB (except the last part). Prevents abuse with millions of 1-byte parts.

---

## Step 12: Garbage Collection

Over time, storage fills with **dead data**:
- Deleted objects that were never physically removed
- Failed multipart upload parts left behind
- Old versions from versioned buckets
- Corrupted data flagged by checksum verification

**Compaction** physically reclaims this space:

<img src="../../system-design-notes/24. S3-like Object Storage/images/compaction.png" alt="Compaction" width="500">

1. Background GC process reads through volume file `data/b`
2. Copies only the **live** objects (not deleted, not corrupted) to a new volume file `data/d`
3. Updates the `object_mapping` SQLite table to point to new offsets (atomic transaction)
4. Deletes the old volume file `data/b`

> **Q: Why not delete objects immediately?**  
> A: **Lazy deletion** is safer. Marking an object deleted first, then physically removing it later, prevents race conditions where a read request arrives mid-delete. Compaction batches up many deletions into one efficient sequential write.

**With erasure coding (8+4):** Garbage collector must delete all 12 shards (8 data + 4 parity) across different nodes when an object is collected.

---

## Step 13: Back-of-the-Envelope Estimation

| Metric | Value |
|--------|-------|
| Total storage target | 100 PB = 10¹¹ MB |
| Storage utilization | 40% |
| Object size mix | 20% small (<1MB), 60% medium (1–64MB), 20% large (>64MB) |
| Median sizes | 0.5MB small, 32MB medium, 200MB large |
| Total objects | ~680 million |
| Metadata per object | 1 KB |
| Total metadata storage | ~0.68 TB (fits in a few DB servers) |
| HDD IOPS | 100–150 random seeks/sec |

---

## Summary: The "Why" Behind Each Decision

| Decision | Why? |
|----------|------|
| Separate metadata store from data store | Independent scaling; metadata access pattern (many tiny lookups) ≠ data access pattern (large sequential reads/writes) |
| Objects are immutable | Simplifies replication (no update propagation); enables versioning; avoids locking |
| Write-ahead log (WAL) volumes | Sequential appends are much faster than random writes on HDD; avoids inode explosion from millions of tiny files |
| SQLite embedded in data nodes | Zero network latency for object lookup; data nodes only need their own index |
| Placement Service as Raft cluster | Critical path — must be highly available; Raft tolerates N/2 node failures |
| Replicate synchronously before ACK | Strong durability guarantee — client''s "success" means data is safe on 3 nodes |
| Erasure coding for cold storage | 50% storage overhead vs 200% for 3× replication; 11 nines vs 6 nines durability |
| Checksums on every object/shard | Detect silent disk corruption (bit rot) without relying on OS error reporting |
| Multipart upload | Resume failed large uploads; parallel chunk uploads; mandatory for >5GB files |
| TIMEUUID for version_id | Globally unique + time-sortable — natural version ordering without extra timestamp column |
| Delete markers (lazy deletion) | Protect against accidental deletion; enable undelete; safe concurrent operations |
| Shard metadata by hash(bucket+key) | Avoids bucket-level hotspots while keeping most queries (bucket+key lookups) shard-local |
| Denormalized listing table | LIST queries are scatter-gather without it; denormalized table by bucket_id makes listing fast |

---

## Quick Interview Q&A Cheat Sheet

**Q: What is the difference between object, block, and file storage?**  
A: Block storage = raw disk blocks (databases, VMs). File storage = hierarchical folder/file tree over a network (NFS/SMB). Object storage = flat key→blob over HTTP (S3, GCS). Object storage is cheapest and most scalable but has no in-place edits and no directory hierarchy. Block storage has the lowest latency. File storage supports shared access by multiple servers.

**Q: How does multipart upload work?**  
A: (1) Client calls `InitiateMultipartUpload` → gets `upload_id`. (2) Client splits file into parts (min 5MB each) and uploads each independently via `UploadPart(upload_id, part_number, bytes)` — can be done in parallel; each returns an ETag checksum. (3) Client calls `CompleteMultipartUpload(upload_id, [{part1, ETag1}, ...])` → server assembles all parts into the final object. Benefit: if part 7 fails, only re-upload part 7, not the whole 50GB file.

**Q: What is erasure coding and when do you use it over replication?**  
A: Erasure coding splits an object into k data shards + m parity shards. Any k of (k+m) shards can reconstruct the full object. Example: 8+4 → 12 shards, can lose any 4. Storage overhead: 12/8 = 1.5× vs 3× for 3-replica replication. Use replication for hot, latency-sensitive data (reads from one node). Use erasure coding for cold/archival storage where durability + cost matter more than read latency.

**Q: How does object versioning work?**  
A: Each PUT to the same key creates a new `version_id` (TIMEUUID) and new `object_id` — old versions are preserved. GET returns the latest version by default; `?versionId=xxx` fetches a specific version. DELETE creates a "delete marker" — doesn''t physically delete; just marks the latest version as deleted. To permanently delete: specify the version_id explicitly in the DELETE request.

**Q: How do you shard the metadata store?**  
A: Shard by `hash(bucket_name + object_name)`. This avoids hotspots (a single huge bucket''s objects spread across all shards) while keeping most queries (which use both bucket+name) shard-local. The `buckets` table is small enough to live on one replicated server. LIST bucket queries require scatter-gather across shards OR a separate denormalized listing table sharded by `bucket_id`.

**Q: How do you ensure data durability at 6 nines?**  
A: (1) Replicate each object to 3 data nodes across different failure domains (different racks/AZs). Synchronous replication — only ACK after all 3 confirm write. (2) Checksums (MD5/SHA256) on every object — detect silent disk corruption on every read. (3) Heartbeats from data nodes to Placement Service — detect dead nodes within 30s → trigger re-replication to a healthy node. With erasure coding (8+4) across failure domains: 11 nines of durability.

**Q: Why do data nodes store objects in large volume files instead of one file per object?**  
A: One file per object causes two problems at scale: (1) Disk block waste — a 1KB file occupies a full 4KB block; (2) Inode exhaustion — OS has a hard limit on inodes; millions of tiny files slow the filesystem. By appending many objects sequentially into large volume files (WAL style), you use disk space efficiently, keep inode count low, and get fast sequential write throughput.

**Q: How does the Placement Service decide where to store an object?**  
A: Placement Service maintains a virtual cluster map: data node → list of drives → current capacity. When a new object arrives: pick a primary node with sufficient free space (avoiding nodes in the same failure domain). Use consistent hashing to deterministically map object UUID → replication group (so the data routing service can find the right nodes for future reads without consulting Placement Service every time).
