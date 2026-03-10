# 21.4 — Caching Libraries

> **Libraries covered:** Caffeine · Ehcache · Redis (Lettuce & Jedis) · Hazelcast · Spring Cache Abstraction
>
> Caching is one of the most impactful performance techniques available to a Java developer, and one of the most dangerous if done carelessly. A cache stores the result of an expensive operation so that subsequent requests can be served from memory instead of repeating the work. The challenge is correctness: a cache that serves stale, inconsistent, or incorrect data is worse than no cache at all. Understanding the architectural tradeoffs between local in-process caches and distributed caches — and knowing when each is appropriate — is as important as knowing the APIs.

---

## 🧠 Caching Architecture Fundamentals

Before looking at specific libraries, it is worth grounding yourself in the two fundamental cache architectures and when to choose each.

A **local cache** (also called an in-process cache) lives inside the JVM heap, in the same memory space as your application. Reads and writes are essentially HashMap lookups — nanoseconds to microseconds. The limitation is that each application instance has its own independent cache. If you have three instances of a service running behind a load balancer, a cache update on instance A is invisible to instances B and C. This means local caches work correctly only for data that is either read-only, or where eventual consistency (different instances briefly serving slightly different data) is acceptable.

A **distributed cache** (like Redis or Hazelcast) is a shared cache that all application instances access over the network. Every instance sees the same data, so cache updates are immediately consistent across the cluster. The tradeoff is network latency — a distributed cache hit takes microseconds to milliseconds rather than nanoseconds. For applications with multiple instances that need cache consistency, a distributed cache is the correct choice.

---

## ⚡ Caffeine — The Fastest Local Cache

Caffeine is the modern successor to Guava Cache and is now the standard choice for in-process caching in the Java ecosystem. Spring Boot uses Caffeine as its default local cache provider when the caffeine JAR is on the classpath. It achieves near-optimal cache hit rates using a sophisticated eviction algorithm called **Window TinyLFU (W-TinyLFU)**, which keeps track of how frequently and recently items were accessed and uses this to make smarter eviction decisions than a simple LRU policy.

### 21.4.1 Caffeine — Core API

