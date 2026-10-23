# Q: Design a music streaming service (Spotify).

**Answer:**

Spotify-style streaming combines **audio delivery** (small files, fast start, CDN), a **metadata/catalog** layer, **playlists** (collaborative, ordered, heavily read), a **listening-event pipeline** (history, recommendations, royalties), and **personalization** (Discover Weekly). Unlike video, audio files are small (3-10 MB), so the hard problems are **play-start latency**, **cost of billions of streams**, **royalty-accurate counting**, and **recommendation quality**, not raw bandwidth per stream.

### Requirements

**Functional**:
- Search and browse tracks, albums, artists, podcasts.
- Play a track instantly; seek; gapless playback and crossfade.
- Playlists (create, edit, collaborative, follow), library (liked songs).
- Personalized home, Daily Mix, Discover Weekly, radio.
- Offline downloads (Premium).
- Listening history, "recently played".
- Cross-device playback control (Spotify Connect).
- Free tier with ads; Premium without.
- Social: follow friends, friend activity, share.
- Artist/label upload pipeline, royalties reporting.
- Podcasts: episodes, RSS ingest, resume position.

**Non-functional**:
- 600M+ users, ~250M+ paid, 100M+ tracks.
- **Play-start p95 < 300 ms** (tap to sound); skip must feel instant.
- Playback availability 99.99%: no rebuffer for the common case.
- Eventually consistent for history and recommendations; **strongly consistent** for playlist edits within a user, and for royalty counts (auditable).
- Licensing: geo-restrictions, DRM, label contracts.

### Capacity Estimation

- 600M MAU, ~200M DAU, avg 2 hours listening/day.
- Avg track ~3.5 min ⇒ ~35 tracks/user/day ⇒ **~7B streams/day** ⇒ ~80k stream starts/sec average, ~250k peak.
- Track size: 160 kbps Vorbis ≈ 4 MB; 320 kbps ≈ 8 MB. Avg delivered ≈ 5 MB.
- Egress: 7B x 5 MB = 35 PB/day ≈ 3.2 Tbps average, ~10 Tbps peak. **CDN is the dominant cost.**
- Catalog storage: 100M tracks x (5 encodings x ~5 MB avg) ≈ 2.5 PB. Tiny compared to video.
- Listening events: ~7B plays x ~5 events (start, progress, pause, skip, end) ≈ 35B events/day ≈ 400k events/sec, ~1 KB each ≈ 35 TB/day raw.
- Playlists: 4B+ playlists; items per playlist avg ~50; heavy read, light write.
- Search: ~100k QPS peak.

> [!NOTE]
> Egress cost and cache hit ratio drive the business. A 1% improvement in CDN/peer cache hit ratio is worth millions per year. Spotify historically also used peer-to-peer for desktop to offload servers (retired in 2014 once CDN economics improved).

### API Design

```
GET  /v1/tracks/{id}                      // metadata
GET  /v1/tracks/{id}/stream-info          // → { manifest_url | file_urls[], format, bitrate, license_token, expires }
GET  /v1/search?q=...&type=track,artist,album,playlist,podcast
GET  /v1/me/player                         // current playback state (cross-device)
PUT  /v1/me/player/play  { device_id, uris[], position_ms }
PUT  /v1/me/player/transfer { device_id }

POST /v1/playlists                         { name, public, collaborative }
POST /v1/playlists/{id}/tracks             { uris[], position?/after_id? }
DELETE /v1/playlists/{id}/tracks           { items[] }
GET  /v1/playlists/{id}?snapshot_id=...    // snapshot_id = optimistic concurrency token

POST /v1/events/playback                   // batched client events (analytics + royalties)
GET  /v1/me/recommendations/home
PUT  /v1/me/tracks  { ids[] }              // like
POST /v1/me/downloads { playlist_id }      // offline sync
```

