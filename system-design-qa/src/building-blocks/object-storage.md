# Q: Object storage (S3) — when, what guarantees, gotchas.

**Answer:**

S3-like object storage is the cheapest, most durable way to keep blobs at scale. It's not a filesystem — it's an immutable key-value store with HTTP API.

### What It Is

```
Bucket: my-bucket
  Key: photos/2025/04/img1.jpg → bytes
  Key: photos/2025/04/img2.jpg → bytes
  Key: backups/db_20250401.sql.gz → bytes
```

Flat namespace. Slashes in keys are just characters; folder prefixes are a convention.

### Properties

- **Durability**: 11 9s (S3 standard). Multiple copies across AZs.
- **Availability**: 99.99% (S3 standard).
- **Strong read-after-write consistency** (since Dec 2020 for S3).
- **Cost**: cheap storage (~$23/TB/month standard), per-request fees, egress is expensive.
- **Throughput**: 3,500 PUT/copy/delete and 5,500 GET per second per partition prefix.

### Storage Classes

| Class | Use | Cost |
|-------|-----|------|
| Standard | Hot, frequent access | $$$ |
| Intelligent-Tiering | Auto-move based on access | $$ |
| Standard-IA | Infrequent access | $ |
| Glacier Instant | Archive, immediate retrieve | $ |
| Glacier Flexible | Archive, minutes to hours retrieve | ¢ |
| Glacier Deep Archive | Archive, hours to retrieve | ¢¢ |

Move data via lifecycle policies (e.g., "after 30 days → IA, after 1 year → Glacier").

### What Object Storage Is GOOD For

- Large files (photos, videos, backups).
- Static website assets.
- Data lake (Parquet, ORC) for analytics.
- Logs / archives.
- ML training data sets.
- Software releases / installer files.

### What It's BAD For

- Frequent updates to the same object (no in-place modify; full rewrite).
- Tiny files (per-request overhead dwarfs payload).
- Low-latency reads (~10–100 ms per GET vs microseconds for Redis).
- Filesystem semantics (no rename — just copy + delete).

### Multipart Upload

Large files split into parts (5 MB – 5 GB each):

```
1. Initiate multipart upload → get UploadId.
2. Upload parts in parallel.
3. Complete (with list of parts + ETags).
```

Resume: re-upload only failed parts. Cap: 10,000 parts → max object 5 TB.

### Presigned URLs

Time-limited URL granting access without sharing credentials:

```python
url = s3.generate_presigned_url('put_object',
    Params={'Bucket': 'b', 'Key': 'k'}, ExpiresIn=300)
# client uploads directly to S3 with this URL
```

Use for direct browser uploads (skip your backend); secure media delivery.

### Versioning

Enable on bucket → every PUT creates a version. Delete = adds a "delete marker"; old versions remain until explicitly removed.

Protects against accidental overwrites/deletes. Cost: storage of all versions.

### Object Lock / Compliance

Write-Once-Read-Many. Object can't be deleted for a retention period (governance or compliance mode).

Used for audit logs, regulatory archives, ransomware protection.

### Cross-Region Replication

Bucket-level: every object replicated to another region (sync or async).

Used for DR or for serving traffic close to users.

### Access Patterns

**Read patterns**:
- Direct from client via presigned URL (most common).
- Through CDN (CloudFront) for caching + edge.
- From backend (analytics jobs).

**Write patterns**:
- Direct from client via presigned URL.
- From backend (uploads, ETL).

For browser apps: presigned URL + CDN. Backend rarely intermediates the bytes.

### Cost Optimization

- Lifecycle policies → cold tier old data.
- Intelligent-Tiering for unpredictable access.
- **Egress** is the most expensive line — use CloudFront / Cloudflare R2 for free egress.
- Avoid `LIST` over millions of keys; use S3 Inventory reports.

### Eventual vs Strong Consistency

Pre-Dec-2020 S3 was eventually consistent for reads after writes. **As of now: strong read-after-write consistency for all operations**.

You can read your write immediately. Same for delete.

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Many tiny files | Pack into bigger objects (e.g., tar/parquet) |
| Hot prefix at scale | Spread by hashing key prefix (`a1b2c3/...` vs `2025/04/...`) |
| Storing PII unencrypted | Enable SSE-S3 or SSE-KMS by default |
| Public bucket leak | Block Public Access + bucket policies + IAM |
| Long polling for new files | Use S3 event notifications (SNS / SQS / Lambda) |
| `LIST` to find files | S3 is KV; maintain own index if you need search |

### S3 Event Notifications

```
PUT object in bucket → SNS / SQS / Lambda trigger
```

Common patterns:
- Trigger image-resize on upload.
- Index new files into Elasticsearch.
- Notify users of new content.

### Multi-Region Architecture

```
client uploads → nearest region's bucket
            │
            ▼ async replicate
       other region buckets
```

Read from nearest region; cross-region reads possible but slower.

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| Treating S3 as filesystem | No rename, no folder ops; just keys |
| Synchronous PUT for every event | Batch into bigger objects |
| Public bucket "for convenience" | Data breaches |
| Not setting lifecycle policy | Storage bill grows forever |
| Logging every request | Cost; use CloudTrail or aggregate |

> [!NOTE]
> S3 is the cheapest hard drive on earth. Architect around its strengths (immutable, durable, KV) and don't fight its weaknesses (latency, mutations).

### Interview Follow-ups

- *"How would you build a CDN-fronted bucket for video?"* — CloudFront in front; signed URLs; bucket private; signed Cookie for whole-folder access.
- *"How to manage millions of small files?"* — Pack into parquet/tar; store index in DB; reduces S3 request cost dramatically.
- *"S3 vs DynamoDB?"* — S3 = large blobs, infrequent access pattern, eventual-now-strong consistency. DynamoDB = small structured records, milliseconds, transactions.
