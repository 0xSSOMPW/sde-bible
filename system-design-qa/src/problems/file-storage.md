# Q: Design a file storage service (Dropbox / Google Drive).

**Answer:**

Tests **chunking, deduplication, sync, conflict resolution, sharing**. The hard parts are the **sync protocol** and **per-user-per-device state**.

### Requirements

**Functional**:
- Upload / download / delete files.
- Sync across devices.
- Share files / folders.
- Versioning.
- Offline access.
- Large files (multi-GB).

**Non-functional**:
- 500M users.
- p99 first-byte download < 500 ms.
- Resumable uploads.
- Dedup to save storage.

### High-Level Architecture

```
Client (desktop, mobile, web)
   │  block uploads + metadata sync
   ▼
Edge / CDN
   │
   ▼
Metadata Service  ◄──►  Sync Service  ◄──►  Notification Service
   │                          │
   ▼                          ▼
Postgres (metadata)     WebSocket / push
   │
   ▼
Block Storage (S3) — actual file content
   │
   ▼
Dedup Service (content-addressed)
```

Two distinct planes: **metadata** (small, transactional) and **blocks** (large, immutable).

### Chunking

A file is split client-side into fixed-size chunks (e.g., 4 MB):

```
photo.jpg (24 MB) → [c1, c2, c3, c4, c5, c6]
each chunk: SHA-256 → content_hash
```

Benefits:
- Resumable upload — retry only failed chunks.
- Dedup: same chunk uploaded once, regardless of which file.
- Parallel upload of chunks.

Variable-length chunking (Rabin fingerprinting) handles inserts/deletes mid-file better; used in production at Dropbox.

### Deduplication

Content-addressed storage:

```
chunk_blob in S3: key = "blocks/<sha256>"
```

Upload flow:

```
client computes hash for each chunk
client asks: "do you have these hashes?"
server replies: "missing c2 and c5"
client uploads only c2 and c5
```

Massive bandwidth savings for common content (system DLLs, popular files).

### Metadata Model

```sql
files (
  file_id,
  user_id,
  folder_id,
  name,
  size,
  modified_at,
  version,
  is_deleted
)

file_chunks (
  file_id,
  version,
  index,
  chunk_hash
)

users, folders, sharing_acl, ...
```

Postgres for transactional integrity.

### Upload Flow

```
1. Client: split file → chunks → hashes.
2. POST /files/init {filename, chunks: [{hash, size}, ...]}
   → server returns presigned URLs for missing chunks.
3. Client uploads missing chunks to S3 in parallel.
4. POST /files/commit {file_id, version, chunks: [hashes]}
   → server records metadata; triggers notify to other devices.
```

Idempotent via the commit step. Resumable: re-init returns the same missing list if anything failed.

### Sync Protocol

Each device tracks a `sync_token` (cursor over server changes).

```
GET /sync?since=token
  → returns list of changes (files added/modified/deleted)
  → new token

client applies changes; downloads new chunks via dedup check
```

Server pushes via WebSocket / long-poll for instant sync.

### Conflict Resolution

Two devices edit offline → conflict on sync.

Strategies:
- **Last-write-wins**: easiest, loses data.
- **Both versions kept**: rename one to `file (Alice's conflict copy).docx`. Dropbox's approach.
- **CRDT for text files**: collaborative editing (Google Docs uses Operational Transformation; modern apps use CRDTs).

For general files, "keep both" is standard.

### Versioning

Each commit increments version. Old versions kept for N days:

```
files: { id, latest_version }
file_versions: { file_id, version, modified_at, chunks: [hash...] }
```

Restore = re-point latest to older version. Cheap (metadata only); chunks already in storage.

### Sharing

Two patterns:

**Link sharing**: `share_token` → ACL: anyone/with-link/specific-users.
**Folder sharing**: ACL on folder; users see it appear in their tree.

Permissions: read / comment / edit / owner.

Cache permissions per (user, file) in Redis with invalidation on ACL change.

### Storage Backend

- Block store: S3 or equivalent. Immutable, content-addressed. 11 9s durability.
- Tier: hot S3 for recent; Glacier for old / cold.
- Encryption: at rest (S3 SSE-KMS); in transit (TLS).
- Per-user encryption (client-side): zero-knowledge mode like Mega/Boxcryptor; trades indexing/search.

### Network Optimization

- **Delta sync**: only changed chunks transferred.
- **LAN sync**: peer-to-peer chunk transfer between same-LAN devices (Dropbox).
- **Block-level dedup**: as above.

### Multi-Region

- Metadata: replicated globally; primary per region or distributed SQL.
- Blocks: stored in nearest region; replicated lazily across regions for redundancy.
- Reads: served from nearest replica.

### Scale Numbers

500M users × avg 10 GB → 5 EB total. Trillions of chunks.

- Metadata: ~10s of TB; fits in sharded Postgres or distributed SQL.
- Block store: provider problem (S3 handles).
- Sync events: 100k+/sec at peak; Kafka.

### Failure Modes

| Failure | Handling |
|---------|---------|
| Upload interrupted | Resume from last successful chunk |
| Sync conflict | Conflict copy; user resolves |
| Block missing in storage (rare) | Re-upload from any device that has it |
| User deletes accidentally | Soft-delete + version restore |
| Massive bulk delete (account compromise) | Rate-limit + delayed actual delete |

### Search

Indexed metadata (filename, content type) in Elasticsearch.

Full-text search of file content:
- Index text-extractable files (`.txt`, `.pdf`, `.docx`) post-upload.
- OCR for images (optional).
- Trade index storage for searchability.

### Trash / Soft Delete

```
on delete: set is_deleted = true, deleted_at = now()
purge job: actually delete blocks (after 30d retention)
```

Allows undo, audit, partial-account-recovery.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Storing whole files in DB | Metadata in DB; bytes in object store |
| No dedup | Storage costs 10× more for popular content |
| Sync over polling only | Use push (WS) for instant updates |
| Holding sync state in client only | Server is source of truth; client cache |
| Last-write-wins on text edits | Lose data; use conflict copies or CRDT |

> [!NOTE]
> The hard parts of Dropbox-like systems are sync semantics and conflict handling, not storage. Block dedup + S3 handles bytes; the user-facing magic is "my files are everywhere and never lost."

### Interview Follow-ups

- *"How would you handle a corrupt sync state on client?"* — Detect via checksum mismatch; trigger full re-sync from server; quarantine local conflicting copy.
- *"How big can a single file be?"* — No real limit; just more chunks. Multi-part upload to S3 handles up to 5 TB per object.
- *"How do you do 'recently modified by team'?"* — Activity log table; index by folder + time; subscribe to changes via WS.
