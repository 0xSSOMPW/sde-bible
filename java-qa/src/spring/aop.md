# Q: What is AOP in Spring? How does it work, and why is the proxy detail important?

**Answer:**

AOP = Aspect-Oriented Programming. Cross-cutting concerns (logging, transactions, security, caching, metrics) extracted from business code into **aspects** that wrap target methods.

Spring's AOP is built on **proxies**, not bytecode weaving (unlike AspectJ).

### Core Vocabulary
| Term | Meaning |
|---|---|
| **Aspect** | A class encapsulating a concern (`@Aspect`) |
| **Join point** | A point where advice can run (Spring AOP: only **method calls**) |
| **Advice** | Code that runs at a join point (`@Before`, `@After`, `@Around`, ...) |
| **Pointcut** | Expression matching join points |
| **Weaving** | Linking aspects to target — Spring does it via runtime proxies |

### Example
```java
@Aspect
@Component
public class TimingAspect {

    @Around("execution(public * com.acme.service..*(..))")
    public Object time(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed();
        } finally {
            long us = (System.nanoTime() - start) / 1000;
            log.info("{} took {}us", pjp.getSignature().toShortString(), us);
        }
    }
}
```

### Advice Types
```java
@Before("execution(* OrderService.create(..))")
void log(JoinPoint jp) { log.info("calling {}", jp.getSignature()); }

@AfterReturning(pointcut = "execution(* OrderService.create(..))", returning = "result")
void onReturn(Order result) { log.info("returned {}", result); }

@AfterThrowing(pointcut = "...", throwing = "ex")
void onThrow(Exception ex) { log.error("failed", ex); }

@After("...")  // finally
void always() { }

@Around("...")
Object around(ProceedingJoinPoint pjp) throws Throwable { ... }
```

### Pointcut Designators (Spring AOP Subset)
```java
execution(public * com.acme..*Service.*(..))   // method execution
within(com.acme.service..*)                     // any method in package
@annotation(Loggable)                           // methods annotated @Loggable
@within(org.springframework.stereotype.Service) // methods in @Service classes
@target(...)  args(...)  this(...)  target(...) bean(orderService)
```

### Reusable Pointcut
```java
@Aspect @Component
public class Pointcuts {
    @Pointcut("execution(* com.acme.service..*(..))")
    void service() {}
}

@Around("com.acme.aspects.Pointcuts.service()")
public Object x(ProceedingJoinPoint p) { ... }
```

### Custom Annotation Pattern (Common)
```java
@Target(METHOD) @Retention(RUNTIME)
public @interface RateLimit { int perSecond(); }

@Aspect @Component
public class RateLimitAspect {
    @Around("@annotation(rateLimit)")
    public Object check(ProceedingJoinPoint pjp, RateLimit rateLimit) throws Throwable {
        if (!limiter.tryAcquire(rateLimit.perSecond())) {
            throw new TooManyRequestsException();
        }
        return pjp.proceed();
    }
}

@Service
public class ApiService {
    @RateLimit(perSecond = 10)
    public void call() { ... }
}
```

### How Proxies Work
- Bean has interface → **JDK dynamic proxy** (interface-based).
- No interface → **CGLIB subclass proxy** (cglib-style bytecode subclass).
- Spring 5+ default for unsuited classes: CGLIB. Force with `@EnableAspectJAutoProxy(proxyTargetClass = true)`.

The proxy intercepts external method calls, runs advice, delegates to the target.

### The Self-Invocation Trap
```java
@Service
class UserService {
    @Transactional public void outer() { inner(); }   // ❌ self-call
    @Transactional public void inner() { ... }
}
```
`outer` calls `this.inner()`, **not** the proxy. Inner's `@Transactional` (or any aspect annotation) is ignored. Same applies to `@Async`, `@Cacheable`, `@PreAuthorize`, custom aspects.

Fixes:
- Move `inner` to another bean.
- Self-inject:
  ```java
  @Autowired @Lazy UserService self;
  public void outer() { self.inner(); }
  ```
- Expose proxy: `@EnableAspectJAutoProxy(exposeProxy = true)` then `((UserService) AopContext.currentProxy()).inner();`.

### Spring AOP vs AspectJ
| | Spring AOP | AspectJ |
|---|---|---|
| Weaving | Runtime (proxy) | Compile time / load time |
| Join points | Method calls only | Constructors, fields, blocks, more |
| Performance | Slight runtime cost | Near-native |
| Self-call problem | Yes | No |
| Setup | Just annotations | Requires AspectJ compiler / agent |

For **field access** or **constructor** interception → AspectJ load-time weaving.

### Order of Aspects
```java
@Aspect @Component @Order(1) class FirstAspect {}
@Aspect @Component @Order(2) class SecondAspect {}
```
Lower `@Order` = wraps outermost = runs first on entry, last on exit.

### Real Examples in Spring Itself
- `@Transactional` — `TransactionAspectSupport`.
- `@Async` — `AnnotationAsyncExecutionInterceptor`.
- `@Cacheable` — `CacheInterceptor`.
- `@PreAuthorize` — `MethodSecurityInterceptor`.

All proxy-based. All affected by self-invocation. Same rules.

### Common Pitfalls
- Self-invocation (above).
- Aspect on `private` / `static` / `final` method → won't work (CGLIB can't subclass).
- Aspect on a non-Spring-managed object (`new` instead of injected) → no proxy → no advice.
- Two aspects with same priority → undefined order.
- Forgetting `@EnableAspectJAutoProxy` (Boot enables it automatically).
