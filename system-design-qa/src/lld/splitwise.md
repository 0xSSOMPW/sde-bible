# Q: Design Splitwise (expense sharing app) - Low-Level Design.

**Answer:**

Splitwise tests three things: modelling money correctly (no floating point, no lost cents), a clean split abstraction (Strategy + Factory), and the **debt simplification** algorithm. Concurrency and the HLD (ledger, idempotency, offline sync) are the usual deep-dive follow-ups.

### Clarifying Questions to Ask

- Only groups, or also friend-to-friend (non-group) expenses? (Both: a "non-group" is a special implicit group.)
- Split types needed? (Equal, Exact amounts, Percentage, Shares.) Multiple payers on one expense?
- Single currency or multi-currency with conversion? Rounding policy?
- Do we simplify debts automatically or on demand? Do we need settlements (payments), edits, deletes, history/audit?
- Notifications, reminders, offline use? Expected scale?

### Requirements

**Functional**
- Register users; create groups; add/remove members.
- Add an expense: who paid, total, how it is split among whom.
- Show balances: per user in a group ("you owe X 300", "Y owes you 120") and overall.
- **Simplify debts** within a group to minimize number of transactions.
- Settle up (record payment between two users).
- Edit/delete an expense (balances must adjust), expense history.

**Non-functional**
- Money is exact: sum of all splits must equal the expense total, to the cent.
- Thread-safe under concurrent expenses in one group.
- Auditable: every balance is explainable by a list of events.
- Idempotent writes (mobile retries).

### Core Entities / Class Diagram

```
 User(id, name, email)
   │ *               *
   └──── Group(id, name, members, ledger)
                │ 1
                │ *
          Expense(id, desc, total, currency, payers[], splitType, splits[], createdAt)
                │ uses
                ▼
   «interface» SplitStrategy ── EqualSplit / ExactSplit / PercentSplit / ShareSplit
                ▲ created by
          SplitStrategyFactory

 Split(userId, amountCents)         Money(cents, currency)
 Settlement(from, to, amount)       Transaction(from, to, amount)   // output of simplify
 BalanceSheet: Map<userId, netCents>   (+ pairwise Map for "who owes whom")
 LedgerEvent (sealed): ExpenseAdded | ExpenseDeleted | SettlementRecorded
```

Net balance convention: `net[u] > 0` -> others owe u; `net[u] < 0` -> u owes others. The sum over a group is always zero (an invariant to assert and test).

### Design Patterns Used and Why

| Pattern | Where | Why |
|---|---|---|
| **Strategy** | `SplitStrategy` | New split types without touching `Expense`/`Group` |
| **Factory** | `SplitStrategyFactory` | Map `SplitType` + inputs to a strategy; keeps callers clean |
| **Observer** | `ExpenseListener` | Notifications, analytics, cache invalidation decoupled |
| **Event Sourcing** | `LedgerEvent` log | Audit, undo/edit, rebuild balances, sync |
| **Facade** | `SplitwiseService` | Single entry point for UI/API |
| **Value Object** | `Money`, `Split` | Immutable, exact arithmetic |

### The Money Rule

Never `double`. Use `long` cents (or `BigDecimal` with scale = currency minor units; JPY=0, KWD=3). Division creates **remainders**: 100.00 / 3 = 33.33 x 3 = 99.99, one cent lost. Rule: compute floors, then distribute the leftover cents one by one (to the first N participants, or by largest fractional remainder, or randomly but deterministically seeded by expense id). Total must reconcile **exactly**.

### Implementation (Java)

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.locks.*;
import java.util.stream.*;

// ===== Value types =====
record Money(long cents, String currency) {
    Money { if (currency == null) throw new IllegalArgumentException(); }
    static Money of(String amount, String cur) {                       // "100.50" -> 10050, no float involved
        return new Money(new java.math.BigDecimal(amount).movePointRight(2)
                .setScale(0, java.math.RoundingMode.UNNECESSARY).longValueExact(), cur);
    }
    @Override public String toString() { return String.format("%s %d.%02d", currency, cents / 100, Math.abs(cents % 100)); }
}

record User(String id, String name) {}
record Split(String userId, long cents) {}

enum SplitType { EQUAL, EXACT, PERCENT, SHARE }

