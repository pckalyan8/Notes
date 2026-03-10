# 🔀 Phase 6.3 — Locks, Atomic Variables & Concurrent Collections
### `java.util.concurrent` — The Professional Concurrency Toolkit (Part 1)

---

## 🧠 Why `java.util.concurrent` Was Created

The `synchronized` keyword and `wait()`/`notify()` are the foundation of Java
concurrency, but they have significant limitations in practice. You cannot try to
acquire a lock and give up if it's busy (no try-lock). You cannot interrupt a thread
that is blocked waiting for a `synchronized` lock. You cannot have multiple readers
accessing data concurrently even when no writer is active. And `wait()`/`notify()` is
notoriously error-prone because condition management is entirely manual.

Java 5 introduced the `java.util.concurrent` (JUC) package, written by Brian Goetz
and Doug Lea, which solved all these problems with a rich library of higher-level
concurrency utilities. Understanding this package is the difference between writing
concurrency code that barely works and writing code that is correct, readable, and
performant.

---

## 🔑 Explicit Locks — The `Lock` Interface

The `Lock` interface provides all the capabilities of `synchronized` plus additional
features. The key implementations are `ReentrantLock`, `ReadWriteLock`, and
`StampedLock`.

### `ReentrantLock` — `synchronized` with Superpowers

`ReentrantLock` is the most commonly used explicit lock. It behaves exactly like a
`synchronized` block — it is reentrant (the same thread can acquire it multiple times
without deadlocking) — but adds critical features that `synchronized` lacks.

```java
import java.util.concurrent.locks.*;

public class SafeCounter {
    private int count = 0;
    // Explicit lock object — fair=true means threads get the lock in order of waiting
    private final ReentrantLock lock = new ReentrantLock(/* fair= */ false);

    public void increment() {
        lock.lock(); // acquire the lock — blocks until available
        try {
            count++; // critical section
        } finally {
            lock.unlock(); // ALWAYS unlock in a finally block — never skip this!
        }
    }

    public int getCount() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```

The `finally` block guaranteeing `unlock()` is not optional — it is mandatory. If
the critical section throws an exception and you forget to unlock, the lock is held
forever, and all other threads waiting for it will block indefinitely. This is
exactly the same reasoning as closing resources in `finally` blocks or with
try-with-resources.

### `tryLock()` — Non-Blocking Lock Acquisition

This is the most important advantage of `ReentrantLock` over `synchronized`. With
`synchronized`, a thread that cannot acquire the lock must wait indefinitely.
With `tryLock()`, a thread can attempt to acquire the lock and do something else
if it's not available — or retry after a timeout.

```java
public class DeadlockAvoidance {
    private final ReentrantLock lockA = new ReentrantLock();
    private final ReentrantLock lockB = new ReentrantLock();

    public boolean transferData() throws InterruptedException {
        // Try to acquire both locks with a timeout — avoids deadlock
        boolean gotA = lockA.tryLock(100, TimeUnit.MILLISECONDS);
        if (!gotA) {
            System.out.println("Could not acquire lockA, backing off...");
            return false;
        }
        try {
            boolean gotB = lockB.tryLock(100, TimeUnit.MILLISECONDS);
            if (!gotB) {
                System.out.println("Could not acquire lockB, releasing lockA...");
                return false; // finally block releases lockA
            }
            try {
                // Both locks acquired — safe to proceed
                System.out.println("Transfer complete!");
                return true;
            } finally {
                lockB.unlock();
            }
        } finally {
            lockA.unlock(); // always runs, even if lockB.tryLock failed
        }
    }
}
```

### `Condition` — A Better `wait()`/`notifyAll()`

`Condition` objects are obtained from a `Lock` and replace `Object.wait()` and
`Object.notify()`. They are more flexible because you can have multiple conditions
per lock — for example, one condition for "not full" and another for "not empty" —
which avoids the inefficiency of `notifyAll()` waking up every waiting thread.

