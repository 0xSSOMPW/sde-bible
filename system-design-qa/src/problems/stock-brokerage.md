# Q: Design an online stock brokerage (Robinhood / Zerodha / Groww).

**Answer:**

A brokerage is a **financial gateway**: it takes customer orders, validates them against risk and cash, routes them to an exchange, tracks fills, and keeps an exact ledger. The hard parts are **correctness of money and orders** (ACID, idempotency, exactly-once effects), **low-latency order routing**, and **market-data fan-out** to hundreds of thousands of watchlists. The matching engine itself belongs to the exchange, but interviewers often ask you to design one too, so cover both.

### Clarify: Broker vs Exchange

| | Broker (Robinhood, Zerodha, Groww) | Exchange (NSE, BSE, NYSE, NASDAQ) |
|---|---|---|
| Customers | Retail investors | Brokers / market makers (members) |
| Core job | Accept orders, risk-check, route, hold customer assets, report | Match buy and sell orders, publish market data, guarantee trades via clearing house |
| Latency need | ~10-50 ms internal; exchange RTT dominates | Microseconds for matching |
| State | Orders, positions, cash ledger, KYC | Order book per symbol |
| Regulation | SEBI / SEC / FINRA: KYC, suitability, audit, segregation of client funds | Market surveillance, circuit breakers |
| Settlement | Instructs clearing, updates client demat / holdings | Clearing corporation novates trades, runs T+1 |

State this up front: "I will design the **broker**, then the **matching engine** as a deep dive because it is the core of any exchange and interviewers often pivot to it."

### Requirements

**Functional**:
- Onboarding, KYC, link bank account, add funds (UPI / ACH).
- Watchlists with live quotes; charts.
- Place, modify, cancel orders: market, limit, stop-loss, stop-limit; intraday vs delivery; fractional shares (US-style brokers).
- Real-time order status (pending, open, partial, filled, rejected, cancelled).
- Portfolio: holdings, positions, realized and unrealized P&L, margin.
- Ledger: funds, charges, taxes; statements and contract notes.
- Corporate actions (splits, dividends, bonus).

**Non-functional**:
- **Zero lost or duplicated orders.** An order the user saw accepted must reach the exchange exactly once.
- Strong consistency for cash and positions (ACID). Never allow negative buying power.
- Order placement p99 < 100 ms internal (excluding exchange).
- Market-data push latency < 200 ms to the phone.
- Handle the **9:15 AM open spike** (all pending AMO orders and thousands of users hitting buy at once).
- Full audit trail (7-8 years retention), reproducible state.
- 99.99% availability during market hours; planned maintenance outside them.

### Capacity Estimation

Zerodha-scale (about 15M clients, a peak of 5-6M concurrently logged in; ~15% of Indian retail volume):

- Orders: ~10M orders/day, but bursty. Open and close are 10-20x average. Peak ~5-10k orders/sec across the platform.
- Order events (each order has ~4-6 events: new, ack, partial, fill, modify): ~50-100k events/sec at peak.
- Market data: NSE publishes ~5k-10k symbols but only ~2-3k active; each tick is ~50-100 bytes; a full feed is ~50-100 MB/s uncompressed. Not per-user, per-symbol.
- Watchlist fan-out: 500k concurrent WebSocket users x 20 symbols x 1 update/sec (after conflation) = 10M pushes/sec, ~100 bytes each = 1 GB/s egress at peak.
- Ledger: ~10-30M postings/day (every trade makes 4-8 postings: cash, brokerage, STT, GST, stamp duty). Tiny by DB standards, but must be correct.
- Storage: orders + events ~1 KB each x 10M/day x 365 x 8 years ≈ 30 TB. Tick history is larger (TBs/day compressed) and lives in a time-series store.

> [!NOTE]
> The workload is **tiny in write volume** compared with social networks, but **zero-tolerance for errors**. The interesting scale problems are fan-out of market data and bursts at 9:15, not raw QPS.

### API Design

```
POST /v1/orders
  Headers: Idempotency-Key: <client-generated UUID>
  Body: { symbol, side: BUY|SELL, type: MARKET|LIMIT|SL|SL-M,
          qty, price?, trigger_price?, product: CNC|MIS|NRML, validity: DAY|IOC }
  → 202 { order_id, status: "RECEIVED" }          // accepted into OMS, not yet at exchange

PUT    /v1/orders/{id}        { qty?, price?, trigger_price? }   // modify (cancel/replace)
DELETE /v1/orders/{id}                                           // cancel
GET    /v1/orders?status=open
GET    /v1/positions            // intraday, net
GET    /v1/holdings             // delivery, settled
GET    /v1/funds                // available margin, buying power
GET    /v1/quotes?symbols=A,B   // snapshot (REST)
WS     /v1/stream               // subscribe {mode: ltp|quote|full, symbols[]} ; order updates push
```

- `202 Accepted`, not `200`: order placement is asynchronous; the final state arrives via WebSocket or polling.
- Rate-limit per user (Zerodha: 10 orders/sec) to protect the exchange link and detect runaway algos.
- Idempotency-Key is **mandatory** on order POST. Network retries from flaky mobile clients are guaranteed.

### Data Model

```
users(id, kyc_status, ucc, risk_profile, ...)
accounts(id, user_id, type, currency)                         -- ledger accounts
ledger_entries(id, txn_id, account_id, amount, direction, ts) -- append-only, double-entry
orders(id, user_id, idempotency_key UNIQUE, symbol, side, type, qty, price,
       trigger_price, product, validity, status, filled_qty, avg_price,
       exchange_order_id, version, created_at, updated_at)
order_events(order_id, seq, type, payload, ts)                -- immutable event log
trades(id, order_id, exchange_trade_id UNIQUE, qty, price, ts)
positions(user_id, symbol, product, net_qty, avg_cost, realized_pnl)
holdings(user_id, symbol, qty, avg_cost, t1_qty, pledged_qty)
margin_blocks(order_id, user_id, amount, released_at)
instruments(token, symbol, tick_size, lot_size, price_band_lo, price_band_hi, circuit_pct)
```

