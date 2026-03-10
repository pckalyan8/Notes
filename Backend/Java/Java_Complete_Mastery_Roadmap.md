# ☕ Java Complete Mastery Roadmap
### From Zero to Software Lead — Every Topic in Order

---

## 📌 How to Use This Roadmap
- Follow the phases **in order** — each builds on the previous
- ✅ Mark topics as you complete them
- Each phase includes **what to learn**, not just topic names
- Estimated total learning time: **12–18 months** (dedicated study)

---

# 🧭 PHASE 0 — History & Ecosystem Understanding
*Before writing a single line of code, understand the WHY*

## 0.1 History of Java
- Origins at Sun Microsystems (1991) — James Gosling, Mike Sheridan, Patrick Naughton
- The "Green Project" and Oak language
- Java 1.0 release (1996) — "Write Once, Run Anywhere"
- Sun Microsystems to Oracle acquisition (2010)
- Open-source movement — OpenJDK
- Timeline of every major release: Java 1.0 → 1.1 → 1.2 → ... → Java 21 (LTS) → Java 22/23/24

## 0.2 Java Release Timeline & Features Per Version
| Version | Year | Key Features |
|---------|------|-------------|
| Java 1.0 | 1996 | Core language, Applets |
| Java 1.1 | 1997 | Inner classes, JDBC, RMI |
| Java 1.2 | 1998 | Swing, Collections Framework, JIT |
| Java 1.3 | 2000 | HotSpot JVM |
| Java 1.4 | 2002 | Assertions, NIO, Regex |
| Java 5 | 2004 | Generics, Annotations, Enums, Autoboxing, for-each |
| Java 6 | 2006 | Scripting API, Web Services |
| Java 7 | 2011 | try-with-resources, Diamond operator, NIO.2 |
| Java 8 (LTS) | 2014 | Lambdas, Streams, Optional, Date/Time API |
| Java 9 | 2017 | Modules (JPMS), JShell |
| Java 10 | 2018 | `var` keyword |
| Java 11 (LTS) | 2018 | HTTP Client, String methods |
| Java 12-13 | 2019 | Switch expressions (preview) |
| Java 14 | 2020 | Records (preview), Pattern matching (preview) |
| Java 15-16 | 2020-21 | Sealed classes, Text blocks |
| Java 17 (LTS) | 2021 | Sealed classes, Pattern matching stable |
| Java 18-20 | 2022-23 | Virtual Threads (preview), Structured Concurrency |
| Java 21 (LTS) | 2023 | Virtual Threads stable, Record patterns, Sequenced Collections |
| Java 22-24 | 2024-25 | String Templates, Unnamed Variables, Unnamed Patterns |

## 0.3 Java Editions & Platforms
- **Java SE** (Standard Edition) — Core platform
- **Java EE / Jakarta EE** (Enterprise Edition) — Web & enterprise
- **Java ME** (Micro Edition) — Embedded/mobile
- **JavaFX** — Desktop GUI platform
- Understanding LTS (Long-Term Support) vs. non-LTS releases
- Oracle JDK vs OpenJDK vs Amazon Corretto vs Adoptium (Eclipse Temurin)

## 0.4 The JVM Ecosystem
- JVM Languages: Kotlin, Scala, Groovy, Clojure
- Why the JVM matters
- JVM vs. native compilation (GraalVM Native Image)

---

# 🏗️ PHASE 1 — Setting Up & Developer Environment
*Get productive before you start coding*

## 1.1 JDK Installation & Management
- Downloading and installing JDK (Oracle JDK / OpenJDK)
- `JAVA_HOME` and `PATH` environment variables
- Managing multiple JDK versions — SDKMAN, jenv
- Verifying installation: `java -version`, `javac -version`

## 1.2 IDEs & Editors
- **IntelliJ IDEA** (recommended) — Community vs Ultimate
- **Eclipse** — History, workspace setup
- **VS Code** — Java extension pack
- IDE shortcuts, live templates, refactoring tools
- Debugging in IDE — breakpoints, watches, evaluate expression

## 1.3 Build Tools
- **Maven**
  - `pom.xml` structure
  - Maven lifecycle: validate, compile, test, package, install, deploy
  - Dependency management, repositories (Maven Central, local)
  - Maven plugins (Surefire, Shade, Compiler)
- **Gradle**
  - `build.gradle` (Groovy DSL) vs `build.gradle.kts` (Kotlin DSL)
  - Tasks, configurations, dependencies
  - Incremental builds and caching
- Maven vs Gradle comparison

## 1.4 Version Control
- Git fundamentals: init, clone, add, commit, push, pull
- Branching strategies: Git Flow, trunk-based development
- GitHub / GitLab / Bitbucket
- `.gitignore` for Java projects

## 1.5 First Java Program
- `Hello, World!` — anatomy of a Java file
- Compiling with `javac` and running with `java`
- Package structure and directory layout
- Understanding `.class` files and bytecode

---

# 📘 PHASE 2 — Java Language Fundamentals
*The absolute bedrock — master every detail*

## 2.1 Java Program Structure
- Class declaration syntax
- The `main()` method — signature and role
- Statements, expressions, and blocks
- Comments: single-line `//`, multi-line `/* */`, Javadoc `/** */`
- Semicolons and whitespace rules

## 2.2 Data Types & Variables
### Primitive Types
- `byte` (8-bit), `short` (16-bit), `int` (32-bit), `long` (64-bit)
- `float` (32-bit IEEE 754), `double` (64-bit IEEE 754)
- `char` (16-bit Unicode), `boolean` (true/false)
- Default values of primitives
- Numeric literals: decimal, hexadecimal (`0x`), binary (`0b`), octal (`0`)
- Underscore in numeric literals (`1_000_000`)

### Reference Types
- Objects and references
- `String` — immutable, String pool, `intern()`
- Arrays — declaration, initialization, multidimensional
- `null` reference

### Type Conversion
- Widening (implicit) conversion
- Narrowing (explicit) casting
- Integer promotion rules
- `instanceof` operator (and pattern matching — Java 16+)

## 2.3 Operators
- Arithmetic: `+`, `-`, `*`, `/`, `%`
- Assignment: `=`, `+=`, `-=`, `*=`, `/=`, `%=`
- Comparison: `==`, `!=`, `<`, `>`, `<=`, `>=`
- Logical: `&&`, `||`, `!`, `&`, `|`, `^`
- Bitwise: `&`, `|`, `^`, `~`, `<<`, `>>`, `>>>`
- Ternary: `condition ? a : b`
- `instanceof`
- Operator precedence and associativity

## 2.4 Control Flow
### Conditionals
- `if`, `if-else`, `if-else if-else`
- Nested if statements
- `switch` statement (traditional)
- `switch` expression (Java 14+) — arrow syntax, `yield`
- Pattern matching in switch (Java 21)

### Loops
- `for` loop — standard and enhanced (for-each)
- `while` loop
- `do-while` loop
- Nested loops
- `break` and `continue`
- Labeled break and continue

## 2.5 Methods
- Method declaration: access modifier, return type, name, parameters
- Method overloading — rules and resolution
- Passing by value vs. passing reference values
- Variable-length arguments (varargs) — `String... args`
- `return` statement
- Recursive methods and base cases
- Stack overflow — understanding recursion limits
- Method signatures and calling conventions

## 2.6 Strings (Deep Dive)
- `String` class — immutability
- String literal pool and `new String()`
- `==` vs `.equals()` vs `.compareTo()`
- Key methods: `length()`, `charAt()`, `substring()`, `indexOf()`, `contains()`, `startsWith()`, `endsWith()`, `replace()`, `trim()`, `strip()`, `toUpperCase()`, `toLowerCase()`, `split()`, `join()`, `formatted()`
- String concatenation and `+` operator performance
- `StringBuilder` — mutable, `append()`, `insert()`, `delete()`, `reverse()`
- `StringBuffer` — thread-safe version of StringBuilder
- `String.format()` and `printf()`
- Text Blocks (Java 15+) — multi-line strings with `"""`
- Regular expressions with `String.matches()`, `replaceAll()`
- `String.valueOf()`, `Integer.parseInt()`, conversions

