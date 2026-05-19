# Reverse Proxy vs API Gateway

## Quick Definitions

| | Reverse Proxy | API Gateway |
|--|--------------|-------------|
| **What** | Sits in front of servers, forwards requests | Sits at the entry point of a microservices system |
| **Focus** | Infrastructure-level routing, SSL, caching | API-level concerns: auth, rate limiting, transformation |
| **Typical user** | Any web service | Microservices / public APIs |
| **Examples** | nginx, HAProxy, Squid | Kong, AWS API Gateway, Apigee, Traefik |

> **Analogy:**  
> Reverse Proxy = Security guard at a building entrance — checks you in, directs you to the right floor.  
> API Gateway = Concierge + security desk — verifies your identity, logs your visit, limits how many times you can enter, translates your request, then routes you.

---

## What Is a Reverse Proxy?

A **reverse proxy** receives requests from clients and forwards them to one or more backend servers. The client only ever sees the proxy — it doesn't know which backend server handled the request.

```
Client → Reverse Proxy → Backend Server A
                       → Backend Server B
                       → Backend Server C
```

### Core Functions

**1. Load balancing** — Distribute requests across backend servers
```
nginx upstream configuration:
  server app1.internal:8080;
  server app2.internal:8080;
  server app3.internal:8080;
```

**2. SSL/TLS termination** — Handle HTTPS at the proxy, use plain HTTP internally
```
Client ──(HTTPS)──→ Reverse Proxy ──(HTTP)──→ Backend
   SSL decrypted here (proxy has the cert)
   Backends don't need certs, cheaper compute
```

**3. Static file serving** — Serve CSS, JS, images directly (no need to hit app servers)

**4. Compression** — gzip responses before sending to client

**5. Caching** — Cache backend responses for repeated requests

**6. Hide backend topology** — Clients don't know internal IPs/ports

### nginx as Reverse Proxy (Example)
```nginx
server {
    listen 443 ssl;
    server_name api.example.com;

    ssl_certificate     /etc/ssl/cert.pem;
    ssl_certificate_key /etc/ssl/key.pem;

    location / {
        proxy_pass http://app_servers;  # forward to upstream
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

upstream app_servers {
    server app1.internal:8080;
    server app2.internal:8080;
    server app3.internal:8080;
}
```

---

## What Is an API Gateway?

An **API gateway** is a reverse proxy with **application-layer intelligence** for APIs. It acts as the single entry point for all client requests in a microservices architecture.

```
Mobile App  ─┐
Web Client  ─┤→ API Gateway → User Service
Partner API ─┘             → Order Service
                           → Payment Service
                           → Notification Service
```

### Core Functions (beyond reverse proxy)

**1. Authentication & Authorization**
```
Client sends JWT or API key.
API Gateway validates:
  - Is the token valid? (verify signature)
  - Is it expired?
  - Does the user have permission for this endpoint?
Only valid requests reach the backend services.
```

**2. Rate Limiting**
```
Rule: 100 requests/minute per API key
API Gateway tracks usage in Redis.
Request 101 → 429 Too Many Requests (backend never sees it).
```

**3. Request/Response Transformation**
```
Client sends old API format (v1) → Gateway translates → Backend receives new format (v2)
Backend sends internal format → Gateway translates → Client receives public format
Useful for API versioning without changing backends
```

**4. Request Routing**
```
GET  /users/*      → User Service
GET  /orders/*     → Order Service
POST /payments/*   → Payment Service

Path-based, header-based, or method-based routing.
```

**5. Protocol Translation**
```
REST (HTTP/1.1) from client → gRPC internally to microservices
GraphQL from client → multiple REST calls to services aggregated
```

**6. Logging & Observability**
```
Every request logged: timestamp, user, endpoint, latency, status code
All in one place — no need to aggregate logs from all services
```

**7. Service Aggregation (BFF Pattern)**
```
Mobile app needs: user profile + order history + notifications in one call
Gateway calls all 3 services and combines the response
Client makes 1 request instead of 3
```

---

## Side-by-Side Comparison

| Feature | Reverse Proxy | API Gateway |
|---------|--------------|-------------|
| **Load balancing** | ✅ | ✅ |
| **SSL termination** | ✅ | ✅ |
| **Static file serving** | ✅ | ❌ (not typical) |
| **Response compression** | ✅ | Sometimes |
| **Authentication** | Basic (e.g., basic auth) | ✅ JWT/OAuth/API keys |
| **Rate limiting** | Basic | ✅ Per-user, per-key, per-endpoint |
| **Request transformation** | ❌ | ✅ |
| **Protocol translation** | ❌ | ✅ REST↔gRPC, WebSocket upgrade |
| **API versioning** | ❌ | ✅ |
| **Analytics/tracing** | Basic access logs | ✅ Per-API metrics |
| **Service aggregation** | ❌ | ✅ |

