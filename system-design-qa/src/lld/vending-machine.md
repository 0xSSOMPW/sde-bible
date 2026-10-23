# Q: Design a Vending Machine (low-level design).

**Answer:**

The textbook **State pattern** question. The machine reacts to the same four buttons (insert money, select product, cancel, dispense) very differently depending on what it is currently doing. The interviewer also probes inventory, change-making (greedy vs DP), failure handling (jams, no change), and what changes when you add cashless payments or a fleet of machines.

### Clarifying Questions to Ask

| Question | Why it matters | Assumption if not answered |
|---|---|---|
| Cash only, or cards/UPI too? | Payment abstraction | Coins and notes first, cashless as an extension |
| Which denominations? | Change-making design | 5, 10, 25, 100 cents and 500, 1000 notes (any set pluggable) |
| Does the machine give change? Is the change float limited? | "Exact change only" edge case | Yes, from a finite cash box |
| One product per transaction? | Cart vs single select | Single selection |
| Product layout? | Slot/rack model | Slot code (A1, B3) with price and quantity |
| Is cancel allowed after inserting money? | Refund flow | Yes, full refund |
| What if product is stuck/jammed? | Error handling | Refund and mark slot faulty |
| Admin functions? | Restock, collect cash, set price | Yes, behind admin auth |
| Concurrent users? | Concurrency | One user at a time physically; but admin and telemetry threads run alongside |
| Remote monitoring? | Telemetry | Yes, short HLD section |

> [!NOTE]
> The question that impresses interviewers: "What do we do if we cannot make change?" Raise it yourself before they do.

### Requirements

**Functional**:
- Display products with code, price, and availability.
- Accept coins and notes; track the running balance.
- Select a product; dispense it if balance and stock are sufficient; return change.
- Cancel at any time before dispensing and get a full refund.
- Refuse the sale (and refund) if exact change cannot be given.
- Admin: restock products, refill coin float, collect cash, change prices, take machine out of service.
- Handle hardware errors (dispense jam, coin jam) safely.

**Non-functional**:
- No lost money: every inserted coin ends up in exactly one place (cash box or refund tray).
- Transactions are atomic: either product and change are delivered, or a full refund is.
- State transitions are explicit and illegal operations are rejected.
- Easily extendable with new payment types, products, change algorithms.
- Thread-safe between the customer flow and admin/telemetry threads.

### Core Entities / Class Diagram

```
                    ┌──────────────────────────────┐
                    │        VendingMachine        │  (Context)
                    │ - state: VendingState        │
                    │ - inventory: Inventory       │
                    │ - cashBox: CashBox           │
                    │ - pending: Map<Denom,Int>    │
                    │ - changeMaker                │
                    │ - dispenser (hardware)       │
                    │ + insertMoney / select /     │
                    │   cancel / dispense          │
                    └───────┬──────────────────────┘
                            │ delegates to
                    ┌───────▼────────┐
                    │ VendingState   │ «interface»
                    └───────┬────────┘
       ┌─────────────┬──────┴───────┬───────────────┐
 ┌─────▼────┐  ┌─────▼─────┐  ┌─────▼──────┐  ┌─────▼────────┐
 │IdleState │  │HasMoney   │  │Dispensing  │  │OutOfService  │
 │          │  │State      │  │State       │  │State         │
 └──────────┘  └───────────┘  └────────────┘  └──────────────┘

 Inventory 1──* Slot(code, Product, qty, jammed)    Product(id, name, priceCents)
 CashBox: Map<Denomination, count>
 ChangeMaker «interface» ◄── GreedyChangeMaker, DpChangeMaker
 Dispenser «interface» (hardware) ◄── MotorDispenser, FakeDispenser (tests)
 PaymentProvider «interface» ◄── CashPayment, CardPayment (extension)
 Listener «interface»: telemetry events
```

**State transition diagram**:

```
            insertMoney                  selectProduct(valid, enough $, change OK)
  ┌──────┐ ───────────► ┌──────────┐ ─────────────────────────────► ┌────────────┐
  │ IDLE │              │ HAS_MONEY│                                 │ DISPENSING │
  └──┬───┘ ◄─────────── └────┬─────┘ ◄───── (dispense done / jam ────└─────┬──────┘
     ▲     cancel/refund     │  ▲            refund) both go to IDLE        │
     │                       └──┘ insertMoney (balance grows)               │
     │                                                                      │
     │  admin.putInService                  fatal hardware error / admin    │
     └──────────────── ┌───────────────┐ ◄────────────────────────────────┘
                       │ OUT_OF_SERVICE│
                       └───────────────┘
```

### Design Patterns Used and Why