## 2.7 Arrays
- Declaring and initializing arrays
- Array access and index bounds (`ArrayIndexOutOfBoundsException`)
- `array.length` property
- Multi-dimensional arrays (2D, 3D, jagged)
- `Arrays` utility class: `sort()`, `binarySearch()`, `fill()`, `copyOf()`, `copyOfRange()`, `equals()`, `toString()`
- Array vs ArrayList comparison
- Passing arrays to methods

## 2.8 Input & Output (Basic)
- `System.out.println()`, `print()`, `printf()`
- Reading from console: `Scanner` class
- `System.in`, `System.err`
- `BufferedReader` + `InputStreamReader` for faster input

---

# 🎯 PHASE 3 — Object-Oriented Programming (OOP)
*Java's core paradigm — understand it deeply*

## 3.1 Classes & Objects
- What is a class? Blueprint vs. instance
- Defining a class: fields, methods, constructors
- Creating objects with `new`
- `this` keyword — current instance reference
- Object references and memory (heap vs. stack)
- `null` and `NullPointerException`

## 3.2 Constructors
- Default (no-arg) constructor
- Parameterized constructors
- Constructor overloading
- Constructor chaining with `this()`
- Copy constructors
- Constructors vs methods — differences

## 3.3 Access Modifiers
- `public` — accessible everywhere
- `protected` — accessible in package + subclasses
- `default` (package-private) — accessible within package
- `private` — accessible only within the class
- Access modifier table across class hierarchy

## 3.4 Encapsulation
- Hiding internal state with `private` fields
- Getter and setter methods
- JavaBeans convention
- Benefits: security, maintainability, flexibility
- Defensive copying in getters/setters

## 3.5 Inheritance
- `extends` keyword — single inheritance in Java
- `super` keyword — accessing parent class members
- Constructor chaining with `super()`
- Method overriding — `@Override` annotation
- Rules for overriding: same signature, covariant return types
- `final` methods (cannot be overridden)
- `final` classes (cannot be extended)
- Multilevel inheritance
- Why Java doesn't support multiple class inheritance

## 3.6 Polymorphism
- **Compile-time (Static) Polymorphism** — method overloading
- **Runtime (Dynamic) Polymorphism** — method overriding + dynamic dispatch
- Upcasting and downcasting
- `instanceof` check before casting
- Pattern matching with `instanceof` (Java 16+)
- Polymorphism in action — reference type vs. object type

## 3.7 Abstraction
- **Abstract classes** — `abstract` keyword, cannot be instantiated
- Abstract methods — must be overridden by concrete subclasses
- Abstract class vs. concrete class
- **Interfaces** — complete abstraction contract
- `interface` keyword, `implements`
- Default methods in interfaces (Java 8+)
- Static methods in interfaces (Java 8+)
- Private methods in interfaces (Java 9+)
- Functional interfaces (Java 8+)
- Multiple interface implementation
- Interface vs. abstract class — when to use which

## 3.8 Interfaces (Deep Dive)
- Marker interfaces: `Serializable`, `Cloneable`, `RandomAccess`
- Comparable interface — `compareTo()`
- Comparator interface — external comparison logic
- Iterable and Iterator interfaces
- Interface default method conflicts and resolution
- Sealed interfaces (Java 17+)

## 3.9 Static Members
- `static` fields — class-level, shared across instances
- `static` methods — no `this` reference
- Static initializer blocks
- Static nested classes
- `static import`
- Singleton pattern using static

## 3.10 Inner Classes
- **Member inner class** — non-static inner class
- **Static nested class** — static inner class
- **Local inner class** — defined inside a method
- **Anonymous inner class** — one-time use, no name
- Accessing outer class members from inner class
- When to use each type

## 3.11 Enumerations (Enums)
- `enum` keyword — fixed set of constants
- Enum fields, methods, constructors
- Enum with abstract methods
- Built-in methods: `values()`, `valueOf()`, `name()`, `ordinal()`
- Enum in switch statements
- `EnumSet` and `EnumMap`
- Implementing interfaces in enums

## 3.12 Records (Java 16+)
- `record` keyword — immutable data carriers
- Auto-generated: constructor, getters, `equals()`, `hashCode()`, `toString()`
- Compact constructors
- Custom methods in records
- Records vs POJOs vs Lombok
- Records with interfaces and generics

## 3.13 Object Class — Root of All Classes
- `toString()` — default and overriding
- `equals()` — default (reference) and overriding (value)
- `hashCode()` — contract with `equals()`, consistent hashing
- `equals()` and `hashCode()` contract — must override together
- `clone()` — shallow vs. deep copy
- `getClass()`, `instanceof`
- `finalize()` — deprecated, alternatives
- `wait()`, `notify()`, `notifyAll()` — threading

## 3.14 SOLID Principles
- **S** — Single Responsibility Principle
- **O** — Open/Closed Principle
- **L** — Liskov Substitution Principle
- **I** — Interface Segregation Principle
- **D** — Dependency Inversion Principle
- Code examples for each principle
- Anti-patterns that violate SOLID

---

# 📦 PHASE 4 — Java Core APIs & Utilities

## 4.1 Collections Framework (Critical)
### Interfaces Hierarchy
- `Iterable` → `Collection` → `List`, `Set`, `Queue`, `Deque`
- `Map` (separate hierarchy)

### List Implementations
- `ArrayList` — dynamic array, O(1) random access, O(n) insert/delete
- `LinkedList` — doubly-linked, O(1) insert/delete at ends
- `Vector` (legacy), `Stack` (legacy)
- `CopyOnWriteArrayList` — thread-safe

### Set Implementations
- `HashSet` — hash table, O(1) operations, no order
- `LinkedHashSet` — insertion-order
- `TreeSet` — sorted order, O(log n), uses `Comparable`/`Comparator`
- `EnumSet` — highly optimized for enums

### Queue & Deque Implementations
- `PriorityQueue` — heap-based, sorted by priority
- `ArrayDeque` — efficient double-ended queue
- `LinkedList` — also implements Deque

### Map Implementations
- `HashMap` — hash table, O(1) avg, no order, allows null key
- `LinkedHashMap` — insertion/access order
- `TreeMap` — sorted keys, O(log n)
- `Hashtable` (legacy), `Properties`
- `EnumMap` — enum keys
- `WeakHashMap` — weak references
- `IdentityHashMap` — reference equality

### Utility Classes
- `Collections` — `sort()`, `binarySearch()`, `shuffle()`, `reverse()`, `unmodifiableList()`, `synchronizedList()`, `frequency()`, `disjoint()`, `nCopies()`
- `Arrays` — bridge between arrays and collections

### Algorithms & Complexity
- Time complexity of all operations per implementation
- Choosing the right collection for the use case
- Fail-fast vs. fail-safe iterators

### Java 9+ Factory Methods
- `List.of()`, `Set.of()`, `Map.of()`, `Map.entry()` — immutable
- `List.copyOf()`, `Map.copyOf()`

### Sequenced Collections (Java 21)
- `SequencedCollection`, `SequencedSet`, `SequencedMap`
- `getFirst()`, `getLast()`, `addFirst()`, `addLast()`, `reversed()`

## 4.2 Generics
- Why generics? Type safety without casting
- Generic classes: `class Box<T>`
- Generic methods: `<T> T method(T input)`
- Bounded type parameters: `<T extends Number>`, `<T super Integer>`
- Wildcards: `<?>`, `<? extends T>`, `<? super T>` (PECS rule)
- Type erasure — how generics work at runtime
- Reifiable vs. non-reifiable types
- Generic interfaces and inheritance
- Raw types and unchecked warnings

