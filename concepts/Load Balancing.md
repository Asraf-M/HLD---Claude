# Load Balancing

## What Is It?

A **load balancer** distributes incoming traffic across multiple server instances so that no single server is overwhelmed.

> **Analogy:** A supermarket with 10 checkout lanes. A greeter at the entrance directs each customer to the shortest queue. No single lane gets all the customers.

---

## Why Load Balance?

| Problem (single server) | Solution (load balancer + multiple servers) |
|------------------------|---------------------------------------------|
| Server hits CPU/memory limit | Spread load across 10 servers |
| Server crashes → full outage | Load balancer detects failure, routes around it |
| Deploy new code → downtime | Rolling deploy: take servers out one by one |
| Single geographic location | Route users to nearest data center |

---

## L4 vs L7 Load Balancing

### L4 — Transport Layer (TCP/UDP)

Routes traffic based on **IP address and port** only. Doesn't inspect the request content.

```
Client → L4 LB → routes based on (src_ip, dst_ip, dst_port) → Server
```

**Fast** — no packet inspection, just TCP forwarding.  
**Dumb** — can't route based on URL path, HTTP headers, cookies.

**Examples:** AWS Network Load Balancer (NLB), HAProxy (TCP mode)

**Use when:** Very high throughput, non-HTTP traffic (game servers, streaming, databases)

---

### L7 — Application Layer (HTTP/HTTPS) ⭐ Most Common

Routes based on **HTTP content** — URL path, headers, cookies, method.

```
GET /api/users → routes to User Service
GET /api/orders → routes to Order Service
GET /images/cat.jpg → routes to CDN / Static Server

Cookie: user_id=123 → always routes to same server (sticky session)
```

**Smarter** — can do content-based routing, SSL termination, compression, caching.  
**Slightly slower** — must parse HTTP headers.

**Examples:** AWS Application Load Balancer (ALB), nginx, Cloudflare, Envoy

**Use when:** HTTP/HTTPS traffic, microservices routing, need SSL termination

---

## Load Balancing Algorithms

### 1. Round Robin ⭐
Each request goes to the next server in sequence.

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A  (cycle repeats)
```

**Simple, works well when all servers are equal.**  
**Problem:** Doesn't account for server load — a slow request on Server A still receives the next request on the cycle.

---

### 2. Weighted Round Robin
Servers with more capacity get more requests.

```
Server A (8 cores)  → weight 4 → gets 4 out of every 7 requests
Server B (4 cores)  → weight 2 → gets 2 out of every 7 requests
Server C (2 cores)  → weight 1 → gets 1 out of every 7 requests
```

**Use when:** Heterogeneous server fleet (some are bigger/faster).

---

### 3. Least Connections ⭐
Send to the server with the **fewest active connections** right now.

```
Server A: 100 active connections
Server B: 50 active connections  ← new request goes here
Server C: 75 active connections
```

**Better than round-robin for variable-length requests** (e.g., some requests take 1ms, others 10 seconds).

---

### 4. Least Response Time
Send to the server with the **lowest average response time**.

Combines connection count + actual latency measurement. Most adaptive but requires ongoing latency tracking.

---

### 5. IP Hash (Sticky by IP)
`server = hash(client_ip) % num_servers`

Same client IP always → same server.

**Use for:** Session-based apps where server state must be consistent per user, WebSocket connections.

**Problem:** Uneven distribution if many users share one IP (NAT, corporate proxy).

---

### 6. Random
Pick a random server. Works surprisingly well at large scale (Law of Large Numbers ensures even distribution over time).

---

## Health Checks

Load balancer continuously checks if servers are alive:

**Active health check (most common):**
```
LB → GET /health → Server A
Server A → 200 OK        (healthy, keep sending traffic)
Server A → 503 / timeout (unhealthy, stop sending traffic)
```

**Passive health check:**
LB monitors real traffic — if Server B returns 5xx or times out on N consecutive requests, mark as unhealthy.

**Recovery:** Once health check passes again, server is added back to rotation.

---

## Session Persistence (Sticky Sessions)

**Problem:** User logs in on Server A. Next request goes to Server B (no login state) → user is logged out.

**Solutions:**

**1. Sticky sessions (server-side):**  
LB routes same user to same server (via cookie or IP hash). Simple but breaks on server failure.

**2. Shared session store (better):**  
Store sessions in Redis — all servers read from the same place. Any server can handle any request.

```
Server A: session = redis.get("session:abc123")
Server B: session = redis.get("session:abc123")  ← same data
```

**Recommendation:** Use shared Redis sessions. Don't rely on sticky sessions — they break failover.

---

## SSL Termination

Load balancer handles HTTPS decryption, forwards plain HTTP to backend servers.

```
Client ──HTTPS──→ Load Balancer ──HTTP──→ Server A
                  (SSL terminated here)  Server B
                                         Server C
