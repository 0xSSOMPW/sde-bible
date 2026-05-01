# Q: What is the difference between Thread, Runnable, and Callable?

**Answer:**

### Thread (Class)
The most basic way to create a thread. Extend the `Thread` class and override `run()`.

```java
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Running in: " + Thread.currentThread().getName());
    }
}

MyThread t = new MyThread();
t.start(); // start(), NOT run() — run() doesn't create a new thread
```

**Downside:** Java doesn't support multiple inheritance. If your class already extends something, you can't extend `Thread`.

### Runnable (Interface) — Preferred
A functional interface with a single `run()` method. Decouples the task from the thread.

```java
Runnable task = () -> System.out.println("Running in: " + Thread.currentThread().getName());

Thread t = new Thread(task);
t.start();

// Or with ExecutorService (better):
ExecutorService executor = Executors.newFixedThreadPool(4);
executor.submit(task);
```

### Callable (Interface) — Returns a Result
Like `Runnable`, but the `call()` method **returns a value** and can **throw checked exceptions**.

```java
Callable<Integer> task = () -> {
    Thread.sleep(1000);
    return 42; // Returns a result
};

ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(task);

Integer result = future.get(); // Blocks until result is ready → 42
```

### Key Differences

| Feature | Thread | Runnable | Callable |
|---|---|---|---|
| **Type** | Class | Functional Interface | Functional Interface |
| **Method** | `run()` | `run()` | `call()` |
| **Return value** | ❌ void | ❌ void | ✅ Returns `V` |
| **Checked exceptions** | ❌ No | ❌ No | ✅ Yes |
| **Result handling** | N/A | N/A | Via `Future<V>` |
| **Use with Executor** | ❌ Not recommended | ✅ `submit(runnable)` | ✅ `submit(callable)` |

### When to Use What?

```
Simple fire-and-forget task?         → Runnable + ExecutorService
Need a result from the task?         → Callable + Future
Need exception propagation?          → Callable
Extending Thread directly?           → Almost never. Use Runnable.
```

> [!IMPORTANT]
> **Never extend `Thread` directly** in modern Java. Use `Runnable` or `Callable` with an `ExecutorService`. Direct thread management (creating, starting, joining threads manually) is error-prone and doesn't scale. Thread pools handle lifecycle, reuse, and scheduling for you.