## 4.3 Exception Handling
### Exception Hierarchy
- `Throwable` → `Error` vs `Exception`
- `Exception` → `RuntimeException` (unchecked) vs checked exceptions
- Common checked: `IOException`, `SQLException`, `ClassNotFoundException`
- Common unchecked: `NullPointerException`, `ArrayIndexOutOfBoundsException`, `ClassCastException`, `IllegalArgumentException`, `IllegalStateException`, `UnsupportedOperationException`
- Errors: `OutOfMemoryError`, `StackOverflowError`, `AssertionError`

### Exception Handling Mechanics
- `try`, `catch`, `finally` blocks
- Multi-catch: `catch (IOException | SQLException e)`
- try-with-resources (Java 7+) — `AutoCloseable`
- Exception chaining — `initCause()`, `getCause()`
- Re-throwing exceptions
- `throws` declaration

### Custom Exceptions
- Extending `Exception` (checked) vs `RuntimeException` (unchecked)
- Best practices for custom exceptions
- Exception messages and constructors

### Best Practices
- Don't swallow exceptions
- Don't catch `Exception` or `Throwable` blindly
- Log exceptions properly
- Fail fast principle
- Exception vs. error codes

## 4.4 Date & Time API (Java 8+)
- Problems with old `Date` and `Calendar` classes
- `LocalDate` — date without time
- `LocalTime` — time without date
- `LocalDateTime` — date + time without timezone
- `ZonedDateTime` — date + time + timezone
- `ZoneId`, `ZoneOffset`
- `Instant` — machine time (epoch seconds)
- `Duration` — time-based amount
- `Period` — date-based amount
- `DateTimeFormatter` — formatting and parsing
- `TemporalAdjusters` — next Monday, last day of month
- `Clock` — for testing
- Legacy interop: `Date.toInstant()`, `Instant.toEpochMilli()`

## 4.5 Optional (Java 8+)
- `Optional<T>` — container for nullable values
- Creating: `Optional.of()`, `Optional.ofNullable()`, `Optional.empty()`
- Consuming: `isPresent()`, `get()`, `ifPresent()`, `ifPresentOrElse()`
- Transforming: `map()`, `flatMap()`, `filter()`
- Fallbacks: `orElse()`, `orElseGet()`, `orElseThrow()`
- Anti-patterns: using `Optional` as field, `isPresent()` + `get()` together
- `Optional` in streams

## 4.6 Math & Number APIs
- `Math` class — `abs()`, `ceil()`, `floor()`, `round()`, `sqrt()`, `pow()`, `log()`, `min()`, `max()`, `random()`
- `BigInteger` — arbitrary precision integers
- `BigDecimal` — arbitrary precision decimals, monetary calculations
- `BigDecimal` scale and rounding modes
- `Random` and `SecureRandom`
- `ThreadLocalRandom`

## 4.7 I/O and NIO
### Classic I/O (`java.io`)
- Streams: `InputStream`, `OutputStream`
- Readers & Writers: `Reader`, `Writer`, `FileReader`, `FileWriter`
- Buffered streams: `BufferedReader`, `BufferedWriter`, `BufferedInputStream`
- `PrintWriter`, `PrintStream`
- `File` class — file metadata, create, delete, list
- Object serialization: `Serializable`, `ObjectInputStream`, `ObjectOutputStream`
- `transient` keyword
- `serialVersionUID`

### NIO (`java.nio`) — Java 4+
- `Path` and `Paths` (Java 7+)
- `Files` utility class — `readAllLines()`, `write()`, `copy()`, `move()`, `delete()`, `exists()`
- `FileSystem`, `FileSystems`
- Channels and Buffers
- `ByteBuffer`, `CharBuffer`
- `FileChannel`, `SocketChannel`, `ServerSocketChannel`
- Memory-mapped files
- `WatchService` — watching directory changes
- `DirectoryStream`, file walking

### NIO.2 (Java 7+)
- `Files.walk()`, `Files.find()`
- `Files.lines()` — lazy reading

## 4.8 Networking
- `InetAddress`, `InetSocketAddress`
- TCP: `Socket`, `ServerSocket`
- UDP: `DatagramSocket`, `DatagramPacket`
- `URL`, `URLConnection`
- `HttpURLConnection`
- HTTP Client API (Java 11+): `HttpClient`, `HttpRequest`, `HttpResponse`
- Async HTTP requests

## 4.9 Reflection API
- `Class<?>` object — `getClass()`, `Class.forName()`
- Inspecting fields, methods, constructors
- Accessing private members — `setAccessible(true)`
- Dynamic invocation — `Method.invoke()`
- Creating instances dynamically
- Annotations via reflection
- Use cases: frameworks, dependency injection, serialization
- Performance implications

## 4.10 Annotations
- Built-in annotations: `@Override`, `@Deprecated`, `@SuppressWarnings`, `@FunctionalInterface`, `@SafeVarargs`
- Meta-annotations: `@Retention`, `@Target`, `@Documented`, `@Inherited`, `@Repeatable`
- Custom annotation definition
- Reading annotations at runtime via reflection
- Annotation processors (compile-time)
- Lombok — code generation with annotations

---

# ⚡ PHASE 5 — Functional Programming in Java
*Java 8 transformed the language — master the functional style*

## 5.1 Lambda Expressions
- Functional interface prerequisite
- Lambda syntax: `(params) -> expression` and `(params) -> { body }`
- Type inference in lambdas
- Effectively final variables in lambdas
- Method references: `ClassName::methodName`
  - Static method reference: `Math::abs`
  - Instance method reference (specific object): `str::contains`
  - Instance method reference (any object): `String::toLowerCase`
  - Constructor reference: `ArrayList::new`
- Lambdas vs. anonymous inner classes

## 5.2 Functional Interfaces (`java.util.function`)
- `Function<T, R>` — `apply()`, `andThen()`, `compose()`
- `BiFunction<T, U, R>`
- `Consumer<T>` — `accept()`, `andThen()`
- `BiConsumer<T, U>`
- `Supplier<T>` — `get()`
- `Predicate<T>` — `test()`, `and()`, `or()`, `negate()`
- `BiPredicate<T, U>`
- `UnaryOperator<T>` — extends Function
- `BinaryOperator<T>` — extends BiFunction
- Primitive specializations: `IntFunction`, `LongConsumer`, `DoubleSupplier`, etc.
- Composing functions — functional pipelines

## 5.3 Streams API (`java.util.stream`)
### Creating Streams
- `Collection.stream()`, `Collection.parallelStream()`
- `Stream.of()`, `Stream.empty()`
- `Arrays.stream()`
- `Stream.generate()`, `Stream.iterate()`
- `IntStream.range()`, `IntStream.rangeClosed()`
- `Files.lines()`
- `BufferedReader.lines()`

### Intermediate Operations (lazy)
- `filter(Predicate)` — keep matching elements
- `map(Function)` — transform elements
- `flatMap(Function)` — flatten nested streams
- `distinct()` — remove duplicates
- `sorted()` / `sorted(Comparator)` — sort
- `peek(Consumer)` — debug/side effect
- `limit(n)` — take first n
- `skip(n)` — skip first n
- `mapToInt()`, `mapToLong()`, `mapToDouble()` — primitive streams
- `takeWhile(Predicate)` — Java 9+
- `dropWhile(Predicate)` — Java 9+

### Terminal Operations (eager)
- `forEach(Consumer)` / `forEachOrdered()`
- `collect(Collector)` — most powerful terminal operation
- `toList()` — Java 16+ direct method
- `count()`
- `findFirst()` / `findAny()`
- `anyMatch()` / `allMatch()` / `noneMatch()`
- `min(Comparator)` / `max(Comparator)`
- `reduce(BinaryOperator)` — fold/aggregate
- `toArray()`
- `sum()`, `average()`, `summaryStatistics()` — on primitive streams

### Collectors (`java.util.stream.Collectors`)
- `toList()`, `toSet()`, `toCollection()`
- `toMap()`, `toConcurrentMap()`
- `groupingBy()` — grouping elements
- `partitioningBy()` — true/false split
- `joining()` — string concatenation
- `counting()`, `summingInt()`, `averagingInt()`
- `toUnmodifiableList()`, `toUnmodifiableSet()`, `toUnmodifiableMap()`
- `collectingAndThen()` — wrapping collectors
- `mapping()`, `flatMapping()`
- Custom `Collector` via `Collector.of()`
- `teeing()` (Java 12+) — two collectors merged

