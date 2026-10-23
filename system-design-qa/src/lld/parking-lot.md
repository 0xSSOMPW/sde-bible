# Q: Design a Parking Lot (low-level design).

**Answer:**

The classic OOD question. The interviewer is not testing whether you know "parking"; they are testing whether you can turn a vague story into entities, pick two or three patterns for the parts that *vary* (pricing, spot selection), keep the hot path thread-safe, and then explain how it grows into a real service.

### Clarifying Questions to Ask

| Question | Why it matters | Assumption if not answered |
|---|---|---|
| One lot or many? Multi-floor? | Decides whether `ParkingLot` is a singleton or an entity with an id | One lot, multiple floors, designed so a `lotId` can be added |
| Which vehicle types? | Drives the vehicle-to-spot compatibility rule | Motorcycle, Car, Truck, EV car |
| Fixed spot sizes? Can a small vehicle use a big spot? | Allocation fallback logic | Yes, a vehicle may use a larger spot if its own type is full |
| How is price computed? | Strategy pattern | Hourly by vehicle type; flat and dynamic pluggable |
| Payment at exit or entry? Cash/card/UPI? | Payment abstraction | Pay at exit, multiple methods |
| Multiple entry/exit gates, concurrent? | Concurrency design | Yes, N entry and M exit gates |
| Reservations / pre-booking? | Extra spot state | Optional extension, sketched below |
| EV charging? | Special spot type with charge session | Yes, EV spots with a per-kWh charge on top of parking |
| Lost ticket policy? | Edge case at exit | Flat penalty fee |
| Display board? | Observer pattern | Per floor availability counts |
| Scale: single building or chain of lots? | Whether to cover HLD | Mention HLD at the end |

> [!NOTE]
> Ask at most 5-6 of these out loud. Spend the saved time on code. State the rest as assumptions on the whiteboard.

### Requirements

**Functional**:
- Lot has multiple floors; each floor has spots of different types (motorcycle, compact, large, EV).
- Vehicle enters through an entry gate; system finds a compatible free spot and issues a ticket.
- Vehicle exits through an exit gate; system computes fee from ticket time, takes payment, frees the spot.
- Pricing is pluggable: hourly, flat, dynamic (peak/off-peak).
- Spot allocation is pluggable: nearest-to-entry, first-free, least-loaded floor.
- Display board shows free spots per type per floor.
- EV spots track a charging session and bill it.
- (Extension) Reservations hold a spot for a time window.

**Non-functional**:
- Two vehicles at two gates must never get the same spot (correctness over speed).
- Entry decision under 100 ms.
- Easy to add a new vehicle type, pricing rule, or payment method without editing core classes (Open/Closed).
- Fail safe: if payment fails, vehicle is not released and spot is not freed.
- Auditable: every ticket, payment, and spot transition is recorded.

### Core Entities / Class Diagram

```
                     ┌────────────────────────┐
                     │  ParkingLot (singleton)│
                     │  floors, gates, config │
                     └──────┬─────────────────┘
          has many          │ has many
        ┌───────────────────┼───────────────────────┐
        ▼                   ▼                       ▼
 ┌─────────────┐     ┌────────────┐         ┌──────────────┐
 │ ParkingFloor│     │ EntryGate  │         │  ExitGate    │
 │  spots      │     │ issue()    │         │  process()   │
 └──────┬──────┘     └─────┬──────┘         └──────┬───────┘
        │ has many         │ uses                  │ uses
        ▼                  ▼                       ▼
 ┌─────────────┐   ┌───────────────────┐   ┌──────────────────┐
 │ ParkingSpot │   │SpotAllocationStrat│   │ PricingStrategy  │
 │ type,state  │   │ «interface»       │   │ «interface»      │
 └─────────────┘   └───────────────────┘   └──────────────────┘
        ▲            ▲        ▲                ▲      ▲      ▲
        │      Nearest   FirstFree        Hourly   Flat   Dynamic
        │
   ┌────┴─────┐      ┌──────────┐     ┌──────────────┐
   │ EVSpot   │      │ Ticket   │────►│ Vehicle      │ (Car, Bike, Truck, EVCar)
   └──────────┘      │ id,spot, │     └──────────────┘
                     │ entryTime│
                     └────┬─────┘
                          ▼
                     ┌──────────┐   ┌──────────────────┐
                     │ Payment  │──►│ PaymentProcessor │ «interface»
                     └──────────┘   └──────────────────┘

  DisplayBoard  ◄── observes ── ParkingFloor (spot state changes)
```

Key relationships:
- `ParkingLot` 1..* `ParkingFloor` 1..* `ParkingSpot`.
- `Ticket` references one `ParkingSpot` and one `Vehicle`; it is the unit of billing.
- Strategies are injected, never `if/else`'d.

### Design Patterns Used and Why