Key choices:
- `orders.version` supports optimistic concurrency for state transitions.
- `UNIQUE(user_id, idempotency_key)` makes duplicate POSTs return the original order.
- `trades.exchange_trade_id UNIQUE` makes fill processing idempotent (exchange may resend).
- Ledger is **append-only**; balances are derived (or maintained as a cached running balance updated in the same transaction).

### High-Level Architecture

```
 Mobile / Web / API clients
        │  HTTPS (orders)                       ▲ WebSocket (quotes, order updates)
        ▼                                       │
  API Gateway (authN, 2FA session, rate limit)  │
        │                                       │
        ▼                                 Ticker / Streamer fleet
  Order Service (OMS) ◄── Risk Engine           ▲  (conflation, subscriptions)
        │   │   (margin, bands, limits)         │
        │   ▼                                   │
        │  Orders DB (Postgres) + Outbox        │
        │   │                                   │
        │   ▼                                   │
        │  Kafka "order-events" ────────────────┤
        ▼                                       │
  Order Router / Exchange Gateway ──FIX/binary──► EXCHANGE ──► Market Data Feed
        ▲            (sessions, seq nums)          │             │ (multicast UDP)
        │                                          ▼             ▼
        └──── Execution Reports (fills) ───────────┘      Feed Handler → Kafka "ticks"
                         │                                        │
                         ▼                                        ├──► Tick store (TSDB)
              Fill Processor → Ledger + Positions (ACID)          └──► Streamer fleet
                         │
                         ▼
        Notifications · Contract notes · Reconciliation · Settlement (T+1)
```

Three independent planes:
1. **Order plane** (low volume, strong consistency): API → OMS → risk → router → exchange → fills → ledger.
2. **Market-data plane** (high volume, loss-tolerant): feed handler → stream → conflation → WebSocket.
3. **Back-office plane** (batch, regulatory): ledger, settlement, reconciliation, reports, tax.

### Order Management System (OMS)

The OMS owns the **order lifecycle**. It is the single writer of order state.

```
RECEIVED ──risk ok──► VALIDATED ──sent──► PENDING_NEW ──exch ack──► OPEN
    │                     │                    │                      │
    ▼ risk fail           ▼ gateway down       ▼ exch reject          ├─ partial fill ──► PARTIALLY_FILLED ─┐
 REJECTED             REJECTED             REJECTED                   │                                      │
                                                                      ├─ full fill ──────────────────────────► FILLED
                                                                      ├─ cancel req ──► PENDING_CANCEL ──► CANCELLED
                                                                      └─ modify req ──► PENDING_REPLACE ──► OPEN (new version)
 TRIGGER_PENDING (stop orders held at the broker until trigger price hit) ──► VALIDATED
 EXPIRED (DAY order at close)
```

Rules:
- Transitions are validated by a state machine (illegal transition → error, alert).
- Each transition writes the order row (`version+1`) **and** an `order_events` row **and** an outbox row in the **same DB transaction**. A relay publishes the outbox to Kafka (transactional outbox, no dual-write).
- `PENDING_*` states exist because exchange confirmation is asynchronous; you cannot assume your cancel worked.

### Pre-Trade Risk Checks

Executed **synchronously before** the order leaves the broker. Reject early; the exchange will reject too, but bad orders waste exchange rate limits and expose the broker to financial risk.

| Check | Detail |
|-------|--------|
| Authentication / session | Valid token, 2FA done, account active, KYC verified, segment enabled |
| Instrument | Tradable, not suspended, not in ban period (F&O), lot size multiple, tick size aligned |
| **Price band** | Limit price within the exchange's dynamic price band (e.g., 5/10/20% of previous close) |
| **Circuit limits** | Reject buys at upper circuit when no sellers; reject orders far from LTP (fat-finger: > X% from LTP) |
| **Buying power / margin** | BUY: cash + collateral (pledged holdings, haircut) >= qty x price + charges. Intraday MIS: leveraged margin. F&O: SPAN + exposure margin |
| **Holdings** | SELL (delivery): qty available >= qty (accounting for T+1 unsettled and pledged) |
| Position / exposure limits | Max order value, max qty per order, max open orders, per-symbol concentration |
| Short selling rules | Disallow naked short in cash segment; auto square-off for MIS |
| Duplicate / wash | Idempotency, self-trade prevention flags |
| Kill switch | Global or per-user, set by risk desk |

**Margin reservation** is the critical atomic step:

```sql
BEGIN;
  -- lock the user's funds row
  SELECT available FROM margin_accounts WHERE user_id = ? FOR UPDATE;
  -- check
  IF available >= required THEN
     UPDATE margin_accounts SET available = available - required, blocked = blocked + required WHERE user_id = ?;
     INSERT INTO margin_blocks(order_id, user_id, amount) VALUES (...);
     INSERT INTO orders(... status='VALIDATED' ...);
     INSERT INTO outbox(...);
  ELSE ROLLBACK/REJECT;
COMMIT;
```

Per-user row lock is fine: one user rarely places >10 orders/sec, and rows are not shared across users. This is one place where **a relational DB with row-level locks beats a clever distributed design**.