### Parallel Streams
- `parallelStream()` vs. `stream()`
- Fork-Join pool
- When to use parallel (large data, CPU-bound, no shared state)
- Thread safety in parallel streams
- `Spliterator`

## 5.4 Comparator (Advanced)
- `Comparator.comparing()`, `Comparator.comparingInt()`
- `thenComparing()` — chaining comparators
- `reversed()` — reverse order
- `Comparator.naturalOrder()`, `Comparator.reverseOrder()`
- `Comparator.nullsFirst()`, `Comparator.nullsLast()`
- `Comparable` vs `Comparator`

---

# 🔀 PHASE 6 — Concurrency & Multithreading
*Critical for high-performance applications*

## 6.1 Threads Fundamentals
- Process vs. Thread
- `Thread` class — `start()`, `run()`, `sleep()`, `join()`, `interrupt()`, `isAlive()`
- `Runnable` interface
- Thread states: NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED
- Thread priorities
- Daemon threads
- `ThreadGroup`
- Thread naming for debugging

## 6.2 Thread Synchronization
- Race conditions and data races
- Critical sections
- `synchronized` keyword — methods and blocks
- Object monitor / intrinsic lock
- `this` lock vs. class lock (`ClassName.class`)
- `wait()`, `notify()`, `notifyAll()` — producer-consumer
- Deadlock — causes, prevention, detection
- Livelock, starvation, thread contention
- `volatile` keyword — visibility, happens-before
- Memory visibility and CPU cache issues
- Java Memory Model (JMM)

## 6.3 `java.util.concurrent` Package
### Locks
- `Lock` interface — `lock()`, `unlock()`, `tryLock()`, `lockInterruptibly()`
- `ReentrantLock` — explicit locking
- `ReadWriteLock`, `ReentrantReadWriteLock`
- `StampedLock` (Java 8+)
- `Condition` — replacement for `wait`/`notify`

### Atomic Variables
- `AtomicInteger`, `AtomicLong`, `AtomicBoolean`, `AtomicReference`
- CAS (Compare-And-Swap) operations
- `getAndIncrement()`, `compareAndSet()`
- `LongAdder`, `LongAccumulator` — high-contention counters

### Concurrent Collections
- `ConcurrentHashMap` — segment locking, `computeIfAbsent()`
- `CopyOnWriteArrayList`, `CopyOnWriteArraySet`
- `ConcurrentLinkedQueue`, `ConcurrentLinkedDeque`
- `BlockingQueue`: `ArrayBlockingQueue`, `LinkedBlockingQueue`, `PriorityBlockingQueue`, `SynchronousQueue`, `DelayQueue`
- `BlockingDeque`: `LinkedBlockingDeque`

### Executor Framework
- `Executor`, `ExecutorService`, `ScheduledExecutorService`
- `Executors` factory: `newFixedThreadPool()`, `newCachedThreadPool()`, `newSingleThreadExecutor()`, `newScheduledThreadPool()`
- `ThreadPoolExecutor` — corePoolSize, maxPoolSize, keepAlive, queue, rejection policy
- Submitting tasks: `submit()`, `execute()`, `invokeAll()`, `invokeAny()`
- `Future<T>` — `get()`, `cancel()`, `isDone()`, `isCancelled()`
- `Callable<T>` — returns value, throws checked exception

### Synchronizers
- `CountDownLatch` — wait for N events
- `CyclicBarrier` — all threads wait at barrier
- `Semaphore` — limit concurrent access
- `Phaser` — flexible barrier
- `Exchanger` — swap data between two threads

### CompletableFuture (Java 8+)
- `CompletableFuture.supplyAsync()`, `runAsync()`
- Chaining: `thenApply()`, `thenAccept()`, `thenRun()`
- Combining: `thenCombine()`, `allOf()`, `anyOf()`
- Error handling: `exceptionally()`, `handle()`, `whenComplete()`
- Custom executor
- `CompletableFuture` vs. `Future`

## 6.4 Fork-Join Framework
- `ForkJoinPool` — work-stealing algorithm
- `RecursiveTask<V>`, `RecursiveAction`
- `fork()`, `join()`, `compute()`
- When to use Fork-Join
- `ForkJoinPool.commonPool()`

## 6.5 Virtual Threads (Java 21) — Project Loom
- Platform threads vs. Virtual threads
- Creating virtual threads: `Thread.ofVirtual().start()`
- `Executors.newVirtualThreadPerTaskExecutor()`
- Structured Concurrency — `StructuredTaskScope`
- Scoped Values — replacing ThreadLocal
- When virtual threads win (IO-bound tasks)
- Pinning issues with `synchronized`
- Virtual threads and reactive programming comparison

## 6.6 Concurrency Patterns
- Producer-Consumer pattern
- Thread pool pattern
- Readers-Writer pattern
- Thread-local storage — `ThreadLocal<T>`
- Immutability as concurrency strategy

---

# 🏛️ PHASE 7 — JVM Internals & Performance
*Understand what happens under the hood*

## 7.1 JVM Architecture
- ClassLoader subsystem
  - Bootstrap, Extension, Application ClassLoaders
  - Custom ClassLoaders
  - Parent delegation model
  - Class loading lifecycle: Loading → Linking (Verify, Prepare, Resolve) → Initialization
- Runtime Data Areas
  - Method Area / Metaspace (Java 8+)
  - Heap — Young Generation (Eden, S0, S1) + Old Generation (Tenured)
  - Java Stack (per thread) — stack frames
  - PC Register (per thread)
  - Native Method Stack
- Execution Engine
  - Interpreter
  - JIT Compiler (C1, C2 — tiered compilation)
  - AOT Compilation (GraalVM)
- Native Method Interface (JNI)

## 7.2 Garbage Collection
- Object lifecycle and reachability
- GC Roots — local variables, static fields, JNI references
- Mark-and-Sweep algorithm
- Generational GC theory — hypothesis and regions
- **GC Algorithms:**
  - Serial GC — single-threaded, small apps
  - Parallel GC — throughput collector, multi-threaded
  - CMS (Concurrent Mark-Sweep) — deprecated in Java 14
  - G1 GC (Garbage-First) — default since Java 9, regions-based
  - ZGC (Z Garbage Collector) — Java 15+, low-latency, <1ms pauses
  - Shenandoah GC — Red Hat, concurrent compaction
  - Epsilon GC — no-op GC for testing
- GC tuning flags: `-Xms`, `-Xmx`, `-Xmn`, `-XX:+UseG1GC`
- GC logs analysis
- Minor GC vs. Major GC vs. Full GC
- Stop-The-World (STW) events
- Memory leaks — detection and prevention
- Weak, Soft, Phantom references

## 7.3 JVM Performance Tuning
- JVM startup flags and options
- Heap sizing strategy
- GC pause time goals
- Metaspace sizing
- JIT compilation flags
- Escape analysis and stack allocation
- Object pooling
- String deduplication
- Compressed OOPs (Ordinary Object Pointers)

## 7.4 Profiling & Monitoring Tools
- **JConsole** — basic JVM monitoring
- **VisualVM** — heap dumps, CPU profiling
- **JMC (Java Mission Control)** + **JFR (Java Flight Recorder)**
- **async-profiler** — low-overhead CPU/memory profiler
- **YourKit**, **JProfiler** — commercial profilers
- `jps` — list Java processes
- `jstack` — thread dump
- `jmap` — heap dump, histogram
- `jstat` — GC statistics
- `jcmd` — all-in-one diagnostic tool
- Heap dump analysis with **Eclipse MAT**

## 7.5 JVM Languages Interop
- Kotlin on JVM
- Groovy for scripting
- Scala for functional
- Calling Java from Kotlin and vice versa

---

