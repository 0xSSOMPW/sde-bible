# Q: How do CDNs work, and when do you use one?

**Answer:**

A **CDN** (Content Delivery Network) is a global network of cache servers (Points of Presence — PoPs) that serve content close to the user, reducing latency and origin load. It's the cheapest single performance optimization a web app can make.

### What CDNs Cache

| Content | Cache value |
|---------|------------|
| Static assets (JS, CSS, images, fonts) | Very high; rarely change |
| Video segments (HLS/DASH chunks) | Very high; large objects |
| API responses (signed/idempotent) | Variable; can be huge win |
| User-personalized HTML | Low — needs careful cache keys |
| WebSocket / SSE | None — pass-through only |

### How a Request Flows

```
1. User in Mumbai requests https://cdn.example.com/img.png
2. DNS → CDN PoP near Mumbai (anycast or geo)
3. PoP looks up img.png in its cache
4. HIT  → return from PoP   (~10 ms latency)
   MISS → fetch from origin (~150 ms first time, then cached for next user)
```

PoPs sometimes form a tree: edge PoP → regional shield PoP → origin. The shield absorbs origin load by deduplicating misses across many edges.

### Cache Key

Defines what counts as "the same object" at the CDN:

```
default: scheme + host + path + query
   GET /img.png?v=1   ≠   GET /img.png?v=2
```

Configure:
- Strip query strings if irrelevant.
- Include `Accept-Encoding` so gzipped vs br vs identity are separate.
- Include `Vary: ...` headers (CDN respects standards).
- Avoid including auth cookies — turns every user into a unique key.

### Cache Headers

Origin tells the CDN how long to cache.

```
Cache-Control: public, max-age=31536000, immutable
   ↑ public to CDN, browser keeps it 1 year, won't even revalidate
```

Versioned assets pattern:

```
/static/app.<hash>.js          # never changes; cache 1 year
/static/app.js                 # mutable; very short cache or no-cache
```

Revalidation:

```
Cache-Control: max-age=0, must-revalidate
ETag: "abc123"

On next request, browser/CDN sends If-None-Match: "abc123"
Origin returns 304 Not Modified — saves bandwidth, costs a roundtrip.
```

### Stale-While-Revalidate

```
Cache-Control: max-age=60, stale-while-revalidate=300
```

CDN serves cached object for 60s. From 60s–360s, serves stale instantly **and** asynchronously refetches. Eliminates user-visible refresh latency.

### Cache Invalidation (Purge)

Three modes:

- **Purge by URL**: nuke specific paths.
- **Purge by tag** (Cloudflare, Fastly): tag assets at insert time, purge by tag — invalidate "all blog posts" with one call.
- **Wait for TTL**: cheapest; works if TTL is acceptable.

Purge is *propagation*, not instant — usually < 30s but not zero.

### Origin Shield

A "shield" PoP sits between edge PoPs and origin. All misses funnel through the shield.

```
100 edge PoPs miss "img.png" → 100 requests at origin?
With shield:                  → 1 request at origin (others served from shield)
```

Critical for protecting origin during traffic spikes.

### CDN for Dynamic Content

Two strategies:

**1. Cache short.**

```
Cache-Control: public, max-age=10
```

Even 10 seconds of caching absorbs huge bursts on hot pages (Reddit front page, breaking news). API responses for read-mostly endpoints.

**2. Edge compute.**

CDN runs your code at PoPs:

- Cloudflare Workers, Vercel Edge Functions, Lambda@Edge, Fastly Compute@Edge.

Use cases: A/B testing, redirects, light personalization, rate limiting, JWT verification. Sub-50ms latency anywhere.

### Geographic Routing

DNS-based: CDN's DNS resolver returns IP of nearest PoP based on resolver's location (which is mostly the user's ISP).

Anycast: same IP advertised from many PoPs via BGP; the network routes to the topologically nearest one. Faster failover but harder to debug.

### Security Features

Modern CDNs are also security layers:

- **DDoS absorption**: PoPs absorb L3/L4 floods.
- **WAF** (web app firewall): rule-based filtering at edge.
- **Bot management**: CAPTCHA, rate limit by fingerprint.
- **TLS termination + cert management**: free certs, automatic renewal.
- **Origin shield**: hide origin IPs.

### When NOT to Use a CDN

- WebSockets / SSE (most CDNs are pass-through; some now support).
- Real-time bidirectional protocols.
- Highly personalized HTML that varies per user (cache will miss).
- Internal services not exposed to the internet.

For uncacheable traffic, the CDN still adds value as a TLS terminator + DDoS shield.

### Cache Hit Ratio

```
hit_ratio = hits / (hits + misses)
```

Targets:
- Static assets: > 95%.
- Dynamic with short cache: 70–90%.
- API caching: varies; measure per-endpoint.

Diagnose low hit ratio:
- Cookies / auth headers in cache key.
- Query string randomization.
- Different `Accept` headers per client.
- TTL too short.

### Costs

Pricing is typically per-GB egress + per-request. Cheaper than your origin's egress. For media-heavy apps, the CDN itself is the dominant cost line — measure and optimize.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| `Cache-Control: no-cache` everywhere "for safety" | Bypasses the CDN; defeats the point |
| Versioning files via query string (`?v=1`) but no cache header | CDN treats them as one URL with stale content |
| Caching authenticated responses | Privacy bug — user A sees user B's data |
| One huge `purge_all` to fix any stale issue | Origin gets pummeled; use tag-based purge |
| Forgetting `Vary: Accept-Encoding` | Compressed/uncompressed get confused |

> [!NOTE]
> The first hour of optimizing a web app's performance is almost always CDN setup: long-lived cached versioned assets, short cached API responses, fingerprinted filenames, edge TLS. Compounding gains for trivial work.

### Interview Follow-ups

- *"How do you cache user-specific data at the edge?"* — Use `Vary: Cookie` only for the specific cookie that identifies cacheable variants (e.g., locale, not session). Or use Edge Workers to assemble personalized + cached fragments.
- *"How do you handle a CDN-cached bad response?"* — Tag-based purge or version bump. Treat broken cache as a real outage class.
- *"How do you serve video?"* — Encode to HLS/DASH segments, signed URLs, long TTL on segments, manifest with shorter TTL. CDN serves the bulk; origin only fingerprints manifests.
