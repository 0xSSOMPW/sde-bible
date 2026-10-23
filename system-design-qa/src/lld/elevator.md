# Q: Design an Elevator System (low-level design).

**Answer:**

The classic "many moving parts" OOD question. It combines the **State pattern** (each car is a state machine), the **Strategy pattern** (which car answers a call, and which floor a car visits next), and **concurrency** (one thread per car plus a controller). Interviewers use it to see whether you separate *dispatching* (system-wide) from *scheduling* (per-car), and whether you handle safety cases like overload, fire mode, and maintenance.

### Clarifying Questions to Ask

| Question | Why it matters | Assumption if not answered |
|---|---|---|
| How many elevators and floors? | Dispatch complexity | 1 to N cars (default 3), 20 floors, sizes configurable |
| Hall call buttons: up/down, or a destination keypad? | Request model | Up/down buttons; destination dispatch as an extension |
| Do cars serve all floors? | Zoning | All floors by default; zones for high-rise |
| Capacity / overload sensor? | Safety rules | Yes, weight limit per car |
| Any priority: VIP, fire, maintenance? | Modes and priority | Fire mode and maintenance mode; VIP optional |
| Optimise for wait time, travel time, or energy? | Algorithm selection | Minimise average wait time |
| Door open/close timing, hold-door button? | Door sub-state | Fixed dwell, hold button optional |
| Concurrency: simultaneous presses? | Thread safety | Yes, many callers, one thread per car |
| Do we need to simulate time? | Testability | Tick-based, injectable timings |
| Scope: software only or physical sensors too? | Hardware interface | Software with sensor interfaces |

> [!NOTE]
> Say out loud: "I'll separate three concerns: the **controller** picks which car, the **car** decides in what order to visit its stops, and the **state machine** decides what a car does right now." That sentence structures the whole interview.

### Requirements

**Functional**:
- Multiple elevators serve a building with F floors.
- Two request types: **hall call** (floor + direction, from outside) and **cabin call** (target floor, from inside a specific car).
- A dispatcher assigns each hall call to the best car.
- Each car services its stops efficiently (do not zigzag).
- Doors open on arrival, stay open a fixed time, close; hold the door while overloaded.
- Emergency (fire) mode: recall all cars to the ground floor, ignore calls, doors open.
- Maintenance mode: car is removed from dispatch.
- Metrics: wait time, trips, utilisation.

**Non-functional**:
- Calls can arrive at any time from many threads; no lost or duplicated requests.
- Cars run independently (a stuck car must not block others).
- Pluggable algorithms (no rewrite to change from SCAN to destination dispatch).
- Deterministic behaviour for tests (injectable tick and door time).
- Safety first: emergency and overload override normal logic.

### Core Entities / Class Diagram

```
 ┌─────────────────────────────────────────────────────┐
 │              ElevatorController (facade)            │
 │  cars: List<Elevator>                               │
 │  selector: ElevatorSelectionStrategy                │
 │  hallCall(floor,dir) cabinCall(carId,floor)         │
 │  fireMode(on)                                       │
 └───────┬─────────────────────────────┬───────────────┘
         │ owns                         │ uses
         ▼                              ▼
 ┌─────────────────────┐      ┌────────────────────────────┐
 │ Elevator (Runnable) │      │ ElevatorSelectionStrategy  │ «interface»
 │ id, floor, direction│      └─────────┬──────────────────┘
 │ stops (LinkedHashSet)│         NearestCar   RoundRobin   Zoned
 │ loadKg, door        │
 │ state: ElevatorState│      ┌────────────────────────────┐
 │ stopStrategy ───────┼────► │ NextStopStrategy           │ «interface»
 └───────┬─────────────┘      └─────────┬──────────────────┘
         │ delegates to                FCFS    SSTF    LOOK(SCAN)
         ▼
 ┌─────────────────┐
 │ ElevatorState   │ «interface»  step(Elevator)
 └───────┬─────────┘
   Idle  Moving  DoorOpen  Emergency  Maintenance

 Request «abstract» ◄── HallCall(floor, Direction)
                    ◄── CabinCall(carId, floor)
 Direction {UP, DOWN, IDLE}      ElevatorListener «interface» (metrics, display)
```

**State diagram of one car**:

```
              addStop                     target reached
   ┌──────┐ ───────────► ┌────────┐ ─────────────────────► ┌───────────┐
   │ IDLE │              │ MOVING │ ◄───────────────────── │ DOOR_OPEN │
   └──┬───┘ ◄─────────── └────────┘   more stops pending   └─────┬─────┘
      ▲    no more stops    │                                    │ no more stops
      └─────────────────────┼────────────────────────────────────┘
                            │ fire alarm (from ANY state)
                            ▼
                      ┌───────────┐  reset   ┌──────┐
                      │ EMERGENCY │ ───────► │ IDLE │      MAINTENANCE: admin only
                      └───────────┘          └──────┘      (exit to IDLE)
```

