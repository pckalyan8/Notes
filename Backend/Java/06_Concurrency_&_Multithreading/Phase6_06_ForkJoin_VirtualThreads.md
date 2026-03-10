# 🔀 Phase 6.6 — Fork-Join Framework & Virtual Threads (Project Loom)
### Recursive Parallelism, Work-Stealing, and the Future of Concurrency

---

## 🏗️ PART A — The Fork-Join Framework

### What Problem Does It Solve?

Imagine you need to find the maximum value in an array of 100 million numbers.
With a regular thread pool, you'd either do it on a single thread (slow) or manually
split the array into chunks, submit each chunk as a task, and reassemble the results
yourself (complex and verbose). The Fork-Join framework was designed exactly for this
kind of **divide-and-conquer** problem: recursively split the work in half until the
chunks are small enough to solve directly, then combine the results back up the tree.

The key innovation is the **work-stealing** algorithm. When a thread finishes its
task and its queue is empty, it *steals* work from the tail of another thread's queue
rather than sitting idle. This keeps all CPU cores busy with minimal coordination
overhead, and it's why parallel streams (`list.parallelStream()`) are fast — they
use the Fork-Join common pool under the hood.

```
Task(0..1M)
├── Task(0..500K)
│   ├── Task(0..250K)  ──► thread 1 solves directly
│   └── Task(250..500K) ──► thread 2 solves directly
└── Task(500K..1M)
    ├── Task(500K..750K) ──► thread 3 solves directly
    └── Task(750K..1M)   ──► thread 4 steals this and solves it
                              ↑ work-stealing in action
```

### `RecursiveTask<V>` — Computes a Value

Extend `RecursiveTask<V>` when your computation returns a result. Override `compute()`
to contain the divide-and-conquer logic: if the problem is small enough (the
*threshold*), solve it directly; otherwise, split it, fork one half, compute the
other half in the current thread, join the forked half, and combine.

```java
import java.util.concurrent.*;

public class ParallelMaxFinder extends RecursiveTask<Long> {
    private static final int THRESHOLD = 10_000; // solve directly if <= this many elements
    private final long[] array;
    private final int start, end;

    public ParallelMaxFinder(long[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        int size = end - start;

        // BASE CASE: small enough to solve directly (sequential)
        if (size <= THRESHOLD) {
            long max = Long.MIN_VALUE;
            for (int i = start; i < end; i++) {
                max = Math.max(max, array[i]);
            }
            return max;
        }

        // RECURSIVE CASE: split in half
        int mid = start + size / 2;

        ParallelMaxFinder leftTask  = new ParallelMaxFinder(array, start, mid);
        ParallelMaxFinder rightTask = new ParallelMaxFinder(array, mid, end);

        // fork() submits leftTask to the pool (runs asynchronously)
        leftTask.fork();

        // compute() on rightTask runs in the CURRENT thread (avoids unnecessary thread creation)
        long rightMax = rightTask.compute();

        // join() blocks until the forked task is done, then returns its result
        long leftMax = leftTask.join();

        return Math.max(leftMax, rightMax);
    }

    public static void main(String[] args) {
        long[] data = new long[10_000_000];
        for (int i = 0; i < data.length; i++) data[i] = (long)(Math.random() * 1_000_000);

        ForkJoinPool pool = new ForkJoinPool(); // uses available processors by default
        long max = pool.invoke(new ParallelMaxFinder(data, 0, data.length));
        System.out.println("Maximum value: " + max);
        pool.shutdown();
    }
}
```

The pattern `task.fork(); thisTask.compute(); task.join()` is the canonical
Fork-Join idiom. Always fork the first subtask and *directly compute* the second in
the current thread. This is important: if you fork both, you create a task and then
immediately block waiting for it, burning a thread for no reason.

### `RecursiveAction` — No Return Value

Extend `RecursiveAction` when your task performs work but doesn't need to return a
value — sorting in-place, parallel array initialisation, or file processing.

```java
public class ParallelArrayFill extends RecursiveAction {
    private static final int THRESHOLD = 5_000;
    private final int[] array;
    private final int start, end;
    private final int value;

    public ParallelArrayFill(int[] array, int start, int end, int value) {
        this.array = array; this.start = start;
        this.end = end; this.value = value;
    }

    @Override
    protected void compute() {
        if (end - start <= THRESHOLD) {
            Arrays.fill(array, start, end, value); // base case: fill directly
            return;
        }
        int mid = start + (end - start) / 2;
        invokeAll( // invokeAll forks both AND joins both — convenient for 2+ subtasks
            new ParallelArrayFill(array, start, mid, value),
            new ParallelArrayFill(array, mid, end, value)
        );
    }
}
```

### `ForkJoinPool.commonPool()` — The Shared Pool