> [!NOTE]
> Risk checks must use the **same consistent view of funds** the ledger uses. A separate eventually-consistent "risk cache" will let a user double-spend buying power during bursts. Keep margin state in the OMS DB shard owned by that user.

### Order Gateway to the Exchange (FIX and Native Protocols)

- **FIX (Financial Information eXchange)**: tag=value text protocol (`35=D` NewOrderSingle, `35=8` ExecutionReport, `35=F` Cancel, `35=G` Replace). Used by most brokers/venues in the West and for institutional flow. India's NSE/BSE use **binary native protocols** over TCP (NNF / ETI) for lower latency, but the concepts are identical.
- Sessions are **stateful and sequenced**: each side keeps incrementing `MsgSeqNum`. On reconnect, a **ResendRequest** replays missed messages. This gives at-least-once delivery with detectable gaps.
- Gateway is a **small number of long-lived TCP sessions** to the exchange, not one per user. Exchanges cap per-session throughput (e.g., ~50-200 orders/sec) so brokers buy multiple sessions/trading-member IDs and shard users across them.
- **Exchange order id** (`37=OrderID`) arrives on ack; store it for cancel/replace.
- Gateway is **colocated** or on a leased line to the exchange; stateless OMS talks to it via internal queue.

Gateway design:
```
OMS ──► per-session outbound queue (ordered) ──► FIX engine (QuickFIX/J or custom)
                                                       │ persists seq nums + message store
                                                       ▼
                                                   Exchange
Exchange ──► ExecutionReport ──► parse ──► Kafka "exec-reports" (keyed by order_id) ──► Fill Processor
```

- Keep an **outbound journal** of every message sent (needed for resend and audit).
- **Order of messages per order must be preserved** → partition by `order_id` (or by user) into a single in-order stream.
- Orders are only marked `PENDING_NEW` *after* the gateway has durably queued them; on crash, unsent messages are replayed, sent-but-unacked ones are queried via order status (`35=H`) or reconciled against the exchange's order book/trade file.

### Order Types

| Type | Behavior | Where evaluated |
|------|----------|-----------------|
| **Market** | Execute at best available; may walk the book. Many exchanges convert to market-with-protection (limit at LTP +/- x%) | Exchange |
| **Limit** | Buy at price <= limit, sell at >= limit; rests in the book if not marketable | Exchange |
| **Stop-loss market (SL-M)** | Dormant until LTP crosses trigger, then becomes a market order | Exchange (NSE supports SL natively) or broker-side |
| **Stop-limit (SL)** | Trigger converts to limit order at a given price | Exchange or broker |
| **IOC** | Immediate-or-cancel: fill what you can now, cancel remainder | Exchange |
| **FOK** | Fill entire qty immediately or cancel | Exchange |
| **GTT / GTC / AMO** | Good-till-triggered / cancelled / After-market order. Held by the **broker** until the right time | Broker |
| **Bracket / cover** | Entry + stop-loss + target as linked orders (OCO) | Broker-managed linkage |

Broker-side triggers (GTT, some stop orders when exchange does not support them): a **trigger service** subscribes to ticks and evaluates `price crosses trigger`. Index triggers by symbol in a sorted structure (two heaps: min-heap of buy-triggers, max-heap of sell-triggers). On each tick, pop all crossed triggers and fire them idempotently (`UPDATE ... WHERE status='TRIGGER_PENDING'` ensures single firing). Must tolerate tick gaps: evaluate `crossed` using range between last and current price, not exact equality.

### Fill Processing, Positions and P&L

On an `ExecutionReport` (partial or full):

```
Fill Processor (consumer of exec-reports, partitioned by order_id):
  BEGIN TXN
    1. INSERT trades(exchange_trade_id UNIQUE ...)   -- ON CONFLICT DO NOTHING → idempotent
    2. UPDATE orders SET filled_qty += q, avg_price = ..., status = ..., version+1
    3. UPDATE positions (net_qty, avg_cost, realized_pnl)
    4. LEDGER postings (double-entry): debit client cash, credit settlement obligation,
         debit client cash for brokerage/STT/GST, credit revenue/tax payable
    5. adjust margin_blocks (release the unused part if price better than limit)
    6. INSERT outbox → "order-updated" for push notification
  COMMIT
```

- **Average price** updates on buys: `new_avg = (old_qty*old_avg + q*p) / (old_qty+q)`. On sells: realize `(p - avg_cost) * q`, avg cost unchanged (weighted-average method; FIFO for tax lot reporting).
- **Unrealized P&L** is **not stored**. It is `net_qty * (LTP - avg_cost)`, computed at read time or client-side from the streamed LTP. Storing it would require updating millions of rows per tick.
- **Day P&L** uses previous close as baseline for holdings: `qty * (LTP - prev_close)`.

### Ledger (Double-Entry, Strong ACID)

Every money movement is a **transaction of >= 2 postings that sum to zero**.

```
Buy 10 INFY @ 1500, brokerage 20, STT 15:
 txn 9001:
   CLIENT_CASH (user 7)          credit  15,035      (decrease asset to client)
   EXCHANGE_SETTLEMENT_PAYABLE   debit   15,000
   BROKERAGE_REVENUE             debit      20
   STT_PAYABLE                   debit      15
 Σ debits = Σ credits
```

- **Append-only** rows; corrections are **reversal entries**, never UPDATE/DELETE.
- Each ledger txn has a unique `txn_id` derived from the business event (`fill:{exchange_trade_id}`), so replays are no-ops.
- Run on **Postgres (or MySQL/Aurora) with serializable or READ COMMITTED + explicit row locks**, synchronous replication to a standby. Shard by `user_id`; double-entry across shards is avoided because the counter-party accounts (exchange, revenue) are **house accounts** posted via a separate async aggregation, or live in the same shard through per-shard house sub-accounts.
- Daily **trial balance**: Σ all debits = Σ all credits; bank balance = Σ client cash + house cash. Any break pages the finance on-call.
- Segregation of client funds: client money is held in a separate bank account from broker funds (regulatory requirement).

