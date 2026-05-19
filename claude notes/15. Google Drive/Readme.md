# System Design: Google Drive (Beginner-Friendly Guide)

---

## What is Google Drive?

Google Drive is a **cloud file storage and sync service** — you upload a file from your laptop, and it magically appears on your phone too. Edit it from the phone, and the laptop gets the update.

**Real-world examples:**
- Google Drive
- Dropbox
- OneDrive (Microsoft)
- iCloud Drive (Apple)

> **Why is this hard to design?**  
> It's not just "store a file." You need files to sync across multiple devices in near real-time, handle conflicts when two people edit simultaneously, save revision history, allow sharing with permissions, and do all of this reliably at petabyte scale — without ever losing a single byte.

---

## Step 1: Understand What We're Building

### Functional Requirements

| Feature | Detail |
|---------|--------|
| Upload & Download | Any file type, up to 10 GB |
| Multi-device sync | Changes on one device appear on all others |
| File sharing | Share with view/edit permissions |
| Revision history | Restore older versions of a file |
| Notifications | Alert when a shared file is edited or deleted |

### Non-Functional Requirements

| Requirement | Why it matters |
|-------------|----------------|
| **Reliability** | Data loss is completely unacceptable |
| **Fast sync** | Users get frustrated waiting for files to sync |
| **Bandwidth efficiency** | Don't re-upload a whole file for a tiny edit |
| **High availability** | Works even during server failures |
| **Scale** | 10 million DAU, 500 PB total storage |

### Scale Assumptions

- Users get **10 GB free storage**
- Average file: **500 KB**
- Each user uploads **2 files/day**
- Total storage needed: **500 PB**

> **500 PB = 500,000,000 GB.** You cannot store this on a single server or even a single data center. Distributed storage is mandatory.

---

## Step 2: Starting Simple — Single Server Design

Before scaling, understand the basics.

### The Namespace Concept

<img src="../../system-design-notes/15. Google Drive/images/namespaces.png" alt="Namespaces" width="400">

Files are organized under a root `drive/` directory. Each user gets their own **namespace** (folder):

```
drive/
  user1/
    recipes/
      chicken_soup.txt
  user2/
    football.mov
    sports.txt
  user3/
    best_pic_ever.png
```

Every file is uniquely identified by: **namespace + relative path**
- e.g., `user1/recipes/chicken_soup.txt`

### Basic APIs

**Upload a file:**
```
POST https://api.example.com/files/upload?uploadType=resumable
```

