# Q: How do you handle exceptions globally in Spring Boot? `@ControllerAdvice`, `ProblemDetail`, RFC 7807.

**Answer:**

Three layers, each broader scope:
1. **`try/catch` in handler** — local, ugly, boilerplate.
2. **`@ExceptionHandler` in controller** — per-controller.
3. **`@ControllerAdvice`** — global across all (or selected) controllers.

### `@ExceptionHandler` (Per-Controller)
```java
@RestController
class OrderController {
    @ExceptionHandler(OrderNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    ErrorResponse notFound(OrderNotFoundException e) {
        return new ErrorResponse(e.getMessage());
    }
}
```

### `@ControllerAdvice` / `@RestControllerAdvice` (Global)
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ProblemDetail> notFound(EntityNotFoundException e) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, e.getMessage());
        pd.setType(URI.create("https://api.acme.com/errors/not-found"));
        pd.setProperty("timestamp", Instant.now());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(pd);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ProblemDetail> validation(MethodArgumentNotValidException e) {
        Map<String, String> errors = e.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(FieldError::getField, FieldError::getDefaultMessage, (a,b)->a));

        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Validation failed");
        pd.setProperty("errors", errors);
        return ResponseEntity.badRequest().body(pd);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ProblemDetail> fallback(Exception e) {
        log.error("Unhandled exception", e);
        return ResponseEntity.internalServerError()
            .body(ProblemDetail.forStatusAndDetail(HttpStatus.INTERNAL_SERVER_ERROR, "Internal error"));
    }
}
```

`@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`.

### `ProblemDetail` (Spring 6+, Boot 3+)
RFC 7807 standard error format.

```json
{
  "type": "https://api.acme.com/errors/not-found",
  "title": "Not Found",
  "status": 404,
  "detail": "Order 42 not found",
  "instance": "/api/orders/42",
  "timestamp": "2026-04-26T10:00:00Z"
}
```

Enable RFC 7807 default behavior:
```yaml
spring:
  mvc:
    problemdetails:
      enabled: true
```

Built-in Spring exceptions (404, 405, 415, etc.) auto-respond with `ProblemDetail`.

### `ResponseStatusException` (Quick Throw)
```java
throw new ResponseStatusException(HttpStatus.NOT_FOUND, "order " + id + " not found");
```

### Custom Exception Hierarchy
```java
public abstract class AppException extends RuntimeException {
    private final HttpStatus status;
    private final String code;

    protected AppException(HttpStatus status, String code, String msg) {
        super(msg);
        this.status = status;
        this.code = code;
    }
    // getters
}

public class OrderNotFoundException extends AppException {
    public OrderNotFoundException(long id) {
        super(HttpStatus.NOT_FOUND, "ORDER_NOT_FOUND", "order " + id);
    }
}

@ExceptionHandler(AppException.class)
public ResponseEntity<ProblemDetail> handle(AppException e) {
    ProblemDetail pd = ProblemDetail.forStatusAndDetail(e.getStatus(), e.getMessage());
    pd.setProperty("code", e.getCode());
    return ResponseEntity.status(e.getStatus()).body(pd);
}
```

### Scope `@ControllerAdvice`
```java
@RestControllerAdvice(basePackages = "com.acme.api.public")
@RestControllerAdvice(annotations = RestController.class)
@RestControllerAdvice(assignableTypes = {OrderController.class, UserController.class})
```

### `ResponseEntityExceptionHandler` Base
For full control over Spring's built-in exceptions, extend it:
```java
@RestControllerAdvice
public class ApiExceptionHandler extends ResponseEntityExceptionHandler {
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex, HttpHeaders h, HttpStatusCode s, WebRequest r) {
        // your custom shape
    }
}
```

### Validation Errors — Field-Level Messages
```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ProblemDetail> handleValidation(MethodArgumentNotValidException e) {
    List<Map<String, String>> errors = e.getBindingResult().getFieldErrors().stream()
        .map(f -> Map.of("field", f.getField(), "message", f.getDefaultMessage()))
        .toList();

    ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
    pd.setTitle("Validation failed");
    pd.setProperty("errors", errors);
    return ResponseEntity.badRequest().body(pd);
}
```

### Constraint Violations on Path/Query Params
Different exception type:
```java
@ExceptionHandler(ConstraintViolationException.class)
public ResponseEntity<ProblemDetail> constraintViolation(ConstraintViolationException e) {
    ...
}
```

### Pitfalls
- **Logging duplication**: don't log + rethrow + log again. Pick one layer.
- **Exposing internals**: never serialize `e.getMessage()` blindly — may leak DB schema, paths.
- **404 vs 200 with empty body**: pick a convention. `Optional<T>` controllers + `orElseThrow` pattern is common.
- **Order of advice**: `@Order` controls precedence when multiple `@ControllerAdvice` could handle the same exception.
- **Async exceptions**: `@ExceptionHandler` doesn't catch errors from inside `@Async` methods — handle there or via `AsyncUncaughtExceptionHandler`.

### Best Practice Checklist
- One global `@RestControllerAdvice`.
- `ProblemDetail` for response shape.
- Domain exceptions → mapped statuses, never raw 500.
- Validation handled separately with field-level breakdown.
- Catch-all `Exception.class` last → log full stack, return generic 500.
