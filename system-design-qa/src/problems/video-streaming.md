# Q: Design a video streaming service (YouTube / Netflix).

**Answer:**

Combines **massive storage**, **adaptive bitrate streaming**, **global CDN**, and **content workflow**. The interview focuses on the **delivery pipeline**, not the user feed.

### Requirements

**Functional**:
- Upload videos.
- Transcode to multiple resolutions/bitrates.
- Stream playback (adaptive bitrate).
- Subtitles, multiple audio tracks.
- Likes/views/recommendations (touch lightly).

**Non-functional**:
- Petabyte-scale storage.
- Tens of Gbps egress per popular video.
- p99 startup < 2 s; rebuffer < 1% of viewing time.
- Multi-region delivery.

### Capacity

YouTube-style:
- 500 hours/min uploaded.
- 1 hour 4K ≈ 7 GB raw; transcoded to multiple variants ≈ 15 GB total per source hour.
- 500h/min × 60 min × 15 GB = 450 TB ingested/min.

Egress dwarfs ingest. Bytes flowing out: 100× upload.

### Pipeline Overview

```
Upload  → Object Storage (raw)  → Transcode Pipeline → Object Storage (chunks + manifests)
                                                          │
                                                          ▼
                                              CDN (edge cache)
                                                          │
                                                          ▼
                                                  Player (HLS / DASH)
```

### Upload

- Client → presigned URL → S3 (or GCS, Azure Blob).
- Multipart upload for large files (resumable).
- On complete: enqueue transcode job.

```
POST /upload/init → { upload_id, presigned_urls[] }
PUT presigned_url (each chunk)
POST /upload/complete → enqueue("transcode", upload_id)
```

Object storage handles durability (11 9s); we don't manage that.

### Transcoding

For each upload, produce multiple representations:

```
1080p / 5 Mbps  ┐
720p  / 2.5 Mbps│
480p  / 1 Mbps  │ HLS or DASH variants
360p  / 600 Kbps│
240p  / 300 Kbps┘
audio (multiple bitrates)
subtitles (VTT)
```

Each representation chunked into ~2–6s segments. Manifest (`m3u8` for HLS, `mpd` for DASH) lists chunks.

Transcoding is CPU-heavy. Use:
- Workers reading from queue, writing to object storage.
- GPU encoders (NVENC) for cost/speed.
- Cloud-managed: AWS MediaConvert, GCP Transcoder, Mux.

Per-video transcode cost ~1× video duration on GPU, ~10× on CPU. Plan capacity by upload rate.

### Adaptive Bitrate (ABR) Streaming

Player monitors bandwidth + buffer; switches representation on the fly:

```
network slow → switch to 480p
network fast → switch to 1080p
```

HLS / DASH define the protocol; player libraries (hls.js, Shaka) implement.

### Delivery via CDN

```
client ──► nearest CDN PoP ──► origin shield ──► object storage
```

Most chunks served from CDN — origin sees ~1% of traffic.

Cache TTL:
- Chunks: weeks/months (immutable; ID-based).
- Manifests: short (~minutes); changes on live or DVR re-segmenting.

Open-source/cheap CDN options: Cloudflare Stream, AWS CloudFront, Akamai. Tier-1 video providers (Netflix) run their own CDN appliances at ISPs (Open Connect).

### Storage Tiering

```
hot (90% of views, recent uploads):   SSD / S3 standard
warm (older but still watched):       S3 Infrequent Access
cold (long-tail, archive):            S3 Glacier
deleted-by-DMCA / takedown:           tombstone for legal compliance
```

Move tiers automatically based on access frequency.

### Metadata

```
videos (Postgres):
  id, owner_id, title, description, duration, status (transcoding/ready),
  created_at, privacy, thumbnails_url, manifest_url

view_events (ClickHouse):
  video_id, user_id, watch_time, ts
```

Counts (views, likes) — denormalize into the videos row periodically. Don't try to keep exact-on-write counts at this scale.

### Recommendations / Feed

Out of scope for the streaming design — but the interviewer may ask:

- Offline pipeline: watch logs → ML embedding → candidate generation.
- Online: rerank candidates per user; serve via low-latency service.
- A/B testing infrastructure.

### Live Streaming Variant

Live = same pipeline, harder:

- Ingest via RTMP / WebRTC / SRT.
- Real-time transcoding (lower latency than VOD).
- Origin caches latest manifest aggressively (short TTL).
- Low-latency HLS / DASH (chunks of ~1s with shared chunk transfer encoding).

End-to-end latency: traditional HLS 30s; LL-HLS 3-6s; WebRTC sub-second.

### DRM / Encryption

Premium content:
- Encrypt chunks at rest (AES-128).
- License server: client requests key after auth check.
- Widevine (Chrome/Android), FairPlay (Apple), PlayReady (Edge/Xbox).

### Watermarking / Anti-Piracy

- Forensic watermark per session (slight visual perturbation traceable to user).
- Token-signed manifest URLs (expire in minutes).

### Failure Modes

| Failure | Handling |
|---------|---------|
| Transcode fails for one variant | Retry; alert; serve other variants |
| Origin overload during viral event | Origin shield + larger CDN TTL |
| Single PoP down | DNS routes around |
| Manifest corrupt | Roll back via versioned URL |
| Storage 5xx | CDN serves cached; retry origin |

### Scale Numbers

YouTube-class:
- Storage: > 1 EB total.
- Egress: hundreds of Tbps.
- Active streams: 10s of millions concurrent.

Infrastructure cost dominated by **egress** and **transcoding**.

### Optimization

- **CMAF**: shared chunk format for HLS + DASH; one set of files serves both.
- **Per-title encoding**: optimize bitrate per video (Netflix paper); animation needs less bitrate than action.
- **AV1 / HEVC**: newer codecs save 30–50% bandwidth at same quality.
- **TCP BBR**: better congestion control than CUBIC for streaming.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Single bitrate, no ABR | Mobile users buffer forever |
| Encoding on the request path | Pre-transcode; on-demand encoding only for long-tail |
| No CDN | Origin egress costs 10× CDN egress |
| Same DB for views + metadata | Views are append-heavy; split to ClickHouse |
| Synchronous transcode in upload handler | Async queue; client polls or webhooks |

> [!NOTE]
> Video is a **bandwidth problem disguised as a streaming problem**. The unique architecture is the pipeline that turns one upload into many representations + chunks + a manifest, then ships those bytes via CDN.

### Interview Follow-ups

- *"How does Netflix's Open Connect work?"* — Custom appliances deployed inside ISP networks; serve from there → no transit cost; ISP wins (less peering).
- *"How would you handle a sudden viral video?"* — CDN absorbs; pre-warm cache to many PoPs; origin shield prevents stampede.
- *"How would you support seeking to arbitrary timestamps?"* — Chunked architecture is inherently seek-friendly; client jumps to nearest segment.
