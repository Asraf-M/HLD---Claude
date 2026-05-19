# System Design: Chat System (Beginner-Friendly Guide)

---

## What is a Chat System?

A chat system lets users send text messages to each other in real time. You already use them every day:

**Real-world examples:**
- WhatsApp (1:1 and group chat)
- Slack (team channels)
- Facebook Messenger
- Discord (servers with channels)

> **Why is this a common interview question?**  
> Because it covers almost every major concept: real-time protocols (WebSocket), stateful servers, message queues, database design, presence detection, and horizontal scaling.

---

## Step 1: Understand What We're Building

### Requirements

| Feature | Detail |
|---------|--------|
| Chat types | 1-on-1 and group (max 100 members) |
| Message types | Text only (up to 100,000 characters) |
| Online indicators | Show who is online/offline |
| Multi-device support | One account, phone + laptop |
| Push notifications | For offline users |
| Scale | 50 million DAU |
| Storage | Permanent chat history |

### Two key questions to ask first:
1. **How does the sender send a message to the server?**
2. **How does the server push a message to the receiver?**

These two problems have different answers — and getting them right is the whole design.

---

## Step 2: How Does the Client Talk to the Server?

### Option A: Regular HTTP (for sending messages)

<img src="../../system-design-notes/12. Chat System/images/basic-design.png" alt="Basic Design" width="400">

When you type a message and hit send:
```
User A → POST /v1/chat/messages → Chat Server → delivers to User B
```

HTTP works fine for the **sender side** — it's a one-time request. Simple.

> **But what about the receiver?** User B needs to receive the message without refreshing the page. HTTP is **request-driven** — the client always has to ask first. The server can't just push a message to you unprompted. That's the core problem.

---

### Option B (Receiver Side): Polling

<img src="../../system-design-notes/12. Chat System/images/polling.png" alt="Polling" width="400">

```
User B repeatedly asks: "Any new messages?"
Server: "No."
User B (1 second later): "Any new messages?"
Server: "No."
User B (1 second later): "Any new messages?"
Server: "Yes! Here it is."
```

> **Why is polling bad?**  
> The client is asking constantly, even when nothing is happening. At 50M users doing this every second, you'd have billions of useless requests per day hammering your servers. Most are empty responses. Waste of CPU and bandwidth.

---

### Option C (Receiver Side): Long Polling

<img src="../../system-design-notes/12. Chat System/images/long-polling.png" alt="Long Polling" width="400">

Instead of asking and immediately getting an answer, the client holds the connection open:
```
User B asks: "Any new messages?"
Server: [holds the connection open... waiting... waiting...]
New message arrives!
Server: "Yes! Here it is." → closes connection
User B immediately reconnects and waits again
```

Better than polling, but:
- The server still holds open thousands of half-done HTTP connections
- If no message comes, the connection times out anyway
- Not truly real-time

---

### Option D (Best): WebSocket

<img src="../../system-design-notes/12. Chat System/images/websocket.png" alt="WebSocket" width="400">

WebSocket is a **persistent, two-way connection** between client and server. Once established, both sides can send data at any time without the overhead of HTTP headers on every message.

**How it's established:**
```
1. Client sends: GET /ws (HTTP request with "Upgrade: websocket" header)
2. Server responds: "Upgrading to WebSocket"
3. Connection is now open and bidirectional indefinitely
```

After that:
- User A sends message → instantly goes through the WebSocket
- Server pushes reply to User B → instantly through User B's WebSocket
- No repeated requests, no polling

> **Analogy:** Regular HTTP is like sending letters back and forth. WebSocket is like an open phone call — both sides can speak whenever they want.

> **Question:** Why not use WebSocket for everything (including login/signup)?  
> **Answer:** WebSocket is optimized for real-time, persistent communication. For stateless operations like login, user profile updates, or fetching friend lists, regular HTTP + REST APIs are simpler, easier to cache, and easier to load-balance. Use the right tool for the job.

---

## Step 3: High-Level Architecture — Two Parts

The system has two distinct parts: **stateless** services and **stateful** services.

### Stateless Services (Regular HTTP)

<img src="../../system-design-notes/12. Chat System/images/high-level-stateless-arch.png" alt="Stateless Architecture" width="400">

These are your normal microservices behind a load balancer:

| Service | What it does |
|---------|-------------|
| **Authentication Service** | Login, logout, token validation |
| **User Profile Service** | Update name, photo, settings |
| **Group Management** | Create group, add/remove members |
| **Service Discovery** | Find which chat server to connect to |

> These are **stateless** — any server can handle any request. The load balancer routes freely between them. Easy to scale horizontally.

### Stateful Services (WebSocket)

