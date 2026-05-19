# System Design: Distributed Email Service (Beginner-Friendly Guide)

---

## What Are We Building?

Gmail. Or something like it.

When you send an email to someone on a different domain (e.g., Gmail → Outlook), there''s a surprisingly complex global infrastructure that:
1. Routes your email from Gmail''s servers to Microsoft''s servers
2. Verifies your identity so you''re not spoofing someone else
3. Checks the email for spam and viruses
4. Delivers it to the recipient''s inbox
5. Notifies the recipient in real-time (without them refreshing the page)

**Scale:** 1.8 billion Gmail users, 400 million Outlook users. ~100,000 emails sent per second globally.

---

## Step 1: Design Scope

**Features:**
- Send and receive emails (across domains)
- Fetch emails (inbox, folders)
- Filter emails (spam protection)
- Search emails (full-text)
- Support email attachments

**Non-functional requirements:**
- **Reliability** — never lose an email
- **Availability** — replicated; survives partial failures
- **Scalability** — 1 billion users
- **Flexibility** — HTTP API (not locked into SMTP/IMAP protocol details)

**Scale estimates:**
| Metric | Estimate |
|--------|---------|
| Users | 1 billion |
| Emails sent per day | 10 per user = 10 billion/day |
| Email sending QPS | ~100,000/sec |
| Email metadata storage | 40 emails/user/day × 50KB × 365 days = **730 PB/year** |
| Attachment storage | 20% of emails × 500KB avg = **1,460 PB/year** |

---

## Step 2: Email 101 — Protocols You Need to Know

### Email Protocols

| Protocol | Role |
|----------|------|
| **SMTP** | Sending email — client → mail server, and server → server |
| **IMAP** | Reading email — keeps emails on server; syncs across devices |
| **POP3** | Reading email — downloads and deletes from server (legacy) |
| **HTTPS** | Modern web clients use HTTP REST APIs instead of IMAP/POP |

> **Our design uses HTTP** — it''s more flexible than IMAP for modern web apps and easier to extend with new features.

### MX Records — How Email Routing Works

<img src="../../system-design-notes/23. Distributed Email Service/images/dns-lookup.png" alt="DNS Lookup MX Record" width="500">

When Gmail sends an email to `bob@outlook.com`, it needs to know where Microsoft''s mail servers are.

The answer is in **DNS MX (Mail Exchanger) records**:
```bash
nslookup -q=mx gmail.com
# gmail.com  mail exchanger = 5   gmail-smtp-in.l.google.com.
# gmail.com  mail exchanger = 10  alt1.gmail-smtp-in.l.google.com.
# gmail.com  mail exchanger = 20  alt2.gmail-smtp-in.l.google.com.
```

Lower number = higher priority. The sender''s SMTP server looks up the recipient domain''s MX records → connects to the highest-priority server → delivers the email.

> **Analogy:** MX records are like a business''s mailing address in a phone book. You look up the company name to find where to send mail.

### How Traditional Email Works

<img src="../../system-design-notes/23. Distributed Email Service/images/traditional-mail-server.png" alt="Traditional Mail Server" width="500">

```
Alice (Outlook) → SMTP → Outlook server
                       → DNS lookup for gmail.com MX records
                       → SMTP → Gmail server
                                → Bob fetches via IMAP
```

### Traditional Email Storage (Maildir)

<img src="../../system-design-notes/23. Distributed Email Service/images/local-dir-storage.png" alt="Local Directory Storage" width="450">

Old mail servers stored each email as a separate file on the local filesystem in a `Maildir` structure:
```
/home/user1/Maildir/
  new/   ← just arrived, not yet read
  cur/   ← read emails
  tmp/   ← emails being written
```

**Problem at scale:** Disk I/O becomes a bottleneck. Server crashes = data loss. Can''t scale horizontally. No high availability.

---

## Step 3: High-Level Architecture

<img src="../../system-design-notes/23. Distributed Email Service/images/high-level-architecture.png" alt="High Level Architecture" width="500">

```
[Webmail (Browser)]
       ↓ HTTPS          ↓ WebSocket
[Web Servers]    [Real-time Servers]
       ↓                 ↓
  ┌─────────────────────────────────┐
  │           Storage Layer         │
  │  Metadata DB  │ Search Store    │
  │  Attachment   │ Distributed     │
  │  Store (S3)   │ Cache (Redis)   │
  └─────────────────────────────────┘
```