# 🧩 PHASE 8 — Design Patterns
*Reusable solutions to recurring problems*

## 8.1 Creational Patterns
- **Singleton** — one instance, thread-safe variants (DCL, Enum singleton, Holder)
- **Factory Method** — subclass decides what to create
- **Abstract Factory** — family of related objects
- **Builder** — step-by-step object construction (Telescoping constructor problem)
- **Prototype** — clone existing objects
- **Object Pool** — reuse expensive objects

## 8.2 Structural Patterns
- **Adapter** — bridge incompatible interfaces
- **Bridge** — separate abstraction from implementation
- **Composite** — tree structures, uniform treatment
- **Decorator** — add behavior dynamically (I/O streams example)
- **Facade** — simplified interface to a complex system
- **Flyweight** — share objects to save memory (String pool)
- **Proxy** — surrogate object (JDK dynamic proxy, CGLib)

## 8.3 Behavioral Patterns
- **Chain of Responsibility** — pass request along a chain
- **Command** — encapsulate requests as objects
- **Interpreter** — grammar for a language
- **Iterator** — traverse collections uniformly
- **Mediator** — centralize communication
- **Memento** — save/restore object state
- **Observer** — event notification (publish-subscribe)
- **State** — change behavior based on internal state
- **Strategy** — interchangeable algorithms
- **Template Method** — define algorithm skeleton
- **Visitor** — operations on elements without modifying them

## 8.4 Architectural Patterns
- **MVC** (Model-View-Controller)
- **MVP** (Model-View-Presenter)
- **MVVM** (Model-View-ViewModel)
- **Repository Pattern**
- **Service Layer Pattern**
- **DAO (Data Access Object) Pattern**
- **Domain-Driven Design (DDD) basics**

---

# 🌐 PHASE 9 — Java Modules System (JPMS)
*Java 9+ modularity*

## 9.1 Module Basics
- Problems with classpath — JAR hell
- Module definition — `module-info.java`
- `module`, `requires`, `exports`, `opens`, `uses`, `provides`
- Named modules, unnamed modules, automatic modules
- Module path vs. classpath

## 9.2 Module Types
- Platform modules (`java.base`, `java.sql`, etc.)
- Application modules
- Automatic modules
- Unnamed module (legacy classpath)

## 9.3 Module Commands
- `--module-path`, `--module`, `--add-modules`, `--add-opens`, `--add-exports`
- `jar --describe-module`
- `jlink` — custom JRE creation
- `jdeps` — module dependency analysis

---

# 💾 PHASE 10 — Database & Persistence

## 10.1 JDBC (Java Database Connectivity)
- JDBC architecture: Driver, Connection, Statement, ResultSet
- Connection URL formats for MySQL, PostgreSQL, Oracle
- `DriverManager.getConnection()`
- `Statement` vs `PreparedStatement` vs `CallableStatement`
- SQL injection prevention with PreparedStatement
- `ResultSet` — iterating, types, concurrency
- Batch updates
- Transactions — `setAutoCommit(false)`, `commit()`, `rollback()`
- Savepoints
- Connection pooling — HikariCP, Apache DBCP, C3P0
- `DataSource` interface

## 10.2 JPA (Java Persistence API) / Jakarta Persistence
- ORM concepts — Object-Relational Mapping
- JPA specification vs. Hibernate implementation
- `@Entity`, `@Table`, `@Id`, `@GeneratedValue`
- `@Column`, `@Transient`, `@Temporal`
- Relationships: `@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany`
- Cascade types: PERSIST, MERGE, REMOVE, REFRESH, DETACH, ALL
- Fetch types: EAGER vs LAZY loading
- N+1 query problem and solutions
- `EntityManager` — `persist()`, `merge()`, `remove()`, `find()`
- `EntityManagerFactory`, `Persistence`
- JPQL (Java Persistence Query Language)
- Named queries
- Criteria API — type-safe queries

## 10.3 Hibernate
- Hibernate configuration (`hibernate.cfg.xml`, `persistence.xml`)
- Session, SessionFactory
- First-level cache (Session) and second-level cache
- HQL (Hibernate Query Language)
- Hibernate-specific annotations
- `@Formula`, `@Where`, `@BatchSize`
- Hibernate Validator (Bean Validation JSR-380)
- `@Valid`, `@NotNull`, `@Size`, `@Min`, `@Max`, `@Pattern`

## 10.4 Spring Data JPA (Preview)
- `JpaRepository`, `CrudRepository`, `PagingAndSortingRepository`
- Query derivation from method names
- `@Query` annotation
- Pagination and sorting
- Auditing: `@CreatedDate`, `@LastModifiedDate`

## 10.5 NoSQL with Java
- MongoDB — MongoDB Java Driver, Spring Data MongoDB
- Redis — Jedis, Lettuce, Spring Data Redis
- Cassandra — DataStax Driver, Spring Data Cassandra
- Elasticsearch — Java High-Level REST Client

---

# 🌱 PHASE 11 — Spring Framework Ecosystem
*Industry standard for enterprise Java*

## 11.1 Spring Core
- Spring philosophy — POJO-based, convention over configuration
- Inversion of Control (IoC) container
- Dependency Injection (DI): Constructor, Setter, Field injection
- `BeanFactory` vs `ApplicationContext`
- Bean scopes: Singleton, Prototype, Request, Session, Application
- Bean lifecycle: `@PostConstruct`, `@PreDestroy`, `InitializingBean`, `DisposableBean`
- `@Component`, `@Service`, `@Repository`, `@Controller`
- `@Autowired`, `@Qualifier`, `@Primary`
- `@Configuration`, `@Bean`
- `@ComponentScan`
- Profiles — `@Profile`, `spring.profiles.active`
- `@Value`, `@PropertySource`
- Spring Expression Language (SpEL)

## 11.2 Spring AOP (Aspect-Oriented Programming)
- AOP concepts: Aspect, Join Point, Advice, Pointcut, Weaving
- `@Aspect`, `@Before`, `@After`, `@Around`, `@AfterReturning`, `@AfterThrowing`
- Pointcut expressions
- JDK dynamic proxy vs. CGLIB proxy
- Use cases: logging, security, transactions, caching

## 11.3 Spring Boot
- Why Spring Boot — auto-configuration, embedded server, opinionated defaults
- `spring-boot-starter-*` dependencies
- `@SpringBootApplication`
- `application.properties` / `application.yml`
- Profiles in Spring Boot
- Actuator — `/health`, `/metrics`, `/info`, `/env`, `/beans`
- Spring Boot DevTools — live reload
- Fat JAR / executable JAR
- Customizing auto-configuration
- `@ConditionalOn*` annotations

## 11.4 Spring MVC / Spring Web
- DispatcherServlet architecture
- `@Controller`, `@RestController`
- `@RequestMapping`, `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`
- `@PathVariable`, `@RequestParam`, `@RequestBody`, `@RequestHeader`
- `@ResponseBody`, `ResponseEntity<T>`
- `ModelAndView` (traditional MVC)
- Exception handling: `@ExceptionHandler`, `@ControllerAdvice`, `@RestControllerAdvice`
- Data validation with `@Valid`, `BindingResult`
- Content negotiation
- File upload/download
- CORS configuration

## 11.5 Spring Data
- Repository pattern in Spring
- `CrudRepository`, `JpaRepository`, `MongoRepository`
- Derived query methods
- `@Query` with JPQL/native SQL
- Paging: `Pageable`, `Page<T>`
- Auditing
- Specifications (type-safe queries)
- QueryDSL integration

## 11.6 Spring Security
- Authentication vs. Authorization
- `SecurityFilterChain` configuration (Spring Security 6)
- `UserDetailsService`, `UserDetails`
- Password encoding: `BCryptPasswordEncoder`
- Form login, HTTP Basic, JWT
- Role-based access: `@PreAuthorize`, `@Secured`, `hasRole()`
- Method-level security
- CSRF protection
- Session management
- OAuth2 / OpenID Connect
- JWT authentication filter
- LDAP authentication

