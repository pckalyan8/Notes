# 🔀 Phase 6.7 — Synchronizers & Concurrency Patterns
### `CountDownLatch`, `CyclicBarrier`, `Semaphore`, `Phaser`, `Exchanger`, and Classic Patterns

---

## 🧠 What Are Synchronizers?

So far you've learned how to protect shared data (with locks) and how to submit and
manage asynchronous work (with executors and `CompletableFuture`). But many real
concurrency problems are about **coordination** between threads — making threads wait
for each other, limiting simultaneous access, or exchanging data at a meeting point.
The synchronizer classes in `java.util.concurrent` are purpose-built tools for exactly
these kinds of coordination problems. Each one captures a specific coordination
pattern so you never have to implement it yourself from `wait()` and `notifyAll()`.

---

## ⏳ `CountDownLatch` — Wait for N Events to Happen

A `CountDownLatch` is initialised with a count `N`. Threads that need to wait call
`await()`, which blocks until the count reaches zero. Other threads call `countDown()`
to decrement the count. When the last `countDown()` brings the count to zero, all
waiting threads are released simultaneously.

The critical characteristic of a `CountDownLatch` is that it is **one-shot** — once
the count reaches zero, it stays at zero and all future `await()` calls return
immediately. You cannot reset it.

Think of it like the starting pistol at a race: once fired (count → 0), the gate
opens and never closes again.

```java
import java.util.concurrent.*;

public class CountDownLatchDemo {

    public static void main(String[] args) throws InterruptedException {
        int workerCount = 5;

        // PATTERN 1: One latch to coordinate the START of all workers
        // Main thread fires the starting gun; all workers begin simultaneously
        CountDownLatch startLatch = new CountDownLatch(1);

        // PATTERN 2: One latch to wait for all workers to FINISH
        CountDownLatch doneLatch = new CountDownLatch(workerCount);

        for (int i = 1; i <= workerCount; i++) {
            final int id = i;
            new Thread(() -> {
                try {
                    // Wait for the starting signal before doing any work
                    startLatch.await();
                    System.out.printf("[Worker-%d] Starting work...%n", id);

                    Thread.sleep((long)(Math.random() * 1000)); // simulate work

                    System.out.printf("[Worker-%d] Finished!%n", id);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    doneLatch.countDown(); // signal "I'm done" regardless of success/failure
                }
            }, "worker-" + id).start();
        }

        System.out.println("All workers ready. Firing starting gun...");
        Thread.sleep(200); // small delay to ensure all workers have called await()
        startLatch.countDown(); // release all workers simultaneously

        System.out.println("Main waiting for all workers to finish...");
        boolean finished = doneLatch.await(10, TimeUnit.SECONDS); // wait with timeout
        if (finished) {
            System.out.println("All workers done! Compiling results...");
        } else {
            System.err.println("Timeout: some workers didn't finish in time");
        }
    }
}
```

`CountDownLatch` is ideal for scenarios like:
- A test harness launching N threads simultaneously to create contention
- A startup sequence where a main thread waits for N services to initialise before accepting requests
- A batch job that waits for N parallel data-loading threads before beginning processing

---

## 🔁 `CyclicBarrier` — All Threads Rendezvous at a Point

A `CyclicBarrier` is initialised with a count `N`. When a thread calls `await()`, it
blocks until `N` threads have all called `await()`. At that point, all N threads are
released simultaneously — and the barrier **resets** automatically for the next round.
This cyclic nature is what distinguishes it from `CountDownLatch`.

Think of it like a relay race where all runners must be at the baton-exchange zone
before anyone can continue. And it repeats for every lap.

