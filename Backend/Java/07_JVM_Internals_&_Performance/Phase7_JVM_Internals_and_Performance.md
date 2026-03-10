# ☕ Phase 7 — JVM Internals & Performance
### Java Complete Mastery Roadmap | Deep Dive Guide

---

> **Goal of this phase:** Understand what actually happens when your Java program runs — from class loading to garbage collection to JIT compilation. This knowledge separates good Java developers from great ones.

---

## 📌 Table of Contents

- [7.1 JVM Architecture](#71-jvm-architecture)
  - [ClassLoader Subsystem](#classloader-subsystem)
  - [Runtime Data Areas](#runtime-data-areas)
  - [Execution Engine](#execution-engine)
  - [Native Method Interface (JNI)](#native-method-interface-jni)
- [7.2 Garbage Collection](#72-garbage-collection)
  - [Object Lifecycle & Reachability](#object-lifecycle--reachability)
  - [GC Roots](#gc-roots)
  - [Mark-and-Sweep Algorithm](#mark-and-sweep-algorithm)
  - [Generational GC Theory](#generational-gc-theory)
  - [GC Algorithms](#gc-algorithms)
  - [GC Tuning Flags](#gc-tuning-flags)
  - [Minor GC vs Major GC vs Full GC](#minor-gc-vs-major-gc-vs-full-gc)
  - [Memory Leaks](#memory-leaks)
  - [Reference Types](#reference-types)
- [7.3 JVM Performance Tuning](#73-jvm-performance-tuning)
- [7.4 Profiling & Monitoring Tools](#74-profiling--monitoring-tools)
- [7.5 JVM Languages Interop](#75-jvm-languages-interop)
- [📋 Quick Reference Cheat Sheet](#-quick-reference-cheat-sheet)

---

# 7.1 JVM Architecture

## Overview

The **Java Virtual Machine (JVM)** is the engine that runs your compiled Java bytecode. It is the reason Java achieves "Write Once, Run Anywhere." The JVM is a **specification** — multiple vendors implement it (Oracle HotSpot, OpenJ9, GraalVM, Amazon Corretto, etc.).

```
┌─────────────────────────────────────────────────────┐
│                  Java Source Code (.java)            │
└───────────────────────┬─────────────────────────────┘
                        │  javac (compiler)
                        ▼
┌─────────────────────────────────────────────────────┐
│                Java Bytecode (.class)                │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│                     JVM                             │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────┐ │
│  │  ClassLoader │  │ Runtime Data  │  │Execution │ │
│  │  Subsystem   │  │     Areas     │  │ Engine   │ │
│  └──────────────┘  └───────────────┘  └──────────┘ │
└─────────────────────────────────────────────────────┘
```

---

## ClassLoader Subsystem

The ClassLoader is responsible for **loading `.class` files into the JVM**. It follows a hierarchical structure.

### The Three Built-in ClassLoaders

```
Bootstrap ClassLoader  (C/C++ native, no Java parent)
        │
        ▼
Extension ClassLoader  (Java, loads from jre/lib/ext)
        │
        ▼
Application ClassLoader (Java, loads from classpath)
        │
        ▼
Custom ClassLoader     (Your own implementation)
```

| ClassLoader | What it loads | Location |
|---|---|---|
| **Bootstrap** | Core Java classes (`java.lang`, `java.util`, etc.) | `$JAVA_HOME/lib/rt.jar` or modules |
| **Extension/Platform** | Java extension libraries | `$JAVA_HOME/lib/ext` or `java.ext.dirs` |
| **Application (System)** | Your application classes | `-classpath` or `-cp` |
| **Custom** | Whatever you define | Defined by you |

### Parent Delegation Model

> **Key Rule:** Before loading a class, a ClassLoader always **delegates to its parent first**. Only if the parent cannot find the class does the child attempt to load it.

```java
// Demonstrating ClassLoader hierarchy
public class ClassLoaderDemo {
    public static void main(String[] args) {
        // Bootstrap ClassLoader loads String (returns null — it's native)
        System.out.println(String.class.getClassLoader());          // null
        
        // Application ClassLoader loads your classes
        System.out.println(ClassLoaderDemo.class.getClassLoader()); // AppClassLoader
        
        // You can get the parent of any ClassLoader
        ClassLoader appCL = ClassLoaderDemo.class.getClassLoader();
        System.out.println(appCL.getParent());         // PlatformClassLoader
        System.out.println(appCL.getParent().getParent()); // null (Bootstrap)
    }
}
```

**Why Parent Delegation?**
- **Security:** Prevents malicious code from replacing `java.lang.String` with a fake version.
- **Uniqueness:** Ensures a class is loaded only once.
- **Consistency:** Core classes are always from the trusted Bootstrap loader.

### Custom ClassLoader

```java
import java.io.*;

public class MyClassLoader extends ClassLoader {
    
    private String classPath;
    
    public MyClassLoader(String classPath) {
        this.classPath = classPath;
    }
    
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classBytes = loadClassBytes(name);
        if (classBytes == null) {
            throw new ClassNotFoundException(name);
        }
        // defineClass converts bytes to a Class object
        return defineClass(name, classBytes, 0, classBytes.length);
    }
    
    private byte[] loadClassBytes(String className) {
        String fileName = classPath + "/" + className.replace('.', '/') + ".class";
        try (FileInputStream fis = new FileInputStream(fileName)) {
            return fis.readAllBytes();
        } catch (IOException e) {
            return null;
        }
    }
    
    public static void main(String[] args) throws Exception {
        MyClassLoader loader = new MyClassLoader("/path/to/classes");
        Class<?> clazz = loader.loadClass("com.example.MyClass");
        Object obj = clazz.getDeclaredConstructor().newInstance();
        System.out.println(obj.getClass().getClassLoader()); // MyClassLoader
    }
}
```

**Use cases for Custom ClassLoaders:**
- Hot reloading (application servers like Tomcat)
- Loading classes from databases or networks
- Implementing plugin systems
- Sandboxing (isolating code)

### Class Loading Lifecycle

Every class goes through this lifecycle exactly **once**:

```
Loading → Linking (Verify → Prepare → Resolve) → Initialization
```

| Phase | What Happens |
|---|---|
| **Loading** | Reads `.class` file bytes, creates `Class` object in Method Area |
| **Verification** | Checks bytecode is valid and safe (no stack overflows, illegal access, etc.) |
| **Preparation** | Allocates memory for static fields, sets **default** values (`0`, `null`, `false`) |
| **Resolution** | Converts symbolic references to direct memory references |
| **Initialization** | Runs static initializers, assigns actual static field values |

```java
public class ClassLifecycleDemo {
    
    // Preparation phase: MAX_SIZE gets 0 (default int)
    // Initialization phase: MAX_SIZE gets 100 (actual value)
    static int MAX_SIZE = 100;
    
    // Static initializer runs during Initialization phase
    static {
        System.out.println("Class being initialized! MAX_SIZE = " + MAX_SIZE);
        MAX_SIZE = 200; // Can be further modified
    }
    
    public static void main(String[] args) {
        System.out.println("Final MAX_SIZE = " + MAX_SIZE); // 200
    }
}
```

**A class is initialized when:**
- First instance is created (`new`)
- A static field is accessed or modified
- A static method is called
- It is the main class
- A subclass is initialized
- Via reflection (`Class.forName()`)

---

## Runtime Data Areas

These are the **memory regions** the JVM creates and manages at runtime.

```
┌───────────────────────────────────────────────────────────────────┐
│                          JVM Memory                               │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                    SHARED AREAS (all threads)                │ │
│  │                                                              │ │
│  │  ┌────────────────────┐    ┌──────────────────────────────┐ │ │
│  │  │    Method Area     │    │            Heap              │ │ │
│  │  │  (Metaspace Java8+)│    │  ┌─────────────┐ ┌────────┐  │ │ │
│  │  │  - Class metadata  │    │  │Young Gen    │ │Old Gen │  │ │ │
│  │  │  - Static fields   │    │  │┌───┐┌──┐┌──┐│ │(Tenured│  │ │ │
│  │  │  - Method bytecode │    │  ││Eden││S0││S1││ │  )     │  │ │ │
│  │  │  - Runtime constants│   │  │└───┘└──┘└──┘│ │        │  │ │ │
│  │  └────────────────────┘    │  └─────────────┘ └────────┘  │ │ │
│  │                            └──────────────────────────────┘ │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │               PER-THREAD AREAS                               │ │
│  │                                                              │ │
│  │  Thread 1              Thread 2              Thread N        │ │
│  │  ┌──────────────┐      ┌──────────────┐      ┌───────────┐  │ │
│  │  │  Java Stack  │      │  Java Stack  │      │Java Stack │  │ │
│  │  │ ┌──────────┐ │      │              │      │           │  │ │
│  │  │ │  Frame 3 │ │      │              │      │           │  │ │
│  │  │ ├──────────┤ │      │              │      │           │  │ │
│  │  │ │  Frame 2 │ │      │              │      │           │  │ │
│  │  │ ├──────────┤ │      │              │      │           │  │ │
│  │  │ │  Frame 1 │ │      │              │      │           │  │ │
│  │  │ └──────────┘ │      │              │      │           │  │ │
│  │  ├──────────────┤      ├──────────────┤      ├───────────┤  │ │
│  │  │  PC Register │      │  PC Register │      │PC Register│  │ │
│  │  ├──────────────┤      ├──────────────┤      ├───────────┤  │ │
│  │  │Native Method │      │Native Method │      │Native Meth│  │ │
│  │  │    Stack     │      │    Stack     │      │   Stack   │  │ │
│  │  └──────────────┘      └──────────────┘      └───────────┘  │ │
│  └──────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

### 1. Method Area / Metaspace

- **Shared** among all threads
- Stores **class-level** data: class metadata, method bytecode, static variables, runtime constant pool
- In **Java 7 and below:** Called "PermGen" (Permanent Generation), was part of heap with fixed size — notorious for `OutOfMemoryError: PermGen space`
- In **Java 8+:** Replaced by **Metaspace** — stored in **native memory** (not heap), grows dynamically

```java
// Static fields live in Method Area / Metaspace
public class MethodAreaDemo {
    static int instanceCount = 0;  // In Method Area
    static final String APP_NAME = "Demo"; // String constant in pool
    
    int id; // Instance field — lives on Heap per object
    
    public MethodAreaDemo() {
        instanceCount++;
        this.id = instanceCount;
    }
}
```

**Key flags:**
```bash
# Limit Metaspace size (default is unlimited — can fill native memory!)
-XX:MetaspaceSize=128m         # Initial size
-XX:MaxMetaspaceSize=256m      # Maximum size
```

### 2. Heap

The **largest memory area** — where **all objects** live. Managed by Garbage Collector.

```java
// All of these are allocated on the Heap
String name = new String("Alice");     // String object on Heap
int[] numbers = new int[1000];         // Array object on Heap
List<String> list = new ArrayList<>(); // ArrayList object on Heap

// Primitives inside objects are also on Heap (as part of the object)
// But local primitive variables are on the Stack
```

**Heap Structure (Generational):**

```
Young Generation (typically 1/3 of heap)
├── Eden Space         — new objects are born here
├── Survivor Space S0  — survivors of 1st GC
└── Survivor Space S1  — survivors of 2nd GC

Old Generation / Tenured Space (typically 2/3 of heap)
└── Objects that survived multiple GC cycles

(Java 8+: No PermGen — replaced by Metaspace in native memory)
```

### 3. Java Stack (Per Thread)

- Each **thread** has its own stack
- Stores **stack frames** — one frame per method call
- When a method is called → frame pushed; when method returns → frame popped
- **Not shared** → naturally thread-safe

Each **stack frame** contains:
- **Local Variable Array** — local variables (including `this`)
- **Operand Stack** — working area for computations
- **Frame Data** — reference to runtime constant pool, return address

```java
public class StackDemo {
    
    public static void main(String[] args) {        // Frame 1 pushed
        int result = add(3, 4);                     // Frame 2 pushed
        System.out.println(result);                 // Frame 3 pushed/popped
    }                                               // Frame 1 popped
    
    static int add(int a, int b) {                 // Frame 2
        int sum = a + b;                           // Local vars: a, b, sum
        return sum;                                // Frame 2 popped
    }
}

// Stack trace when error occurs:
// Exception in thread "main" java.lang.ArithmeticException
//     at StackDemo.add(StackDemo.java:9)   <-- Frame 2
//     at StackDemo.main(StackDemo.java:4)  <-- Frame 1
```

**StackOverflowError:**

```java
// Infinite recursion fills the stack
public class StackOverflow {
    static int count = 0;
    
    static void recurse() {
        count++;
        recurse(); // No base case — stack fills up!
    }
    
    public static void main(String[] args) {
        try {
            recurse();
        } catch (StackOverflowError e) {
            System.out.println("Stack overflow at depth: " + count);
            // Typically prints around 5000-20000 depending on -Xss setting
        }
    }
}
```

**Stack size flag:**
```bash
-Xss512k   # Set thread stack size to 512KB (default is ~512KB-1MB)
```

### 4. PC Register (Program Counter)

- One per thread
- Holds the **address of the current JVM instruction** being executed
- If current method is native, PC is undefined

### 5. Native Method Stack

- Supports **native methods** (written in C/C++, called via JNI)
- Similar structure to Java Stack but for native code

---

## Execution Engine

The component that **executes** the bytecode.

### 1. Interpreter

- Reads and executes bytecode **one instruction at a time**
- **Fast to start** (no compilation needed)
- **Slow execution** — decodes same instructions repeatedly

### 2. JIT Compiler (Just-In-Time)

The JIT solves the interpreter's performance problem by **compiling hot bytecode to native machine code** at runtime.

```
Bytecode → Interpreter (starts fast) 
         → JIT notices "hot" methods (called many times)
         → JIT compiles hot methods to native machine code
         → Native code runs much faster (no interpretation overhead)
```

**HotSpot JVM has two JIT compilers (Tiered Compilation):**

| Compiler | Level | Speed | Optimization |
|---|---|---|---|
| **C1 (Client)** | Tier 1-3 | Fast compile | Basic optimizations |
| **C2 (Server)** | Tier 4 | Slow compile | Aggressive optimizations |

**Tiered Compilation (default since Java 8):**

```
Tier 0: Interpreter
Tier 1: C1 — no profiling
Tier 2: C1 — limited profiling  
Tier 3: C1 — full profiling (counting invocations, branches)
Tier 4: C2 — aggressively optimized native code
```

**JIT Optimizations include:**
- **Inlining:** Replaces method calls with the method body
- **Loop unrolling:** Reduces loop overhead
- **Dead code elimination:** Removes unreachable code
- **Escape analysis:** Detects if an object never escapes a method (then allocates it on stack instead of heap)
- **Constant folding:** Pre-computes constant expressions

```java
// JIT inlining example — JIT will inline this small method
public class JITDemo {
    
    // JIT will inline this — no method call overhead at runtime
    private static int add(int a, int b) {
        return a + b;
    }
    
    public static void main(String[] args) {
        long sum = 0;
        for (int i = 0; i < 1_000_000; i++) {
            sum += add(i, i + 1); // After enough iterations, JIT compiles this
        }
        System.out.println(sum);
    }
}
```

**Useful JIT flags:**
```bash
-XX:+PrintCompilation          # Print methods being JIT compiled
-XX:CompileThreshold=10000     # Number of invocations before JIT compiles
-XX:+TieredCompilation         # Enable tiered (default)
-Xint                          # Interpreter-only mode (no JIT — much slower)
-Xcomp                         # Compile everything upfront
```

### 3. AOT Compilation (GraalVM)

**Ahead-of-Time Compilation** compiles Java to native binary *before* runtime.

```bash
# Using GraalVM Native Image
native-image -jar myapp.jar myapp
./myapp  # Starts in milliseconds, no JVM needed
```

**Trade-offs:**

| | JIT (HotSpot) | AOT (GraalVM Native) |
|---|---|---|
| **Startup time** | Slow (JVM warmup) | Very fast (ms) |
| **Peak throughput** | Very high (optimized over time) | Lower (no runtime profiling) |
| **Memory** | Higher | Lower |
| **Use case** | Long-running servers | Serverless, CLIs, containers |

---

## Native Method Interface (JNI)

Allows Java to call **native C/C++ code**.

```java
// Java side
public class NativeDemo {
    
    // Declare native method
    public native void hello();
    
    static {
        // Load native library
        System.loadLibrary("NativeLib");
    }
    
    public static void main(String[] args) {
        new NativeDemo().hello(); // Calls C code
    }
}
```

```c
// C side (NativeLib.c)
#include <jni.h>
#include <stdio.h>

JNIEXPORT void JNICALL Java_NativeDemo_hello(JNIEnv *env, jobject obj) {
    printf("Hello from native C code!\n");
}
```

**JNI Use Cases:** OS-level operations, hardware access, legacy C/C++ libraries, performance-critical code.

> ⚠️ **Best Practice:** Minimize JNI usage — it's complex, error-prone, and bypasses Java safety guarantees.

---

# 7.2 Garbage Collection

## Object Lifecycle & Reachability

An object is considered **live** (reachable) if it can be accessed by any path from a **GC Root**. If not reachable — it's **garbage** and can be collected.

```java
public class ObjectLifecycle {
    public static void main(String[] args) {
        // obj1 is reachable — reference held by local var
        String obj1 = new String("alive");
        
        // Create object, but immediately lose reference
        new StringBuilder("orphan"); // Immediately eligible for GC
        
        // Circular references — both are garbage if nothing else references them
        Object a = new Object();
        Object b = new Object();
        a = b = null; // Both eligible for GC (JVM handles cycles, unlike reference counting)
    }
}
```

---

## GC Roots

**GC Roots** are the starting points of the reachability analysis. An object is alive if it's reachable from any GC Root.

**GC Roots include:**
1. **Local variables** in active stack frames (all threads)
2. **Static fields** of loaded classes
3. **JNI references** from native code
4. **Active thread objects** themselves
5. **System class loader** and loaded classes
6. **Synchronization monitors** (objects used as locks)

```java
public class GCRootsDemo {
    
    static Object staticRoot = new Object(); // ← GC Root (static field)
    
    public void method() {
        Object localRoot = new Object();     // ← GC Root (local variable while method runs)
        Object nonRoot = new Object();
        nonRoot = null;                      // ← No longer a GC Root — eligible for GC
        
        // localRoot stays alive until method returns
    } // ← localRoot is no longer a root after this
}
```

---

## Mark-and-Sweep Algorithm

The fundamental GC algorithm (all modern GCs are variations of this):

```
Phase 1 — MARK:
  Start from all GC Roots
  Traverse all object references (graph traversal)
  Mark every reachable object as "alive"

Phase 2 — SWEEP:
  Scan the entire heap
  Any object NOT marked = garbage → free its memory

Phase 3 — COMPACT (in some GCs):
  Move live objects together to eliminate fragmentation
  Update all references to point to new locations
```

```
Before GC:
[Root A] → [Obj 1] → [Obj 2]
[Root B] → [Obj 3]
           [Obj 4]  ← No reference — GARBAGE
           [Obj 5] → [Obj 4] ← Obj 5 also unreachable — GARBAGE

After Mark: Obj1, 2, 3 marked ALIVE; Obj4, 5 not marked

After Sweep: Obj4 and Obj5 memory freed

After Compact: [Obj1][Obj2][Obj3][   free space   ]
```

**Problems with simple Mark-and-Sweep:**
- **Stop-The-World (STW):** All application threads must pause during GC
- **Fragmentation:** After sweep, free space is scattered — compaction solves this but is expensive

---

## Generational GC Theory

Based on the **Generational Hypothesis:** *"Most objects die young"*

In a typical Java application:
- ~80-98% of objects become garbage within a few milliseconds
- Long-lived objects (caches, singletons) are rare

**Design consequence:** Divide the heap into generations, collect the young generation frequently (cheap) and old generation rarely (expensive).

```
Heap:
┌────────────────────────────────────────────────────────┐
│         Young Generation (minor GC — frequent, fast)  │
│  ┌──────────────────┐  ┌───────────────┐              │
│  │    Eden Space    │  │  S0  │   S1   │              │
│  │  New objects     │  │ Surv │  Surv  │              │
│  │  born here       │  │ ivor │  ivor  │              │
│  └──────────────────┘  └───────────────┘              │
├────────────────────────────────────────────────────────┤
│         Old Generation / Tenured (major GC — rare)    │
│  ┌────────────────────────────────────────────────┐   │
│  │  Long-lived objects promoted from Young Gen    │   │
│  └────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────┘
```

### Object Promotion Flow

```
Step 1: New objects created in Eden
Step 2: Eden fills up → Minor GC triggered
Step 3: Reachable Eden objects copied to S0 (age = 1)
Step 4: Eden and old S1 cleared
Step 5: Next Minor GC: S0 + Eden survivors → S1 (age++)
Step 6: Objects reaching age threshold (default 15) → promoted to Old Gen
Step 7: Old Gen fills → Major/Full GC
```

```java
// Demonstrating object aging
public class ObjectAging {
    public static void main(String[] args) throws InterruptedException {
        // These will be short-lived (die in Eden or S0/S1)
        for (int i = 0; i < 100_000; i++) {
            String temp = "temp" + i; // temp goes out of scope immediately
        }
        
        // This will be long-lived (promoted to Old Gen)
        List<String> longLived = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            longLived.add("persistent" + i);
        }
        
        Thread.sleep(5000); // longLived still referenced — stays alive
        System.out.println(longLived.size());
    }
}
```

---

## GC Algorithms

### 1. Serial GC

```bash
-XX:+UseSerialGC
```

- **Single-threaded** for both Minor and Major GC
- **Stop-The-World** during GC
- **Best for:** Small apps, single-core, minimal heap

```
Application Threads: ████████░░░░░░░░████████░░░░░░░░
GC Thread:                   ████████        ████████
                        STW pause      STW pause
```

### 2. Parallel GC

```bash
-XX:+UseParallelGC
-XX:ParallelGCThreads=8  # Number of GC threads
```

- **Multiple threads** for GC (parallel)
- Still **Stop-The-World** but pauses are shorter (more threads = faster GC work)
- **Best for:** Batch processing, throughput-focused apps

```
Application Threads: ████████░░░░████████░░░░
GC Threads:                  ████             ████
                         (4 parallel GC threads)
```

### 3. CMS (Concurrent Mark-Sweep) — Deprecated Java 14

```bash
-XX:+UseConcMarkSweepGC  # Removed in Java 14
```

- **Concurrent marking** — most GC work done alongside application threads
- **Lower pause times** than Parallel GC
- **Drawback:** Fragmentation (no compaction), higher CPU overhead, complex
- **Deprecated in Java 9, removed in Java 14** — replaced by G1

### 4. G1 GC (Garbage-First) — Default since Java 9

```bash
-XX:+UseG1GC  # Default since Java 9, no need to set explicitly
-XX:MaxGCPauseMillis=200    # Target max pause time (milliseconds)
-XX:G1HeapRegionSize=16m    # Region size (1MB-32MB, must be power of 2)
```

**G1 is fundamentally different** — it doesn't use fixed Young/Old spaces. Instead, it divides the heap into equal-sized **regions** (~2048 regions by default).

```
G1 Heap (regions, each ~1-32MB):
┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
│E │E │S │O │O │E │H │O │E │S │O │  │
└──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
E=Eden  S=Survivor  O=Old  H=Humongous(large objects)  _=Free

Regions are dynamically assigned — no fixed boundaries!
```

**G1 GC Process:**

1. **Young GC:** Collect only Eden + Survivor regions (STW, parallel)
2. **Concurrent Marking:** Mark live objects concurrently (low pause)
3. **Mixed GC:** Collect Young + some Old regions (prioritizes regions with most garbage = "Garbage First")
4. **Full GC (fallback):** Rare, STW — when concurrent collection can't keep up

```java
// Large objects (> G1HeapRegionSize/2) go directly to Humongous regions
byte[] bigArray = new byte[20 * 1024 * 1024]; // 20MB — goes to Humongous
```

**G1 vs Parallel GC Trade-off:**

| | Parallel GC | G1 GC |
|---|---|---|
| **Throughput** | Higher | Slightly lower |
| **Pause time** | Unpredictable, longer | Predictable, configurable |
| **Best for** | Batch jobs | Interactive apps, large heaps |

### 5. ZGC — Java 15+ (Stable)

```bash
-XX:+UseZGC
-XX:SoftMaxHeapSize=32g  # ZGC target heap size
```

- **Goal: Sub-millisecond GC pauses** regardless of heap size (even terabyte heaps)
- Almost entirely **concurrent** — application barely pauses
- Works with **colored pointers** (stores GC metadata in pointer bits)
- **Load barriers** — intercepts object access to do GC work during normal execution

```
Application Threads: ██████████████████████████████████
ZGC Threads:         ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
(concurrent — barely any STW pauses!)
STW pauses: <1ms
```

**Trade-off:** ~15-20% more CPU usage due to concurrent work.

### 6. Shenandoah GC — Red Hat

```bash
-XX:+UseShenandoahGC
```

- Similar goals to ZGC — ultra-low pause times
- Uses **concurrent compaction** — compacts heap while app runs
- Available in OpenJDK builds

### 7. Epsilon GC — No-op GC

```bash
-XX:+UseEpsilonGC
```

- **Does NOT collect garbage** — ever!
- Memory grows until JVM crashes with `OutOfMemoryError`
- **Use cases:** Performance testing (isolate GC overhead), short-lived scripts where GC would never trigger anyway, testing GC-related code

---

## GC Tuning Flags

```bash
# --- Heap Sizing ---
-Xms512m                    # Initial heap size (start size)
-Xmx2g                      # Maximum heap size
-Xmn512m                    # Young generation size (for Parallel GC)
-XX:NewRatio=2              # Old/Young ratio (default 2 = 2/3 Old, 1/3 Young)
-XX:SurvivorRatio=8         # Eden/Survivor ratio (default 8 = 8 Eden, 1 S0, 1 S1)

# --- GC Algorithm Selection ---
-XX:+UseSerialGC
-XX:+UseParallelGC
-XX:+UseG1GC                # Default Java 9+
-XX:+UseZGC                 # Java 15+
-XX:+UseShenandoahGC

# --- G1 Specific ---
-XX:MaxGCPauseMillis=200    # Target pause time (G1 tries to meet this)
-XX:G1HeapRegionSize=16m    # Region size
-XX:G1NewSizePercent=20     # Min % of heap for young gen
-XX:G1MaxNewSizePercent=60  # Max % of heap for young gen

# --- GC Logging (Java 9+) ---
-Xlog:gc                    # Basic GC logging
-Xlog:gc*                   # Verbose GC logging
-Xlog:gc*:file=gc.log       # Log to file

# --- Tenuring ---
-XX:MaxTenuringThreshold=15  # Age before promotion to Old Gen (default 15)
-XX:InitialTenuringThreshold=7

# --- Metaspace ---
-XX:MetaspaceSize=128m
-XX:MaxMetaspaceSize=512m

# --- GC Overhead Limit ---
-XX:GCTimeRatio=99          # Max % of time spent in GC (default 99 = 1% in GC)
```

**Best Practice for Heap Sizing:**
```bash
# Production recommendation: set -Xms == -Xmx to avoid heap resizing overhead
java -Xms4g -Xmx4g -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -jar app.jar
```

---

## Minor GC vs Major GC vs Full GC

| Type | Collects | Frequency | STW Pause | Trigger |
|---|---|---|---|---|
| **Minor GC** | Young Gen only | Very frequent | Short (ms) | Eden full |
| **Major GC** | Old Gen | Occasional | Longer | Old Gen full |
| **Full GC** | Entire heap + Metaspace | Rare (bad sign) | Longest | Old Gen full + Metaspace full / explicit `System.gc()` |

```java
// Never call this in production — triggers Full GC
System.gc();  // BAD — only a hint, but usually causes a Full GC

// You can disable it:
// -XX:+DisableExplicitGC
```

**GC Log Example (G1, Java 9+ format):**
```
[2.347s][info][gc] GC(0) Pause Young (Normal) (G1 Evacuation Pause) 51M->18M(256M) 4.320ms
[5.123s][info][gc] GC(1) Pause Young (Normal) (G1 Evacuation Pause) 85M->22M(256M) 3.891ms
[12.456s][info][gc] GC(5) Pause Full (System.gc()) 124M->45M(256M) 234.567ms
```

Reading this: `GC(0)` = first GC event, `51M->18M(256M)` = heap went from 51MB to 18MB of a 256MB heap, `4.320ms` = pause time.

---

## Memory Leaks

Java has GC, so why do memory leaks happen? Because **GC only collects unreachable objects**. If you hold references to objects you no longer need, GC can't collect them.

### Common Memory Leak Patterns

**1. Static Collections Growing Forever**

```java
// BAD — Static cache never cleared
public class CacheManager {
    // Static field = GC Root = cache items can never be GC'd
    private static final Map<String, byte[]> cache = new HashMap<>();
    
    public static void addToCache(String key, byte[] data) {
        cache.put(key, data); // Data accumulates forever!
    }
}

// FIX — Use bounded cache or WeakReference values
private static final Map<String, WeakReference<byte[]>> cache = new HashMap<>();
// Or use Caffeine / Guava Cache with size/time limits
```

**2. Listeners Never Removed**

```java
// BAD — Adding listeners without removing them
public class EventSource {
    private List<EventListener> listeners = new ArrayList<>();
    
    public void addListener(EventListener l) { listeners.add(l); }
    // No removeListener! Objects that registered are held forever.
}

// FIX — Always provide and call removeListener
public void removeListener(EventListener l) { listeners.remove(l); }
```

**3. ThreadLocal Not Cleaned Up**

```java
// BAD — ThreadLocal in thread pools (threads are reused, ThreadLocal survives!)
public class RequestContext {
    private static ThreadLocal<UserSession> session = new ThreadLocal<>();
    
    public static void setSession(UserSession s) { session.set(s); }
    // Missing: session.remove() after request completes!
}

// FIX — Always clean up in finally block
try {
    RequestContext.setSession(new UserSession());
    processRequest();
} finally {
    RequestContext.remove(); // CRITICAL for thread pools
}
```

**4. Inner Class Holding Outer Class Reference**

```java
// BAD — Non-static inner class implicitly holds reference to outer class
public class Activity {
    private byte[] data = new byte[1000000]; // 1MB
    
    // This inner class holds an implicit reference to Activity!
    class DataProcessor implements Runnable {
        @Override
        public void run() { /* uses data */ }
    }
    
    void start() {
        // Even if Activity is done, DataProcessor keeps it alive!
        new Thread(new DataProcessor()).start();
    }
}

// FIX — Use static inner class
static class DataProcessor implements Runnable { ... }
```

**5. Unclosed Resources**

```java
// BAD — If exception thrown between open and close, resource leaks
Connection conn = dataSource.getConnection();
PreparedStatement ps = conn.prepareStatement(sql);
ResultSet rs = ps.executeQuery(); // If this throws, conn and ps never closed!
conn.close();

// FIX — try-with-resources
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql);
     ResultSet rs = ps.executeQuery()) {
    // Auto-closed on exit, even if exception
}
```

---

## Reference Types

Java provides **4 reference strength levels** to control how GC treats objects.

```java
import java.lang.ref.*;

// 1. STRONG Reference (default) — object NEVER collected while reference exists
String strong = new String("strong"); // Won't be GC'd until strong = null

// 2. SOFT Reference — collected only when JVM NEEDS memory (out-of-memory situations)
// Great for memory-sensitive caches
SoftReference<byte[]> softRef = new SoftReference<>(new byte[1024]);
byte[] data = softRef.get(); // Returns null if GC'd
if (data == null) {
    // Recreate the data — cache miss
    data = loadDataFromDisk();
}

// 3. WEAK Reference — collected at NEXT GC cycle (regardless of memory)
// Used in WeakHashMap — keys are weakly referenced
WeakReference<Object> weakRef = new WeakReference<>(new Object());
Object obj = weakRef.get(); // Might return null anytime after next GC

// 4. PHANTOM Reference — collected AFTER finalization
// Used for cleanup actions, replacement for finalize()
ReferenceQueue<Object> queue = new ReferenceQueue<>();
PhantomReference<Object> phantomRef = new PhantomReference<>(new Object(), queue);
// phantomRef.get() ALWAYS returns null — used for tracking GC, not accessing object
```

### WeakHashMap — Practical Use

```java
// Keys are weakly referenced — entry removed when key has no strong reference
WeakHashMap<ImageKey, BufferedImage> imageCache = new WeakHashMap<>();

ImageKey key = new ImageKey("logo.png");
imageCache.put(key, loadImage("logo.png"));

// If key becomes unreachable (no other strong reference to it),
// the entry is automatically removed from the map!
key = null; // Now the entry can be GC'd
System.gc();
System.out.println(imageCache.size()); // May be 0 after GC
```

**Reference Summary:**

| Reference | Collected When | Use Case |
|---|---|---|
| Strong | Never (while referenced) | Default — everything |
| Soft | Low memory | Image/object caches |
| Weak | Next GC | Canonical maps, `WeakHashMap` |
| Phantom | After finalization | Resource cleanup, tracking |

---

# 7.3 JVM Performance Tuning

## Heap Sizing Strategy

```bash
# RULE: -Xms == -Xmx in production (prevents resizing pauses)
# RULE: Max heap = total RAM - OS - other JVMs - off-heap memory
# RULE: Leave at least 25-30% headroom before Full GC

# Example for 16GB server with one JVM:
# OS: ~1GB, Others: ~1GB, Off-heap: ~2GB
# Heap: 16 - 1 - 1 - 2 = 12GB, leave 30% headroom → use 8-10GB
java -Xms8g -Xmx8g -jar app.jar
```

## GC Pause Time Goals

```bash
# G1 — Set a pause target
-XX:MaxGCPauseMillis=100     # 100ms target (G1 will try to meet this)
                              # Lower = less work per GC = more frequent GCs
                              # Higher = more work per GC = less frequent GCs

# SLA Example: web app with 99th percentile latency requirement of 200ms
# Set MaxGCPauseMillis=100 to leave buffer
```

## JIT Compilation Tuning

```bash
# Control when JIT kicks in
-XX:CompileThreshold=10000   # Calls before JIT compiles (default 10000)
                              # Lower = JIT compiles sooner (faster peak but longer warmup)

# Force inline small methods
-XX:MaxInlineSize=35         # Max bytecodes to inline (default 35)
-XX:MaxFreqInlineSize=325    # Max for frequently called methods (default 325)

# Print JIT decisions
-XX:+PrintCompilation        # See what JIT is compiling
-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining  # See inlining decisions
```

## Escape Analysis

The JVM can detect if an object never "escapes" a method (not stored in heap fields, not returned, not passed to other threads). If so, it can allocate on the **stack** instead of heap — no GC overhead!

```java
// JVM may allocate this on stack (not heap) via escape analysis
public long sumSquares(int n) {
    // Point never escapes this method
    Point p = new Point(0, 0); // Might be stack-allocated!
    long sum = 0;
    for (int i = 0; i < n; i++) {
        p.x = i;
        p.y = i;
        sum += p.x * p.x + p.y * p.y;
    }
    return sum; // p is never returned or stored externally
}

// Can't escape-analyze this — p is stored in a field
private Point storedPoint;
public void storePoint() {
    storedPoint = new Point(1, 2); // MUST be on heap
}
```

```bash
-XX:+DoEscapeAnalysis       # Enable (default on)
-XX:-DoEscapeAnalysis       # Disable (for comparison)
```

## String Deduplication (G1 Only)

```bash
# G1 can find String objects with identical char arrays and make them share the array
-XX:+UseStringDeduplication   # Enable (Java 8u20+, G1 only)
-XX:+PrintStringDeduplicationStatistics  # See deduplication stats
```

```java
// These two String objects have different identity but same value
// String deduplication will make them share the underlying char[]
String a = new String("hello world");
String b = new String("hello world");
// After deduplication: both point to same char[] → saves memory
```

## Object Pooling

```java
// For expensive-to-create objects, use object pools
// Apache Commons Pool example:
import org.apache.commons.pool2.impl.GenericObjectPool;

GenericObjectPool<ExpensiveConnection> pool = 
    new GenericObjectPool<>(new ConnectionFactory());
pool.setMaxTotal(10);
pool.setMinIdle(2);

ExpensiveConnection conn = pool.borrowObject();
try {
    conn.doWork();
} finally {
    pool.returnObject(conn); // Return to pool, not GC'd
}
```

## Compressed OOPs

```bash
# OOP = Ordinary Object Pointer
# With 64-bit JVM: each reference = 8 bytes
# Compressed OOPs: encode references in 32 bits (4 bytes) — saves ~30% heap!
# Enabled automatically when -Xmx <= ~32GB

-XX:+UseCompressedOops       # Default when heap < ~32GB
-XX:-UseCompressedOops       # Disable (needed for heap > ~32GB)

# Check if enabled:
java -XX:+PrintFlagsFinal -version | grep UseCompressedOops
```

## Key Performance Best Practices

```java
// 1. Use StringBuilder for string concatenation in loops
// BAD:
String result = "";
for (int i = 0; i < 1000; i++) {
    result += i;  // Creates 1000 String objects!
}

// GOOD:
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append(i);  // One object, mutated in place
}
String result = sb.toString();

// 2. Avoid autoboxing in hot paths
// BAD:
List<Integer> numbers = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    numbers.add(i);  // Boxes every int → 1M Integer objects on heap
}

// BETTER: Use primitive collections (Eclipse Collections, Trove, etc.)
// Or use arrays if fixed-size

// 3. Pre-size collections
// BAD:
List<String> list = new ArrayList<>();  // Default capacity 10, resizes many times
for (int i = 0; i < 10000; i++) list.add("item" + i);

// GOOD:
List<String> list = new ArrayList<>(10000);  // Pre-sized, no resizing

// 4. Use streams carefully — parallel is not always faster
// Parallel is good for: large data (>10k items), CPU-bound, no shared state
// Sequential is better for: small data, IO-bound, stateful operations
```

---

# 7.4 Profiling & Monitoring Tools

## JConsole

Built-in, graphical JVM monitoring tool.

```bash
# Start your app with JMX enabled
java -Dcom.sun.management.jmxremote \
     -Dcom.sun.management.jmxremote.port=9090 \
     -Dcom.sun.management.jmxremote.authenticate=false \
     -Dcom.sun.management.jmxremote.ssl=false \
     -jar app.jar

# Open JConsole
jconsole
# Connect to localhost:9090
```

**JConsole shows:** Heap usage, threads, classes loaded, CPU, MBeans.

**Best for:** Quick health checks, learning JVM metrics.

## VisualVM

```bash
# Download from https://visualvm.github.io/
# Or included with older JDKs at $JAVA_HOME/bin/visualvm

# Attach to running process or connect via JMX
```

**VisualVM features:**
- **Heap dump analysis** — snapshot of all live objects
- **Thread dump** — see what every thread is doing
- **CPU profiling** — find hotspot methods
- **Memory profiling** — find memory leaks
- **GC activity graph**

```java
// Trigger a heap dump programmatically
import com.sun.management.HotSpotDiagnosticMXBean;
import java.lang.management.ManagementFactory;

void dumpHeap(String filePath) throws Exception {
    MBeanServer server = ManagementFactory.getPlatformMBeanServer();
    HotSpotDiagnosticMXBean bean = ManagementFactory.newPlatformMXBeanProxy(
        server, "com.sun.management:type=HotSpotDiagnostic", 
        HotSpotDiagnosticMXBean.class);
    bean.dumpHeap(filePath, true); // true = live objects only
}
```

## Java Mission Control (JMC) + Java Flight Recorder (JFR)

JFR is a **low-overhead production profiler** built into the JVM. Overhead is typically < 1%.

```bash
# Start JFR recording from command line
java -XX:+FlightRecorder \
     -XX:StartFlightRecording=duration=60s,filename=recording.jfr \
     -jar app.jar

# Or start/stop via jcmd on a running process
jcmd <pid> JFR.start duration=60s filename=recording.jfr
jcmd <pid> JFR.stop

# View with JMC
jmc
```

```java
// JFR custom events (Java 9+)
import jdk.jfr.*;

@Label("Database Query")
@Category("Database")
@StackTrace(true)
public class QueryEvent extends Event {
    @Label("SQL Query")
    String sql;
    
    @Label("Duration (ms)")
    long durationMs;
}

// Usage
QueryEvent event = new QueryEvent();
event.begin();
try {
    event.sql = "SELECT * FROM users WHERE id = ?";
    executeQuery();
} finally {
    event.durationMs = event.getDuration().toMillis();
    event.commit(); // Records event to JFR stream
}
```

## Command-Line Diagnostic Tools

```bash
# jps — List Java processes
jps -l   # Show main class names
# Output: 12345 com.example.MyApp

# jstack — Thread dump (what is every thread doing right now?)
jstack <pid>
jstack -l <pid>   # With lock info
# Use to diagnose: deadlocks, high CPU threads, thread leaks

# jmap — Heap information
jmap -heap <pid>              # Heap summary
jmap -histo <pid>             # Object histogram (top objects by count/size)
jmap -histo:live <pid>        # Only live objects (triggers GC first)
jmap -dump:live,format=b,file=heap.hprof <pid>  # Full heap dump

# jstat — GC statistics in real-time
jstat -gc <pid> 1000          # GC stats every 1 second
jstat -gcutil <pid> 1000      # GC utilization percentages
# Output columns: S0C S1C S0U S1U EC EU OC OU MC MU YGC YGCT FGC FGCT GCT

# jcmd — All-in-one (modern replacement for jmap/jstack)
jcmd <pid> help               # List available commands
jcmd <pid> VM.flags           # JVM flags
jcmd <pid> VM.system_properties
jcmd <pid> Thread.print       # Thread dump
jcmd <pid> GC.heap_info       # Heap info
jcmd <pid> GC.run             # Trigger GC
jcmd <pid> VM.native_memory   # Native memory usage
```

## Reading Thread Dumps with jstack

```bash
jstack 12345 > thread-dump.txt
```

```
# Sample thread dump sections:
"main" #1 prio=5 os_prio=0 cpu=234ms elapsed=5.32s tid=0x7f000c000e00 nid=0x1234 waiting on condition [0x7fff5fbff000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(java.base@17.0.1/Native Method)
        at com.example.MyApp.main(MyApp.java:15)

"pool-1-thread-1" #12 prio=5 os_prio=0 cpu=89ms elapsed=4.21s tid=0x7f000c100e00 nid=0x1235 waiting for monitor entry [0x7fff5faff000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.example.Resource.doWork(Resource.java:42)
        - waiting to lock <0x000000076b234a98> (a com.example.Resource)
        at com.example.Worker.run(Worker.java:15)
```

**Analyzing thread dumps:**
- `BLOCKED` — thread waiting for a lock → potential deadlock
- `WAITING` / `TIMED_WAITING` — thread waiting for signal
- All threads in `BLOCKED` waiting for same lock → **deadlock detected**

```bash
# jstack can detect deadlocks automatically
jstack <pid> | grep -A 10 "deadlock"
```

## Heap Dump Analysis with Eclipse MAT

```bash
# Generate heap dump
jcmd <pid> GC.heap_dump /tmp/heap.hprof

# Open in Eclipse Memory Analyzer Tool (MAT)
# Download from: https://eclipse.dev/mat/
```

**In MAT, look for:**
- **Dominator Tree** — objects holding the most retained heap
- **Leak Suspects** — automatic leak detection
- **OQL (Object Query Language)** — query heap like a database

```sql
-- OQL example: Find all String objects longer than 1000 chars
SELECT * FROM java.lang.String s WHERE s.value.length > 1000

-- Find objects of a specific class
SELECT * FROM com.example.MyBigObject
```

## async-profiler

Low-overhead, production-safe CPU and memory profiler.

```bash
# Download from https://github.com/async-profiler/async-profiler

# CPU profiling for 30 seconds
./profiler.sh -d 30 -f flamegraph.html <pid>

# Memory profiling (allocation profiling)
./profiler.sh -d 30 -e alloc -f alloc.html <pid>

# Lock profiling
./profiler.sh -d 30 -e lock -f lock.html <pid>
```

The output is a **flame graph** — a visual representation of where CPU time is spent.

```
Flame Graph (wider = more time spent):
     ████████ main()
   ██████████████ processOrders()
 ███ calculateTax()  ████████ lookupProduct()  ████ updateDB()
      taxEngine()         ██ dbQuery() ← WIDE = HOT SPOT
```

## JVM Monitoring Summary

| Tool | Purpose | Overhead | Production Safe? |
|---|---|---|---|
| JConsole | Basic monitoring | Low | Yes |
| VisualVM | Profiling + heap analysis | Medium-High | No (affects performance) |
| JMC + JFR | Production profiling | Very Low (<1%) | Yes |
| jstack | Thread dump | Very Low | Yes |
| jmap -histo | Object counts | Low | Yes |
| jmap -dump | Full heap dump | High (STW) | Use carefully |
| jstat | GC stats | Very Low | Yes |
| jcmd | All-in-one | Varies | Usually |
| async-profiler | CPU/Memory flame graphs | Low | Yes |
| MAT | Heap dump analysis | N/A (offline) | N/A |

---

# 7.5 JVM Languages Interop

The JVM runs many languages — all compile to the same bytecode format Java does.

## Kotlin on JVM

Kotlin is **100% Java-interoperable** — you can call Java from Kotlin and Kotlin from Java seamlessly.

```kotlin
// Kotlin class
data class User(val name: String, val age: Int) {
    fun greet() = "Hello, $name!"
}
```

```java
// Calling Kotlin from Java
User user = new User("Alice", 30);      // Kotlin data class
System.out.println(user.getName());     // Kotlin properties become getters
System.out.println(user.greet());       // Regular methods work normally
System.out.println(user.component1());  // Kotlin destructuring component
```

```kotlin
// Calling Java from Kotlin
import java.util.ArrayList

val list = ArrayList<String>()  // Java class
list.add("Hello")               // Java method
val upper = list[0].uppercase() // Kotlin extension on Java String
```

**Key Kotlin-Java interop notes:**
- Kotlin `val` → generates getter (no setter)
- Kotlin `var` → generates getter + setter
- `@JvmStatic` annotation needed for Kotlin `object` methods to be called as Java static methods
- `@JvmField` exposes Kotlin properties as public Java fields
- Kotlin null safety: Java types become "platform types" in Kotlin (nullable warning)

```kotlin
// @JvmStatic makes this callable as Utils.compute() from Java
object Utils {
    @JvmStatic
    fun compute(x: Int): Int = x * 2
}
```

## Groovy for Scripting

Groovy is dynamically typed, very concise, great for scripting and DSLs.

```groovy
// Groovy — concise, dynamic
def numbers = [1, 2, 3, 4, 5]
def evens = numbers.findAll { it % 2 == 0 }
println evens  // [2, 4]

// Calling Java from Groovy — seamless
import java.util.HashMap
def map = new HashMap<String, Integer>()
map.put("one", 1)
println map["one"]  // 1 (Groovy subscript notation)
```

```java
// Calling Groovy from Java
import groovy.lang.GroovyShell;
import groovy.lang.Script;

GroovyShell shell = new GroovyShell();
Script script = shell.parse("return 2 + 2");
Object result = script.run(); // Returns 4
```

**Groovy use cases:** Gradle build scripts, Jenkins pipelines, scripting, testing with Spock framework.

## Scala for Functional Programming

Scala blends OOP and functional programming. Very popular for data engineering (Apache Spark is written in Scala).

```scala
// Scala — functional + OOP
case class Person(name: String, age: Int)

val people = List(Person("Alice", 30), Person("Bob", 25))
val adults = people.filter(_.age >= 18).map(_.name)
println(adults)  // List(Alice, Bob)
```

```java
// Calling Scala from Java (Scala compiles to Java bytecode)
// Scala case class generates Java-compatible getters
Person p = new Person("Alice", 30);
System.out.println(p.name()); // Note: Scala uses method-style, not getBeans style
System.out.println(p.age());
```

**Scala-Java interop tips:**
- Scala collections and Java collections are different — use `JavaConverters`
- Scala `Option[T]` ≠ Java `Optional<T>`
- Scala traits compile to Java interfaces (with default methods for implementations)

## Clojure

Clojure is a Lisp dialect on the JVM — purely functional, immutable-first.

```clojure
;; Clojure — functional, immutable
(def numbers [1 2 3 4 5])
(def evens (filter even? numbers))
(println (reduce + evens))  ; 6

;; Calling Java from Clojure
(import 'java.util.ArrayList)
(def list (ArrayList.))
(.add list "hello")
(println list) ; [hello]
```

## JVM Language Comparison

| Language | Paradigm | Typing | Strengths |
|---|---|---|---|
| **Java** | OOP | Static | Enterprise, ecosystem, readability |
| **Kotlin** | OOP + FP | Static | Android, null safety, conciseness |
| **Scala** | OOP + FP | Static | Big data (Spark), type system, functional |
| **Groovy** | OOP | Dynamic | Scripting, DSLs, Gradle, Spock |
| **Clojure** | Functional | Dynamic | Immutability, concurrency, simplicity |

---

# 📋 Quick Reference Cheat Sheet

## JVM Architecture Summary

```
JVM
├── ClassLoader Subsystem
│   ├── Bootstrap → Extension → Application → Custom
│   └── Parent Delegation Model
├── Runtime Data Areas
│   ├── Shared: Method Area (Metaspace), Heap (Eden, S0, S1, Old Gen)
│   └── Per-Thread: Stack (Frames), PC Register, Native Method Stack
└── Execution Engine
    ├── Interpreter (start fast, run slow)
    ├── JIT C1 (compile fast, basic optimization)
    ├── JIT C2 (compile slow, aggressive optimization)
    └── AOT / GraalVM Native (compile ahead of time)
```

## GC Algorithm Selection Guide

```
Which GC should I use?

App type?
├── Small/single-core app → Serial GC (-XX:+UseSerialGC)
├── Batch/throughput priority → Parallel GC (-XX:+UseParallelGC)
├── Interactive, large heap (Java 9-20) → G1 GC (default)
├── Ultra-low latency (Java 15+) → ZGC (-XX:+UseZGC)
├── Ultra-low latency (OpenJDK) → Shenandoah (-XX:+UseShenandoahGC)
└── Performance testing → Epsilon (-XX:+UseEpsilonGC)
```

## Essential JVM Flags

```bash
# Must-know production flags:
java \
  -Xms4g -Xmx4g \                           # Fix heap size
  -XX:+UseG1GC \                             # Use G1 (or UseZGC for Java 15+)
  -XX:MaxGCPauseMillis=200 \                 # G1 pause target
  -XX:MetaspaceSize=256m \                   # Metaspace initial
  -XX:MaxMetaspaceSize=512m \                # Metaspace limit
  -Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=20m \  # GC logging
  -XX:+HeapDumpOnOutOfMemoryError \          # Auto heap dump on OOM
  -XX:HeapDumpPath=/tmp/heapdump.hprof \     # Dump location
  -jar app.jar
```

## Common Memory Problems & Solutions

| Problem | Symptom | Diagnosis | Solution |
|---|---|---|---|
| Memory leak | Heap grows until OOM | Heap dump → MAT | Fix strong references |
| Too many Full GCs | High latency spikes | GC logs → `FGC` count | Tune heap, fix leaks |
| High GC overhead | `GCOverheadLimitExceeded` | GC logs | Increase heap or fix leaks |
| Thread leak | Thread count grows | jstack → thread count | Fix thread cleanup |
| Metaspace OOM | `OutOfMemoryError: Metaspace` | `-XX:+PrintClassHistogram` | Set `MaxMetaspaceSize`, check ClassLoader leaks |
| StackOverflow | `StackOverflowError` | Stack trace depth | Fix recursion, increase `-Xss` |

## Reference Strength Quick Guide

```java
// Strong  — never GC'd while referenced   → default
// Soft    — GC'd on low memory            → SoftReference<T>   → caches
// Weak    — GC'd on next GC               → WeakReference<T>   → WeakHashMap
// Phantom — GC'd after finalization       → PhantomReference<T> → cleanup
```

---

## 🏆 Key Takeaways for Phase 7

1. **The JVM is not magic** — it's a well-designed machine with predictable behavior. Understanding it lets you write better code and diagnose problems faster.

2. **GC pauses are your enemy in latency-sensitive apps** — choose the right GC algorithm, tune it, and design your code to minimize object allocation.

3. **Profile first, optimize second** — never guess where your performance problem is. Use JFR, async-profiler, or VisualVM to find real hotspots.

4. **Memory leaks in Java are reference leaks** — the GC is fine; you're holding references you didn't mean to. Static collections, listeners, and ThreadLocals are the usual suspects.

5. **Understand the difference between heap and metaspace** — Java 8+ moved class metadata out of heap into Metaspace. Set `MaxMetaspaceSize` in production.

6. **JFR is your production profiler of choice** — < 1% overhead, built in, and reveals everything from GC events to custom business events.

7. **The JVM ecosystem is bigger than Java** — Kotlin, Scala, Groovy, and Clojure all run on the same JVM. Understanding the JVM helps you work with all of them.

---

*Phase 7 complete. Proceed to Phase 8 — Design Patterns.*