Two types:
- **Simple upload** — small files, one shot
- **Resumable upload** — large files; if connection drops, resume from where it stopped (don't restart from scratch)

**Download a file:**
```
GET https://api.example.com/files/download
```

**Get revision history:**
```
GET https://api.example.com/files/list_revisions
```

> **Question:** Why does upload need a "resumable" mode?  
> **Answer:** A 10 GB file on a shaky connection could take hours. If it fails at 90%, you'd lose everything and restart. Resumable upload tracks which parts succeeded, so a retry only uploads the missing pieces.

---

## Step 3: Moving to Distributed Systems

The single server fails at scale. Three key improvements:

### 1. Amazon S3 for Storage + Replication

<img src="../../system-design-notes/15. Google Drive/images/replication.png" alt="Replication" width="500">

Instead of storing files on your own server, use **S3 (object storage)**:

| Strategy | How it works |
|----------|-------------|
| **Same-region replication** | Copy stored across 3 availability zones in the same region |
| **Cross-region replication** | Copy stored in a completely different geographic region (e.g., US + EU) |

> **Question:** Why replicate across regions?  
> **Answer:** A natural disaster (flood, fire, earthquake) can wipe out an entire data center — or an entire region. Cross-region replication ensures your files survive even if one AWS region goes completely offline. This is how Google achieves "11 nines" (99.999999999%) durability.

### 2. Sharding by User ID

Split the metadata database across multiple servers based on `user_id`. User 1's data goes to shard A; User 2's to shard B. No single database handles all queries.

### 3. Load Balancer

Distributes incoming requests across multiple web/API servers. If one server crashes, the load balancer routes to healthy ones.

---

## Step 4: Handling Sync Conflicts

<img src="../../system-design-notes/15. Google Drive/images/sync-conflicts.png" alt="Sync Conflicts" width="500">

> **Scenario:** User 1 and User 2 both edit the same shared file at the same time. Who wins?

**Rule: First processed version wins.**

```
User 1 saves → processed first → ACCEPTED → becomes "latest version"
User 2 saves → arrives second → CONFLICT DETECTED

System creates a conflict copy:
"report.docx (User 2's conflicted copy 2026-05-08)"

User 2 is shown both versions and can:
- Keep their version
- Keep User 1's version
- Merge manually
```

> **Analogy:** Like two people editing the same Google Doc when offline. When they reconnect, the system shows both versions and lets the person resolve it.

---

## Step 5: Improved High-Level Design

<img src="../../system-design-notes/15. Google Drive/images/high-level-design.png" alt="High Level Design" width="500">

```
User (Browser / Mobile App)
        |
   [Load Balancer]
        |
   [API Servers]          ← Auth, profile, metadata management
        |
   ┌────┴──────────────────────────────────┐
   |              |              |         |
[Block       [Metadata     [Notification  [Cold
 Servers]     DB+Cache]     Service]     Storage]
   |
[Cloud Storage (S3)]
```

### What Each Component Does

**Block Servers:**
> Files are split into **4 MB chunks** (blocks). Each block gets a unique hash value and is stored independently in cloud storage. To reconstruct a file, the blocks are joined back together in sequence.

**Cloud Storage (S3):**
> Stores all the actual file chunks. Cheap, durable, infinitely scalable.

**Cold Storage:**
> Files that haven't been accessed in a long time (e.g., 6 months) are moved to cheaper "cold storage" (like Amazon Glacier). Reading from cold storage is slower but costs a fraction of regular storage.

**Metadata Database + Cache:**
> Stores everything *about* files — names, paths, sizes, versions, permissions, block references. The actual file content lives in S3; the metadata lives here.

**Notification Service:**
> A **publisher/subscriber** system. When a file changes, the notification service tells all relevant devices: "Hey, there's an update!" Clients then pull the changes.

**Offline Backup Queue:**
> If a device is offline when a change happens, the change event is queued here. When the device reconnects, it processes the queue and syncs.

---

## Step 6: Block Servers — The Key to Efficiency

This is the most important innovation in the design.

<img src="../../system-design-notes/15. Google Drive/images/delta-sync.png" alt="Delta Sync" width="400">

### Why Split Files into Blocks?

> **Question:** Why not just upload/download the entire file each time?  
> **Answer:** Imagine you have a 1 GB Word document. You fix one typo. Without blocks, you'd re-upload the entire 1 GB. With 4 MB blocks, you only re-upload the one 4 MB block that changed. That's **250x less data transferred**.

This technique is called **Delta Sync** — only sync what changed.

### How Block Servers Process a File

<img src="../../system-design-notes/15. Google Drive/images/file-sync.png" alt="File Sync" width="400">

```
File comes in
    ↓
Split into blocks (Block 1, Block 2, ... Block N)
    ↓
Compress each block (reduce size)
    ↓
Encrypt each block (security)
    ↓
Upload to Cloud Storage (S3)
```

Each block gets a **hash (SHA-256)**. The hash is like a fingerprint — if two blocks have the same hash, they contain identical data.

**On file update:** Compare new block hashes with old block hashes:
```
Block 1: old hash = abc123 | new hash = abc123  → UNCHANGED, skip upload
Block 2: old hash = def456 | new hash = xyz789  → CHANGED, upload new Block 2
Block 3: old hash = ghi789 | new hash = ghi789  → UNCHANGED, skip upload
```

Only Block 2 gets uploaded. **Massive bandwidth savings.**

> **Analogy:** Imagine a bookshelf where each shelf is numbered. You only need to replace the shelf where you added a new book — not rebuild the entire bookshelf.

### Deduplication

If two different users upload the exact same file (e.g., a popular software installer), block deduplication means the file is only stored **once** in S3, even though it shows up in both users' drives. Same hash = same block = same storage object.

---

## Step 7: Metadata Database — What Gets Stored Where

<img src="../../system-design-notes/15. Google Drive/images/metadata-database.png" alt="Metadata Database" width="500">

### Tables

**User Table**
```
user_id | user_name | created_at
```

**Workspace Table** (each user's Drive space)
```
id | owner_id | is_shared | created_at
```

**File Table**
```
id | file_name | relative_path | is_directory | latest_version | checksum | workspace_id | created_at | last_modified
```

**File Version Table** (revision history)
```
id | file_id | device_id | version_number | last_modified
```

**Block Table** (which chunks make up a file version)
```
block_id | file_version_id | block_order
```

**Device Table** (for multi-device sync)
```
device_id | user_id | last_logged_in_at
```

> **Question:** Why is there a separate `block` table instead of just storing the file?  
> **Answer:** A file is assembled from many blocks, and the same block can be referenced by multiple file versions. Version 5 and Version 6 of a file may share 9 out of 10 blocks. The block table allows this reuse — you don't need to store 9 identical blocks twice. This is content-addressable storage.

> **Question:** Why track `device_id` in file versions?  
> **Answer:** So the system knows which device made which change. If you have 3 devices and your phone made a change while offline, the system can identify that the phone's version hasn't been uploaded yet when it reconnects.

---

## Step 8: File Upload Flow — Exactly What Happens

<img src="../../system-design-notes/15. Google Drive/images/upload-flow.png" alt="Upload Flow" width="500">

Two things happen simultaneously when you upload a file:

### Track 1: File Content
```
① Client splits file into blocks
② Client sends blocks to Block Servers
③ Block Servers compress + encrypt each block
④ Blocks uploaded to Cloud Storage (S3)
⑤ Block Servers notify API Servers: "upload complete"
⑥ API Servers update metadata status: pending → uploaded
```

### Track 2: File Metadata (runs in parallel)
```
① Client sends file metadata to API Servers
   (name, path, size, block hashes, etc.)
② API Servers save metadata with status = "pending"
③ Notification Service tells other devices: "new file incoming..."
```

### When Both Tracks Complete
```
④ Status updated to "uploaded"
⑤ Notification Service tells Client 2 (another device):
   "File is ready — sync now"
⑥ Client 2 downloads and syncs
```

> **Question:** Why is the status first "pending" then "uploaded"?  
> **Answer:** The file takes time to upload and process. Other devices shouldn't try to download a half-uploaded file. "Pending" means "a file is coming but not ready yet." "Uploaded" means "safe to download."

---

## Step 9: File Download Flow — How Sync Works

<img src="../../system-design-notes/15. Google Drive/images/download-flow.png" alt="Download Flow" width="550">

When someone edits a file on Device A, Device B (or Device C) needs to sync.

### Two Scenarios

**Scenario 1: Device B is ONLINE when the change happens**
```
① Notification Service: "Hey Device B, file X was updated"
② Device B: "Give me the updated metadata" → API Servers
③ API Servers query Metadata DB → return what changed
④ Device B: "Give me the changed blocks" → Block Servers
⑤ Block Servers fetch from Cloud Storage → return blocks
⑥ Device B assembles blocks → file is now synced
```

**Scenario 2: Device B is OFFLINE when the change happens**
```
① Change event saved to Offline Backup Queue
   (holds it until Device B comes back online)
② Device B reconnects
③ Device B sees queued events → triggers the same flow as above
④ Device B syncs all missed changes in order
```

> **Question:** Why use Long Polling for the notification service instead of WebSocket?  
> **Answer:** Google Drive doesn't need true real-time sync (like a chat app). Files syncing within a few seconds is acceptable. Long polling is simpler to implement and scale than WebSocket. The client holds a connection open; when a change occurs, the server responds immediately. The client then re-connects for the next update.

---

## Step 10: Storage Optimization Strategies

### De-duplication

Use the block hash (SHA-256) to detect identical blocks:
```
User A uploads presentation.pptx → Block 3 hash = abc123
User B uploads same presentation.pptx → Block 3 hash = abc123

System: "I already have block abc123 — don't store again"
```

Both users see the file in their drives, but S3 stores it only once.

### Version Limits

Unlimited revision history = unlimited storage cost. Practical limits:
- Keep only the last **100 versions**
- For frequently edited files, keep versions from the **last 30 days**
- Users can "permanently delete" a version to free space

### Cold Storage for Inactive Files

```
File not accessed for 6+ months → automatically moved to S3 Glacier

S3 Glacier costs: ~$0.004/GB/month (vs $0.023/GB/month for standard S3)
That's ~6x cheaper for files nobody is accessing
```

> **Trade-off:** Retrieving from cold storage takes minutes instead of milliseconds. Acceptable for "archived" files, not for actively used ones.

---

## Step 11: Failure Handling

| Failure scenario | Recovery strategy |
|-----------------|------------------|
| Load Balancer fails | Secondary load balancer automatically activates |
| Block Server crashes | Other block servers pick up its pending upload tasks |
| Metadata DB master fails | Promote a replica to master; redirect write traffic |
| Cloud Storage unavailable | Fetch from a different region's replica (cross-region replication) |
| Notification Service crash | Clients reconnect to alternative notification servers; queue holds missed events |

> **Question:** What if a client loses connection mid-block-upload?  
> **Answer:** Resumable uploads. The client and server track which blocks were successfully uploaded. On reconnect, the client only uploads the blocks that weren't confirmed received. The already-uploaded blocks are not re-sent.

---

## Step 12: Full Architecture Summary

```
                    User (Browser / Mobile)
                           |
                    [Load Balancer]
                           |
                    [API Servers]
                    - Auth + Sessions
                    - File metadata CRUD
                    - Sharing + permissions
                           |
         __________________|_______________________________
        |                  |                   |           |
 [Block Servers]    [Metadata Cache]    [Metadata DB]  [Notification
  - split files      (Redis)            (MySQL/         Service]
  - compress                             Postgres)      - long polling
  - encrypt                                             - offline queue
  - delta sync
        |
 [Cloud Storage (S3)]
  - blocks stored by hash
  - same + cross-region replication
        |
 [Cold Storage (Glacier)]
  - inactive files
  - 6x cheaper
```

---

## Summary: The "Why" Behind Each Decision

| Decision | Why? |
|----------|------|
| Split files into 4 MB blocks | Delta sync — only upload changed blocks; massive bandwidth savings |
| Hash each block (SHA-256) | Detect unchanged blocks (skip upload); enable deduplication |
| Compress + encrypt blocks | Reduce storage size; protect user data in transit and at rest |
| Separate metadata DB from file storage | Different scaling needs; SQL for structured queries, S3 for blobs |
| Resumable uploads | Large files fail mid-transfer; resume from checkpoint instead of restarting |
| Long polling for notifications | Simple, scalable; files don't need millisecond sync like chat |
| Offline backup queue | Changes don't get lost when a device is offline |
| Cross-region S3 replication | Data survives regional disasters; legal compliance for some regions |
| Cold storage for inactive files | 6x cost reduction for files nobody is accessing |
| Block deduplication | Same block uploaded by 1,000 users = stored once; massive storage savings |
| File version table | Revision history without duplicating identical blocks across versions |

---

## Quick Interview Q&A Cheat Sheet

**Q: How does file sync across devices work?**  
A: Client detects a file change → splits into blocks → uploads only changed blocks to Block Servers → Block Servers store in S3 → Notification Service tells other devices → they request updated metadata → download changed blocks → reassemble file.

**Q: What is delta sync and why is it important?**  
A: Only upload/download the file blocks that changed. A 1 GB file with a 1 KB edit only transfers 4 MB (one block), not 1 GB. Detected by comparing SHA-256 hashes of blocks before and after the change.

**Q: How do you handle two users editing the same file simultaneously?**  
A: First processed version wins and becomes the new "latest." The second user gets a conflict copy ("file (conflicted copy)"). The user can then merge manually or choose one version. No edit is silently lost.

**Q: What database do you use for metadata vs. file content?**  
A: Metadata (names, paths, permissions, block references) → relational DB (MySQL/PostgreSQL). File content (blocks) → object storage (S3). They scale independently; S3 is orders of magnitude cheaper per GB than DB storage.

**Q: How do you ensure data never gets lost (11 nines durability)?**  
A: S3 replicates every block across 3+ availability zones (same-region) and optionally to a second region (cross-region replication). Periodic CRC checksums detect silent corruption and trigger re-replication.

**Q: How does offline sync work?**  
A: Changes made while a device is offline are queued in the Offline Backup Queue. When the device reconnects, it processes the queue in order — downloading all missed changes sequentially. Local changes made offline are uploaded and conflict-checked on reconnect.

**Q: How do you save storage costs at petabyte scale?**  
A: (1) Block deduplication — identical blocks stored once, referenced by many; (2) Compression — each block compressed before storage; (3) Version limits — keep only N recent versions; (4) Cold storage — inactive files moved to Glacier at 6x lower cost.