| Component | Responsibility |
|-----------|---------------|
| **Web Servers** | HTTP request handling — send/fetch/delete email, manage folders |
| **Real-time Servers** | WebSocket connections for push notifications ("you have new mail") |
| **Metadata DB** | Stores email metadata: subject, from, to, body, timestamps |
| **Attachment Store** | Object storage (S3/GCS) for large binary files |
| **Distributed Cache** | Redis caches recent emails for fast inbox loading |
| **Search Store** | Elasticsearch for full-text search across email content |

---

## Step 4: Email Sending Flow

<img src="../../system-design-notes/23. Distributed Email Service/images/email-sending-flow.png" alt="Email Sending Flow" width="500">

**Step by step:**
1. User writes email in browser → clicks Send → HTTPS request to Load Balancer
2. Load balancer rate-limits (prevents spam bursts) → routes to a Web Server
3. Web server validates email (size limit, valid recipient addresses)
   - **Same domain** (alice@gmail.com → bob@gmail.com): skip SMTP, deliver internally → check spam first
   - **Different domain**: send externally via SMTP
4. Valid email → **Outgoing Queue** (Kafka); Invalid → **Error Queue**
5. **SMTP Outgoing workers** pull from queue → run spam check + virus scan
6. Email stored in sender''s "Sent" folder → SMTP connection to recipient''s mail server → deliver

> **Why Kafka queue between web server and SMTP worker?** If the recipient''s mail server is temporarily down, the message stays in the queue for retry (with exponential backoff). Without the queue, the user would get an immediate error and the email would be lost.

---

## Step 5: Email Receiving Flow

<img src="../../system-design-notes/23. Distributed Email Service/images/email-receiving-flkow.png" alt="Email Receiving Flow" width="500">

**Step by step:**
1. External email arrives → Load Balancer → SMTP Server cluster
2. SMTP server applies **acceptance policy**: invalid recipients → reject immediately (don''t even accept the email)
3. Large attachments → stored in Object Store (S3); email references the object URL
4. Email goes to **Incoming Email Queue**
5. **Mail processing workers** run:
   - **Spam check** — ML classifier scores spam probability
   - **Virus scan** — inspect attachments for malware
6. Clean email → stored in all 4 storage systems (metadata DB, search index, attachment store, cache)
7. **Real-time servers** notified → push WebSocket message to recipient''s open browser tab: "You have 1 new email"
8. If user is offline → they''ll see the new email next time they open their inbox (fetched via HTTP API)

---

## Step 6: API Design

```
POST /v1/messages              ← send an email
GET  /v1/folders               ← list all folders (Inbox, Sent, Trash...)
GET  /v1/folders/{id}/messages ← list emails in a folder (paginated)
GET  /v1/messages/{id}         ← get full email content
DELETE /v1/messages/{id}       ← delete email
```

**Example: list all folders**
```json
[
  { "id": "inbox",  "name": "Inbox",  "unread_count": 3 },
  { "id": "sent",   "name": "Sent",   "unread_count": 0 },
  { "id": "trash",  "name": "Trash",  "unread_count": 0 }
]
```

**Example: get email**
```json
{
  "user_id": "user123",
  "from":    { "name": "Alice", "email": "alice@gmail.com" },
  "to":      [{ "name": "Bob", "email": "bob@outlook.com" }],
  "subject": "Project Update",
  "body":    "Hi Bob, here''s the latest...",
  "is_read": false
}
```

---

## Step 7: Metadata Database Design

### Why Not Relational (SQL)?

At 1 billion users × 40 emails/day × 50KB = 730 PB/year, we need:
- **High write throughput** (emails arriving continuously)
- **Fast per-user range queries** (show inbox sorted by newest first)
- **Horizontal scaling** (can''t fit everything on one SQL server)

SQL databases can''t scale horizontally easily. We need a distributed wide-column store (like **Cassandra** or Google BigTable).

### Schema — Legend

<img src="../../system-design-notes/23. Distributed Email Service/images/legend.png" alt="Key Legend" width="300">

```
K  = Partition Key (determines which node stores this row)
C↓ = Clustering Key descending (sorts rows newest-first within a partition)
```

### Table 1: folders_by_user

<img src="../../system-design-notes/23. Distributed Email Service/images/folders-table.png" alt="Folders Table" width="300">

```
folders_by_user
  user_id   UUID  (K)   ← partition key: all folders for one user on same node
  folder_id UUID        ← unique folder identifier
  folder_name TEXT      ← "Inbox", "Sent", "Trash", or custom name
```

Query: `SELECT * FROM folders_by_user WHERE user_id = ?`

### Table 2: emails_by_user