Java provides a single JVM-wide `ForkJoinPool.commonPool()` that is used by parallel
streams and by `RecursiveTask`/`RecursiveAction` when you use `compute()` without
explicit pool submission. Its parallelism level is `Runtime.getRuntime().availableProcessors() - 1`.

```java
// These all use ForkJoinPool.commonPool() under the hood:
list.parallelStream().map(...).collect(...);
ForkJoinTask.invoke(myTask); // invoked outside a pool — uses commonPool
Arrays.parallelSort(array);
```

For application code, create your own `ForkJoinPool` rather than relying on the
common pool. This prevents your CPU-intensive work from interfering with other users
of the common pool, and lets you control parallelism.

```java
// Custom pool with explicit parallelism level
ForkJoinPool myPool = new ForkJoinPool(
    4, // parallelism: max 4 worker threads
    ForkJoinPool.defaultForkJoinWorkerThreadFactory,
    null, // uncaught exception handler
    false // asyncMode: false = LIFO (depth-first, better for divide-and-conquer)
);
long result = myPool.invoke(new ParallelMaxFinder(data, 0, data.length));
myPool.shutdown();
```

### Choosing the Right Threshold

The threshold — the point at which you stop recursing and solve directly — is a
critical tuning parameter. Too small, and you create thousands of tiny tasks with
more overhead than benefit. Too large, and you don't split enough to keep all cores
busy. A reasonable starting point is: tasks that take between 100 microseconds and
10 milliseconds of CPU time each. Profile and measure — there is no universal formula.

### When to Use Fork-Join

Fork-Join excels at **CPU-bound, divide-and-conquer** problems over large data
structures: parallel sorting, parallel tree traversal, parallel reduction, parallel
matrix operations. It is designed for computational work, not I/O. If tasks spend
time waiting for network or disk, their threads block, work-stealing breaks down, and
a regular fixed thread pool is a better choice.

---

## 🌟 PART B — Virtual Threads (Java 21 — Project Loom)

### The Platform Thread Problem

Every Java thread you've worked with so far is a **platform thread** — a direct
wrapper around an OS thread. Platform threads are heavy: each one allocates a stack
of 512 KB to 1 MB, and modern operating systems can typically only handle tens of
thousands of them before performance degrades severely. On a 16-core server running
a typical web application, you might have 200–500 platform threads. If each thread
spends 95% of its time waiting for a database response, you are blocking 200–500
OS threads, wasting enormous resources doing nothing.

The standard response to this in the industry has been *reactive programming* —
libraries like RxJava and Project Reactor that replace blocking calls with callback
chains. Reactive code is non-blocking and very efficient, but it comes at the cost of
extreme code complexity. A simple sequential `"fetch user → fetch orders → save result"`
becomes a chain of callbacks, operators, and error handlers that are hard to read,
hard to debug, and hard to reason about.

**Project Loom (Java 21) solves this with Virtual Threads.** A virtual thread is an
extremely lightweight thread managed by the JVM, not the OS. It costs approximately
1 KB of memory (versus 1 MB for a platform thread). When a virtual thread blocks on
an I/O operation, it is **unmounted** from its carrier platform thread (the OS thread
returns to a pool immediately), and when the I/O completes, the virtual thread is
remounted onto any available carrier thread to continue. This happens transparently —
your code looks sequential and blocking, but underneath, threads are never actually
idle.

```
Platform thread model:
  Request 1 ──► Platform Thread A ──► [waiting for DB 95% of the time] ──► respond
  Request 2 ──► Platform Thread B ──► [waiting for DB 95% of the time] ──► respond
  ...max ~500 concurrent requests on a typical server...

Virtual thread model:
  Request 1 ──► Virtual Thread 1 ──► blocks on DB ──► Platform Thread A gets freed
                                   ←── Platform Thread A picks it back up when DB responds
  Request 2 ──► Virtual Thread 2 ──► blocks on DB ──► Platform Thread A handles another
  ...millions of concurrent requests on the same server...
```

### Creating Virtual Threads

```java
// Method 1: Thread.ofVirtual()
Thread vt = Thread.ofVirtual()
    .name("my-virtual-thread")
    .start(() -> System.out.println("Hello from virtual thread!"));
vt.join();

// Method 2: Virtual thread factory
ThreadFactory virtualFactory = Thread.ofVirtual().factory();
Thread vt2 = virtualFactory.newThread(() -> System.out.println("Another virtual thread"));
vt2.start();

// Method 3: Executor (the preferred production approach)
// Creates one new virtual thread per submitted task — unlike platform thread pools,
// virtual threads are so cheap that you don't need to pool them!
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10_000; i++) { // can easily handle 100,000+ tasks
        int taskId = i;
        executor.submit(() -> {
            // Simulate blocking I/O — virtual thread blocks here,
            // but the underlying platform thread is NOT blocked
            Thread.sleep(1000);
            System.out.printf("Task %d completed on %s%n",
                taskId, Thread.currentThread());
        });
    }
} // try-with-resources: automatically shuts down when the block exits (Java 19+)
```