---

## Tool Comparison

### nginx (Reverse Proxy)
- Extremely fast, lightweight C process
- Excellent for static files, SSL termination, load balancing
- API gateway features require Lua scripting (nginx Plus / OpenResty)
- Best for: traditional web apps, static sites, simple microservices

### Kong (API Gateway built on nginx/OpenResty)
- Open source API gateway
- Plugin system: rate limiting, auth, logging, tracing — all as plugins
- Admin API for dynamic configuration (no config file reload needed)
- Best for: microservices needing rich API management

### AWS API Gateway
- Fully managed, serverless
- Integrates natively with Lambda, ECS, ALB
- Pay per request model
- Best for: AWS-native architectures, serverless

### Traefik
- Automatic service discovery (reads Docker/Kubernetes labels)
- Built-in Let's Encrypt HTTPS
- Best for: Kubernetes/Docker environments, auto-configuration

### Apigee (Google Cloud)
- Enterprise API management
- Advanced analytics, monetization, developer portal
- Best for: large enterprises exposing public APIs

---

## The Service Mesh Connection

An API gateway handles **north-south traffic** (external clients → services).  
A **service mesh** (Envoy/Istio) handles **east-west traffic** (service → service, internal).

```
Internet
   ↓
API Gateway  ← north-south (external entry point)
   ↓
Service A ──→ Service B  ← east-west (internal, handled by service mesh)
           ──→ Service C
```

**Service mesh** (Istio/Envoy) provides:
- mTLS between services (mutual authentication)
- Internal load balancing and circuit breaking
- Distributed tracing (without code changes)
- Traffic policies (canary deployments, traffic splitting)

**Use both:** API Gateway at the edge, service mesh internally.

---

## When to Use What

**Use a Reverse Proxy when:**
- Simple web app with 1-3 backends
- You need SSL termination and load balancing
- Serving static files alongside dynamic content
- You don't need per-user API controls

**Use an API Gateway when:**
- You have many microservices (5+)
- External clients (mobile apps, partners) need a stable API surface
- You need per-user authentication and rate limiting
- You want to hide your internal service topology from clients
- You need API versioning and transformation

**Use Both (common pattern):**
```
CDN (CloudFront) → API Gateway (Kong/AWS GW) → Services
                                ↕
                    Each service has nginx as its own local reverse proxy
```

---

## Interview Q&A

**Q: What is the difference between a reverse proxy and an API gateway?**  
A: A reverse proxy handles infrastructure-level concerns: forwarding requests to backends, SSL termination, load balancing, and compression. An API gateway does all that plus API-level concerns: authentication/authorization, rate limiting per user/key, request/response transformation, API versioning, and analytics. An API gateway is essentially a reverse proxy with application intelligence for microservices.

**Q: Why do microservices architectures need an API gateway?**  
A: Without an API gateway, every client must know the address of every service, handle authentication for each service, and deal with multiple API versions. An API gateway provides: a single stable entry point, centralized authentication (validate token once, not in every service), centralized rate limiting, and the ability to change internal service topology without breaking clients.

**Q: What is the BFF (Backend for Frontend) pattern?**  
A: Different clients (mobile, web, partner) have different data needs. Instead of one generic API gateway, you create separate gateways tailored for each client type. The mobile BFF returns minimal data optimized for mobile bandwidth. The web BFF returns richer data. The partner BFF enforces strict rate limiting and authentication. Each BFF aggregates calls to multiple internal services and shapes the response for its specific client.

**Q: How does an API gateway relate to a service mesh?**  
A: An API gateway handles north-south traffic — external clients entering the system. A service mesh (Istio/Envoy) handles east-west traffic — internal communication between services. They're complementary. The API gateway is the single external entry point with authentication, rate limiting, and API management. The service mesh provides mTLS, circuit breaking, and distributed tracing between services inside the cluster.

**Q: Where does SSL termination happen and why?**  
A: SSL is terminated at the reverse proxy or API gateway (the entry point), not at the backend services. The proxy holds the certificate and decrypts HTTPS. Internal traffic uses plain HTTP. Benefits: backends don't need certificates, SSL computation is centralized (hardware acceleration possible), and certificate management is in one place. The internal network is trusted (within a VPC/datacenter) so plain HTTP is acceptable.
