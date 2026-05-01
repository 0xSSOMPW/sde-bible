# Q: What is CompletableFuture and how does it compare to Future?

**Answer:**

### The Problem with Future
`Future` is blocking. `future.get()` blocks the calling thread until the result is ready. There's no way to chain tasks, combine results, or handle errors without blocking.

```java
Future<Integer> future = executor.submit(() -> expensiveComputation());
Integer result = future.get(); // 😴 Thread is BLOCKED here, doing nothing
```

### CompletableFuture — Non-Blocking, Composable
`CompletableFuture` (Java 8+) is a powerful asynchronous programming API that supports **non-blocking callbacks, chaining, combining, and error handling**.

```java
CompletableFuture.supplyAsync(() -> fetchUserFromDB(userId))   // Async task
    .thenApply(user -> enrichWithProfile(user))                 // Transform result
    .thenAccept(user -> sendWelcomeEmail(user))                 // Consume result
    .exceptionally(ex -> { log.error("Failed", ex); return null; });  // Handle errors
// No blocking! Everything runs asynchronously.
```

### Key Operations

**1. Creating:**
```java
CompletableFuture.supplyAsync(() -> "result");   // Returns a value
CompletableFuture.runAsync(() -> doSomething());  // Void (no return)
```

**2. Transforming (thenApply = map):**
```java
CompletableFuture<String> name = 
    CompletableFuture.supplyAsync(() -> getUser(1))
                     .thenApply(User::getName);
```

**3. Consuming (thenAccept):**
```java
future.thenAccept(result -> System.out.println("Got: " + result));
```

**4. Chaining (thenCompose = flatMap):**
```java
CompletableFuture<Order> order = 
    getUserAsync(1)
        .thenCompose(user -> getOrdersAsync(user.getId()));  // Returns another CF
```

**5. Combining two futures:**
```java
CompletableFuture<String> userFuture = getUserAsync();
CompletableFuture<List<Order>> ordersFuture = getOrdersAsync();

CompletableFuture<String> combined = userFuture.thenCombine(ordersFuture,
    (user, orders) -> user.getName() + " has " + orders.size() + " orders");
```

**6. Waiting for all / any:**
```java
CompletableFuture.allOf(future1, future2, future3).join(); // Wait for ALL
CompletableFuture.anyOf(future1, future2, future3).join(); // Wait for FIRST
```

### Error Handling

```java
CompletableFuture.supplyAsync(() -> riskyOperation())
    .thenApply(result -> process(result))
    .exceptionally(ex -> {
        log.error("Failed", ex);
        return fallbackValue;          // Recover with a default
    })
    .handle((result, ex) -> {         // Access both result and exception
        if (ex != null) return "error";
        return result;
    });
```

### Future vs CompletableFuture

| Feature | Future | CompletableFuture |
|---|---|---|
| **Blocking** | ✅ `get()` blocks | ❌ Callbacks are non-blocking |
| **Chaining** | ❌ | ✅ `thenApply`, `thenCompose` |
| **Combining** | ❌ | ✅ `thenCombine`, `allOf`, `anyOf` |
| **Error handling** | Try-catch around `get()` | `exceptionally()`, `handle()` |
| **Manual completion** | ❌ | ✅ `complete()`, `completeExceptionally()` |

> [!TIP]
> Think of `CompletableFuture` as Java's equivalent of JavaScript `Promise`. `thenApply` = `.then()`, `thenCompose` = `.then()` that returns another Promise, `exceptionally` = `.catch()`.