## 11.7 Spring Transaction Management
- `@Transactional` — propagation, isolation, rollback rules
- Transaction propagation: REQUIRED, REQUIRES_NEW, NESTED, SUPPORTS, NOT_SUPPORTED, MANDATORY, NEVER
- Isolation levels: READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE
- Rollback rules — checked vs unchecked exceptions
- Programmatic transactions with `TransactionTemplate`

## 11.8 Spring Testing
- `@SpringBootTest` — full context
- `@WebMvcTest` — controller layer only
- `@DataJpaTest` — JPA layer only
- `MockMvc` — test HTTP endpoints
- `@MockBean`, `@SpyBean`
- `TestRestTemplate`, `WebTestClient`
- Test slices and test configuration

## 11.9 Spring WebFlux (Reactive)
- Reactive programming concepts
- Project Reactor — `Mono<T>`, `Flux<T>`
- Reactive operators: `map()`, `flatMap()`, `filter()`, `zip()`, `merge()`
- Backpressure
- `@RestController` in WebFlux
- `RouterFunction`, `HandlerFunction` (functional style)
- Reactive MongoDB, R2DBC
- WebClient — non-blocking HTTP client

## 11.10 Spring Messaging
- Spring Integration — enterprise integration patterns
- Spring AMQP — RabbitMQ integration
- Spring Kafka — Kafka integration
- `@KafkaListener`, `KafkaTemplate`
- WebSocket support — `@EnableWebSocket`, STOMP

---

# 🧪 PHASE 12 — Testing
*Professional developers write tests*

## 12.1 JUnit 5 (Jupiter)
- `@Test`, `@DisplayName`, `@Disabled`
- Lifecycle: `@BeforeEach`, `@AfterEach`, `@BeforeAll`, `@AfterAll`
- Assertions: `assertEquals()`, `assertTrue()`, `assertNull()`, `assertThrows()`, `assertAll()`
- `@ParameterizedTest` — `@ValueSource`, `@CsvSource`, `@MethodSource`, `@EnumSource`
- `@RepeatedTest`
- `@Nested` — inner test classes
- `@Tag` — filtering tests
- `@TempDir` — temporary directories
- `@ExtendWith` — custom extensions
- JUnit 5 vs JUnit 4 differences

## 12.2 Mockito
- `@Mock`, `@InjectMocks`, `@Spy`, `@Captor`
- `Mockito.mock()`, `Mockito.spy()`
- Stubbing: `when().thenReturn()`, `when().thenThrow()`, `doReturn().when()`
- Verification: `verify()`, `times()`, `never()`, `atLeastOnce()`
- `ArgumentCaptor` — capturing arguments
- `ArgumentMatchers` — `any()`, `eq()`, `anyString()`
- `@MockitoExtension` with JUnit 5
- Mocking static methods (Mockito 3.4+)
- Mocking constructors

## 12.3 Test Pyramid & Strategy
- Unit tests — fast, isolated, test one thing
- Integration tests — test multiple components
- End-to-end tests — full system tests
- Test coverage targets — line, branch, mutation
- Test naming conventions
- AAA pattern: Arrange-Act-Assert
- BDD style: Given-When-Then

## 12.4 Other Testing Tools
- **AssertJ** — fluent assertion library
- **Hamcrest** — matcher library
- **WireMock** — mock HTTP servers
- **Testcontainers** — Docker-based integration tests (real databases)
- **H2** — in-memory database for tests
- **Awaitility** — async testing
- **RestAssured** — REST API testing

## 12.5 Code Quality Tools
- **Jacoco** — code coverage reports
- **SonarQube / SonarCloud** — static code analysis
- **Checkstyle** — code style enforcement
- **PMD** — common code defects
- **SpotBugs** — bug pattern detection
- **Pitest** — mutation testing

---

# 🔌 PHASE 13 — APIs & Web Services

## 13.1 REST API Design
- REST principles (stateless, uniform interface, client-server, cacheable, layered)
- HTTP methods semantics: GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS
- HTTP status codes — 1xx, 2xx, 3xx, 4xx, 5xx
- URI design best practices
- Request/response headers
- Content negotiation (`Accept`, `Content-Type`)
- HATEOAS — hypermedia as the engine of application state
- API versioning strategies: URL path, query param, header
- Pagination: offset-based, cursor-based, keyset-based
- Filtering, sorting, searching in REST APIs
- Idempotency

## 13.2 OpenAPI / Swagger
- OpenAPI 3.0 specification
- Springdoc-OpenAPI integration
- `@Operation`, `@ApiResponse`, `@Schema`
- Swagger UI
- Code generation from OpenAPI spec

## 13.3 GraphQL with Java
- GraphQL concepts — schema, queries, mutations, subscriptions
- Spring for GraphQL
- Schema definition language (SDL)
- Resolvers / DataFetchers
- N+1 problem in GraphQL — DataLoader
- GraphQL vs REST

## 13.4 gRPC with Java
- Protocol Buffers (protobuf)
- `.proto` file definition
- gRPC service types: Unary, Server streaming, Client streaming, Bidirectional
- Java gRPC stub generation
- gRPC vs REST

## 13.5 SOAP Web Services
- WSDL (Web Services Description Language)
- JAX-WS (Jakarta XML Web Services)
- `@WebService`, `@WebMethod`
- Apache CXF
- SOAP vs REST

---

# 📨 PHASE 14 — Messaging & Event-Driven Systems

## 14.1 Apache Kafka
- Kafka architecture — broker, producer, consumer, topic, partition, offset
- Kafka consumer groups
- Kafka producer API — `KafkaProducer`, `ProducerRecord`
- Kafka consumer API — `KafkaConsumer`, `ConsumerRecord`
- Kafka Streams — stream processing
- Kafka Connect — data pipeline
- Schema Registry — Avro, Protobuf, JSON Schema
- Exactly-once semantics
- Spring Kafka — `@KafkaListener`, `KafkaTemplate`

## 14.2 RabbitMQ
- AMQP protocol
- Exchanges (Direct, Topic, Fanout, Headers)
- Queues, bindings, routing keys
- Message acknowledgement
- Dead letter queues
- Spring AMQP

## 14.3 Other Messaging
- ActiveMQ
- AWS SQS / SNS
- Redis Pub/Sub
- Event-driven architecture patterns

---

# ☁️ PHASE 15 — Cloud, DevOps & Microservices

## 15.1 Docker for Java
- Dockerfile for Java apps
- Multi-stage builds — compile then package
- JVM in containers — memory/CPU limits
- `-XX:+UseContainerSupport` flag
- Docker Compose for local development
- JIB — build Docker images without Dockerfile
- Distroless images

## 15.2 Kubernetes for Java
- Pods, Deployments, Services, ConfigMaps, Secrets
- Liveness and readiness probes
- Resource limits and requests
- Rolling updates
- Horizontal Pod Autoscaler
- Helm charts for Java apps
- Spring Boot Actuator + Kubernetes probes

## 15.3 CI/CD Pipelines
- **Jenkins** — Jenkinsfile, pipelines
- **GitHub Actions** — workflows, Java setup action
- **GitLab CI** — `.gitlab-ci.yml`
- **CircleCI**, **TravisCI**
- Build → Test → SonarQube → Docker build → Push → Deploy
- Blue-green deployments
- Canary deployments

## 15.4 Microservices with Spring Boot
- Microservices principles — single responsibility, loose coupling
- Service discovery — Eureka, Consul
- API Gateway — Spring Cloud Gateway
- Load balancing — Ribbon (legacy), Spring Cloud LoadBalancer
- Circuit breaker — Resilience4j (`@CircuitBreaker`, `@Retry`, `@RateLimiter`, `@Bulkhead`)
- Distributed configuration — Spring Cloud Config Server
- Distributed tracing — Zipkin, Jaeger, Micrometer Tracing
- Service mesh — Istio
- Saga pattern — orchestration vs choreography
- Event sourcing
- CQRS (Command Query Responsibility Segregation)