| Pattern | Where | Why |
|---|---|---|
| **State** | `VendingState` and 4 implementations | Behaviour of every operation differs per state; avoids giant `if (state == ...)` in every method; illegal ops are rejected by the state itself |
| **Strategy** | `ChangeMaker` (greedy, DP) | Algorithm choice depends on denominations and float; swap at runtime |
| **Strategy / Adapter** | `PaymentProvider`, `Dispenser` | Hide hardware and gateway differences |
| **Observer** | `MachineListener` for telemetry/alerts | Low stock, jam, cash-low alerts without coupling to the machine |
| **Singleton** | (Deliberately not used) | A machine is an object you instantiate per device; avoids global state |
| **Command** (optional) | Admin operations (Restock, Collect) | Audit trail and undo |
| **Factory** | `ProductFactory` / `StateFactory` | Build from config; keep states as shared stateless instances |

> [!NOTE]
> Keep the State objects **stateless** and let the context hold the data. That lets you share one instance of each state and keeps the transitions easy to read.

### Code

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.locks.*;

// ---------- Money ----------
enum Denomination {
    NICKEL(5), DIME(10), QUARTER(25), DOLLAR(100), FIVE(500), TEN(1000);
    final int cents;
    Denomination(int c) { cents = c; }
}

final class Money {
    static int total(Map<Denomination, Integer> m) {
        return m.entrySet().stream().mapToInt(e -> e.getKey().cents * e.getValue()).sum();
    }
    static void add(Map<Denomination, Integer> into, Map<Denomination, Integer> from) {
        from.forEach((d, n) -> into.merge(d, n, Integer::sum));
    }
}

// ---------- Domain ----------
class Product {
    final String id, name; final int priceCents;
    Product(String id, String name, int priceCents) { this.id = id; this.name = name; this.priceCents = priceCents; }
    @Override public String toString() { return name + "@" + priceCents; }
}

class Slot {
    final String code; private Product product; private int qty; private boolean jammed;
    Slot(String code, Product p, int qty) { this.code = code; this.product = p; this.qty = qty; }
    Product product() { return product; }
    int qty() { return qty; }
    boolean available() { return !jammed && qty > 0; }
    void decrement() { if (qty == 0) throw new IllegalStateException("empty"); qty--; }
    void restock(int n) { qty += n; }
    void setPrice(int cents) { product = new Product(product.id, product.name, cents); }
    void markJammed(boolean j) { jammed = j; }
    boolean jammed() { return jammed; }
}

class Inventory {
    private final Map<String, Slot> slots = new LinkedHashMap<>();
    void addSlot(Slot s) { slots.put(s.code, s); }
    Optional<Slot> find(String code) { return Optional.ofNullable(slots.get(code)); }
    Collection<Slot> all() { return slots.values(); }
}

// ---------- Exceptions ----------
class VendingException extends RuntimeException { VendingException(String m) { super(m); } }
class DispenseJamException extends RuntimeException { DispenseJamException(String m) { super(m); } }

// ---------- Cash box ----------
class CashBox {
    private final Map<Denomination, Integer> coins = new EnumMap<>(Denomination.class);
    CashBox() { for (Denomination d : Denomination.values()) coins.put(d, 0); }
    synchronized void deposit(Map<Denomination, Integer> m) { Money.add(coins, m); }
    synchronized void withdraw(Map<Denomination, Integer> m) {
        m.forEach((d, n) -> {
            if (coins.get(d) < n) throw new IllegalStateException("float underflow " + d);
            coins.merge(d, -n, Integer::sum);
        });
    }
    synchronized Map<Denomination, Integer> snapshot() { return new EnumMap<>(coins); }
    synchronized Map<Denomination, Integer> collectAll(Denomination keepAtLeast, int keepEach) {
        Map<Denomination, Integer> out = new EnumMap<>(Denomination.class);
        coins.replaceAll((d, n) -> {                       // keep a float of small coins for change
            int keep = d.cents <= keepAtLeast.cents ? Math.min(n, keepEach) : 0;
            out.put(d, n - keep); return keep;
        });
        return out;
    }
}

// ---------- Strategy: change making ----------
interface ChangeMaker {
    /** Returns coins to hand back, or empty if impossible with the available float. */
    Optional<Map<Denomination, Integer>> make(int amount, Map<Denomination, Integer> available);
}

/** Largest denomination first. O(D). Optimal and correct for canonical sets like 1-5-10-25. */
class GreedyChangeMaker implements ChangeMaker {
    public Optional<Map<Denomination, Integer>> make(int amount, Map<Denomination, Integer> avail) {
        Map<Denomination, Integer> out = new EnumMap<>(Denomination.class);
        List<Denomination> ds = new ArrayList<>(List.of(Denomination.values()));
        ds.sort(Comparator.comparingInt((Denomination d) -> d.cents).reversed());
        int left = amount;
        for (Denomination d : ds) {
            int n = Math.min(left / d.cents, avail.getOrDefault(d, 0));
            if (n > 0) { out.put(d, n); left -= n * d.cents; }
        }
        return left == 0 ? Optional.of(out) : Optional.empty();
    }
}