### The Key Mental Model Shift

With virtual threads, the rules change. You no longer need to avoid blocking calls.
You no longer need to size thread pools carefully. You no longer need reactive
libraries just for concurrency. Write your code in the simplest, most natural way —
sequential, blocking — and the JVM handles the efficiency transparently.

```java
// Old world (reactive): handle concurrency explicitly with non-blocking operators
CompletableFuture<String> result = fetchUser(id)
    .thenCompose(user -> fetchOrders(user))
    .thenCompose(orders -> calculateTotal(orders))
    .thenApply(total -> "Total: " + total);

// New world (virtual threads): write it sequentially — it's just as efficient
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> {
        User user = fetchUser(id);      // blocks, but that's fine now
        List<Order> orders = fetchOrders(user); // blocks again, still fine
        double total = calculateTotal(orders);
        return "Total: " + total;
    });
}
```

### Structured Concurrency (Java 21+) — `StructuredTaskScope`

Structured Concurrency is a companion feature to Virtual Threads that makes
fan-out async operations safe and readable. The key idea is that concurrent tasks
should have a clear parent-child relationship: when the parent scope exits, all child
tasks are either complete or cancelled. This prevents the most common concurrency
bug: a task continuing to run after it should have been cancelled, leaking resources.

```java
import java.util.concurrent.*;

public class StructuredExample {

    String fetchUser(int id) throws InterruptedException {
        Thread.sleep(200); // simulate I/O
        return "Alice";
    }

    List<String> fetchOrders(int userId) throws InterruptedException {
        Thread.sleep(300); // simulate I/O
        return List.of("Order-1", "Order-2");
    }

    // Build a dashboard by fetching user and orders in parallel,
    // with automatic cleanup if either fails
    record Dashboard(String user, List<String> orders) {}

    Dashboard buildDashboard(int userId) throws ExecutionException, InterruptedException {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {

            // Fork two tasks to run in parallel
            StructuredTaskScope.Subtask<String> userTask =
                scope.fork(() -> fetchUser(userId));

            StructuredTaskScope.Subtask<List<String>> ordersTask =
                scope.fork(() -> fetchOrders(userId));

            // Wait for both to complete (or one to fail)
            scope.join();

            // If either task failed, this throws the exception
            scope.throwIfFailed();

            // Both succeeded — combine results
            return new Dashboard(userTask.get(), ordersTask.get());

        } // scope.close() here: if join() is interrupted, all child tasks are cancelled
    }

    public static void main(String[] args) throws Exception {
        var service = new StructuredExample();
        Dashboard dashboard = service.buildDashboard(42);
        System.out.println("User: " + dashboard.user());
        System.out.println("Orders: " + dashboard.orders());
    }
}
```

`ShutdownOnFailure` is the most common scope policy: if any task fails, all
remaining tasks are cancelled and the exception is re-thrown. There is also
`ShutdownOnSuccess`, which cancels all tasks the moment any one succeeds — useful
for the "race" pattern (query multiple sources, use the first response).

### Scoped Values — Replacing `ThreadLocal` for Virtual Threads

`ThreadLocal<T>` is a common way to pass context (like a user session or request ID)
through a call stack without threading it through every method parameter. With virtual
threads, `ThreadLocal` still works but has a performance concern: virtual threads are
so cheap and numerous that the map of `ThreadLocal` values could consume significant
memory. `ScopedValue` (Java 21, preview) is the modern replacement — it binds a
value for a specific scope and its child scopes without the map overhead.

```java
import java.util.concurrent.ScopedValue;

public class RequestContext {
    // Define a scoped value (like a thread-local but better)
    static final ScopedValue<String> REQUEST_ID = ScopedValue.newInstance();
    static final ScopedValue<String> USER_NAME  = ScopedValue.newInstance();

    void handleRequest(String requestId, String userName) {
        // Bind the values for the duration of this scope
        ScopedValue.where(REQUEST_ID, requestId)
                   .where(USER_NAME, userName)
                   .run(() -> {
                       processRequest(); // all code here and in callees can read the values
                   });
    }

    void processRequest() {
        // Read the scoped values anywhere in the call tree
        System.out.printf("[%s] Processing for user %s%n",
            REQUEST_ID.get(), USER_NAME.get());
        callDeepMethod();
    }

    void callDeepMethod() {
        // Still accessible deep in the call stack — no need to pass as parameters
        System.out.println("Deep method, same request: " + REQUEST_ID.get());
    }
}
```

### What Virtual Threads Don't Fix