Why not event-sourced NoSQL for the ledger? You lose multi-row atomic transactions and ad-hoc constraints (`balance >= 0`), which are precisely what you need.

### Idempotency and Exactly-Once

True exactly-once delivery does not exist; we build **at-least-once delivery + idempotent effects**.

| Boundary | Duplicate source | Protection |
|----------|------------------|-----------|
| Client → API | Retries, double taps | `Idempotency-Key`, unique index → return original order |
| OMS DB → Kafka | Relay crash after publish | Transactional outbox; consumers dedupe by `event_id` |
| OMS → Exchange | Gateway restart, resend | Client Order ID (`ClOrdID`) unique per day; exchange rejects duplicates; sequence numbers |
| Exchange → Broker | ResendRequest, replays | Dedupe on `exchange_trade_id` (unique constraint) |
| Fill → Ledger | Consumer re-reads offset | Ledger txn id = business key; `ON CONFLICT DO NOTHING` |

Never "send order, then write DB" or "write DB, then send order" as two independent steps. Write the order + outbox atomically, and send **from** the outbox.

### Market Data Ingest and Fan-out

```
Exchange multicast (UDP) ──► Feed Handler (colocated)
        │ decode binary → normalized tick {token, ltp, ltq, bid/ask depth, volume, ts}
        ▼
   Kafka "ticks" (partition by token)  ──► Tick Store (TimescaleDB / kdb+ / ClickHouse)
        │                                      (charts, candles: 1m/5m/1d via continuous aggregates)
        ▼
   Ticker / Streamer fleet (stateless WebSocket servers)
```

**Conflation**: never push every tick. Most retail UIs need ~1-4 updates per second per symbol.
- Per symbol keep `latest_tick` in memory. A timer (e.g., every 250-500 ms) flushes only symbols that **changed** since last flush.
- Slow client? Drop intermediate values (latest-wins), do **not** queue unbounded. Stale tick is useless; the latest value is all that matters.

**Subscription fan-out** with 100k+ concurrent watchlists:
- Each streamer node holds N websocket connections (say 50-100k with epoll/Netty/Go).
- Node maintains `symbol → set(connections)` in memory. A tick for `INFY` is looked up and written to those connections. Cost is O(subscribers of that symbol).
- Node subscribes to Kafka/Redis pub-sub **only for symbols it has subscribers for** (reference-counted).
- Popular symbols (NIFTY, RELIANCE) have a huge fan-out per node. Serialize the message **once** (binary, e.g., Protobuf or custom packed struct as Zerodha Kite Ticker does with 8-byte-per-field binary), then write the same byte buffer to every socket.
- Scale horizontally: add streamer nodes; use consistent LB by user id so reconnects land predictably; heartbeat every few seconds.
- Compression: binary packed ticks beat JSON by 5-10x; per-message-deflate is optional.
- **Depth (level 2)** only for active order-entry screens: subscribe on demand, 5-level, higher bandwidth.
- Tiered modes like Kite (`ltp`, `quote`, `full`) let clients choose payload size.

Market data is **loss-tolerant** (next tick overwrites); order events are **loss-intolerant**. Use separate channels so a tick storm cannot delay a fill notification.

### Portfolio, Positions and Buying Power Reads

- `/holdings` and `/positions` are read from the OMS DB (primary or sync replica), enriched with LTP from an in-memory cache.
- Cache per-user margin snapshot in Redis **only for display**; never for authorization.
- At open, millions of clients refresh at once: serve LTP from a local in-process cache; avoid DB reads on hot paths other than the user's own rows.

### Settlement (T+1)

```
T:   Trade executed. Broker updates positions (intraday) immediately.
T+0 EOD: Exchange/clearing corporation sends trade files; broker reconciles with its own trades.
         Net obligations per client + broker computed (funds payin/payout, securities payin/payout).
T+1: Securities delivered to buyer's demat (via depository: CDSL / NSDL / DTCC).
     Funds settled with clearing corp. Holdings move from "T1 qty" to settled qty.
```

- Buying a stock: funds blocked at order; debited at trade; shares appear in holdings immediately as **unsettled** (can be sold intraday as BTST at risk) and become settled at T+1.
- Selling: shares **blocked** at order (cannot double sell); sale proceeds usable for new buys immediately (broker-funded with margin) but withdrawable only after settlement.
- Failure to deliver (short delivery) → **auction** by the exchange; broker handles client debit and penalty.
- US moved to T+1 in May 2024; India completed T+1 in Jan 2023 and has optional T+0 for select stocks.
- Corporate actions processed on **ex-date** with ratio adjustment of qty and avg cost; dividends credited as ledger entries on payout date.

### The Matching Engine (Exchange Core Deep Dive)

Even though brokers do not match, interviewers love this. Explain the data structure first, then the system architecture.

**Order book per symbol**: two sides, ordered by **price-time priority**.

```
BIDS (buy)  best = highest price           ASKS (sell)  best = lowest price
 price  queue (FIFO by arrival)             price  queue (FIFO by arrival)
 101.0  [B1:200][B4:50]                     101.5  [S2:100]
 100.5  [B2:300]                            102.0  [S1:400][S3:150]
 100.0  [B3:100]                            102.5  [S4:75]
```

