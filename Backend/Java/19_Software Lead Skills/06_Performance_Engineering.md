# 19.6 — Performance Engineering in Java

> **Goal:** Learn how to identify, measure, and eliminate performance bottlenecks in Java systems — using profiling, load testing, benchmarking, and architectural optimizations. The golden rule: always measure before you optimize.

---

## 🧠 The First Principle: Measure Before You Optimize

Performance engineering is often done backwards. Engineers guess where the bottleneck is, optimize based on intuition, and end up with faster code in the wrong place — while the actual problem remains. Donald Knuth's famous observation that "premature optimization is the root of all evil" is often misquoted as a dismissal of performance work entirely; what it actually means is that you should not optimize code that isn't a bottleneck, because you pay the cost in complexity and maintainability without gaining anything.

The correct process is: measure the system under realistic load, identify the actual bottleneck, form a hypothesis about why it's slow, make one targeted change, measure again to confirm the improvement. This sounds obvious but it requires discipline — especially when "I bet the problem is the database" is a tempting shortcut past the measurement step.

---

## 🔍 Identifying Bottlenecks — Profiling First

A profiler is a tool that measures where a program spends its time (CPU profiling) or where it allocates memory (allocation profiling). It attaches to a running JVM and produces a report showing you exactly which methods are consuming the most resources. Without a profiler, you're guessing. With one, you can see the truth in minutes.

### async-profiler — Low-Overhead Production Profiling

`async-profiler` is the most widely used Java profiler for production systems because it has very low overhead (typically 1–3% CPU) and can profile a running JVM without restarting it. It captures CPU samples and allocation events using OS-level APIs, giving you an accurate picture without the observer effect that plagues older profilers.

```bash
# Download async-profiler
wget https://github.com/async-profiler/async-profiler/releases/download/v3.0/async-profiler-3.0-linux-x64.tar.gz
tar xzf async-profiler-3.0-linux-x64.tar.gz

# Find your Java process ID
jps -l
# Output: 12345 com.example.OrderService

# Profile CPU for 30 seconds, generate a flame graph
./profiler.sh -d 30 -f /tmp/flamegraph.html 12345

# Profile memory allocations for 30 seconds
./profiler.sh -e alloc -d 30 -f /tmp/alloc-flamegraph.html 12345
```

The output is a **flame graph** — a visualization where the x-axis represents the total proportion of time sampled, and each horizontal block represents a method call. Wide blocks at the top of a stack are the methods consuming the most time. The first time you see your application's flame graph, it's often surprising: you'll find that 40% of CPU time is being spent somewhere you never expected.

### Reading a Flame Graph in a Spring Boot App

A typical finding might look like this:

```
[Wide block] OrderService.processOrder (35% of all CPU time)
  └─ [Wide block] ProductRepository.findByIds (28% — dominates!)
       └─ HibernateJpaDialect.translateExceptionIfPossible
            └─ AbstractEntityPersister.load
                 └─ [JDBC calls]
```

This flame graph is telling you: 28% of your CPU time is being spent loading products one at a time. This is the N+1 query problem — a perfect candidate for optimization. Without the profiler, you might have spent time optimizing your JSON serialization or adding caching for the wrong layer.

### JVM Built-in Tools

The standard JDK ships with diagnostic tools that don't require installing anything extra. They're less powerful than async-profiler but always available:

```bash
# jstack: Dump all thread stacks — invaluable for diagnosing deadlocks and thread contention
jstack 12345 > thread-dump.txt
# Look for threads in BLOCKED state, or threads all waiting on the same lock

# jmap: Generate a heap histogram — shows which classes are consuming the most memory
jmap -histo:live 12345 | head -30
# Output:
#  num     instances         bytes  class name
#    1:      892345      71387600  [B (byte arrays)
#    2:      445678      35654240  java.lang.String
# If you see unexpected classes with huge byte counts, you have a memory leak

# jstat: Monitor GC behavior in real time
jstat -gcutil 12345 1000  # Print GC stats every 1000ms
# Output: S0  S1  E    O    M    YGC YGCT  FGC FGCT  GCT
#         0.0 0.0 72.5 85.3 97.2  42 1.234   3 0.892 2.126
# High "O" (Old generation) approaching 100% means you're heading for a Full GC event
```