### Design Patterns Used and Why

| Pattern | Where | Why |
|---|---|---|
| **State** | `ElevatorState` (Idle, Moving, DoorOpen, Emergency, Maintenance) | What "step" means depends on the state; emergency can pre-empt cleanly; removes flag soup |
| **Strategy** | `ElevatorSelectionStrategy` (which car), `NextStopStrategy` (which stop next) | Algorithms are the point of the question and must be swappable |
| **Observer** | `ElevatorListener` | Floor indicators, metrics, and logging without coupling to the car |
| **Facade** | `ElevatorController` | Single entry point for hall/cabin calls and system modes |
| **Command** (optional) | `Request` objects | Queue, log, replay, or prioritise requests uniformly |
| **Singleton** | Not used | One controller per building is created by composition root; avoids global state |
| **Producer-Consumer** | Calls (producers) and car thread (consumer) | Decouples button presses from motion |

> [!NOTE]
> Two separate Strategy interfaces is the key insight. People who conflate "which elevator" and "which floor next" end up with an unreadable god-algorithm.

### Code

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.util.concurrent.locks.*;
import java.util.function.IntPredicate;

// ---------- Basic types ----------
enum Direction { UP, DOWN, IDLE }

abstract class Request {
    final long createdAtNanos = System.nanoTime();
}
class HallCall extends Request {
    final int floor; final Direction direction;
    HallCall(int floor, Direction d) { this.floor = floor; this.direction = d; }
}
class CabinCall extends Request {
    final int carId, floor;
    CabinCall(int carId, int floor) { this.carId = carId; this.floor = floor; }
}

interface ElevatorListener { void on(int carId, String event, int floor); }

// ---------- Strategy: which floor next (per car) ----------
interface NextStopStrategy {
    /** pending is in arrival order. Return the next target floor, or empty if nothing to do. */
    Optional<Integer> next(int current, Direction dir, List<Integer> pending);
}

/** First come, first served: simple and fair, but zigzags. */
class FcfsStrategy implements NextStopStrategy {
    public Optional<Integer> next(int cur, Direction dir, List<Integer> p) {
        return p.isEmpty() ? Optional.empty() : Optional.of(p.get(0));
    }
}

/** Shortest seek time first: nearest stop. Good throughput, can starve far floors. */
class SstfStrategy implements NextStopStrategy {
    public Optional<Integer> next(int cur, Direction dir, List<Integer> p) {
        return p.stream().min(Comparator.comparingInt(f -> Math.abs(f - cur)));
    }
}

/** LOOK (SCAN variant): keep going in the current direction while stops remain ahead, then reverse. */
class LookStrategy implements NextStopStrategy {
    public Optional<Integer> next(int cur, Direction dir, List<Integer> p) {
        if (p.isEmpty()) return Optional.empty();
        if (dir == Direction.IDLE) return new SstfStrategy().next(cur, dir, p);
        Optional<Integer> ahead = dir == Direction.UP
            ? p.stream().filter(f -> f >= cur).min(Integer::compare)
            : p.stream().filter(f -> f <= cur).max(Integer::compare);
        if (ahead.isPresent()) return ahead;
        // nothing ahead: reverse; the nearest stop in the new direction
        return dir == Direction.UP ? p.stream().max(Integer::compare) : p.stream().min(Integer::compare);
    }
}

// ---------- State ----------
interface ElevatorState {
    void step(Elevator e) throws InterruptedException;     // one unit of work, runs on the car's thread
    String name();
}

final class States {
    static final ElevatorState IDLE = new IdleState(), MOVING = new MovingState(),
        DOOR_OPEN = new DoorOpenState(), EMERGENCY = new EmergencyState(), MAINTENANCE = new MaintenanceState();
}

class IdleState implements ElevatorState {
    public void step(Elevator e) throws InterruptedException {
        e.awaitWork();                                       // blocks without busy-waiting
        if (e.hasStopAt(e.floor())) e.transition(this, States.DOOR_OPEN);
        else if (e.hasStops())      e.transition(this, States.MOVING);
    }
    public String name() { return "IDLE"; }
}

class MovingState implements ElevatorState {
    public void step(Elevator e) throws InterruptedException {
        Optional<Integer> target = e.nextTarget();
        if (target.isEmpty()) { e.transition(this, States.IDLE); return; }
        if (target.get() == e.floor()) { e.transition(this, States.DOOR_OPEN); return; }
        e.sleepTick();                                       // travel time for one floor
        e.stepToward(target.get());
        if (e.hasStopAt(e.floor())) e.transition(this, States.DOOR_OPEN);   // serve stops on the way
    }
    public String name() { return "MOVING"; }
}

