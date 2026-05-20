# Q: Design a web crawler.

**Answer:**

Classic distributed-system problem. Tests **URL frontier management**, **politeness**, **duplicate detection**, **content storage**, and **scaling crawl rate**.

### Requirements

**Functional**:
- Crawl billions of URLs.
- Respect `robots.txt` and rate limits per host.
- Detect duplicates (same URL, same content).
- Extract links to extend frontier.
- Re-crawl based on freshness policy.

**Non-functional**:
- Throughput: 1B pages/month → ~400 QPS sustained.
- Storage: ~10 KB per page average → 10 TB/month.
- Politeness: don't get banned.

### High-Level Architecture

```
            ┌──────────────┐
            │  Seed URLs   │
            └──────┬───────┘
                   ▼
            ┌──────────────────┐
            │  URL Frontier    │  ← priority + politeness queues
            └──────┬───────────┘
                   ▼
            ┌──────────────────┐
            │   Fetcher Pool   │  ← N workers; DNS, HTTP
            └──────┬───────────┘
                   ▼
            ┌──────────────────┐
            │   Parser         │  ← HTML/JS, extract links + content
            └──────┬───────────┘
                   ▼
   ┌───────────────┼────────────────┐
   ▼               ▼                ▼
 Content       Link extraction   Duplicate
 Storage         to frontier      detection
 (S3)                              (Bloom + content hash)
```

### URL Frontier

The work queue of URLs to fetch. Two requirements:

1. **Priority**: important URLs (high PageRank, fresh news) crawled sooner.
2. **Politeness**: same host crawled with delay (no DDoS).

Implementation:
- Top level: priority queue by URL score.
- Second level: per-host queue with rate limit.

```
priority_queue → host_queues[host] → fetch_workers
```

Each host queue dequeues at the host's allowed rate (e.g., 1 req/sec per host).

### DNS

DNS is a bottleneck. Each fetch needs an A-record lookup.

- Cache DNS results aggressively (TTL).
- Run a DNS resolver fleet (Unbound, dnsmasq) close to fetchers.
- Pre-resolve for known hosts.

### Politeness

`robots.txt`:
- Fetch once per host per day.
- Respect `Disallow`, `Crawl-delay`.
- Cache locally.

Rate limit per host:
- Token bucket per host, refilled by `Crawl-delay` or default (1 req/sec, conservative).
- User-Agent identifying the bot + contact info.

### Fetching

HTTP GET. Considerations:

- **HTTP/2**: persistent connections, multiplex. Faster.
- **Robots-respect**: skip disallowed paths.
- **Timeouts**: 10–30s.
- **Size limits**: 10 MB max page; drop larger.
- **Redirects**: follow up to 5.
- **Status codes**: 200 → parse; 3xx → follow; 4xx → blacklist; 5xx → retry later.

Some pages are JS-rendered. Headless browsers (Chrome via Puppeteer/Playwright) handle them but cost 10–100× more.

Production: lightweight HTTP for most; headless for known SPA hosts.

### Parsing

- HTML parsing: BeautifulSoup, lxml, or Cheerio.
- Extract: text content, links (`<a href>`), meta tags, canonical URL.
- Normalize URLs: lowercase host, strip trailing slash, sort query params, remove fragments.

```python
canonical("HTTPS://Example.COM/PATH/?b=1&a=2#frag")
  → "https://example.com/path?a=1&b=2"
```

### Duplicate Detection

Two kinds:

**1. URL dedup**: have I seen this URL?

```
Bloom filter:    1B URLs × 8 bits ≈ 1 GB; 0.1% false-positive
Exact check on hit: lookup in DB
```

**2. Content dedup**: same page, different URL.

```
content_hash = SHA-256(normalized_text)
seen_hashes (Cassandra) holds content_hash → first_url
```

Common at scale: mirrors, syndicated content, duplicate sitemaps. Skip if content matches.

### Scheduling for Freshness

Different sites change at different rates. Track `Last-Modified` + observed change frequency:

```
news.example.com    → re-crawl every hour
wikipedia.org/page  → re-crawl every day
archive.org         → re-crawl monthly
```

Promote pages back to frontier based on schedule.

### Storage

| Data | Store |
|------|-------|
| Raw HTML | S3 / object storage (cheap, write-once) |
| Parsed text + metadata | HBase / Cassandra |
| URL frontier | Kafka or Redis (priority queue) |
| Visited URLs | Bloom filter + Cassandra |
| Link graph (for PageRank) | Graph DB or wide-column |
| Search index | Elasticsearch / inverted index |

### Distributed Coordination

Fetchers are stateless workers. Coordination via:
- Frontier queue (Kafka): partitioned by host hash.
- Locks per host (Redis SETNX or per-shard ownership) to enforce politeness.

If a fetcher crashes, its host queue is reassigned. URLs in-flight may be retried (at-least-once).

### Scaling Beyond One DC

- Each region crawls hosts closest to it (lower latency, better robots-compliance via IP).
- Discovered URLs flow back to global frontier.
- Deduplication happens globally (one source of truth for "seen" set).

### Trap Avoidance

Bad sites can trap crawlers:
- Infinite calendars (`/calendar/2024/01/01`, `/calendar/2024/01/02`, ...).
- Session-ID URLs (every link different).
- Spider traps with fake links.

Mitigations:
- Per-host page count cap.
- Pattern detection (path depth > N, calendar-like patterns).
- Diff URLs against canonical forms.

### Adversarial Content

- Malware-serving pages → safe-browse check on download.
- Rate-limit re-crawl of malware hosts to zero.
- Don't follow `nofollow` links (per robots conventions).

### Failure Modes

| Failure | Handling |
|---------|---------|
| One host returns 500s | Backoff, eventually skip |
| Network partition for a region | Other regions continue; reconnect resumes |
| Frontier exhausted | Re-seed from priority list / sitemap discovery |
| Bloom filter false positive (URL marked seen but wasn't) | Acceptable; eventually re-discovered |
| Cassandra slow | Buffer locally; backpressure to fetchers |

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Single frontier without per-host queues | DDoSing hosts; getting banned |
| No URL normalization | Crawling the same page N times under different URLs |
| Synchronous DNS | DNS bottleneck; cache + async resolver |
| One giant DB for everything | Different access patterns; separate stores |
| No robots.txt respect | Legal + ethical issue + bans |

> [!NOTE]
> Politeness is a **first-class design constraint**, not an afterthought. A 1000-worker crawler that ignores `robots.txt` gets your ASN banned from the internet in days.

### Interview Follow-ups

- *"How do you crawl JS-heavy sites?"* — Headless browser tier; sample-based (don't run heavy renderer on every page).
- *"How do you discover new URLs?"* — Sitemaps, RSS feeds, social media references, partner data feeds.
- *"What's PageRank's role in priority?"* — Higher PageRank → higher crawl priority (more freshness). Computed offline; injected as URL metadata.