```java
import com.github.benmanes.caffeine.cache.*;
import java.util.concurrent.TimeUnit;

// ── Manual cache — you handle loading logic yourself ─────────────────────────
Cache<Long, Product> productCache = Caffeine.newBuilder()
    .maximumSize(10_000)                     // Evict when entry count exceeds 10,000
    .expireAfterWrite(10, TimeUnit.MINUTES)  // Expire 10 minutes after insertion or update
    .expireAfterAccess(5, TimeUnit.MINUTES)  // ALSO expire if not accessed in 5 minutes
    .recordStats()                           // Enable hit/miss rate statistics
    .removalListener((key, value, cause) ->  // Called when any entry is removed
        log.debug("Cache evicted: key={}, cause={}", key, cause))
    .build();

// get() — the fundamental cache-aside operation
// If key is present, returns cached value. If absent, runs the loader function ONCE,
// stores the result, and returns it. Thread-safe: concurrent requests for the same key
// will block until the first one completes, then return the same value.
Product product = productCache.get(42L, id -> productRepository.findById(id)
    .orElseThrow(() -> new ProductNotFoundException(id)));

// put() — explicit write
productCache.put(42L, updatedProduct);

// getIfPresent() — returns null on miss (no loading)
Product cached = productCache.getIfPresent(42L);

// invalidate() — remove a specific entry (call after update/delete)
productCache.invalidate(42L);
productCache.invalidateAll();       // Clear the entire cache

// getAll() — batch load multiple keys efficiently (cache fills gaps in a single call)
Map<Long, Product> products = productCache.getAll(
    Set.of(1L, 2L, 3L),
    ids -> productRepository.findAllById(ids).stream()
        .collect(Collectors.toMap(Product::getId, Function.identity()))
);

// Statistics — crucial for understanding cache effectiveness
CacheStats stats = productCache.stats();
System.out.printf("Hit rate:     %.1f%%%n", stats.hitRate() * 100);    // e.g., 87.3%
System.out.printf("Miss count:   %d%n",     stats.missCount());
System.out.printf("Eviction cnt: %d%n",     stats.evictionCount());
System.out.printf("Avg load ns:  %.0f%n",   stats.averageLoadPenalty()); // Load time in nanoseconds

// ── LoadingCache — cache with an automatic loader ─────────────────────────────
// Preferred over manual cache when there is always one canonical way to load a value
LoadingCache<Long, User> userCache = Caffeine.newBuilder()
    .maximumSize(5_000)
    .expireAfterWrite(15, TimeUnit.MINUTES)
    .refreshAfterWrite(10, TimeUnit.MINUTES) // Async refresh: reload in background BEFORE expiry
    // When a key has been written for more than 10 minutes, the NEXT access triggers
    // a background reload. The stale value is still served immediately (no blocking).
    // This eliminates the "thundering herd" problem on cache expiry.
    .build(userId -> userRepository.findById(userId)
        .orElseThrow(() -> new UserNotFoundException(userId)));

// Usage is identical — get() loads on miss, refreshAfterWrite reloads proactively
User user = userCache.get(userId);

// ── AsyncLoadingCache — loading happens asynchronously ───────────────────────
AsyncLoadingCache<Long, User> asyncCache = Caffeine.newBuilder()
    .maximumSize(5_000)
    .expireAfterWrite(15, TimeUnit.MINUTES)
    .buildAsync(userId -> userRepository.findByIdAsync(userId));  // Returns CompletableFuture

CompletableFuture<User> futureUser = asyncCache.get(userId);
// Good for reactive applications or when you want non-blocking cache lookups
```

#### Caffeine Size-Based vs. Time-Based Eviction

Understanding eviction is critical to using any cache correctly. Caffeine supports two complementary eviction strategies that can be combined.

Size-based eviction (`maximumSize`) keeps the cache from growing unboundedly in memory. When the entry count exceeds the maximum, Caffeine uses the W-TinyLFU algorithm to select the entry least likely to be needed again. This is always a good idea — a cache without a size limit is a memory leak waiting to happen.

Time-based eviction removes entries based on age. `expireAfterWrite` evicts entries a fixed time after they were inserted or last updated, regardless of access — good for data that goes stale on a predictable schedule (like a price list that refreshes hourly). `expireAfterAccess` evicts entries that haven't been accessed recently — good for user sessions or infrequently-accessed records that should be kept in cache as long as they're actively being used. Use `refreshAfterWrite` when you want to update the cached value proactively (before it expires) to avoid the latency spike that occurs when an expired entry must be synchronously reloaded.

---

## 🗄️ Ehcache — Local and Distributed Caching

Ehcache is the longest-standing caching library in the Java ecosystem, maintained by Terracotta (now part of Software AG). It supports both pure in-process caching and distributed caching (via Terracotta Cluster). It integrates with JPA/Hibernate as the standard second-level cache provider, which is one of its most important use cases.

### 21.4.2 Ehcache — Programmatic Configuration