| Pattern | Where | Why |
|---|---|---|
| **Strategy** | `PricingStrategy`, `SpotAllocationStrategy` | Pricing and allocation rules change per business; swap without touching gates |
| **Singleton** | `ParkingLot` | One lot per JVM, global access point (but inject it in tests, see mistakes) |
| **Factory** | `VehicleFactory`, `PricingStrategyFactory` | Create objects from input (plate scan, config) without callers knowing concrete classes |
| **Observer** | `DisplayBoard` observes floors | Free-spot counts update without the floor knowing about displays |
| **State** (light) | `SpotState` (FREE, OCCUPIED, RESERVED, OUT_OF_SERVICE) | Spot behaviour depends on state; enum plus CAS transitions is enough |
| **Template/Decorator** | EV charging fee added on top of parking fee | Compose fees without subclass explosion |
| **Command** (optional) | `ExitCommand` for audit/undo | Mentioned in follow-ups |

> [!NOTE]
> Do not force all patterns in. Interviewers prefer two patterns justified well (Strategy for price, Strategy for allocation) over six name-dropped ones.

### Code

```java
import java.time.*;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.util.stream.*;

// ---------- Enums ----------
enum VehicleType { MOTORCYCLE, CAR, TRUCK, EV_CAR }

enum SpotType { MOTORCYCLE, COMPACT, LARGE, EV }

enum SpotState { FREE, OCCUPIED, RESERVED, OUT_OF_SERVICE }

enum PaymentMethod { CASH, CARD, UPI }

enum PaymentStatus { SUCCESS, FAILED }

// ---------- Vehicle ----------
abstract class Vehicle {
    private final String plate;
    private final VehicleType type;
    protected Vehicle(String plate, VehicleType type) { this.plate = plate; this.type = type; }
    public String plate() { return plate; }
    public VehicleType type() { return type; }
    /** Spot types this vehicle can use, in preference order. */
    public abstract List<SpotType> compatibleSpots();
}

class Motorcycle extends Vehicle {
    Motorcycle(String p) { super(p, VehicleType.MOTORCYCLE); }
    public List<SpotType> compatibleSpots() {
        return List.of(SpotType.MOTORCYCLE, SpotType.COMPACT, SpotType.LARGE);
    }
}
class Car extends Vehicle {
    Car(String p) { super(p, VehicleType.CAR); }
    public List<SpotType> compatibleSpots() { return List.of(SpotType.COMPACT, SpotType.LARGE); }
}
class Truck extends Vehicle {
    Truck(String p) { super(p, VehicleType.TRUCK); }
    public List<SpotType> compatibleSpots() { return List.of(SpotType.LARGE); }
}
class EVCar extends Vehicle {
    EVCar(String p) { super(p, VehicleType.EV_CAR); }
    // prefers an EV spot (charger), falls back to a normal one with no charging
    public List<SpotType> compatibleSpots() {
        return List.of(SpotType.EV, SpotType.COMPACT, SpotType.LARGE);
    }
}

class VehicleFactory {
    static Vehicle create(VehicleType t, String plate) {
        return switch (t) {
            case MOTORCYCLE -> new Motorcycle(plate);
            case CAR -> new Car(plate);
            case TRUCK -> new Truck(plate);
            case EV_CAR -> new EVCar(plate);
        };
    }
}

// ---------- Spot (thread-safe via CAS on state) ----------
class ParkingSpot {
    private final String id;
    private final SpotType type;
    private final int floor;
    private final int distanceFromEntry;           // used by NearestStrategy
    private final AtomicReference<SpotState> state = new AtomicReference<>(SpotState.FREE);
    private volatile Vehicle parked;

    ParkingSpot(String id, SpotType type, int floor, int distanceFromEntry) {
        this.id = id; this.type = type; this.floor = floor; this.distanceFromEntry = distanceFromEntry;
    }
    String id() { return id; }
    SpotType type() { return type; }
    int floor() { return floor; }
    int distance() { return distanceFromEntry; }
    SpotState state() { return state.get(); }
    boolean isFree() { return state.get() == SpotState.FREE; }

    /** Atomically claim the spot. Only one caller can win. */
    boolean tryOccupy(Vehicle v) {
        if (state.compareAndSet(SpotState.FREE, SpotState.OCCUPIED)) { parked = v; return true; }
        return false;
    }
    boolean tryReserve() { return state.compareAndSet(SpotState.FREE, SpotState.RESERVED); }
    boolean occupyReserved(Vehicle v) {
        if (state.compareAndSet(SpotState.RESERVED, SpotState.OCCUPIED)) { parked = v; return true; }
        return false;
    }
    void release() { parked = null; state.set(SpotState.FREE); }
    void markOutOfService() { state.set(SpotState.OUT_OF_SERVICE); }
    boolean supportsCharging() { return type == SpotType.EV; }
}

// ---------- Observer: floor publishes, display board subscribes ----------
interface SpotListener { void onChange(int floor, SpotType type, int freeNow); }

class DisplayBoard implements SpotListener {
    private final Map<String, Integer> counts = new ConcurrentHashMap<>();
    @Override public void onChange(int floor, SpotType type, int freeNow) {
        counts.put("F" + floor + "-" + type, freeNow);
    }
    String render() {
        return counts.entrySet().stream().sorted(Map.Entry.comparingByKey())
            .map(e -> e.getKey() + "=" + e.getValue()).collect(Collectors.joining(" | "));
    }
}

// ---------- Floor ----------
class ParkingFloor {
    private final int number;
    private final List<ParkingSpot> spots = new ArrayList<>();
    private final Map<SpotType, AtomicInteger> freeCount = new EnumMap<>(SpotType.class);
    private final List<SpotListener> listeners = new CopyOnWriteArrayList<>();

    ParkingFloor(int number) {
        this.number = number;
        for (SpotType t : SpotType.values()) freeCount.put(t, new AtomicInteger());
    }
    void addSpot(ParkingSpot s) { spots.add(s); freeCount.get(s.type()).incrementAndGet(); }
    void subscribe(SpotListener l) { listeners.add(l); }
    int number() { return number; }
    List<ParkingSpot> spots() { return Collections.unmodifiableList(spots); }
    int free(SpotType t) { return freeCount.get(t).get(); }

    void spotTaken(ParkingSpot s) { notifyChange(s.type(), freeCount.get(s.type()).decrementAndGet()); }
    void spotFreed(ParkingSpot s) { notifyChange(s.type(), freeCount.get(s.type()).incrementAndGet()); }
    private void notifyChange(SpotType t, int now) { listeners.forEach(l -> l.onChange(number, t, now)); }
}

// ---------- Strategy: spot allocation ----------
interface SpotAllocationStrategy {
    /** Candidates ordered by preference; caller tries to claim them in order. */
    List<ParkingSpot> rank(List<ParkingFloor> floors, Vehicle v);
}

class NearestSpotStrategy implements SpotAllocationStrategy {
    public List<ParkingSpot> rank(List<ParkingFloor> floors, Vehicle v) {
        List<SpotType> pref = v.compatibleSpots();
        return floors.stream().flatMap(f -> f.spots().stream())
            .filter(s -> s.isFree() && pref.contains(s.type()))
            .sorted(Comparator.<ParkingSpot>comparingInt(s -> pref.indexOf(s.type()))  // exact type first
                .thenComparingInt(ParkingSpot::floor)
                .thenComparingInt(ParkingSpot::distance))
            .collect(Collectors.toList());
    }
}

class FirstFreeSpotStrategy implements SpotAllocationStrategy {
    public List<ParkingSpot> rank(List<ParkingFloor> floors, Vehicle v) {
        List<SpotType> pref = v.compatibleSpots();
        List<ParkingSpot> out = new ArrayList<>();
        for (SpotType t : pref)
            for (ParkingFloor f : floors)
                for (ParkingSpot s : f.spots()) if (s.isFree() && s.type() == t) out.add(s);
        return out;
    }
}

// ---------- Ticket ----------
class Ticket {
    private static final AtomicLong SEQ = new AtomicLong(1000);
    private final String id = "T" + SEQ.incrementAndGet();
    private final Vehicle vehicle;
    private final ParkingSpot spot;
    private final Instant entryTime;
    private volatile Instant exitTime;
    private volatile Instant chargeStart;     // EV only
    private volatile double kwhConsumed;

    Ticket(Vehicle v, ParkingSpot s, Instant now) { vehicle = v; spot = s; entryTime = now; }
    String id() { return id; }
    Vehicle vehicle() { return vehicle; }
    ParkingSpot spot() { return spot; }
    Instant entryTime() { return entryTime; }
    Instant exitTime() { return exitTime; }
    void close(Instant t) { exitTime = t; }
    void startCharging(Instant t) { chargeStart = t; }
    void addKwh(double kwh) { kwhConsumed += kwh; }
    double kwh() { return kwhConsumed; }
}

// ---------- Strategy: pricing ----------
interface PricingStrategy {
    double price(Ticket t, Instant exit);
}

class HourlyPricing implements PricingStrategy {
    private final Map<VehicleType, Double> ratePerHour;
    HourlyPricing(Map<VehicleType, Double> rates) { this.ratePerHour = rates; }
    public double price(Ticket t, Instant exit) {
        long minutes = Math.max(1, Duration.between(t.entryTime(), exit).toMinutes());
        long hours = (minutes + 59) / 60;                       // round up, minimum 1 hour
        return hours * ratePerHour.getOrDefault(t.vehicle().type(), 20.0);
    }
}

class FlatPricing implements PricingStrategy {
    private final double flat;
    FlatPricing(double flat) { this.flat = flat; }
    public double price(Ticket t, Instant exit) { return flat; }
}

/** Decorator-style: wraps another strategy and applies a peak multiplier. */
class DynamicPricing implements PricingStrategy {
    private final PricingStrategy base;
    private final java.util.function.Supplier<Double> occupancy;   // 0.0 .. 1.0
    DynamicPricing(PricingStrategy base, java.util.function.Supplier<Double> occupancy) {
        this.base = base; this.occupancy = occupancy;
    }
    public double price(Ticket t, Instant exit) {
        double occ = occupancy.get();
        double surge = occ > 0.9 ? 1.5 : occ > 0.7 ? 1.2 : 1.0;
        return base.price(t, exit) * surge;
    }
}

/** Adds EV charging on top of parking fee. */
class EvChargingSurcharge implements PricingStrategy {
    private final PricingStrategy base;
    private final double perKwh;
    EvChargingSurcharge(PricingStrategy base, double perKwh) { this.base = base; this.perKwh = perKwh; }
    public double price(Ticket t, Instant exit) { return base.price(t, exit) + t.kwh() * perKwh; }
}

// ---------- Payment ----------
interface PaymentProcessor { PaymentStatus pay(String ticketId, double amount); }

class CashPayment implements PaymentProcessor {
    public PaymentStatus pay(String id, double amt) { return PaymentStatus.SUCCESS; }
}
class CardPayment implements PaymentProcessor {
    public PaymentStatus pay(String id, double amt) {
        // call gateway with idempotency key = ticket id
        return amt > 0 ? PaymentStatus.SUCCESS : PaymentStatus.FAILED;
    }
}
class PaymentFactory {
    static PaymentProcessor of(PaymentMethod m) {
        return switch (m) { case CASH -> new CashPayment(); case CARD, UPI -> new CardPayment(); };
    }
}

// ---------- Exceptions ----------
class LotFullException extends RuntimeException {
    LotFullException(String m) { super(m); }
}
class InvalidTicketException extends RuntimeException {
    InvalidTicketException(String m) { super(m); }
}
class PaymentFailedException extends RuntimeException {
    PaymentFailedException(String m) { super(m); }
}

// ---------- Parking lot (singleton, thread-safe) ----------
class ParkingLot {
    private static volatile ParkingLot instance;
    private final List<ParkingFloor> floors = new ArrayList<>();
    private final Map<String, Ticket> active = new ConcurrentHashMap<>();   // ticketId -> Ticket
    private final Set<String> platesInside = ConcurrentHashMap.newKeySet();
    private volatile SpotAllocationStrategy allocation = new NearestSpotStrategy();
    private volatile PricingStrategy pricing;
    private final Clock clock;

    private ParkingLot(Clock clock) { this.clock = clock; }

    static ParkingLot getInstance() {
        if (instance == null) {
            synchronized (ParkingLot.class) {
                if (instance == null) instance = new ParkingLot(Clock.systemUTC());
            }
        }
        return instance;
    }
    static ParkingLot forTest(Clock c) { return new ParkingLot(c); }   // avoids singleton in tests

    void addFloor(ParkingFloor f) { floors.add(f); }
    void setAllocation(SpotAllocationStrategy s) { allocation = s; }
    void setPricing(PricingStrategy p) { pricing = p; }
    List<ParkingFloor> floors() { return floors; }

    double occupancy() {
        long total = floors.stream().mapToLong(f -> f.spots().size()).sum();
        long free = floors.stream().flatMap(f -> f.spots().stream()).filter(ParkingSpot::isFree).count();
        return total == 0 ? 0 : 1.0 - (double) free / total;
    }

    /** Entry gate calls this. */
    Ticket park(Vehicle v) {
        if (!platesInside.add(v.plate()))
            throw new IllegalStateException("Vehicle already inside: " + v.plate());
        try {
            // Optimistic: rank, then CAS each candidate. A loser simply tries the next one.
            for (int attempt = 0; attempt < 3; attempt++) {
                for (ParkingSpot s : allocation.rank(floors, v)) {
                    if (s.tryOccupy(v)) {
                        floorOf(s).spotTaken(s);
                        Ticket t = new Ticket(v, s, clock.instant());
                        active.put(t.id(), t);
                        return t;
                    }
                }
            }
            throw new LotFullException("No spot for " + v.type());
        } catch (RuntimeException e) {
            platesInside.remove(v.plate());        // roll back on failure
            throw e;
        }
    }

    /** Exit gate calls this. Spot is freed only after payment succeeds. */
    double exit(String ticketId, PaymentMethod method) {
        Ticket t = active.get(ticketId);
        if (t == null) throw new InvalidTicketException("Unknown or already used: " + ticketId);
        Instant now = clock.instant();
        double fee = pricing.price(t, now);
        PaymentStatus st = PaymentFactory.of(method).pay(ticketId, fee);
        if (st != PaymentStatus.SUCCESS) throw new PaymentFailedException("Payment failed for " + ticketId);
        if (active.remove(ticketId, t)) {          // remove is the single-use guard
            t.close(now);
            t.spot().release();
            floorOf(t.spot()).spotFreed(t.spot());
            platesInside.remove(t.vehicle().plate());
        }
        return fee;
    }

    private ParkingFloor floorOf(ParkingSpot s) {
        return floors.stream().filter(f -> f.number() == s.floor()).findFirst().orElseThrow();
    }
}

// ---------- Gates (thin facades) ----------
class EntryGate {
    private final String id; private final ParkingLot lot;
    EntryGate(String id, ParkingLot lot) { this.id = id; this.lot = lot; }
    Ticket admit(Vehicle v) {
        Ticket t = lot.park(v);
        System.out.printf("[%s] %s -> %s on floor %d (%s)%n",
            id, v.plate(), t.spot().id(), t.spot().floor(), t.id());
        return t;
    }
}
class ExitGate {
    private final String id; private final ParkingLot lot;
    ExitGate(String id, ParkingLot lot) { this.id = id; this.lot = lot; }
    double release(String ticketId, PaymentMethod m) {
        double fee = lot.exit(ticketId, m);
        System.out.printf("[%s] ticket %s paid %.2f%n", id, ticketId, fee);
        return fee;
    }
}

// ---------- Demo ----------
public class Main {
    public static void main(String[] args) throws Exception {
        ParkingLot lot = ParkingLot.forTest(Clock.systemUTC());
        DisplayBoard board = new DisplayBoard();
        for (int f = 0; f < 2; f++) {
            ParkingFloor floor = new ParkingFloor(f);
            floor.subscribe(board);
            for (int i = 0; i < 2; i++) floor.addSpot(new ParkingSpot("F" + f + "-M" + i, SpotType.MOTORCYCLE, f, i));
            for (int i = 0; i < 3; i++) floor.addSpot(new ParkingSpot("F" + f + "-C" + i, SpotType.COMPACT, f, 10 + i));
            floor.addSpot(new ParkingSpot("F" + f + "-L0", SpotType.LARGE, f, 20));
            floor.addSpot(new ParkingSpot("F" + f + "-E0", SpotType.EV, f, 5));
            lot.addFloor(floor);
        }
        lot.setPricing(new EvChargingSurcharge(
            new DynamicPricing(new HourlyPricing(Map.of(
                VehicleType.CAR, 40.0, VehicleType.MOTORCYCLE, 15.0,
                VehicleType.TRUCK, 80.0, VehicleType.EV_CAR, 50.0)), lot::occupancy),
            12.0));

        EntryGate in1 = new EntryGate("IN-1", lot), in2 = new EntryGate("IN-2", lot);
        ExitGate out1 = new ExitGate("OUT-1", lot);

        Ticket a = in1.admit(VehicleFactory.create(VehicleType.CAR, "KA01AB1234"));
        Ticket b = in2.admit(VehicleFactory.create(VehicleType.EV_CAR, "KA02EV9999"));
        b.addKwh(8.5);                                        // charging session reported by charger
        System.out.println(board.render());

        // Concurrency check: 20 threads fight for a 14-spot lot, no spot may be double-booked
        ExecutorService pool = Executors.newFixedThreadPool(8);
        Set<String> assigned = ConcurrentHashMap.newKeySet();
        AtomicInteger full = new AtomicInteger();
        CountDownLatch done = new CountDownLatch(20);
        for (int i = 0; i < 20; i++) {
            final int n = i;
            pool.submit(() -> {
                try {
                    Ticket t = in1.admit(new Car("PLATE" + n));
                    if (!assigned.add(t.spot().id())) throw new AssertionError("double booking!");
                } catch (LotFullException e) { full.incrementAndGet(); }
                finally { done.countDown(); }
            });
        }
        done.await(); pool.shutdown();
        System.out.println("assigned=" + assigned.size() + " rejected=" + full.get());

        out1.release(a.id(), PaymentMethod.CARD);
        try { out1.release(a.id(), PaymentMethod.CARD); }       // double exit with same ticket
        catch (InvalidTicketException e) { System.out.println("blocked: " + e.getMessage()); }
        out1.release(b.id(), PaymentMethod.UPI);
    }
}
```