### Java Flight Recorder (JFR) + Java Mission Control (JMC)

JFR is a production-grade, low-overhead profiling framework built into the JVM since Java 11. It continuously records events (GC, thread state, method calls, I/O, exceptions, lock contention) with less than 1% overhead — safe enough to run in production continuously.

```java
// Enable JFR programmatically in your application
// This is useful for capturing a recording of a specific operation

import jdk.jfr.Recording;
import java.nio.file.Path;
import java.time.Duration;

public class PerformanceDiagnostics {
    
    public static void captureRecording(Runnable operation, Path outputFile) throws Exception {
        try (Recording recording = new Recording()) {
            // Configure what to record
            recording.enable("jdk.CPUSample").withPeriod(Duration.ofMillis(10));
            recording.enable("jdk.ObjectAllocationInNewTLAB");
            recording.enable("jdk.JavaMonitorWait");  // Lock contention
            recording.enable("jdk.GCHeapSummary");
            recording.enable("jdk.ExceptionStatistics");
            
            recording.start();
            operation.run();  // Execute the code under measurement
            recording.stop();
            
            recording.dump(outputFile);
            // Open the .jfr file in Java Mission Control for visual analysis
        }
    }
}
```

Open the resulting `.jfr` file in JMC (available free from Oracle or Eclipse Adoptium) to see a rich timeline of JVM behavior during your recording, including memory allocation hot spots, lock contention, GC events, and CPU hot methods.

---

## ⚡ Load Testing — JMeter and Gatling

Profiling tells you where time is spent inside the JVM. Load testing tells you how your system behaves under concurrent user load — something that individual profiling cannot reveal, because many performance problems (connection pool exhaustion, lock contention, thundering herd effects) only manifest under concurrent access.

### Gatling — Code-First Load Testing

Gatling is a modern load testing tool popular in Java ecosystems because its scenarios are written in Scala (or Java), making them version-controllable and reviewable like application code.

```java
// Gatling simulation in Java DSL (Gatling 3.9+)
// Place this in src/test/java/simulations/

import io.gatling.javaapi.core.*;
import io.gatling.javaapi.http.*;
import static io.gatling.javaapi.core.CoreDsl.*;
import static io.gatling.javaapi.http.HttpDsl.*;

public class OrderServiceLoadTest extends Simulation {

    // Configure the base URL and common headers
    HttpProtocolBuilder httpProtocol = http
        .baseUrl("http://localhost:8080")
        .acceptHeader("application/json")
        .contentTypeHeader("application/json");

    // Define a realistic user journey: browse products, add to cart, place order
    ScenarioBuilder purchaseFlow = scenario("Purchase Flow")
        .exec(
            http("List products")
                .get("/api/v1/products?category=electronics&limit=20")
                .check(status().is(200))
        )
        .pause(2, 5)  // Simulate user reading the page (2-5 second pause)
        .exec(
            http("View product detail")
                .get("/api/v1/products/#{productId}")
                .check(status().is(200))
        )
        .pause(3, 8)
        .exec(
            http("Place order")
                .post("/api/v1/orders")
                .body(StringBody("""
                    {
                        "items": [{"productId": "#{productId}", "quantity": 1}]
                    }
                    """))
                .check(status().is(201))
                .check(jsonPath("$.orderId").saveAs("orderId"))
        );

    {
        // Define the load shape:
        // - Ramp up from 0 to 50 concurrent users over 60 seconds
        // - Hold at 50 users for 5 minutes
        // - Ramp down over 30 seconds
        setUp(
            purchaseFlow.injectOpen(
                rampUsers(50).during(60),
                constantUsersPerSec(50).during(300),
                rampUsersPerSec(50).to(0).during(30)
            )
        ).protocols(httpProtocol)
         .assertions(
             global().responseTime().percentile(95).lt(500),  // p95 < 500ms
             global().successfulRequests().percent().gt(99.0)  // 99%+ success rate
         );
    }
}
```

Run it with: `mvn gatling:test`. Gatling generates an HTML report with response time percentiles, throughput over time, and error rates — everything you need to establish a performance baseline and detect regressions.