// ===== Strategy =====
interface SplitStrategy {
    /** Returns splits whose cents sum EXACTLY to total. Throws if inputs invalid. */
    List<Split> split(long totalCents, List<String> participants, List<Long> values);
}

class EqualSplit implements SplitStrategy {
    public List<Split> split(long total, List<String> users, List<Long> ignored) {
        int n = users.size();
        long base = total / n, remainder = total % n;               // remainder in [0, n)
        List<Split> out = new ArrayList<>();
        for (int i = 0; i < n; i++)
            out.add(new Split(users.get(i), base + (i < remainder ? 1 : 0)));   // first r users pay 1 cent more
        return out;
    }
}

class ExactSplit implements SplitStrategy {
    public List<Split> split(long total, List<String> users, List<Long> cents) {
        long sum = cents.stream().mapToLong(Long::longValue).sum();
        if (sum != total) throw new IllegalArgumentException("exact amounts sum " + sum + " != total " + total);
        return IntStream.range(0, users.size()).mapToObj(i -> new Split(users.get(i), cents.get(i))).toList();
    }
}

/** Weighted split used by both PERCENT (weights in basis points, sum 10000) and SHARE (any positive ints). */
abstract class WeightedSplit implements SplitStrategy {
    abstract void validate(List<Long> weights);
    public List<Split> split(long total, List<String> users, List<Long> weights) {
        validate(weights);
        long totalWeight = weights.stream().mapToLong(Long::longValue).sum();
        long[] share = new long[users.size()];
        long[] rem = new long[users.size()];                         // fractional part scaled by totalWeight
        long assigned = 0;
        for (int i = 0; i < share.length; i++) {
            long num = total * weights.get(i);                       // fits in long for realistic amounts
            share[i] = num / totalWeight;
            rem[i] = num % totalWeight;
            assigned += share[i];
        }
        long left = total - assigned;                                // < number of users
        Integer[] order = IntStream.range(0, share.length).boxed().toArray(Integer[]::new);
        Arrays.sort(order, (a, b) -> Long.compare(rem[b], rem[a]));  // largest remainder method
        for (int k = 0; k < left; k++) share[order[k]]++;
        return IntStream.range(0, share.length).mapToObj(i -> new Split(users.get(i), share[i])).toList();
    }
}
class PercentSplit extends WeightedSplit {                           // percent given as basis points: 33.33% = 3333
    void validate(List<Long> w) {
        if (w.stream().mapToLong(Long::longValue).sum() != 10_000) throw new IllegalArgumentException("percent must total 100");
    }
}
class ShareSplit extends WeightedSplit {                             // e.g. 2 shares vs 1 share
    void validate(List<Long> w) { if (w.stream().anyMatch(x -> x <= 0)) throw new IllegalArgumentException("shares > 0"); }
}

// ===== Factory =====
final class SplitStrategyFactory {
    private static final Map<SplitType, SplitStrategy> REGISTRY = new EnumMap<>(SplitType.class);
    static {
        REGISTRY.put(SplitType.EQUAL, new EqualSplit());
        REGISTRY.put(SplitType.EXACT, new ExactSplit());
        REGISTRY.put(SplitType.PERCENT, new PercentSplit());
        REGISTRY.put(SplitType.SHARE, new ShareSplit());
    }
    static SplitStrategy get(SplitType t) { return Objects.requireNonNull(REGISTRY.get(t)); }
}
```

```java
// ===== Domain: Expense, Settlement, events =====
record Expense(String id, String description, Money total, Map<String, Long> paidCents, List<Split> splits, long ts) {
    Expense {
        long paid = paidCents.values().stream().mapToLong(Long::longValue).sum();
        long owed = splits.stream().mapToLong(Split::cents).sum();
        if (paid != total.cents() || owed != total.cents()) throw new IllegalArgumentException("paid/owed must equal total");
    }
}
record Settlement(String id, String from, String to, long cents, long ts) {}
record Transaction(String from, String to, long cents) {
    @Override public String toString() { return from + " pays " + to + " " + cents / 100 + "." + String.format("%02d", cents % 100); }
}