class DoorOpenState implements ElevatorState {
    public void step(Elevator e) throws InterruptedException {
        e.openDoor();
        while (e.overloaded() && e.isRunning()) {            // safety: never close on overload
            e.publish("OVERLOAD");
            e.sleepTick();
        }
        e.serveCurrentFloor();                               // remove stop, add boarded passengers' targets
        e.sleepDoor();
        e.closeDoor();
        e.transition(this, e.hasStops() ? States.MOVING : States.IDLE);
    }
    public String name() { return "DOOR_OPEN"; }
}

class EmergencyState implements ElevatorState {              // fire recall: go to ground, open, park
    public void step(Elevator e) throws InterruptedException {
        if (e.floor() != e.minFloor()) { e.sleepTick(); e.stepToward(e.minFloor()); return; }
        if (!e.isDoorOpen()) e.openDoor();
        e.awaitStateChange(this);
    }
    public String name() { return "EMERGENCY"; }
}

class MaintenanceState implements ElevatorState {
    public void step(Elevator e) throws InterruptedException { e.awaitStateChange(this); }
    public String name() { return "MAINTENANCE"; }
}

// ---------- Elevator (context, one thread each) ----------
class Elevator implements Runnable {
    private final int id, minFloor, maxFloor, capacityKg;
    private final long tickMs, doorMs;
    private final IntPredicate serves;                       // zoning: which floors this car serves
    private final NextStopStrategy strategy;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition changed = lock.newCondition();
    private final LinkedHashSet<Integer> stops = new LinkedHashSet<>();             // guarded by lock
    private final Map<Integer, List<Integer>> boarding = new HashMap<>();           // pickup floor -> dropoffs
    private final List<ElevatorListener> listeners = new CopyOnWriteArrayList<>();
    private volatile ElevatorState state = States.IDLE;
    private volatile Direction direction = Direction.IDLE;
    private volatile int floor;
    private volatile int loadKg;
    private volatile boolean doorOpen, running = true;

    Elevator(int id, int minFloor, int maxFloor, int capacityKg, long tickMs, long doorMs,
             IntPredicate serves, NextStopStrategy strategy) {
        this.id = id; this.minFloor = minFloor; this.maxFloor = maxFloor; this.capacityKg = capacityKg;
        this.tickMs = tickMs; this.doorMs = doorMs; this.serves = serves; this.strategy = strategy;
        this.floor = minFloor;
    }

    // ----- accessors -----
    int id() { return id; }  int floor() { return floor; }  int minFloor() { return minFloor; }
    Direction direction() { return direction; }  ElevatorState state() { return state; }
    boolean isRunning() { return running; }  boolean isDoorOpen() { return doorOpen; }
    boolean overloaded() { return loadKg > capacityKg; }
    void setLoadKg(int kg) { loadKg = kg; }
    void subscribe(ElevatorListener l) { listeners.add(l); }
    void publish(String ev) { listeners.forEach(l -> l.on(id, ev, floor)); }
    boolean serves(int f) { return f >= minFloor && f <= maxFloor && serves.test(f); }
    boolean available() { return state != States.EMERGENCY && state != States.MAINTENANCE; }

    int pendingCount() { lock.lock(); try { return stops.size(); } finally { lock.unlock(); } }
    boolean hasStops() { return pendingCount() > 0; }
    boolean hasStopAt(int f) { lock.lock(); try { return stops.contains(f); } finally { lock.unlock(); } }
    Optional<Integer> nextTarget() {
        lock.lock();
        try { return strategy.next(floor, direction, new ArrayList<>(stops)); } finally { lock.unlock(); }
    }
    int estimate(int from, int to) { return Math.abs(floor - from) + Math.abs(from - to) + 2 * pendingCount(); }

    // ----- inputs (called from controller threads) -----
    boolean addStop(int f) {
        if (!available() || !serves(f)) return false;
        lock.lock();
        try { stops.add(f); changed.signalAll(); return true; } finally { lock.unlock(); }   // Set: duplicates collapse
    }
    /** Destination dispatch: dropoff becomes a stop only after the passenger boards at 'from'. */
    boolean addTrip(int from, int to) {
        if (!available() || !serves(from) || !serves(to)) return false;
        lock.lock();
        try {
            boarding.computeIfAbsent(from, k -> new ArrayList<>()).add(to);
            stops.add(from); changed.signalAll(); return true;
        } finally { lock.unlock(); }
    }