### Walkthrough of the Key Flows

**Entry**
1. Gate scans plate, `VehicleFactory` builds the `Vehicle`.
2. `ParkingLot.park` asks the allocation strategy for a ranked candidate list.
3. For each candidate it calls `tryOccupy` (CAS FREE to OCCUPIED). First winner wins; the loser moves to the next spot.
4. A `Ticket` is created, stored in `active`, floor counters decremented, listeners (display board) notified.

**Exit**
1. Look up the ticket in `active` (unknown/used ticket gives `InvalidTicketException`).
2. `PricingStrategy.price` computes fee.
3. `PaymentProcessor.pay`. On failure nothing is released.
4. `active.remove(ticketId, t)` is the one-shot guard so a double scan cannot free the spot twice.
5. Spot released, counters incremented, display refreshed.

### Spot Allocation: Nearest vs First-Free

| Strategy | How | Pros | Cons |
|---|---|---|---|
| First-free | Scan floors in order | O(n) simple, deterministic | Fills floor 0 first; upper floors idle, uneven wear |
| Nearest-to-entry | Sort by floor, distance | Shortest walk, best UX | Contention: everyone fights for the same top spots |
| Least-loaded floor | Pick floor with most free | Balances load, less traffic per ramp | Longer walks |
| Random among top-K | Shuffle best K | Reduces CAS collisions | Slightly worse distance |