sealed interface LedgerEvent permits ExpenseAdded, ExpenseDeleted, SettlementRecorded {}
record ExpenseAdded(Expense expense) implements LedgerEvent {}
record ExpenseDeleted(String expenseId) implements LedgerEvent {}
record SettlementRecorded(Settlement settlement) implements LedgerEvent {}
```

```java
// ===== Group: balances derived from the event log =====
class Group {
    final String id, name;
    private final Set<String> members = new LinkedHashSet<>();
    private final List<LedgerEvent> events = new ArrayList<>();           // append-only source of truth
    private final Map<String, Expense> liveExpenses = new LinkedHashMap<>();
    private final Map<String, Long> net = new HashMap<>();                // projection: + owed to me, - I owe
    private final Map<String, Map<String, Long>> pairwise = new HashMap<>(); // pairwise[debtor][creditor] = cents
    private final Set<String> processedRequestIds = new HashSet<>();      // idempotency
    private final ReadWriteLock lock = new ReentrantReadWriteLock();

    Group(String id, String name, Collection<String> initial) { this.id = id; this.name = name; members.addAll(initial); }

    void addMember(String u) { lock.writeLock().lock(); try { members.add(u); } finally { lock.writeLock().unlock(); } }

    boolean addExpense(String requestId, Expense e) {
        lock.writeLock().lock();
        try {
            if (!processedRequestIds.add(requestId)) return false;       // duplicate retry: no-op
            for (Split s : e.splits()) requireMember(s.userId());
            for (String p : e.paidCents().keySet()) requireMember(p);
            events.add(new ExpenseAdded(e));
            liveExpenses.put(e.id(), e);
            applyExpense(e, +1);
            return true;
        } finally { lock.writeLock().unlock(); }
    }

    void deleteExpense(String expenseId) {
        lock.writeLock().lock();
        try {
            Expense e = liveExpenses.remove(expenseId);
            if (e == null) throw new NoSuchElementException(expenseId);
            events.add(new ExpenseDeleted(expenseId));                   // never mutate history; compensate
            applyExpense(e, -1);
        } finally { lock.writeLock().unlock(); }
    }

    void settle(Settlement s) {
        lock.writeLock().lock();
        try {
            events.add(new SettlementRecorded(s));
            // payer 'from' reduces his debt to 'to'
            adjust(s.from(), s.cents()); adjust(s.to(), -s.cents());
            addPair(s.from(), s.to(), -s.cents());
        } finally { lock.writeLock().unlock(); }
    }

    private void applyExpense(Expense e, int sign) {
        // each participant: net += paid - owed
        Map<String, Long> delta = new HashMap<>();
        e.paidCents().forEach((u, c) -> delta.merge(u, c, Long::sum));
        e.splits().forEach(s -> delta.merge(s.userId(), -s.cents(), Long::sum));
        delta.forEach((u, d) -> adjust(u, sign * d));
        // pairwise: every debtor owes each payer proportional to payer's share of total
        long total = e.total().cents();
        for (Split s : e.splits()) {
            for (var p : e.paidCents().entrySet()) {
                if (s.userId().equals(p.getKey())) continue;
                long part = s.cents() * p.getValue() / total;            // exact for the single-payer common case
                addPair(s.userId(), p.getKey(), sign * part);
            }
        }
    }

    private void adjust(String u, long d) { net.merge(u, d, Long::sum); }
    private void addPair(String debtor, String creditor, long c) {
        // maintain a single signed number per unordered pair: store only debtor->creditor, net against reverse
        Map<String, Long> m = pairwise.computeIfAbsent(debtor, k -> new HashMap<>());
        m.merge(creditor, c, Long::sum);
    }
    private void requireMember(String u) { if (!members.contains(u)) throw new IllegalArgumentException(u + " not in group"); }

    Map<String, Long> balances() {
        lock.readLock().lock();
        try { return Map.copyOf(net); } finally { lock.readLock().unlock(); }
    }

    /** How much 'debtor' owes 'creditor' directly (after netting both directions). */
    long owes(String debtor, String creditor) {
        lock.readLock().lock();
        try {
            long a = pairwise.getOrDefault(debtor, Map.of()).getOrDefault(creditor, 0L);
            long b = pairwise.getOrDefault(creditor, Map.of()).getOrDefault(debtor, 0L);
            return a - b;
        } finally { lock.readLock().unlock(); }
    }

    List<LedgerEvent> events() { lock.readLock().lock(); try { return List.copyOf(events); } finally { lock.readLock().unlock(); } }

    /** Invariant check: balances always sum to zero. */
    boolean consistent() { return balances().values().stream().mapToLong(Long::longValue).sum() == 0; }
}
```

```java
// ===== Debt simplification: greedy min-cash-flow with two heaps =====
final class DebtSimplifier {
    private record Node(String user, long cents) {}