    // ----- state machine plumbing -----
    /** Compare-and-set style transition so a late step cannot overwrite EMERGENCY. */
    boolean transition(ElevatorState from, ElevatorState to) {
        lock.lock();
        try {
            if (state != from) return false;
            state = to; changed.signalAll();
            publish("STATE:" + to.name());
            return true;
        } finally { lock.unlock(); }
    }
    void force(ElevatorState to) {                           // emergency, maintenance, reset
        lock.lock();
        try {
            state = to;
            if (to == States.EMERGENCY) { stops.clear(); boarding.clear(); direction = Direction.DOWN; }
            changed.signalAll(); publish("STATE:" + to.name());
        } finally { lock.unlock(); }
    }
    void awaitWork() throws InterruptedException {
        lock.lock();
        try { while (running && state == States.IDLE && stops.isEmpty()) changed.await(); }
        finally { lock.unlock(); }
    }
    void awaitStateChange(ElevatorState s) throws InterruptedException {
        lock.lock();
        try { while (running && state == s) changed.await(); } finally { lock.unlock(); }
    }
    void sleepTick() throws InterruptedException { Thread.sleep(tickMs); }
    void sleepDoor() throws InterruptedException { Thread.sleep(doorMs); }
    void stepToward(int target) {
        direction = target > floor ? Direction.UP : Direction.DOWN;
        floor += direction == Direction.UP ? 1 : -1;
        publish("MOVED");
    }
    void openDoor() { doorOpen = true; publish("DOOR_OPEN"); }
    void closeDoor() { doorOpen = false; publish("DOOR_CLOSE"); }
    void serveCurrentFloor() {
        lock.lock();
        try {
            stops.remove(floor);
            List<Integer> drops = boarding.remove(floor);
            if (drops != null) stops.addAll(drops);
            if (stops.isEmpty()) direction = Direction.IDLE;
        } finally { lock.unlock(); }
        publish("SERVED");
    }
    void stop() {
        running = false;
        lock.lock(); try { changed.signalAll(); } finally { lock.unlock(); }
    }

    @Override public void run() {
        try { while (running) state.step(this); }
        catch (InterruptedException ie) { Thread.currentThread().interrupt(); }
    }
}

// ---------- Strategy: which car answers a hall call ----------
interface ElevatorSelectionStrategy {
    Elevator select(List<Elevator> cars, HallCall call);
}

class NearestCarStrategy implements ElevatorSelectionStrategy {
    public Elevator select(List<Elevator> cars, HallCall c) {
        return cars.stream().filter(e -> e.available() && e.serves(c.floor))
            .min(Comparator.comparingInt(e -> cost(e, c))).orElse(null);
    }
    int cost(Elevator e, HallCall c) {
        int d = Math.abs(e.floor() - c.floor);
        Direction ed = e.direction();
        if (ed == Direction.IDLE) return d;
        boolean onTheWay = (ed == Direction.UP && c.floor >= e.floor()) || (ed == Direction.DOWN && c.floor <= e.floor());
        if (onTheWay && ed == c.direction) return d;         // same direction and ahead: pick up while passing
        return d + 20 + 2 * e.pendingCount();                // must finish current sweep first
    }
}

class RoundRobinStrategy implements ElevatorSelectionStrategy {
    private final AtomicInteger next = new AtomicInteger();
    public Elevator select(List<Elevator> cars, HallCall c) {
        for (int i = 0; i < cars.size(); i++) {
            Elevator e = cars.get(Math.floorMod(next.getAndIncrement(), cars.size()));
            if (e.available() && e.serves(c.floor)) return e;
        }
        return null;
    }
}

// ---------- Metrics (Observer) ----------
class MetricsListener implements ElevatorListener {
    final ConcurrentHashMap<Integer, LongAdder> floorsTravelled = new ConcurrentHashMap<>();
    final LongAdder stopsServed = new LongAdder();
    public void on(int car, String ev, int floor) {
        if (ev.equals("MOVED")) floorsTravelled.computeIfAbsent(car, k -> new LongAdder()).increment();
        if (ev.equals("SERVED")) stopsServed.increment();
    }
}

// ---------- Controller (facade) ----------
class ElevatorController {
    private final List<Elevator> cars;
    private final ExecutorService pool;
    private volatile ElevatorSelectionStrategy selector;

    ElevatorController(List<Elevator> cars, ElevatorSelectionStrategy selector) {
        this.cars = cars; this.selector = selector;
        this.pool = Executors.newFixedThreadPool(cars.size());
        cars.forEach(pool::submit);                          // one thread per car
    }
    void setSelector(ElevatorSelectionStrategy s) { selector = s; }
    void subscribe(ElevatorListener l) { cars.forEach(c -> c.subscribe(l)); }

    boolean hallCall(int floor, Direction dir) {
        Elevator e = selector.select(cars, new HallCall(floor, dir));
        if (e == null) return false;
        System.out.printf("hall call %d %s -> car %d%n", floor, dir, e.id());
        return e.addStop(floor);
    }
    boolean cabinCall(int carId, int floor) { return cars.get(carId).addStop(floor); }