## 15.5 Observability
- **Logging** — SLF4J + Logback, Log4j2, structured logging (JSON), MDC (Mapped Diagnostic Context)
- **Metrics** — Micrometer, Prometheus, Grafana
- **Tracing** — OpenTelemetry, Jaeger, Zipkin
- **Alerting** — Grafana Alerts, PagerDuty
- Centralized logging — ELK Stack (Elasticsearch, Logstash, Kibana)

## 15.6 Cloud Platforms
- **AWS** — EC2, ECS, EKS, Lambda, RDS, S3, SQS, SNS, API Gateway, CloudWatch
- **GCP** — GKE, Cloud Run, Cloud SQL, Pub/Sub
- **Azure** — AKS, Azure Functions, Azure Service Bus
- AWS SDK for Java v2
- Spring Cloud AWS

## 15.7 Serverless Java
- AWS Lambda with Java — cold start problem
- GraalVM Native Image for faster startup
- Quarkus — supersonic subatomic Java (GraalVM-first)
- Micronaut — ahead-of-time compilation

---

# 🔐 PHASE 16 — Security

## 16.1 Application Security Fundamentals
- OWASP Top 10
- SQL injection — prevention with PreparedStatement
- XSS (Cross-Site Scripting) — output encoding
- CSRF protection
- Insecure deserialization
- Security misconfiguration
- Sensitive data exposure

## 16.2 Authentication & Authorization
- Session-based authentication
- Token-based authentication (JWT)
  - JWT structure: Header.Payload.Signature
  - Access tokens and refresh tokens
  - JWT validation
  - JJWT library
- OAuth 2.0 flows — Authorization Code, Client Credentials, Implicit, PKCE
- OpenID Connect (OIDC)
- SAML 2.0
- API keys

## 16.3 Cryptography in Java
- `java.security` package
- `MessageDigest` — SHA-256, SHA-512
- `Cipher` — AES, RSA
- `KeyPairGenerator`, `KeyGenerator`
- `Mac` — HMAC
- `SecureRandom`
- Digital signatures — `Signature`
- X.509 certificates
- TLS/SSL configuration — `SSLContext`, `TrustManager`, `KeyManager`
- HTTPS in Java clients

## 16.4 Secrets Management
- Environment variables
- HashiCorp Vault integration
- AWS Secrets Manager
- Never hardcode credentials

---

# 📊 PHASE 17 — Data Structures & Algorithms in Java
*Essential for interviews and optimization*

## 17.1 Core Data Structures (Implement from Scratch)
- Array, Dynamic Array
- Linked List (Singly, Doubly, Circular)
- Stack (Array-based, LinkedList-based)
- Queue (Circular array, LinkedList-based)
- Deque
- Binary Tree, BST (Binary Search Tree)
- AVL Tree (Self-balancing)
- Red-Black Tree (HashMap internals)
- Heap (Min-heap, Max-heap) — `PriorityQueue` internals
- Hash Table — hash functions, collision resolution (chaining, open addressing)
- Trie (Prefix Tree)
- Graph (Adjacency List, Adjacency Matrix)

## 17.2 Algorithms
### Sorting
- Bubble, Selection, Insertion — O(n²)
- Merge Sort — O(n log n), stable
- Quick Sort — O(n log n) avg, in-place
- Heap Sort — O(n log n)
- Counting Sort, Radix Sort — O(n+k)
- `Arrays.sort()` and `Collections.sort()` internals (TimSort)

### Searching
- Linear Search — O(n)
- Binary Search — O(log n) — iterative and recursive
- `Arrays.binarySearch()`, `Collections.binarySearch()`
- Ternary Search

### Graph Algorithms
- BFS (Breadth-First Search) — shortest path unweighted
- DFS (Depth-First Search) — cycle detection, topological sort
- Dijkstra's Algorithm — shortest path weighted
- Bellman-Ford — negative weights
- Floyd-Warshall — all-pairs shortest path
- Kruskal's and Prim's — Minimum Spanning Tree
- Topological Sort — Kahn's and DFS-based
- Union-Find (Disjoint Set Union)

### Dynamic Programming
- Memoization vs tabulation
- Classic problems: Fibonacci, Knapsack, LCS, LIS, Coin Change, Edit Distance
- Matrix Chain Multiplication
- DP on trees and graphs

### Other Algorithms
- Two Pointers, Sliding Window
- Divide and Conquer
- Greedy algorithms
- Backtracking
- Bit manipulation tricks

## 17.3 Complexity Analysis
- Big O notation — O(1), O(log n), O(n), O(n log n), O(n²), O(2^n), O(n!)
- Time complexity vs Space complexity
- Amortized analysis
- Best, average, worst case

---

# 🏗️ PHASE 18 — Architecture & System Design
*For senior and lead engineers*

## 18.1 Software Architecture Fundamentals
- Architectural styles: Monolith, SOA, Microservices, Serverless, Event-driven
- Architectural quality attributes: scalability, availability, reliability, maintainability, performance, security
- CAP Theorem (Consistency, Availability, Partition Tolerance)
- BASE vs ACID
- Eventual consistency

## 18.2 System Design Topics
- Load balancing strategies (Round Robin, Least Connections, IP Hash)
- Horizontal vs Vertical scaling
- Caching strategies — Cache-aside, Write-through, Write-back, Read-through
- Cache invalidation — TTL, event-based, manual
- CDN (Content Delivery Network)
- Database sharding — horizontal partitioning
- Database replication — master-slave, master-master
- Read replicas
- Consistent hashing
- Message queues for async processing
- Rate limiting algorithms — Token Bucket, Leaky Bucket, Fixed Window, Sliding Window
- API pagination strategies
- Database indexing strategies — B-tree, hash, composite
- Full-text search — Elasticsearch
- Blob storage — S3
- WebSockets vs Server-Sent Events vs Long Polling

## 18.3 Common System Design Patterns
- Circuit Breaker
- Bulkhead
- Retry with Exponential Backoff
- Timeout
- Fallback
- Sidecar
- Ambassador
- Strangler Fig (Monolith migration)
- Anti-Corruption Layer

## 18.4 Clean Architecture
- Hexagonal Architecture (Ports and Adapters)
- Clean Architecture (Uncle Bob) — Entities, Use Cases, Interface Adapters, Frameworks
- Onion Architecture
- Dependency rule — inner layers don't depend on outer
- Domain-Driven Design (DDD)
  - Ubiquitous Language
  - Bounded Context
  - Aggregates, Entities, Value Objects
  - Domain Events
  - Repositories
  - Application Services vs Domain Services

## 18.5 12-Factor App Methodology
1. Codebase — one codebase, tracked in version control
2. Dependencies — explicitly declare and isolate
3. Config — store in environment
4. Backing services — treat as attached resources
5. Build, release, run — strictly separate stages
6. Processes — stateless, share-nothing
7. Port binding — export services via port binding
8. Concurrency — scale out via process model
9. Disposability — fast startup, graceful shutdown
10. Dev/prod parity — keep environments similar
11. Logs — treat as event streams
12. Admin processes — run as one-off processes

---

# 👥 PHASE 19 — Software Lead Skills
*Technical leadership beyond coding*

## 19.1 Code Review Best Practices
- What to look for: correctness, performance, security, maintainability, tests
- Giving constructive feedback
- Receiving feedback graciously
- Review checklists
- PR size guidelines
- Automated checks before human review

## 19.2 Technical Documentation
- Javadoc — writing effective API documentation
- Architecture Decision Records (ADRs)
- README and contributing guides
- API documentation with OpenAPI
- Runbooks and operational guides
- Onboarding documentation

## 19.3 Agile & Project Management
- Scrum — sprints, standups, retrospectives, backlog refinement
- Kanban — WIP limits, flow
- Story pointing and estimation
- Technical debt management
- Breaking down epics to user stories to tasks
- Definition of Done

## 19.4 Mentoring & Team Leadership
- Teaching junior developers
- Pair programming
- Code review as a teaching tool
- 1:1 conversations
- Setting technical direction
- Building psychological safety
- Knowledge sharing — tech talks, brown bags

