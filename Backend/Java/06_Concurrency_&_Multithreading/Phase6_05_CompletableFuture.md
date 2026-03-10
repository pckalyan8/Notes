# 🔀 Phase 6.5 — `CompletableFuture` in Java
### Asynchronous Pipelines Without the Callback Hell

---

## 🧠 The Problem With Plain `Future`

You saw in Phase 6.4 that `Future<V>` is a handle on a result that isn't ready yet.
But `Future` has a fundamental limitation: it is **purely blocking**. To get the
result, you must call `get()`, which parks the calling thread until the computation
finishes. There is no way to say "when this computation finishes, automatically run
the next step." There is no way to compose two futures — to say "when both A and B
are done, combine their results." There is no non-blocking way to attach error
handling. Every reactive or asynchronous operation ends up as a blocking `get()` call
somewhere, which defeats much of the purpose of running things asynchronously.

Java 8 introduced `CompletableFuture<T>` to solve this. It implements `Future<T>`
(so it's compatible with existing code), but adds a rich API for composing, combining,
and reacting to asynchronous computations — without the caller needing to block.

Think of `CompletableFuture` as a pipeline builder. Instead of "do this task, then
block and wait for the answer, then manually do the next thing," you describe the
entire pipeline upfront: "do task A, then transform its result, then combine with task
B, and if anything goes wrong, recover by doing C." The whole pipeline runs
asynchronously, and the calling thread is never blocked.

---

## 🏗️ Creating a `CompletableFuture`

### `supplyAsync` and `runAsync` — Kicking Off Async Work

`CompletableFuture.supplyAsync()` starts a `Supplier<T>` (returns a value) on a
thread pool. `CompletableFuture.runAsync()` starts a `Runnable` (no return value).
Without specifying an executor, both use the common `ForkJoinPool`. In production,
always provide your own executor so you have full control over the thread pool.

```java
import java.util.concurrent.*;

// --- Without custom executor (uses ForkJoinPool.commonPool()) ---
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
    // runs on a ForkJoinPool thread
    return fetchUserFromDatabase(42); // some slow operation
});

// --- With a custom executor (strongly preferred in production) ---
ExecutorService ioPool = Executors.newFixedThreadPool(8); // dedicated pool for I/O

CompletableFuture<String> future2 = CompletableFuture.supplyAsync(
    () -> fetchUserFromDatabase(42),
    ioPool // run on THIS pool, not the common pool
);

// runAsync — for tasks with no return value (logging, cache-warming, etc.)
CompletableFuture<Void> runFuture = CompletableFuture.runAsync(
    () -> warmUpCache(),
    ioPool
);

// Creating an already-completed future — useful in tests or for synchronous values
CompletableFuture<Integer> immediate = CompletableFuture.completedFuture(42);
```

Why avoid the common `ForkJoinPool` in production? Because it is shared by the entire
JVM — your code, framework internals, and parallel streams all compete for its
threads. A single slow I/O task submitted to the common pool can starve all other
uses. Always isolate I/O-bound async work on its own dedicated pool.

---

## 🔗 Transformation — `thenApply`, `thenAccept`, `thenRun`

These are the primary pipeline-building methods. Each attaches a callback that runs
when the preceding stage completes, and each returns a new `CompletableFuture`
representing that stage's result.

`thenApply(Function<T, R>)` transforms the result — like `map()` in streams.
`thenAccept(Consumer<T>)` consumes the result but produces no new value (the
returned `CompletableFuture<Void>` signals when the consumer has run).
`thenRun(Runnable)` runs after completion but receives no value from the previous stage.

```java
CompletableFuture<String> pipeline = CompletableFuture
    .supplyAsync(() -> fetchUserName(42), ioPool)       // Stage 1: fetch name
    .thenApply(name -> name.toUpperCase())               // Stage 2: transform it
    .thenApply(upper -> "Hello, " + upper + "!");        // Stage 3: format it

// The pipeline is running asynchronously. When we call get(), we block just once.
String greeting = pipeline.get();
System.out.println(greeting); // "Hello, ALICE!"

// thenAccept: consume the result — used for side effects at the end of a pipeline
CompletableFuture<Void> logPipeline = CompletableFuture
    .supplyAsync(() -> computeReport(), ioPool)
    .thenApply(report -> formatReport(report))
    .thenAccept(formatted -> saveToFile(formatted)); // no return value

// thenRun: run after completion, ignore the result
CompletableFuture<Void> notification = pipeline
    .thenRun(() -> System.out.println("Pipeline complete!"));
```

### The `Async` Suffix — Controlling Which Thread Runs the Callback

Every transformation method has an `Async` variant: `thenApplyAsync`, `thenAcceptAsync`,
`thenRunAsync`. The non-Async version runs the callback on *whichever thread completed
the previous stage* (often the thread pool thread). The Async version submits the
callback to a specified executor (or the common pool if none given).

This matters when the callback is slow or when you want to return the thread pool
thread to the pool immediately after producing the result:

```java
CompletableFuture.supplyAsync(() -> fetchData(), ioPool)    // I/O pool thread
    .thenApplyAsync(data -> heavyComputation(data), cpuPool) // CPU pool thread
    .thenAcceptAsync(result -> saveResult(result), ioPool);  // back to I/O pool
```

This pattern — different stages on different pools — is how you build truly
non-blocking, resource-efficient async pipelines in production.

---

## 🔀 Combining Futures — `thenCompose` and `thenCombine`

### `thenCompose` — Chaining Dependent Async Operations (Flatmap)

`thenApply` works when your callback returns a plain value. But what if your callback
itself returns a `CompletableFuture`? Using `thenApply` would give you a
`CompletableFuture<CompletableFuture<T>>` — a nested future that you'd have to
unwrap manually. `thenCompose` is the solution — it is the async equivalent of
`flatMap()`: it takes a function that returns a `CompletableFuture`, and flattens the
nesting so you get `CompletableFuture<T>`.

```java
// Scenario: fetch user, then fetch their orders (both are async operations)

CompletableFuture<User> fetchUser = CompletableFuture
    .supplyAsync(() -> getUser(42), ioPool);

// With thenApply — WRONG: produces CompletableFuture<CompletableFuture<List<Order>>>
CompletableFuture<CompletableFuture<List<Order>>> nestedWrong =
    fetchUser.thenApply(user -> getOrdersForUser(user)); // getOrdersForUser returns CF

// With thenCompose — CORRECT: produces CompletableFuture<List<Order>>
CompletableFuture<List<Order>> orders = fetchUser
    .thenCompose(user -> getOrdersForUser(user)); // flattens the nesting

// Real pipeline: user → orders → total value
CompletableFuture<Double> totalValue = CompletableFuture
    .supplyAsync(() -> getUser(42), ioPool)
    .thenCompose(user -> getOrdersForUser(user))          // async: user → orders
    .thenCompose(orderList -> calculateTotal(orderList))   // async: orders → total
    .thenApply(total -> total * 1.1);                      // sync: apply tax
```

The mental model: use `thenApply` when your transformation is synchronous (returns a
plain value). Use `thenCompose` when your transformation is asynchronous (returns
another `CompletableFuture`).

### `thenCombine` — Merging Two Independent Futures

`thenCombine` runs two futures **in parallel**, then combines their results when both
complete. This is perfect for fetching data from two independent sources simultaneously.

```java
// Fetch user profile and user preferences in parallel
CompletableFuture<UserProfile> profileFuture = CompletableFuture
    .supplyAsync(() -> fetchProfile(42), ioPool);

CompletableFuture<UserPreferences> prefsFuture = CompletableFuture
    .supplyAsync(() -> fetchPreferences(42), ioPool);

// When BOTH are done, combine the results
CompletableFuture<UserView> userView = profileFuture.thenCombine(
    prefsFuture,
    (profile, prefs) -> new UserView(profile, prefs) // BiFunction: (T, U) -> V
);
// Both fetches ran in parallel — total time ≈ max(profile time, prefs time)
```

---

## 🚀 Waiting for Multiple Futures — `allOf` and `anyOf`

### `allOf` — Wait for All to Complete

`allOf` takes a varargs array of `CompletableFuture<?>` and returns a
`CompletableFuture<Void>` that completes when *all* of them complete. Note that it
doesn't carry the results directly — you collect results yourself after it completes.

```java
List<String> urls = List.of("url1", "url2", "url3", "url4", "url5");

// Launch all fetches in parallel
List<CompletableFuture<String>> fetches = urls.stream()
    .map(url -> CompletableFuture.supplyAsync(() -> fetch(url), ioPool))
    .collect(Collectors.toList());

// Create an allOf future and chain collection of results
CompletableFuture<List<String>> allResults = CompletableFuture
    .allOf(fetches.toArray(new CompletableFuture[0]))
    .thenApply(v ->
        fetches.stream()
               .map(CompletableFuture::join) // join() is like get() but unchecked
               .collect(Collectors.toList())
    );

List<String> contents = allResults.get(); // wait for all and get the list
System.out.println("Fetched " + contents.size() + " pages");
```

### `anyOf` — Race Condition (First to Win)

`anyOf` returns a `CompletableFuture<Object>` that completes when *any one* of the
input futures completes first. The result is the result of that first-completing
future. The other futures continue running (they are not cancelled automatically).

```java
// Query three redundant data sources — use whichever answers first
CompletableFuture<String> fromPrimary   = CompletableFuture.supplyAsync(() -> queryPrimary());
CompletableFuture<String> fromReplica1  = CompletableFuture.supplyAsync(() -> queryReplica1());
CompletableFuture<String> fromReplica2  = CompletableFuture.supplyAsync(() -> queryReplica2());

CompletableFuture<Object> fastest = CompletableFuture.anyOf(fromPrimary, fromReplica1, fromReplica2);
String result = (String) fastest.get(); // use whichever answered fastest
System.out.println("Got answer: " + result);
```

---

## ❗ Error Handling — `exceptionally`, `handle`, `whenComplete`

Error handling in async pipelines requires specific methods because exceptions from
background threads are captured inside the `CompletableFuture` and surface only when
you call `get()` or when you attach an error handler. There are three approaches.

### `exceptionally(Function<Throwable, T>)` — Recover from Failure

`exceptionally` is the async `catch` block. If the pipeline completes exceptionally,
the function receives the exception and returns a recovery value of the same type.
If the pipeline succeeds, this stage passes the value through unchanged.

```java
CompletableFuture<String> result = CompletableFuture
    .supplyAsync(() -> fetchFromExternalService())  // might throw
    .thenApply(data -> processData(data))
    .exceptionally(ex -> {
        System.err.println("Service failed: " + ex.getMessage());
        return "default_value"; // fallback value on failure
    });

String value = result.get(); // never throws — either real result or "default_value"
```

### `handle(BiFunction<T, Throwable, R>)` — Handle Both Success and Failure

`handle` is called in both the success and failure cases. One of its arguments will
be `null` — if the pipeline succeeded, `throwable` is null; if it failed, `result`
is null. This is useful when you need to perform cleanup or logging regardless of outcome.

```java
CompletableFuture<String> handled = CompletableFuture
    .supplyAsync(() -> fetchData())
    .handle((result, ex) -> {
        if (ex != null) {
            System.err.println("Operation failed: " + ex.getMessage());
            return "fallback";   // recover
        }
        return result.toUpperCase(); // transform success
    });
```

### `whenComplete(BiConsumer<T, Throwable>)` — Observe Without Changing the Result

`whenComplete` is called for both success and failure, just like `handle`, but it
does not change the result — it's purely for side effects like logging or metrics.
The result (or exception) passes through unchanged.

```java
CompletableFuture<String> observed = CompletableFuture
    .supplyAsync(() -> fetchData())
    .whenComplete((result, ex) -> {
        if (ex != null) {
            metrics.incrementCounter("api.error");
            logger.error("Async operation failed", ex);
        } else {
            metrics.incrementCounter("api.success");
            logger.info("Operation succeeded: {}", result);
        }
        // result or exception is still propagated after this — not changed
    });
```

---

## ✋ `join()` vs `get()`

Both `join()` and `get()` block the calling thread until the `CompletableFuture`
completes. The difference is exception handling. `get()` throws checked exceptions
(`InterruptedException` and `ExecutionException`) so you must catch them. `join()`
throws unchecked `CompletionException` instead, which is convenient but means
exceptions can propagate without forced handling. Use `get()` in code where you want
explicit exception handling; use `join()` inside stream pipelines where checked
exceptions are inconvenient.

```java
// get() — checked exceptions
try {
    String result = future.get(5, TimeUnit.SECONDS);
} catch (ExecutionException | InterruptedException | TimeoutException e) { /* */ }

// join() — unchecked, convenient inside lambdas
String result = future.join(); // throws CompletionException if failed

// Inside a stream pipeline — join() works; get() would require try/catch everywhere
List<String> results = futures.stream()
    .map(CompletableFuture::join) // clean — no try/catch needed
    .collect(Collectors.toList());
```

---

## 🧪 Complete Real-World Example — Async API Aggregation

This example shows a realistic async pipeline that fetches a user, their orders, and
product details all concurrently, with error handling and timeout.

```java
import java.util.concurrent.*;
import java.util.*;
import java.util.stream.*;

public class UserDashboardService {

    private final ExecutorService ioPool = Executors.newFixedThreadPool(10);

    // Simulated async API calls
    CompletableFuture<User> fetchUser(int id) {
        return CompletableFuture.supplyAsync(() -> {
            simulateDelay(200);
            return new User(id, "Alice", "alice@example.com");
        }, ioPool);
    }

    CompletableFuture<List<Order>> fetchOrders(int userId) {
        return CompletableFuture.supplyAsync(() -> {
            simulateDelay(300);
            return List.of(new Order(101, userId, "P1"), new Order(102, userId, "P2"));
        }, ioPool);
    }

    CompletableFuture<Product> fetchProduct(String productId) {
        return CompletableFuture.supplyAsync(() -> {
            simulateDelay(150);
            if (productId.equals("FAIL")) throw new RuntimeException("Product not found");
            return new Product(productId, "Product " + productId, 29.99);
        }, ioPool);
    }

    CompletableFuture<Dashboard> buildDashboard(int userId) {
        // Step 1: Fetch user and orders in parallel (independent)
        CompletableFuture<User> userFuture = fetchUser(userId);
        CompletableFuture<List<Order>> ordersFuture = fetchOrders(userId);

        // Step 2: Once orders arrive, fetch all products in parallel
        CompletableFuture<List<Product>> productsFuture = ordersFuture
            .thenCompose(orders -> {
                // Create a fetch future for each product
                List<CompletableFuture<Product>> productFutures = orders.stream()
                    .map(o -> fetchProduct(o.productId())
                        .exceptionally(ex -> new Product(o.productId(), "Unknown", 0))) // recover
                    .collect(Collectors.toList());

                // Wait for all product fetches
                return CompletableFuture
                    .allOf(productFutures.toArray(new CompletableFuture[0]))
                    .thenApply(v -> productFutures.stream()
                        .map(CompletableFuture::join)
                        .collect(Collectors.toList()));
            });

        // Step 3: Combine user + orders + products into a Dashboard
        return userFuture
            .thenCombine(ordersFuture, (user, orders) ->
                new PartialDashboard(user, orders))
            .thenCombine(productsFuture, (partial, products) ->
                new Dashboard(partial.user(), partial.orders(), products))
            .orTimeout(5, TimeUnit.SECONDS) // Java 9+: fail if too slow
            .exceptionally(ex -> Dashboard.errorDashboard("Timed out or failed: " + ex.getMessage()));
    }

    static void simulateDelay(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }

    public static void main(String[] args) throws Exception {
        UserDashboardService service = new UserDashboardService();
        long start = System.currentTimeMillis();

        Dashboard dashboard = service.buildDashboard(42).get();
        System.out.printf("Dashboard built in %d ms%n", System.currentTimeMillis() - start);
        // Prints ~max(200, 300+150) ms ≈ 450ms instead of 200+300+150 = 650ms sequential

        service.ioPool.shutdown();
    }

    // Records for the example
    record User(int id, String name, String email) {}
    record Order(int id, int userId, String productId) {}
    record Product(String id, String name, double price) {}
    record PartialDashboard(User user, List<Order> orders) {}
    record Dashboard(User user, List<Order> orders, List<Product> products) {
        static Dashboard errorDashboard(String msg) { return new Dashboard(null, List.of(), List.of()); }
    }
}
```

---

## 🕐 Timeouts — `orTimeout` and `completeOnTimeout` (Java 9+)

```java
// orTimeout: complete exceptionally with TimeoutException if not done in time
CompletableFuture<String> bounded = future
    .orTimeout(3, TimeUnit.SECONDS); // throws TimeoutException after 3s

// completeOnTimeout: complete with a default value if not done in time (no exception)
CompletableFuture<String> withDefault = future
    .completeOnTimeout("default-result", 3, TimeUnit.SECONDS);
```

Always apply timeouts to your `CompletableFuture` pipelines in production. An async
pipeline without a timeout can hang indefinitely if a downstream service is slow or
unresponsive.

---

## ⚠️ Best Practices

**Always specify an executor for async operations.** The default common `ForkJoinPool`
is shared JVM-wide. Submitting I/O-bound work to it can starve parallel streams and
framework internals. Use a dedicated `ExecutorService` for your async operations.

**Use `thenCompose` for async-to-async chaining, `thenApply` for sync transformations.**
The most common mistake is using `thenApply` when the callback returns a
`CompletableFuture`, producing nested futures. If your callback starts another async
operation, always reach for `thenCompose`.

**Apply `orTimeout` to every pipeline that touches external systems.** Databases,
HTTP services, message brokers — any external call can hang. A `CompletableFuture`
without a timeout is a latent hung-thread bug.

**Don't call `get()` or `join()` deep inside a pipeline.** Calling `get()` inside a
`thenApply` lambda blocks the thread pool thread, defeating the purpose of async
programming. Only call `get()`/`join()` at the boundaries — at the very end of the
pipeline when you need the result.

**Use `whenComplete` for cross-cutting concerns** like metrics and logging. It
observes without interfering with the result — keep this behaviour separate from your
business logic transformations.

---

## 🔑 Key Takeaways

- `CompletableFuture` solves the `Future` limitation by enabling non-blocking composition of async operations instead of forced blocking with `get()`.
- `supplyAsync` starts async work that returns a value; `runAsync` starts async work with no return.
- Always specify a custom executor — avoid the default `ForkJoinPool.commonPool()` for I/O-bound work.
- `thenApply` transforms a result synchronously; `thenCompose` chains when your transformation is itself async (flattens nested futures).
- `thenCombine` runs two futures in parallel and merges results when both complete.
- `allOf` waits for all futures; `anyOf` uses whichever finishes first.
- Error handling: `exceptionally` for recovery, `handle` for handling both success and failure, `whenComplete` for observation only.
- Always apply `orTimeout` or `completeOnTimeout` in production pipelines that touch external systems.
