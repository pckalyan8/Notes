# 🔀 Phase 6.2 — Thread Synchronization & the Java Memory Model
### Race Conditions, `synchronized`, `volatile`, Deadlock, and the JMM

---

## 🧠 The Core Problem: Shared Mutable State

The root cause of almost every concurrency bug is the combination of two things
happening at the same time: multiple threads **sharing** the same data, and at least
one of them **mutating** it. If data is immutable (nobody changes it), sharing is
completely safe. If data is not shared (each thread has its own copy), mutation is
completely safe. Problems only arise when you have both sharing *and* mutation at the
same time.

Before looking at solutions, you need to deeply understand the two kinds of problems
that arise: **atomicity violations** (race conditions) and **visibility failures**.

---

## 💥 Race Conditions — The Atomicity Problem

A **race condition** occurs when the correctness of a computation depends on the
relative timing of multiple threads. The most classic example is a shared counter.

```java
public class UnsafeCounter {
    private int count = 0; // shared mutable state

    public void increment() {
        count++; // THIS IS NOT ATOMIC
    }

    public int getCount() { return count; }
}

// Test it:
UnsafeCounter counter = new UnsafeCounter();
Runnable task = () -> {
    for (int i = 0; i < 10_000; i++) {
        counter.increment();
    }
};

Thread t1 = new Thread(task, "T1");
Thread t2 = new Thread(task, "T2");
t1.start(); t2.start();
t1.join();  t2.join();
System.out.println("Expected: 20000, Actual: " + counter.getCount());
// Output might be: 14,872 or 17,341 or 19,998 — different every run!
```

Why does `count++` fail? Because it compiles down to *three operations*:
first read the value of `count` from memory, then add 1 to it, then write the new
value back to memory. If two threads both execute step 1 before either executes step
3, they both read the same value (say, `100`), both add 1 (getting `101`), and both
write `101` back. One increment was completely lost. This is a **write-write race
condition**.

The key insight: anything that looks like one line in Java may not be one atomic
operation at the CPU level. Even assigning a `long` or `double` value (which are
64-bit) is not guaranteed to be atomic on 32-bit JVMs because it may be done as two
32-bit operations.

---

## 👁️ Visibility Failures — The Memory Visibility Problem

The second kind of problem is subtler. Modern CPUs don't read and write to main
memory on every operation — they work from local **processor caches** (L1, L2, L3).
The JVM and JIT compiler also perform optimisations like keeping variables in
registers or reordering instructions. The result is that when Thread A writes a
value to a variable, Thread B may *never see that write* because B is reading from
its own cache that hasn't been updated.

```java
// A classic visibility bug
public class StopFlag {
    private boolean stopped = false; // no visibility guarantee!

    public void run() {
        while (!stopped) {  // Thread B might cache this as 'false' FOREVER
            // ... do work ...
        }
        System.out.println("Stopped!");
    }

    public void stop() {
        stopped = true; // Thread A sets this, but Thread B might never see it
    }
}
```

In this example, Thread B might loop forever even after Thread A sets `stopped = true`
because the JVM is free to cache `stopped` in Thread B's register and never re-read
it from main memory. This is not a theoretical concern — it happens in practice,
especially on multi-core hardware and with JIT-compiled code.

---

## 🔒 The `synchronized` Keyword — Mutual Exclusion

The `synchronized` keyword is Java's oldest and simplest synchronization mechanism.
It solves **both** the atomicity and visibility problems at once.

Every Java object has an intrinsic **monitor lock** (also called a *mutex* or
*intrinsic lock*). When a thread enters a `synchronized` block or method, it
**acquires** the lock. Any other thread that tries to enter a synchronized block
guarded by the *same* lock will **block** (enter BLOCKED state) until the first
thread exits the block and **releases** the lock.

This ensures only one thread at a time can execute the protected code — mutual
exclusion. It also flushes the thread's cache to main memory on release and reloads
from main memory on acquire, solving the visibility problem.

### Synchronized Methods

```java
public class SafeCounter {
    private int count = 0;

    // 'this' object's lock is used — the entire method is critical section
    public synchronized void increment() {
        count++; // now truly atomic — only one thread at a time
    }

    public synchronized int getCount() {
        return count; // synchronized getter ensures visibility too
    }
}
```