```java
import org.ehcache.*;
import org.ehcache.config.*;
import org.ehcache.config.builders.*;
import org.ehcache.config.units.*;

// ── Building a CacheManager (the top-level container for all caches) ──────────
CacheManager cacheManager = CacheManagerBuilder.newCacheManagerBuilder()
    .withCache("products",                               // Named cache
        CacheConfigurationBuilder.newCacheConfigurationBuilder(
            Long.class, Product.class,                   // Key and value types
            ResourcePoolsBuilder.newResourcePoolsBuilder()
                .heap(1_000, EntryUnit.ENTRIES)          // 1,000 entries in JVM heap
                .offheap(100, MemoryUnit.MB)             // 100MB in off-heap memory (no GC pressure)
                .disk(10, MemoryUnit.GB, true)           // 10GB on disk (persistent=true survives restart)
        )
        .withExpiry(ExpiryPolicyBuilder.timeToLiveExpiration(Duration.ofMinutes(10)))
        .withExpiry(ExpiryPolicyBuilder.timeToIdleExpiration(Duration.ofMinutes(5)))
    )
    .build(true);  // true = initialise immediately

// ── Using the cache ────────────────────────────────────────────────────────────
Cache<Long, Product> productCache = cacheManager.getCache("products", Long.class, Product.class);

productCache.put(42L, product);
Product cached = productCache.get(42L);        // null if not present
productCache.remove(42L);
productCache.clear();

// ── Ehcache as Hibernate Second-Level Cache ────────────────────────────────────
// This is Ehcache's most important Spring Boot use case.
// Add spring-boot-starter-cache, ehcache3, and hibernate-jcache to pom.xml.
// Configure in application.yml:
//
// spring.jpa.properties.hibernate.cache.use_second_level_cache: true
// spring.jpa.properties.hibernate.cache.region.factory_class: jcache
// spring.cache.jcache.config: classpath:ehcache.xml

// Then annotate your entities:
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE) // Cache this entity in Hibernate L2 cache
public class Product {
    @Id private Long id;
    private String name;
    private BigDecimal price;

    @OneToMany
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE) // Also cache the collection
    private List<Review> reviews;
}

// With L2 cache, a second findById() call for the same ID hits the cache,
// not the database — even across different Hibernate Sessions.
Product p1 = productRepository.findById(42L).get(); // Hits database, stores in L2 cache
Product p2 = productRepository.findById(42L).get(); // Served from L2 cache — no SQL
```

---

## 🔴 Redis — The Distributed Cache Standard

Redis (Remote Dictionary Server) is the dominant distributed caching solution in the modern Java ecosystem. It is an in-memory data structure store that supports strings, lists, sets, sorted sets, hashes, bitmaps, and more — accessible over a network. In a Spring Boot application, Redis is used for distributed session storage, distributed caching, rate limiting, pub/sub messaging, and distributed locks.

There are two Java client libraries for Redis: **Lettuce** (reactive, non-blocking, the Spring Boot default) and **Jedis** (blocking, thread-per-connection, simpler API). Spring Data Redis works with either.

### 21.4.3 Redis with Spring Data Redis

