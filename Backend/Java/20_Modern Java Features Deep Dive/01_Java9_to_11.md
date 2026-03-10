# 20.1 — Java 9 to 11 Features

> **Versions covered:** Java 9 (2017) · Java 10 (2018) · Java 11 LTS (2018)
>
> These three releases transformed everyday Java coding. Java 9 introduced modularity and small but significant API additions. Java 10 gave us `var`. Java 11 — the first LTS after Java 8 — solidified the HTTP Client, enriched String and File APIs, and closed many daily pain points.

---

## ☕ Java 9 Features

### 20.1.1 JPMS — Java Platform Module System

The **Java Platform Module System** (also called Project Jigsaw) is the most architecturally significant change since Java 5. Before JPMS, the Java runtime was one giant `rt.jar` containing every class from `java.lang` to Swing to CORBA. As a result:

- You could not ship a lightweight JVM for a microservice without carrying the whole thing.
- Any class in `rt.jar` was accessible via reflection, creating security and encapsulation nightmares.
- "JAR hell" — where two JARs on the classpath expose the same package — was a genuine production problem.

JPMS solves all three. A **module** is a named, self-describing grouping of packages with explicit declarations of what it **exports** (makes visible) and what it **requires** (depends on).

#### Declaring a Module

Every module has a `module-info.java` file at the root of its source tree. This file is both documentation and enforcement.

```java
// module-info.java — lives at src/main/java/module-info.java
module com.example.orderservice {

    // Declare dependencies on other modules
    requires java.sql;                  // We use JDBC
    requires java.net.http;             // We use HttpClient (Java 11+)
    requires com.example.common;        // Another module we wrote

    // Transitive: anyone who requires orderservice also automatically gets common
    requires transitive com.example.common;

    // Export our public API packages — only these are visible to other modules
    exports com.example.orderservice.api;
    exports com.example.orderservice.model;

    // This package is only accessible to specific modules (e.g. test frameworks)
    exports com.example.orderservice.internal to com.example.orderservice.test;

    // Opens a package for deep reflection — needed by frameworks like Spring, Jackson
    // (opens allows reflective access to private members, exports only allows compile-time access)
    opens com.example.orderservice.model to com.fasterxml.jackson.databind;

    // Service Provider Interface (SPI) declarations
    uses com.example.orderservice.spi.PaymentProvider;   // We consume this service
    provides com.example.orderservice.spi.PaymentProvider  // We provide an implementation
        with com.example.orderservice.impl.StripePaymentProvider;
}
```

#### Module Types You Should Know

There are four kinds of modules. **Named modules** have a `module-info.java` and are the kind you write yourself. **Unnamed modules** are classic JARs on the classpath — every class in them belongs to the unnamed module, which can read everything (backward compatibility). **Automatic modules** are JARs placed on the *module path* without a `module-info.java`; the JVM derives a module name from the JAR filename and exports all packages — useful as a migration bridge. **Platform modules** are the JDK itself, now properly modularised (e.g. `java.base`, `java.sql`, `java.logging`).

#### Using `jlink` to Build a Custom JRE

One of the biggest practical wins of JPMS is `jlink`, which builds a minimal custom JRE containing only the modules your application needs.

```bash
# Analyse what modules your application actually uses
jdeps --module-path out --print-module-deps MyApp.jar
# Output: java.base,java.sql,java.net.http

# Build a custom JRE with only those modules — result is ~35 MB instead of 200 MB
jlink \
  --module-path $JAVA_HOME/jmods:out \
  --add-modules com.example.orderservice \
  --output custom-jre \
  --compress=2 \
  --strip-debug \
  --no-header-files
```

> **Best Practice:** For microservices containerised with Docker, combining `jlink` with a distroless base image can cut your image size from 400 MB down to 50–80 MB. This directly reduces cold-start time in Kubernetes.

> **Important:** JPMS adoption is gradual. Many popular frameworks (Spring, Hibernate) run on the unnamed module via the classpath, which is perfectly fine. You don't need to convert everything to named modules to benefit from JPMS — just understand the model.

---

### 20.1.2 JShell — The Java REPL

JShell is Java's interactive Read-Eval-Print Loop. Before JShell, testing a single expression meant creating a class, writing a `main` method, compiling, and running. JShell removes all that friction.