    /** Destination dispatch: passenger enters target at the lobby kiosk. */
    boolean tripRequest(int from, int to) {
        Elevator best = cars.stream().filter(e -> e.available() && e.serves(from) && e.serves(to))
            .min(Comparator.comparingInt(e -> e.estimate(from, to))).orElse(null);
        return best != null && best.addTrip(from, to);
    }
    void fireMode(boolean on) { cars.forEach(c -> c.force(on ? States.EMERGENCY : States.IDLE)); }
    void maintenance(int carId, boolean on) { cars.get(carId).force(on ? States.MAINTENANCE : States.IDLE); }
    void shutdown() { cars.forEach(Elevator::stop); pool.shutdownNow(); }
}

// ---------- Demo ----------
public class Main {
    public static void main(String[] args) throws Exception {
        List<Elevator> cars = new ArrayList<>();
        for (int i = 0; i < 3; i++)
            cars.add(new Elevator(i, 0, 9, 800, 30, 60, f -> true, new LookStrategy()));
        ElevatorController ctl = new ElevatorController(cars, new NearestCarStrategy());
        MetricsListener metrics = new MetricsListener();
        ctl.subscribe(metrics);
        ctl.subscribe((car, ev, fl) -> {
            if (ev.equals("DOOR_OPEN") || ev.equals("OVERLOAD"))
                System.out.printf("  car %d %s at floor %d%n", car, ev, fl);
        });

        ctl.hallCall(5, Direction.UP);
        ctl.hallCall(2, Direction.DOWN);
        ctl.hallCall(7, Direction.DOWN);
        Thread.sleep(120);
        ctl.cabinCall(0, 8);
        ctl.cabinCall(1, 1);

        cars.get(2).setLoadKg(900);                          // overload: door must stay open
        ctl.hallCall(0, Direction.UP);
        Thread.sleep(400);
        cars.get(2).setLoadKg(300);                          // passenger steps out
        Thread.sleep(1200);

        System.out.println("destination dispatch: 3 -> 9");
        ctl.tripRequest(3, 9);
        Thread.sleep(1500);

        System.out.println("FIRE!");
        ctl.fireMode(true);
        Thread.sleep(800);
        cars.forEach(c -> System.out.printf("  car %d floor=%d state=%s%n", c.id(), c.floor(), c.state().name()));
        System.out.println("hall call during fire accepted? " + ctl.hallCall(4, Direction.UP));
        ctl.fireMode(false);

        System.out.println("stops served=" + metrics.stopsServed.sum() + " floors=" + metrics.floorsTravelled);
        ctl.shutdown();
    }
}
```

### Scheduling Algorithms Compared

| Algorithm | How it picks the next stop | Pros | Cons | Use |
|---|---|---|---|---|
| **FCFS** | Oldest request first | Trivial, fair | Zigzag, long travel | Baseline, low traffic |
| **SSTF** | Nearest stop | Short average seek | Starvation of far floors, no direction awareness | Light traffic |
| **SCAN** | Sweep to the end of the shaft, then reverse | Predictable, no starvation | Wasteful trips to the extremes | Disk-style, rarely used for cars |
| **LOOK** | Sweep until the last request in that direction, then reverse | Predictable and efficient, no starvation | Slightly longer wait for the opposite direction | **Default for cars** |
| **Destination dispatch** | Passengers enter the target at the lobby; the system groups riders by destination and assigns a car | Up to 25-30% faster in heavy traffic, fewer stops per rider | Needs kiosks; passengers must follow the assigned car; harder algorithm | Large office towers |

Hall-call assignment (which car): **nearest car** with a cost function (distance plus penalties for wrong direction or busy), **round robin** (simple, ignores state), **zoned** (each car serves floors X to Y). Real controllers minimise *estimated time to destination* across all waiting passengers.

> [!NOTE]
> Say "LOOK" not "SCAN" when asked what a real elevator does. SCAN goes to the physical end; LOOK reverses at the last pending request. An elevator never travels to the top floor for nothing.

### Special Modes

| Mode | Behaviour | Implementation |
|---|---|---|
| **Fire / emergency** | Cancel all stops, descend to the ground floor, open doors, park, ignore calls | `force(EMERGENCY)` from the controller; `available()` false so dispatch skips it |
| **Maintenance** | Out of dispatch, accepts only technician commands | `force(MAINTENANCE)`; existing stops cleared or finished first (policy) |
| **Overload** | Door held open and buzzer until weight under limit | Check in `DoorOpenState`; never `closeDoor()` while overloaded |
| **VIP / priority** | Reserved car, express to the target without intermediate stops | `PriorityRequest`; the controller picks an idle car and sets an `express` flag that makes the stop strategy ignore other stops |
| **Hold door** | Extends dwell on button | `holdDoor(ms)` adds to the dwell in `DoorOpenState` |
| **Off-peak parking** | Idle cars return to lobby or spread across floors | `IdleState` schedules a "park" request after N minutes |
| **Up-peak / down-peak** | Morning: idle cars wait at lobby; evening: bias towards upper floors | A `TrafficMode` strategy plugged into the selector |

### Concurrency Considerations

- **Threads**: one `Elevator` runs per thread in an executor; the controller is called from arbitrary threads (UI, kiosks, sensors).
- **Shared state**: only `stops`/`boarding` are mutated by both sides. A per-car `ReentrantLock` guards them; `floor`, `direction`, `state`, `loadKg` are `volatile` for lock-free reads by the dispatcher.
- **Condition variable** (`changed`) lets an idle car sleep until work arrives. No busy waiting, no lost wake-ups (condition check is in a loop under the lock).
- **State changes under the lock** with `transition(from, to)`: a car thread finishing a "move" cannot overwrite an `EMERGENCY` state set by the controller one microsecond earlier.
- **Dispatcher races**: it reads `floor`/`direction` without a global lock, so its decision may be a few ms stale. That is acceptable; the worst case is a slightly suboptimal assignment, not a correctness issue. The assignment itself (`addStop`) is atomic.
- **No global lock**: a global lock would serialise all cars. Per-car locks keep cars independent.
- **Duplicate presses**: `LinkedHashSet` collapses repeated presses of the same floor.
- **Deadlock avoidance**: the controller never holds two car locks at once; listeners are invoked outside critical sections where possible (copy-on-write list).
- **Shutdown**: set `running=false`, signal the condition, then `shutdownNow()` so waiting threads exit.
- **Alternative**: single-threaded event loop per car (an `ArrayBlockingQueue<Command>`), where the car thread is the only writer. Simpler to reason about, same performance.

### Extensibility

| New requirement | Change |
|---|---|
| New algorithm (e.g. SCAN, energy-optimised) | New `NextStopStrategy` or `ElevatorSelectionStrategy` |
| Destination dispatch | `tripRequest` already shown; add `Trip` object with pickup and dropoff; assignment cost uses estimated time to destination |
| Express/sky lobby cars | `serves(IntPredicate)` zoning per car |
| Disabled-access priority | `Request.priority`; stop strategy orders by priority first |
| Different capacities | `capacityKg` per car (already) |
| Display in lobby and cabin | `ElevatorListener` implementations |
| Access control (key card to floors) | `AccessPolicy` consulted in `cabinCall`/`tripRequest` |
| Predictive parking from history | `IdleState` consults a `ParkingPredictor` strategy |
| Elevator groups (low-rise, high-rise banks) | `ElevatorBank` composite holding cars; controller routes by bank |
| Additional states (door fault) | New `ElevatorState`; transitions from `DoorOpen` only |
| Real hardware | Replace `sleepTick` with sensor callbacks behind a `Motor`/`DoorDriver` interface |

### Scaling to Distributed / HLD

One building is a real-time embedded system, not a web service. The distributed angle is a **fleet and building-management platform**.

```
 Elevator group controller (per building, on-prem, hard real-time, offline-capable)
        │  sensors/actuators: motor drive, door, load cell, floor encoder
        │  MQTT / OPC-UA
        ▼
 Edge gateway ──► Kafka ──► Telemetry (TSDB) ──► Dashboards, predictive maintenance (ML)
        │                       │
        └──── commands ◄────────┘ (mode changes, config)       Mobile app ──► API ──► call elevator
                                                               (Bluetooth/QR) → destination dispatch