- Stream URLs are **short-lived signed URLs** (CDN token) bound to user/region, not permanent.
- Playlist writes carry a `snapshot_id` (Spotify's actual design); server rejects or rebases when stale.

### Data Model

```
tracks(track_id, isrc, title, album_id, duration_ms, explicit, popularity, available_markets[],
       audio_files{format→file_id,bitrate}, rights_holders[])
albums(album_id, label, release_date, upc, cover_art_id)
artists(artist_id, name, genres[], followers)
track_artists(track_id, artist_id, role)

users(user_id, country, plan: FREE|PREMIUM|FAMILY, ...)
playlists(playlist_id, owner_id, name, collaborative, snapshot_id, updated_at)
playlist_items(playlist_id, item_id, track_id, rank_key, added_by, added_at)   -- ordered
user_library(user_id, track_id, added_at)                                      -- likes
listening_history(user_id, ts, track_id, ms_played, context, device)           -- append-only
follows(follower_id, followee_id, type)

playback_state(user_id, device_id, track_id, position_ms, ts)                  -- Connect
offline_licenses(user_id, device_id, track_id, expires_at)
stream_events(event_id, user_id, track_id, ms_played, ts, ...)                 -- royalty source
```

### High-Level Architecture

```
 Clients (mobile / desktop / web / TV / car / speaker)
    │ metadata, search, playlists (HTTPS/gRPC)       │ audio bytes            │ playback events
    ▼                                                 ▼                        ▼
 Edge / API Gateway (auth, rate limit, geo)       CDN (multi-vendor + own)   Event Gateway
    │                                                 ▲ (cache miss)             │
    ├─► Catalog Service ── Cassandra/Bigtable         │                          ▼
    ├─► Search Service ── Elasticsearch / Lucene   Origin Storage (GCS/S3)    Kafka / Pub/Sub
    ├─► Playlist Service ── Cassandra/MySQL+Redis       ▲                       │
    ├─► Library / Social Service                        │                       ├─► Streaming (Flink/Dataflow):
    ├─► Player/Connect Service (WebSocket "dealer")     │                       │    royalty counts, session state
    ├─► Recommendation Service (serves precomputed)     │                       ├─► Cassandra: history
    └─► Licensing / DRM Service                  Audio Ingest + Transcode       ├─► Data Lake (GCS/BigQuery)
                                                  (from labels / DDEX)          └─► Batch ML (Discover Weekly, etc.)
```

Separation of concerns:
- **Delivery path** (CDN) is stateless and ultra-hot.
- **Control path** (metadata, playlists, Connect) is small payload, high QPS.
- **Data path** (events → analytics/ML/royalties) is async and high throughput.

### Audio Ingest and Transcoding

Content arrives from labels/distributors via **DDEX** feeds (metadata XML + audio masters, WAV/FLAC) or from artists via distributors.

```
Label delivery (DDEX package: audio + artwork + metadata + rights/territories)
   ▼
Validation (ISRC/UPC, audio QC: clipping, silence, loudness, duplicates via audio fingerprint)
   ▼
Transcode workers (queue → autoscaled pool; FFmpeg/libvorbis/AAC encoder)
   ├─ Ogg Vorbis 96 kbps (low / data saver)
   ├─ Ogg Vorbis 160 kbps (normal)
   ├─ Ogg Vorbis 320 kbps (Premium high)
   ├─ AAC 128/256 kbps (web, Safari, some devices)
   └─ (HE-AAC 24 kbps for very-low bandwidth; FLAC lossless tier launched 2025)
   ▼
Loudness normalization metadata (ReplayGain/LUFS), waveform & fingerprint, chunk index
   ▼
Object storage (immutable, content-addressed file_id)  →  CDN origin
   ▼
Catalog update + markets/availability flags → searchable
```

Notes:
- Encode **once per format**, store forever. 5 encodings x 100M tracks is only a few PB.
- Files are **immutable and content-addressed** (`file_id = hash`); re-encode creates a new id, old cached copies stay valid.
- Loudness normalization avoids volume jumps between tracks; store a gain value, apply in the client (no re-encode).
- Fingerprinting (Chromaprint-like) deduplicates uploads and powers content ID/copyright matching.

### Streaming: Chunked, Progressive, CDN, Prefetch

Unlike video, audio does not need heavy adaptive bitrate ladders per segment. Two common approaches:

| Approach | Used by | Notes |
|----------|---------|-------|
| **Progressive download of whole file with HTTP Range requests** (file split in ~64-512 KB chunks) | Spotify (historically: 128 KB-ish chunks, small initial chunk) | Simple, caching-friendly, fast start by fetching the first bytes immediately |
| **HLS / DASH segmented ABR** (2-10 s segments, several bitrates) | Apple Music, YouTube Music, many web players | Native ABR on network changes, more manifests/requests |

Spotify-style playback:
1. Client asks `stream-info`; gets signed URL(s) and DRM key (below).
2. Fetches **first chunk (~first few hundred KB) with a Range request** and starts decoding immediately.
3. Keeps downloading ahead into a buffer of ~30-60 s; rate adapts to network.
4. **Bitrate switching**: chosen at track start from user setting + measured bandwidth; in-track switching is rare (switch at next track).
5. Chunks are plain HTTP, so **any CDN edge can cache them**; hot tracks are almost entirely cache hits.

**Play-start latency budget (target < 300 ms)**:
```
tap → resolve track + license (cached) ........... 20-40 ms
      DNS/TLS (connection reuse, HTTP/2/3) ........ 0-50 ms
      first chunk from CDN edge (~100-300 KB) ..... 50-150 ms
      decode + start output ....................... 20-40 ms
```
Techniques:
- **Prefetch next track**: when playing track N, start fetching the first chunk(s) of N+1 (from queue/shuffle/Autoplay prediction) at ~70-80% of N, so skip/next is instant.
- **Predictive prefetch of first seconds** for tracks in a freshly opened playlist or Discover Weekly (first 10-20 s of top few tracks).
- **Warm connections** (HTTP/2 or QUIC kept alive), TLS 1.3 0-RTT.
- **Cache first chunks of popular tracks in device storage** (the "cache"/offline-lite) and at edge.
- Use **smaller first chunk** and larger subsequent chunks.
- Edge placement: multi-CDN with per-region steering (Akamai/Fastly/Google CDN, own PoPs); mid-tier cache shield to protect origin.
- **Pre-warm CDN** for new releases (Friday release day, major album drops, Taylor Swift effect): push to edges before release time.

**CDN cost control**:
- Long-tail catalog vs. head: ~top 1-5% tracks account for ~50-80% of streams, so CDN caches the head; the tail comes from origin or regional caches.
- Multi-CDN price arbitrage and steering; peering/ISP-embedded caches (Netflix Open Connect style) at very large scale.
- Default bitrate sensible (160 kbps) rather than 320; higher quality for Premium/WiFi only.
- Client-side cache of recently played tracks (reuses bytes on repeat plays; Spotify caches up to GBs).
- Efficient codec: Opus/HE-AAC at lower bitrates for cellular.
- Historic P2P among desktop clients (until 2014) reduced server bytes to ~10%.

### DRM and Licensing

Reality: music is licensed with territorial rights; the service must enforce **who can play what, where**.

- **Territorial availability**: `available_markets[]` per track checked at `stream-info` using user's account country (not just IP; IP used for fraud/consistency checks, 14-day travel rule).
- **Encryption**: audio files encrypted at rest (AES-128-CTR streaming cipher in Spotify's case) or via CENC/Widevine/FairPlay/PlayReady on web/devices. A **content key** is released per `(user, track)` by the license service after entitlement checks.
- **Entitlement**: Free vs Premium (bitrate, offline, on-demand vs shuffle only).
- **License token**: signed JWT carrying `user_id, track_id, plan, device_id, expiry`; validated at CDN or key server.
- **Offline licenses** are time-bounded (e.g., must go online every 30 days) and device-bound; on expiry, downloads become unplayable until re-validated.
- Anti-piracy (stream ripping) is mitigated by encryption + per-device keys + rate/anomaly detection, but analog hole cannot be fully closed.

### Playlists

Requirements: ordered list of tracks; millions of reads/sec; edits by owner and collaborators; reorder with drag-and-drop; Spotify scale (4B+ playlists).

**Storage**: key-value/wide-column partitioned by `playlist_id`: `playlist_items` rows clustered by order key. Reads are "give me the full list or a page" → single-partition read. (Spotify historically used MySQL/Postgres + Cassandra; the playlist service originally used a snapshot/revision model.)

**Ordering - the core design question**:

| Approach | Insert in middle | Pros | Cons |
|----------|------------------|------|------|
| Integer position (1..n) | Renumber O(n) | Simple | Many writes, conflict-prone |
| Linked list (next pointer) | O(1) update 2 rows | Cheap insert | O(n) traversal, hard to page |
| **Fractional indexing** (rank = midpoint string/float) | O(1) single row | Single write, easy sort/paging | Keys grow; periodic rebalance needed |
| CRDT sequence (RGA / LSEQ / Logoot) | O(1), commutative | True multi-writer merge | Metadata overhead, tombstones |

**Fractional indexing**: each item has `rank_key` (lexicographic string, e.g., `"a0"`, `"a1"`). To insert between `"a0"` and `"a1"` generate `"a0V"` (midpoint). No other rows change. Sorting by `rank_key` yields order. When keys get long (hundreds of chars after many insertions at the same spot), **rebalance** the playlist in the background (rewrite keys evenly, bump `snapshot_id`).

```
items:  [A:"a0"] [B:"a1"] [C:"a2"]
move C between A and B → C.rank = mid("a0","a1") = "a0V"   // one UPDATE
result: A("a0"), C("a0V"), B("a1")
```

**Collaborative editing**:
- Concurrent edits from multiple users on the same playlist: **OT (operational transform)** with a central server ordering ops, or **CRDT** for offline/decentralized merge.
- Practical choice: **server as sequencer with optimistic concurrency**. Each edit sends `base_snapshot_id`; server applies ops (insert after item id X, remove item id Y, move) **by item identity, not by index**, so concurrent edits rarely conflict. If `base_snapshot_id` is stale, server **rebases** the op against the intervening ops (simple OT: ops keyed by stable item ids) and returns new snapshot + delta. Clients get live updates via WebSocket push.
- Deletes are tombstoned (`removed_at`) so a concurrent "insert after X" where X was deleted still resolves (fall back to nearest surviving neighbor or end).
- **Offline edits** (mobile): queue ops locally with item-id anchors; replay on reconnect; CRDT (RGA) gives automatic convergence if you need guaranteed offline merge. Most deployments pick OT-with-server for simplicity and use CRDT only for offline-heavy collaboration.

**Read path**: playlist page ← cache (Redis/Memcached) keyed by `(playlist_id, snapshot_id)`; immutable snapshots make caching trivial and CDN-able for public playlists. Followers count and metadata separately cached.

**Large/popular playlists** (Today's Top Hits, 30M+ followers): hot partitions. Replicate to cache layer and CDN; writes are rare (editorial). Follower list is not stored in the playlist partition; use counters + sharded follower index.

### Search

- Index: tracks, albums, artists, playlists, podcasts, users. Sizes: ~100M tracks, mostly small docs.
- **Elasticsearch/OpenSearch (Lucene)** sharded by hash; or in-house engine. Fields analyzed for prefix, n-gram, phonetic (misspellings, "Beyonce" vs "Beyoncé"), multilingual tokenization.
- **Ranking**: text relevance + popularity + personalization (user's taste, language, recent plays) + freshness. Learning-to-rank model rescoring the top ~200 hits.
- **Typeahead / autocomplete**: prefix index (completion suggester / trie) over popular queries and entities; latency budget < 50 ms; cache top prefixes.
- **Query understanding**: detect intent (artist, song, lyrics, mood "chill playlist"), voice queries (ASR → NLU → search), entity linking.
- Index freshness: new releases are indexed within minutes via CDC from the catalog (Kafka → indexer).
- Market filter: only show content available in the user's country.

### Listening History (Kafka → Cassandra)

```
Client batches events ──► Event Gateway ──► Kafka "playback-events" (partition by user_id)
                                             │
                    ┌────────────────────────┼─────────────────────────────────┐
                    ▼                        ▼                                 ▼
          History Writer             Streaming jobs (Flink)             Data lake (hourly/daily)
        (Cassandra: user_id,        - session detection                 - ML features
         ts DESC → track)            - royalty counter (30 s rule)      - batch recs
                                     - "recently played", Now Playing
                                     - fraud/bot detection
```

- **Cassandra** schema: `PRIMARY KEY ((user_id), ts DESC)`; one partition per user, latest first, TTL or tiering for old rows. Fast "recently played" (page of 50) and "last N" queries. Writes ~400k/sec spread across the cluster.
- Consistency: LOCAL_QUORUM writes; history is eventually consistent across regions.
- Why Kafka: decouples ingestion from consumers; replay when a downstream job has a bug; multiple consumers (history, recs, royalties).
- Client batching: store events locally and flush every ~30 s or on app background; handle offline (flush later with original timestamps).
- **Exactly-once effects**: each event carries a client-generated `event_id`; consumers dedupe (idempotent upsert in Cassandra with `event_id` or `(user, ts, track)` as key; Flink with checkpointed state / Kafka transactions).
- Privacy: private session mode skips history and recommendations signals (but still counts for royalties).

### Recommendations (Discover Weekly)

Three families of signals:
1. **Collaborative filtering** (users who listen to similar things): matrix factorization (ALS) or implicit-feedback models on the user-track play matrix (hundreds of millions x 100M tracks, extremely sparse) producing **user and track embeddings** (e.g., 100-300 dims).
2. **Content-based**: audio features from raw audio (CNN on spectrograms: tempo, key, timbre, danceability), lyrics NLP, and metadata. Solves **cold start** for new tracks.
3. **Text/NLP signals**: blog posts, playlist titles, web mentions (Echo Nest heritage) feeding "cultural vectors".

**Discover Weekly pipeline** (batch, weekly, personalized 30 songs per user):
```
Listening logs (data lake) ──► Feature + embedding training (Spark/Beam/TensorFlow)
    ▼
User taste vectors + track vectors (2-tower or matrix factorization)
    ▼
Candidate generation: ANN search (ScaNN/Annoy/FAISS/HNSW) for nearest tracks to the user vector,
    excluding tracks the user already played/knows
    ▼
Ranking model: predicts engagement (listen-through, save) with context features
    ▼
Diversity / business filters: artist cap, language, explicit, freshness, fairness for new artists
    ▼
Write to user's "DW" shard (key-value) on Sunday night; serve Monday morning (reads only)
```

- **Precompute for the heavy tier** (home shelves, Discover Weekly, Release Radar, Daily Mix) so serving is a key-value read. Use **real-time rerank** (contextual bandits) for the last mile: time of day, device, current session, recent skips.
- **Feedback loop**: skips, saves, repeats, adds-to-playlist, listen-through % are implicit labels. Beware popularity bias and filter bubbles; use exploration (bandits, 5-10% exploratory slots).
- **Cold start**:
  - *New user*: onboarding artist picks, country/language defaults, popular local charts, demographic/device priors; learn fast from first 10 plays.
  - *New track/artist*: content embeddings from audio + metadata, similar-artist links, editorial placement, Release Radar to followers, and an exploration budget.
- **Radio/Autoplay**: seed → nearest neighbors in embedding space + session-aware sequence model (RNN/Transformer next-track prediction).
- Serving: Recommendation Service reads cached shelves; fallback to popular-in-region if the personalized key is missing.
- Offline evaluation (NDCG, hit-rate) + online A/B testing (retention, hours listened, skip-rate).

### Offline Mode

- User marks a playlist/album "Download" → client requests `offline_licenses` for each track (device-bound, expiring), downloads **encrypted files** to local storage in chosen quality.
- Files encrypted with a key wrapped by a device key; cannot be copied to other devices.
- **Periodic check-in**: must connect at least every 30 days to renew licenses; revoke when subscription lapses (check on next online).
- **Sync**: server tracks device download set (max 5 devices x 10k tracks); changes to a playlist while online propagate and trigger delta download; removed tracks get cleaned up.
- **Offline play counting**: events (with signed timestamps) are queued on-device and uploaded on reconnect; server accepts events with plausible timestamps (within license window) and dedupes by event_id. Royalty accuracy still holds because payouts are monthly.
- Storage management: LRU cache for "streamed" cache vs pinned downloads; background download only on WiFi/charging if configured.
- Edge case: track removed from catalog / market restriction changes → license revoked next check-in.

### Royalty and Play Counting (The 30-Second Rule)

A **stream counts as a paid play only after 30 seconds** of listening (Spotify rule; ~5 s for sub-playback on some partners; podcasts differ). This drives **royalty payouts to labels and artists**.

Why it needs rigor: payouts are ~$thousands of millions/year; mistakes are legally and financially serious.

- Pro-rata model: `payout_to_rights_holder = (their_streams / total_streams) x revenue_pool` per market per month (alternative user-centric model: each user's subscription fee is split among artists that user listened to).
- Pipeline: raw playback events → streaming validation → **royalty-eligible stream** records → daily/monthly aggregate by `(track, market, plan, rights_holder)`.
- Events include `ms_played`, `start_ts`, `end_ts`, `context`, `device`, `offline` flag, `private session`.
- **Rules**: `ms_played >= 30000`; repeat loops allowed; seeking forward doesn't add played ms (use played time, not position); exclude detected fraud/bot streams.
- **Dedup**: stable `event_id`; streaming job with exactly-once state; **batch reconcile** from the immutable data lake (the **lambda/kappa** pattern: streaming for fast dashboards, batch for authoritative payout).
- **Audit**: keep raw events immutable (S3/GCS, years); payout reports reproducible by re-running the batch with the same inputs. Labels get monthly statements and can audit.
- **Fraud**: streaming farms (bots playing tracks 31 s on loop), account sharing, VPN country fraud. Detect via per-user behavior (24/7 play, unusual repeat patterns), per-device clusters, ML; remove or demonetize flagged streams; **minimum thresholds** (e.g., 1,000 streams/year) before payout.
- **Display counts** (public play count under the track) are separate: eventually-consistent counters (sharded counters / approximate), not payout-authoritative.

### Podcasts

- Two models: **hosted** (creators upload via Spotify for Podcasters) and **RSS ingest** (crawl feeds periodically with conditional GET, parse, download enclosure audio, transcode).
- Episodes are long (30-180 min): use **segmented/ranged progressive** with a **resume position** stored per user (`episode_position(user_id, episode_id, ms)`), synced across devices.
- **Dynamic ad insertion (DAI)**: stitch ads into the stream server-side (manifest manipulation) so each listener hears personalized ads while the creator's file is untouched; counts impressions for advertisers.
- Chapters, transcripts (ASR), search over transcripts; new-episode notifications via push fan-out to followers.
- Royalties differ: ad revenue share/subscription-based exclusive deals rather than pro-rata per-stream.
- Video podcasts: reuse video pipeline (HLS) for the video episodes.

### Social Features

- **Follow graph** (user→user, user→artist, user→playlist): store as adjacency lists in Cassandra/Graph store; counters separately.
- **Friend Activity** feed: show what friends are listening to now. Use last-played-event per user (latest-wins) published to followers via pub/sub; limit to ~N friends; opt-in; hide in private session.
- Sharing links, collaborative playlists, Blend (merged taste), Wrapped (annual batch analytics; generated from the data lake with heavy cache and CDN-served assets).
- Notifications (new releases from followed artists): fan-out on write for artists with small followings, **fan-out on read / batched pull** for mega-artists (same hybrid as news feed).

### Free Tier and Ads

- Free users: shuffle-only on mobile, ads every few tracks, 160 kbps cap, limited skips.
- **Ad decisioning**: Ad Server receives `(user, context, device, country, track genre)`; auction/direct-sold selection under frequency caps and brand-safety constraints (no alcohol ads for minors, no competitor ads adjacent).
- **Audio ads** are short audio files served like tracks (pre-fetched during the previous song to guarantee no gap); also display/video takeover ads.
- **Impression tracking**: client sends beacon at ad start/25/50/75/100% completion; server dedupes; advertiser billing needs accuracy → same event pipeline with strict dedup and audit.
- Eligibility checks enforced **server-side** at `stream-info` (don't trust the client to play ads); content/skip limits enforced by license service.
- Targeting uses a privacy-safe feature store (age bucket, country, inferred interests); consent management (GDPR, CCPA).

### Spotify Connect and Cross-Device Playback

Goal: control playback on any device (phone as remote for speaker); playback state is **server-authoritative**, and the audio streams **directly from CDN to the target device** (not through the phone).

```
Phone (controller) ──WS "dealer"──► Connect Service ──push──► Speaker (player device)
                                         │                        │
                                  playback_state(user)            └─► CDN (audio bytes)
                              (current device, track, position,
                               queue, shuffle, volume, ts)
```

- Every device keeps a persistent WebSocket (Spotify calls it "dealer") to the Connect service, registered under `user_id`.
- `transfer(device_id)`: server updates the "active device"; sends a command to the target with `(context, track, position_ms, timestamp)`; target fetches audio and seeks; old device is told to pause. Position is extrapolated using `server_ts` so devices agree within ~100 ms.
- State store: Redis or in-memory per-user actor sharded by `user_id` (consistent hashing); persisted occasionally for crash recovery. One "active device" invariant per user avoids two devices playing.
- Commands are **idempotent with sequence numbers** (`state_version`); stale commands are ignored.
- Local network discovery (mDNS/Zeroconf) allows direct control for LAN speakers; cloud path is the fallback.
- **Group sessions / Jam**: shared queue (collaborative playlist semantics) with a session id; multiple controllers, one or more players.
- Handles device going offline: heartbeat timeout (~30-60 s) → mark inactive; last known position used for resume.

### Gapless Playback and Crossfade

- **Gapless**: encoders (MP3) add padding; Vorbis/AAC with explicit **encoder delay and padding metadata** let decoders trim exactly. Client **pre-buffers next track** (decoded PCM ahead) and splices sample-accurately at the boundary. Live albums/DJ mixes depend on it.
- **Crossfade**: need audio of both tracks overlapping for N seconds (0-12 s). Client starts decoding track N+1 N seconds early and mixes with volume ramps (equal-power curves) in the audio output stage. Requires prefetch of next track's first N seconds earlier than for gapless; prefetch timing is aware of crossfade setting.
- **Loudness normalization** (target ~ -14 LUFS) applied per track gain so crossfades don't jump volume.
- Skipping during crossfade, seeking, and repeat-one all need state-machine care in the player (track A fading out while B ramps in; cancel B if user hits back).

### Scale and Global Deployment

- **Regions**: metadata/Playlist/Connect services deployed in multiple regions (Spotify runs on Google Cloud); users routed to nearest region by GeoDNS/Anycast.
- **Data**: Cassandra multi-DC for user data (history, library, follows) with LOCAL_QUORUM; Playlist data homed in the user's region with async cross-region replication; catalog is read-mostly and replicated everywhere (read replica / cache) since writes come from a single ingest pipeline.
- **Cache layers**: client cache → CDN → regional cache → origin object store; metadata cached at API edge (track metadata is immutable-ish with TTL).
- **Service architecture**: microservices (1000+ at Spotify), gRPC internally, backstage developer portal; squads own services. In the interview mention domain services, not all.
- **Release-day spikes** (Friday new music, album drops): pre-warm caches and metadata, scale search/index ahead of time, rate-limit non-critical jobs.
- **Capacity planning** by time zone: usage follows daylight and commute patterns; autoscale compute, but CDN capacity is contract-based, planned months ahead.

### Storage Choices

| Data | Store | Why |
|------|-------|-----|
| Audio files (all encodings) | Object storage (GCS/S3), immutable | Cheap, durable (11 9s), CDN origin |
| Audio delivery | Multi-CDN + edge caches | Cost and latency; head-tail caching |
| Track/album/artist metadata | Cassandra/Bigtable + Redis/Memcached cache | Read-heavy, key lookups, replicated globally |
| Search index | Elasticsearch/OpenSearch | Full-text, prefix, ranking |
| Playlists | Cassandra (or MySQL/Postgres + cache), clustered by rank_key | Single-partition reads, ordered |
| Listening history | Cassandra `(user_id, ts DESC)` + Kafka | Write-heavy time-series per user |
| User library / follows | Cassandra / graph-ish adjacency lists | Partition by user |
| Playback/Connect state | Redis / in-memory actor per user | Low-latency, ephemeral |
| Royalty events and ledger | Kafka → data lake (GCS/Parquet) → BigQuery; payout ledger in Postgres | Immutable raw + auditable relational payouts |
| ML features and embeddings | Feature store (Bigtable/Redis) + vector index (ANN) | Low-latency serving |
| Batch recommendations output | Key-value store (Cassandra/Bigtable) | Written weekly, read at home load |
| Ads and billing | Relational (Postgres) + event stream | Accuracy |
| Analytics / A/B | BigQuery / Druid / ClickHouse | Ad hoc + dashboards |

### Failure Modes

| Failure | Handling |
|---------|---------|
| CDN provider outage / region degradation | Multi-CDN steering by health; client retries alternate host; fall back to origin with throttling |
| Origin storage slow (cache stampede on release day) | Origin shield, request coalescing, pre-warm new releases |
| Network drops mid-track | Buffered ahead 30-60 s; resume with Range from last byte; cache partial |
| Playlist concurrent edit conflict | Snapshot id + rebase by item id; return merged result; client retries |
| Kafka lag / downstream failure | Events durable; consumers replay; history delayed but not lost |
| Duplicate / lost playback events | Client event_id dedupe; at-least-once + idempotent writers; batch reconciliation for payouts |
| Recommendation job fails | Serve last week's / popular-by-region fallback; alert; rerun partial |
| Connect service node dies | Sticky sharding with failover; devices reconnect and re-register; state reloaded from store |
| License/DRM service down | Cache entitlements briefly; allow already-authorized streams; offline continues; alert |
| Search index lag | Serve stale; reindex via CDC; fallback to catalog lookup by id |
| Clock skew on client (offline abuse) | Server validates timestamps; license expiry uses server-trusted time and monotonic checks |
| Bot / streaming fraud | Stream-level fraud scoring; demonetize; account bans |
| Hot partition (viral track/playlist) | Cache + replicate; sharded counters; request coalescing |

### Privacy and Compliance

- GDPR/CCPA: data export (`/me/export` of history), deletion within 30 days (Kafka compaction + lake delete jobs by user_id), consent for ads personalization.
- Private session: no history, no tasteprofile updates, hidden from friend activity; still counted for royalties.
- Children: Kids/Family plans with explicit-content filters, no personalized ads for under-13.
- Public playlists can leak taste; default settings, "make secret" option.
- Anonymize data used for ML (pseudonymous user ids); strict access controls.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Treating audio like video with full ABR ladder | Small files: progressive Range download + a few bitrates; start with first chunk |
| Serving audio through app servers | Signed URLs to CDN; app servers only do control plane |
| Integer position for playlist order | Fractional index (or CRDT/OT), single-row inserts |
| Playlist edits by index | Edit by stable item id with snapshot-based rebase |
| Counting plays from client `play` call | Server-side validated `ms_played >= 30 s`, deduped, audited |
| One store for history, royalties, and display counts | Separate: Cassandra history, immutable lake for payouts, approximate counters for display |
| Online-only recommendations from scratch | Batch precompute heavy lifting; light real-time rerank |
| Ignoring cold start | Content-based audio embeddings, onboarding, exploration slots |
| Forgetting licensing/geo | Market-level availability checks and DRM keys per entitlement |
| No prefetch of next track | Prefetch next first chunk; crossfade needs earlier fetch |
| Offline without license expiry | Device-bound keys with periodic check-in |
| Trusting the client for ads/skip limits | Enforce in license and stream-info on server |

> [!NOTE]
> Spotify's moat is not bandwidth, it is the **data flywheel**: play events → embeddings → Discover Weekly → more listening. And its biggest cost is **CDN egress plus royalties**. Mention both; senior interviewers listen for cost awareness.

### Interview Follow-ups

- *"How do you get play-start under 300 ms?"* — Immutable files on CDN, signed URL, Range request for first ~100-300 KB, warm HTTP/2 or QUIC connection, cached track metadata and licenses, prefetch of next track and first seconds of tracks in the opened context, small first chunk.
- *"Why not HLS/DASH like video?"* — Audio files are small; progressive Range download is simpler and cache-friendly. HLS/DASH is fine and gives native ABR; used by Apple Music and web players. Choice is a trade-off of simplicity vs ABR flexibility.
- *"Which codecs and bitrates?"* — Ogg Vorbis 96/160/320 kbps (Spotify), AAC 128/256 for web/Apple devices, optional lossless FLAC, HE-AAC/Opus for low bandwidth. Choose by device codec support and perceived quality per bit.
- *"How do you deal with a new album drop at midnight?"* — Pre-ingest and encrypt early, embargo availability until release time (flag flip `available_from`), pre-warm CDN edges, scale metadata and search ahead, throttle batch jobs.
- *"How do you cut CDN cost?"* — Cache hit ratio (head/tail), multi-CDN pricing, ISP peering/embedded caches, sensible default bitrate, client-side caching, efficient codecs, avoid wasteful prefetch (prefetch only first chunk, cancel if skipped).
- *"How is DRM enforced?"* — Encrypted files; content key issued per (user, track, device) after entitlement check; token-signed URLs; offline licenses are device-bound and expiring; platform DRM (Widevine/FairPlay) on web/TV.
- *"How do you enforce territorial restrictions?"* — Account country + `available_markets` on track; checked at `stream-info`; IP geolocation as a secondary signal; tracks gray out and disappear from search in unavailable markets.
- *"How do collaborative playlists work with concurrent edits?"* — Edits by item id relative anchors, server as sequencer with snapshot-based optimistic concurrency and rebase (OT-lite), push updates over WebSocket; CRDT if offline merge is essential.
- *"OT vs CRDT?"* — OT needs a central server to transform ops and has simpler data; CRDT merges without a coordinator and supports offline peers but carries per-element metadata and tombstones. For a server-centric app like playlists OT/sequencer is simpler; CRDT shines for offline-first.
- *"How does fractional indexing fail?"* — Repeated inserts at the same spot lengthen keys; two clients may generate identical keys concurrently (add random jitter/client-id suffix to break ties); fix by periodic rebalance and tiebreak on `(rank_key, item_id)`.
- *"How do you store playlist order if you reorder often?"* — Update one row's rank key, not N rows. Keep a `snapshot_id` incremented per change for cache keys and optimistic checks.
- *"How do you fetch a playlist with 10,000 tracks?"* — Paginate by `(rank_key, item_id)` cursor; hydrate track metadata via batched cache lookups (multi-get); clients render lazily.
- *"What does 'following a playlist' cost for a 30M-follower playlist?"* — Follower relation stored separately from playlist rows; counter is a sharded/approximate counter; never write per-follower rows into the playlist partition; push notifications for updates via batched fan-out.
- *"How do you design search?"* — Lucene-based sharded index with prefix/phonetic analyzers, popularity and personalization features, learning-to-rank rescoring, autocomplete trie, market filter, CDC from catalog to indexer for freshness.
- *"How do you handle misspellings and multilingual queries?"* — Edit-distance fuzzy matching, n-gram analyzers, transliteration, query rewriting from logs (did-you-mean), and language detection per query.
- *"Why Cassandra for history?"* — Massive write volume, partition per user with time-clustering, linear scale-out, multi-DC replication, no joins needed. Trade-off: eventual consistency and limited ad hoc queries (use the lake for analytics).
- *"How do you build the 'recently played' list?"* — Read the user's Cassandra partition top-N; dedupe by track/context; cache briefly; update from stream immediately for the local device.
- *"How does Discover Weekly work?"* — Offline pipeline on listening logs: collaborative filtering embeddings plus audio/NLP content features, ANN candidate generation excluding known tracks, ranking and diversity filters, results written to a key-value store weekly and just served on Monday.
- *"Why precompute rather than compute at request time?"* — Candidate generation over 100M tracks is expensive; weekly cadence is acceptable; serving becomes a key read. Real-time rerank handles freshness for the final page.
- *"How do you handle cold start for a new user?"* — Onboarding artist picks, locale/language defaults, regional charts, device/time priors, quick learning from first plays with high exploration rate.
- *"Cold start for a new track?"* — Content embeddings from audio and metadata, similar-artist links, Release Radar to followers, editorial placement, exploration quota to collect early engagement signals.
- *"How do you avoid filter bubbles and popularity bias?"* — Exploration slots (bandits), diversity constraints, artist caps, novelty re-ranking, boosting under-exposed content, and measuring long-term retention not just clicks.
- *"What counts as a stream for royalties?"* — At least 30 seconds of actual playback (played ms, not position), not fraudulent, in a licensed market, deduped; paid by pro-rata share of the revenue pool per market per rights holder.
- *"How do you make play counting exactly-once?"* — At-least-once events with client `event_id`, idempotent consumers, streaming for fast views, and batch recomputation from the immutable lake as the source of truth for payout.
- *"How do you prevent stream fraud?"* — Behavioral anomaly detection (24/7 playing, 31-second loops, repeated tiny catalogs), device and IP clustering, minimum payout thresholds, demonetizing flagged streams, account sanctions, label reporting.
- *"How does offline mode stay secure?"* — Encrypted downloads bound to a device key, license expiry (about 30 days), periodic check-in, revocation on subscription end, signed play-event queue for later upload.
- *"What if a user's offline clock is changed to extend the license?"* — Use monotonic counters and last-seen server time stored securely; detect backwards jumps and require online refresh.
- *"How does Spotify Connect work?"* — Devices hold persistent WebSockets to a Connect service; playback state is server-held per user; transfer sends a command to the target device that then pulls audio directly from CDN; sequence numbers make commands idempotent; only one active device per user.
- *"How do you keep devices in sync on position?"* — Server timestamps with `(position_ms, server_ts)` and clients extrapolate; periodic state reports; seek commands overwrite.
- *"How do gapless and crossfade work?"* — Encoder-delay metadata for sample-accurate trimming, prefetched and decoded next track, mixing in the output stage with equal-power ramps, earlier prefetch for crossfade windows, normalized loudness to avoid jumps.
- *"How do you implement skip limits for free users?"* — Server-side counter per user per hour in Redis, checked when the client requests the next stream-info; client UI mirrors but is not trusted.
- *"How do ads get inserted without gaps?"* — Prefetch the ad audio during the preceding track; ad server decisions pre-computed; beacons confirm impressions; for podcasts use server-side dynamic ad insertion.
- *"How are podcasts different?"* — Long audio with resume position sync, RSS ingestion, dynamic ad insertion, chapters/transcripts, different royalty/ad-revenue model, and video variants.
- *"How do you handle friend activity at scale?"* — Latest-played event per user, pub/sub to online followers, bounded friend list, caching; opt-in and hidden in private mode; avoid fan-out to mega-followers.
- *"How do you scale to 600M users globally?"* — Region-local services, Cassandra multi-DC, CDN for bytes, caches for metadata, Kafka for events, autoscaling stateless services, and precomputed ML outputs; plan capacity around release days and time zones.
- *"What are the main cost drivers?"* — CDN egress, royalties (the largest by far), compute for ML/batch, and storage (small). Optimize egress via hit ratio and bitrate policy; royalties are contractual.
- *"How would you do A/B testing for recommendations?"* — Deterministic hashing of user into experiment buckets, metrics from the event pipeline (skip rate, streams per user, retention), guardrail metrics, and long-running holdouts.
- *"How would you add lossless / spatial audio?"* — New encodings in the transcode pipeline (FLAC/ALAC, Dolby Atmos stems), larger files (about 5-6x) so bandwidth and CDN cost rise; entitlement for the tier, device capability negotiation, higher prefetch buffers.
- *"How do you handle the removal of a track (takedown)?"* — Flip availability flag in catalog; propagate through CDC to search and caches; invalidate playlist item hydration; revoke offline licenses at next check-in; keep playlist items as greyed-out placeholders.
- *"How do you ensure metadata correctness (duplicate tracks, wrong artist)?"* — ISRC/UPC identifiers, audio fingerprint dedupe, merge tooling for catalog ops, label-provided metadata via DDEX, human review for conflicts.
- *"How would you design lyrics sync?"* — Time-coded lyrics (LRC-like) from provider, served by id and fetched alongside playback; client highlights by current position; cached on CDN.