```bash
# Launch JShell from the terminal
$ jshell

|  Welcome to JShell -- Version 21
|  For an introduction type: /help intro

# You can type any expression and immediately see the result
jshell> 2 + 2
$1 ==> 4

# Declare a variable — JShell infers the type
jshell> var greeting = "Hello, Java 9!"
greeting ==> "Hello, Java 9!"

# Define a method — no class needed
jshell> int square(int n) { return n * n; }
|  created method square(int)

jshell> square(7)
$4 ==> 49

# Import classes as needed
jshell> import java.util.stream.*

jshell> IntStream.range(1, 6).map(n -> n * n).sum()
$6 ==> 55

# List defined variables and methods
jshell> /vars
jshell> /methods

# Exit
jshell> /exit
```

> **Use case:** JShell is invaluable for quickly checking API behaviour, experimenting with stream pipelines, verifying regex patterns, or teaching Java concepts without boilerplate. Think of it as a scratch pad that runs real Java.

---

### 20.1.3 Collection Factory Methods — `List.of()`, `Set.of()`, `Map.of()`

Before Java 9, creating a small immutable collection was verbose:

```java
// Pre-Java 9: Creating an unmodifiable list required two steps
List<String> old = Collections.unmodifiableList(Arrays.asList("a", "b", "c"));
```

Java 9 added concise factory methods that return **truly immutable** collections — not just unmodifiable wrappers, but implementations specifically built to reject all mutations.

```java
// Java 9+: Clean, concise, immutable collections
List<String> languages = List.of("Java", "Kotlin", "Scala");
Set<Integer> primes    = Set.of(2, 3, 5, 7, 11);
Map<String, Integer>  scores = Map.of(
    "Alice", 95,
    "Bob",   87,
    "Carol", 92
);

// For more than 10 map entries, use Map.ofEntries()
Map<String, Integer> largeMap = Map.ofEntries(
    Map.entry("key1", 1),
    Map.entry("key2", 2),
    Map.entry("key3", 3)
    // ... up to any size
);

// These collections are truly immutable — all of these throw UnsupportedOperationException
languages.add("Groovy");       // ❌ throws
primes.remove(2);              // ❌ throws
scores.put("Dave", 88);        // ❌ throws
```

**Critical distinction from `Arrays.asList()`:** `Arrays.asList()` returns a fixed-size list that *allows `set()`* but throws on `add()` or `remove()`. `List.of()` is completely immutable — no mutation at all.

**Other important properties to remember:**
- `Set.of()` and `Map.of()` do **not** guarantee insertion order (unlike `LinkedHashSet`).
- `List.of()` and `Set.of()` **do not allow null elements** — they throw `NullPointerException`. If you need a collection that can contain nulls, use `ArrayList` or `LinkedHashSet`.
- `Map.of()` throws `IllegalArgumentException` on duplicate keys.

```java
// Null handling — a common gotcha
List<String> withNull = List.of("a", null, "c");  // ❌ NullPointerException at creation time!

// Duplicate keys — another gotcha  
Map<String, Integer> dupKeys = Map.of("key", 1, "key", 2);  // ❌ IllegalArgumentException!
```

---

### 20.1.4 Stream Enhancements — `takeWhile()`, `dropWhile()`, `ofNullable()`, `iterate()`

Java 9 added four new Stream methods that fill in important gaps.

#### `takeWhile(Predicate)` and `dropWhile(Predicate)`

These work on **ordered streams** and deal with prefixes — they operate sequentially through the stream and stop (or skip) based on when a condition first becomes false. This makes them very different from `filter()`, which tests every element.

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// takeWhile: take elements FROM THE FRONT as long as the predicate holds.
// Stops the moment the predicate is false — does NOT look at remaining elements.
List<Integer> taken = numbers.stream()
    .takeWhile(n -> n < 5)
    .collect(Collectors.toList());
// Result: [1, 2, 3, 4]
// Note: element 5 failed the test, so 6,7,8,9,10 were also skipped — even though
// none of them are < 5 either. This is the key difference from filter().

// dropWhile: skip elements from the FRONT as long as the predicate holds.
// Once the predicate is false, all remaining elements pass through.
List<Integer> dropped = numbers.stream()
    .dropWhile(n -> n < 5)
    .collect(Collectors.toList());