**The key principle in load test design** is to model **realistic user behavior**, not just endpoint hammer tests. A test that sends 1000 concurrent requests to the same endpoint tells you much less than one that simulates 1000 users going through a realistic workflow, with realistic think times between actions.

---

## 📏 JMH — Java Microbenchmark Harness

JMH is the standard tool for benchmarking small pieces of Java code — a specific method, algorithm, or data structure operation. It's specifically designed to avoid the many pitfalls of naive benchmarking in the JVM (JIT warmup, dead code elimination, branch prediction effects, GC interference).

The classic mistake is benchmarking without JMH:

```java
// ❌ WRONG: This is not a valid benchmark
// The JVM JIT compiler will optimize this loop after ~10,000 iterations (warmup),
// making later iterations far faster than the first few. You're not measuring 
// steady-state performance — you're measuring a mix of interpreted and compiled execution.
long start = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) {
    result = myMethod(i);
}
long elapsed = System.nanoTime() - start;
System.out.println("Time: " + elapsed / 1_000_000 + "ms");
```

JMH handles all of these subtleties correctly:

```java
// Add JMH dependency to pom.xml:
// <dependency>
//   <groupId>org.openjdk.jmh</groupId>
//   <artifactId>jmh-core</artifactId>
//   <version>1.37</version>
// </dependency>

@BenchmarkMode(Mode.AverageTime)     // Report average time per operation
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@State(Scope.Thread)                  // Each thread gets its own state instance
@Warmup(iterations = 5, time = 1)    // 5 warmup iterations to let JIT compile
@Measurement(iterations = 10, time = 1) // 10 measured iterations
@Fork(2)                              // Run in 2 separate JVM processes for stability
public class StringConcatenationBenchmark {

    private final int LIST_SIZE = 1000;
    private List<String> words;

    @Setup
    public void setup() {
        // Prepare input data that represents realistic usage
        words = IntStream.range(0, LIST_SIZE)
            .mapToObj(i -> "word" + i)
            .collect(Collectors.toList());
    }

    @Benchmark
    public String concatenateWithPlus() {
        // Naive approach: creates a new String object on each iteration
        String result = "";
        for (String word : words) {
            result += word + " ";
        }
        return result;
    }

    @Benchmark
    public String concatenateWithStringBuilder() {
        // Efficient approach: one mutable buffer, no intermediate objects
        StringBuilder sb = new StringBuilder(LIST_SIZE * 8);  // Pre-size to avoid resizing
        for (String word : words) {
            sb.append(word).append(' ');
        }
        return sb.toString();
    }

    @Benchmark
    public String concatenateWithJoin() {
        // Modern approach using String.join
        return String.join(" ", words);
    }
}
```

Run with: `mvn package && java -jar target/benchmarks.jar`

Typical output:
```
Benchmark                                    Mode  Cnt    Score    Error  Units
StringConcatenationBenchmark.concatenateWithPlus     avgt   20  1847.234 ± 34.521  us/op
StringConcatenationBenchmark.concatenateWithStringBuilder  avgt   20    12.873 ±  0.412  us/op
StringConcatenationBenchmark.concatenateWithJoin avgt   20     9.124 ±  0.203  us/op
```

The `+` operator is 143x slower than `String.join()` for 1000 strings. Without a benchmark, you'd rely on intuition; with JMH, you have proof.

---

## 🗄️ Database Query Optimization

Database performance is often the dominant bottleneck in Java web applications. The most common culprits are the ones identifiable through SQL `EXPLAIN ANALYZE`.

### EXPLAIN ANALYZE in PostgreSQL

Every time you suspect a slow query, run it through `EXPLAIN ANALYZE` before guessing at a fix. It shows the query execution plan and actual execution statistics:

```sql
-- A slow query: find all recent orders for customers in a specific region
EXPLAIN ANALYZE
SELECT o.id, o.created_at, c.name, c.email
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.region = 'NORTH_AMERICA'
  AND o.created_at > NOW() - INTERVAL '30 days'
ORDER BY o.created_at DESC;

-- Example output:
-- Sort  (cost=12450.83..12457.71 rows=2750 width=72) (actual time=2847.321..2849.104 rows=2847)
--   Sort Key: o.created_at DESC
--   ->  Hash Join  (cost=4234.15..12295.81 rows=2750 width=72) (actual time=234.521..2834.218 rows=2847)
--         ->  Seq Scan on orders o  (cost=0.00..7432.11 rows=52341 width=24) (actual time=0.023..1832.421 rows=52341)
--               Filter: (created_at > (now() - '30 days'::interval))
--         ->  Hash  (cost=3823.92..3823.92 rows=32818 width=52) (actual time=233.421..233.421 rows=32818)
--               ->  Seq Scan on customers c  (cost=0.00..3823.92 rows=32818 width=52) (actual time=0.021..223.421 rows=32818)
--                     Filter: ((region)::text = 'NORTH_AMERICA'::text)
-- Planning Time: 3.421 ms
-- Execution Time: 2851.234 ms   <-- 2.85 seconds!
```

The `Seq Scan` on both tables is the problem — the database is reading every single row in both tables and filtering in memory. Adding indexes on the filtered columns changes this dramatically:

```sql
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
CREATE INDEX idx_customers_region ON customers(region);

-- Now EXPLAIN ANALYZE shows:
-- Index Scan on orders o  (actual time=0.043..8.234 rows=2847)
-- Index Scan on customers c  (actual time=0.021..4.123 rows=8234)
-- Execution Time: 14.521 ms   <-- 196x faster!
```

In your Spring application, the Spring Data JPA `@Query` annotation is where you'd write this query, and the index creation belongs in a Flyway migration file:

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Without @Query, Spring Data would generate a JPQL query that might
    // not be as efficient. For performance-critical queries, always review
    // the generated SQL with spring.jpa.show-sql=true
    @Query("""
        SELECT o FROM Order o
        JOIN FETCH o.customer c
        WHERE c.region = :region
          AND o.createdAt > :since
        ORDER BY o.createdAt DESC
        """)
    List<Order> findRecentOrdersByRegion(
        @Param("region") String region,
        @Param("since") LocalDateTime since
    );
}
```

```sql
-- V5__add_order_performance_indexes.sql (Flyway migration)
CREATE INDEX CONCURRENTLY idx_orders_created_at ON orders(created_at DESC);
CREATE INDEX CONCURRENTLY idx_customers_region ON customers(region);
-- CONCURRENTLY means the index builds without locking the table — safe for production
```

---

## 🗃️ Caching Strategy and Implementation

Caching is the most effective performance improvement for read-heavy workloads, but it requires careful thought — an incorrect caching strategy can serve stale data or cause subtle consistency bugs worse than the original performance problem.

### Caffeine — High-Performance Local Cache

Caffeine is the standard choice for in-process (local) caching in Java. It's dramatically faster than older alternatives like Ehcache or Guava Cache, using a sophisticated eviction algorithm (TinyLFU) that achieves near-optimal hit rates.

```java
// Direct Caffeine usage — maximum control
@Configuration
public class CacheConfig {
    
    @Bean
    public Cache<Long, Product> productCache() {
        return Caffeine.newBuilder()
            .maximumSize(10_000)              // Evict least-used items after 10k entries
            .expireAfterWrite(5, TimeUnit.MINUTES)  // Items expire 5 min after being written
            .expireAfterAccess(2, TimeUnit.MINUTES) // Also expire if not accessed for 2 min
            .recordStats()                    // Enable hit rate metrics
            .build();
    }
}

@Service
public class ProductService {
    
    private final Cache<Long, Product> productCache;
    private final ProductRepository repository;

    // Cache-aside pattern: look in cache first, load from DB on miss
    public Product findById(Long id) {
        return productCache.get(id, productId -> {
            // This lambda only runs on a cache miss — it loads from the database
            log.debug("Cache miss for product {}, loading from database", productId);
            return repository.findById(productId)
                .orElseThrow(() -> new ProductNotFoundException(productId));
        });
    }
    