> [!NOTE]
> At scale, "nearest" creates a hot spot: all gates race for the same spot and most CAS attempts fail. Fix by taking the top-K candidates and picking randomly, or by giving each gate its own zone.

### Concurrency Considerations

- **Two cars, one spot**: solved by `AtomicReference.compareAndSet(FREE, OCCUPIED)`. No global lock; contention is per-spot. The loser retries with the next candidate.
- **Stale ranking**: `rank()` reads `isFree()` without a lock, so the list may be stale. That is fine because the CAS is the source of truth.
- **Counters**: `AtomicInteger` per type per floor; the display board may lag by a few ms (eventual consistency is acceptable for signage).
- **Ticket single use**: `ConcurrentHashMap.remove(key, value)` is atomic, so two exit gates scanning the same ticket cannot both release.
- **Same plate twice**: `platesInside` set blocks a cloned plate from entering twice.
- **Alternative with locks**: `synchronized` per-floor `allocate()` is simpler to reason about but serialises all entries on a floor. With 2-4 gates it is fine; CAS scales better.
- **Visibility**: `parked`, `exitTime` are `volatile`; state is atomic.
- **Reservation expiry**: a scheduled executor flips `RESERVED` back to `FREE` after the grace window using `compareAndSet(RESERVED, FREE)` so a late arrival who already occupied it is not clobbered.