```java
// pom.xml: spring-boot-starter-data-redis (includes Lettuce by default)
// application.yml:
// spring.data.redis.host: localhost
// spring.data.redis.port: 6379
// spring.data.redis.password: ${REDIS_PASSWORD}

// ── RedisTemplate — the low-level Redis operations API ────────────────────────
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        // Use String for keys (human-readable, easier to debug in Redis CLI)
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        // Use Jackson for value serialization (JSON — readable and cross-language)
        template.setValueSerializer(new Jackson2JsonRedisSerializer<>(Object.class));
        template.setHashValueSerializer(new Jackson2JsonRedisSerializer<>(Object.class));
        return template;
    }
}

@Service
@RequiredArgsConstructor
public class ProductCacheService {
    private final RedisTemplate<String, Object> redisTemplate;
    private static final Duration TTL = Duration.ofMinutes(30);

    // Store a product — key is "product:{id}", value is JSON-serialized Product
    public void cacheProduct(Product product) {
        String key = "product:" + product.getId();
        redisTemplate.opsForValue().set(key, product, TTL);
    }

    // Retrieve a product
    public Optional<Product> getCachedProduct(Long id) {
        String key = "product:" + id;
        Product cached = (Product) redisTemplate.opsForValue().get(key);
        return Optional.ofNullable(cached);
    }

    // Delete a product from cache
    public void evictProduct(Long id) {
        redisTemplate.delete("product:" + id);
    }

    // ── Redis Hash — store an entity's fields as a hash (efficient updates) ───
    public void cacheUserSession(String sessionId, Map<String, String> sessionData) {
        String key = "session:" + sessionId;
        redisTemplate.opsForHash().putAll(key, sessionData);
        redisTemplate.expire(key, Duration.ofHours(24));
    }

    public Map<Object, Object> getSession(String sessionId) {
        return redisTemplate.opsForHash().entries("session:" + sessionId);
    }

    // ── Redis List — a queue for task processing ───────────────────────────────
    public void enqueueEmailTask(EmailTask task) {
        redisTemplate.opsForList().leftPush("email:queue", task);
    }

    public EmailTask dequeueEmailTask() {
        return (EmailTask) redisTemplate.opsForList().rightPop("email:queue");
    }

    // ── Redis Sorted Set — leaderboard with scores ────────────────────────────
    public void updateScore(String playerId, double score) {
        redisTemplate.opsForZSet().add("leaderboard", playerId, score);
    }

    public Set<Object> getTopPlayers(int count) {
        // Highest scores first: reverseRange returns from highest to lowest
        return redisTemplate.opsForZSet().reverseRange("leaderboard", 0, count - 1);
    }

    // ── Atomic increment — distributed counter ────────────────────────────────
    public long incrementPageView(String pageId) {
        return redisTemplate.opsForValue().increment("pageviews:" + pageId);
    }

    // ── Distributed lock — prevent concurrent access across instances ──────────
    public boolean acquireLock(String lockKey, String lockValue, Duration expiry) {
        // SET key value NX PX milliseconds — atomic set-if-not-exists
        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, lockValue, expiry);
        return Boolean.TRUE.equals(acquired);
    }

    public void releaseLock(String lockKey, String expectedValue) {
        // Compare-and-delete: only release if we own the lock
        // Use a Lua script for atomicity
        String luaScript = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;
        redisTemplate.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            List.of(lockKey),
            expectedValue
        );
    }
}
```

### 21.4.4 Lettuce vs. Jedis

**Lettuce** is the default Redis client in Spring Boot. It is built on Netty and uses a single non-blocking connection that is shared across all threads — a natural fit for reactive and virtual thread applications. It supports Redis Sentinel, Redis Cluster, and Redis Pub/Sub natively.

**Jedis** uses a blocking, synchronous model with a connection pool — one connection per thread. It is simpler to reason about and debug, and is the better choice for applications where reactive programming is not a goal. To use Jedis instead of Lettuce in Spring Boot, exclude the Lettuce starter and add the Jedis starter in `pom.xml`.

```java
// Lettuce (default) — connection is shared, non-blocking
// Use RedisTemplate or ReactiveRedisTemplate (for reactive apps)

// Jedis — straightforward blocking API, useful for scripts or batch jobs
@Configuration
public class JedisConfig {
    @Bean
    public JedisConnectionFactory jedisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName("localhost");
        config.setPort(6379);
        JedisClientConfiguration jedisConfig = JedisClientConfiguration.builder()
            .usePooling()
            .poolConfig(new JedisPoolConfig())
            .build();
        return new JedisConnectionFactory(config, jedisConfig);
    }
}
```

---

## 🌐 Hazelcast — Distributed Caching and Computing

Hazelcast is an in-memory data grid that extends caching into distributed computing. Unlike Redis (a separate process), Hazelcast can run **embedded** inside your Java application, making every instance a node of the cluster — no separate cache server required. It also supports running as a standalone cluster with client-server topology.

Hazelcast provides distributed versions of Java's standard collections: `IMap` (distributed `ConcurrentHashMap`), `IList`, `IQueue`, `ISet`, and `MultiMap`. It also supports distributed locks, event listeners, distributed queries, and entry processors (running code on the node where data lives, instead of transferring data to the client).

### 21.4.5 Hazelcast — Embedded Mode