/** Bounded min-coin DP. Works for any denomination set and finite float. O(items * amount). */
class DpChangeMaker implements ChangeMaker {
    public Optional<Map<Denomination, Integer>> make(int amount, Map<Denomination, Integer> avail) {
        List<Denomination> items = new ArrayList<>();                       // expand to individual coins
        avail.forEach((d, n) -> { for (int i = 0; i < Math.min(n, amount / d.cents); i++) items.add(d); });
        int n = items.size(), INF = Integer.MAX_VALUE / 2;
        int[][] dp = new int[n + 1][amount + 1];
        for (int[] row : dp) Arrays.fill(row, INF);
        dp[0][0] = 0;
        for (int i = 1; i <= n; i++) {
            int c = items.get(i - 1).cents;
            for (int a = 0; a <= amount; a++) {
                dp[i][a] = dp[i - 1][a];                                    // skip coin
                if (a >= c && dp[i - 1][a - c] + 1 < dp[i][a]) dp[i][a] = dp[i - 1][a - c] + 1;  // take
            }
        }
        if (dp[n][amount] >= INF) return Optional.empty();
        Map<Denomination, Integer> out = new EnumMap<>(Denomination.class);
        int a = amount;
        for (int i = n; i >= 1; i--) {                                      // reconstruct
            if (dp[i][a] != dp[i - 1][a]) { Denomination d = items.get(i - 1); out.merge(d, 1, Integer::sum); a -= d.cents; }
        }
        return Optional.of(out);
    }
}

// ---------- Hardware abstraction ----------
interface Dispenser {
    void dispense(Slot slot) throws DispenseJamException;
    void returnCoins(Map<Denomination, Integer> coins);
}

class ConsoleDispenser implements Dispenser {
    private final Set<String> jamOn = new HashSet<>();                      // for fault injection in demo/tests
    void injectJam(String code) { jamOn.add(code); }
    public void dispense(Slot s) {
        if (jamOn.contains(s.code)) throw new DispenseJamException("Motor jam at " + s.code);
        System.out.println("  >> dispensed " + s.product().name);
    }
    public void returnCoins(Map<Denomination, Integer> c) { System.out.println("  >> returned " + c + " = " + Money.total(c)); }
}

// ---------- Observer ----------
interface MachineListener { void onEvent(String type, String detail); }

// ---------- State ----------
interface VendingState {
    default void insertMoney(VendingMachine m, Denomination d) { throw new VendingException("Cannot insert money in " + name()); }
    default void selectProduct(VendingMachine m, String code)  { throw new VendingException("Cannot select in " + name()); }
    default void cancel(VendingMachine m)                      { throw new VendingException("Nothing to cancel in " + name()); }
    default void dispense(VendingMachine m)                    { throw new VendingException("Cannot dispense in " + name()); }
    String name();
}

class IdleState implements VendingState {
    public void insertMoney(VendingMachine m, Denomination d) { m.acceptCoin(d); m.setState(m.hasMoney); }
    public void selectProduct(VendingMachine m, String code)  { throw new VendingException("Insert money first"); }
    public void cancel(VendingMachine m) { /* no-op, idempotent */ }
    public String name() { return "IDLE"; }
}

class HasMoneyState implements VendingState {
    public void insertMoney(VendingMachine m, Denomination d) { m.acceptCoin(d); }
    public void selectProduct(VendingMachine m, String code) {
        Slot s = m.inventory().find(code).orElseThrow(() -> new VendingException("Invalid code " + code));
        if (!s.available()) throw new VendingException(s.jammed() ? "Slot faulty" : "Sold out");
        int balance = m.balance();
        if (balance < s.product().priceCents)
            throw new VendingException("Need " + (s.product().priceCents - balance) + " more cents");
        // Change must be makeable from float PLUS the coins just inserted
        int changeDue = balance - s.product().priceCents;
        if (!m.canMakeChange(changeDue)) throw new VendingException("Exact change only; cannot return " + changeDue);
        m.setSelected(s);
        m.setState(m.dispensing);
        m.dispense();                                  // auto-advance
    }
    public void cancel(VendingMachine m) { m.refundAll(); m.setState(m.idle); }
    public String name() { return "HAS_MONEY"; }
}