### Extensibility

| New requirement | Change |
|---|---|
| New vehicle type (e.g. Bus) | Add enum value, `Bus` class with `compatibleSpots()`, factory case. No change to lot or gates |
| New pricing (weekend, membership) | New `PricingStrategy` implementation, set via `setPricing` |
| Pricing by tiers (first hour free) | New strategy or a decorator wrapping `HourlyPricing` |
| Reservations | Add `Reservation` entity; `tryReserve()` already exists; strategy filters `RESERVED` spots unless the reservation id matches |
| Handicap spots | New `SpotType.ACCESSIBLE`; allocation prefers it when `vehicle.hasPermit()` |
| Valet | `ValetService` calls `park()` and stores a key tag on the ticket |
| Monthly pass | `PassHolder` check in `exit()` skips payment; model as `PaymentProcessor` that always succeeds |
| Different payment | New `PaymentProcessor` plus a factory case |
| Multiple lots | Remove singleton, introduce `lotId` and a `LotRegistry` |
| Overstay penalty | Wrap pricing with `PenaltyPricing` decorator |
| EV charging session | `ChargerService` calls `ticket.startCharging/addKwh`; billing already adds it |

**Reservation sketch**:
```java
class Reservation {
    String id; ParkingSpot spot; Instant from, until; String plate;
}
// ReservationService.book(): spot.tryReserve(); schedule expiry at from+grace
// Entry gate: if plate has reservation -> spot.occupyReserved(vehicle)
```

