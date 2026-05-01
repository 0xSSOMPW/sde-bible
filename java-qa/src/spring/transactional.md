# Q: How does @Transactional work in Spring?

**Answer:**

`@Transactional` is Spring's declarative transaction management annotation. It wraps a method in a database transaction — if the method succeeds, the transaction commits; if it throws an exception, the transaction rolls back.

### How It Works Under the Hood
Spring creates a **proxy** around the annotated bean. The proxy intercepts method calls, begins a transaction before the method, and commits/rolls back after.

```
Client → [Proxy: begin TX] → [Actual Method] → [Proxy: commit TX] → Return
                                    │
                          throws exception?
                                    │
                        [Proxy: rollback TX] → Propagate exception
```

### Basic Usage
```java
@Service
public class OrderService {

    @Transactional
    public void createOrder(Order order) {
        orderRepository.save(order);            // DB write 1
        paymentService.processPayment(order);   // DB write 2
        inventoryService.deductStock(order);     // DB write 3
        // If ANYTHING throws → ALL 3 writes are rolled back
    }
}
```

### Rollback Rules

```java
// Default: rolls back on unchecked exceptions (RuntimeException) ONLY
@Transactional
public void process() { throw new RuntimeException(); } // ✅ Rolls back

@Transactional
public void process() throws IOException { throw new IOException(); } // ❌ Does NOT rollback!

// Explicit: roll back on checked exceptions too
@Transactional(rollbackFor = Exception.class)
public void process() throws IOException { throw new IOException(); } // ✅ Rolls back
```

### Propagation Levels

| Propagation | Behavior |
|---|---|
| `REQUIRED` (default) | Join existing TX, or create a new one if none exists |
| `REQUIRES_NEW` | Always create a new TX (suspend current if exists) |
| `MANDATORY` | Must run inside an existing TX (throws if none) |
| `SUPPORTS` | Run in TX if one exists, otherwise run without |
| `NOT_SUPPORTED` | Always run without TX (suspend current if exists) |
| `NEVER` | Must NOT run in a TX (throws if one exists) |

### The Self-Invocation Trap (Most Common Bug!)

```java
@Service
public class OrderService {

    public void processOrder(Order order) {
        createOrder(order); // ❌ Calling @Transactional method from same class!
    }

    @Transactional
    public void createOrder(Order order) {
        // This is NOT transactional when called from processOrder()!
        // The proxy is bypassed because it's an internal method call.
    }
}
```

**Why?** Spring's proxy only intercepts calls that come **through the proxy** (from outside the class). Internal method calls bypass the proxy entirely.

**Fix options:**
1. Move the transactional method to a separate service.
2. Inject `self` reference: `@Autowired private OrderService self;` then call `self.createOrder()`.
3. Use AspectJ mode (compile-time weaving) instead of proxy.

> [!IMPORTANT]
> The two most critical interview points: (1) `@Transactional` only rolls back **unchecked exceptions** by default — use `rollbackFor = Exception.class` for checked exceptions, and (2) **self-invocation bypasses the proxy** — the annotation is silently ignored on internal calls.