// Result: [5, 6, 7, 8, 9, 10]

// Real-world use case: processing a log file where entries are timestamped
// and you want everything after a certain timestamp
List<LogEntry> relevantLogs = allLogs.stream()
    .dropWhile(entry -> entry.getTimestamp().isBefore(startTime))
    .collect(Collectors.toList());
```

#### `Stream.ofNullable(T t)`

Creates a stream of zero or one element — empty if the value is null, or a single-element stream otherwise. Solves a common problem when bridging nullable values into stream pipelines.

```java
// Before Java 9: bridging a nullable value into a stream required ugly null checks
String value = possiblyNullSource.getValue();
Stream<String> stream = value != null ? Stream.of(value) : Stream.empty();

// Java 9+: clean and expressive
Stream<String> stream = Stream.ofNullable(possiblyNullSource.getValue());

// Very useful in flatMap to discard nulls from a stream of optionally-present values
List<String> results = items.stream()
    .flatMap(item -> Stream.ofNullable(item.getDescription()))  // skip items with null description
    .collect(Collectors.toList());
```

#### `Stream.iterate()` with a Predicate (Java 9 enhancement)

Java 8 had `Stream.iterate(seed, unaryOp)` but it was infinite — you always needed a `limit()`. Java 9 adds a predicate argument to make it self-terminating, like a for-loop:

```java
// Java 8: infinite stream — must use limit() to stop
Stream.iterate(0, n -> n + 2)
      .limit(10)
      .forEach(System.out::println);  // 0, 2, 4, 6, 8, 10, 12, 14, 16, 18

// Java 9: finite iterate with a stop condition — much more like a for-loop
// Stream.iterate(seed, hasNextPredicate, nextFunction)
Stream.iterate(0, n -> n < 20, n -> n + 2)
      .forEach(System.out::println);  // 0, 2, 4, 6, 8, 10, 12, 14, 16, 18
// Equivalent to: for (int n = 0; n < 20; n += 2) { ... }

// Traversing a linked structure (e.g., walking up a directory tree)
Stream.iterate(startDir, dir -> dir.getParent() != null, dir -> dir.getParent())
      .forEach(dir -> System.out.println(dir.getFileName()));
```

---

### 20.1.5 Optional Enhancements — `ifPresentOrElse()`, `stream()`, `or()`

Java 9 added three new methods to `Optional<T>` that cover cases the original Java 8 API didn't handle cleanly.

```java
Optional<String> name = findUserName(userId);

// ifPresentOrElse(): handle both the present AND absent case in one call.
// Java 8 had ifPresent() but no clean way to also handle the absent case.
name.ifPresentOrElse(
    value -> System.out.println("Found user: " + value),
    ()    -> System.out.println("User not found")
);

// or(): provide an alternative Optional if this one is empty.
// Different from orElse() which provides a value — this provides another Optional,
// which is useful for chaining fallback strategies.
Optional<String> resolvedName = name
    .or(() -> findUserNameInCache(userId))   // try cache if primary source is empty
    .or(() -> Optional.of("Anonymous"));     // final fallback

// stream(): convert Optional to a Stream of 0 or 1 element.
// This is incredibly useful for flatMapping a Stream<Optional<T>> into a Stream<T>.
List<Optional<String>> optionals = List.of(
    Optional.of("Alice"),
    Optional.empty(),
    Optional.of("Carol"),
    Optional.empty()
);

// Without stream(): ugly filter + map
List<String> names1 = optionals.stream()
    .filter(Optional::isPresent)
    .map(Optional::get)
    .collect(Collectors.toList());

// With stream(): clean flatMap
List<String> names2 = optionals.stream()
    .flatMap(Optional::stream)   // Each Optional becomes a 0-or-1 element stream
    .collect(Collectors.toList());
// Result: ["Alice", "Carol"]
```

---

### 20.1.6 ProcessHandle API

`ProcessHandle` gives Java programs first-class access to OS processes — something previously requiring OS-specific native code or hacking `Runtime.exec()`.

```java
// Get information about the current JVM process
ProcessHandle current = ProcessHandle.current();
ProcessHandle.Info info = current.info();