### Scaling to Distributed / HLD

A chain of lots (airport, malls) needs a service, not a JVM singleton.

```
 Gate controller (edge) ──► API Gateway ──► Parking Service (stateless, N pods)
        ▲                                         │
        │ sensor events (MQTT)                    ├──► Redis: availability counters, spot locks
 IoT: per-spot ultrasonic/inductive               ├──► PostgreSQL: tickets, payments, spots
 sensors, ANPR cameras ──► Kafka ──► Occupancy    └──► Kafka: events ──► billing, analytics, display
                                      Service
```

**Schema (simplified)**:
```sql
parking_lot(lot_id PK, name, city, timezone)
floor(floor_id PK, lot_id FK, number)
spot(spot_id PK, floor_id FK, type, state, distance, version)   -- version for optimistic lock
ticket(ticket_id PK, lot_id, plate, spot_id FK, entry_ts, exit_ts, status)
payment(payment_id PK, ticket_id FK UNIQUE, amount, method, status, idempotency_key UNIQUE)
reservation(res_id PK, spot_id, plate, from_ts, until_ts, status)
```

**Spot claim in a DB**: `UPDATE spot SET state='OCCUPIED', version=version+1 WHERE spot_id=? AND state='FREE'` and check affected rows = 1. Same CAS idea, now in SQL. Or `SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1` to pick the nearest free spot without waiting on other transactions.

**Availability counters**: keep `free:{lot}:{floor}:{type}` in Redis using `DECR`/`INCR`; reconcile periodically from the DB because counters drift. Display boards subscribe through pub/sub.

**IoT**: per-spot sensors publish occupied/free over MQTT to an edge gateway, then Kafka. Sensors confirm reality (a car that parked in the wrong spot, or a car with no ticket). Use sensors as a reconciliation signal, not as the source of truth for allocation.