```java
import java.util.*;
import java.util.concurrent.locks.*;

public class BetterBoundedQueue<T> {
    private final Queue<T> queue = new ArrayDeque<>();
    private final int capacity;
    private final ReentrantLock lock = new ReentrantLock();

    // Two separate conditions — more precise than a single wait/notify
    private final Condition notFull  = lock.newCondition(); // signal when space appears
    private final Condition notEmpty = lock.newCondition(); // signal when item appears

    public BetterBoundedQueue(int capacity) { this.capacity = capacity; }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await(); // wait for "not full" condition
            }
            queue.add(item);
            notEmpty.signal(); // signal ONLY threads waiting for "not empty"
            // (not notifyAll — no need to wake the producers!)
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await(); // wait for "not empty" condition
            }
            T item = queue.poll();
            notFull.signal(); // signal ONLY threads waiting for "not full"
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

The `signal()` (instead of `signalAll()`) is safe here because we *know* exactly what
kind of thread is waiting on each condition — `notFull.signal()` will only wake up a
producer, not a consumer. This is more efficient than `notifyAll()` which would wake
everyone up to re-check their conditions.

### `ReadWriteLock` — Concurrent Readers, Exclusive Writers

For data that is read frequently but written rarely (caches, configuration, lookup
tables), `ReentrantReadWriteLock` provides a huge performance improvement. Multiple
threads can hold the read lock simultaneously, but the write lock is exclusive —
writing blocks all readers and other writers.

```java
import java.util.concurrent.locks.*;
import java.util.*;

public class CachedConfiguration {
    private Map<String, String> config = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock  = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    // Multiple threads can call get() concurrently with NO blocking between them
    public String get(String key) {
        readLock.lock();
        try {
            return config.get(key);
        } finally {
            readLock.unlock();
        }
    }

    // Only one thread can call put() at a time, and it blocks all readers
    public void put(String key, String value) {
        writeLock.lock();
        try {
            config.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }

    // Pattern: read → validate → upgrade to write if needed
    public void reloadIfStale() {
        readLock.lock();
        boolean isStale;
        try {
            isStale = checkIfStale();
        } finally {
            readLock.unlock();
        }
        // NOTE: Cannot upgrade directly from read to write lock in Java!
        // Must release read lock first, then acquire write lock.
        if (isStale) {
            writeLock.lock();
            try {
                // Re-check condition after acquiring write lock (another thread may have updated)
                if (checkIfStale()) {
                    loadNewConfig();
                }
            } finally {
                writeLock.unlock();
            }
        }
    }

    private boolean checkIfStale() { return false; /* implementation */ }
    private void loadNewConfig()   { /* implementation */ }
}
```

### `StampedLock` (Java 8+) — Optimistic Reading

`StampedLock` is the most advanced lock. It supports three modes. **Write locking**
works like an exclusive lock. **Read locking** works like a shared read lock.
**Optimistic reading** is the new mode — it returns a stamp without actually acquiring
any lock, then you check whether the stamp is still valid after reading. If no write
happened during your read, you're done. If a write *did* happen, you fall back to a
proper read lock. This eliminates lock overhead for the common case of no concurrent
writes.

```java
import java.util.concurrent.locks.StampedLock;

public class Point {
    private double x, y;
    private final StampedLock sl = new StampedLock();

    public void move(double deltaX, double deltaY) {
        long stamp = sl.writeLock(); // exclusive write
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            sl.unlockWrite(stamp);
        }
    }

