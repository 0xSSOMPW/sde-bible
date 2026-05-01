# Q: How does Spring's `@Cacheable` work? Caveats around proxies, keys, and invalidation.

**Answer:**

Spring Cache abstraction = annotations (`@Cacheable`, `@CacheEvict`, `@CachePut`) backed by a pluggable provider (Caffeine, Redis, Ehcache, Hazelcast).

### Enable
```java
@EnableCaching
@Configuration class CacheConfig { }
```

Default provider: `ConcurrentMapCacheManager` (heap, no eviction). Real apps use Caffeine or Redis.

### Annotations
| Annotation | Effect |
|---|---|
| `@Cacheable` | Lookup cache; on miss, run method, store result |
| `@CachePut` | Always run method, store result (refresh) |
| `@CacheEvict` | Remove entry (or all) |
| `@Caching` | Compose multiple |

### Examples
```java
@Cacheable(value = "products", key = "#id")
public Product findById(Long id) { return repo.findById(id).orElseThrow(); }

@CachePut(value = "products", key = "#p.id")
public Product update(Product p) { return repo.save(p); }

@CacheEvict(value = "products", key = "#id")
public void delete(Long id) { repo.deleteById(id); }

@CacheEvict(value = "products", allEntries = true)
public void clearAll() { }
```

### Key Generation
- Default: `SimpleKeyGenerator` — combines all params.
- SpEL via `key`:
  ```java
  @Cacheable(value="users", key="#email.toLowerCase()")
  @Cacheable(value="orders", key="#root.methodName + '-' + #status + '-' + #page")
  ```
- Custom: implement `KeyGenerator`.

### `condition` and `unless`
```java
@Cacheable(value="products", key="#id", condition="#id > 0", unless="#result == null")
public Product find(long id) { ... }
```
- `condition` evaluated **before** call → skip caching when false.
- `unless` evaluated **after** call → skip storing when true.

### Caffeine Setup
```xml
<dependency>
  <groupId>com.github.ben-manes.caffeine</groupId>
  <artifactId>caffeine</artifactId>
</dependency>
```

```yaml
spring:
  cache:
    type: caffeine
    cache-names: products, users
    caffeine:
      spec: maximumSize=10000,expireAfterWrite=10m,recordStats
```

Programmatic per-cache config:
```java
@Bean
CacheManager cacheManager() {
    CaffeineCacheManager m = new CaffeineCacheManager("products", "users");
    m.setCaffeine(Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofMinutes(10))
        .recordStats());
    return m;
}
```

### Redis Setup
```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 600000   # ms
      cache-null-values: false
```

```java
@Bean
RedisCacheManager cacheManager(RedisConnectionFactory cf) {
    return RedisCacheManager.builder(cf)
        .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeValuesWith(SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer())))
        .withCacheConfiguration("products", RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofHours(1)))
        .build();
}
```

### Self-Invocation Trap (BIG One)
Spring caching is proxy-based. **Internal calls bypass the proxy.**

```java
@Service
class ProductService {
    @Cacheable("products")
    public Product find(Long id) { ... }

    public Product findAndCount(Long id) {
        return find(id);   // ❌ same-class call → no proxy → no cache!
    }
}
```

Fixes:
- Move method to a different bean.
- Self-inject:
  ```java
  @Lazy @Autowired ProductService self;
  public Product findAndCount(Long id) { return self.find(id); }
  ```
- Use `AopContext.currentProxy()` (requires `@EnableCaching(exposeProxy = true)`).

### Other Pitfalls

**1. Caching `null`**
Default behavior: `null` cached. Often you don't want that:
```java
@Cacheable(value="users", key="#id", unless="#result == null")
```
Or for Redis, set `cache-null-values: false`.

**2. Mutable returned objects**
Caller mutates → next cache hit returns mutated. Use immutable types or defensive copies.

**3. Cache stampede / thundering herd**
Many concurrent misses → all hit DB. Caffeine handles this by default (`AsyncCache` or sync loading). Redis caches don't — implement single-flight or use Caffeine in front of Redis.

**4. Stale data**
TTL too long → stale; too short → cache useless. Combine TTL with explicit `@CacheEvict` on writes.

**5. Multi-arg key surprises**
```java
@Cacheable("orders")
List<Order> list(int page, int size, Sort sort) { ... }
// SimpleKey(page, size, sort) — sort.toString() may not be deterministic across instances
```
Define an explicit `key` SpEL.

### `@CacheEvict(beforeInvocation = true)`
Default: evict **after** method returns. If method throws, cache remains. Set `beforeInvocation = true` to evict regardless.

### `@Caching` (Compose)
```java
@Caching(evict = {
    @CacheEvict(value="orders", key="#id"),
    @CacheEvict(value="orderSummaries", allEntries=true)
})
public void delete(Long id) { ... }
```

### When NOT to Cache
- Highly write-heavy data — cache invalidation cost dwarfs read savings.
- Per-user data with low repeat rate.
- Anything sensitive without thinking through eviction on auth/role changes.
