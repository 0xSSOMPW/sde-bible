# Q: `@Controller` vs `@RestController`. How do request mappings, validation, and content negotiation work?

**Answer:**

### `@Controller` vs `@RestController`
- `@Controller` — returns view names (Thymeleaf, JSP). Methods must `@ResponseBody` to return raw data.
- `@RestController` = `@Controller` + `@ResponseBody` on every method. JSON/XML by default.

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping("/{id}")
    Order get(@PathVariable Long id) { ... }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    Order create(@RequestBody @Valid CreateOrderRequest req) { ... }

    @PutMapping("/{id}")
    Order update(@PathVariable Long id, @RequestBody @Valid Order o) { ... }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    void delete(@PathVariable Long id) { ... }
}
```

### Mapping Annotations
```java
@RequestMapping(value="/x", method=GET)   // generic
@GetMapping("/x")                          // shortcut
@PostMapping @PutMapping @DeleteMapping @PatchMapping
```

### Parameters
| Source | Annotation | Example |
|---|---|---|
| URL path variable | `@PathVariable` | `/users/{id}` |
| Query string | `@RequestParam` | `?page=1&size=20` |
| Header | `@RequestHeader` | `Authorization` |
| Cookie | `@CookieValue` | |
| JSON body | `@RequestBody` | POST body |
| Form (`x-www-form-urlencoded`) | `@RequestParam` per field | |
| File upload | `@RequestPart` / `MultipartFile` | |
| Whole request | `HttpServletRequest` | |

### Validation
Add `spring-boot-starter-validation`. Use Jakarta Bean Validation:

```java
public record CreateOrderRequest(
    @NotBlank String customerEmail,
    @Min(1) int quantity,
    @Size(max = 500) String notes
) {}

@PostMapping
Order create(@RequestBody @Valid CreateOrderRequest req) { ... }
// Invalid → 400 with MethodArgumentNotValidException
```

For path/query params:
```java
@GetMapping("/orders")
List<Order> list(
    @RequestParam @Min(0) int page,
    @RequestParam @Max(100) int size) { ... }

// requires @Validated on the controller class
```

### Response Status & Headers
```java
@PostMapping
ResponseEntity<Order> create(@RequestBody @Valid CreateOrderRequest req) {
    Order o = service.create(req);
    return ResponseEntity
        .created(URI.create("/api/orders/" + o.id()))
        .header("X-Trace-Id", traceId())
        .body(o);
}
```

### Content Negotiation
Spring picks `HttpMessageConverter` based on `Accept` header + `produces` attribute.

```java
@GetMapping(value="/{id}", produces={"application/json","application/xml"})
Order get(@PathVariable Long id) { ... }
```

Add Jackson XML / YAML modules to support those.

### `consumes` (Body Type Restriction)
```java
@PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
```
Wrong `Content-Type` → 415.

### Common Annotations
- `@CrossOrigin` — CORS at controller level (or use a global CORS config).
- `@ModelAttribute` — bind form fields to a POJO.
- `@SessionAttribute`, `@RequestAttribute`.

### Error Handling
Per-controller:
```java
@ExceptionHandler(OrderNotFoundException.class)
@ResponseStatus(HttpStatus.NOT_FOUND)
ProblemDetail handleNotFound(OrderNotFoundException e) {
    return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, e.getMessage());
}
```

Global → see `@ControllerAdvice` (separate question).

### `ResponseEntity` vs Direct Return
- Direct return — cleaner when status is always the same (`@ResponseStatus`).
- `ResponseEntity` — when you need to vary status, headers, or body shape.

### Async Responses
- `Callable<T>` — runs on `TaskExecutor`, frees the servlet thread.
- `DeferredResult<T>` — completed from another thread.
- `CompletableFuture<T>` — wraps async pipelines.
- `ResponseBodyEmitter` / `SseEmitter` — server-sent events / streams.

### WebFlux Variant
Same annotations, but methods return `Mono<T>` / `Flux<T>` and run reactively.

```java
@RestController
class OrderControllerR {
    @GetMapping("/{id}")
    Mono<Order> get(@PathVariable Long id) { return repo.findById(id); }
}
```

### Test
```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mvc;
    @MockBean OrderService service;

    @Test
    void createOrder() throws Exception {
        when(service.create(any())).thenReturn(new Order(1L));
        mvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""{"customerEmail":"a@b.com","quantity":2}"""))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1));
    }
}
```
