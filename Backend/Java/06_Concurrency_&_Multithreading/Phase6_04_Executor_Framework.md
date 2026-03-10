# 🔀 Phase 6.4 — The Executor Framework
### Thread Pools, `Future`, `Callable`, and `ThreadPoolExecutor`

---

## 🧠 Why Not Just Create Threads Directly?

You learned in Phase 6.1 that creating a `Thread` is straightforward. So why does
almost all production Java code use an Executor instead? The answer comes down to
three serious problems with raw thread creation in real applications.

**Creation cost.** Creating an OS-level thread requires allocating a stack (typically
512 KB to 1 MB), registering it with the OS scheduler, and initialising runtime
structures. In a server handling thousands of requests, creating a new thread per
request would be catastrophically expensive — the overhead of thread creation would
dwarf the time spent doing actual work.

**Unbounded resource consumption.** If you create a thread per incoming task and a
burst of 10,000 requests arrives, you get 10,000 threads. Each holds a megabyte of
stack, the OS scheduler thrashes trying to context-switch between all of them, and
the JVM likely runs out of memory. You need a way to *bound* the number of threads.

**No lifecycle management.** With raw threads, there is no built-in way to wait for
a group of tasks to complete, handle their exceptions in a unified way, or shut down
cleanly. You have to build all of that infrastructure yourself.

The Executor framework solves all three problems: it decouples the submission of a
task from the mechanics of how that task is run, reuses threads via pooling, and
provides lifecycle management.

---

## 🏗️ The Executor Hierarchy

The framework is built from a small set of interfaces that form a hierarchy.

`Executor` is the simplest — just one method: `execute(Runnable command)`. It
decouples task submission from task execution, but provides nothing beyond that.

`ExecutorService` extends `Executor` with lifecycle methods (`shutdown()`,
`awaitTermination()`), the ability to submit tasks that return results
(`submit(Callable<T>)`), and methods to run multiple tasks at once (`invokeAll()`,
`invokeAny()`).

`ScheduledExecutorService` extends `ExecutorService` with methods to schedule tasks
to run after a delay or repeatedly at a fixed rate or interval.

```
Executor
  └── ExecutorService
        └── ScheduledExecutorService
```

The concrete implementations you will use are `ThreadPoolExecutor` (which backs
most of the factory-created pools) and `ScheduledThreadPoolExecutor`.

---

## 🏭 Creating Executors — The `Executors` Factory

The `Executors` utility class provides factory methods for the most common thread pool
configurations. You will use these in most situations.

```java
import java.util.concurrent.*;

// Fixed thread pool: exactly N threads, always. Ideal for CPU-bound work where
// you want N = number of CPU cores. Tasks beyond N queue up and wait.
ExecutorService fixed = Executors.newFixedThreadPool(4);

// Cached thread pool: creates new threads on demand, reuses idle ones, removes
// threads idle for 60 seconds. Good for short-lived async tasks. Unbounded — can
// create thousands of threads under load, which is a risk in high-traffic code.
ExecutorService cached = Executors.newCachedThreadPool();

// Single-thread executor: exactly one thread, tasks execute in FIFO order.
// Useful for tasks that must execute sequentially with no concurrency.
ExecutorService single = Executors.newSingleThreadExecutor();

// Scheduled pool: supports delayed and periodic task execution
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

// Virtual thread executor (Java 21+): creates a new virtual thread per task.
// Discussed in detail in the Virtual Threads file.
ExecutorService virtual = Executors.newVirtualThreadPerTaskExecutor();
```

---

## 📤 Submitting Tasks — `Runnable` vs `Callable`

An `ExecutorService` accepts tasks in two forms. `Runnable` tasks have no return
value and cannot throw checked exceptions. `Callable<V>` tasks return a value of
type `V` and can throw checked exceptions — they are the right choice whenever you
need to get a result back from a background computation.

```java
ExecutorService executor = Executors.newFixedThreadPool(2);

// Submitting a Runnable — returns a Future<?> where get() always returns null
Future<?> runnableFuture = executor.submit(() ->
    System.out.println("Fire and forget task"));

// Submitting a Callable<Integer> — returns a Future<Integer>
Future<Integer> callableFuture = executor.submit(() -> {
    Thread.sleep(1000); // simulate a slow computation
    return 42;          // return a result
});

// execute() is for Runnable only, no return value, no exception forwarding
executor.execute(() -> System.out.println("Plain execution"));

executor.shutdown(); // initiate orderly shutdown
```