    /**
     * Greedy: repeatedly match the largest creditor with the largest debtor; the smaller of the two is fully settled.
     * Each step settles at least one person => at most (n-1) transactions. O(n log n).
     */
    static List<Transaction> simplify(Map<String, Long> net) {
        PriorityQueue<Node> creditors = new PriorityQueue<>((a, b) -> Long.compare(b.cents, a.cents)); // max-heap
        PriorityQueue<Node> debtors = new PriorityQueue<>((a, b) -> Long.compare(b.cents, a.cents));   // max-heap of |debt|
        net.forEach((u, v) -> {
            if (v > 0) creditors.add(new Node(u, v));
            else if (v < 0) debtors.add(new Node(u, -v));
        });
        List<Transaction> out = new ArrayList<>();
        while (!creditors.isEmpty() && !debtors.isEmpty()) {
            Node cr = creditors.poll(), db = debtors.poll();
            long pay = Math.min(cr.cents, db.cents);
            out.add(new Transaction(db.user, cr.user, pay));
            if (cr.cents > pay) creditors.add(new Node(cr.user, cr.cents - pay));
            if (db.cents > pay) debtors.add(new Node(db.user, db.cents - pay));
        }
        return out;
    }
}
```

> [!NOTE]
> The *minimum* number of transactions is NP-hard (it reduces to partitioning the nets into the maximum number of zero-sum subsets: a group of k zero-sum people needs k-1 payments, so you want as many zero-sum subsets as possible - subset-sum flavour). The greedy gives at most n-1 payments, which is optimal for the common case and a tiny number above optimal otherwise. Exact solutions use bitmask DP O(n * 2^n), fine for n <= ~20 (groups are small).

```java
// ===== Service facade + observer =====
interface ExpenseListener { void onExpenseAdded(Group g, Expense e); }

class SplitwiseService {
    private final Map<String, User> users = new ConcurrentHashMap<>();
    private final Map<String, Group> groups = new ConcurrentHashMap<>();
    private final List<ExpenseListener> listeners = new CopyOnWriteArrayList<>();
    private final AtomicLongIdGen ids = new AtomicLongIdGen();

    static class AtomicLongIdGen { private final java.util.concurrent.atomic.AtomicLong n = new java.util.concurrent.atomic.AtomicLong();
        String next(String p) { return p + n.incrementAndGet(); } }

    User register(String name) { User u = new User(ids.next("u"), name); users.put(u.id(), u); return u; }
    Group createGroup(String name, User... members) {
        Group g = new Group(ids.next("g"), name, Arrays.stream(members).map(User::id).toList());
        groups.put(g.id, g); return g;
    }

