# Q: WebSockets vs Server-Sent Events vs Long Polling — picking real-time transport.

**Answer:**

Three ways to push data from server to client. Each picks a point on the **bidirectionality × overhead × infrastructure-cost** triangle.

### Long Polling

Client makes HTTP request; server holds it open until data arrives (or timeout), then responds. Client immediately reopens.

```
client ──GET /events──► server (holds for 30s)
client ◄──response─────  server (data arrives or timeout)
client ──GET /events──► server (next poll)
```

Pros:
- Plain HTTP. Works through proxies, firewalls, CDN.
- No new protocol.

Cons:
- Connection setup overhead per cycle.
- Server holds many idle connections.
- Latency = connection-reopen time.

Use for: legacy environments where WS / SSE aren't supported.

### Server-Sent Events (SSE)

Single long-lived HTTP/1.1 or HTTP/2 connection. Server sends a stream of text events.

```
GET /stream HTTP/1.1
Accept: text/event-stream

Response:
HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"type": "msg", "body": "hi"}\n\n
data: {"type": "msg", "body": "hello"}\n\n
...
```

Pros:
- One direction (server → client) — perfect for feeds, notifications.
- Auto-reconnect built into browser API.
- Plain HTTP — proxies, CDN edge support.
- Built-in `id` for resume.

Cons:
- One-way.
- Text only (binary needs base64).
- Some legacy proxies buffer SSE responses.

Use for: live notifications, server logs, dashboard updates, AI chat streaming.

### WebSockets

Persistent bidirectional connection over a single TCP socket. Initiated via HTTP `Upgrade`.

```
GET / HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: ...

← HTTP/1.1 101 Switching Protocols
```

After upgrade, the socket carries WS frames in either direction.

Pros:
- True bidirectional.
- Low overhead per message (few bytes).
- Binary or text.

Cons:
- Stateful — server must track each connection.
- LB / CDN support varies (sticky required).
- Reconnect logic non-trivial.

Use for: chat, collaborative editing, gaming, trading platforms.

### Comparison

| Feature | Long Poll | SSE | WebSocket |
|---------|----------|-----|-----------|
| Direction | C↔S | S→C | C↔S |
| Overhead/message | High | Low | Lowest |
| Setup cost | High (per cycle) | One-time | One-time |
| Browser API | `fetch`, manual | `EventSource` | `WebSocket` |
| Auto-reconnect | Manual | Built-in | Manual |
| Binary | No | Base64 only | Yes |
| Proxy/CDN friendly | High | Medium-high | Low (sticky) |
| Server connection count | High (idle) | Medium | Medium |

### Connection Cost Per Server

Each connection consumes:
- OS file descriptor (1).
- TCP socket buffers (~64 KB default).
- Application memory (KB to MB depending on state).

```
Typical server: 100k concurrent connections (8 GB RAM allocated)
Tuned (epoll/io_uring, lean app): 1M+ connections
```

Scale-out: shard connections across N gateways; LB by client ID or sticky hashing.

### Choosing in 30 Seconds

```
Need bidirectional? ─── yes ──► WebSocket
              │
              no
              │
Server pushes events only? ─── yes ──► SSE
              │
              no
              │
Don't care about latency, just need updates? ─── Long Poll
```

### Real-Time at Scale Patterns

Whichever transport, the backend looks similar:

```
event source ──► message bus (Kafka/Redis pub-sub) ──► connection gateway ──► clients

gateway:
  - holds N connections
  - subscribes to topics relevant to its connected users
  - filters and pushes events to each client
```

Sticky LB → user always lands on the same gateway → that gateway holds the subscription.

### Heartbeats / Keepalives

All three need application-level keepalives:
- WS: send ping/pong every 30s.
- SSE: server sends comment `:keepalive\n` every 15s.
- LP: timeout the long-poll at 30–60s; client reconnects.

Without heartbeats, idle connections die silently behind NATs / proxies.

### Auth

- WS: usually JWT in URL query param or sub-protocol header (cookies work too).
- SSE: same-origin cookies or `Authorization` header (no headers in browser `EventSource`; use cookies or query token).
- LP: standard cookie / header auth.

Token rotation: send signed short-lived token via WS / SSE message; refresh before expiry.

### Backpressure

If client can't keep up:
- WS: app reads from socket; OS buffers fill; producer blocks or drops.
- SSE: similar; some servers drop oldest.
- LP: client controls pace (next poll happens after processing).

Server policy: queue per-connection (bounded), drop oldest on overflow, or disconnect slow consumers.

### CDN / Edge

- WS: limited; some edges (Cloudflare, Fly.io) support; CloudFront does not natively.
- SSE: HTTP-native; most CDNs work if buffering disabled.
- LP: native HTTP; full CDN support.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using WS where SSE would do | One-way? Skip the complexity |
| No reconnect logic on client | Long outages = unrecoverable session |
| Holding state per connection in app memory | Crash loses sessions; persist if state matters |
| Single WS gateway as SPOF | Multi-instance with sticky LB |
| Pushing through CDN without WS support | Doesn't work; bypass CDN for WS path |

> [!NOTE]
> Default to SSE for "server → client updates" use cases — it's HTTP, simple, infra-friendly. Reach for WS only when you need bidirectional. Long-poll is legacy.

### Interview Follow-ups

- *"How would you scale WebSockets to 10M concurrent?"* — Shard gateways; per-user sticky LB; pub/sub bus (Redis or Kafka) for per-user routing.
- *"How is WebRTC different?"* — P2P data/media; needs signaling (often WS); used for low-latency media (voice/video).
- *"WebTransport?"* — Modern alternative to WS over HTTP/3 (QUIC); multiplexed streams, unreliable datagrams. Not widely adopted yet.