    public double distanceFromOrigin() {
        // Optimistic read — no lock acquired, just a stamp
        long stamp = sl.tryOptimisticRead();
        double currentX = x;  // might be mid-write, but we'll check
        double currentY = y;

        if (!sl.validate(stamp)) { // did a write happen while we were reading?
            // Fall back to a proper read lock
            stamp = sl.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```

Use `StampedLock` when you have a hot read path (many threads reading frequently) and
writes are rare. The optimistic path has essentially zero overhead when there's no
contention. Note that `StampedLock` is NOT reentrant — the same thread cannot acquire
it twice without deadlocking.

---

## ⚛️ Atomic Variables — Lock-Free Thread Safety

The `java.util.concurrent.atomic` package provides classes that perform atomic
operations using hardware-level **CAS (Compare-And-Swap)** instructions. CAS is a
single CPU instruction that atomically does: "if the current value equals the
expected value, replace it with the new value; otherwise do nothing." This is
lock-free — no thread is ever blocked waiting for another.

### `AtomicInteger`, `AtomicLong`, `AtomicBoolean`, `AtomicReference`

```java
import java.util.concurrent.atomic.*;

AtomicInteger counter = new AtomicInteger(0);

// All of these are atomic — guaranteed to work correctly from multiple threads
counter.incrementAndGet(); // ++counter (atomic), returns new value
counter.getAndIncrement(); // counter++ (atomic), returns old value
counter.addAndGet(5);      // counter += 5, returns new value
counter.get();             // just read the current value

// CAS — the building block of lock-free algorithms
// "If current value is 10, change it to 20; otherwise do nothing"
boolean succeeded = counter.compareAndSet(10, 20);
System.out.println("CAS succeeded: " + succeeded);

// getAndUpdate / updateAndGet with a lambda — for complex atomic transformations
AtomicInteger max = new AtomicInteger(0);
int candidate = 42;
// Atomically sets max to Math.max(current, candidate)
max.updateAndGet(current -> Math.max(current, candidate));

// AtomicReference — for reference types
AtomicReference<String> ref = new AtomicReference<>("initial");
ref.compareAndSet("initial", "updated"); // atomic reference swap
System.out.println(ref.get()); // "updated"
```

### `LongAdder` and `LongAccumulator` — High-Contention Counters

`AtomicLong` uses a single memory location, so under very high contention (many
threads hitting the same counter simultaneously), CAS operations fail repeatedly and
threads keep retrying — this is called a **contention storm**. `LongAdder` solves
this by maintaining a set of cells, each updated independently, and summing them only
when you call `sum()`. This dramatically reduces contention.

```java
import java.util.concurrent.atomic.*;

// Use LongAdder for counters under heavy contention — benchmarks often show
// 10–100× better throughput than AtomicLong when many threads increment simultaneously
LongAdder adder = new LongAdder();
adder.increment(); // called from many threads
adder.add(5);
long total = adder.sum(); // only accurate at a quiescent point (low contention reads)

// LongAccumulator: generalises LongAdder for any binary operation
LongAccumulator maxAccum = new LongAccumulator(Long::max, Long.MIN_VALUE);
maxAccum.accumulate(100);
maxAccum.accumulate(50);
maxAccum.accumulate(200);
long max = maxAccum.get(); // 200 — the maximum value accumulated
```

The practical guideline: use `AtomicLong` when you need to read the value frequently
and need exact point-in-time accuracy. Use `LongAdder` when you mostly increment and
only occasionally read the total.

---

## 📦 Concurrent Collections

Regular collections like `ArrayList` and `HashMap` are not thread-safe. You could
wrap them with `Collections.synchronizedList()` or `synchronized` blocks, but this
creates a single global lock that all threads contend for. The concurrent collections
use smarter strategies — segment locking, copy-on-write, or lock-free algorithms —
to achieve much better concurrency.

### `ConcurrentHashMap` — The Most Important Concurrent Collection

`ConcurrentHashMap` is a thread-safe `HashMap` with exceptional performance. In Java
8+, it uses a combination of CAS and per-bucket locking to achieve high concurrency
for reads (which take no locks at all) and writes (which only lock the specific bucket
being modified, not the whole map).

```java
import java.util.concurrent.*;

ConcurrentHashMap<String, Integer> wordCount = new ConcurrentHashMap<>();

// Basic operations are thread-safe
wordCount.put("java", 1);
wordCount.get("java");

// computeIfAbsent — atomically initialises a value if the key is absent
// This is critically important for avoiding race conditions in initialisation
wordCount.computeIfAbsent("python", key -> 0);

// compute — atomically compute a new value from the old value (or null if absent)
// The lambda runs atomically — no lost updates!
String word = "java";
wordCount.compute(word, (key, oldValue) -> oldValue == null ? 1 : oldValue + 1);

// merge — combines a new value with an existing one atomically
wordCount.merge("java", 1, Integer::sum); // add 1 to existing, or initialise to 1

// These atomic compound operations are the KEY advantage over synchronizedMap.
// With synchronizedMap, you'd need a synchronized block for the whole check-then-act:
// synchronized(map) { if (!map.containsKey(k)) map.put(k, compute()); }
// ConcurrentHashMap's methods do this atomically without a global lock.

// forEach and other bulk operations accept a parallelism threshold
wordCount.forEach(2, // use parallelism when there are 2+ entries
    (key, value) -> System.out.printf("%s: %d%n", key, value));
```

Note that while individual operations are thread-safe, *compound operations* that
read-then-write require the atomic methods like `compute`, `merge`, and
`computeIfAbsent`. Never do `if (!map.containsKey(k)) map.put(k, v)` across two
separate calls even on `ConcurrentHashMap` — that's still a race condition.

### `CopyOnWriteArrayList` and `CopyOnWriteArraySet`

These collections work by making a fresh copy of the underlying array on every write.
Reads take no locks at all and see a consistent snapshot. Writes are expensive but
safe — they never interfere with ongoing reads.

```java
import java.util.concurrent.*;

CopyOnWriteArrayList<String> listeners = new CopyOnWriteArrayList<>();

// Reading is completely lock-free and very fast
for (String listener : listeners) { // snapshot — modifications during iteration are invisible
    System.out.println(listener);
}

// Writing is slow (copies the whole array) but thread-safe
listeners.add("listener-1");    // creates a new copy of the array
listeners.remove("listener-1"); // creates another new copy
```

Use `CopyOnWriteArrayList` when reads vastly outnumber writes — the event listener
pattern is the classic use case. Never use it for large lists with frequent writes —
the copying cost becomes prohibitive.

### Blocking Queues — `BlockingQueue` Implementations

`BlockingQueue` extends `Queue` with blocking `put()` (waits if full) and blocking
`take()` (waits if empty). They are the backbone of producer-consumer patterns in
Java and are what `ThreadPoolExecutor` uses internally for its work queue.

```java
import java.util.concurrent.*;

// ArrayBlockingQueue — bounded, backed by an array, FIFO
BlockingQueue<String> bounded = new ArrayBlockingQueue<>(100); // capacity: 100

// LinkedBlockingQueue — optionally bounded, backed by nodes, FIFO
BlockingQueue<String> linkedQ = new LinkedBlockingQueue<>(50); // bounded at 50
BlockingQueue<String> unbounded = new LinkedBlockingQueue<>();  // unbounded (Integer.MAX_VALUE)

// Putting and taking
bounded.put("task-1");      // blocks if full
String task = bounded.take(); // blocks if empty
bounded.offer("task-2", 100, TimeUnit.MILLISECONDS); // try to put, give up after 100ms
bounded.poll(100, TimeUnit.MILLISECONDS);             // try to take, return null after 100ms

// PriorityBlockingQueue — unbounded, elements processed by priority, not FIFO
PriorityBlockingQueue<Integer> pq = new PriorityBlockingQueue<>();
pq.put(5); pq.put(1); pq.put(3);
System.out.println(pq.take()); // 1 — smallest first

// SynchronousQueue — zero capacity! Each put() must rendezvous with a take()
// The producer blocks until a consumer is ready, and vice versa.
// Used in CachedThreadPool to hand off tasks directly to idle threads.
SynchronousQueue<String> handoff = new SynchronousQueue<>();

// DelayQueue — elements can only be taken after their delay expires
// Used for scheduling: scheduled executors, cache expiry, session timeout
import java.util.concurrent.Delayed;
class ScheduledTask implements Delayed {
    private final String name;
    private final long executeAt; // nanoTime to execute
    public ScheduledTask(String name, long delayMs) {
        this.name = name;
        this.executeAt = System.nanoTime() + TimeUnit.MILLISECONDS.toNanos(delayMs);
    }
    public long getDelay(TimeUnit unit) {
        return unit.convert(executeAt - System.nanoTime(), TimeUnit.NANOSECONDS);
    }
    public int compareTo(Delayed other) {
        return Long.compare(getDelay(TimeUnit.NANOSECONDS), other.getDelay(TimeUnit.NANOSECONDS));
    }
}
```

### `ConcurrentLinkedQueue` and `ConcurrentLinkedDeque`

These are lock-free, non-blocking concurrent queues. They use CAS internally and
never block — `poll()` returns `null` if empty instead of waiting. Use them when you
want a shared work queue without the blocking semantics.

```java
ConcurrentLinkedQueue<String> nonBlocking = new ConcurrentLinkedQueue<>();
nonBlocking.offer("item");
String head = nonBlocking.poll(); // returns null if empty, never blocks
```

---

## 🗺️ Choosing the Right Collection

| Scenario | Recommended Collection |
|----------|----------------------|
| General-purpose concurrent map | `ConcurrentHashMap` |
| Read-heavy, write-rare list (listeners) | `CopyOnWriteArrayList` |
| Producer-consumer with bounded capacity | `ArrayBlockingQueue` or `LinkedBlockingQueue` |
| Priority-based task queue | `PriorityBlockingQueue` |
| Direct handoff between threads | `SynchronousQueue` |
| Time-delayed task scheduling | `DelayQueue` |
| Lock-free non-blocking queue | `ConcurrentLinkedQueue` |
| Sorted concurrent map | `ConcurrentSkipListMap` |
| Sorted concurrent set | `ConcurrentSkipListSet` |

---

## ⚠️ Best Practices

**Use `try/finally` with every explicit lock** — the `unlock()` must be in the
`finally` block unconditionally. A single missed `unlock()` call on an exception
path will cause all subsequent threads to block forever.

**Never downgrade from `ReentrantLock` to `synchronized` without a good reason.**
Once you use `ReentrantLock`, its Conditions, and `tryLock`, you get far better
control. Use `synchronized` for simple, short critical sections in non-performance-
critical code where its automatic release on exception is the main benefit.

**Use `ConcurrentHashMap`'s atomic compound operations** (`compute`, `merge`,
`computeIfAbsent`) rather than checking and then putting in separate calls, even
when using `ConcurrentHashMap`. Separate operations are still subject to race conditions.

**Prefer `LongAdder` over `AtomicLong` for high-throughput counters.** If you're
building a metrics system where thousands of threads increment counters per second,
`LongAdder` will outperform `AtomicLong` significantly.

**Do not use `synchronizedList()` or `synchronizedMap()` in new code.** They use a
single global lock for all operations and don't provide atomic compound operations.
The concurrent collections in `java.util.concurrent` are almost always better.

---

## 🔑 Key Takeaways

- `ReentrantLock` adds `tryLock()` (timeout-based acquisition) and `lockInterruptibly()` over `synchronized` — always use `finally` to unlock.
- `Condition` replaces `wait()`/`notify()` and allows multiple conditions per lock, enabling more precise signalling.
- `ReadWriteLock` enables concurrent reads while serialising writes — great for read-heavy data.
- `StampedLock` adds optimistic reading — a near-zero-cost read path when writes are rare.
- Atomic classes (`AtomicInteger`, etc.) use CAS hardware instructions for lock-free thread safety on single variables.
- `LongAdder` beats `AtomicLong` under high contention by sharding the counter across cells.
- `ConcurrentHashMap` uses per-bucket locking and lock-free reads — far superior to `synchronizedMap`. Always use its atomic compound operations.
- `CopyOnWriteArrayList` is ideal for read-heavy, write-rare collections like event listener lists.
- `BlockingQueue` is the foundation of producer-consumer patterns — it handles all the waiting logic for you.