```java
import com.hazelcast.core.*;
import com.hazelcast.config.*;
import com.hazelcast.map.*;

@Configuration
public class HazelcastConfig {

    @Bean
    public HazelcastInstance hazelcastInstance() {
        Config config = new Config();
        config.setClusterName("my-app-cluster");

        // Configure the distributed map with eviction and TTL
        MapConfig mapConfig = new MapConfig("products")
            .setMaxSizeConfig(new MaxSizeConfig(10_000, MaxSizePolicy.PER_NODE))
            .setEvictionConfig(new EvictionConfig()
                .setEvictionPolicy(EvictionPolicy.LFU)
                .setSize(10_000))
            .setTimeToLiveSeconds(600)    // 10-minute TTL
            .setMaxIdleSeconds(300);      // 5-minute idle timeout

        config.addMapConfig(mapConfig);

        // Network discovery — members find each other automatically on the same network
        NetworkConfig networkConfig = config.getNetworkConfig();
        JoinConfig joinConfig = networkConfig.getJoin();
        joinConfig.getMulticastConfig().setEnabled(true);  // Auto-discovery via multicast
        // For Kubernetes: use Hazelcast Kubernetes Discovery
        // For AWS: use Hazelcast AWS Discovery

        return Hazelcast.newHazelcastInstance(config);
    }
}

@Service
@RequiredArgsConstructor
public class ProductDistributedService {
    private final HazelcastInstance hazelcast;

    // ── IMap — distributed, consistent, partitioned map ──────────────────────
    private IMap<Long, Product> getProductMap() {
        return hazelcast.getMap("products");
    }

    public void cacheProduct(Product product) {
        getProductMap().put(product.getId(), product);
    }

    // putIfAbsent — atomic: only inserts if key is not already present
    public boolean cacheIfAbsent(Long id, Product product) {
        return getProductMap().putIfAbsent(id, product) == null;
    }

    // ── Entry Processor — execute logic on the node that owns the data ────────
    // Avoids network round-trip for read-modify-write operations
    public void incrementViewCount(Long productId) {
        getProductMap().executeOnKey(productId, entry -> {
            Product p = entry.getValue();
            if (p != null) {
                p.setViewCount(p.getViewCount() + 1);
                entry.setValue(p);  // Update in-place on the owning node
            }
            return null;
        });
    }

    // ── Distributed Query — query across all cluster nodes ────────────────────
    public Collection<Product> findProductsByCategory(String category) {
        return getProductMap().values(
            Predicates.equal("category", category)
        );
    }

    // ── Distributed Lock — cluster-wide mutual exclusion ──────────────────────
    public void updateInventory(Long productId, int delta) {
        FencedLock lock = hazelcast.getCPSubsystem().getLock("inventory-lock:" + productId);
        lock.lock();
        try {
            Product p = getProductMap().get(productId);
            p.setStock(p.getStock() + delta);
            getProductMap().put(productId, p);
        } finally {
            lock.unlock();
        }
    }
}
```

---

## 🌿 Spring Cache Abstraction

The Spring Cache abstraction (`@Cacheable`, `@CacheEvict`, `@CachePut`) sits on top of any underlying cache implementation — Caffeine, Ehcache, Redis, Hazelcast, or even a simple `ConcurrentHashMap`. You configure which provider to use in `application.yml`, and your service code uses only Spring annotations, which means you can swap cache implementations without changing business logic.

### 21.4.6 Spring Cache Annotations in Depth

