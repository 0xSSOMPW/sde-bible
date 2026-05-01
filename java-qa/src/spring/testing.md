# Q: How do you test Spring Boot apps? Slices, `@SpringBootTest`, `@MockBean`, Testcontainers.

**Answer:**

Spring Boot offers **test slices** (load minimal context for layer under test) and **full-context tests** (load entire app). Pick the smallest scope that exercises what you're testing.

### Test Pyramid in Boot
| Type | Annotation | Scope |
|---|---|---|
| Unit | none (pure JUnit/Mockito) | One class, fastest |
| Slice | `@WebMvcTest`, `@DataJpaTest`, etc. | One layer, mocks rest |
| Integration | `@SpringBootTest` | Full context, slower |
| E2E | `@SpringBootTest(webEnvironment=RANDOM_PORT)` + Testcontainers | Real server + DB |

### Unit Test (No Spring)
```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock OrderRepository repo;
    @Mock PaymentClient pay;
    @InjectMocks OrderService service;

    @Test
    void createsOrder() {
        when(repo.save(any())).thenAnswer(i -> i.getArgument(0));
        Order o = service.create(new CreateOrderRequest(...));
        assertThat(o.id()).isNotNull();
        verify(pay).charge(any());
    }
}
```
Fastest. No Spring overhead. Use whenever possible.

### `@WebMvcTest` (Controller Slice)
Loads only MVC infrastructure (controllers, filters, advice). Other beans must be mocked.

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mvc;
    @MockBean OrderService service;     // service mocked, controller real

    @Test
    void getOrder() throws Exception {
        when(service.find(1L)).thenReturn(new Order(1L, "PAID"));
        mvc.perform(get("/api/orders/1"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.status").value("PAID"));
    }
}
```

### `@DataJpaTest` (Repository Slice)
- Configures in-memory DB (H2) by default.
- Wraps each test in transaction + rollback.
- Loads only JPA components.

```java
@DataJpaTest
class OrderRepositoryTest {
    @Autowired OrderRepository repo;
    @Autowired TestEntityManager em;

    @Test
    void findsByStatus() {
        em.persist(new Order("PAID"));
        em.persist(new Order("REFUNDED"));
        assertThat(repo.findByStatus("PAID")).hasSize(1);
    }
}
```

To use the real DB:
```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
```

### Other Slices
| Slice | Loads |
|---|---|
| `@JsonTest` | Jackson + JSON test utilities |
| `@RestClientTest` | `RestTemplate` / `RestClient` + `MockRestServiceServer` |
| `@WebFluxTest` | WebFlux equivalent of `@WebMvcTest` |
| `@DataMongoTest`, `@DataRedisTest`, `@DataR2dbcTest` | Per-store slices |
| `@WebServiceClientTest` | SOAP client |

### `@SpringBootTest` (Full Context)
```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class OrderApiIT {
    @Autowired TestRestTemplate http;

    @Test
    void createOrder() {
        var resp = http.postForEntity("/api/orders", new CreateOrderRequest(...), Order.class);
        assertThat(resp.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    }
}
```

WebEnvironment options:
- `MOCK` (default) — `MockMvc`-style, no real server.
- `RANDOM_PORT` — real Tomcat on random port.
- `DEFINED_PORT` — uses `server.port`.
- `NONE` — no servlet env.

### `@MockBean` and `@SpyBean`
Replace a bean in the context with a Mockito mock/spy.

```java
@SpringBootTest
class CheckoutTest {
    @MockBean PaymentGateway gateway;     // replaces real bean

    @Test
    void doesntCallProduction() {
        when(gateway.charge(any())).thenReturn(Receipt.ok());
        ...
    }
}
```

> [!IMPORTANT]
> Boot 3.4+ deprecated `@MockBean`/`@SpyBean` in favor of `@MockitoBean`/`@MockitoSpyBean`.

### Testcontainers (Real DB / Kafka / Redis)
```xml
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>postgresql</artifactId>
  <scope>test</scope>
</dependency>
```

```java
@SpringBootTest
@Testcontainers
class OrderApiIT {
    @Container
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
    }

    // tests here use a real Postgres
}
```

Boot 3.1+ has built-in Testcontainers support via `@ServiceConnection`:
```java
@Container @ServiceConnection
static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16");
// Boot auto-wires datasource, no @DynamicPropertySource needed
```

### Test Configuration
```java
@TestConfiguration
class TestConfig {
    @Bean ClockProvider fixedClock() { return () -> Clock.fixed(...); }
}
```
Use `@Import(TestConfig.class)` or place in `src/test/java`.

### Useful Annotations
| | |
|---|---|
| `@TestPropertySource` | Override properties for one test class |
| `@DirtiesContext` | Force context reload (slow, use sparingly) |
| `@Transactional` | Wrap each test in tx + rollback (works with `@SpringBootTest`) |
| `@Sql` | Run SQL scripts before/after tests |
| `@WithMockUser` | Inject a fake authenticated user (Spring Security) |
| `@Tag("slow")` | Group tests for selective runs |

### Common Pitfalls
- **Context caching** — Spring caches contexts by config. Different `@TestPropertySource` / `@MockBean` combos = new context. `@DirtiesContext` defeats caching → slow build.
- **`@MockBean` invalidates cache** — every unique combination spawns a fresh context. Centralize mocks in shared test classes.
- **`@Transactional` with REST** — request runs in a different thread; rollback applies to the test's own thread. For HTTP tests, use Testcontainers + manual cleanup.
- **Random port** — get via `@LocalServerPort int port` or `TestRestTemplate`.
- **Slow tests due to full context** — most tests should be unit or slice tests.

### Recipe
- Service logic → unit test with mocks.
- Controller serialization → `@WebMvcTest`.
- Repository queries → `@DataJpaTest` (or with Testcontainers for Postgres-specific SQL).
- Full app smoke → `@SpringBootTest` + Testcontainers, kept few in number.