```java
import java.util.concurrent.*;

public class CyclicBarrierDemo {

    public static void main(String[] args) throws InterruptedException {
        int threadCount = 4;
        int rounds = 3;

        // Optional barrier action: runs once when all threads reach the barrier,
        // before they are all released. Useful for combining partial results.
        Runnable barrierAction = () ->
            System.out.printf("--- Round complete! Synchronisation point reached by all %d threads ---%n", threadCount);

        CyclicBarrier barrier = new CyclicBarrier(threadCount, barrierAction);

        for (int i = 0; i < threadCount; i++) {
            final int threadId = i;
            new Thread(() -> {
                try {
                    for (int round = 1; round <= rounds; round++) {
                        // Simulate variable-length work in each round
                        Thread.sleep((long)(Math.random() * 500) + 100);
                        System.out.printf("[Thread-%d] Finished round %d, waiting at barrier...%n",
                            threadId, round);

                        barrier.await(); // wait for all threads to reach this point
                        // All threads are released together here — barrier resets automatically

                        System.out.printf("[Thread-%d] Continuing round %d...%n", threadId, round);
                    }
                } catch (InterruptedException | BrokenBarrierException e) {
                    // BrokenBarrierException: thrown if barrier is broken (e.g., another thread
                    // timed out or was interrupted while waiting)
                    Thread.currentThread().interrupt();
                }
            }, "thread-" + i).start();
        }
    }
}
```

`CyclicBarrier` is ideal for parallel algorithms that process data in phases, where
each phase must complete across all workers before the next phase begins — for
example, parallel simulation (all timestep calculations must finish before advancing
to the next timestep), parallel sorting phases, or rendering stages in a game engine.

A `CyclicBarrier` can be **broken** if any waiting thread is interrupted or if the
`await()` times out. When broken, all threads currently at `await()` throw
`BrokenBarrierException`, and subsequent calls also throw it until the barrier is
reset with `barrier.reset()`. Always handle `BrokenBarrierException`.

---

## 🚦 `Semaphore` — Limit Concurrent Access to a Resource

A `Semaphore` maintains a set of **permits**. `acquire()` takes a permit (blocking if
none are available), and `release()` returns one. This is the classic tool for
limiting the number of threads that can access a resource simultaneously — a
connection pool, rate limiter, or access-controlled file system.

A `Semaphore(1)` behaves like a mutex (exactly one thread at a time), but unlike a
`ReentrantLock`, it is **non-reentrant** and can be released by a *different* thread
than the one that acquired it. This makes it useful for producer-consumer signalling.

```java
import java.util.concurrent.*;

// Limit database connections: only 5 threads can query the DB simultaneously
public class ConnectionPool {
    private static final int MAX_CONNECTIONS = 5;
    private final Semaphore semaphore = new Semaphore(MAX_CONNECTIONS, /* fair= */ true);

    public void executeQuery(String query, int threadId) throws InterruptedException {
        System.out.printf("[Thread-%d] Waiting for DB connection... (available: %d)%n",
            threadId, semaphore.availablePermits());

        semaphore.acquire(); // block until a permit is available
        try {
            System.out.printf("[Thread-%d] Connected! Executing: %s%n", threadId, query);
            Thread.sleep(200); // simulate query time
            System.out.printf("[Thread-%d] Done with query.%n", threadId);
        } finally {
            semaphore.release(); // ALWAYS release in finally, just like a lock
            System.out.printf("[Thread-%d] Connection released. (available: %d)%n",
                threadId, semaphore.availablePermits());
        }
    }

    public static void main(String[] args) throws InterruptedException {
        ConnectionPool pool = new ConnectionPool();
        ExecutorService executor = Executors.newFixedThreadPool(20); // 20 threads

        // 20 threads competing for only 5 DB connections
        for (int i = 1; i <= 20; i++) {
            final int id = i;
            executor.submit(() -> {
                try {
                    pool.executeQuery("SELECT * FROM orders WHERE id = " + id, id);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }

        executor.shutdown();
        executor.awaitTermination(30, TimeUnit.SECONDS);
    }
}
```

The `fair` parameter (set to `true` above) ensures that threads acquire permits in
the order they requested them, preventing starvation. Unfair semaphores have higher
throughput but can allow some threads to wait much longer than others.

`Semaphore` also has `tryAcquire()` (non-blocking), `tryAcquire(timeout, unit)`
(with timeout), and `acquireUninterruptibly()` (ignores interrupts) — the same
flexibility pattern you've seen with `ReentrantLock`.

---

## 🎯 `Phaser` — Flexible, Reusable Multi-Phase Barrier

`Phaser` is the most flexible of the synchronizers. It generalises both
`CountDownLatch` and `CyclicBarrier`: the number of registered parties can change
dynamically, and you can advance through multiple named phases. It is ideal for
complex coordination scenarios where tasks may join or leave the party dynamically.