**Other HLD points**: partition by `lot_id` (all operations are lot-local, so sharding is trivial); gates keep a local cache so entry keeps working when the cloud is unreachable (offline mode, reconcile later); idempotent payment via ticket id; multi-lot search ("nearest lot with space") reads from a read replica or a geo-indexed store.

### Common Mistakes

| Mistake | Why it hurts | Fix |
|---|---|---|
| `if (type == CAR) ... else if (type == TRUCK)` in the lot | Violates Open/Closed | Polymorphism or strategy |
| Pricing logic inside `Ticket` or `ParkingLot` | Can't change rules without editing core | `PricingStrategy` |
| Check `isFree()` then set occupied | Race: both pass the check | CAS or lock around check-and-set |
| Freeing the spot before payment succeeds | Car leaves unpaid, spot reused | Pay first, release after |
| Using `double` for money | Rounding errors | `BigDecimal` or integer minor units in production |
| Singleton everywhere | Untestable, hidden global state | Inject, or use `forTest` constructor |
| Ignoring the clock | Can't unit-test hourly price | Inject `Clock` |
| One giant `Spot` subclass per vehicle | Explosion of classes | Spot type enum plus compatibility rule |
| Not handling ticket reuse | Spot freed twice | Atomic `remove(key, value)` |
| No failure path for lot full | NPE or hang | Typed exception, display "FULL" |

### Interview Follow-ups