<img src="../../system-design-notes/23. Distributed Email Service/images/emails-table.png" alt="Emails Table" width="350">

```
emails_by_user
  user_id     UUID      (K)    ← partition key
  email_id    TIMEUUID  (C↓)   ← clustering key, newest first (time-embedded UUID)
  from        TEXT
  to          LIST<TEXT>
  subject     TEXT
  body        TEXT
  attachments LIST<filename|size>
```

> **Why TIMEUUID for email_id?** A TIMEUUID embeds a timestamp into a UUID. This gives us: (1) unique IDs globally, (2) natural sort order by time (newest first), and (3) the creation time encoded without a separate column.

### Table 3: attachments

<img src="../../system-design-notes/23. Distributed Email Service/images/attachments.png" alt="Attachments Table" width="350">

```
attachments
  email_id  TIMEUUID  (C)  ← which email
  filename  TEXT      (K)  ← unique within email
  url       TEXT           ← S3/GCS object URL
```

Attachments stored in **object storage (S3)** — cheap, durable, high-throughput for large binary files. The Cassandra table just stores the metadata + URL reference.

### Handling Read/Unread Status

**Problem:** In Cassandra, you can''t filter on non-partition/clustering key columns. So `WHERE is_read = false` is not allowed efficiently.

**Naive solution:** Fetch all emails, filter in application memory. Works for small mailboxes, terrible for 50,000 emails.

**Real solution: Denormalize into two tables**

<img src="../../system-design-notes/23. Distributed Email Service/images/read-unread-emails.png" alt="Read Unread Emails" width="500">

```
unread_emails (user_id, folder_id, email_id, from, subject, preview)
read_emails   (user_id, folder_id, email_id, from, subject, preview)
```

When a user reads an email:
1. Delete from `unread_emails`
2. Insert into `read_emails`

Counting unread emails = `SELECT COUNT(*) FROM unread_emails WHERE user_id=? AND folder_id=?` — fast, partition-local query.

> **Cassandra rule of thumb:** Design tables around query patterns, not around normalized data models. Denormalization is expected and necessary.

### emails_by_folder (for folder view)

```
emails_by_folder
  user_id   UUID      (K)
  folder_id UUID      (K)    ← composite partition key
  email_id  TIMEUUID  (C↓)   ← newest first
  from      TEXT
  subject   TEXT
  preview   TEXT      ← first 200 chars of body
  is_read   BOOLEAN
```

> **Why store `preview` here?** When showing the inbox, we don''t need the full body. Storing a 200-character preview in the folder table means the inbox list query fetches one table — no joins, no extra lookups.

---

## Step 8: Email Search

### Email Search vs. Web Search

|  | Google Web Search | Email Search |
|--|-------------------|-------------|
| Scope | Entire internet | User''s own mailbox |
| Sorting | By relevance | By date, or other attributes |
| Accuracy | Can lag (indexing delay OK) | Immediate — email appears in search right after receipt |
| Privacy | Public content | Private per-user — user X can never see user Y''s emails |

### Elasticsearch for Email Search

<img src="../../system-design-notes/23. Distributed Email Service/images/elasticsearch.png" alt="Elasticsearch" width="500">

```
[Send/Receive/Delete/Read email]
         ↓ (async via Kafka)
    [Kafka]
         ↓
[Kafka Consumers]
         ↓
[Elasticsearch Cluster]

[Search query] → (sync) → [RESTful API] → [Elasticsearch Cluster]
```

**Why async (Kafka) for indexing, sync for search?**
- Indexing is slow (Elasticsearch must parse, tokenize, build inverted index) — if done synchronously it would slow down sending/receiving
- Searching is real-time — users expect instant results

**Partitioning:** Elasticsearch shards by `user_id` → all a user''s emails on the same node → search queries are partition-local, not cross-cluster

**Supports:**
- Full-text search (`"invoice from Amazon"`)
- Filter queries (`has:attachment`, `from:boss@company.com`)
- Date ranges (`before:2026-01-01`)
- Boolean combinations

### Alternative: Custom Search with LSM Trees

<img src="../../system-design-notes/23. Distributed Email Service/images/lsm-tree.png" alt="LSM Tree" width="500">

If Elasticsearch doesn''t scale far enough, build a custom search engine using **Log-Structured Merge (LSM) Trees** — the same technique used by Cassandra, BigTable, and RocksDB.

**How LSM trees work:**
```
New emails → Level 0 (in-memory buffer)  ← very fast writes
           → when full: flush to Level 1 (disk, many small files)
           → merge: compact Level 1 into Level 2 (fewer, larger files)
           → merge: compact Level 2 into Level 3 (even larger)
           → ...
```