## 19.5 Technical Decision Making
- Evaluating technology choices (build vs. buy)
- Proof of concept (POC) planning
- Risk assessment
- Migration planning
- Legacy system modernization
- Technical roadmapping

## 19.6 Performance Engineering
- Identifying bottlenecks — profiling first, optimize second
- Load testing — JMeter, Gatling, k6
- Benchmarking — JMH (Java Microbenchmark Harness)
- Database query optimization — EXPLAIN ANALYZE
- Caching implementation strategy
- Async processing for non-critical paths

---

# 🔧 PHASE 20 — Modern Java Features Deep Dive
*Stay current with the latest Java*

## 20.1 Java 9–11 Features
- **Java 9:** JPMS modules, JShell, `List.of()`, `Map.of()`, `Stream.takeWhile()`, `Stream.dropWhile()`, `Optional.ifPresentOrElse()`, `ProcessHandle`
- **Java 10:** Local-variable type inference (`var`), `List.copyOf()`, `Collectors.toUnmodifiableList()`
- **Java 11:** `String.isBlank()`, `String.lines()`, `String.strip()`, `String.repeat()`, `Files.readString()`, `Files.writeString()`, HTTP Client API stable, `Predicate.not()`

## 20.2 Java 12–16 Features
- **Java 12:** `switch` expression (preview), `String.indent()`, `String.transform()`
- **Java 13:** Text blocks (preview), `switch` expression updates
- **Java 14:** Records (preview), Pattern Matching `instanceof` (preview), Helpful NullPointerExceptions
- **Java 15:** Sealed classes (preview), Text blocks stable, Hidden classes
- **Java 16:** Records stable, Pattern Matching `instanceof` stable, `Stream.toList()`, Vector API (incubator)

## 20.3 Java 17–21 Features (LTS milestones)
- **Java 17:** Sealed classes stable, Pattern matching in switch (preview), Foreign Function & Memory API (incubator)
- **Java 18:** UTF-8 by default, `@snippet` in Javadoc, Simple Web Server
- **Java 19:** Virtual Threads (preview), Structured Concurrency (incubator), Record Patterns (preview)
- **Java 20:** Scoped Values (incubator), improved previews
- **Java 21:** Virtual Threads stable, Record Patterns stable, Pattern Matching in switch stable, Sequenced Collections, String Templates (preview), Unnamed Classes (preview)

## 20.4 Java 22–24 Features
- String Templates
- Unnamed Variables and Patterns (`_`)
- Unnamed Classes and Instance Main Methods
- Flexible Constructor Bodies
- Stream Gatherers (Java 22)
- Class-file API
- Vector API enhancements
- Foreign Function & Memory API stable

---

# 📚 PHASE 21 — Essential Libraries & Ecosystem

## 21.1 Utility Libraries
- **Apache Commons** — Commons Lang, Commons IO, Commons Collections, Commons Math
- **Google Guava** — Immutable collections, `Multimap`, `BiMap`, `Table`, `Cache`, `Preconditions`, `Optional`, `EventBus`
- **Lombok** — `@Data`, `@Builder`, `@Slf4j`, `@AllArgsConstructor`, `@Value`
- **MapStruct** — type-safe bean mapping
- **ModelMapper** — object mapping

## 21.2 JSON & Serialization
- **Jackson** — `ObjectMapper`, `@JsonProperty`, `@JsonIgnore`, `@JsonSerialize`, `@JsonDeserialize`, `@JsonTypeInfo`, custom serializers, Jackson Modules (Java Time, Kotlin)
- **Gson** — Google's JSON library, `GsonBuilder`
- **JSON-B** (Jakarta JSON Binding)
- **Avro** — schema-based binary serialization
- **Protobuf** — Google's binary serialization
- **Kryo** — fast binary serialization

## 21.3 HTTP Clients
- Java 11 `HttpClient` (built-in)
- **OkHttp** — Square's HTTP client
- **Apache HttpClient 5**
- **Feign** — declarative HTTP client
- **RestTemplate** (Spring, legacy)
- **WebClient** (Spring, reactive)

## 21.4 Caching Libraries
- **Caffeine** — high-performance local cache
- **Ehcache** — distributed/local cache
- **Redis** via Lettuce or Jedis
- **Hazelcast** — distributed caching/computing
- Spring Cache abstraction — `@Cacheable`, `@CacheEvict`, `@CachePut`

## 21.5 Reactive Libraries
- **Project Reactor** (`Mono`, `Flux`)
- **RxJava** (ReactiveX)
- **Mutiny** (Quarkus reactive library)
- Reactive Streams specification

## 21.6 Alternative Frameworks
- **Quarkus** — cloud-native, GraalVM, fast startup
- **Micronaut** — AOT compilation, dependency injection at compile time
- **Vert.x** — reactive toolkit
- **Helidon** (Oracle) — lightweight microservices
- **Dropwizard** — REST API framework
- **Play Framework** (Scala/Java)

---

# 🎓 APPENDIX — Resources & Learning Path

## Recommended Books
1. *Effective Java* — Joshua Bloch (3rd Ed.) — **MUST READ**
2. *Java Concurrency in Practice* — Brian Goetz — **MUST READ**
3. *Clean Code* — Robert C. Martin
4. *Clean Architecture* — Robert C. Martin
5. *Design Patterns* — Gang of Four (GoF)
6. *Head First Java* — Sierra & Bates (beginners)
7. *Java Performance* — Scott Oaks
8. *Designing Data-Intensive Applications* — Martin Kleppmann
9. *Building Microservices* — Sam Newman
10. *The Pragmatic Programmer* — Hunt & Thomas

## Recommended Practice Platforms
- **LeetCode** — DSA interview prep
- **HackerRank** — Java-specific challenges
- **Exercism.io** — mentored practice
- **Codewars** — kata-style problems
- **Project Euler** — mathematical programming
- **Spring Guides** — https://spring.io/guides
- **Baeldung** — https://www.baeldung.com

## Certifications (Optional but Valued)
- Oracle Certified Associate (OCA) — Java SE Programmer I
- Oracle Certified Professional (OCP) — Java SE Programmer II
- Spring Professional Certification
- AWS Certified Developer — Associate

## Open Source Contribution Path
- Contribute to Apache Commons, Spring Boot, or other Java OSS projects
- Start with "good first issue" labels
- Read project contributing guides
- Learn to navigate large codebases

---

## 🗺️ Estimated Timeline

| Phase | Topic | Duration |
|-------|-------|----------|
| 0 | History & Ecosystem | 1 week |
| 1 | Setup & Tools | 1 week |
| 2 | Fundamentals | 4–6 weeks |
| 3 | OOP | 4–6 weeks |
| 4 | Core APIs | 4–6 weeks |
| 5 | Functional Programming | 2–3 weeks |
| 6 | Concurrency | 4–6 weeks |
| 7 | JVM Internals | 2–3 weeks |
| 8 | Design Patterns | 3–4 weeks |
| 9 | Modules | 1 week |
| 10 | Database & Persistence | 4–5 weeks |
| 11 | Spring Ecosystem | 6–8 weeks |
| 12 | Testing | 2–3 weeks |
| 13 | APIs & Web Services | 2–3 weeks |
| 14 | Messaging | 2–3 weeks |
| 15 | Cloud & DevOps | 4–6 weeks |
| 16 | Security | 2–3 weeks |
| 17 | DSA | 6–8 weeks |
| 18 | Architecture | 4–6 weeks |
| 19 | Lead Skills | Ongoing |
| 20 | Modern Java | 2–3 weeks |
| 21 | Libraries | 2–3 weeks |

> **Total: ~12–18 months** of dedicated study alongside hands-on project work.
> Build at least **3–5 real projects** that span multiple phases to cement understanding.

---

*This roadmap was designed to take you from absolute beginner to a confident Software Lead with deep Java expertise. Every topic matters — revisit earlier phases as your understanding deepens.*