Data structure:
- Each side is a **sorted map of price → FIFO queue of orders** (TreeMap / red-black tree, or a heap of price levels, or a fixed array indexed by tick if the price range is bounded).
- Plus `HashMap<orderId, OrderNode>` for O(1) cancel (each order is a node in a doubly linked list inside its price-level queue).
- Best-price access O(1) (cache pointer to top-of-book), insert at new price O(log P) where P = number of distinct price levels (small, ~100s), insert at existing price O(1), cancel O(1), match O(1) per fill.
- Heaps give O(log n) pop of best price but poor for arbitrary cancels; a sorted map or array-of-levels is preferred in production.

**Matching rule (limit buy order arrives)**:
```
while order.qty > 0 and asks not empty and asks.bestPrice <= order.price:
    resting = asks.bestLevel.head                     # oldest at best price first (time priority)
    fillQty = min(order.qty, resting.qty)
    emit Trade(price = resting.price, qty = fillQty)  # passive order's price wins
    reduce both; if resting.qty == 0 remove from queue; if level empty remove level
if order.qty > 0 and type == LIMIT: rest remainder in bids at order.price (at queue tail)
```
Trades occur at the **resting order's price**. Price improvement goes to the aggressor.

**Architecture: LMAX Disruptor style (single-threaded, event sourced)**

```
 Gateways (FIX/binary) ─► [Ring buffer / Sequencer] ─► Journal (disk, replicated) ─► Matching Engine (1 thread)
   (many threads,           assigns global seq no.        (write ahead, batch fsync)        │ deterministic, in-memory
    parse, validate)        one writer per symbol shard                                      ▼
                                                                              Output ring ─► Market data publisher
                                                                                         ─► Execution reports to gateways
 Hot standby replays the same input journal and stays in lock-step (state machine replication).
```

Why a **single thread per symbol (or symbol shard)**?
- No locks, no contention, no cache-line bouncing → a single core handles millions of orders/sec (LMAX reported ~6M orders/sec at < 1 microsecond latency on a single thread in 2011).
- **Determinism**: same input sequence ⇒ same output. This makes recovery, replication, and testing trivial.
- Everything in RAM; **no DB in the critical path**. Concurrency is replaced by *sequencing*.
- Mechanical sympathy: pre-allocated ring buffer (power-of-two size, sequence numbers instead of locks), no GC pauses (object pools, off-heap, or C++), CPU pinning, busy-spin wait, avoid false sharing with padded counters.

**Scaling**: shard by symbol. Each shard has its own sequencer + engine thread. A symbol never spans shards, so no cross-shard coordination for matching.

**Durability and HA** (event sourcing):
1. Input events are appended to a **journal** (replicated) *before* being matched.
2. State = deterministic fold over the journal; take periodic **snapshots** of the book to bound replay time.
3. **Hot standby** consumes the same journal (via replication/Aeron/Raft-like log) and applies it; on primary failure, standby is promoted within a second or two. Output deduplication by sequence number prevents duplicate execution reports.
4. Disaster recovery site gets the journal asynchronously (or synchronously across two metro DCs).

**Java sketch (price-time priority, single-threaded)**:

```java
final class OrderBook {
    private final TreeMap<Long, Level> bids = new TreeMap<>(Comparator.reverseOrder()); // price in ticks
    private final TreeMap<Long, Level> asks = new TreeMap<>();
    private final Map<Long, Order> byId = new HashMap<>();   // O(1) cancel

    static final class Order { long id, price, qty; boolean buy; Order prev, next; Level level; }
    static final class Level  { long price; Order head, tail; long totalQty; }

    /** Called only from the single matching thread, in sequence-number order. */
    void onNewLimit(Order o, Consumer<Trade> out) {
        TreeMap<Long, Level> opp = o.buy ? asks : bids;
        while (o.qty > 0 && !opp.isEmpty()) {
            Level best = opp.firstEntry().getValue();
            boolean crosses = o.buy ? best.price <= o.price : best.price >= o.price;
            if (!crosses) break;
            while (o.qty > 0 && best.head != null) {
                Order rest = best.head;
                long q = Math.min(o.qty, rest.qty);
                out.accept(new Trade(rest.id, o.id, best.price, q));  // trade at resting price
                o.qty -= q; rest.qty -= q; best.totalQty -= q;
                if (rest.qty == 0) { unlink(rest); byId.remove(rest.id); }
            }
            if (best.head == null) opp.remove(best.price);
        }
        if (o.qty > 0) rest(o);   // add remainder at tail of its price level (time priority)
    }

    void onCancel(long id) {
        Order o = byId.remove(id);
        if (o != null) { unlink(o); if (o.level.head == null) side(o).remove(o.level.price); }
    }
    // unlink(), rest(), side() omitted: O(1) doubly-linked-list ops
}
```

Production notes: use `long` ticks not `double`; replace `TreeMap` with primitive arrays or a skip list to avoid allocation; pre-pool `Order` objects; write trades to a ring buffer instead of invoking a consumer.

**Market order**: matches until filled or book exhausted; remainder is cancelled (or protected by a price collar). **Stop orders**: a separate trigger list; on trade price crossing, inject as new order into the sequencer.

**Fairness**: price-time priority (FIFO) is the default; some venues use pro-rata (futures, options) or size-priority. Matching must be timestamped by the sequencer, not by gateway arrival, to be deterministic.

### Order-at-Open Spike (9:15 AM / Pre-Open Auction)

- Pre-open session (9:00-9:08 NSE): orders collected, **call auction** computes an equilibrium open price that maximizes executed volume; then continuous trading from 9:15.
- Thousands of users submit AMOs / market orders at the same instant; broker sees 10-20x normal QPS for a few seconds.