```java
import java.util.concurrent.*;

public class PhaserDemo {

    public static void main(String[] args) throws InterruptedException {
        Phaser phaser = new Phaser(1); // 1 = the main thread ("overseer")

        // Register and launch 3 tasks
        for (int i = 1; i <= 3; i++) {
            phaser.register(); // register a new party dynamically
            final int id = i;
            new Thread(() -> {
                System.out.printf("[Task-%d] Phase 0: Initialising...%n", id);
                sleep(id * 100);

                // Arrive at end of phase 0 and wait for all others
                phaser.arriveAndAwaitAdvance(); // phase 0 → phase 1

                System.out.printf("[Task-%d] Phase 1: Processing...%n", id);
                sleep(id * 100);

                phaser.arriveAndAwaitAdvance(); // phase 1 → phase 2

                System.out.printf("[Task-%d] Phase 2: Cleaning up...%n", id);
                sleep(50);

                phaser.arriveAndDeregister(); // done — remove from the party
            }, "task-" + id).start();
        }

        // Main thread oversees each phase
        System.out.println("[Main] Waiting for phase 0 to complete...");
        phaser.arriveAndAwaitAdvance(); // wait for all phase 0
        System.out.println("[Main] Phase 0 done. Starting phase 1...");

        phaser.arriveAndAwaitAdvance(); // wait for all phase 1
        System.out.println("[Main] Phase 1 done. All tasks now in cleanup.");

        phaser.arriveAndDeregister(); // main deregisters
        phaser.awaitAdvance(phaser.getPhase()); // wait for phase 2 to finish
        System.out.println("[Main] All phases complete!");
    }

    static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

---

## 🤝 `Exchanger` — Two Threads Swap Data at a Meeting Point

`Exchanger<V>` allows exactly two threads to swap objects at a synchronisation point.
When the first thread calls `exchange(myData)`, it blocks until the second thread also
calls `exchange(itsData)`. At that moment, each thread gets the other's data and both
proceed. This is the cleanest mechanism for a two-thread handoff pattern.

```java
import java.util.concurrent.*;

public class ExchangerDemo {