<img src="../../system-design-notes/12. Chat System/images/high-level-statefull-arch.png" alt="Stateful Architecture" width="400">

These maintain **persistent WebSocket connections**:

| Service | What it does |
|---------|-------------|
| **Chat Servers** | Maintain WebSocket connections; deliver messages |
| **Presence Servers** | Track who is online/offline |

> These are **stateful** — User A's WebSocket is tied to a specific chat server. You can't just route to any server. If User A is on Chat Server 1, all of A's messages must go through Server 1.

---

## Step 4: Complete High-Level Design

<img src="../../system-design-notes/12. Chat System/images/high-level-design.png" alt="High Level Design" width="450">

```
User (Web / Mobile)
        |           \
     [HTTP]        [WebSocket]
        |                \
 [Load Balancer]    [Chat Servers] ←→ [Presence Servers]
        |                 |                  |
 [API Servers]      [Message Sync      [KV Store]
   - Auth             Queue]           (online status)
   - Profile               |
   - Groups         [KV Store]
                    (chat history)
        |
 [Notification Servers]
   → APNs / FCM (push notifications)
```

### Why a Key-Value Store (not a SQL DB) for chat history?

> **Question:** Can't we just use MySQL to store messages?  
> **Answer:** At 50M DAU sending many messages, you'd have billions of rows. SQL struggles here because:
> - Indexes grow huge → random access gets slow
> - Sharding SQL is complex
> - The read pattern (give me all messages in conversation X after time T) maps perfectly to KV stores with time-based keys

**Real systems:**
- Facebook Messenger uses **HBase**
- Discord uses **Cassandra**
- WhatsApp uses **Mnesia** (Erlang KV store)

---

## Step 5: Service Discovery — Which Chat Server Should I Connect To?

<img src="../../system-design-notes/12. Chat System/images/zookeeper.png" alt="Zookeeper" width="400">

When you log in, your app needs to know **which WebSocket server to connect to**. You can't just pick one randomly — you need to be routed intelligently based on:
- Which server is least loaded?
- Which server is geographically closest?

**Apache Zookeeper** solves this:

```
1. User A opens app → logs in via HTTP → hits Load Balancer
2. Load Balancer → API Servers
3. API Servers query Zookeeper: "Find me the best chat server"
4. Zookeeper returns: "Chat Server 2"
5. User A opens WebSocket connection to Chat Server 2
```

> **Analogy:** Zookeeper is like the maitre d' at a restaurant. You walk in, they check which tables are free and seat you at the best available one. You don't just walk in and sit anywhere.

---

## Step 6: Message Flow — 1-on-1 Chat (Deep Dive)

<img src="../../system-design-notes/12. Chat System/images/one-to-one-chat-flow.png" alt="One-to-one Chat Flow" width="400">

Here's exactly what happens when User A sends a message to User B:

```
① User A sends message via WebSocket → Chat Server 1
② Chat Server 1 requests a unique message_id from the ID Generator
③ Chat Server 1 stores message in Message Sync Queue (Kafka)
④ Message is persisted in KV Store (chat history)

If User B is ONLINE (5a):
   → Chat Server 1 looks up which server User B is on → Chat Server 2
   → Publishes message to Chat Server 2
   → Chat Server 2 delivers via WebSocket to User B

If User B is OFFLINE (5b):
   → Push notification sent via APNs/FCM
   → User B gets a phone notification
   → On next app open, messages sync from KV Store
```

> **Question:** Why is there an ID Generator? Can't we use timestamps?  
> **Answer:** Two messages can arrive at the exact same millisecond. Timestamps alone don't guarantee uniqueness or order. We use **Snowflake IDs** — 64-bit IDs that embed a timestamp, machine ID, and sequence number. This guarantees both uniqueness AND chronological ordering.

---

## Step 7: Message Flow — Group Chat

<img src="../../system-design-notes/12. Chat System/images/group-chat-flow.png" alt="Group Chat Flow" width="400">

Group chat works differently. When User A sends a message to a group with members B and C:

```
User A → Chat Server 1
              |
    __________|___________
   |                      |
[Message Sync Queue B]  [Message Sync Queue C]
   |                      |
 User B gets it         User C gets it
```

Each group member has their **own inbox (message sync queue)**. When a group message is sent, a copy is placed into each member's queue.

> **Question:** Why give each person their own queue instead of one shared group queue?  
> **Answer:** Each user has different read status, different devices, and different online/offline states. A personal queue makes it easy to track "what has User B seen vs. what has User C seen." It's simpler to reason about per-user delivery.