Mitigations:
- **Pre-scale** gateways and OMS before 9:00 (scheduled autoscaling; do not rely on reactive scaling for a 2-second burst).
- **Admission control**: per-user and global token bucket; reject early with `429` rather than melt.
- Release AMOs from a queue in **paced batches** with the pre-open window, not all in the first millisecond.
- Keep the critical path tiny: auth (cached JWT) → risk (single row lock) → outbox → exchange queue. Defer notifications, analytics, and non-critical writes.
- Shard OMS DB by user id so the burst spreads across shards; use connection pooling (PgBouncer) with a strict cap.
- Cache static data (instrument master, price bands) in-process; refresh on schedule.
- Backpressure the exchange link: bounded per-session queue; apply priority (cancels > modifies > new orders so users can always exit).

### Cancel / Replace Races

An exchange is asynchronous, so a cancel may arrive after the order was already filled.

```
t0  Client sends CANCEL for order O (state OPEN)
t1  Exchange matches O fully (fill report in flight)
t2  Exchange receives CANCEL → rejects ("too late to cancel", 35=9 OrderCancelReject)
t3  Broker receives fill, then cancel-reject → final state FILLED
```

Rules:
- Cancel request moves order to `PENDING_CANCEL`; **do not release margin yet**. Release only on confirmed `CANCELLED`.
- Fills can still arrive in `PENDING_CANCEL`. Accept them.
- **Replace** = cancel + new with a new ClOrdID linked via `OrigClOrdID`. Price change loses time priority at the exchange; qty reduction typically keeps it. Quantity increase or price change resets priority.
- UI shows "cancellation requested" not "cancelled", until confirmed.

### Partial Fills

- Order quantity 100, fills of 30, 50: `filled_qty=80`, status `PARTIALLY_FILLED`; remaining qty continues resting.
- Positions and ledger updated **per fill**, not at order completion.
- Cancelling a partially filled order cancels only the remaining 20; the 80 stay executed.
- Margin for the unfilled part stays blocked until fill or cancel. On price improvement (buy limit 100 filled at 98), release the difference.
- IOC partials: remainder is auto-cancelled by the exchange; broker receives a final report.

### Fractional Shares

Common with Robinhood/Webull/Stash; not on Indian exchanges.

- Exchange only trades whole shares. The broker **aggregates** fractional orders internally: it acts as principal or nets customer orders and trades whole shares in its own omnibus account at the exchange, then **allocates** fractions to customers in an internal ledger.
- Store qty as **decimal(18,8)** or integer in 1e-8 units; money in minor units (no floats).
- Dollar-based orders ("buy $50 of AAPL") convert to qty at execution price; rounding rules and residual (dust) handled by a house account.
- Fractional positions are tracked per customer; the broker's omnibus holds the whole-share total. Reconcile `Σ customer fractions = broker holding ± dust`.
- Fractional shares are typically not transferable out (ACATS) and may execute in batched windows rather than instantly.

### Exchange Link Failure

| Situation | Handling |
|-----------|---------|
| TCP session drops | Reconnect; **Logon with sequence reset off**; exchange resends missed messages (`ResendRequest`); we resume our unacked orders |
| Sequence gap | Detect (`MsgSeqNum` jump), request resend; if irrecoverable, reconcile via order-status queries and trade files |
| No ack for order (unknown state) | Mark `UNKNOWN/PENDING_NEW`; query exchange; **do not** resubmit blindly (could duplicate) |
| Exchange down / trading halt | Reject new orders with clear message; keep open orders as-is; do not cancel locally |
| Primary gateway crashes | Hot standby gateway takes the session (only one active logon per session id), reads persisted seq nums and journal |
| Broker side backlog | Rate-limit new orders; prioritize cancels; shed load at the edge |
| Cancel-on-disconnect | Some venues cancel all open orders on disconnect; know the contract and show users correctly |
| Circuit breaker market-wide halt | Pause routing, show banner, resume on exchange notification |

**Reconciliation** is the safety net: after market close (and intraday for risk), compare broker's orders/trades with the **exchange's trade file** and the clearing corp's obligations. Differences produce **break reports** investigated by ops; unknown fills are booked after verification.

### Audit and Regulatory

