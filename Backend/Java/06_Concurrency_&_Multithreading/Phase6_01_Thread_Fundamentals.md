# 🔀 Phase 6.1 — Thread Fundamentals in Java
### The Building Blocks: Processes, Threads, and Lifecycle

---

## 🧠 Process vs. Thread — What's the Real Difference?

Before writing a single line of concurrent Java, you need a clear mental model of
what a thread *is*. A **process** is an isolated program in execution — it has its own
memory space, file handles, and system resources. When you run a Java program, the OS
creates a process for it.

A **thread** is a *lightweight unit of execution inside a process*. A single process
can contain multiple threads, and all those threads **share the same heap memory**.
This sharing is what makes threads both powerful (they can communicate easily) and
dangerous (they can corrupt each other's data).

```
Process (JVM)
├── Heap Memory (SHARED by all threads)
│   ├── Objects, static fields, class data
│
├── Thread 1 ──► Stack (PRIVATE to Thread 1) + PC Register
├── Thread 2 ──► Stack (PRIVATE to Thread 2) + PC Register
└── Thread 3 ──► Stack (PRIVATE to Thread 3) + PC Register
```

Each thread has its own **call stack** (local variables, method call frames) and its
own **program counter** (which instruction it's currently executing). The heap,
however, is shared — which is why concurrent modification of shared objects requires
careful synchronization.

---

## 🏗️ Creating Threads — Three Ways

### Way 1: Extend the `Thread` Class

The most direct way. You subclass `Thread` and override its `run()` method, which
contains the code you want to execute concurrently.

```java
public class MyThread extends Thread {

    private final String taskName;

    public MyThread(String taskName) {
        // super(taskName) also sets the thread's name — useful for debugging
        super(taskName);
        this.taskName = taskName;
    }

    @Override
    public void run() {
        // This code runs in the new thread
        for (int i = 1; i <= 5; i++) {
            System.out.printf("[%s] Step %d%n", taskName, i);
            try {
                Thread.sleep(100); // pause 100ms to simulate work
            } catch (InterruptedException e) {
                // ALWAYS restore the interrupt flag when catching InterruptedException
                Thread.currentThread().interrupt();
                System.out.println(taskName + " was interrupted");
                return; // exit the task cleanly
            }
        }
    }
}

public class ThreadCreationDemo {
    public static void main(String[] args) throws InterruptedException {
        MyThread t1 = new MyThread("Worker-A");
        MyThread t2 = new MyThread("Worker-B");

        t1.start(); // start() creates the new OS thread and calls run() on it
        t2.start(); // NEVER call run() directly — that runs on the current thread!

        t1.join(); // main thread waits until t1 finishes
        t2.join(); // main thread waits until t2 finishes
        System.out.println("Both workers done.");
    }
}
```

The most critical mistake beginners make here is calling `thread.run()` instead of
`thread.start()`. Calling `run()` directly just executes the method on the *current*
thread — no new thread is created, no concurrency happens. Always call `start()`.

### Way 2: Implement the `Runnable` Interface (Preferred)

`Runnable` is a functional interface with a single `run()` method. Implementing it
rather than extending `Thread` is almost always the better design choice, because
Java supports single inheritance — if your class extends `Thread`, it can't extend
anything else. Using `Runnable` keeps your task logic separate from the threading
mechanism (separation of concerns).

```java
public class PrintTask implements Runnable {

    private final String message;
    private final int repetitions;

    public PrintTask(String message, int repetitions) {
        this.message = message;
        this.repetitions = repetitions;
    }

    @Override
    public void run() {
        for (int i = 0; i < repetitions; i++) {
            System.out.printf("[%s] %s (#%d)%n",
                Thread.currentThread().getName(), message, i + 1);
        }
    }
}

// Using Runnable with Thread
Runnable task = new PrintTask("Hello", 3);
Thread t = new Thread(task, "my-thread");
t.start();

// Using Runnable as a lambda (Java 8+) — by far the most common form today
Thread t2 = new Thread(() -> System.out.println("Quick task"), "quick-thread");
t2.start();
```

### Way 3: Using `Callable` and `Future` (for Return Values)

`Runnable` cannot return a value or throw checked exceptions. When you need either
of those, use `Callable<V>`, which returns a `Future<V>`. This is covered in depth
in the Executor Framework file, but the basic structure is worth seeing here.

```java
import java.util.concurrent.*;

Callable<Integer> computation = () -> {
    Thread.sleep(500); // simulate a slow computation
    return 42;         // return a result
};

ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(computation);

// Do other work while the computation runs...

Integer result = future.get(); // blocks until the result is ready
System.out.println("Result: " + result); // 42
executor.shutdown();
```

---

## 🔄 Thread Lifecycle — The Six States

Every Java thread is always in exactly one of six states, defined in the
`Thread.State` enum. Understanding these states is essential for debugging
concurrency issues and reading thread dumps.

```
NEW ──► RUNNABLE ──► TERMINATED
              │
              ├──► BLOCKED         (waiting to acquire a monitor lock)
              ├──► WAITING         (indefinitely waiting for notification)
              └──► TIMED_WAITING   (waiting with a timeout)
              
  BLOCKED/WAITING/TIMED_WAITING all transition back to RUNNABLE
  when their condition is met.
```

**NEW** — The thread object has been created (`new Thread(...)`) but `start()` has
not yet been called. The OS thread doesn't exist yet.

**RUNNABLE** — The thread has been started and is either actively running on a CPU
core, or is ready to run and waiting for the OS scheduler to assign it a core.
From Java's perspective there is no distinction between "running" and "ready to run."

**BLOCKED** — The thread is trying to enter a `synchronized` block or method but
another thread currently holds the monitor lock. The thread is stuck until that lock
is released.

**WAITING** — The thread is indefinitely waiting for another thread to perform a
specific action. This happens when you call `Object.wait()`, `Thread.join()` with no
timeout, or `LockSupport.park()`. The thread will not run again until explicitly
notified.

**TIMED_WAITING** — Like WAITING, but with a timeout. Caused by `Thread.sleep(n)`,
`Object.wait(n)`, `Thread.join(n)`, or `LockSupport.parkNanos(n)`. The thread
automatically returns to RUNNABLE when the timeout expires.

**TERMINATED** — The thread's `run()` method has finished (either normally or by
throwing an uncaught exception). A terminated thread cannot be restarted.

```java
// Demonstrating thread states
public class ThreadStateDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            try { Thread.sleep(2000); }
            catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        }, "demo-thread");

        System.out.println("Before start: " + t.getState()); // NEW

        t.start();
        Thread.sleep(50); // give it a moment to start sleeping
        System.out.println("While sleeping: " + t.getState()); // TIMED_WAITING

        t.join();
        System.out.println("After finish: " + t.getState()); // TERMINATED
    }
}
```

---

## ⚙️ Key Thread Methods

### `start()` vs `run()`

This deserves repeating: `start()` creates a new OS thread and schedules `run()` for
execution on it. `run()` called directly is just a regular method call on the
*calling* thread. Never call `run()` to start concurrency.

### `sleep(long millis)` — Pause Without Releasing Locks

`Thread.sleep()` pauses the current thread for at least the specified number of
milliseconds. Unlike `wait()`, `sleep()` does **not** release any monitor locks the
thread holds — this is a critical distinction that trips up many developers.

```java
// Thread.sleep() is a static method — it always affects the CURRENT thread
try {
    System.out.println("Going to sleep...");
    Thread.sleep(1000); // sleep 1 second
    System.out.println("Woke up!");
} catch (InterruptedException e) {
    // Always handle InterruptedException — see below
    Thread.currentThread().interrupt(); // restore interrupt flag
}
```

### `join()` — Wait for a Thread to Finish

`join()` causes the *calling* thread to pause until the *target* thread terminates.
It is the primary way to wait for another thread's work to complete before
proceeding.

```java
Thread worker = new Thread(() -> {
    System.out.println("Worker: doing heavy lifting...");
    try { Thread.sleep(2000); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    System.out.println("Worker: done.");
});

worker.start();
System.out.println("Main: waiting for worker...");
worker.join();   // main thread blocks here until worker finishes
System.out.println("Main: worker has finished, continuing.");

// join with timeout — don't wait forever
worker.join(5000); // wait at most 5 seconds
if (worker.isAlive()) {
    System.out.println("Worker took too long!");
}
```

### `interrupt()` — Requesting Cooperative Cancellation

Interruption in Java is **cooperative** — one thread can request that another thread
stop, but the target thread must check for and respect the request. Calling
`interrupt()` on a thread does two things: if the thread is sleeping or waiting,
it throws `InterruptedException` immediately; if the thread is running, it sets the
thread's *interrupt flag* to `true`.

```java
Thread longTask = new Thread(() -> {
    System.out.println("Task started");
    while (!Thread.currentThread().isInterrupted()) { // check the flag in a loop
        // ... do a unit of work ...
        System.out.println("Working...");
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            // sleep threw because of an interrupt — restore the flag and exit
            Thread.currentThread().interrupt(); // RE-SET the flag (sleep clears it)
            System.out.println("Interrupted during sleep, exiting cleanly.");
            return;
        }
    }
    System.out.println("Exiting because interrupt flag was set.");
});

longTask.start();
Thread.sleep(1500);
longTask.interrupt(); // ask the task to stop
longTask.join();
System.out.println("Task has terminated.");
```

The golden rule of `InterruptedException`: **never swallow it silently** (never catch
it and do nothing). Either re-throw it, or restore the interrupt flag with
`Thread.currentThread().interrupt()` before returning. If you swallow it, the thread
machinery that depends on the flag (like `ExecutorService` shutdown) will break.

### `isAlive()`, `getState()`, `getName()`, `getId()`

```java
Thread t = new Thread(() -> { /* ... */ }, "my-worker");
System.out.println(t.getName());    // "my-worker"
System.out.println(t.getId());      // some unique long ID
System.out.println(t.isAlive());    // false (not started yet)
t.start();
System.out.println(t.isAlive());    // true (running)
System.out.println(t.getState());   // RUNNABLE (or TIMED_WAITING if it called sleep)
```

---

## 🔱 Thread Priority

Every thread has a priority between `Thread.MIN_PRIORITY` (1) and
`Thread.MAX_PRIORITY` (10), with `Thread.NORM_PRIORITY` (5) as the default. The OS
scheduler *hints* use priority when deciding which thread to run, but priorities are
**not a guarantee** — the actual behaviour is platform-dependent and can be completely
ignored by some OS schedulers.

```java
Thread highPriority = new Thread(() -> { /* ... */ });
highPriority.setPriority(Thread.MAX_PRIORITY); // hint: prefer this thread

Thread lowPriority = new Thread(() -> { /* ... */ });
lowPriority.setPriority(Thread.MIN_PRIORITY);
```

In practice, you should design your programs so they are correct regardless of
scheduling order, and avoid relying on priorities to control execution sequence.

---

## 👻 Daemon Threads

A **daemon thread** is a background thread that exists to serve user (non-daemon)
threads. The JVM exits when all user threads have finished — it does *not* wait for
daemon threads to complete. Daemon threads are automatically killed when the JVM shuts
down, which means their `finally` blocks and shutdown logic may not run.

```java
Thread heartbeat = new Thread(() -> {
    while (true) { // runs "forever"
        System.out.println("Heartbeat ping...");
        try { Thread.sleep(1000); }
        catch (InterruptedException e) { return; }
    }
});

// Must be set BEFORE start() is called
heartbeat.setDaemon(true);
heartbeat.start();

// When the main thread ends, the JVM exits and kills the daemon thread too
Thread.sleep(3000);
System.out.println("Main thread done — JVM will exit, taking the daemon with it.");
```

Use daemon threads for background housekeeping tasks — log flushing, cache
eviction, GC-assist threads — that don't need to complete their work when the
application exits. Never use daemon threads for tasks that write data to databases
or files, because the write may be abruptly terminated.

---

## 👥 `ThreadGroup`

`ThreadGroup` is a legacy mechanism that groups threads and daemons together into a
tree structure, allowing bulk operations like `interrupt()` on the whole group.
In modern Java, it is largely superseded by `ExecutorService` and structured
concurrency. You will occasionally encounter it in older code.

```java
ThreadGroup group = new ThreadGroup("workers");
Thread t1 = new Thread(group, () -> { /* ... */ }, "worker-1");
Thread t2 = new Thread(group, () -> { /* ... */ }, "worker-2");
t1.start();
t2.start();

group.interrupt(); // interrupts all threads in the group

System.out.println("Active threads in group: " + group.activeCount());
```

---

## 🏷️ Thread Naming — An Underappreciated Practice

Always name your threads and thread pools with meaningful names. When a production
incident occurs and you take a thread dump, you will be looking at hundreds of
`Thread-0`, `Thread-1`, `Thread-2` entries — or, with good naming, at
`order-processor-1`, `db-writer-2`, `event-consumer-3`. The latter is infinitely
more useful for diagnosis.

```java
// Name via Thread constructor
Thread t = new Thread(() -> { /* ... */ }, "order-processor-1");

// Name via ThreadFactory (for thread pools — covered in Executor file)
ThreadFactory namedFactory = r -> {
    Thread thread = new Thread(r);
    thread.setName("my-pool-thread-" + thread.getId());
    return thread;
};
ExecutorService pool = Executors.newFixedThreadPool(4, namedFactory);
```

---

## 🧪 Complete Working Example — Parallel Tasks with Join

```java
import java.util.*;
import java.util.concurrent.*;

public class ParallelDownloaderSimulation {

    // Simulates downloading a file — returns the file size
    static long simulateDownload(String url) throws InterruptedException {
        System.out.printf("[%s] Starting download: %s%n",
            Thread.currentThread().getName(), url);
        long size = (long)(Math.random() * 1000) + 100; // random 100–1100 ms
        Thread.sleep(size); // simulate network time
        System.out.printf("[%s] Finished: %s (%d KB)%n",
            Thread.currentThread().getName(), url, size);
        return size;
    }

    public static void main(String[] args) throws InterruptedException {
        List<String> urls = List.of(
            "https://example.com/file1.zip",
            "https://example.com/file2.zip",
            "https://example.com/file3.zip"
        );

        long startTime = System.currentTimeMillis();
        List<Thread> threads = new ArrayList<>();

        // Start a thread for each download
        for (String url : urls) {
            Thread t = new Thread(() -> {
                try {
                    simulateDownload(url);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }, "downloader-" + urls.indexOf(url));
            threads.add(t);
            t.start();
        }

        // Wait for ALL downloads to finish
        for (Thread t : threads) {
            t.join();
        }

        long elapsed = System.currentTimeMillis() - startTime;
        System.out.printf("All downloads done in %d ms (parallel!)%n", elapsed);
        // This will be ~max(individual times) rather than sum(individual times)
    }
}
```

---

## ⚠️ Best Practices

**Always name your threads** — it costs nothing and saves hours during debugging. Pass
the name as the second argument to the `Thread` constructor.

**Always handle `InterruptedException` correctly** — either re-throw it or call
`Thread.currentThread().interrupt()` to restore the flag. Never silently swallow it
with an empty `catch` block.

**Prefer `Runnable` over extending `Thread`** — it separates your task logic from
the threading mechanism and doesn't waste your one inheritance slot.

**Use `ExecutorService` instead of raw threads** in most production code. Raw `Thread`
objects are the assembly language of concurrency — you should rarely use them directly
in application code, just as you rarely write raw assembly. The Executor framework
gives you thread pooling, lifecycle management, and much better error handling.

**Don't rely on thread priorities** for correctness — your program must behave
correctly regardless of scheduling order.

**Don't use daemon threads for I/O** — any thread that commits data to a database,
writes to a file, or sends network messages should be a user thread.

---

## 🔑 Key Takeaways

- A thread is a unit of execution that shares heap memory with other threads in the same process but has its own stack and program counter.
- Create threads by implementing `Runnable` (preferred) or extending `Thread`. Use `Callable` when you need a return value.
- Always call `start()` to create a new thread — calling `run()` directly is just a regular method call.
- The six thread states are NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, and TERMINATED.
- `sleep()` pauses the current thread but **does not release locks**; `wait()` pauses and **does release** the monitor lock.
- `join()` makes the calling thread wait for the target thread to finish.
- Handle `InterruptedException` properly — always restore the interrupt flag or re-throw.
- Always name your threads — thread dumps become readable instead of cryptic.
- Daemon threads are killed when the JVM exits — never use them for tasks that must complete.
