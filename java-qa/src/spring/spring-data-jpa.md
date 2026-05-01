# Q: How does Spring Data JPA work? Repositories, queries, N+1, fetch types.

**Answer:**

Spring Data JPA = repository abstraction over JPA (Hibernate by default). Define an interface, get a working DAO at runtime via dynamic proxy.

### Repository Hierarchy
```
Repository<T, ID>            (marker)
 └─ CrudRepository           (save, findById, delete, count)
    └─ PagingAndSortingRepository
       └─ JpaRepository      (flush, batch, findAll w/ Sort, getReferenceById)
```

### Define
```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByCustomerId(Long customerId);
    Optional<Order> findByIdAndStatus(Long id, OrderStatus status);
    long countByStatus(OrderStatus status);
    boolean existsByEmail(String email);
}
```

No implementation. Spring generates one.

### Query Derivation (From Method Names)
```java
findBy / readBy / queryBy / countBy / existsBy ...
findByXAndY findByXOrY findByXBetween findByXIn findByXLike findByXNotNull
findByXOrderByYDesc findFirst10ByXOrderByCreatedAtDesc
```

### `@Query` (JPQL/HQL)
```java
@Query("select o from Order o where o.customer.email = :email and o.status = :status")
List<Order> activeFor(@Param("email") String email, @Param("status") OrderStatus status);

@Query(value = "select * from orders where total > ?1", nativeQuery = true)
List<Order> highValue(BigDecimal threshold);
```

### Modifying
```java
@Modifying
@Transactional
@Query("update Order o set o.status = :s where o.id = :id")
int updateStatus(@Param("id") Long id, @Param("s") OrderStatus s);
```

`@Modifying` required for UPDATE/DELETE/INSERT JPQL.

### Pagination & Sort
```java
Page<Order> page = repo.findByStatus(OrderStatus.PAID,
    PageRequest.of(0, 20, Sort.by("createdAt").descending()));

page.getContent();    // current page
page.getTotalPages(); // → triggers a count query
```

Use `Slice<T>` instead of `Page<T>` to skip the count query — cheaper for infinite scroll.

### N+1 Problem (The Big One)
```java
@Entity
class Order {
    @ManyToOne(fetch = FetchType.LAZY) Customer customer;
}

orders.forEach(o -> System.out.println(o.getCustomer().getName()));
// 1 query for orders + N queries (one per order's customer) → N+1
```

**Fixes**:

**1. JOIN FETCH**
```java
@Query("select o from Order o join fetch o.customer where o.status = :s")
List<Order> findWithCustomer(@Param("s") OrderStatus s);
```

**2. `@EntityGraph`**
```java
@EntityGraph(attributePaths = {"customer", "items"})
List<Order> findByStatus(OrderStatus s);
```

**3. Batch fetching** (Hibernate)
```java
@BatchSize(size = 50)
@OneToMany ... List<Item> items;
```

**4. DTO projection** — best when you only need a subset
```java
public interface OrderSummary {
    Long getId();
    String getCustomerName();
    BigDecimal getTotal();
}

@Query("select o.id as id, o.customer.name as customerName, o.total as total from Order o")
List<OrderSummary> summaries();
```

### FetchType Default Recap
| Relation | Default |
|---|---|
| `@OneToOne` | EAGER |
| `@ManyToOne` | EAGER |
| `@OneToMany` | LAZY |
| `@ManyToMany` | LAZY |

> [!IMPORTANT]
> Make all `@*ToOne` LAZY (`fetch = FetchType.LAZY`). Eager loads cascade — one entity ends up loading half the schema.

### Lazy Init Outside Transaction
```java
Order o = repo.findById(1L).get();   // tx ends here
o.getItems().size();                 // 💥 LazyInitializationException
```
Fix: keep transaction open (`@Transactional`), use `JOIN FETCH`, or `@EntityGraph`.

### `getReferenceById` vs `findById`
```java
Order o = repo.findById(1L).orElseThrow();     // SELECT now
Order ref = repo.getReferenceById(1L);          // proxy, no SELECT until access
```
Useful when assigning `@ManyToOne` relations without loading the parent:
```java
order.setCustomer(customerRepo.getReferenceById(customerId));
```

### Custom Repository (Beyond Generated Methods)
```java
public interface OrderRepositoryCustom {
    List<Order> search(OrderSearchCriteria c);
}

public class OrderRepositoryImpl implements OrderRepositoryCustom {
    @PersistenceContext EntityManager em;

    public List<Order> search(OrderSearchCriteria c) { /* CriteriaBuilder */ }
}

public interface OrderRepository extends JpaRepository<Order, Long>, OrderRepositoryCustom { }
```

### Specifications (Dynamic Queries)
```java
public interface OrderRepository extends JpaRepository<Order, Long>, JpaSpecificationExecutor<Order> {}

Specification<Order> spec = Specification
    .where(OrderSpecs.statusEq(PAID))
    .and(OrderSpecs.totalGt(100));

repo.findAll(spec, PageRequest.of(0, 20));
```

### Auditing
```java
@EnableJpaAuditing
public class JpaConfig { }

@Entity
@EntityListeners(AuditingEntityListener.class)
class Order {
    @CreatedDate Instant createdAt;
    @LastModifiedDate Instant updatedAt;
    @CreatedBy String createdBy;
}
```

### Common Pitfalls
- **Open Session In View** (`spring.jpa.open-in-view`) — defaults to `true`. Hides N+1 by keeping session alive across the view layer. **Disable it** in production APIs.
- **Bidirectional `toString()`** → infinite loop. Exclude collections.
- **`save()` returns the managed entity** — assign back: `order = repo.save(order);`.
- **`@Transactional` on private/self-call** — proxy bypassed, no transaction. See `@Transactional` deep-dive.
- **Cascade `ALL` on `@ManyToMany`** — deleting one side wipes the other. Avoid.
