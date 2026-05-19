# Q: Optimistic vs Pessimistic locking in JPA — when to use which?

**Answer:**

Both locks solve **lost updates** (two transactions read the same row, both write back, one overwrites the other). They differ in *when* they detect conflict and *how much* they hold off other readers/writers.

### The Lost-Update Problem

```
T1: SELECT balance FROM account WHERE id=1   --> 100
T2: SELECT balance FROM account WHERE id=1   --> 100
T1: UPDATE account SET balance = 100 - 30 WHERE id=1
T2: UPDATE account SET balance = 100 - 50 WHERE id=1
                                       ^^ overwrites T1's update, balance is 50, not 20
```

Both committed. Neither saw the other. Money disappears.

### Optimistic Locking (Version Column)

Assumes conflicts are rare. Detects them at commit time and fails fast.

```java
@Entity
class Account {
    @Id Long id;
    BigDecimal balance;

    @Version
    long version;
}
```

JPA-generated SQL on update:

```sql
UPDATE account
   SET balance = ?, version = version + 1
 WHERE id = ? AND version = ?
```

If `version` doesn't match → 0 rows affected → JPA throws `OptimisticLockException` (Spring: `ObjectOptimisticLockingFailureException`).

Caller retries:

```java
@Retryable(retryFor = ObjectOptimisticLockingFailureException.class,
           maxAttempts = 3, backoff = @Backoff(50))
public void debit(Long id, BigDecimal amt) {
    var a = repo.findById(id).orElseThrow();
    a.setBalance(a.getBalance().subtract(amt));
    repo.save(a);   // version check happens on flush
}
```

### Pessimistic Locking (Database Row Lock)

Assumes conflicts are common. Locks rows in the DB so others wait.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Account findForUpdate(@Param("id") Long id);
```

Generated SQL:

```sql
SELECT * FROM account WHERE id = ? FOR UPDATE
```

Other transactions doing `SELECT ... FOR UPDATE` on the same row **block** until this transaction commits/rolls back.

JPA lock modes:

| Mode | SQL | Use |
|------|-----|-----|
| `PESSIMISTIC_READ` | `FOR SHARE` (Postgres) / `LOCK IN SHARE MODE` (MySQL) | Read but prevent others' writes |
| `PESSIMISTIC_WRITE` | `FOR UPDATE` | Exclusive — most common |
| `PESSIMISTIC_FORCE_INCREMENT` | `FOR UPDATE` + bump `@Version` | Force a version change even on read |

Timeout to avoid hanging forever:

```java
@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000")
```

### Choosing Between Them

| Aspect | Optimistic | Pessimistic |
|--------|-----------|-------------|
| Conflict assumption | Rare | Common |
| Cost on happy path | None | Lock overhead, contention |
| Failure mode | Exception → retry | Wait → possible deadlock |
| Long transactions | Fine | Dangerous (holds locks) |
| Cross-system (HTTP + DB) | Best fit (DB lock doesn't span an HTTP request) | Doesn't fit |
| Reporting / analytical queries on same row | Doesn't block | Blocks |

**Use optimistic when**:
- Read-modify-write across HTTP requests (the typical web app).
- Reads >> writes, conflicts are rare.
- Edits go through a UI and you want "someone else changed this — refresh?" semantics.

**Use pessimistic when**:
- Short-lived service-internal transactions with high contention (inventory, seat booking, ledger postings).
- You'd rather queue than retry.
- Logic between read and write is complex enough that retry is wasteful.

### Worked Example: Inventory Decrement

Pessimistic:

```java
@Transactional
public void reserve(Long sku, int qty) {
    Item item = repo.findForUpdate(sku);          // FOR UPDATE
    if (item.getStock() < qty) throw new OutOfStock();
    item.setStock(item.getStock() - qty);
}
```

Optimistic (more concurrent throughput, can spurious-fail under load):

```java
@Retryable(retryFor = OptimisticLockException.class, maxAttempts = 5)
@Transactional
public void reserve(Long sku, int qty) {
    Item item = repo.findById(sku).orElseThrow();
    if (item.getStock() < qty) throw new OutOfStock();
    item.setStock(item.getStock() - qty);          // version-checked update
}
```

Under high contention, optimistic burns CPU on retries. Under low contention, pessimistic introduces unnecessary blocking. Measure.

### Pitfalls

| Pitfall | Fix |
|---------|-----|
| `@Version` on a Hibernate-managed dirty entity bypassed via native UPDATE | Always go through the entity for writes, or manually bump version |
| Pessimistic lock held across remote calls | Never hold a DB row lock during an HTTP/gRPC call |
| Deadlocks from inconsistent lock order | Always lock rows in the same order (e.g., by primary key) |
| Retry storms after optimistic failure | Add backoff with jitter; cap retries; surface to user |
| Forgetting that `findForUpdate` outside `@Transactional` is a no-op | The lock needs an open transaction |

### Hybrid: Skip Locked (Worker Queues)

```sql
SELECT * FROM jobs
 WHERE status = 'pending'
 ORDER BY id
 LIMIT 1
 FOR UPDATE SKIP LOCKED
```

Multiple workers grab different rows without blocking each other. Available via:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHint(name = "jakarta.persistence.lock.timeout", value = "-2")  // SKIP_LOCKED hint depends on dialect
```

Postgres/MySQL support; useful for outbox patterns and job queues.

> [!NOTE]
> Neither lock helps if you write directly with native SQL that bypasses Hibernate. Mixing modes is fine — but be explicit about it in code review.

### Interview Follow-ups

- *"What is the default isolation level?"* — `READ_COMMITTED` for most JDBC drivers. `REPEATABLE_READ` for MySQL. Neither prevents lost updates without `@Version` or `FOR UPDATE`.
- *"Does `SERIALIZABLE` make this unnecessary?"* — Theoretically yes (in Postgres, via SSI), but with retry storms on conflict. Most teams stay at READ_COMMITTED and use explicit locking.
- *"`@Lock` vs `@Transactional(isolation=...)`?"* — Different layers. Isolation is the policy; lock is the mechanism on a specific query.