---

## 🔮 `Future<V>` — A Handle on an Asynchronous Result

When you submit a `Callable` to an `ExecutorService`, you get back a `Future<V>`.
This is a handle to a computation that may not have completed yet. The key methods are:

`get()` — blocks the calling thread until the result is available, then returns it.
If the callable threw an exception, `get()` wraps it in an `ExecutionException`.

`get(timeout, unit)` — like `get()` but throws `TimeoutException` if the result
doesn't arrive within the specified time. Always use this form in production — never
wait indefinitely.

`cancel(mayInterruptIfRunning)` — attempts to cancel the task. If `true`, it sends
an interrupt to the running thread.

`isDone()` — returns `true` if the computation finished (normally, by exception, or
by cancellation). Non-blocking.

`isCancelled()` — returns `true` if cancelled before completion.

```java
ExecutorService executor = Executors.newFixedThreadPool(2);

Callable<String> task = () -> {
    Thread.sleep(500);
    return "computation result";
};

Future<String> future = executor.submit(task);

// Do other work while the task runs in the background
System.out.println("Task submitted, doing other work...");
Thread.sleep(100);
System.out.println("isDone: " + future.isDone()); // false — still running

try {
    // Block until result is ready, but no longer than 2 seconds
    String result = future.get(2, TimeUnit.SECONDS);
    System.out.println("Result: " + result);

} catch (TimeoutException e) {
    System.out.println("Task timed out — cancelling");
    future.cancel(true); // interrupt the running thread

} catch (ExecutionException e) {
    // The callable threw an exception — unwrap and handle it
    Throwable cause = e.getCause();
    System.err.println("Task failed: " + cause.getMessage());

} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // always restore the interrupt flag
}

executor.shutdown();
```

### Submitting Multiple Tasks — `invokeAll()` and `invokeAny()`

`invokeAll()` submits a collection of `Callable`s and returns a list of completed
`Future`s — it blocks until **all** tasks have finished (or the timeout expires).
`invokeAny()` returns the result of whichever task completes first (successfully),
and cancels the remaining tasks.

```java
List<Callable<Integer>> tasks = List.of(
    () -> { Thread.sleep(300); return 1; },
    () -> { Thread.sleep(100); return 2; },
    () -> { Thread.sleep(200); return 3; }
);

// invokeAll — waits for all, returns all results in submission order
List<Future<Integer>> results = executor.invokeAll(tasks);
for (Future<Integer> f : results) {
    System.out.println(f.get()); // 1, 2, 3 (in order, regardless of completion time)
}

// invokeAny — returns the FIRST successful result (2 here, since it's fastest)
// and cancels the other two tasks
Integer fastest = executor.invokeAny(tasks);
System.out.println("Fastest result: " + fastest); // 2
```

`invokeAny` is excellent for the "redundant request" pattern: send the same query to
multiple data sources and use whichever answers first.

---

## ⚙️ `ThreadPoolExecutor` — Full Manual Control

All the `Executors` factory methods create `ThreadPoolExecutor` instances under the
hood. When you need fine-grained control over pool behaviour — sizing, queue type,
rejection policy — you construct a `ThreadPoolExecutor` directly.

The constructor signature reveals its full flexibility:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    2,                        // corePoolSize: always-alive threads (even when idle)
    4,                        // maximumPoolSize: maximum threads ever created
    60L,                      // keepAliveTime: how long idle threads beyond core size live
    TimeUnit.SECONDS,         // keepAliveTime unit
    new ArrayBlockingQueue<>(100), // work queue: where submitted tasks wait
    new ThreadFactory() {     // how to create new threads
        private int count = 0;
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r);
            t.setName("my-pool-" + (++count));
            t.setDaemon(false);
            return t;
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy() // what to do when queue is full
);
```

### Understanding the Thread Pool Lifecycle

This is the subtlest part of `ThreadPoolExecutor` and gets many developers wrong.
Here is exactly how it manages threads as tasks are submitted:

When the number of running threads is **below corePoolSize**, a new thread is created
for each incoming task, even if idle threads exist.

When running threads equal **corePoolSize**, new tasks go onto the **work queue**
(not into new threads). This is why `newFixedThreadPool(4)` uses a
`LinkedBlockingQueue` — it queues everything once all 4 core threads are busy.

When the **work queue is full** AND the thread count is below **maximumPoolSize**,
a new thread is created to handle the task immediately (bypassing the queue).

When the **work queue is full** AND the thread count equals **maximumPoolSize**, the
**rejection policy** is invoked.

```
submitted task
     │
     ▼