    /** values: percent in basis points / shares / exact cents; ignored for EQUAL. */
    Expense addExpense(String requestId, Group g, String desc, Money total, String payerId,
                       SplitType type, List<String> participants, List<Long> values) {
        List<Split> splits = SplitStrategyFactory.get(type).split(total.cents(), participants, values);
        Expense e = new Expense(ids.next("e"), desc, total, Map.of(payerId, total.cents()), splits, System.currentTimeMillis());
        if (g.addExpense(requestId, e)) listeners.forEach(l -> l.onExpenseAdded(g, e));
        return e;
    }
    void addListener(ExpenseListener l) { listeners.add(l); }
}
```

```java
// ===== Demo =====
public class Main {
    public static void main(String[] args) throws Exception {
        SplitwiseService svc = new SplitwiseService();
        User a = svc.register("Alice"), b = svc.register("Bob"), c = svc.register("Carol");
        Group trip = svc.createGroup("Goa trip", a, b, c);
        svc.addListener((g, e) -> System.out.println("notify: " + e.description() + " " + e.total()));

        // Alice pays 100.00 split equally among 3 -> 33.34, 33.33, 33.33 (remainder cent goes to first)
        svc.addExpense("req-1", trip, "Dinner", Money.of("100.00", "INR"), a.id(), SplitType.EQUAL,
                List.of(a.id(), b.id(), c.id()), null);
        // Bob pays 60.00 split 50/30/20 percent
        svc.addExpense("req-2", trip, "Cab", Money.of("60.00", "INR"), b.id(), SplitType.PERCENT,
                List.of(a.id(), b.id(), c.id()), List.of(5000L, 3000L, 2000L));
        // Duplicate retry is ignored
        svc.addExpense("req-2", trip, "Cab", Money.of("60.00", "INR"), b.id(), SplitType.PERCENT,
                List.of(a.id(), b.id(), c.id()), List.of(5000L, 3000L, 2000L));

        System.out.println(trip.balances());                  // sums to zero
        System.out.println(trip.consistent());                // true
        DebtSimplifier.simplify(trip.balances()).forEach(System.out::println);

        trip.settle(new Settlement("s1", c.id(), a.id(), 2000, System.currentTimeMillis()));
        System.out.println("Carol owes Alice: " + trip.owes(c.id(), a.id()));

        // Concurrency smoke test: 1000 concurrent expenses must keep the invariant
        ExecutorService pool = Executors.newFixedThreadPool(8);
        for (int i = 0; i < 1000; i++) {
            final int n = i;
            pool.submit(() -> svc.addExpense("c-" + n, trip, "x", Money.of("10.00", "INR"), a.id(),
                    SplitType.EQUAL, List.of(a.id(), b.id(), c.id()), null));
        }
        pool.shutdown(); pool.awaitTermination(5, TimeUnit.SECONDS);
        System.out.println(trip.consistent());                // still true
    }
}
```

### Simplification Walkthrough

Nets (cents): A +5000, B -2000, C -3000 and D +0.
Creditors: [A 5000]; Debtors: [C 3000, B 2000].
Step 1: A<-C 3000 (C settled, A left 2000). Step 2: A<-B 2000. Result: 2 transactions instead of possibly 3 pairwise ones.

```
Before (pairwise)          After (simplified)
 B ─20─► A                  B ─20─► A
 C ─30─► A                  C ─30─► A     (same here)
 C ─ 5─► B   (chain)        with chains A→B→C collapsed to A→C
```

Chain case: A owes B 10, B owes C 10 -> simplify to A owes C 10 (B drops out). The heap algorithm discovers this because B's net is zero.

### Concurrency Considerations

- **Per-group lock**: expenses within one group touch the same balances, so a `ReadWriteLock` per group serializes writes and allows concurrent reads. Different groups never contend.
- Updating `net` and `pairwise` must be atomic together (invariant: sum of nets = 0), hence the write lock covers both, not independent `ConcurrentHashMap` merges.
- **Idempotency key** (`requestId`) inside the lock - mobile clients retry; second apply is a no-op.
- Edits = delete + add under the same lock (or a single `ExpenseEdited` event) so readers never see the half state.
- Optimistic alternative: version number per group; reject stale writes (compare-and-set), client retries.
- Never compute balances by summing floating values; integers make `merge` associative and commutative, so replay order does not matter.

### Extensibility

- **New split**: implement `SplitStrategy`, register in factory (Open/Closed). E.g., "by item" for restaurant bills with tax/tip allocated proportionally.
- **Multi-currency**: store each expense in its original currency; keep a `Map<currency, net>` per user, or convert at an *as-of-expense-date* rate stored with the expense (never re-convert historic rates silently). Simplify per currency, or settle in a base currency with a snapshot rate.
- **Multiple payers**: `paidCents` map already supports it.
- **Recurring expenses**: scheduler emits `ExpenseAdded` events.
- **Comments/receipts/categories**: fields on `Expense`; receipts stored as object-store URLs.
- **Rounding policy** as an injected `RemainderPolicy` (first-N, largest-remainder, random-by-seed).

### Event-Sourced Ledger

Truth = append-only events (`ExpenseAdded`, `ExpenseDeleted`, `SettlementRecorded`). Balances are a **projection** (cached, rebuildable by replay).

- Audit and "why do I owe this?" comes for free.
- Edit/delete never lose history: append compensating events.
- Bug fix in the projection logic -> replay and fix every balance.
- Sync: clients fetch events after `lastSeq`; snapshots every N events to bound replay.
- Trade-off: complexity, snapshotting, event schema versioning.

### Scaling to Distributed / HLD (short)

```
Mobile/Web ─► API GW ─► Splitwise Service (stateless) ─► PostgreSQL (sharded by group_id)
                              │                             ├─ users, groups, group_members
                              │                             ├─ expenses, expense_splits, expense_payers
                              │                             ├─ ledger_events (group_id, seq, payload)
                              └─► Kafka ─► Notification svc (push/email), Balance projector, Analytics
                    Redis: cached balances per group, idempotency keys (TTL 24h)