    // Invalidate the cache when a product is updated
    public Product updateProduct(Long id, ProductUpdateRequest request) {
        Product updated = repository.save(/* ... */);
        productCache.invalidate(id);  // Force the next read to reload from DB
        return updated;
    }
}
```

### Spring Cache Abstraction — Simpler Caching with Annotations

For less performance-critical caching, Spring's cache abstraction lets you add caching declaratively without changing your service logic:

```java
@Service
@CacheConfig(cacheNames = "products")  // Default cache name for all methods in this class
public class ProductService {

    // Cache the result. Next call with same ID returns cached value — no DB hit.
    @Cacheable(key = "#id")
    public ProductDTO getProduct(Long id) {
        // This method body only executes on a cache miss
        return productRepository.findById(id)
            .map(productMapper::toDTO)
            .orElseThrow(() -> new ProductNotFoundException(id));
    }

    // Remove the cached entry when the product is updated
    @CacheEvict(key = "#id")
    public ProductDTO updateProduct(Long id, ProductUpdateRequest request) {
        // ...
    }

    // Update the cached value when a product is saved (avoids an extra DB read)
    @CachePut(key = "#result.id")
    public ProductDTO createProduct(CreateProductRequest request) {
        // Returns the new product, which Spring stores in the cache automatically
        return productMapper.toDTO(productRepository.save(/* ... */));
    }
}
```

Configure the underlying cache provider (Caffeine for local, Redis for distributed) in `application.yml`:

```yaml
spring:
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=10000,expireAfterWrite=5m,recordStats
```

### When Local Cache vs. Distributed Cache

Local cache (Caffeine) is appropriate when you can tolerate slightly stale data, when the cached data is the same for all users, and when you have a small enough dataset that each application instance can hold it in memory. Its advantage is zero network latency — a cache hit is just a HashMap lookup.

Distributed cache (Redis via Spring Data Redis) is appropriate when you have multiple application instances and need consistency between them (updating a product on instance A should be visible on instance B), when the dataset is too large to fit in each instance's memory, or when you need cache survival across application restarts.

---

## 📊 Async Processing for Non-Critical Paths

Not everything needs to happen synchronously in the request-response cycle. Identifying operations that the user doesn't need to wait for and moving them to asynchronous execution is one of the most impactful architectural performance improvements.

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final EmailService emailService;
    private final InventoryService inventoryService;
    private final ApplicationEventPublisher eventPublisher;

    // The user only needs to wait for the order to be saved.
    // Email and analytics can happen in the background.
    @Transactional
    public OrderResponse createOrder(CreateOrderRequest request) {
        Order order = orderRepository.save(buildOrder(request));
        
        // Publish an event — listeners handle the async work
        // The transaction commits BEFORE the event is processed
        eventPublisher.publishEvent(new OrderCreatedEvent(order.getId()));
        
        // Return immediately — user gets a fast response
        return OrderResponse.from(order);
    }
}

@Component
public class OrderCreatedEventListener {
    
    // @Async means this runs in a different thread — doesn't block the HTTP response
    // @TransactionalEventListener ensures this runs AFTER the creating transaction commits
    @Async
    @TransactionalEventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        // These operations are important but the user doesn't wait for them:
        emailService.sendOrderConfirmation(event.getOrderId());     // ~200ms
        analyticsService.trackOrderCreated(event.getOrderId());     // ~50ms
        // Total: ~250ms of work the user used to wait for, now async
    }
}

// Configure the thread pool for @Async tasks
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean("taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-task-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

The performance win here is direct: before, the user's request took 300ms (100ms DB + 200ms email + 50ms analytics) and waited for all of it. After, the response time is 110ms (100ms DB + 10ms event publish) and the rest happens in the background. For the user, the service just became 3x faster.

---

## 💡 Key Takeaways

Performance engineering is a discipline, not a one-time activity. It requires a culture of measurement — establishing baselines, tracking regressions in CI (using Gatling for load tests and JMH for micro-benchmarks), and treating performance regressions as bugs. The tools covered here — async-profiler for CPU profiling, JFR/JMC for JVM internals, Gatling for load testing, JMH for micro-benchmarks, EXPLAIN ANALYZE for database queries, and Caffeine/Redis for caching — form a complete toolkit for a Java performance engineer. Master the workflow: measure first, hypothesize, change one thing, measure again.