Virtual threads are not a silver bullet. They shine for **I/O-bound** work — anything
that blocks on network, database, or file operations. They do not improve
**CPU-bound** computation — if your code spends all its time calculating, not waiting,
having more threads doesn't help. For CPU-bound work, the Fork-Join framework and
parallel streams remain the right tools.

There is also the **pinning** problem. When a virtual thread executes code inside a
`synchronized` block or a native method call, it is "pinned" to its carrier platform
thread — the carrier cannot be released even if the virtual thread blocks. This turns
a virtual thread into a platform thread for the duration of the pinned section.
Pinning doesn't cause incorrect behaviour but can cause performance degradation in
high-concurrency scenarios. The fix is to replace `synchronized` with `ReentrantLock`
in the critical sections, which virtual threads can unmount from.

```java
// Pinning problem: synchronized holds the carrier thread
synchronized (lock) {
    callBlockingIO(); // pins the carrier — BAD for virtual threads
}

// Fix: use ReentrantLock instead
lock.lock();
try {
    callBlockingIO(); // virtual thread can unmount here — GOOD
} finally {
    lock.unlock();
}
```

---

## 🧪 Complete Example — Virtual Thread Web Server Simulation

```java
import java.util.concurrent.*;
import java.util.*;

public class VirtualThreadServer {

    // Simulate handling an HTTP request with I/O
    static String handleRequest(int requestId) throws InterruptedException {
        String threadType = Thread.currentThread().isVirtual() ? "virtual" : "platform";
        System.out.printf("[Request-%d] Started on %s thread%n", requestId, threadType);

        Thread.sleep(100); // simulate DB query

        System.out.printf("[Request-%d] DB done, processing...%n", requestId);

        Thread.sleep(50); // simulate computation

        return "Response for request " + requestId;
    }

    public static void main(String[] args) throws InterruptedException {
        int numRequests = 1000; // simulate 1000 concurrent requests
        long start = System.currentTimeMillis();

        // With virtual threads: no thread pool sizing, no tuning needed
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            List<Future<String>> futures = new ArrayList<>();
            for (int i = 1; i <= numRequests; i++) {
                final int id = i;
                futures.add(executor.submit(() -> handleRequest(id)));
            }
            // Collect results
            int success = 0;
            for (Future<String> f : futures) {
                try { f.get(); success++; }
                catch (ExecutionException e) { System.err.println("Request failed: " + e.getCause()); }
            }
            System.out.printf("%d requests completed in %d ms%n",
                success, System.currentTimeMillis() - start);
            // ~150ms total (parallel), not 150ms * 1000 = 150 seconds (sequential)
        }
    }
}
```

---

## ⚠️ Best Practices

**Use virtual threads for I/O-bound tasks, Fork-Join for CPU-bound parallelism.** These
are complementary tools for different problems, not competitors. A modern high-performance
application will often use both.

**Don't pool virtual threads.** Platform threads are expensive to create and you pool
them to reuse them. Virtual threads are cheap enough to create per-task — pooling them
provides no benefit and wastes the simplicity that virtual threads offer.

**Replace `synchronized` with `ReentrantLock` in hot paths when using virtual threads.**
Pinning in `synchronized` blocks prevents virtual threads from unmounting, negating
their efficiency advantage for I/O-heavy code.

**Set the threshold carefully in Fork-Join tasks.** Profile both the overhead of task
creation and the work done in the base case to find the right balance. Start with
chunks that take 100μs–10ms each.

**Use `invokeAll(leftTask, rightTask)` in `RecursiveAction`** when you have exactly
two subtasks of equal importance and don't need intermediate results — it's cleaner
than manual fork/join pairing.

---

## 🔑 Key Takeaways

**Fork-Join Framework** is designed for CPU-bound divide-and-conquer work. Use `RecursiveTask<V>` when your computation returns a value, `RecursiveAction` when it doesn't. The work-stealing algorithm keeps all cores busy. Always fork one branch and compute the other directly in the current thread to avoid wasteful thread creation.

**Virtual Threads** (Java 21) are JVM-managed lightweight threads that unmount from their carrier OS thread when blocked on I/O. They cost ~1 KB each, enabling millions of concurrent operations on a single JVM. Write sequential, blocking code — the JVM handles efficiency transparently.

**Structured Concurrency** with `StructuredTaskScope` ensures that forked tasks have a clear lifecycle: when the scope exits, all subtasks are either complete or cancelled. This prevents resource leaks from orphaned tasks.

**Scoped Values** are the modern replacement for `ThreadLocal` in virtual-thread-heavy code — they carry context through a call tree without the memory overhead of per-thread maps.

**Pinning** is the main gotcha: avoid `synchronized` in hot paths when using virtual threads because it pins the carrier thread. Use `ReentrantLock` instead.
