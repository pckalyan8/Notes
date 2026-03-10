# 🔀 Phase 6 — Concurrency & Multithreading in Java
### Complete Study Guide Index

---

## Why This Phase Matters More Than Any Other

Concurrency is the single hardest topic in Java — and also one of the most important
for building real-world, high-performance applications. Almost every production Java
system you will ever work on is concurrent: web servers handle thousands of requests
simultaneously, background jobs run while users interact with the UI, and data
pipelines process streams of events in parallel.

Concurrency bugs are also among the most dangerous: they are non-deterministic (they
may appear only under specific timing conditions), they are hard to reproduce in
development, and they can silently corrupt data without throwing an exception. A solid
understanding of this phase is what separates a junior developer who writes code that
*mostly* works from a senior engineer who writes code that *always* works correctly.

---

## 📂 Files in This Phase

| File | Topic | Core Concepts |
|------|-------|---------------|
| `Phase6_01_Thread_Fundamentals.md` | Threads 101 | Thread lifecycle, states, Runnable, daemon threads |
| `Phase6_02_Synchronization_JMM.md` | Synchronization & Memory Model | Race conditions, `synchronized`, `volatile`, JMM, deadlock |
| `Phase6_03_Locks_Atomics_Collections.md` | `java.util.concurrent` — Part 1 | `ReentrantLock`, `StampedLock`, `AtomicInteger`, concurrent collections |
| `Phase6_04_Executor_Framework.md` | Executor Framework | Thread pools, `Future`, `Callable`, `ThreadPoolExecutor` |
| `Phase6_05_CompletableFuture.md` | CompletableFuture | Async pipelines, combining, error handling |
| `Phase6_06_ForkJoin_VirtualThreads.md` | Fork-Join & Virtual Threads | Work-stealing, Project Loom, structured concurrency |
| `Phase6_07_Concurrency_Patterns.md` | Patterns | Producer-Consumer, thread-local, immutability, synchronizers |

---

## 🗺️ The Mental Map of Java Concurrency

Java concurrency evolved across several generations. Understanding which generation
each tool belongs to helps you know *why* it exists and when to reach for it.

```
GENERATION 1 — Java 1.0 (1996)
  Raw Thread class, synchronized keyword, wait/notify
  ↓ Problem: Error-prone, verbose, low-level

GENERATION 2 — Java 5 (2004) — java.util.concurrent (JSR-166, Brian Goetz)
  Executors, Locks, Atomic variables, Concurrent collections
  ↓ Problem: Still callback-heavy; async composition was painful

GENERATION 3 — Java 8 (2014)
  CompletableFuture — async pipelines with lambda composition
  ↓ Problem: Platform threads are expensive (1MB stack); I/O blocks them

GENERATION 4 — Java 21 (2023) — Project Loom
  Virtual Threads — millions of lightweight threads
  Structured Concurrency — safe task lifecycle management
```

---

## ⚡ The Core Mental Models

Three mental models underpin everything in this phase. Internalise these before
diving into the syntax.

**The Visibility Problem.** When Thread A writes to a variable, Thread B may not see
that write — not because of a bug, but because modern CPUs and compilers cache values
in registers and processor-local caches for performance. Java's Memory Model defines
exactly when writes made by one thread are *guaranteed* to be visible to another.
The `volatile` keyword and `synchronized` blocks are how you establish those guarantees.

**The Atomicity Problem.** Even a simple operation like `counter++` is not atomic —
it is actually three steps: read the value, increment it, write it back. If two
threads do this simultaneously on the same variable, both might read the same value,
both increment it, and you lose one increment. This is a *race condition*, and it
produces results that are incorrect but never throw exceptions, making them
particularly insidious.

**The Ordering Problem.** The Java compiler and the JVM are allowed to reorder
instructions for performance, as long as the result is correct *within a single
thread*. But this reordering can cause other threads to see operations happening in
an unexpected order. The Java Memory Model's *happens-before* relationship is how
you reason about ordering guarantees between threads.

---

## 📐 Study Advice

Work through this phase in order. Thread fundamentals and synchronization *must* come
before the `java.util.concurrent` tools, because you need to understand what problems
those tools were built to solve. Build at least one real project that exercises
concurrency — a bounded producer-consumer queue, a simple thread pool, or a parallel
file processor. Reading alone is not enough for this phase.