- *"Why is Strategy a good fit for pricing?"* — Pricing rules change often and independently of parking mechanics. Strategy isolates the algorithm behind one method, so adding weekend pricing is one new class and one `setPricing` call; the lot never changes (Open/Closed).
- *"Why is `ParkingLot` a singleton and what is wrong with that?"* — A physical lot is unique, so a singleton models it. The cost is global state and hard testing. I keep `getInstance()` for convenience but expose a constructor taking a `Clock` for tests, and in a multi-lot system I drop the singleton for a registry.
- *"How do you prevent two cars getting the same spot?"* — CAS on the spot state. `compareAndSet(FREE, OCCUPIED)` succeeds for exactly one thread. The loser tries the next ranked spot. In a database that becomes a conditional `UPDATE ... WHERE state='FREE'` with an affected-rows check.
- *"Why CAS and not `synchronized`?"* — Contention is per spot, not per lot. A coarse lock serialises all gates. CAS gives lock-free progress. If simplicity matters more than throughput, a per-floor lock is acceptable.
- *"What if rank() returns a stale list?"* — Harmless. Ranking is advisory; the CAS is the authority. Worst case a few retries.
- *"What if all candidates in the list are taken by the time I try?"* — Re-rank and retry a bounded number of times, then throw `LotFullException`. Display board shows FULL.
- *"How do you add a new vehicle type?"* — Add enum value, a `Vehicle` subclass with `compatibleSpots()`, and a factory case. Nothing else changes. If it needs a new spot size, add a `SpotType`.
- *"Can a motorcycle park in a car spot?"* — By my assumption yes, as fallback after motorcycle spots are full; `compatibleSpots()` encodes the order. Business can change it by editing that list, not the allocator.
- *"How do you do dynamic pricing?"* — `DynamicPricing` decorates a base strategy and multiplies by a surge factor derived from occupancy. It is a Decorator and a Strategy at once. Price should be locked at entry or shown on the board, otherwise customers are surprised.
- *"How do you compute hourly price for 61 minutes?"* — Round up to 2 hours (or use grace period or per-15-minute blocks). It is a product decision; I make the rounding rule a parameter.
- *"Where is the Observer pattern?"* — `ParkingFloor` publishes `onChange`; `DisplayBoard` subscribes. The floor does not know what displays exist, so adding a mobile app feed is just another listener.
- *"Why State pattern for spots? Isn't an enum enough?"* — For four states with trivial behaviour an enum with guarded transitions is enough. If each state had rich behaviour (maintenance workflow, reservation timers) I would promote it to State classes.
- *"How would you model reservations?"* — A `Reservation` entity with a time window; the spot goes `RESERVED` via CAS at booking time or lazily at window start; expiry job releases it after a grace period. Only the matching plate may occupy a reserved spot.
- *"A reserved spot is far in the future. Do you block it now?"* — No. Block only inside `[from - buffer, until]`. Otherwise you waste capacity. Store reservations as intervals and check overlap at allocation.
- *"How do you handle lost tickets?"* — Look up by plate (ANPR) to find entry time; if unknown charge a flat max fee. Keep this as a `LostTicketPolicy` strategy.
- *"What if payment fails or the network drops mid-payment?"* — The spot is not freed. Payment is idempotent on the ticket id, so retry is safe and never double charges. Gate stays closed; attendant override is audited.
- *"How would you handle EV charging?"* — EV spots carry a charger. A session starts on plug-in, a meter reports kWh, and `EvChargingSurcharge` adds it to the fee. Idle fees (car stays after full charge) are another decorator.
- *"What if an EV car can't find an EV spot?"* — It falls back to a normal spot via `compatibleSpots()` order, with no charging.
- *"How do SOLID principles apply here?"* — S: gate, lot, pricing each have one job. O: new pricing/vehicle without edits. L: any `Vehicle` or `PricingStrategy` substitutes. I: small interfaces (`PaymentProcessor` has one method). D: lot depends on strategy interfaces injected, not concretes.
- *"How would you unit test this?"* — Inject a fake `Clock` for pricing; test each strategy in isolation with tickets built by hand; test allocation with a tiny floor; stress test with many threads and assert no spot appears twice. Test the exit rollback path with a failing `PaymentProcessor` stub.
- *"How do you test the race condition deterministically?"* — Use a `CountDownLatch` start gate to release N threads at once, collect assigned spot ids in a concurrent set, assert size equals spots used and no duplicates. Repeat many times; also use jcstress-style tools for rigor.
- *"What if the entry gate crashes after the spot is claimed but before the ticket prints?"* — The spot stays occupied with no one parked: a leak. Mitigate with a reservation-with-TTL step (claim as `RESERVED`, confirm to `OCCUPIED` on barrier open) and a sensor/timeout reaper.
- *"How do you detect a car parked in the wrong spot?"* — Sensors per spot compare observed state to expected state; mismatch raises an alert. Not a core-LD concern, but mention the IoT feed.
- *"How do you make the allocation fair or avoid hot spots?"* — Randomise among the top-K candidates or shard spots by gate zone, which cuts CAS collisions.
- *"How would you handle multiple parking lots?"* — Add `lotId`, remove the singleton, create a `LotRegistry`. Services are stateless; state sits in the DB. Shard by lot id.
- *"How do you keep display counts accurate in a distributed setup?"* — Redis counters updated in the same flow as the DB write (or via outbox + Kafka), periodic reconcile job against `COUNT(*)` of spots, and board subscribed via pub/sub. Eventual consistency of a few seconds is fine.
- *"What happens if Redis counters drift?"* — They are a cache, not the truth. A nightly (or minutely) reconcile recomputes from the spot table. Allocation never trusts the counter for correctness, only the DB CAS.
- *"What are the DB indexes?"* — `spot(floor_id, type, state)` for "next free of type"; `ticket(plate, status)` for lost-ticket lookups; `payment(ticket_id)` unique; `reservation(spot_id, from_ts, until_ts)` for overlap checks.
- *"How would you implement the pick-nearest query in SQL?"* — `SELECT spot_id FROM spot WHERE lot_id=? AND type IN (...) AND state='FREE' ORDER BY floor, distance LIMIT 1 FOR UPDATE SKIP LOCKED`. SKIP LOCKED means concurrent gates pick different rows instead of blocking.
- *"How do you do offline mode if the cloud is down?"* — Edge controller holds a local copy of its lot's spots and issues tickets locally; syncs on reconnect with idempotent events keyed by ticket id. Conflicts are rare because a lot is owned by one edge cluster.
- *"How do you prevent fraud, such as ticket sharing or plate swapping?"* — Tie ticket to plate via ANPR; verify at exit; one-time-use guard; audit trail; alert on mismatches.
- *"How would you add a monthly pass?"* — `PassService.isValid(plate)` at entry marks the ticket `PASS`; at exit a zero-cost pricing strategy or a `PassPaymentProcessor` is used. No change to the lot core.
- *"How do you handle time zones and DST?"* — Store instants in UTC; convert using the lot's timezone only for display and for time-of-day pricing rules.
- *"How would you apply different prices per floor or spot type?"* — Pricing takes a `Ticket`, which links the spot, so a strategy can look at `spot.type()` or `spot.floor()`. Or compose `PricingStrategy` per spot type with a map.
- *"Which design pattern would you use for the gate workflow with several steps (scan, allocate, print, open)?"* — Chain of Responsibility or a simple Template Method; Command if steps need undo and audit.
- *"How would you do undo (e.g. wrong ticket scanned)?"* — Command pattern: each `ExitCommand` records what it changed (spot, payment) and can issue a compensating action, such as refund and re-occupy, through a saga in the distributed version.
- *"How would you expose this as an API?"* — `POST /lots/{id}/entries` returns the ticket; `POST /tickets/{id}/exit` with payment method; `GET /lots/{id}/availability`. Idempotency-Key header on exit/payment.
- *"How would you scale reads for availability?"* — Cache per-lot availability in Redis with short TTL; push updates to clients via WebSocket or SSE; CDN for the static lot list.
- *"What metrics would you track?"* — Occupancy by hour, average dwell time, entry latency p99, allocation retries per entry, payment failure rate, revenue per spot, lost-ticket rate, sensor-vs-system mismatch.
- *"What would you do differently with more time?"* — Replace `double` with `Money`, add the reservation and valet flows, persist through a repository interface, and add an outbox for events.