threads < corePoolSize? ──Yes──► create new thread
     │No
     ▼
queue not full? ──Yes──► enqueue task
     │No
     ▼
threads < maxPoolSize? ──Yes──► create new thread
     │No
     ▼
invoke RejectionHandler
```

### Rejection Policies

When both the queue and the thread pool are at maximum capacity, one of four built-in
rejection policies is applied:

`AbortPolicy` (the default) — throws `RejectedExecutionException`. Good when you
want to know when the system is overwhelmed.

`CallerRunsPolicy` — runs the task on the calling thread (the thread that called
`execute()`). This naturally slows down the producer, creating back-pressure. Often
the best choice for long-running processes where you don't want to lose tasks.

`DiscardPolicy` — silently drops the task. Use only when tasks are truly optional.

`DiscardOldestPolicy` — discards the oldest waiting task in the queue to make room
for the new one. Use for time-sensitive tasks where newer data is more valuable.

```java
// Custom rejection policy — log and fall back to synchronous execution
executor.setRejectedExecutionHandler((r, pool) -> {
    System.err.println("Task rejected! Queue full. Pool size: "
        + pool.getPoolSize() + ". Running synchronously.");
    if (!pool.isShutdown()) {
        r.run(); // run on caller's thread as fallback
    }
});
```

---

## 📅 Scheduled Executors

`ScheduledExecutorService` runs tasks after a delay or at regular intervals. It
replaces both `Timer` (which was error-prone and single-threaded) and raw
`Thread.sleep()` loops for scheduled work.

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

// schedule: run once after a delay
ScheduledFuture<?> onceDelayed = scheduler.schedule(
    () -> System.out.println("Runs after 5 seconds"),
    5, TimeUnit.SECONDS
);

// scheduleAtFixedRate: run every N time units from the start, regardless of
// how long the task takes. If a task takes longer than the period, the next
// run starts immediately after (no overlapping — runs are sequential).
ScheduledFuture<?> periodic = scheduler.scheduleAtFixedRate(
    () -> System.out.println("Heartbeat at " + System.currentTimeMillis()),
    0,    // initial delay
    1,    // period
    TimeUnit.SECONDS
);

// scheduleWithFixedDelay: wait N time units AFTER the previous task finishes
// before starting the next. Good when you don't want tasks to pile up.
ScheduledFuture<?> withDelay = scheduler.scheduleWithFixedDelay(
    () -> System.out.println("Polling..."),
    0,    // initial delay
    500,  // delay between end of one run and start of next
    TimeUnit.MILLISECONDS
);

// Cancel the periodic task after 5 seconds
Thread.sleep(5000);
periodic.cancel(false);  // false = don't interrupt if running
withDelay.cancel(false);
scheduler.shutdown();
```

The key difference between `scheduleAtFixedRate` and `scheduleWithFixedDelay` is
important in practice. A rate schedule says "run every 1 second, starting now." A
delay schedule says "wait 1 second after each run finishes, then run again." For a
task that occasionally takes 3 seconds, the rate schedule would fire the next run
immediately after the previous one (to catch up), while the delay schedule would
wait a full second after the 3-second task completes.

---

## 🔄 Proper Executor Shutdown

An `ExecutorService` holds threads that keep the JVM alive. If you never shut it
down, your application will never exit. Always shut down your executors.

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

// Submit tasks...

// STEP 1: Initiate orderly shutdown — no new tasks accepted, existing tasks finish
executor.shutdown();

try {
    // STEP 2: Wait for all running tasks to complete (up to 60 seconds)
    if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
        // STEP 3: Tasks haven't finished in time — force shutdown
        System.err.println("Forcing shutdown — tasks still running after 60s");
        List<Runnable> abandoned = executor.shutdownNow(); // interrupts running tasks
        System.err.println("Abandoned tasks: " + abandoned.size());

        // STEP 4: Wait a bit more for interrupted tasks to clean up
        if (!executor.awaitTermination(10, TimeUnit.SECONDS)) {
            System.err.println("Executor did not terminate cleanly");
        }
    }
} catch (InterruptedException e) {
    executor.shutdownNow(); // interrupted while waiting — force it
    Thread.currentThread().interrupt();
}
```

The difference between `shutdown()` and `shutdownNow()` is crucial. `shutdown()`
says "stop accepting new tasks but finish what's in the queue and running." It does
not interrupt anything. `shutdownNow()` says "stop everything immediately" — it
interrupts running threads and returns the list of tasks that were waiting in the
queue but never ran.

---

## 🧪 Complete Example — Parallel Web Scraper

```java
import java.util.*;
import java.util.concurrent.*;