All writes are **sequential** (appends only) → maximizes disk throughput. Reads may need to check multiple levels, but bloom filters speed this up.

**Trade-offs:**

| Approach | Pros | Cons |
|----------|------|------|
| Elasticsearch | Off-the-shelf; full-featured; fast to deploy | Separate system to maintain; scaling limits |
| Custom LSM | Fine-tuned for email; can be the primary DB | Months of engineering effort to build correctly |

---

## Step 9: Email Deliverability — Getting Emails to the Inbox

This is a real-world operational challenge. Setting up an email server is easy. Getting emails into the recipient''s inbox (and not spam) is hard.

### Authentication Standards

| Standard | What it does |
|----------|-------------|
| **SPF** (Sender Policy Framework) | DNS record listing which IPs are allowed to send email for your domain. Receiving server checks: "did this IP have permission?" |
| **DKIM** (DomainKeys Identified Mail) | Sender''s server cryptographically signs each email. Receiver verifies the signature. Proves email wasn''t tampered with. |
| **DMARC** | Policy for what to do when SPF/DKIM fail: `reject`, `quarantine`, or `none`. Also requests reporting. |

Together: prevent email spoofing and phishing ("from: ceo@yourcompany.com" sent by a hacker).

### IP Reputation Best Practices

- **Dedicated IPs** — shared IPs mean one spammer hurts everyone on that IP
- **IP warm-up** — gradually increase sending volume over 2–6 weeks to build reputation with ISPs
- **Classify sending traffic** — transactional emails (password reset) on separate IPs from marketing emails
- **Process bounces immediately** — hard bounce (invalid address) = never email that address again
- **Handle complaints** — register with ISP feedback loops; ban accounts that get too many spam complaints
- **Exponential backoff on delivery failures** — retry after 30min, 1hr, 4hr, 24hr... give up after 5 days (RFC 5321)

---

## Step 10: Conversation Threading

When Bob replies to Alice''s email, and Alice replies to that, they form a thread. How do we model this?

**Email headers carry threading metadata:**
```json
{
  "Message-ID":  "<7BA04B2A@gmail.com>",
  "In-Reply-To": "<ABC123@outlook.com>",
  "References":  ["<ABC123@outlook.com>", "<DEF456@outlook.com>"]
}
```

- `Message-ID` — unique identifier for this email
- `In-Reply-To` — the `Message-ID` of the email being replied to
- `References` — full chain of `Message-IDs` in the thread

**Thread ID generation:**
```
thread_id = SHA256(sorted list of all Message-IDs in the thread)
```

Store: `thread_id → [email_id_1, email_id_2, email_id_3]` ordered by timestamp. When loading the conversation view, fetch the full thread from this mapping.

---

## Step 11: Scalability and High Availability

### Multi-Datacenter Setup

<img src="../../system-design-notes/23. Distributed Email Service/images/multi-dc-example.png" alt="Multi DC Example" width="500">

**Active-Passive multi-region:**
- Primary region (US) handles all traffic
- Secondary region (EU) mirrors all data via async replication
- If US datacenter goes down: DNS MX records point to EU (TTL = 5 minutes → full failover in <5 min)

> **Why not active-active?** Email consistency is critical — we don''t want two datacenters accepting the same email and both storing it. Active-passive keeps one source of truth.

**Consistency choice:** We trade **availability for consistency** — during a failover or network partition, email operations may be briefly unavailable. But when they work, data is correct. (No phantom duplicates, no lost emails from conflicting writes.)

### Component Scaling

| Component | Scaling Strategy |
|-----------|----------------|
| Web Servers | Stateless → horizontal scaling behind load balancer |
| Real-time Servers | Stateful (WebSocket connections) → consistent hash routing |
| SMTP Workers | Stateless → add more consumers to Kafka consumer groups |
| Metadata DB (Cassandra) | Consistent hashing → add nodes, data auto-rebalances |
| Search (Elasticsearch) | Add more nodes; re-shard user data across new nodes |
| Attachment Store | S3 is infinitely scalable by design |
| Cache (Redis) | Cluster mode with consistent hashing |

### Attachment Deduplication

If 10,000 people forward the same 10MB PDF:
- Naive: store 10MB × 10,000 = 100 GB
- Smart: store once, reference by hash

```
SHA256(file bytes) → content-addressed key in object storage
attachments table: { sha256_hash → S3_url, reference_count }
email references: attachment_hash = "abc123..."
```

