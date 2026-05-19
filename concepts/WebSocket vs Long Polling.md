# WebSocket vs Long Polling

## The Core Problem

HTTP is a **request-response** protocol — the client asks, the server answers. The server can't proactively send data to the client.

But many applications need the server to **push data to the client**:
- Chat: new message arrives → notify recipient
- Uber: driver location updates every second
- Stock ticker: price changes in real-time
- Notifications: someone liked your post

Three main approaches to solve this: Long Polling, Server-Sent Events, and WebSocket.

---

## Approach 1: Short Polling (Naive)

Client asks the server repeatedly on a timer.

```
Client → GET /messages?after=123   (every 1 second)
Server → 200 { messages: [] }      (usually empty)
Client → GET /messages?after=123   (1 second later)
Server → 200 { messages: [] }      (still empty)
Client → GET /messages?after=123   (1 second later)
Server → 200 { messages: [newMsg] } (finally something!)
```

**Problems:**
- Most requests return empty responses (wasted)
- High server load even with no new data
- Latency = up to 1 polling interval (1 second delay to receive a message)

**Verdict:** Only for very low-frequency updates or prototype code.

---

## Approach 2: Long Polling

Client sends a request; server **holds it open** until data is available or timeout.

```
Client → GET /messages?after=123   (HTTP request)
         ... server holds connection open ...
         ... 28 seconds pass, no new messages ...
         ... new message arrives at 30s ...
Server → 200 { messages: [newMsg] } (responds immediately with data)
Client → GET /messages?after=124   (immediately sends next long poll)
```

**How it works:**
1. Client sends request
2. Server checks for new data
3. If no data: server holds the connection open (sleeping/waiting, not blocking a thread in modern async frameworks)
4. When data arrives OR timeout reached → server responds
5. Client immediately sends a new long-poll request

**Pros:**
- Near real-time (latency = time until server responds, typically <1s)
- Works over standard HTTP — no special infrastructure needed
- Passes through corporate firewalls/proxies

**Cons:**
- Each "push" requires a new HTTP request (HTTP headers overhead)
- Server must hold many open connections
- Not truly bidirectional (client sends data via separate POST requests)

**Best for:** Simple chat, notifications when WebSocket isn't available, corporate environments with strict firewall rules.

---

## Approach 3: Server-Sent Events (SSE)

Client opens one HTTP connection; server streams data **one-way** to the client.

```
Client → GET /events  (single long-lived HTTP request)
Server → 200 (keeps connection open, streams data)
         data: {"type": "message", "text": "Hello"}
         
         data: {"type": "notification", "count": 3}
         
         data: {"type": "typing", "user": "Alice"}
```

**Pros:**
- Simple — native browser support (`EventSource` API)
- Automatic reconnection built-in
- Works over HTTP/2 (multiplexed)