class DispensingState implements VendingState {
    public void dispense(VendingMachine m) {
        Slot s = m.selected();
        try {
            m.dispenser().dispense(s);
            s.decrement();
            m.commitSale(s);                           // move pending coins to cash box, return change
            m.setState(m.idle);
        } catch (DispenseJamException e) {
            s.markJammed(true);
            m.publish("JAM", s.code);
            m.refundAll();                             // customer never pays for a failed vend
            m.setState(m.idle);
        }
    }
    // everything else is rejected by the defaults: customer cannot cancel mid-motor
    public String name() { return "DISPENSING"; }
}

class OutOfServiceState implements VendingState {
    public void cancel(VendingMachine m) { /* nothing held in this state */ }
    public String name() { return "OUT_OF_SERVICE"; }
}

// ---------- Context ----------
class VendingMachine {
    final VendingState idle = new IdleState(), hasMoney = new HasMoneyState(),
                       dispensing = new DispensingState(), outOfService = new OutOfServiceState();
    private volatile VendingState state = idle;
    private final ReentrantLock lock = new ReentrantLock();                // one customer transaction at a time

    private final Inventory inventory;
    private final CashBox cashBox = new CashBox();
    private final Map<Denomination, Integer> pending = new EnumMap<>(Denomination.class);
    private volatile ChangeMaker changeMaker;
    private final Dispenser dispenser;
    private final List<MachineListener> listeners = new CopyOnWriteArrayList<>();
    private Slot selected;

    VendingMachine(Inventory inv, Dispenser d, ChangeMaker cm) { inventory = inv; dispenser = d; changeMaker = cm; }

    // ----- public API: lock + delegate -----
    void insertMoney(Denomination d)   { locked(() -> state.insertMoney(this, d)); }
    void selectProduct(String code)    { locked(() -> state.selectProduct(this, code)); }
    void cancel()                      { locked(() -> state.cancel(this)); }
    void dispense()                    { state.dispense(this); }            // internal, lock already held

    private void locked(Runnable r) { lock.lock(); try { r.run(); } finally { lock.unlock(); } }

    // ----- admin (same lock, only when not mid-transaction) -----
    void restock(String code, int n) {
        locked(() -> { inventory.find(code).orElseThrow().restock(n); inventory.find(code).get().markJammed(false); });
    }
    void refillFloat(Map<Denomination, Integer> coins) { locked(() -> cashBox.deposit(coins)); }
    Map<Denomination, Integer> collectCash() { lock.lock(); try { return cashBox.collectAll(Denomination.QUARTER, 5); } finally { lock.unlock(); } }
    void setOutOfService(boolean oos) {
        locked(() -> {
            if (oos) { if (state != idle && state != hasMoney) throw new VendingException("Busy"); refundAll(); setState(outOfService); }
            else if (state == outOfService) setState(idle);
        });
    }
    void setChangeMaker(ChangeMaker cm) { changeMaker = cm; }
    void subscribe(MachineListener l) { listeners.add(l); }

    // ----- helpers used by states -----
    void setState(VendingState s) { System.out.println("  [state] " + state.name() + " -> " + s.name()); state = s; }
    VendingState state() { return state; }
    Inventory inventory() { return inventory; }
    Dispenser dispenser() { return dispenser; }
    Slot selected() { return selected; }
    void setSelected(Slot s) { selected = s; }
    int balance() { return Money.total(pending); }
    void acceptCoin(Denomination d) { pending.merge(d, 1, Integer::sum); }
    void publish(String type, String detail) { listeners.forEach(l -> l.onEvent(type, detail)); }

    boolean canMakeChange(int due) {
        if (due == 0) return true;
        Map<Denomination, Integer> avail = cashBox.snapshot();
        Money.add(avail, pending);                      // customer's own coins may be used as change
        return changeMaker.make(due, avail).isPresent();
    }

    void commitSale(Slot s) {
        int due = balance() - s.product().priceCents;
        Map<Denomination, Integer> pool = cashBox.snapshot();
        Money.add(pool, pending);
        Map<Denomination, Integer> change = due == 0 ? Map.of() : changeMaker.make(due, pool)
            .orElseThrow(() -> new IllegalStateException("change vanished"));   // guarded by canMakeChange
        cashBox.deposit(pending);                       // pay in
        cashBox.withdraw(change);                       // pay out
        pending.clear();
        if (!change.isEmpty()) dispenser.returnCoins(change);
        publish("SALE", s.code + ":" + s.product().priceCents);
        if (s.qty() <= 1) publish("LOW_STOCK", s.code);
        selected = null;
    }

    void refundAll() {
        if (pending.isEmpty()) return;
        Map<Denomination, Integer> back = new EnumMap<>(pending);
        pending.clear();                                // clear first: refund can never happen twice
        dispenser.returnCoins(back);
    }
}