- Every order has an immutable event history (who, when, IP/device, price, qty, state changes). Retain 5-8 years; WORM storage (S3 Object Lock).
- **Clock sync** (NTP/PTP, MiFID II requires microsecond-level timestamps for HFT, 100 ms-1 s for retail) so audit trails are comparable with the exchange.
- Contract notes, trade confirmations, daily/ledger statements on a schedule; 
- Surveillance: pattern detection (wash trades, layering, front-running employees), reports to regulator (SEBI, SEC CAT / FINRA OATS equivalent).
- Best execution (US): route to venue giving best price; payment-for-order-flow disclosure (Robinhood's famous controversy).
- Client money segregation, margin collection rules (peak margin penalties in India), appropriate disclosures.
- Data residency: Indian brokers must store financial data in India.

### Security and 2FA

- Login: password + **TOTP / SMS OTP / biometric**; device binding; session expiry daily (SEBI mandates re-login). Enforce 2FA for login and for sensitive actions (withdrawal, bank change).
- **Order authorization**: optional PIN/biometric for order placement; TPIN / e-DIS for selling holdings (depository authorization).
- Tokens short-lived (JWT 15 min + refresh), bound to device; revoke on password change.
- API access: API keys + secret, **IP whitelisting** for algo trading (SEBI mandate), per-key rate limits, static IP.
- Anomaly detection: new device + immediate withdrawal; unusual order sizes; account-takeover patterns (ATO) → step-up auth or freeze.
- Encrypt PII (PAN, Aadhaar, bank) at rest with envelope encryption/KMS; field-level encryption; strict access logs.
- Withdrawals: only to **pre-verified** bank accounts; cool-off period after adding a new account.
- Network: mTLS between services; exchange link on private/leased line; WAF and DDoS protection at edge.

### Storage Choices

| Data | Store | Why |
|------|-------|-----|
| Orders, order events, positions | PostgreSQL / MySQL, sharded by user_id | ACID, row locks, mature tooling |
| Ledger | PostgreSQL (separate cluster), append-only | Double-entry, constraints, audits |
| Margin / funds snapshot (display) | Redis | Fast reads; not source of truth |
| Instrument master, price bands | In-memory + Postgres | Static for the day |
| Live tick state | In-memory on streamer nodes; Redis for last-value cache | Latency |
| Tick history / candles | TimescaleDB / ClickHouse / kdb+ | Time-series compression, aggregates |
| Event bus | Kafka | Ordered per-key, durable, replayable |
| Exchange message journal | Local NVMe + replicated log | Sequence resend, audit |
| Audit archive, contract notes | S3 + Object Lock, Glacier | Cheap, immutable |
| User/KYC documents | Encrypted object store | Compliance |
| Analytics / surveillance | Data warehouse (BigQuery / Redshift) | Offline queries |

### Failure Modes

| Failure | Handling |
|---------|---------|
| OMS node crashes mid-order | Order row + outbox committed atomically; relay resumes; unsent orders re-driven from outbox |
| DB primary fails | Synchronous standby promoted (RPO=0); in-flight requests retried via idempotency key |
| Kafka outage | Outbox accumulates in DB; orders still accepted up to a limit, then 503; no loss |
| Streamer node dies | Clients reconnect (backoff + jitter), resubscribe, receive snapshot then deltas |
| Market data feed gap | Detect via sequence; request retransmission/snapshot; mark quotes stale in UI after N seconds |
| Duplicate execution report | `exchange_trade_id` unique → no-op |
| Fill arrives for unknown order | Park in "orphan fills" queue; alert; ops resolves (may be order sent just before crash) |
| Ledger imbalance detected | Block settlement; page finance; fix via reversal entries |
| Margin cache stale | Not used for authorization; DB row lock is authoritative |
| Fat-finger order | Price collar / max value check rejects it |
| Runaway algo | Per-key rate limit + kill switch + order-to-trade ratio monitors |
| Matching engine primary fails | Hot standby replays journal, promoted, outputs deduped by sequence |
| Region outage | DR site; recover from replicated journal; market-hours runbooks rehearsed |

### Multi-Region and Deployment

- Exchange connectivity is geographically fixed (colocation in Mumbai for NSE). Order plane is **single-primary region** with warm DR elsewhere. Active-active is unnecessary and dangerous for order state.
- Read paths (quotes, charts, static content) are CDN/edge-served and multi-region.
- Deploys: **never during market hours** for the order plane; blue/green outside hours with smoke tests against exchange simulator or pre-open.
- **Chaos and replay testing**: replay yesterday's order flow from the journal against a staging engine; compare outputs byte-for-byte.

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using floats for price/money | Integer minor units / ticks; `decimal` for qty |
| Treating the broker as the matcher | Clarify: broker routes; exchange matches |
| Balance as a mutable column without ledger | Double-entry append-only ledger; derive balance |
| Dual write: DB then Kafka/exchange | Transactional outbox |
| Pushing every tick to every client | Conflate and drop for slow clients |
| Storing unrealized P&L per tick | Compute at read time from LTP |
| Assuming cancel succeeds | `PENDING_CANCEL` state; confirm from exchange |
| Retrying an order whose ack is unknown | Query status first; ClOrdID de-duplication |
| Eventually-consistent risk check | Row lock on user margin in OMS DB |
| Releasing margin on cancel request | Release only on confirmed cancel |
| Locks and DB in matching path | Single-threaded in-memory engine + journal |
| No reconciliation | Daily and intraday recon against exchange files |
| Forgetting audit and regulatory needs | Immutable logs, clock sync, retention |

> [!NOTE]
> Interview framing: **"Correctness on the order path, availability on the data path."** State this and design the two planes with different consistency models: ACID Postgres + outbox for orders and ledger; lossy conflated streaming for quotes.

### Interview Follow-ups

- *"Why not let the broker match orders internally?"* — Regulations require exchange execution for price discovery (and best execution). Internalization exists for market makers / payment-for-order-flow in the US and for fractional shares, but must still respect NBBO.
- *"How do you guarantee an order is not sent twice to the exchange?"* — Idempotency key at API; atomic order+outbox write; unique `ClOrdID` per day; the exchange rejects duplicate ClOrdIDs; on unknown state query status before any resend.
- *"What if the exchange acks but your DB write of the ack fails?"* — The exec report stays in the durable Kafka/journal until processed; consumer retries; idempotent by exchange order id. On restart, reconcile with exchange order status.
- *"How do you avoid double-spending buying power under concurrency?"* — Atomic check-and-reserve in one DB transaction with `SELECT ... FOR UPDATE` on the user's margin row (or a conditional `UPDATE ... WHERE available >= x`). Reservations are releasable and tracked per order.
- *"Why Postgres and not Cassandra/Dynamo for orders?"* — Need multi-row ACID (order + margin + outbox), constraints, and serializable semantics. Volume is low enough (thousands per second) for sharded relational DBs.
- *"How do you shard the OMS?"* — By `user_id` hash; all of a user's orders, margins, positions live on one shard, so every transaction is single-shard. House accounts are aggregated asynchronously.
- *"How does the matching engine achieve millions of orders/sec?"* — Single thread, in-memory, no locks, no I/O in hot path, pre-allocated ring buffer, cache-friendly structures, batched journaling, CPU pinning (LMAX Disruptor design).
- *"Isn't single-threaded a bottleneck / SPOF?"* — Scale by sharding symbols; each shard has hot standby fed by the same deterministic journal. Failover is a promote, not a rebuild.
- *"How is the matching engine made durable without a DB?"* — Event sourcing: replicated append-only input journal + periodic snapshots. State is rebuilt by replay; determinism guarantees identical state.
- *"How do you prevent the engine's journal write from slowing matches?"* — Batch journaling and pipelining: while batch N is fsync'd, the engine processes N+1 provisionally; outputs are released only after the journal ack (or after replica ack).
- *"Heap or TreeMap for the book?"* — A heap gives best price in O(1) but arbitrary cancel is O(n) or needs lazy deletion; sorted map of price levels with FIFO queues gives O(log P) insert of a new level, O(1) cancel via id map, O(1) best. Arrays indexed by tick are fastest when price range is bounded.
- *"How do you handle price-time priority on a modify?"* — Qty decrease keeps priority; price change or qty increase is cancel+new and goes to the back of the queue.
- *"How do you implement stop-loss orders?"* — Either natively on the exchange or broker-side trigger service holding triggers in sorted structures per symbol; fire when last traded price crosses trigger, idempotently once; handle gaps by comparing against price range, not equality.
- *"What happens to stop-loss at a gap-down open?"* — Trigger fires at open; a stop-market executes at whatever price is available (can be far below trigger); a stop-limit may not execute at all. Explain slippage clearly to users.
- *"How do you serve live quotes to 500k users?"* — Stateless streamer nodes; per-symbol subscription maps in memory; binary packed ticks serialized once; conflation (250-500 ms); drop-for-slow-consumer; Kafka partitioned by symbol; add nodes horizontally.
- *"What if a client is slow or offline?"* — Latest-wins buffer; drop intermediate ticks; on reconnect send a snapshot then deltas; cap per-connection buffer and disconnect offenders.
- *"How do you make the 9:15 open not melt your system?"* — Pre-scaling by schedule, admission control, tiny critical path, DB sharding with pool caps, paced AMO release, static data cached, deferred non-critical work, priority for cancels.
- *"How do you handle a corporate action like a 1:2 split?"* — Batch job on ex-date: multiply qty, divide avg cost and open-order prices/quantities; adjust historical charts; post ledger entries for dividends; all idempotent keyed by corporate-action id.
- *"How do you compute P&L at scale?"* — Realized P&L stored at fill time; unrealized derived on read or client-side using streamed LTP; day P&L vs previous close. Never recompute millions of rows per tick.
- *"How do you reconcile with the exchange?"* — Post-close trade files and clearing obligations vs own trades; automated diff; breaks routed to ops; intraday checks via order-status queries and drop-copy FIX sessions.
- *"What is drop copy?"* — A separate read-only FIX session that mirrors all execution reports for risk/reconciliation systems, independent of the order-entry path.
- *"How do you ensure the ledger is correct?"* — Double-entry (Σ=0 per txn), append-only, idempotent txn ids, daily trial balance, bank reconciliation, reversal entries for corrections.
- *"How do you support margin / leverage and square-off?"* — Margin engine computes exposure continuously (positions marked to LTP); at X% margin utilisation issue a margin call; at breach, auto-square-off by sending offsetting market orders (intraday MIS positions at ~3:20 PM regardless).
- *"How do you handle a market-wide circuit breaker?"* — Exchange halts trading at 10/15/20% index moves; broker receives status; rejects/holds new orders; keeps open orders; resumes on exchange signal.
- *"How do you prevent insider/employee front-running?"* — Access controls, segregated duties, audit of employee accounts, surveillance of order timestamps vs employee activity, restricted lists.
- *"How do you test the OMS?"* — Exchange simulator speaking FIX; deterministic replay of recorded flows; property tests on ledger invariants; chaos tests killing gateway mid-flow; load tests replaying 9:15 traffic.
- *"How would you add options / F&O?"* — Add SPAN margin computation, lot sizes, expiry handling, option chain data (large symbol universe), Greeks calculated client-side or in a pricing service, auto-exercise/assignment at expiry, more complex multi-leg orders.
- *"Payment-for-order-flow: how would your routing work?"* — Smart order router evaluates venues (exchange, wholesalers) by price improvement and fill rate; record routing rationale for best-execution audits.
- *"How do you handle a user's two devices placing the same order?"* — Different idempotency keys → two orders; the product can add a duplicate-order warning (same symbol/qty/price within seconds) but must not silently drop.
- *"How do you do fractional shares settlement?"* — Broker holds whole shares in omnibus at the clearing firm; internal ledger allocates fractions; reconcile sum of fractions with omnibus; house account absorbs rounding dust.
- *"Where would you use event sourcing here?"* — Order lifecycle (order_events as source of truth), matching engine journal, ledger as append-only. Not for user profiles.
- *"What are alternatives to Kafka between OMS and gateway?"* — In-process disruptor queue or Aeron for lowest latency; Kafka is fine at tens of thousands of msgs/sec and gives replay/audit. Choose by latency budget.
- *"How do you handle clock skew?"* — Exchange timestamps are authoritative for trade time; broker uses NTP/PTP; sequence numbers, not timestamps, order events.
- *"How would you design for 10x growth?"* — More OMS shards, more exchange sessions/member IDs, more streamer nodes, tick store partitioning, separate clusters per segment (equity / F&O / commodity), read replicas for reports.