```

**Benefits:**
- Backend servers don't need SSL certificates or decryption overhead
- LB manages certificate renewal in one place (Let's Encrypt auto-renewal)
- Internal network (LB → Server) stays private and can use HTTP

---

## Horizontal Scaling Pattern

```
                     ┌─────────────┐
                     │ Load        │
Users ──────────────→│ Balancer    │─────→ Server 1
                     │             │─────→ Server 2
                     │             │─────→ Server 3
                     └─────────────┘
                           ↑
                    Add Server 4 here
                    when traffic grows
```

Servers are **stateless** — any server handles any request. State lives in Redis/DB.

---

## Global Load Balancing (DNS-based)

For multi-region setups:

```
user in Tokyo → DNS resolves "api.example.com" → Tokyo DC IP
user in NYC   → DNS resolves "api.example.com" → NYC DC IP
```

**Approaches:**
- **GeoDNS:** DNS returns different IPs based on client location
- **Anycast:** Same IP advertised from multiple DCs; BGP routes to nearest
- **AWS Route 53 / Cloudflare:** Built-in latency-based routing

---

## Load Balancer Redundancy

The load balancer itself can be a single point of failure.

**Solution:** Active-passive LB pair with **Virtual IP (VIP)** failover:
```
Active LB  ← VIP 10.0.0.1 (all traffic)
Passive LB ← waiting

Active LB crashes →
Passive LB takes over VIP → traffic continues within seconds
```

Or: use DNS + multiple LB IPs with short TTL.

---

## Common Tools

| Tool | Layer | Use Case |
|------|-------|---------|
| **nginx** | L7 | HTTP reverse proxy + LB, most common |
| **HAProxy** | L4 + L7 | High-performance, flexible |
| **AWS ALB** | L7 | AWS HTTP/HTTPS with path routing |
| **AWS NLB** | L4 | AWS ultra-high throughput TCP |
| **Envoy** | L7 | Service mesh (Kubernetes, Istio) |
| **Cloudflare** | L7 | Global + DDoS protection |

---

## Interview Q&A

**Q: What is the difference between L4 and L7 load balancing?**  
A: L4 routes based on IP/port — fast but no content awareness. L7 inspects HTTP headers/URL/cookies — can do path-based routing, sticky sessions via cookies, SSL termination, content-based routing to microservices. Use L4 for non-HTTP or ultra-high-throughput. Use L7 for web applications and microservices.

**Q: What load balancing algorithm would you choose and why?**  
A: Least Connections for most web applications — adapts to variable request lengths. Round Robin when requests are roughly equal in cost and all servers are identical. Weighted Round Robin for heterogeneous fleets. IP Hash when sticky sessions are required (WebSocket, stateful protocols).

**Q: How do you make servers stateless to work with load balancing?**  
A: Move all state out of the server process. Sessions → Redis. File uploads → S3. Database → shared DB cluster. Config → environment variables or config service. Any server can handle any request because nothing is stored locally on the server.

**Q: What happens when a server behind a load balancer crashes?**  
A: Health checks detect the failure (active: failed /health ping; passive: consecutive 5xx responses). Load balancer removes the server from rotation. In-flight requests on that server are lost (client retries). New requests go to remaining healthy servers. When the server recovers and passes health checks, it's added back.

**Q: What is SSL termination and why is it done at the load balancer?**  
A: The load balancer decrypts HTTPS from clients and forwards plain HTTP to backend servers. Benefits: backend servers don't need TLS certificates or decryption CPU overhead; certificate management is centralized at the LB; internal traffic uses fast plain HTTP. The trade-off is that internal traffic is unencrypted (mitigated by private network/VPC isolation).