```java
// application.yml — choose the cache implementation
// spring.cache.type: caffeine
// spring.cache.caffeine.spec: maximumSize=10000,expireAfterWrite=10m
// (or spring.cache.type: redis, hazelcast, ehcache, etc.)

@Service
@CacheConfig(cacheNames = "products") // Default cache name for all methods in this class
public class ProductService {

    // ── @Cacheable — cache the return value ───────────────────────────────────
    // On the first call, the method body executes and the return value is cached.
    // On subsequent calls with the same key, the cached value is returned directly
    // WITHOUT executing the method body.
    @Cacheable(key = "#id")
    public ProductDto getProduct(Long id) {
        // This code runs ONLY on a cache miss
        log.debug("Loading product {} from database", id);
        return productRepository.findById(id)
            .map(productMapper::toDto)
            .orElseThrow(() -> new ProductNotFoundException(id));
    }

    // Conditional caching — only cache if the condition is met
    @Cacheable(key = "#id", condition = "#id > 0", unless = "#result.archived")
    public ProductDto getProductConditional(Long id) {
        return productRepository.findById(id).map(productMapper::toDto).orElseThrow();
        // condition: applied before method call — don't cache if id <= 0
        // unless: applied after method call — don't cache if the result is archived
    }

    // ── @CachePut — always execute the method AND update the cache ────────────
    // Unlike @Cacheable which skips the method on hit, @CachePut ALWAYS executes
    // the method and stores the result. Used to keep the cache fresh after writes.
    @CachePut(key = "#result.id")  // Use the returned object's ID as the cache key
    public ProductDto updateProduct(Long id, UpdateProductRequest request) {
        Product product = productRepository.findById(id).orElseThrow();
        // ... apply updates ...
        Product saved = productRepository.save(product);
        return productMapper.toDto(saved);
        // The returned DTO is automatically stored in the cache, replacing any previous entry
    }

    // ── @CacheEvict — remove entries from the cache ───────────────────────────
    // Called after a write operation to ensure the cache doesn't serve stale data
    @CacheEvict(key = "#id")
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
        // Cache entry for this ID is removed after the method runs
    }

    // allEntries=true: clear the entire cache (use carefully — can cause cache stampede)
    @CacheEvict(allEntries = true)
    public void clearAllProductCache() {
        log.info("Product cache cleared");
    }

    // beforeInvocation=true: evict BEFORE the method runs (ensures eviction even on exception)
    @CacheEvict(key = "#id", beforeInvocation = true)
    public void deleteProductSafely(Long id) {
        productRepository.deleteById(id);
    }

    // ── @Caching — combine multiple cache operations on one method ─────────────
    @Caching(
        evict = {
            @CacheEvict(cacheNames = "products",       key = "#id"),
            @CacheEvict(cacheNames = "product-search", allEntries = true) // Invalidate search cache too
        }
    )
    public void deleteProductAndClearSearch(Long id) {
        productRepository.deleteById(id);
    }
}
```

> **Critical caveat:** Spring's cache annotations work through Spring AOP, which means they only apply when a method is called through the Spring proxy — i.e., from outside the bean. If `methodA()` in the same class calls `@Cacheable methodB()`, the cache is bypassed because the call goes through `this` (the real object) rather than the proxy. This is one of the most common Spring caching mistakes. The solution is to self-inject the service (using `@Autowired ApplicationContext` and getting the bean) or extract the cached method to a separate service class.

---

## 💡 Phase 21.4 — Key Takeaways

Caching done well requires understanding two things beyond the API: when to use a local cache vs. a distributed one, and how to handle cache invalidation correctly. Caffeine is the best in-process cache available in Java — use it when you have a single instance, when eventual consistency is acceptable, or when you need extremely low latency reads. Redis is the right choice when you have multiple application instances and need consistent, shared cache state. Ehcache shines specifically as Hibernate's second-level cache. Hazelcast suits systems that need distributed computing beyond just caching — distributed locks, entry processors, and distributed queries. The Spring Cache abstraction ties it together, letting you write annotation-driven caching code once and swap the underlying provider as your deployment topology evolves. Always monitor your cache hit rates — a hit rate below 70–80% usually means either the cache is too small, the TTL is too short, or the cached data isn't being accessed repeatedly enough to justify caching it.