System.out.println("PID: " + current.pid());
info.command().ifPresent(cmd -> System.out.println("Command: " + cmd));
info.user().ifPresent(user -> System.out.println("User: " + user));
info.startInstant().ifPresent(t -> System.out.println("Started: " + t));
info.totalCpuDuration().ifPresent(d -> System.out.println("CPU time: " + d));

// Find a specific process by PID and destroy it
Optional<ProcessHandle> process = ProcessHandle.of(1234);
process.ifPresent(p -> {
    System.out.println("Is alive: " + p.isAlive());
    p.destroy();      // Graceful termination (SIGTERM)
    // p.destroyForcibly();  // Force kill (SIGKILL)
});

// List all running processes visible to the JVM
ProcessHandle.allProcesses()
    .filter(p -> p.info().command().map(c -> c.contains("java")).orElse(false))
    .forEach(p -> System.out.println("Java process: " + p.pid()));

// Execute a process and handle completion asynchronously
Process proc = new ProcessBuilder("ls", "-la").start();
ProcessHandle handle = proc.toHandle();
handle.onExit().thenAccept(p -> System.out.println("Process " + p.pid() + " exited"));
```

---

## ☕ Java 10 Features

### 20.1.7 Local Variable Type Inference — `var`

`var` is arguably the most impactful Java 10 change in daily coding. It lets the compiler infer the type of a local variable from its initialiser, eliminating redundancy without losing type safety.

**Crucially, Java remains statically typed.** `var` is not like JavaScript's `var` (which is dynamic). The inferred type is fixed at compile time and full IDE tooling works exactly as before.

```java
// Before var: repeating type information twice
ArrayList<Map<String, List<Integer>>> data = new ArrayList<Map<String, List<Integer>>>();
Map<String, List<String>> groupedNames = new HashMap<String, List<String>>();

// With var: the right-hand side already tells you the type — no need to repeat it
var data         = new ArrayList<Map<String, List<Integer>>>();
var groupedNames = new HashMap<String, List<String>>();

// var is especially clean with loop variables and streams
for (var entry : scoreMap.entrySet()) {      // entry is Map.Entry<String, Integer>
    System.out.println(entry.getKey() + " = " + entry.getValue());
}

// var with streams — readable and avoids verbose generic type spellings
var result = users.stream()
    .filter(user -> user.isActive())
    .map(User::getName)
    .collect(Collectors.toUnmodifiableList());

// try-with-resources: very clean with var
try (var connection = dataSource.getConnection();
     var statement  = connection.prepareStatement(sql)) {
    // ...
}
```

#### Where `var` Cannot Be Used

`var` is strictly limited to **local variable declarations with an initialiser**. Understanding the boundaries is essential:

```java
// ❌ Cannot use var for fields
class Order {
    var id = 0;           // Compile error — only for local variables
}

// ❌ Cannot use var for method parameters
void process(var item) { }  // Compile error

// ❌ Cannot use var for return types
var getTotal() { return total; }  // Compile error

// ❌ Cannot use var without an initialiser
var x;  // Compile error — compiler needs the right-hand side to infer the type

// ❌ Cannot use var when the initialiser is null
var name = null;  // Compile error — null has no type to infer from

// ❌ Cannot use var with array initialisers (without new)
var numbers = {1, 2, 3};  // Compile error

// ✅ These all work
var i = 0;
var message = "Hello";
var list = new ArrayList<String>();
var numbers = new int[]{1, 2, 3};  // This works — explicit type on the right
```

#### When to Use `var` and When Not To

The goal of `var` is **readability**, not laziness. Use it when the type is already obvious from the right-hand side. Avoid it when removing the type makes the code harder to understand.

```java
// ✅ Good use: right-hand side makes the type completely obvious
var users        = new ArrayList<User>();
var connection   = dataSource.getConnection();
var mapper       = new ObjectMapper();
var count        = users.size();   // clearly int

// ❌ Poor use: removes useful type information, making intent unclear
var result = processData(items);   // What type does processData() return? Not obvious.
var x      = getValue();           // Is this int? long? double? String?

// ❌ Poor use in lambda — lambdas are already concise, var doesn't help
var f = (String s) -> s.toUpperCase();  // Just write: Function<String,String> f = ...
                                          // or better: capture it in a properly typed variable