When a thread calls `increment()`, it acquires the lock on `this`. Any other thread
that tries to call `increment()` or `getCount()` on the *same* `SafeCounter` object
will block until the first thread exits the method.

### Synchronized Blocks — Prefer This Form

Synchronized blocks let you specify exactly which object's lock to use, and they
minimise the scope of the critical section — which is always better for performance
because the narrower the lock, the less time other threads are blocked.

```java
public class SafeBank {
    private final Object transferLock = new Object(); // a dedicated lock object
    private int balance;

    public SafeBank(int initialBalance) {
        this.balance = initialBalance;
    }

    public void deposit(int amount) {
        // Only the actual balance modification needs synchronization
        // Validation and logging can happen outside the lock
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        System.out.printf("[%s] Depositing %d%n", Thread.currentThread().getName(), amount);

        synchronized (transferLock) { // acquire the lock
            balance += amount;
        } // release the lock here

        System.out.printf("[%s] New balance: %d%n", Thread.currentThread().getName(), balance);
    }

    public int getBalance() {
        synchronized (transferLock) {
            return balance; // must also be synchronized for visibility
        }
    }
}
```

### The `this` Lock vs. the Class Lock

For synchronized *instance* methods, the lock object is `this` (the specific
instance). For synchronized *static* methods, the lock object is the `Class` object
(`ClassName.class`). These are completely different locks — a synchronized static
method and a synchronized instance method can run simultaneously.

```java
public class LockExample {

    public synchronized void instanceMethod() {
        // Acquires lock on 'this' instance
    }

    public static synchronized void staticMethod() {
        // Acquires lock on LockExample.class object
        // Completely independent of the instance lock above
    }

    public void explicitLock() {
        synchronized (this) { /* instance lock */ }
        synchronized (LockExample.class) { /* class lock */ }
    }
}
```

### Reentrancy — A Thread Can Re-Acquire Its Own Lock

Java's intrinsic locks are **reentrant**: a thread that already holds a lock can
acquire it again without deadlocking. This allows synchronized methods to call
other synchronized methods on the same object.

```java
public synchronized void outerMethod() {
    System.out.println("outer");
    innerMethod(); // This does NOT deadlock — the current thread already holds the lock
}

public synchronized void innerMethod() {
    System.out.println("inner"); // can be entered by the same thread
}
```

---

## 📡 `volatile` — Visibility Without Mutual Exclusion

When you only need to solve the **visibility** problem (not the atomicity problem),
`volatile` is a lighter-weight alternative to `synchronized`. A write to a `volatile`
variable is immediately flushed to main memory, and a read from a `volatile` variable
always reads from main memory, bypassing CPU caches. This guarantees visibility
across threads.

```java
public class StopFlagFixed {
    // volatile ensures Thread B always sees the latest value written by Thread A
    private volatile boolean stopped = false;

    public void run() {
        while (!stopped) {
            // Thread B will always see updates to 'stopped'
        }
        System.out.println("Stopped cleanly!");
    }

    public void stop() {
        stopped = true; // guaranteed to be visible to all threads immediately
    }
}
```

However, `volatile` does **not** provide atomicity. The `volatile` keyword on a
`count` variable would fix the visibility issue but would NOT fix the `count++` race
condition — the read-modify-write cycle is still non-atomic.

```java
// volatile solves visibility but NOT atomicity
private volatile int count = 0;

// STILL A RACE CONDITION:
count++; // read (from main mem), add 1, write (to main mem) — 3 ops, not atomic
```

The practical rule: use `volatile` for simple flag variables (booleans) that one
thread writes and others read, where the write is unconditional (not a
read-modify-write). Use `synchronized` or `AtomicInteger` for anything more complex.

---

## 🧩 `wait()`, `notify()`, `notifyAll()` — Thread Coordination

Beyond mutual exclusion, threads often need to **coordinate**: Thread B needs to
pause until Thread A has produced some data, for example. This is the
Producer-Consumer pattern, and it's implemented with `wait()`, `notify()`, and
`notifyAll()`. These methods are on `Object` (not `Thread`) and can only be called
inside a `synchronized` block on the same object.

When a thread calls `object.wait()`:
1. It releases the lock on `object`.
2. It enters WAITING state and parks itself.
3. It will wake up only when another thread calls `object.notify()` or
   `object.notifyAll()` on the same object, **and** it can re-acquire the lock.