public class ParallelScraper {

    // Simulates fetching a URL — may fail
    static String fetchUrl(String url) throws Exception {
        Thread.sleep((long)(Math.random() * 500) + 100);
        if (Math.random() < 0.1) throw new Exception("Network error for " + url);
        return "Content of " + url + " (" + Thread.currentThread().getName() + ")";
    }

    public static void main(String[] args) throws InterruptedException {
        List<String> urls = List.of(
            "https://example.com/page1", "https://example.com/page2",
            "https://example.com/page3", "https://example.com/page4",
            "https://example.com/page5"
        );

        int cpuCores = Runtime.getRuntime().availableProcessors();
        ExecutorService executor = Executors.newFixedThreadPool(cpuCores,
            r -> { Thread t = new Thread(r, "scraper-" + Thread.currentThread().getId()); return t; });

        // Submit all URLs as Callable tasks
        List<Future<String>> futures = new ArrayList<>();
        for (String url : urls) {
            futures.add(executor.submit(() -> fetchUrl(url)));
        }

        // Collect results in submission order
        int success = 0, failed = 0;
        for (int i = 0; i < urls.size(); i++) {
            try {
                String content = futures.get(i).get(3, TimeUnit.SECONDS);
                System.out.println("✓ " + content);
                success++;
            } catch (ExecutionException e) {
                System.err.println("✗ Failed: " + urls.get(i) + " — " + e.getCause().getMessage());
                failed++;
            } catch (TimeoutException e) {
                futures.get(i).cancel(true);
                System.err.println("✗ Timed out: " + urls.get(i));
                failed++;
            }
        }

        System.out.printf("%nResults: %d succeeded, %d failed%n", success, failed);

        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS);
    }
}
```

---

## ⚠️ Best Practices

**Always shut down your executors.** An executor with live threads will keep the JVM
from exiting. Use `shutdown()` + `awaitTermination()` in a `finally` block or
`@PreDestroy` method in Spring. Use try-with-resources when possible (Java 19+
made `ExecutorService` implement `AutoCloseable`).

**Size fixed thread pools correctly.** For CPU-bound tasks (heavy computation),
use `Runtime.getRuntime().availableProcessors()`. For I/O-bound tasks (network,
database, file I/O), you can use significantly more threads because they spend most
of their time waiting — a typical rule of thumb is `cores × (1 + wait_time / cpu_time)`.

**Never use `newCachedThreadPool()` in a server application** without careful
consideration. Under heavy load, it will create a thread per task with no upper limit,
which can exhaust memory. Always prefer a bounded fixed-size pool with an appropriate
queue and rejection policy for production server code.

**Always use `future.get(timeout, unit)` rather than `future.get()`** in production
code. An unbounded `get()` will block the calling thread forever if the task hangs —
always specify a timeout and have a plan for when it's exceeded.

**Handle `ExecutionException` properly.** The cause of an `ExecutionException` is
the exception thrown by the task. Always unwrap it with `e.getCause()` to get the
real error, and handle it appropriately rather than wrapping it in a generic error
message.

---

## 🔑 Key Takeaways

- The Executor framework decouples task submission from thread management, solving the cost and resource problems of raw thread creation.
- `execute(Runnable)` runs a task; `submit(Callable<T>)` runs a task and returns a `Future<T>` for its result.
- `Future.get(timeout, unit)` blocks the caller until the result is ready — always use a timeout.
- `ThreadPoolExecutor` has a precise lifecycle: fill core threads → queue tasks → create extra threads → reject when queue full and at max threads.
- The four rejection policies are `AbortPolicy` (throw), `CallerRunsPolicy` (run on caller — applies back-pressure), `DiscardPolicy` (silent drop), and `DiscardOldestPolicy`.
- `scheduleAtFixedRate` fires at fixed wall-clock intervals; `scheduleWithFixedDelay` waits a fixed time *after* each run completes.
- Always call `shutdown()` then `awaitTermination()`, with `shutdownNow()` as a fallback.
- For CPU-bound work, pool size ≈ number of CPU cores. For I/O-bound work, you can safely use many more threads.