> **Question:** What about groups with 100 people?  
> **Answer:** Each message gets copied 100 times (one per member's queue). This is fine at small scale. For very large groups (Slack channels with thousands of members), you'd use fanout on read instead — only copy when someone actually requests the messages.

---

## Step 8: Data Models — What Gets Stored in the DB?

### 1-on-1 Chat Table

<img src="../../system-design-notes/12. Chat System/images/one-to-one-chat.png" alt="One to One Chat Schema" width="300">

| Column | Type | Purpose |
|--------|------|---------|
| `message_id` | bigint (PK) | Unique, sortable, time-ordered |
| `message_from` | bigint | Sender's user_id |
| `message_to` | bigint | Recipient's user_id |
| `content` | text | The message text |
| `created_at` | timestamp | When it was sent |

### Group Chat Table

<img src="../../system-design-notes/12. Chat System/images/group-chat.png" alt="Group Chat Schema" width="300">

| Column | Type | Purpose |
|--------|------|---------|
| `channel_id` | bigint (composite PK) | Which group/channel |
| `message_id` | bigint (composite PK) | Unique within this channel |
| `user_id` | bigint | Sender |
| `content` | text | Message text |
| `created_at` | timestamp | When sent |

> **Question:** Why is the group chat primary key `(channel_id, message_id)` and not just `message_id`?  
> **Answer:** This is a **composite primary key**. Cassandra and similar KV stores use the first part (`channel_id`) as the partition key — all messages for a group live on the same node, making range queries fast. The `message_id` within a partition determines sort order. "Give me the last 20 messages for group 42" becomes a single efficient query.

---

## Step 9: Message Synchronization Across Multiple Devices

<img src="../../system-design-notes/12. Chat System/images/message-synchronization.png" alt="Message Synchronization" width="400">

User A has both a phone and a laptop. They read a message on their laptop. How does the phone know which messages are "new"?

Each device tracks **`cur_max_message_id`** — the ID of the latest message it has received.

```
Phone:   cur_max_message_id = 653
Laptop:  cur_max_message_id = 842

Phone reconnects → asks: "Give me all messages with ID > 653"
Server returns messages 654 to 842 → phone catches up
```

A message is "new" for a device if:
1. It belongs to the logged-in user's conversations AND
2. Its `message_id` is greater than the device's `cur_max_message_id`

> **Analogy:** It's like a bookmark in a book. Each device has its own bookmark. When you open the book on a different device, you start reading from where *that device's* bookmark is, not from where you left off on the other device.

---

## Step 10: Online Presence — The Green Dot

How does the app know if someone is online? This is the **presence system**.

### Heartbeat Mechanism

<img src="../../system-design-notes/12. Chat System/images/heartbeat-mechanism.png" alt="Heartbeat Mechanism" width="400">

When you're online, your app sends a **heartbeat** — a small "I'm still here" ping — to the Presence Server every few seconds.

```
Client → heartbeat → Presence Server  (every 5 seconds)
Presence Server stores: {user_id: "A", last_seen: now, status: "online"}

If NO heartbeat for 30 seconds:
→ Presence Server marks User A as "offline"
```

> **Why 30 seconds?** A balance between accuracy (detect offline quickly) and false positives (don't mark someone offline if they just have a bad signal for 2 seconds).

**Stored in Redis** (KV Store):
```
user_A → { status: "online", last_active_at: 1746700000 }
```

Redis is perfect here: sub-millisecond reads, built-in TTL (auto-expire after 30s if no update).

> **Question:** What if the phone loses internet for 10 seconds then comes back?  
> **Answer:** The heartbeat will miss a beat or two, but the 30-second threshold means the user won't be marked offline. When the connection resumes, heartbeats continue and the status stays green.

### Fanout — Telling Friends About Your Status

<img src="../../system-design-notes/12. Chat System/images/fanout-presence.png" alt="Fanout Presence" width="400">

When User A goes online/offline, their friends need to know. This uses **pub/sub channels**:

```
User A goes online
      ↓
Presence Server publishes to channels: A-B, A-C, A-D
      ↓
User B (subscribed to A-B) receives "A is online" → green dot appears
User C (subscribed to A-C) receives "A is online" → green dot appears
User D (subscribed to A-D) receives "A is online" → green dot appears
```

Each friendship pair has a dedicated channel. When A's status changes, all three friends instantly know.

> **Question:** What if User A has 5,000 friends? Do we need 5,000 channels?  
> **Answer:** Yes — and this is why presence at huge scale (Twitter, Facebook) is hard. For a small group chat app (max 100 members), one channel per friendship pair works fine. For massive social networks, presence is only shown for close friends or contacts you've recently talked to.

---

## Step 11: Full Architecture — Putting It All Together

```
                    User (Web / Mobile)
                    /              \
               [HTTP]           [WebSocket]
                /                      \
        [Load Balancer]         [Real-Time Service]
               |               ┌──────────────────┐
        [API Servers]          │  [Chat Servers]  │
        - Auth                 │  [Presence Svrs] │
        - User Profile         └──────────────────┘
        - Group Mgmt                   |
        - Service Discovery     [Message Sync Queue]
               |                 (Kafka per user)
        [Notification               |
         Servers]           ┌───────┴──────────┐
         → APNs/FCM         │   [KV Store]      │
                            │ (chat history,    │
                            │  online status)   │
                            └───────────────────┘
```

### Security: How is data protected?

> **AppKey + AppSecret** for API authentication. For true privacy (like WhatsApp), use **End-to-End Encryption (E2EE)**:
> - Each device generates a public/private key pair
> - Messages are encrypted with the recipient's public key before leaving the sender's device
> - Server stores and routes the encrypted blob — never sees plaintext
> - Only the recipient's private key can decrypt

---

## Step 12: Scaling to 50M DAU

**Back-of-envelope math:**
- 50M DAU × 40 messages/day = **2 billion messages/day** = ~23,000/sec
- Each WebSocket server handles ~500K connections → need **~100 chat servers**
- Each message to a group of 10 = 230K deliveries/sec → fanout workers scale horizontally

### What changes as you scale?

| Component | Scaling strategy |
|-----------|-----------------|
| Chat Servers | Add more; use consistent hashing to route users |
| KV Store (Cassandra) | Add nodes; partition by `chat_id` |
| Presence (Redis) | Cluster mode; shard by `user_id` |
| Message Queue (Kafka) | Add partitions; one partition per group |
| Push Notifications | Already handled by APNs/FCM infrastructure |

> **Question:** How do messages cross between chat servers? If User A is on Server 1 and User B is on Server 2, how does Server 1 deliver to Server 2?  
> **Answer:** **Redis Pub/Sub or Kafka** acts as the message bus between servers. Server 1 publishes the message to a topic for User B. Server 2 (which holds B's WebSocket connection) subscribes to that topic and receives the message, then delivers it to B over the WebSocket.

---

## Summary: The "Why" Behind Each Decision

| Decision | Why? |
|----------|------|
| WebSocket for message delivery | Only protocol that allows true server-push without polling |
| HTTP for stateless APIs | Simpler, cacheable, easier to load-balance than WebSocket |
| KV Store (Cassandra) for messages | Write-heavy, time-range reads, scales horizontally better than SQL |
| Snowflake IDs for message ordering | Unique + chronologically sortable; timestamps alone aren't enough |
| Heartbeat for presence detection | Simple, reliable; no complex connection tracking needed |
| Per-user message sync queue | Individual read cursors; handles multi-device naturally |
| Zookeeper for service discovery | Routes users to optimal chat servers based on load/location |
| Per-friendship pub/sub channels | Real-time presence updates without polling |
| Message Sync Queue (Kafka) | Decouples message receipt from delivery; handles offline users |
| Separate stateful/stateless tiers | Scale each independently; stateless servers are cheap to add |

---

## Quick Interview Q&A Cheat Sheet

**Q: Why WebSocket over HTTP for chat?**  
A: HTTP requires the client to initiate every request. WebSocket is bidirectional — the server can push messages to the client at any time. For real-time chat, you need the server to push, not wait for a client request.

**Q: How do you scale WebSocket servers (they're stateful)?**  
A: Use consistent hashing at the load balancer to route each user to the same server (sticky routing). For cross-server delivery, use Redis Pub/Sub or Kafka as a message bus between servers.

**Q: SQL or NoSQL for chat messages?**  
A: NoSQL (Cassandra/HBase). Partition by `chat_id`, cluster by `message_id`. Write-heavy, time-range read pattern, needs horizontal sharding — all things NoSQL handles better than SQL at this scale.

**Q: How does offline message delivery work?**  
A: Message is stored in KV Store regardless of recipient status. If offline, a push notification (APNs/FCM) is sent. When the user opens the app, they request all messages with `message_id > cur_max_message_id` from their KV store.

**Q: How does the presence (green dot) system work?**  
A: Client sends a heartbeat ping every 5 seconds. If no heartbeat for 30 seconds, the Presence Server marks the user offline in Redis. Friends learn of status changes via pub/sub channels — one channel per friendship pair.

**Q: How do you handle a user on 3 different devices?**  
A: Each device has its own `cur_max_message_id` cursor and its own WebSocket connection. When a message arrives, it's fanned out to all active device connections for that user. Each device independently syncs from its own cursor on reconnect.

**Q: What is `cur_max_message_id`?**  
A: A per-device variable tracking the highest message_id the device has received. On reconnect, the device fetches all messages with ID greater than this value to catch up on missed messages — without redownloading everything.