```

**Principles**:
- Safety logic (overspeed, door interlock, fire recall) stays in certified hardware/firmware, never in the cloud.
- The cloud handles analytics, configuration, remote diagnostics, mobile apps, and analytics-driven tuning of dispatch parameters.
- Elevator call from a phone: app sends `(userId, from, to)` to the building gateway; the gateway authorises (access control) and calls `tripRequest`.

**Schema** (telemetry and ops):
```sql
building(building_id PK, name, floors, timezone)
elevator(elevator_id PK, building_id FK, bank, capacity_kg, min_floor, max_floor, firmware, status)
trip(trip_id PK, elevator_id, from_floor, to_floor, requested_at, picked_up_at, dropped_at, user_id NULL)
elevator_event(event_id PK, elevator_id, type, floor, ts, payload JSON)     -- DOOR_FAULT, OVERLOAD, ...
fault(fault_id PK, elevator_id, code, severity, opened_at, resolved_at)
maintenance_visit(visit_id PK, elevator_id, technician, ts, notes)
```

**IoT**: encoders and door sensors stream at 1-10 Hz; vibration and motor current feed a predictive maintenance model (replace the cable before it fails). Partition the stream by `elevator_id`; compact old data into hourly aggregates.

### Common Mistakes

| Mistake | Why it hurts | Fix |
|---|---|---|
| One god `Elevator` class with `if (direction == UP) ...` | Unreadable, untestable | State pattern, strategies |
| Merging "which car" and "which floor next" | Cannot change one algorithm without the other | Two strategy interfaces |
| Busy-wait loop `while(true){ if(hasWork)... }` | Burns CPU | `Condition.await()` / blocking queue |
| Single global lock | Cars block each other | Per-car lock, volatile reads |
| Ignoring direction when picking up a hall call | Car going down stops for an up call, passengers ride the wrong way | Direction-aware cost, or per-direction stop sets |
| Not handling duplicate presses | Duplicate stops and extra door cycles | `Set` of stops |
| No emergency override path | Safety failure | `force()` bypasses normal transitions |
| Closing door while overloaded | Safety failure | Check weight before closing |
| Mutating state from the dispatcher thread without the lock | Lost updates | Lock plus `transition(from,to)` |
| Hard-coded sleeps | Slow tests | Injectable tick/door timings or a fake clock |
| Treating the whole design as one `class Elevator` and `class Building` | Misses requests vs scheduling vs state | Separate controller, car, state |

### Interview Follow-ups

- *"Why State pattern for the car?"* — A car's reaction to the same event differs by state: `addStop` while `MOVING` just queues, while `EMERGENCY` rejects. Each state encapsulates its behaviour, transitions are explicit, and adding `MAINTENANCE` or `DOOR_FAULT` means adding a class, not editing a giant switch.
- *"Why two strategies?"* — Dispatching (which car) is a global optimisation across cars; scheduling (which floor next) is a local decision inside one car. They change independently: you can swap round-robin for nearest-car without touching LOOK.
- *"Explain LOOK and how it differs from SCAN."* — Both sweep in one direction and serve stops in order. SCAN goes to the end of the shaft before reversing, LOOK reverses at the last pending request in the current direction. LOOK avoids useless travel and starvation.
- *"Why not SSTF?"* — It has the best average travel but can starve distant floors if nearby requests keep coming. LOOK bounds waiting time to roughly one sweep.
- *"When would FCFS be fine?"* — One car, low traffic, or a tiny building. Easy to explain and implement as a baseline.
- *"How do you handle a hall call going UP when a car is moving DOWN past that floor?"* — The car must not stop (passengers would ride the wrong way). Keep separate up-stops and down-stops (two `TreeSet`s) per car; going down, serve only down-stops and cabin stops; reverse and serve up-stops on the way back. My `nearest car` cost already penalises wrong-direction cars.
- *"How is cost computed for dispatch?"* — Distance if idle or if the car is moving towards the call in the same direction; otherwise distance plus a penalty for finishing the current sweep and the number of pending stops. Production systems estimate time-to-destination including door times and stops.
- *"What if all cars are busy or out of service?"* — Return false or queue the call in a controller-level pending queue and re-dispatch when a car becomes available (listener on `STATE:IDLE`).
- *"What about reassigning calls?"* — Calls can be reassigned if a better car becomes available before the assigned car arrives (dynamic reallocation), or if a car goes into maintenance. Keep unfulfilled hall calls in a controller map keyed by (floor, direction) and re-run the selector.
- *"How do you prevent starvation?"* — LOOK is naturally starvation-free; for SSTF add aging: increase the effective priority of a request as its wait grows. A max-wait guarantee triggers an assignment override.
- *"How do you handle overload?"* — In `DoorOpenState` the car holds the door and publishes `OVERLOAD` until the weight sensor reads below capacity. New cabin calls remain queued. Also bypass hall calls when the load is above ~80 percent (full car skips pickups).
- *"What about the emergency/fire case?"* — The controller calls `force(EMERGENCY)` on every car: clear stops, descend to ground, open doors, ignore calls until a key reset. Cars are excluded from dispatch via `available()`. Firefighter mode (phase 2) allows manual control with a key.
- *"What about power failure?"* — Firmware (not this layer) brings the car to the nearest floor with battery backup and opens the door. Software state is persisted so the controller recovers pending calls after reboot, and cars re-home.
- *"How would you implement VIP or priority requests?"* — Add `priority` to `Request`; the controller selects the least-loaded or idle car and sends an express flag that suppresses intermediate stops; the stop strategy orders priority first. Needs an anti-abuse rule so VIP does not starve everyone.
- *"How do you do destination dispatch?"* — The passenger keys the destination at a lobby kiosk; the controller groups people by direction and target and assigns a car (minimising estimated time). The car adds the pickup as a stop and the dropoff as a stop only after boarding (`addTrip` shows this). Cabin has no floor buttons, so nobody changes the plan midway.
- *"Why is destination dispatch more efficient?"* — Cars carry riders with similar destinations, so there are fewer intermediate stops and fewer door cycles.
- *"How do you do zoning in a 100-floor tower?"* — Split into banks: low zone (1-30), mid, high, with sky lobbies. Each car's `serves` predicate encodes its range; cars skip floors outside it (express shaft to the zone start). The selector filters by `serves`.
- *"How do you test the scheduling strategies?"* — Pure functions: `next(current, direction, pending)` has no side effects, so unit test with tables of inputs. Dispatcher tests use stub elevators with preset floor/direction.
- *"How do you test the state machine and concurrency without real time?"* — Inject tick/door durations (tiny values) or a `Clock`/`Sleeper` interface; use a `CountDownLatch` on events from the listener ("DOOR_OPEN at floor 5") rather than `Thread.sleep`. Test emergency pre-emption by forcing it mid-move and asserting the car ends at ground with doors open.
- *"What if two threads press buttons at once?"* — `addStop` takes the per-car lock; both are added, duplicates collapse. The car thread is woken via the condition.
- *"Can the dispatcher and the car deadlock?"* — The dispatcher takes at most one car lock at a time and never calls into another car while holding it. Car threads never take the controller lock. Therefore no cycles.
- *"Why volatile for floor and direction?"* — The dispatcher reads them often and does not need a consistent snapshot; volatile gives visibility without lock contention. For a fully consistent snapshot use a lock or an immutable snapshot object published atomically.
- *"What about thread per elevator versus event loop?"* — Thread per car is simple and the count is tiny (under 100). A single-threaded event loop (actor per car with a mailbox) removes explicit locking: only the owner mutates state. I would choose actors if the model grows.
- *"How do you simulate time?"* — Replace `Thread.sleep` with a `Clock` interface and a scheduler; a simulated clock lets you run an hour of traffic in a second for dispatch tuning and unit tests.
- *"How would you measure performance?"* — Average and p95 wait time (call to pickup), ride time, stops per trip, energy per trip, utilisation, door cycles, and fraction of full cars that skipped calls. The metrics listener aggregates events and reports per bank.
- *"How do you pick the best algorithm?"* — Simulate with real traffic profiles (up-peak, down-peak, lunch, random) and compare average and tail wait times. Pick per time of day with a `TrafficMode` strategy.
- *"How do you model hall buttons vs cabin buttons?"* — `HallCall(floor, direction)` is dispatched by the controller; `CabinCall(carId, floor)` goes straight to that car. Both are `Request`s so they can be logged and prioritised the same way.
- *"What about access control (keycards)?"* — A policy object checks the user's allowed floors in `cabinCall`/`tripRequest`; unauthorised floors are rejected. Fire mode overrides it.
- *"What if a request is for the floor the car is already on?"* — `hasStopAt(floor())` is true in `IdleState`, so it transitions to `DOOR_OPEN` immediately and reopens the door.
- *"How do you handle an out-of-range floor or invalid request?"* — `serves(floor)` rejects it; `addStop` returns false and the controller reports an error. Validate at the boundary, not deep inside the car.
- *"Which SOLID principles?"* — S: car, controller, strategies, states each have one reason to change. O: new algorithm or state without editing existing classes. L: any strategy or state can replace another. I: small interfaces (`ElevatorListener`, `NextStopStrategy`). D: controller depends on `ElevatorSelectionStrategy` abstraction; elevator depends on `NextStopStrategy`.
- *"Where is Observer used?"* — `ElevatorListener` for displays, metrics, logging; it keeps the car ignorant of who listens.
- *"Could the whole thing be a table-driven FSM?"* — Yes: a transition table `(state, event) -> (action, nextState)` is more declarative and verifiable; State classes read better when each state has substantial behaviour.
- *"How would you add undo or audit?"* — Wrap requests as `Command`s and write them to an event log (who pressed what, when, which car served it). For a replay/debug use case you can re-feed the log into a simulator.
- *"How do you persist state?"* — Pending hall calls and mode flags go to a small local store (journal); on restart, cars re-home, and unserved hall calls are re-dispatched. Cabin calls are lost since passengers would re-press.
- *"How do you expose it as a service?"* — REST/gRPC on the building gateway: `POST /hall-calls`, `POST /cars/{id}/cabin-calls`, `POST /trips`, `GET /cars`. Real-time updates over WebSocket or MQTT. Idempotency key on each request.
- *"How would you scale to hundreds of buildings?"* — Each building has its own controller (the real-time loop never leaves the building). The cloud only collects telemetry, manages configuration, and runs analytics, partitioned by `building_id`.
- *"What would predictive maintenance need?"* — Door cycle counts, motor current, vibration, trips per day, fault codes: stream to a TSDB, train a model for time-to-failure, open a ticket before a breakdown.
- *"What would you do with more time?"* — Direction-aware stop sets, request aging, reassignment of pending hall calls, a simulated clock for traffic simulation, and persistence of pending calls.