When user deletes email: `reference_count - 1`. Delete from S3 only when `reference_count = 0`.

---

## Summary: The "Why" Behind Each Decision

| Decision | Why? |
|----------|------|
| HTTP/REST API (not IMAP/POP) | Flexibility; easier to extend; works naturally with web clients |
| Kafka queues for send/receive | Decouples receipt from processing; enables retry; absorbs traffic spikes |
| Cassandra for metadata | Write-heavy (100K emails/sec); scales horizontally; fast per-user partition queries |
| TIMEUUID for email_id | Globally unique + time-sortable + no extra timestamp column |
| Two tables for read/unread | Cassandra can''t filter on non-key columns; denormalization solves it |
| S3 for attachments | Cheap, durable, designed for large binary objects |
| Elasticsearch for search | Full-text search; filter queries; shard by user_id for partition-local search |
| Async indexing via Kafka | Indexing is slow; don''t block email receipt; search results are eventually consistent |
| LSM trees (alternative) | Sequential writes maximize disk throughput for write-heavy email indexing |
| WebSocket for real-time push | Server needs to notify client of new email unprompted; HTTP polling is wasteful |
| Multi-DC active-passive | Email must never be lost; prefer brief unavailability over inconsistent data |
| SPF/DKIM/DMARC | Prove email authenticity; prevent spoofing; ensure inbox delivery |

---

## Quick Interview Q&A Cheat Sheet

**Q: How does email routing work between different providers (Gmail → Outlook)?**  
A: Sender''s SMTP server does a DNS MX record lookup for the recipient''s domain. MX records return a list of mail server hostnames with priorities. Sender connects to the highest-priority server via SMTP over TLS → delivers the email. The receiving server stores it → recipient fetches via IMAP/HTTP.

**Q: Why Cassandra for email storage instead of SQL?**  
A: 730 PB/year of email metadata can''t fit on one SQL server. Cassandra scales horizontally with consistent hashing. Email access patterns (fetch user''s emails sorted by time) map perfectly to Cassandra''s partition key + clustering key model. No complex joins needed. Write-heavy workload handled by LSM-tree storage internals.

**Q: How do you support unread email counts efficiently in Cassandra?**  
A: Denormalize into two separate tables: `read_emails` and `unread_emails`. When user reads an email: delete from `unread_emails`, insert into `read_emails`. Unread count = `COUNT(*)` on `unread_emails` partition — fast, local to one node. Cassandra doesn''t support filtering on non-partition columns, so denormalization is the standard pattern.

**Q: How do you implement email search?**  
A: Elasticsearch cluster, sharded by `user_id`. Each incoming email is asynchronously indexed via Kafka (so email receipt isn''t blocked by indexing). Users search against Elasticsearch synchronously. Supports full-text, filters (`from:boss`, `has:attachment`), date ranges. Alternative: custom LSM-tree-based search engine for maximum scale.

**Q: How do you notify users of new emails in real-time?**  
A: WebSocket connection between browser and Real-time Servers. When mail processing workers finish storing a new email, they notify the Real-time Server via Kafka/pub-sub → Real-time Server pushes a WebSocket message to the user''s browser tab. Offline users see new emails on next page load via HTTP API.

**Q: What is SPF/DKIM/DMARC and why does it matter?**  
A: SPF: DNS record listing authorized sending IPs for your domain — receiving server verifies the sender''s IP is in the list. DKIM: sender cryptographically signs the email; receiver verifies with public key from DNS — proves email wasn''t tampered with. DMARC: policy for what to do when SPF/DKIM fail (reject, quarantine, allow). Together they prevent spoofing, phishing, and ensure emails land in inbox rather than spam.

**Q: How do you prevent losing an email if the receiving mail server is temporarily down?**  
A: Queue-based retry. Incoming email → Kafka outgoing queue. SMTP workers consume with exponential backoff retry: 30min → 1hr → 4hr → 24hr. Give up after 5 days (RFC 5321). If the queue itself is down, MX DNS points to multiple SMTP servers — sender retries against another MX server. Message replicated across Kafka replicas → survives single broker failure.

**Q: How do you scale to 1 billion users?**  
A: Partition every data store by `user_id`. Cassandra: consistent hashing distributes users across nodes. Elasticsearch: route by `user_id` so each user''s search index is shard-local. SMTP workers: Kafka consumer groups, scale horizontally. Real-time servers: consistent hash routing so all of one user''s browser sessions connect to the same server. Add nodes to each layer independently.