```java
import java.util.*;

public class BoundedQueue<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;

    public BoundedQueue(int capacity) {
        this.capacity = capacity;
    }

    // Producer calls this — blocks if the queue is full
    public synchronized void put(T item) throws InterruptedException {
        while (queue.size() == capacity) { // WHILE loop, not if — see why below
            wait(); // releases lock and waits for a signal
        }
        queue.add(item);
        System.out.printf("[%s] Produced: %s (size=%d)%n",
            Thread.currentThread().getName(), item, queue.size());
        notifyAll(); // wake up any threads waiting to take
    }

    // Consumer calls this — blocks if the queue is empty
    public synchronized T take() throws InterruptedException {
        while (queue.isEmpty()) { // WHILE loop — critical for correctness
            wait(); // releases lock and waits for a signal
        }
        T item = queue.poll();
        System.out.printf("[%s] Consumed: %s (size=%d)%n",
            Thread.currentThread().getName(), item, queue.size());
        notifyAll(); // wake up any threads waiting to put
        return item;
    }
}
```

The `while` loop instead of `if` is essential. When a thread wakes up from `wait()`,
the condition it was waiting for might not be true anymore (another thread could have
consumed the item first — this is called a **spurious wakeup**). Using `while` causes
the thread to re-check the condition and go back to waiting if it's still not met.

Use `notifyAll()` rather than `notify()` unless you are absolutely certain that only
one waiting thread can make progress. `notify()` wakes up just one waiting thread
(chosen arbitrarily), which can lead to missed signals if the wrong thread is woken.

---

## 🌐 The Java Memory Model (JMM)

The Java Memory Model (JMM) is the formal specification that defines *when* writes
made by one thread are guaranteed to be visible to reads by another thread. It does
this through the concept of **happens-before** relationships.

A **happens-before** relationship between two operations A and B means: if A
happens-before B, then A's effects (all writes) are guaranteed to be visible to B.
The key happens-before rules are:

**Monitor lock rule**: An unlock of a monitor happens-before every subsequent lock of
that same monitor. (Everything inside a `synchronized` block is visible to the next
thread that acquires the same lock.)

**Volatile variable rule**: A write to a `volatile` variable happens-before every
subsequent read of that variable.

**Thread start rule**: `Thread.start()` happens-before any action in the started thread.
(Everything the parent thread did before calling `start()` is visible to the child thread.)

**Thread join rule**: All actions in a thread happen-before `Thread.join()` returns.
(Everything a thread did is visible to the thread that called `join()` on it.)

**Program order rule**: Within a single thread, each action happens-before the next
action in program order.

The JMM is what justifies everything you do with `synchronized` and `volatile`. If
two threads access a variable and there is no happens-before relationship between
their accesses, the results are **undefined** — the read may see an old value, a
partially-written value, or anything the JVM chooses.

---

## ☠️ Deadlock — When Threads Wait for Each Other Forever

A **deadlock** occurs when two or more threads are each waiting for a resource held
by the other, and none can proceed.

```java
public class DeadlockExample {
    private final Object lockA = new Object();
    private final Object lockB = new Object();

    // Thread 1 acquires A, then tries to get B
    public void method1() throws InterruptedException {
        synchronized (lockA) {
            System.out.println("T1: acquired lockA, waiting for lockB...");
            Thread.sleep(100); // Give T2 time to acquire lockB
            synchronized (lockB) { // DEADLOCK: T2 holds lockB and wants lockA
                System.out.println("T1: acquired both locks");
            }
        }
    }

    // Thread 2 acquires B, then tries to get A
    public void method2() throws InterruptedException {
        synchronized (lockB) {
            System.out.println("T2: acquired lockB, waiting for lockA...");
            Thread.sleep(100);
            synchronized (lockA) { // DEADLOCK: T1 holds lockA and wants lockB
                System.out.println("T2: acquired both locks");
            }
        }
    }
}
// This program will hang forever — neither thread can proceed
```

The **fix** is straightforward: always acquire multiple locks **in the same order**
across all threads. If every thread acquires `lockA` before `lockB`, the deadlock
cannot occur because no thread will hold `lockB` while waiting for `lockA`.

```java
// FIXED: both methods acquire locks in the same order (A, then B)
public void method1() {
    synchronized (lockA) {
        synchronized (lockB) { System.out.println("T1: done"); }
    }
}
public void method2() {
    synchronized (lockA) { // same order — T2 now waits for A first
        synchronized (lockB) { System.out.println("T2: done"); }
    }
}
```