// One legitimate use of var in lambdas (Java 11): adding annotations to parameters
// (This was the main reason var was allowed in lambda params in Java 11)
var sorted = list.stream().sorted(
    (@NonNull var a, @NonNull var b) -> a.compareTo(b)
);
```

---

### 20.1.8 `List.copyOf()`, `Map.copyOf()`, `Set.copyOf()`

These create an immutable copy of an existing collection. Unlike wrapping with `Collections.unmodifiableList()`, which is just a view (mutations to the original still show through), `copyOf()` creates a completely independent snapshot.

```java
List<String> mutable = new ArrayList<>(List.of("a", "b", "c"));

List<String> immutableCopy = List.copyOf(mutable);  // Independent immutable copy

mutable.add("d");  // Modifying the original...
System.out.println(immutableCopy);  // [a, b, c] — copy is unaffected

// Also filters out nothing — but does NOT allow null elements (same as List.of)
// If the source contains nulls, List.copyOf() throws NullPointerException.
```

### 20.1.9 `Collectors.toUnmodifiableList()`, `toUnmodifiableSet()`, `toUnmodifiableMap()`

Collect a stream directly into an unmodifiable collection:

```java
// Before Java 10: two-step process
List<String> names = users.stream()
    .map(User::getName)
    .collect(Collectors.collectingAndThen(
        Collectors.toList(),
        Collections::unmodifiableList
    ));

// Java 10+: one clean step
List<String> names = users.stream()
    .map(User::getName)
    .collect(Collectors.toUnmodifiableList());
```

---

## ☕ Java 11 Features

### 20.1.10 String API Enhancements

Java 11 added several String methods that were conspicuously missing from the standard library.

```java
// isBlank() — true if the string is empty OR contains only whitespace
// DIFFERENT from isEmpty(): "   ".isEmpty() == false, but "   ".isBlank() == true
"".isBlank();          // true
"   ".isBlank();       // true
"  hi  ".isBlank();    // false
"hello".isBlank();     // false

// lines() — splits a multi-line string into a Stream<String> lazily
// Handles \n, \r\n, and \r line endings correctly
String multiLine = "Line 1\nLine 2\nLine 3";
long lineCount = multiLine.lines().count();   // 3
multiLine.lines().forEach(System.out::println);

// strip(), stripLeading(), stripTrailing() — Unicode-aware whitespace trimming
// The old trim() only removed ASCII whitespace (chars <= '\u0020').
// strip() uses Character.isWhitespace(), which correctly handles all Unicode whitespace.
String padded = "  hello  ";
padded.strip();          // "hello"
padded.stripLeading();   // "hello  "
padded.stripTrailing();  // "  hello"

// repeat(int count) — repeat a string n times
"ha".repeat(3);          // "hahaha"
"-".repeat(40);          // "----------------------------------------" (divider line)
"  ".repeat(level);      // Useful for indentation in code generators

// Practical combination — parsing CSV-like text
String data = """
    Alice,25,Engineer
    Bob,30,Manager
    Carol,28,Designer
    """;

List<String[]> records = data.lines()
    .map(String::strip)
    .filter(line -> !line.isBlank())
    .map(line -> line.split(","))
    .collect(Collectors.toList());
```

---

### 20.1.11 Files API Enhancements — `readString()` and `writeString()`

Before Java 11, reading an entire file as a String required boilerplate. Java 11 made it a one-liner.

```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;

// Before Java 11: reading a file as String was clunky
byte[] bytes = Files.readAllBytes(Path.of("config.json"));
String content = new String(bytes, StandardCharsets.UTF_8);

// Java 11: clean and direct
String content = Files.readString(Path.of("config.json"));
// Defaults to UTF-8; can also specify charset:
String content = Files.readString(Path.of("data.txt"), StandardCharsets.ISO_8859_1);

// Writing a String to a file
Files.writeString(Path.of("output.json"), jsonContent);
// With specific charset and options:
Files.writeString(
    Path.of("log.txt"),
    "Entry: " + LocalDateTime.now(),
    StandardCharsets.UTF_8,
    StandardOpenOption.CREATE,
    StandardOpenOption.APPEND
);