```

**Schema (core)**
```
expenses(id PK, group_id, description, total_cents, currency, created_by, created_at, deleted_at)
expense_splits(expense_id, user_id, owed_cents)      PK(expense_id,user_id)
expense_payers(expense_id, user_id, paid_cents)
settlements(id, group_id, from_user, to_user, cents, currency, created_at)
group_balances(group_id, user_id, currency, net_cents)   -- projection, updated in the same txn
idempotency_keys(key PK, response, created_at)
```

- **Sharding**: by `group_id` - all expense/balance/ledger rows of a group co-locate, so a group's write is a single-shard transaction. Users belong to many groups: the "total across groups" view is a scatter-gather or a denormalized per-user summary updated via events (eventual).
- **Consistency**: within a group strong (one DB txn: insert expense + splits + update `group_balances` + append event). Cross-group totals and notifications are eventually consistent.
- **Idempotency**: client-generated `Idempotency-Key` header; unique constraint on key; return stored response on retry.
- **Notifications**: transactional outbox -> Kafka -> push/email workers; batch and debounce ("5 new expenses").
- **Offline sync**: client creates expenses locally with client-generated UUIDs and a local queue; on reconnect, uploads in order; server dedupes by id. Conflicts (edit vs delete) resolved by server sequence number, last-writer-wins on non-financial fields, with deletes as tombstones; client re-pulls events since `lastSeq`.
- **Hot groups** (thousands of members, rare): per-group serialization via a partitioned queue (Kafka key = group_id) keeps ordering without row locks.
- **Reads**: balances cached in Redis; invalidate on event.

### Common Mistakes

| Mistake | Why wrong | Fix |
|---|---|---|
| `double` for amounts | 0.1+0.2 errors, drifting totals | `long` cents or `BigDecimal` |
| `total / n` and ignoring remainder | Cents vanish; sums != total | Floor + distribute leftover |
| Percent splits as `double` | 33.33 x 3 = 99.99 | Basis points + largest remainder |
| Storing pairwise debts only | Cannot simplify, drift | Store net per user as source of truth (plus events) |
| Mutating balances on edit | No audit, cannot undo | Append reversal events |
| Locking only the map, not the update | Torn invariant (sum != 0) | One lock per group around the whole update |
| Simplifying across groups | Merges unrelated debts; surprising to users | Simplify within a group only |
| Greedy by arbitrary order | More transactions | Max creditor with max debtor via heaps |
| No validation (splits not equal to total) | Corrupt ledger | Check in `Expense` constructor |
| Treating settlement as a negative expense hack | Confusing history | Dedicated `Settlement` event |
| Ignoring retries | Double-counted expense | Idempotency key |

### Testing Strategy

- Property tests: for random expenses and strategies, `sum(splits) == total` and `sum(net) == 0`.
- Rounding: 100/3, 1/3, 0.01 / 7, 10000 bps weights with odd totals.
- Simplify: result applied to nets brings everyone to zero; `size <= n - 1`.
- Concurrency: N threads adding expenses; invariant holds; replaying events reproduces balances.
- Idempotency: same `requestId` 100x -> one expense.

### Interview Follow-ups

- *"Why not floats?"* — Binary floating point cannot represent 0.1; errors accumulate and totals would not match to the cent. Use integer minor units or `BigDecimal`.
- *"Why Strategy for splits?"* — Each split type is a different algorithm over the same inputs; the Strategy removes if/else from `Expense`, and a new type is a new class plus one registry entry.
- *"What does the Factory add?"* — Decouples callers (UI/API pass an enum) from concrete classes, centralizes validation and wiring, and can inject dependencies (e.g., rounding policy).
- *"How do you handle the remainder cent?"* — Floor each share, then give the leftover cents to participants by largest fractional remainder (fair) or first-N (simple, deterministic). Total always reconciles.
- *"How do you split $100 among 3 equally?"* — 3334, 3333, 3333 cents. The extra cent goes to a deterministic participant; rotate across expenses if fairness matters.
- *"Explain debt simplification."* — Compute each person's net; creditors vs debtors; repeatedly settle largest debtor against largest creditor. Each step eliminates at least one person -> at most n-1 transactions.
- *"Is that optimal?"* — Not always. Optimal minimization is NP-hard (maximize the number of zero-sum subgroups). Greedy is within a few transactions of optimal; exact bitmask DP for small n.
- *"Why heaps?"* — Need the max of a changing set repeatedly: O(log n) per step, O(n log n) total vs O(n^2) with scans.
- *"Does simplification change what people owe?"* — No: nets are preserved, so everyone ends up with the same net position; only the payment graph changes. Caveat: users may be asked to pay someone they never transacted with - some products make simplify optional.
- *"Who owes whom directly vs simplified?"* — Keep both views: pairwise (explainable) and simplified (efficient payments).
- *"How do you handle multiple payers?"* — `paidCents` per user; net = paid - owed for each. Pairwise attribution is proportional to each payer's share (needs care with rounding; derive from nets if exactness needed).
- *"How to handle expense edits and deletes?"* — Event sourcing: append `ExpenseDeleted`/`ExpenseEdited`; the projection applies the inverse and the new. History retained for audit.
- *"How do you make it thread-safe?"* — Per-group `ReadWriteLock`; updating net and pairwise under one write lock; idempotency keys inside the lock. Different groups are independent.
- *"Why not a global lock?"* — Serializes unrelated groups. Contention is only within a group.
- *"Optimistic vs pessimistic locking?"* — Groups are small and low contention; pessimistic in-memory is fine. In the DB, `SELECT ... FOR UPDATE` on `group_balances` or version-checked updates (optimistic) both work.
- *"Multi-currency?"* — Keep native currency per expense; convert only for display or at settlement using a stored snapshot rate. Per-currency nets avoid retroactive rate drift. Rounding in the target currency's minor unit.
- *"What happens when exchange rates change?"* — Historic expenses keep their original rate; a `settlement` in another currency records the rate used. Revaluation is a reporting concern.
- *"How do you handle a user leaving a group with a non-zero balance?"* — Disallow until settled, or keep as "former member" with a frozen balance.
- *"How do you model settle up?"* — A `Settlement` event adjusting nets by amount (payer's net up, receiver's net down); not an expense. It can be partial.
- *"Design the DB schema."* — expenses, splits, payers, settlements, group_balances (projection), ledger_events, idempotency_keys; shard by group_id.
- *"How would you shard and why group_id?"* — Group is the transactional boundary; co-locating keeps writes single-shard. User-centric aggregates built asynchronously.
- *"How do you show a user's total across all groups?"* — Maintain a denormalized `user_balances` projection updated by events (eventually consistent), or scatter-gather on demand with caching.
- *"How do you guarantee idempotency?"* — Client UUID per create request, unique constraint on `(user, key)`; the server returns the previously stored result on repeat.
- *"How does offline mode work?"* — Local DB, client UUIDs, outbox queue; sync by sequence number; tombstones for deletes; deterministic server merge; recompute local projection from events.
- *"How do you send notifications reliably?"* — Transactional outbox in the same DB txn; relay publishes to Kafka; consumers are idempotent; at-least-once with dedupe.
- *"What if two people add expenses at the same time?"* — Both succeed in some serial order; since adds commute (addition on integers), the final balances are order-independent.
- *"How do you test correctness?"* — Invariants (sum nets = 0, sum splits = total), property-based tests, event replay equals live state, concurrency stress.
- *"How would you add percentage/shares/itemized splits?"* — New strategies; itemized allocates tax and tip proportionally to each person's item subtotal using the same largest-remainder helper.
- *"Rounding with BigDecimal instead of cents?"* — Same idea: `setScale(2, RoundingMode.DOWN)` for each share, then add the difference in cents to chosen participants. Never `HALF_UP` per share (sums drift).
- *"How do you prevent abuse / fraud?"* — Membership checks, rate limits, audit trail, soft deletes, permissions (who may edit whose expenses).
- *"Where does Observer fit?"* — Expense-added events trigger notifications, activity feed, cache invalidation, analytics without coupling to the core.
- *"How would you support 'friends' without a group?"* — Create an implicit 2-person group keyed by the sorted user pair; same code path.
- *"What if payer is not a participant?"* — Allowed: paid > 0, owed = 0; nets handle it naturally.
- *"How do you undo a settlement?"* — Append a reversing `SettlementRecorded` with the opposite direction (or `SettlementVoided`), never delete.