**Cons:**
- **One-way only** — server → client (client can't send data through the SSE connection)
- Limited to text/UTF-8 (no binary data)

**Best for:** News feeds, dashboards, notifications, stock tickers — anything where server pushes updates but client doesn't need to send back data on the same channel.

```javascript
// Browser-side
const source = new EventSource('/events');
source.onmessage = (event) => {
  console.log(JSON.parse(event.data));
};
```

---

## Approach 4: WebSocket ⭐ (Best for Real-Time Bidirectional)

A **persistent, full-duplex TCP connection** between client and server. Either side can send data at any time.

### WebSocket Handshake
Starts as an HTTP upgrade request:

```
Client → HTTP Upgrade Request:
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade

Server → 101 Switching Protocols
Connection now upgraded to WebSocket
```

After the handshake: raw TCP connection with WebSocket framing (not HTTP anymore).

### Full-Duplex Communication
```
Client ←──────────────────→ Server
           WebSocket
         (persistent TCP)

Client sends: "Hello"         → Server receives instantly
Server sends: "New message!"  → Client receives instantly
```

**Pros:**
- Full-duplex — both sides send data freely
- Very low overhead — no HTTP headers per message (just WebSocket frames)
- Low latency — no new connection per message
- Supports binary data

**Cons:**
- Stateful — server must track which socket belongs to which user
- Doesn't work through some corporate proxies/firewalls
- Load balancers must support sticky sessions or WebSocket routing
- More complex than HTTP (connection management, heartbeat/ping, reconnection logic)

**Best for:** Chat, multiplayer games, collaborative editing (Google Docs), live trading, Uber driver tracking.

---

## Side-by-Side Comparison

| | Short Polling | Long Polling | SSE | WebSocket |
|--|--------------|-------------|-----|-----------|
| **Direction** | Client→Server | Server→Client (pull) | Server→Client | **Both** |
| **Connection** | New per request | New per message | One persistent | One persistent |
| **Latency** | Up to interval | Low (~ms) | Low (~ms) | Lowest (~ms) |
| **Overhead** | High (empty responses) | Medium (HTTP headers) | Low | Lowest (frames) |
| **Binary support** | No | No | No | ✅ Yes |
| **Firewall friendly** | ✅ Yes | ✅ Yes | ✅ Yes | Sometimes No |
| **Complexity** | Very Low | Low | Low | High |
| **Server state** | Stateless | Stateless | Stateless | **Stateful** |

---

## WebSocket in System Design

### The Statefulness Problem

WebSocket connections are tied to a specific server instance. If you have 10 server instances, a user connected to Server 3 can't directly receive a message sent to Server 7.

```
User A (socket on Server 1) ← message from User B (socket on Server 4)?
```

**Solution: Pub/Sub with Redis or Kafka**

```
User B sends message
    ↓
Server 4 receives it
    ↓
Server 4 publishes to Redis channel "user_A_messages"
    ↓
Server 1 is subscribed to "user_A_messages"
    ↓
Server 1 pushes via WebSocket to User A
```

Every WebSocket server subscribes to Redis channels for its connected users. Cross-server delivery works via pub/sub.

### Load Balancing WebSockets

WebSocket connections are long-lived. Once established, they must stay with the same server.

**Solutions:**
1. **Sticky sessions (IP hash)** — same client IP always → same server. Simple but uneven on NAT.
2. **Consistent hash routing** — hash on user ID → always routes to the same server node.
3. **Gateway + routing layer** — a WebSocket gateway maps user ID → server node.

---

## When to Use What

| Use Case | Recommended |
|----------|------------|
| Chat (WhatsApp, Slack) | **WebSocket** |
| Live location tracking (Uber) | **WebSocket** |
| Multiplayer game | **WebSocket** |
| Collaborative editing (Google Docs) | **WebSocket** |
| News feed / social notifications | **SSE** or Long Polling |
| Stock ticker / dashboard | **SSE** or WebSocket |
| Simple notification count | **Long Polling** |
| Corporate-restricted environment | **Long Polling** |

**Rule of thumb:**
- Need client → server communication in real-time? → WebSocket
- Server just pushes updates, no client data on same channel? → SSE
- Simple, low-frequency, firewall-constrained? → Long Polling

---

## Interview Q&A

**Q: What is the difference between WebSocket and HTTP?**  
A: HTTP is request-response — client initiates every exchange. WebSocket starts with an HTTP upgrade handshake, then converts to a persistent full-duplex TCP connection. Either side can send data at any time without a new request. No HTTP headers per message — just lightweight WebSocket frames. Much lower overhead and latency for bidirectional real-time communication.

**Q: What is long polling and when would you use it?**  
A: Client sends an HTTP request; server holds it open until data is available or timeout, then responds. Client immediately sends another request. Simulates server push over regular HTTP. Use it when WebSocket isn't available (corporate firewalls, old infrastructure), for low-to-medium frequency updates, or when server-push simplicity matters more than performance.

**Q: What is the main architectural challenge with WebSockets at scale?**  
A: Statefulness — each WebSocket connection is tied to a specific server. When User A (on Server 1) needs to receive a message sent by User B (on Server 4), cross-server delivery is needed. Solution: each WebSocket server subscribes to a pub/sub system (Redis, Kafka). When Server 4 gets a message for User A, it publishes to a Redis channel; Server 1 (which holds User A's socket) is subscribed and delivers it.

**Q: What is the difference between SSE and WebSocket?**  
A: SSE is one-way (server → client only), uses regular HTTP, has automatic reconnection, and is simpler to implement. WebSocket is full-duplex (both directions), requires a TCP upgrade, has lower overhead per message, and supports binary data. Use SSE when you only need server-to-client updates. Use WebSocket when clients also need to send data in real-time (chat, games, collaborative editing).