// ---------- Demo ----------
public class Main {
    public static void main(String[] args) {
        Inventory inv = new Inventory();
        inv.addSlot(new Slot("A1", new Product("p1", "Chips", 150), 3));
        inv.addSlot(new Slot("A2", new Product("p2", "Soda", 125), 1));
        inv.addSlot(new Slot("B1", new Product("p3", "Candy", 65), 5));
        ConsoleDispenser hw = new ConsoleDispenser();
        VendingMachine vm = new VendingMachine(inv, hw, new GreedyChangeMaker());
        vm.subscribe((t, d) -> System.out.println("  [event] " + t + " " + d));

        Map<Denomination, Integer> floatCoins = new EnumMap<>(Denomination.class);
        floatCoins.put(Denomination.QUARTER, 10); floatCoins.put(Denomination.DIME, 10);
        floatCoins.put(Denomination.NICKEL, 10); floatCoins.put(Denomination.DOLLAR, 5);
        vm.refillFloat(floatCoins);

        System.out.println("1) happy path with change");
        vm.insertMoney(Denomination.DOLLAR); vm.insertMoney(Denomination.DOLLAR);
        vm.selectProduct("A1");                       // 200 - 150 = 50 change

        System.out.println("2) cancel after inserting");
        vm.insertMoney(Denomination.QUARTER); vm.cancel();

        System.out.println("3) insufficient funds, then top up");
        vm.insertMoney(Denomination.DOLLAR);
        try { vm.selectProduct("A1"); } catch (VendingException e) { System.out.println("  ! " + e.getMessage()); }
        vm.insertMoney(Denomination.QUARTER); vm.insertMoney(Denomination.QUARTER);
        vm.selectProduct("A1");

        System.out.println("4) sold out");
        vm.insertMoney(Denomination.DOLLAR); vm.insertMoney(Denomination.QUARTER);
        vm.selectProduct("A2");
        vm.insertMoney(Denomination.DOLLAR); vm.insertMoney(Denomination.QUARTER);
        try { vm.selectProduct("A2"); } catch (VendingException e) { System.out.println("  ! " + e.getMessage()); }
        vm.cancel();

        System.out.println("5) jam -> auto refund");
        hw.injectJam("B1");
        vm.insertMoney(Denomination.DOLLAR);
        vm.selectProduct("B1");

        System.out.println("6) illegal op in IDLE");
        try { vm.selectProduct("A1"); } catch (VendingException e) { System.out.println("  ! " + e.getMessage()); }

        System.out.println("7) out of service");
        vm.setOutOfService(true);
        try { vm.insertMoney(Denomination.DOLLAR); } catch (VendingException e) { System.out.println("  ! " + e.getMessage()); }
        vm.setOutOfService(false);

        // DP vs greedy on a float where greedy fails: need 30 with {25 x1, 10 x3}
        Map<Denomination, Integer> tricky = new EnumMap<>(Denomination.class);
        tricky.put(Denomination.QUARTER, 1); tricky.put(Denomination.DIME, 3);
        System.out.println("greedy=" + new GreedyChangeMaker().make(30, tricky)
            + " dp=" + new DpChangeMaker().make(30, tricky));
    }
}
```

### Walkthrough

1. **Insert money** in `IDLE`: coin goes to the `pending` map (not the cash box yet), state moves to `HAS_MONEY`.
2. **Select product** in `HAS_MONEY`: validate slot, stock, balance, and *can we make the change?* using float plus the customer's pending coins. Only then move to `DISPENSING`.
3. **Dispense**: motor runs. On success: decrement stock, move pending to cash box, withdraw and return change, back to `IDLE`. On jam: mark slot faulty, refund every coin, back to `IDLE`.
4. **Cancel** in `HAS_MONEY`: full refund; `IDLE`.
5. Money lives in `pending` until the sale commits, so a power loss or jam never mixes customer coins with the machine's float.

### Change-Making: Greedy vs DP

| | Greedy | Bounded DP |
|---|---|---|
| Idea | Take the biggest coin that fits | Min-coin subset-sum over the actual coins in the box |
| Complexity | O(D), D = denominations | O(coins x amount) |
| Correct when | Denominations are canonical (1-5-10-25) **and** float is unlimited | Always (finite float, odd denominations) |
| Fails when | Float limited or odd set, e.g. need 30 from one 25 and three 10s: greedy takes 25 then is stuck at 5; DP finds 10+10+10 | Never fails if a solution exists |
| Use | Default, tiny and fast | When the sale was rejected by greedy and you want a second chance, or non-canonical currency |

A pragmatic combo: try greedy first, fall back to DP only if greedy returns empty. Both inputs are tiny (amounts under a few thousand cents), so DP is cheap on an embedded CPU. Divide by the GCD of denominations to shrink the DP table (all cents are multiples of 5 here).

> [!NOTE]
> "Exact change only" is a real vending display. It means the machine decided it cannot guarantee change. Show it proactively when the low-denomination float drops below a threshold, not only after the customer inserted money.

### Concurrency Considerations

- **Physically one customer**, but the software has several actors: the customer UI thread, an admin console, a telemetry/heartbeat thread, and hardware callbacks (coin acceptor, motor done).
- A single `ReentrantLock` around every state-changing call makes transactions atomic and removes the need for per-field synchronisation inside states.
- `state` is `volatile` so telemetry can read it lock-free for health checks.
- `CashBox` methods are `synchronized` because telemetry reads the float for reporting while a sale runs.
- **Admin during a sale**: restock/collect wait on the same lock, so they cannot fire mid-dispense. `setOutOfService` refuses when `DISPENSING`.
- **Hardware callbacks** (coin inserted) should enqueue events to a single-threaded executor that calls the machine, which gives an ordered event loop. Do not call the context from interrupt-style callbacks directly.
- **Timeouts**: a `ScheduledExecutorService` auto-cancels and refunds if the customer walks away in `HAS_MONEY` after N seconds. The timeout takes the same lock and checks state again (it may have changed).
- **Multi-machine** concurrency is not an in-process problem: each machine is an independent aggregate; see HLD below.

### Extensibility

| New requirement | Change |
|---|---|
| Card / UPI / NFC payment | `PaymentProvider` interface; new `PaymentPendingState` between `IDLE` and `DISPENSING` (authorise, then capture only after a successful vend; void on jam) |
| Multiple products per transaction | Add a `Cart`; `HAS_MONEY` accumulates selections; dispense each in turn; refund partial on failure |
| Dynamic pricing / promotions | `PricingPolicy` strategy returning a price for slot and time |
| Loyalty / mobile app unlock | `Authenticator` before `IDLE` becomes `READY` |
| Refrigeration/temperature safety | Sensor listener moves machine to `OUT_OF_SERVICE` above a threshold |
| New currency | New `Denomination` set (or load from config), `DpChangeMaker` |
| Receipt printing | Listener on `SALE` event |
| Admin authentication | `AdminService` facade with roles; admin methods not on the customer interface (Interface Segregation) |
| Coin jam handling | `Dispenser`/`CoinAcceptor` raise errors; machine refunds if possible, else goes `OUT_OF_SERVICE` and raises a ticket |
| Restock suggestions | Telemetry plus analytics service |

> [!NOTE]
> Split the API into `CustomerPanel` (insert/select/cancel) and `AdminPanel` (restock/collect/price). That is Interface Segregation and also a security boundary.

### Error and Jam Handling

| Failure | Handling |
|---|---|
| Motor jam | Refund all, mark slot faulty, emit `JAM`, keep other slots working |
| Repeated jams on many slots | Counter threshold, then `OUT_OF_SERVICE` and alert |
| Coin return jam | Cannot hand back money: log a liability, show "Call service", go out of service |
| Insufficient change | Reject the sale before dispensing; show "Exact change only" |
| Power loss mid-sale | Persist `pending` and `state` in non-volatile storage; on boot, refund pending, or if the motor ran, complete the commit (write-ahead journal) |
| Product sensor says not dropped | Treat as jam (refund) |
| Tampering/vibration | Sensor listener, alarm event, optional lock |

### Scaling to Distributed / HLD

One machine is an embedded device; a company operates tens of thousands. The "system" is a fleet-management platform.

```
 Vending machine (edge)              Cloud
 ┌──────────────────────┐   MQTT/TLS    ┌────────────────┐    ┌──────────────┐
 │ VendingMachine core  │ ────────────► │ IoT Gateway    │───►│ Kafka        │
 │ local DB (SQLite)    │ ◄──────────── │ (device auth)  │    └──┬───┬───┬───┘
 │ sensors, card reader │   commands    └────────────────┘       │   │   │
 └──────────────────────┘ (price, OOS)                           ▼   ▼   ▼
                                                           Telemetry  Sales   Alerts/
                                                           (TSDB)     (OLAP)  Ticketing
        Mobile app / ops dashboard ──► REST API ──► Postgres (fleet, inventory, orders)