Other deadlock prevention strategies include using `tryLock()` with a timeout
(covered in the Locks file), using a lock ordering based on object ID (for dynamic
situations), and minimising the number of locks held simultaneously.

---

## 😴 Livelock and Starvation

**Livelock** is like deadlock but the threads are not paused — they are actively
running and politely yielding to each other but never making progress. Imagine two
people in a narrow corridor both stepping aside at the same time to let the other
pass, and then both stepping the same way again, indefinitely.

**Starvation** occurs when one or more threads can never get CPU time or access to
a resource because other threads are always preferred. A low-priority thread can
starve if high-priority threads continuously run. A thread waiting on a lock can
starve if the lock is always quickly re-acquired by other threads before it gets
a chance.

---

## 🧪 Complete Example — Thread-Safe Stack

```java
import java.util.*;

public class SynchronizedStack<T> {
    private final LinkedList<T> items = new LinkedList<>();
    private final int maxSize;

    public SynchronizedStack(int maxSize) {
        this.maxSize = maxSize;
    }

    public synchronized void push(T item) throws InterruptedException {
        while (items.size() >= maxSize) {
            System.out.printf("[%s] Stack full, waiting...%n",
                Thread.currentThread().getName());
            wait();
        }
        items.addFirst(item);
        System.out.printf("[%s] Pushed: %s (size=%d)%n",
            Thread.currentThread().getName(), item, items.size());
        notifyAll();
    }

    public synchronized T pop() throws InterruptedException {
        while (items.isEmpty()) {
            System.out.printf("[%s] Stack empty, waiting...%n",
                Thread.currentThread().getName());
            wait();
        }
        T item = items.removeFirst();
        System.out.printf("[%s] Popped: %s (size=%d)%n",
            Thread.currentThread().getName(), item, items.size());
        notifyAll();
        return item;
    }

    public static void main(String[] args) throws InterruptedException {
        SynchronizedStack<Integer> stack = new SynchronizedStack<>(3);

        Thread producer = new Thread(() -> {
            for (int i = 1; i <= 6; i++) {
                try {
                    stack.push(i);
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }, "producer");

        Thread consumer = new Thread(() -> {
            for (int i = 0; i < 6; i++) {
                try {
                    Thread.sleep(250); // consumer is slower
                    stack.pop();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }, "consumer");

        producer.start();
        consumer.start();
        producer.join();
        consumer.join();
    }
}
```

---

## ⚠️ Best Practices

**Keep synchronized blocks as small as possible.** Synchronization blocks other
threads. Only put the minimum code necessary inside the block — compute values
outside, only modify shared state inside.

**Never call external or unknown methods while holding a lock.** Calling a method
you don't control while holding a lock is an invitation for deadlock — that method
might try to acquire another lock in an incompatible order.

**Always use `while`, not `if`, around `wait()` calls.** Spurious wakeups are
real, and the condition you waited for may have been consumed by another thread
before you re-acquire the lock. A `while` loop protects you from both.

**Prefer `notifyAll()` over `notify()`** unless you have a specific reason and
can prove that only one waiting thread can ever make progress.

**Lock on a private final dedicated lock object**, not `this` or the class. If you
lock on `this`, external code can synchronize on your object too, creating
unexpected contention and potential deadlocks.

**Acquire multiple locks in a consistent global order** across all code paths. This
is the most reliable way to prevent deadlock.

---

## 🔑 Key Takeaways

- Race conditions happen when multiple threads perform non-atomic read-modify-write operations on shared state.
- Visibility failures happen because CPUs and JITs cache values; writes by one thread may not be seen by another without proper synchronization.
- `synchronized` solves both atomicity and visibility by providing mutual exclusion and establishing a happens-before relationship.
- `volatile` solves only visibility, not atomicity — use it for single-write, multi-read flags.
- `wait()` releases the lock and parks the thread; `notify()`/`notifyAll()` wakes waiting threads; always use `while` loops around `wait()`.
- The Java Memory Model defines happens-before relationships — without one between two thread accesses to a variable, behaviour is undefined.
- Deadlock happens when threads hold locks and wait for each other — prevent it by acquiring multiple locks in a consistent order.
