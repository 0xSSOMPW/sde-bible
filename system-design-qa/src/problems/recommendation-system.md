# Q: Design a recommendation system.

**Answer:**

Tests the **two-stage retrieval + ranking** architecture, **embedding-based similarity**, **online vs offline pipeline**, and **scale considerations**.

### Requirements

**Functional**:
- Given a user, return top-N items (videos, products, posts).
- Update recommendations as user behavior changes.
- Cold-start for new users.
- Diversity / freshness rules.

**Non-functional**:
- 100M users, 100M items.
- p99 < 100 ms recommendation lookup.
- Updates within hours of new behavior.

### The Two-Stage Architecture

```
User request
   │
   ▼
[Candidate Generation]    ← retrieve 1000s of candidates
   │
   ▼
[Ranking]                  ← ML model scores top 100
   │
   ▼
[Reranking / Diversity]    ← apply business rules
   │
   ▼
Top 10
```

This pattern fits YouTube, Pinterest, TikTok, Spotify, Amazon. The stages have different scale + complexity profiles.

### Stage 1: Candidate Generation

Fast retrieval of "potentially relevant" items. Sources:

- **Collaborative filtering**: users who watched X also watched Y.
- **Content-based**: items similar to what the user has interacted with.
- **Embedding nearest-neighbor**: vector similarity search.
- **Popular**: trending items in user's region/category.
- **Recently watched**: time-based history.

Each generates ~1000 candidates. Union, dedup.

### Embedding-Based Retrieval

Train a model to produce **user embeddings** and **item embeddings** in the same space:

```
user vector  = encoder(user_history)
item vector  = encoder(item_features)

dot product  ≈ predicted affinity
```

Find top-K items via Approximate Nearest Neighbor (ANN) search:

- **Faiss** (Meta), **HNSW**, **ScaNN** (Google), **Annoy** (Spotify).
- Sub-millisecond on millions of items.
- Hosted services: Pinecone, Weaviate, Qdrant, Vertex AI Matching Engine.

### Stage 2: Ranking

The 1000 candidates go through a heavier ML model:

- Features: user × item × context × cross-features.
- Model: gradient-boosted trees (LightGBM, XGBoost) or DNN.
- Output: predicted CTR / engagement score.

Optimization target depends on platform — clicks, watch time, purchases.

### Stage 3: Reranking

Apply business rules:

- Diversity (don't return 10 of the same category).
- Freshness (boost recent items).
- Filtering (don't repeat what user already watched).
- Calibration (sponsored items mixed in).

### Architecture

```
                       client request
                            │
                            ▼
                       Rec Service (online)
                            │
        ┌───────────────────┼──────────────────┐
        ▼                   ▼                  ▼
  User Profile        Candidate Sources   Feature Store
   (Redis)        ├── ANN index            (per-user features)
                  ├── popularity cache     (per-item features)
                  └── recently-watched
                            │
                            ▼
                       Ranker (ML model)
                            │
                            ▼
                       Top N
```

### Offline Pipeline

Heavy lifting happens offline:

```
1. Logs (impressions, clicks) → Kafka → S3
2. Daily/hourly training jobs (Spark / Flink):
   - Compute user features (last-N actions).
   - Compute item features (popularity, embedding).
   - Train ranker.
3. Push artifacts to online stores (Redis, vector DB).
```

Cadence:
- Item embeddings: nightly.
- User features: every few minutes (recent behavior).
- Model: weekly or daily retraining.

### Online Pipeline

Real-time:

```
on user action (click, like):
   stream → Kafka → online feature updater
   update user's recent-actions feature → Redis (latency < 100 ms)
```

Recommendations served < 100 ms later reflect the action.

### Storage

| Data | Store |
|------|-------|
| User profile + history | Redis / DynamoDB |
| Item embedding index | Faiss / Pinecone |
| Feature store | Redis (online) + Parquet (offline) |
| Logs (training data) | Kafka → S3 |
| Model artifacts | S3 / model registry |

### Cold Start

**New user**: no history.
- Onboarding form (interests).
- Default to popular items in user's region.
- Demographic-based recommendations.

**New item**: no engagement signal.
- Content-based features (text embedding from title/description).
- Exploration boost (intentionally show to small audience to gather signal).

### A/B Testing

Recommendations are highly empirical. Standard:

- Bucket users (consistent hash).
- Serve different models / candidate sets per bucket.
- Track engagement metrics per bucket.
- Promote winners.

Online evaluation > offline metrics. Many improvements look great offline and fail in A/B test.

### Fairness / Filter Bubbles

- Diversity reranking caps single-category in top N.
- Exploration: occasionally surface items outside the user's typical preference.
- Bias audits across protected attributes.

### Scaling Numbers

```
Candidate generation:
  ANN lookup on 100M items, top 1000 → ~5 ms
  Other sources (popular, recent): in-memory, ~1 ms

Ranking:
  1000 candidates × N features → light DNN forward pass → ~20 ms

Total online budget: ~50 ms — leaving margin for network + serialization
```

Inference often on GPU servers; one box can serve thousands of QPS.

### Failure Modes

| Failure | Handling |
|---------|---------|
| ANN index unavailable | Fall back to popular items |
| Feature store down | Use cached features from app memory |
| ML model returns bad scores | Sanity caps; failover to previous model version |
| Cold start for new user | Demographic fallback |

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| One huge model for candidates + ranking | Two-stage is efficient and modular |
| Recompute embeddings online | Batch precompute; ANN at query time |
| Recommendations latency 500 ms+ | Pre-fetch on session start; cache results 1 min |
| No A/B testing infrastructure | Can't tell if changes help; build platform first |
| Ignoring diversity | All TVs → user fatigue |

> [!NOTE]
> The interesting design choice is **what counts as a candidate**. Get the candidate set right and ranking is easy; get it wrong and no model rescues you.

### Interview Follow-ups

- *"What's the difference between collaborative and content-based?"* — Collaborative uses user-item interaction patterns. Content-based uses item features (text, image). Most production systems blend both.
- *"How would you build for explainability?"* — Track which candidate source produced each recommendation; surface as "because you watched X."
- *"How do you handle adversarial content (clickbait)?"* — Train on long-term engagement (watch time, return visits) not just clicks; explicit downranking signals from users.