```

**Schema**:
```sql
machine(machine_id PK, location, model, status, last_seen)
slot(machine_id, code, product_id, price_cents, qty, capacity, faulty, PRIMARY KEY(machine_id, code))
product(product_id PK, name, sku)
sale(sale_id PK, machine_id, slot_code, price_cents, paid_cents, change_cents, method, ts, status)
cash_float(machine_id, denomination, count, updated_at)
event(event_id PK, machine_id, type, detail JSON, ts)     -- JAM, LOW_STOCK, DOOR_OPEN
restock_visit(visit_id, machine_id, route_id, ts, items JSON)
```

**Points to make**:
- **Offline first**: the machine must sell with no network. Sales are queued locally with a UUID and uploaded with at-least-once delivery; the server dedupes on `sale_id`.
- **Commands from cloud** (change price, disable slot) are versioned and acknowledged; the device pulls on reconnect.
- **Telemetry**: heartbeat every minute, low-stock and jam events immediately; time series DB for temperature and sales rate.
- **Route optimisation**: low-stock events and sales velocity feed a restocking planner so a driver visits only machines that need it.
- **Cashless payments**: tokenised via a gateway (never store card data), authorise at selection, capture on successful dispense, void on jam. Idempotency key is the `sale_id`.
- **Security**: per-device certificates, signed firmware (OTA), admin actions audited.
- **Reconciliation**: nightly compare `sale` totals to physical cash collected and to the float table to detect leakage.

### Common Mistakes

| Mistake | Why it hurts | Fix |
|---|---|---|
| Boolean flags (`hasMoney`, `isDispensing`) and nested ifs | Impossible state combinations, bugs when you add a state | State pattern with explicit transitions |
| States that hold customer data | State objects cannot be shared, easy to leak data between sales | Keep data in the context |
| Putting coins in the cash box before the sale completes | Refund needs to take them back; jam corrupts float | Hold in `pending`, commit on success |
| Dispense first, then check change | Customer got product but cannot get change | Check `canMakeChange` before moving to `DISPENSING` |
| Ignoring that inserted coins can be used as change | Rejects valid sales | Include `pending` in the available pool |
| Greedy only with limited float | Wrongly says "no change" | Bounded DP fallback |
| Not handling a jam | Money lost or stolen | Auto refund, mark slot faulty |
| Refund can run twice | Money out of nowhere | Clear `pending` before returning coins |
| `static` global machine | Cannot test, cannot model a fleet | Instantiate per device and inject hardware |
| Mixing admin and customer API | Anyone can restock or collect cash | Segregate interfaces and require auth |
| Using `double` for prices | Rounding drift | Integer cents |

### Interview Follow-ups

- *"Why State pattern here instead of if/else?"* — Four operations times four states is 16 behaviours. With flags every method becomes a nest of conditions and adding `MAINTENANCE` touches all of them. With State, each class owns its behaviour; adding a state is one class and a few transitions; illegal operations are rejected by default methods.
- *"Who owns the transitions, the state or the context?"* — Here the states trigger transitions (`m.setState`), which keeps each rule next to its behaviour. The alternative is a transition table in the context. I prefer states owning it for readability and a table when states are numerous.
- *"Should state objects be singletons?"* — They can be shared instances because they hold no data; the context carries all mutable data. That also makes them thread-safe.
- *"State vs Strategy, what is the difference?"* — Same structure, different intent. Strategy: the client picks an algorithm and it does not change by itself. State: the object changes its own behaviour by switching the state object as events occur.
- *"Which other patterns do you use?"* — Strategy for change-making, Observer for telemetry, Adapter for hardware (`Dispenser`), optionally Command for admin operations and Factory for building products from config.
- *"What if the customer inserts money and walks away?"* — A timeout schedules an auto-refund (or keeps credit for a short window). The timer takes the machine lock, re-checks the state is still `HAS_MONEY`, then refunds and returns to `IDLE`.
- *"What if we cannot make change?"* — Reject before dispensing, offer cancel/refund, display "Exact change only". Also warn proactively when small-coin float is low and alert the operator.
- *"Greedy or DP for change?"* — Greedy is optimal for canonical denominations with infinite supply, but with a finite float it can wrongly fail (30 from one 25 and three 10s). Bounded DP always finds a solution if one exists. I try greedy, fall back to DP.
- *"What is the complexity of your DP?"* — O(n x A) time and space for n physical coins and amount A. Reduce A by dividing by the GCD of denominations and cap n by `amount/denomination`. Fine for embedded.
- *"How do you minimise coin usage or preserve small coins?"* — Min-coin DP minimises count. To preserve scarce denominations, weight their cost higher, turning it into a min-cost bounded knapsack.
- *"What if a user inserts a counterfeit or unsupported coin?"* — The acceptor hardware rejects it and returns it; software only sees valid denominations. Invalid values never touch `pending`.
- *"How do you guarantee no money is lost?"* — Money is only in `pending`, `cashBox`, or the refund tray; moves happen in `commitSale`/`refundAll` which clear `pending` first. A write-ahead journal covers power loss.
- *"What if the power fails during dispensing?"* — Persist state and `pending` before moving to `DISPENSING`, and a journal entry after the motor sensor confirms. On reboot: no confirmation means refund; confirmation means finish the commit.
- *"How do you test this?"* — Unit test each state with a fake `Dispenser` and a tiny inventory: assert allowed and rejected operations per state; inject jams; compare greedy and DP on tricky floats; property test that total money in equals total money out across random sequences; use a fake clock for timeouts.
- *"How would you test the jam scenario?"* — `ConsoleDispenser.injectJam("B1")`; assert refund equals what was inserted, slot is faulty, state is `IDLE`, and `JAM` event emitted.
- *"How do you add card payments?"* — A `PaymentProvider` interface. Card flow: selection, authorise (hold), dispense, capture; on jam, void the authorisation. Add a `PaymentPendingState`; the cash path is unchanged. Machines often accept both so keep the pending credit abstract (`Credit` rather than coin map).
- *"What if I want multi-item carts?"* — Introduce `Cart` in context; `HAS_MONEY` allows `addToCart`; checkout validates total and change; dispensing loops, tracks which items were delivered, and refunds the unfulfilled items on failure.
- *"How do you handle admin restock concurrently with a sale?"* — Same lock, so restock waits until the transaction ends. For very long restock visits put the machine `OUT_OF_SERVICE` or `MAINTENANCE` so customers are told.
- *"How do you secure admin functions?"* — Separate `AdminPanel` interface; authentication (PIN, keycard, signed token); audit log of who restocked or collected cash; cloud-side RBAC.
- *"Which SOLID principle does this show?"* — O: new state or change algorithm without editing existing ones. S: each state one responsibility, `CashBox` one job. L: any `ChangeMaker` or `Dispenser` is substitutable. I: customer and admin panels split. D: context depends on `Dispenser` and `ChangeMaker` abstractions.
- *"How is inventory modelled and why not a Map<Product,Integer>?"* — Real machines have slots (coils) with capacity, a price per slot, and per-slot faults. The same product can be in two slots. A `Slot` entity captures this.
- *"How would you do dynamic pricing?"* — A `PricingPolicy` strategy consulted at selection time (time of day, temperature, expiry). The displayed price must equal the charged price, so lock the price at selection.
- *"How do you track low stock and send alerts?"* — `commitSale` publishes `LOW_STOCK` when qty crosses a threshold; the listener forwards to a cloud alert; a restock planner batches visits.
- *"What telemetry would you collect?"* — Heartbeat, state, per-slot qty, float by denomination, temperature, door open, jam counts, sales events, firmware version, connectivity.
- *"How does the machine sync with the cloud if offline?"* — Local SQLite as source of truth; outbox table of unsent events; retry with backoff; server dedupes on `sale_id`; commands from the cloud carry version numbers.
- *"How would you model out-of-service?"* — A state that rejects everything except admin `putInService`, entered by admin command or fatal errors (coin return jam, overheating). Entry action refunds pending credit.
- *"What happens if someone selects while in DISPENSING?"* — Rejected by the default method: "Cannot select in DISPENSING". Optionally queue it, but vending is one transaction at a time.
- *"Can the customer cancel while the product is dropping?"* — No. After the motor is committed, cancel is rejected; this prevents refund plus product. The state's default `cancel` throws.
- *"How would you design the display?"* — Observer on the machine: state and balance changes publish messages ("Insert coins", "Balance 1.50", "Sold out"). The view subscribes and never reads internal fields.
- *"How do you handle exact price vs rounding and currency?"* — Integer minor units everywhere, a `Currency` value type if multi-currency, rounding only at presentation.
- *"What if two coins are inserted at the exact same time?"* — The coin acceptor serialises them physically; in software the machine lock serialises the calls, so each `acceptCoin` is atomic.
- *"Would you use a state machine library?"* — For two or three transitions no. For a larger lifecycle (payment, maintenance, remote lock) a table-driven FSM (Spring Statemachine or hand-written enum-based) makes transitions auditable. The State pattern is what interviewers expect to see.
- *"How would you implement an undo of the last transaction?"* — Command objects record effects (coins moved, stock decremented) so a compensating command can reverse them; in practice vending machines do not undo, they refund via support, so I would log a `sale` row and a `refund` row instead.
- *"How would the design change for a fleet of 100,000 machines?"* — Separate the device software (this design) from the cloud platform: MQTT ingestion, Kafka, time-series for telemetry, Postgres for inventory and orders, OLAP for analytics, device shadow for commands. Partition by `machine_id`.
- *"How do you prevent replay or duplicate sales upload?"* — `sale_id` UUID generated on the device; server insert is idempotent (unique constraint) so retries are safe.
- *"How do you roll out firmware safely?"* — Signed OTA, staged rollout by percentage, automatic rollback if the heartbeat fails after update, and never update while a sale is in progress (only in `IDLE`).