// Practical example: read a JSON config, modify, write back
Path configPath = Path.of("application.json");
String json = Files.readString(configPath);
String updated = json.replace("\"debug\": false", "\"debug\": true");
Files.writeString(configPath, updated);
```

---

### 20.1.12 HTTP Client API (Stable in Java 11)

The new `java.net.http.HttpClient` was incubating in Java 9/10 and became stable and final in Java 11. It replaces the ancient `HttpURLConnection` with a modern, fluent, async-capable API that supports HTTP/1.1 and HTTP/2.

```java
import java.net.http.*;
import java.net.URI;
import java.time.Duration;

// Step 1: Create a reusable client (create once, share across the application)
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)          // Prefer HTTP/2, fall back to HTTP/1.1
    .connectTimeout(Duration.ofSeconds(10))
    .followRedirects(HttpClient.Redirect.NORMAL)
    .build();

// Step 2: Build a request
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users/42"))
    .header("Accept", "application/json")
    .header("Authorization", "Bearer " + token)
    .timeout(Duration.ofSeconds(30))
    .GET()
    .build();

// Step 3a: Synchronous send — blocks the calling thread
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
System.out.println("Status: " + response.statusCode());
System.out.println("Body: " + response.body());
System.out.println("Content-Type: " + response.headers().firstValue("Content-Type"));

// Step 3b: Asynchronous send — returns CompletableFuture, never blocks
CompletableFuture<HttpResponse<String>> futureResponse =
    client.sendAsync(request, HttpResponse.BodyHandlers.ofString());

futureResponse
    .thenApply(HttpResponse::body)
    .thenAccept(body -> processResponse(body))
    .exceptionally(ex -> { log.error("HTTP call failed", ex); return null; });

// POST request with a JSON body
HttpRequest postRequest = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/orders"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString("""
        {
            "productId": "SKU-001",
            "quantity": 2,
            "customerId": "cust-42"
        }
        """))
    .build();

HttpResponse<String> postResponse = client.send(postRequest, HttpResponse.BodyHandlers.ofString());
if (postResponse.statusCode() == 201) {
    System.out.println("Order created: " + postResponse.body());
}

// Downloading a file directly to disk (stream response body to a file)
HttpResponse<Path> fileResponse = client.send(
    HttpRequest.newBuilder().uri(URI.create("https://files.example.com/report.pdf")).GET().build(),
    HttpResponse.BodyHandlers.ofFile(Path.of("report.pdf"))
);
System.out.println("Downloaded to: " + fileResponse.body());
```

> **Best Practice:** Create one `HttpClient` instance and reuse it across all requests. Each client maintains its own connection pool, so creating a new client per request defeats the purpose of HTTP/2 multiplexing and wastes resources.

---

### 20.1.13 `Predicate.not()` — Negating Method References

Java 8 introduced `Predicate.negate()` for instances, but it didn't work cleanly with method references. Java 11 adds the static `Predicate.not()` factory that wraps any method reference in a negation.

```java
import java.util.function.Predicate;

List<String> lines = List.of("Hello", "", "World", "  ", "Java 11");

// Before Java 11: awkward lambda workaround to negate a method reference
List<String> nonBlank1 = lines.stream()
    .filter(s -> !s.isBlank())   // works but not as expressive
    .collect(Collectors.toList());

// Java 11+: use Predicate.not() to cleanly negate a method reference
List<String> nonBlank2 = lines.stream()
    .filter(Predicate.not(String::isBlank))   // reads naturally: "not blank"
    .collect(Collectors.toList());

// Another practical example — filtering out nulls and empties from a stream
List<String> cleaned = rawValues.stream()
    .filter(Predicate.not(Objects::isNull))
    .filter(Predicate.not(String::isEmpty))
    .collect(Collectors.toList());
```

---

## 💡 Phase 20.1 — Key Takeaways

Java 9–11 were three of the most productivity-enhancing releases in Java's history. JPMS brought true modular encapsulation. JShell removed the barrier to experimentation. `var` reduced repetition without sacrificing type safety. The collection factory methods made immutable data structures a natural default. The new String methods closed long-standing gaps. The HTTP Client replaced a decades-old pain point with a modern, fluent API. Together, these releases represent the platform making a serious commitment to developer ergonomics — and Java 11's LTS status means these features are production-stable in almost every enterprise environment you'll encounter.