    public static void main(String[] args) throws InterruptedException {
        Exchanger<String> exchanger = new Exchanger<>();

        // Thread A: the producer
        Thread producer = new Thread(() -> {
            try {
                String produced = "DataBuffer-A";
                System.out.println("[Producer] Produced: " + produced);
                String received = exchanger.exchange(produced); // blocks until consumer arrives
                System.out.println("[Producer] Received back: " + received);
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        }, "producer");

        // Thread B: the consumer
        Thread consumer = new Thread(() -> {
            try {
                String empty = "EmptyBuffer-B";
                System.out.println("[Consumer] Offering: " + empty);
                String received = exchanger.exchange(empty); // blocks until producer arrives
                System.out.println("[Consumer] Received: " + received);
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        }, "consumer");

        producer.start();
        Thread.sleep(100);
        consumer.start();

        producer.join(); consumer.join();
    }
}
```

`Exchanger` is well-suited for double-buffer designs in data pipelines: one thread
fills a buffer while another processes the previous buffer, then they swap — the
filler gets the empty buffer and the processor gets the newly-filled one.

---

## 🧵 PART 2 — Classic Concurrency Patterns

### Pattern 1: Producer-Consumer

The fundamental concurrency pattern. One set of threads produces items; another set
consumes them. A `BlockingQueue` handles all the synchronisation — producers block
when full, consumers block when empty. With `BlockingQueue`, the naive
`wait()`/`notifyAll()` implementation from earlier is completely replaced.

```java
import java.util.concurrent.*;

public class ProducerConsumerPattern {

    record Task(int id, String description) {}

    static class Producer implements Runnable {
        private final BlockingQueue<Task> queue;
        private final int producerId;

        Producer(BlockingQueue<Task> queue, int id) {
            this.queue = queue; this.producerId = id;
        }

        @Override
        public void run() {
            for (int i = 1; i <= 10; i++) {
                Task task = new Task(i, "Task from Producer-" + producerId);
                try {
                    queue.put(task); // blocks if queue is full — back-pressure!
                    System.out.printf("[Producer-%d] Enqueued: %s%n", producerId, task);
                    Thread.sleep(50);
                } catch (InterruptedException e) { Thread.currentThread().interrupt(); return; }
            }
        }
    }

    static class Consumer implements Runnable {
        private final BlockingQueue<Task> queue;
        private final int consumerId;
        private volatile boolean running = true;

        Consumer(BlockingQueue<Task> queue, int id) {
            this.queue = queue; this.consumerId = id;
        }

        public void stop() { this.running = false; }

        @Override
        public void run() {
            while (running || !queue.isEmpty()) {
                try {
                    Task task = queue.poll(500, TimeUnit.MILLISECONDS); // timeout to check 'running'
                    if (task != null) {
                        System.out.printf("[Consumer-%d] Processing: %s%n", consumerId, task);
                        Thread.sleep(100); // simulate work
                    }
                } catch (InterruptedException e) { Thread.currentThread().interrupt(); return; }
            }
            System.out.printf("[Consumer-%d] Shutting down.%n", consumerId);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<Task> queue = new ArrayBlockingQueue<>(20);
        List<Consumer> consumers = List.of(new Consumer(queue, 1), new Consumer(queue, 2));

        ExecutorService executor = Executors.newFixedThreadPool(4);
        executor.submit(new Producer(queue, 1));
        executor.submit(new Producer(queue, 2));
        consumers.forEach(executor::submit);

        executor.shutdown();
        executor.awaitTermination(30, TimeUnit.SECONDS);
        consumers.forEach(Consumer::stop); // signal consumers to drain and exit
    }
}
```

### Pattern 2: `ThreadLocal` — Per-Thread State

`ThreadLocal<T>` provides a variable where each thread has its own independent copy.
Reads and writes to a `ThreadLocal` are completely isolated between threads — no
synchronisation needed because there is no sharing. This is ideal for things like
database connections, transaction contexts, date formatters, or request IDs in a
web server.

```java
import java.text.SimpleDateFormat;

// SimpleDateFormat is NOT thread-safe — using a ThreadLocal gives each thread its own instance
public class DateFormatterPool {
    private static final ThreadLocal<SimpleDateFormat> formatter =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

    public static String format(java.util.Date date) {
        return formatter.get().format(date); // each thread gets its own SDF — no race conditions
    }
}

// Web server pattern: store request-scoped data in ThreadLocal for the duration of a request
public class RequestContext {
    private static final ThreadLocal<String> requestId = new ThreadLocal<>();
    private static final ThreadLocal<String> userId    = new ThreadLocal<>();

    public static void setContext(String reqId, String uid) {
        requestId.set(reqId);
        userId.set(uid);
    }

    public static String getRequestId() { return requestId.get(); }
    public static String getUserId()    { return userId.get(); }

    // CRITICAL: Always remove ThreadLocal values when done!
    // Thread pools reuse threads — if you don't clean up, the next request
    // on the same thread will see the previous request's data.
    public static void clearContext() {
        requestId.remove();
        userId.remove();
    }
}

// In a request handler:
// try {
//     RequestContext.setContext(requestId, userId);
//     handleRequest();
// } finally {
//     RequestContext.clearContext(); // ALWAYS clean up in a finally block
// }
```

The most dangerous `ThreadLocal` mistake is forgetting to call `remove()` in thread
pool environments. Since thread pools reuse threads, a thread that handled Request A
might later handle Request B. If Request A's handler set a `ThreadLocal` value and
didn't clean up, Request B will unexpectedly see Request A's data. Always clean up in
a `finally` block.

### Pattern 3: Immutability — The Safest Concurrency Strategy

An **immutable object** is one whose state cannot change after construction. Immutable
objects can be shared between any number of threads with no synchronisation, no locks,
and no risk of corruption, because there is nothing to corrupt. This is not just a
nice-to-have — it is the single most powerful concurrency strategy available.

```java
// A properly immutable class
public final class ImmutablePoint {  // final: prevents subclassing that could break immutability
    private final double x;  // final: guaranteed not to change after construction
    private final double y;

    public ImmutablePoint(double x, double y) { this.x = x; this.y = y; }

    // Accessors — no setters
    public double getX() { return x; }
    public double getY() { return y; }

    // Transformations return NEW objects — the original is unchanged
    public ImmutablePoint translate(double dx, double dy) {
        return new ImmutablePoint(x + dx, y + dy);
    }

    public double distanceTo(ImmutablePoint other) {
        double dx = this.x - other.x, dy = this.y - other.y;
        return Math.sqrt(dx*dx + dy*dy);
    }
}

// Java Records are immutable by default — the most concise way to create value objects
record Money(BigDecimal amount, String currency) {
    Money add(Money other) {
        if (!this.currency.equals(other.currency))
            throw new IllegalArgumentException("Cannot add different currencies");
        return new Money(this.amount.add(other.amount), this.currency);
    }
}

// All instances of ImmutablePoint and Money can be freely shared between threads
```

Design for immutability where possible. When you need a mutable shared object,
document its thread-safety contract explicitly. The `@Immutable` and `@ThreadSafe`
annotations from the `net.jcip.annotations` library (or equivalent custom ones)
communicate intent clearly to other developers.

### Pattern 4: Read-Write Splitting with Volatile

For a simple pattern — one writer, many readers — you can sometimes use `volatile`
without any locks. The single writer ensures atomicity (no concurrent writes), and
`volatile` provides visibility.

```java
// Configuration holder: updated rarely, read constantly
public class FeatureFlags {
    // volatile ensures all threads always see the latest reference
    private volatile Map<String, Boolean> flags = Map.of("featureX", false, "featureY", true);

    // Only called by one admin/config thread at a time
    public void updateFlags(Map<String, Boolean> newFlags) {
        // Publish a new immutable map atomically (just a reference assignment)
        this.flags = Map.copyOf(newFlags);
    }

    // Called by thousands of request-handling threads concurrently — no synchronisation needed
    public boolean isEnabled(String feature) {
        return flags.getOrDefault(feature, false);
    }
}
```

---

## 📊 Synchronizer Comparison Reference

| Tool | Initialisation | Waiting call | Signalling call | Resettable | Use Case |
|------|----------------|--------------|-----------------|------------|----------|
| `CountDownLatch` | count N | `await()` | `countDown()` | No | Wait for N events once |
| `CyclicBarrier` | party count N | `await()` | `await()` | Yes (auto) | All threads rendezvous repeatedly |
| `Semaphore` | permit count N | `acquire()` | `release()` | N/A | Limit concurrent access |
| `Phaser` | initial parties | `arriveAndAwaitAdvance()` | `arriveAndDeregister()` | Yes | Dynamic multi-phase coordination |
| `Exchanger` | none | `exchange(v)` | `exchange(v)` | N/A | Two-thread data swap |

---

## ⚠️ Best Practices

**Always call `countDown()`/`semaphore.release()` in a `finally` block.** If a thread
in a barrier-based pattern throws an exception and doesn't countdown/release, all
waiting threads will hang forever.

**Handle `BrokenBarrierException` in `CyclicBarrier`.** When any participant fails,
the barrier breaks and all waiting threads receive this exception. Have a clear policy
for what to do when a barrier breaks — usually it means the entire computation must
be retried from scratch.

**Always remove `ThreadLocal` values in thread pool threads.** Call `threadLocal.remove()`
in a `finally` block at the end of your task. Failure to do this causes subtle data
leakage between logically separate tasks that happen to reuse the same thread.

**Design data as immutable by default.** Make fields `final`, classes `final` where
appropriate, return defensive copies from getters, and use Java's built-in immutable
collections (`List.of()`, `Map.of()`). Mutability should be a conscious choice, not a
default.

**Use `Semaphore` for resource pools rather than a custom synchronised counter.** The
`Semaphore` abstracts all the wait/notify logic correctly and handles fairness,
interruption, and timeouts cleanly out of the box.

---

## 🔑 Key Takeaways

`CountDownLatch` is a one-shot gate: N things must happen before waiting threads are
released. It cannot be reset. `CyclicBarrier` is a repeating meeting point: N threads
must all arrive before any are released, and it resets automatically. `Semaphore`
limits the number of threads in a section simultaneously — think of it as a pool of
N tickets. `Phaser` is the most flexible: supports dynamic registration, multiple
phases, and arbitrary coordination logic. `Exchanger` is specifically for two threads
to swap data at a synchronisation point.

The Producer-Consumer pattern is the foundation of most real concurrent systems —
always implement it using a `BlockingQueue`. `ThreadLocal` gives each thread its own
copy of a variable — great for non-thread-safe resources like `SimpleDateFormat`, but
always clean up in thread-pool environments. Immutability is the most powerful
concurrency technique of all — if state cannot change, it cannot be corrupted.
