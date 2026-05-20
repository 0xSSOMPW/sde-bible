# Q: Design a search autocomplete (typeahead).

**Answer:**

Type "fa" → see "facebook," "facetime," "facelift" instantly. Tests **trie / prefix data structures**, **ranking**, **personalization**, and **scale at fan-out**.

### Requirements

**Functional**:
- Suggest top-K completions as user types.
- Ranked by popularity, recency, personalization.
- Multi-word queries.
- Misspelling tolerance (optional but expected).

**Non-functional**:
- p99 < 100 ms (it's per keystroke).
- 50k+ QPS at peak.
- Real-time updates as trending terms emerge.

### Capacity Estimation

- 10B searches/day → suggestions are typed ~3× more often → 30B keystrokes/day.
- ~350k QPS, peak 700k.
- Latency budget extremely tight; can't hit a remote DB per keystroke.

### Architecture

```
client keystroke
       │
       ▼ debounced (~100 ms)
   CDN / edge
       │
       ▼
   Autocomplete service (stateless)
       │ in-memory trie / inverted index
       ▼
   prefix lookup, rank, return top K
```

In-memory data structure is mandatory for the QPS + latency.

### Data Structures

**Trie**:

```
            (root)
              │
              a─── b─── o─── u─── ...
                    │
                    c─── i─── ...
                    │
                    e─── ...
                          │
                    "ace" (popularity 100k)
                    "ace ventura" (popularity 5k)
```

Each leaf node stores top-K completions sorted by score. Walk to the prefix node; return its cached top-K.

Pros: O(prefix length) lookup, naturally hierarchical.
Cons: memory-heavy (one node per char × many strings); inefficient if many keys with the same prefix.

**Compressed Trie (Patricia / Radix)**:

Merges single-child chains. Same operations, less memory.

**Inverted index of n-grams**:

For each prefix length 1–5, map prefix → top-K completions.

```
"fa"  -> ["facebook", "facetime", "facelift", ...]
"fac" -> ["facebook", "facetime", "facelift", ...]
"face"-> ["facebook", "facetime", "facelift", ...]
```

Pros: O(1) lookup.
Cons: storage = sum over prefix lengths; precompute step.

Production: hybrid — compressed trie + cached top-K at each node.

### Ranking

Initial score = popularity (search count). Refinements:

- **Recency**: decay older searches.
- **Personalization**: bias by user's history, location, language.
- **Diversity**: don't return 10 variations of "facebook"; spread suggestions.
- **Type**: brand vs query vs entity vs longtail.

Score updated periodically; trie reflects fresh scores after batch rebuild.

### Build Pipeline

```
Search logs (Kafka)
       │
       ▼
  Aggregator (Flink / Spark)
       │
       ▼
  Top-N queries per prefix
       │
       ▼
  Trie builder: produces compact binary trie
       │
       ▼
  Distribute to autocomplete servers
       │
       ▼
  Servers atomically swap to new trie
```

Rebuild cadence: hourly for recent trends; daily for full rebuild.

### Real-Time Updates

Pure batch is slow. To capture surging trends:
- Stream recent searches.
- Per-prefix counter (Count-Min Sketch) updated live.
- Online ranker blends batch trie + recent counts.

### Serving

Each autocomplete server holds the full trie in RAM (~few GB for English). LB routes any request to any server (stateless).

```
on request:
  1. tokenize query into prefix
  2. walk trie to prefix node
  3. fetch node.top_k
  4. apply personalization rerank
  5. return top 10
```

Per-server: single-digit ms response.

### Misspelling Tolerance

Edit-distance search:
- BK-tree (Burkhard-Keller) for fuzzy matching.
- Phonetic encoding (Soundex / Metaphone).
- Symspell algorithm: precompute deletes within edit distance N.

For autocomplete: light correction (edit distance 1) on the prefix; mark suggestions as corrections.

### Personalization

Per-user trie / index would explode memory. Practical:
- Personal recent searches in a small per-user cache.
- Merge with global top-K at rerank time.
- Heavier personalization via separate ML model called after global lookup.

### Caching at CDN

Repeated queries from many users for the same prefix:
- Cache at edge for 1–5 minutes.
- Cache hit ratio > 50% on common prefixes.
- Personalization breaks caching; restrict to non-personalized tier or use Edge KV.

### Multi-Lingual

Per-language trie. Detect language from query, browser locale, account setting. Route to the appropriate trie.

### Scale Numbers

- 700k QPS at peak.
- Each server can serve ~50k QPS in-memory.
- → 15–20 servers behind LB.
- Trie size: ~5 GB English; servers commodity 64 GB RAM.

### Failure Modes

| Failure | Handling |
|---------|---------|
| Server dies | LB removes; warm spare takes over (trie loaded at startup) |
| Trie corrupt | Roll back to previous snapshot; alert |
| Trending term not in trie yet | Live counter blend catches it |
| Spam abuse pushes "evil_term" up | Anti-spam filters at log ingest |

### Observability

- Per-prefix latency.
- Top suggestions tracked for monitoring (sanity).
- Click-through rate on suggestions = quality metric.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Hitting DB per keystroke | Service-level cache or in-memory trie |
| Recomputing top-K on every request | Pre-compute at trie build time |
| No popularity decay | Stale rankings forever |
| Storing search history of every user in autocomplete server | Privacy + memory; use separate personal index |
| Returning unsanitized strings | XSS risk; encode |

> [!NOTE]
> The decisive design choice is *where* the trie lives — in-memory in the server, or as a managed index (Elasticsearch suggester). In-memory wins on latency; ES wins on operational simplicity at moderate QPS.

### Interview Follow-ups

- *"How would you A/B test suggestion ranking?"* — Split traffic; log impressions + clicks; compare CTR. Pick winner.
- *"How do you handle multi-word prefix?"* — Tokenize last token; suggest completions of last token + use earlier tokens as context for rerank.
- *"How would you do this without trees in production?"* — Elasticsearch's edge-ngram tokenizer + completion suggester. Slower than custom but operationally simpler.
