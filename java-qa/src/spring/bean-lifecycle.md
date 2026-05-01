# Q: What are Bean Scopes and the Bean Lifecycle in Spring?

**Answer:**

### Bean Scopes
A bean's scope defines **how many instances** Spring creates and **how long they live**.

| Scope | Instances | Lifecycle | Use Case |
|---|---|---|---|
| `singleton` (default) | 1 per ApplicationContext | App startup → shutdown | Stateless services, repositories |
| `prototype` | New instance per injection/request | Created on demand, NOT destroyed by Spring | Stateful objects, builders |
| `request` | 1 per HTTP request | Request start → end | Request-scoped data |
| `session` | 1 per HTTP session | Session start → invalidation | User session data |
| `application` | 1 per ServletContext | App startup → shutdown | Global web-app state |

```java
@Component
@Scope("prototype")
public class ShoppingCart { /* new instance per injection */ }
```

### Bean Lifecycle

```
                  Bean Lifecycle
                  
1. Instantiation    → Constructor called
2. Populate Props   → Dependencies injected (@Autowired)
3. BeanNameAware    → setBeanName() if interface implemented
4. BeanFactoryAware → setBeanFactory()
5. Pre-Init         → @PostConstruct / BeanPostProcessor.postProcessBeforeInitialization()
6. Init             → InitializingBean.afterPropertiesSet() / custom init-method
7. Post-Init        → BeanPostProcessor.postProcessAfterInitialization()
8. ═══ Bean is READY to use ═══
9. Pre-Destroy      → @PreDestroy
10. Destroy         → DisposableBean.destroy() / custom destroy-method
```

### Practical Example

```java
@Component
public class DatabaseConnectionPool {
    
    private HikariDataSource dataSource;
    
    @Autowired
    public DatabaseConnectionPool(DataSourceProperties props) {
        // Step 1-2: Constructor + injection
    }
    
    @PostConstruct  // Step 5: Called after all dependencies are injected
    public void init() {
        this.dataSource = createPool();
        log.info("Connection pool initialized with {} connections", poolSize);
    }
    
    @PreDestroy  // Step 9: Called before bean is destroyed (app shutdown)
    public void cleanup() {
        dataSource.close();
        log.info("Connection pool closed gracefully");
    }
}
```

### Singleton Gotcha with Prototype

```java
@Component // Singleton by default
public class OrderService {
    @Autowired
    private ShoppingCart cart; // Prototype-scoped
    // ❌ PROBLEM: Same cart instance is used for ALL requests!
    // The prototype bean is injected ONCE into the singleton.
}

// ✅ Fix: Use ObjectProvider or @Lookup
@Component
public class OrderService {
    @Autowired
    private ObjectProvider<ShoppingCart> cartProvider;
    
    public void process() {
        ShoppingCart cart = cartProvider.getObject(); // New instance each time
    }
}
```

> [!CAUTION]
> Injecting a **prototype-scoped bean into a singleton** is a common mistake. The prototype is created once during singleton initialization and reused forever. Use `ObjectProvider`, `@Lookup`, or `ObjectFactory` to get a new prototype instance each time.
