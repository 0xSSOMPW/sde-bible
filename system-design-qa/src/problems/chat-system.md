# Q: Design a chat system (WhatsApp / Slack / Discord).

**Answer:**

Classic real-time problem. Tests your ability to combine **persistent connections**, **message ordering**, **delivery guarantees**, and **multi-device sync**.

### Requirements

**Functional**:
- 1:1 and group chat (up to 200+ members).
- Real-time delivery; sent / delivered / read receipts.
- Message history.
- Online presence.
- Attachments / images.
- Push notifications when offline.

**Non-functional**:
- 1B users, 50M concurrent connections.
- Messages delivered within 1 second.
- Persistent history across devices.
- Sent messages are durable (no loss).

### Capacity Estimation

- 50M concurrent → connection servers.
- 100B msgs/day → 1.1M msgs/sec.
- Storage per message: 200 B body + 100 B metadata → 30 TB/day for text.
- Media (separate storage): vastly larger; defer to object storage.

### High-Level Architecture

```
Mobile / Web client
       │   persistent WebSocket
       ▼
  ┌──────────────┐
  │ Connection   │ ← stateful gateway, holds N WS connections
  │ Service      │
  └──────┬───────┘
         │ Kafka (per-user routing)
         ▼
  ┌──────────────┐
  │ Chat Service │ ← stateless, applies business rules
  └──────┬───────┘
         ▼
  ┌──────────────┐      ┌─────────────────┐
  │  Messages DB │      │ Presence Service│
  │  (Cassandra) │      │   (Redis)       │
  └──────────────┘      └─────────────────┘
         │
         ▼
   Push gateway (APNs / FCM) for offline users
```

### Connection Layer

**WebSocket** for browser/mobile bidirectional. Alternative: gRPC bidi streams (mobile), MQTT (low-power IoT).

Connection service:
- Maintains map `user_id + device_id → connection`.
- Sticky LB: hash by user_id so the user's traffic always lands on the same gateway.
- On disconnect: enqueue offline messages.

Single Java/Go process can hold 50–100k WS connections; cluster to N servers for 50M total.

### Message Flow (1:1)

```
1. Alice sends "hi" via her WS to her gateway.
2. Gateway publishes to Kafka topic, key=conversation_id.
3. Chat service consumes:
     - Assign message_id (snowflake, monotonic within conversation).
     - Persist to messages DB.
     - Publish to "delivery" topic key=recipient_user_id.
4. Delivery worker consumes:
     - Look up Bob's gateway (via Redis presence).
     - If online: forward via WS.
     - If offline: write to "undelivered" set, fire push notification.
5. Alice sees "sent" tick when Kafka ack returns.
6. Bob's client ACKs receipt → "delivered" tick.
7. Bob views message → "read" tick.
```

### Group Chat

For groups up to ~200 members, fan-out is per-message:
- Lookup group members.
- Publish a delivery event per member.

For larger groups (Slack channels with thousands):
- Use a **pull model** for inactive members; only push to actively connected ones.
- Channel feed pre-built lazily.

### Message Ordering

Per-conversation ordering is critical (replies must come after their parent).

Achieve via:
- Per-conversation Kafka partition (key=conversation_id).
- Snowflake IDs that encode timestamp → client orders by ID.
- Server assigns the ID (clients don't generate; their clocks lie).

Across conversations, no global order required.

### Data Model

**Messages** (Cassandra):

```
messages:
  partition key:  conversation_id
  clustering key: message_id (snowflake, ts-ordered)
  fields:         sender_id, body, attachments, sent_at
```

Read pattern: `SELECT * WHERE conversation_id = X AND message_id < Y LIMIT 50` — for "load more" history. Cassandra fast for this.

**Conversations**:

```
user_conversations (DynamoDB / Postgres):
  user_id, conversation_id, last_read_message_id, unread_count
```

For inbox view: "list conversations sorted by latest message."

**Presence** (Redis):

```
SET presence:user:42 "online" EX 30        ← refresh on each heartbeat
```

TTL handles "offline" detection without explicit disconnect (covers crashes, dropped networks).

### Delivery & Read Receipts

Three states per message per recipient:
- Sent (server ack).
- Delivered (recipient device ACKed).
- Read (user opened conversation past this ID).

Stored as `last_delivered_message_id` and `last_read_message_id` per recipient per conversation. Updated on receipt; broadcast back to sender.

### Multi-Device

User has phone + laptop. All devices receive each message; reads sync.

Maintain device list per user; deliver to each device with its own WS. Sync `last_read` cross-device.

### Offline / Push Notifications

When recipient offline:

```
1. Store message in user's pending queue (Redis list or DynamoDB).
2. Fire push notification (APNs for iOS, FCM for Android).
3. On reconnect, client requests undelivered messages since `last_seen_id`.
```

Push is best-effort; don't rely on it for delivery — full sync on reconnect is the source of truth.

### End-to-End Encryption (Signal Protocol)

WhatsApp / Signal / iMessage encrypt messages so the server can't read them:

- **Double Ratchet**: forward-secret keys per message, ratcheted on every send.
- **X3DH**: initial key agreement using prekeys.
- **Multi-device**: each device has its own keypair; messages encrypted per device.

Server's role: ferry encrypted blobs. Server never sees plaintext.

Trade: server can't index/search; backups are encrypted too. Group chat key management is hard.

### Search

If E2E: search must be on-device.

If not E2E: index in Elasticsearch keyed by user × conversation, with per-user ACL filters at query time.

### Failure Modes

| Failure | Handling |
|---------|---------|
| Connection gateway crashes | Client reconnects; pulls undelivered since `last_message_id` |
| Message DB write fails | Producer retries; client retries; idempotent by client-message-id |
| Kafka partition unavailable | Producer buffers; longer windows tolerable |
| Push gateway down | Message still stored; delivered when user opens app |

Always design for "message reaches the DB" — every other layer is a delivery optimization.

### Idempotency

Each client message has a `client_message_id` (UUID). Server dedups so retries don't duplicate.

### Scale

```
50M concurrent → ~500 gateways at 100k conns each
1M msgs/sec → 10–20 Kafka brokers, multi-topic
30 TB/day → Cassandra cluster sized for ~1 year retention then archive
```

### Multi-Region

- Hot conversations: route to home region of either user; sync across regions async.
- Long-distance conversations: pick a "home region" per conversation (e.g., creator's region) and route there.
- Latency hit: ~100 ms extra for cross-region delivery — usually acceptable for chat.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| HTTP long-polling instead of WS | Higher overhead, worse latency |
| Same DB for messages + counts + presence | Different access patterns; split |
| Storing every "delivered/read" event in a topic | Massive write amplification; coalesce server-side |
| Skipping idempotency | Duplicate messages on retry |
| Trusting client timestamps | Clocks lie; server stamps |

> [!NOTE]
> The hardest part isn't real-time; it's reconnect / offline / multi-device sync. Design that path first; live messaging is the easy case.

### Interview Follow-ups

- *"How do you handle a user who opens the app after a week offline?"* — Client sends `last_seen_message_id` per conversation; server returns delta; pagination if large.
- *"How do you scale to 10M users in one group (broadcast channel)?"* — Pure fan-out is too expensive; use a pub/sub model — server stores once, all clients poll/subscribe. Hybrid with the small-group fan-out for engagement features.
- *"What's the difference between WhatsApp and Slack design?"* — WhatsApp: P2P-ish, E2E, ephemeral history. Slack: workspace-centric, channels are first-class, search/retention paramount, federated identity. Different priorities → different architectures.
