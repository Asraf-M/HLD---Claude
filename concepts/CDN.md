# CDN (Content Delivery Network)

## What Is It?

A CDN is a **globally distributed network of servers** (called edge servers or Points of Presence — PoPs) that cache and serve content from locations physically close to users.

> **Analogy:** Instead of everyone flying to one central warehouse to pick up packages, Amazon has local distribution centers in every city. Users get their packages faster.

**Without CDN:** User in Tokyo requests an image from a server in New York → ~150ms round trip.  
**With CDN:** User in Tokyo requests from a CDN edge server in Tokyo → ~5ms round trip.

---

## What CDNs Serve

**Static content (primary use case):**
- Images, videos, audio files
- HTML, CSS, JavaScript files
- Fonts, icons
- Software downloads

**Dynamic content (advanced CDNs):**
- API responses (edge caching with short TTLs)
- Personalized pages (edge compute — Cloudflare Workers, Lambda@Edge)

---

## How a CDN Works

### First Request (Cache Miss)

```
User (Tokyo) → CDN Edge (Tokyo) → [MISS] → Origin Server (New York)
                                          ← file returned
CDN Edge (Tokyo) ← file cached ← Origin
User (Tokyo) ← file served from CDN
```

1. User requests `image.jpg`
2. CDN edge server in Tokyo doesn't have it (cache miss)
3. CDN fetches from the origin server (your backend)
4. CDN **caches the file** at the Tokyo edge
5. CDN returns the file to the user

### Subsequent Requests (Cache Hit)

```
User 2 (Osaka) → CDN Edge (Tokyo) → [HIT] → serves from cache
User 3 (Seoul) → CDN Edge (Tokyo) → [HIT] → serves from cache
```

All future users in Asia get the file directly from Tokyo — origin server is not touched.

---

## CDN Architecture

```
                    ┌─────────────────────────────────┐
                    │         Origin Server            │
                    │   (your actual web server/S3)   │
                    └─────────┬───────────────────────┘
                              │  (only fetched on miss)
          ┌───────────────────┼───────────────────┐
          ↓                   ↓                   ↓
   CDN PoP (NYC)       CDN PoP (London)    CDN PoP (Tokyo)
   Edge servers         Edge servers        Edge servers
          ↑                   ↑                   ↑
   US users              EU users            Asia users
```

Major CDN providers: **Cloudflare, Akamai, AWS CloudFront, Google Cloud CDN, Fastly**

---

## Cache Headers — How CDN Knows What to Cache

### Cache-Control header (set by origin server)

```http
Cache-Control: max-age=86400, public
```

- `max-age=86400` — cache for 86,400 seconds (1 day)
- `public` — can be cached by CDN (and browsers)
- `private` — only browser caches it, not CDN
- `no-cache` — must revalidate with origin before serving
- `no-store` — never cache (sensitive data)

### ETag (content fingerprint)

```http
ETag: "abc123def456"
```

CDN sends `If-None-Match: "abc123def456"` on revalidation.  
Origin returns `304 Not Modified` if unchanged — saves bandwidth.

---

## Cache Invalidation

Problem: you deployed a new version of `app.js` but CDN still serves the old one.

### Solutions

**1. URL Versioning (Best Practice)**
```html
<!-- Old -->
<script src="/app.js"></script>

<!-- New: file hash in filename -->
<script src="/app.abc123.js"></script>
```
New URL = cache miss = fresh file served automatically. Old URL expires naturally.

**2. Cache Purge API**
Most CDNs let you programmatically invalidate cached files:
```bash
# CloudFront
aws cloudfront create-invalidation --distribution-id ABC --paths "/*"

# Cloudflare
curl -X DELETE "https://api.cloudflare.com/client/v4/zones/{zone}/purge_cache"
```

**3. Short TTL**
Set `max-age=60` for frequently changing content. Old content auto-expires in 60 seconds. Trade-off: more origin requests.

---

## Pull CDN vs Push CDN

### Pull CDN (Most Common)
- CDN fetches content from origin **on first request** (lazy loading)
- Easy to set up — just point CDN at your origin domain
- **Used by:** Cloudflare, CloudFront, Google CDN
- Best for: websites, web apps with unpredictable content patterns

### Push CDN
- You **proactively upload** content to CDN before anyone requests it
- Better for large files (videos, software) you know will be popular
- **Used by:** Akamai (for large media), MaxCDN
- Best for: known, large static assets (video files, game downloads)

---

## CDN and HTTPS

CDNs handle **TLS termination** at the edge:
- User's HTTPS connection terminates at the CDN edge (close to user → low latency)
- CDN connects to origin over HTTPS or HTTP internally

CDN providers offer **free TLS certificates** via Let's Encrypt (Cloudflare, Fastly).

---

## Benefits Summary

| Benefit | How |
|---------|-----|
| **Lower latency** | Serve from nearby edge server |
| **Reduced origin load** | Most requests hit CDN, not origin |
| **Handles traffic spikes** | CDN absorbs burst traffic; origin stays stable |
| **DDoS protection** | CDN absorbs/filters attack traffic before it hits origin |
| **Global availability** | If one PoP goes down, others serve traffic |
| **Bandwidth cost reduction** | Data transfer from CDN often cheaper than from origin |

---

## When NOT to Use CDN

- **Private/sensitive content** that must not be cached on third-party servers
- **Highly dynamic content** that changes per-user per-request (use `private` cache or skip CDN)
- **Very small applications** with geographically concentrated users

---

## CDN in System Design Interviews

**Design YouTube/Netflix:** All videos served through CDN. Origin stores master files. CDN caches popular videos at edge servers closest to viewers.

**Design Twitter/Instagram:** Profile images, static assets on CDN. API responses usually not cached (user-specific).

**Design Google Drive:** Files downloaded through CDN. Direct upload goes to origin/storage.

**Common pattern:**
```
Static files → CDN (long TTL, versioned URLs)
User-specific API → origin (no CDN)
Public API responses → CDN (short TTL, 30-60 seconds)
```

---

## Interview Q&A

**Q: What is a CDN and why do we use it?**  
A: A CDN is a distributed network of edge servers geographically close to users that cache and serve static content. We use it to reduce latency (serve from nearby node vs distant origin), reduce origin server load (most requests hit cache), and handle traffic spikes. A Tokyo user gets content from a Tokyo edge server in ~5ms vs ~150ms from New York.

**Q: What is the difference between a cache hit and a cache miss in a CDN?**  
A: Cache hit: the edge server already has the requested file → serves it immediately without contacting the origin. Cache miss: file not cached yet → edge fetches from origin, caches it, then serves it. Only the first request for a file causes a cache miss; all subsequent requests from that region are cache hits.

**Q: How do you handle CDN cache invalidation after deploying new code?**  
A: Best practice: use content-addressed filenames (include file hash in URL, e.g., `app.abc123.js`). New build = new URL = automatic cache miss for updated files, old files expire naturally. Alternatively, use the CDN provider's purge/invalidation API to forcibly remove specific cached files. Short TTLs also work for frequently changing content at the cost of more origin requests.

**Q: What is the difference between pull CDN and push CDN?**  
A: Pull CDN: CDN fetches content from origin on first request (lazy). Easy to set up; best for websites with many small/medium files. Push CDN: you proactively upload content to CDN before requests arrive. Better for large known files (videos) you want pre-distributed globally.
