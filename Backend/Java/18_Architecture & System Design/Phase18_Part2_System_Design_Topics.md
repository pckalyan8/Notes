# Phase 18 — Part 2: System Design Topics

> System design is the discipline of making architectural choices about how to build systems that are fast, reliable, and scalable. It is less about writing code and more about understanding the trade-offs between different approaches to storing, moving, and processing data. Every concept in this section is a direct answer to a specific scaling challenge that real systems face.

---

## 1. Load Balancing

A **load balancer** is a component that distributes incoming requests across multiple servers (a "pool" or "cluster") to ensure that no single server becomes a bottleneck. It sits between clients and your servers, accepting all incoming traffic and intelligently forwarding it.

**Why you need it:** A single server can only handle so many requests per second. When that limit is reached, you have two options: make the server bigger (vertical scaling) or add more servers (horizontal scaling). Load balancing enables horizontal scaling by making multiple servers appear as one to the outside world.

```
                    Clients (millions of requests)
                           │
                           ▼
                  ┌────────────────┐
                  │  Load Balancer  │
                  └────────┬───────┘
                           │
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │  Server 1   │ │  Server 2   │ │  Server 3   │
    │  (active)   │ │  (active)   │ │  (active)   │
    └─────────────┘ └─────────────┘ └─────────────┘
```

### Load Balancing Algorithms

**Round Robin:** Each request is sent to the next server in a circular sequence. Simple and even distribution — but blind to server health or current load. If Server 2 happens to be processing a heavy batch job, Round Robin will still keep sending it new requests at the same rate.

**Weighted Round Robin:** Like Round Robin, but servers are assigned weights proportional to their capacity. A server with 16 cores gets weight 2, a server with 8 cores gets weight 1 — the powerful server handles twice as many requests. Useful when your server pool has heterogeneous hardware.

**Least Connections:** Send the next request to the server with the fewest active connections. Better than Round Robin for workloads where requests have variable processing time — a server that's idle gets new work while busy servers finish their existing work. This is the most common algorithm for real-world use.

**IP Hash:** Hash the client's IP address to consistently route the same client to the same server. This provides "sticky sessions" — important when your servers hold session state. But it's fragile: if a server goes down, all its clients are disrupted, and the hashing may not distribute load evenly.

**Random:** Select a server at random. Surprisingly effective at scale due to the law of large numbers. "Power of Two Choices" is a refined variant: pick 2 servers at random, send to the less-loaded one. This provides most of the benefit of Least Connections with far less coordination overhead.

```java
// Implementing a simple load balancer in Java — illustrative, not production
// Production load balancers are hardware appliances or software like Nginx, HAProxy, or AWS ALB

public class LoadBalancer {
    private final List<Server> servers;
    private final AtomicInteger roundRobinIndex = new AtomicInteger(0);

    // ─────────────────────────────────────────────────────
    // Round Robin — O(1) per selection, no server knowledge needed
    public Server nextRoundRobin() {
        // AtomicInteger ensures thread-safety without a lock
        int index = roundRobinIndex.getAndIncrement() % servers.size();
        return servers.get(index);
    }

    // ─────────────────────────────────────────────────────
    // Least Connections — O(n) per selection, always routes to least busy
    public Server leastConnections() {
        return servers.stream()
            .min(Comparator.comparingInt(Server::getActiveConnections))
            .orElseThrow(() -> new RuntimeException("No servers available"));
    }

    // ─────────────────────────────────────────────────────
    // Weighted Round Robin — proportional distribution based on server capacity
    public Server weightedRoundRobin() {
        // Build a weighted pool: a server with weight 3 appears 3 times in the list
        List<Server> weightedPool = servers.stream()
            .flatMap(s -> Collections.nCopies(s.getWeight(), s).stream())
            .collect(Collectors.toList());
        int index = roundRobinIndex.getAndIncrement() % weightedPool.size();
        return weightedPool.get(index);
    }

    // ─────────────────────────────────────────────────────
    // IP Hash — same client always goes to same server (sticky sessions)
    public Server ipHash(String clientIp) {
        int hash = Math.abs(clientIp.hashCode()); // Hash the client IP
        return servers.get(hash % servers.size());
    }
}

// Health checks — the load balancer must remove unhealthy servers automatically
// In Spring Boot + Netflix Eureka / AWS ALB, health checks are built-in
@Component
public class HealthCheckScheduler {
    private final LoadBalancer loadBalancer;

    @Scheduled(fixedDelay = 5000) // Check every 5 seconds
    public void runHealthChecks() {
        for (Server server : loadBalancer.getAllServers()) {
            boolean healthy = pingServer(server); // HTTP GET /health
            if (!healthy) {
                loadBalancer.markUnhealthy(server); // Remove from rotation
                alertingService.notify("Server " + server.getId() + " is down");
            } else if (server.isMarkedUnhealthy()) {
                loadBalancer.markHealthy(server); // Re-add when it recovers
            }
        }
    }
}
```

**Layer 4 vs Layer 7 load balancing:** Layer 4 (transport layer) load balancers work at the TCP level — they route based on IP and port without looking at the content. They're fast because they do minimal processing. Layer 7 (application layer) load balancers understand HTTP — they can route based on URL paths, headers, cookies, or request body content. AWS Application Load Balancer (ALB) is Layer 7. This matters: you can route `/api/orders/*` to one server cluster and `/api/products/*` to another.

---

## 2. Horizontal vs. Vertical Scaling

**Vertical scaling (scaling up):** Give your existing server more resources — a bigger CPU, more RAM, faster disk. It's the simplest form of scaling because your application doesn't need to change at all. But it has a hard ceiling (you can't add infinite RAM to one machine), and it typically requires downtime. A single 64-core server is also a single point of failure.

**Horizontal scaling (scaling out):** Add more servers to your pool. This requires your application to be **stateless** — since requests may land on any server, no server should hold in-memory state that another server doesn't have. Sessions must be stored externally (Redis, database). Horizontal scaling theoretically has no ceiling and provides natural redundancy.

```java
// Making a Spring Boot application stateless — prerequisite for horizontal scaling
// Problem: HttpSession stores data in-memory on one server — breaks with load balancing
// Solution: Store session data in Redis, which all servers can access

// Before (stateful — won't work with horizontal scaling):
@GetMapping("/dashboard")
public String dashboard(HttpSession session) {
    User user = (User) session.getAttribute("currentUser"); // Data lives on ONE server
    // If Load Balancer routes next request to a different server, "currentUser" is GONE
    return userDashboard(user);
}

// After (stateless with Spring Session + Redis):
// Add spring-session-data-redis dependency and configure Redis
// Spring automatically stores sessions in Redis — any server can find any session
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800) // 30-minute sessions
public class SessionConfig {
    // Spring Session transparently intercepts HttpSession calls and reads/writes to Redis
    // Now any server in the pool can serve any user's request — true horizontal scaling
}

// Your controller code is unchanged — Spring Session handles the Redis storage transparently
@GetMapping("/dashboard")
public String dashboard(HttpSession session) {
    User user = (User) session.getAttribute("currentUser");
    // This now reads from Redis — works regardless of which server handles the request
    return userDashboard(user);
}
```

---

## 3. Caching

Caching stores the result of an expensive operation (database query, computation, external API call) in a fast-access store so that subsequent requests for the same data can be served quickly without repeating the expensive work.

**Why caching is so powerful:** Reading from Redis (in-memory) typically takes 0.1-1ms. Reading from a database (disk + network) typically takes 5-50ms. Caching can make your system appear 10-100x faster for cached data, and dramatically reduces database load.

### Caching Strategies

**Cache-Aside (Lazy Loading):** The application checks the cache first. On a cache miss, it fetches from the database and populates the cache. This is the most common and versatile strategy.

```java
@Service
public class ProductService {
    private final ProductRepository repository;
    private final RedisTemplate<String, Product> redisTemplate;

    // Cache-Aside pattern: check cache, fall back to database
    public Product getProduct(Long productId) {
        String cacheKey = "product:" + productId;
        ValueOperations<String, Product> ops = redisTemplate.opsForValue();

        // Step 1: Try the cache first — O(1) and very fast
        Product cached = ops.get(cacheKey);
        if (cached != null) {
            return cached; // Cache HIT — return immediately without touching the database
        }

        // Step 2: Cache MISS — fetch from the database (slow path)
        Product product = repository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));

        // Step 3: Populate the cache for future requests
        // TTL of 1 hour — product data changes infrequently
        ops.set(cacheKey, product, Duration.ofHours(1));

        return product;
    }

    // Invalidate cache when data changes
    public Product updateProduct(Long productId, ProductUpdate update) {
        Product product = repository.save(update.applyTo(getProduct(productId)));
        // Remove the stale cache entry — next read will repopulate with fresh data
        redisTemplate.delete("product:" + productId);
        return product;
    }
}
```

**Write-Through:** Every write goes to both the database and the cache simultaneously. Cache is never stale because writes always keep it updated. But every write is slightly slower (two writes instead of one), and data that's rarely read still occupies cache space.

```java
// Write-Through: update both cache and database on every write
public Product updateProductWriteThrough(Long productId, ProductUpdate update) {
    Product product = repository.save(update.applyTo(getProduct(productId)));
    // Write to cache immediately after writing to database
    redisTemplate.opsForValue().set("product:" + productId, product, Duration.ofHours(1));
    return product;
    // Advantage: cache always current; Disadvantage: extra write latency for every update
}
```

**Write-Back (Write-Behind):** Writes go only to the cache first, and are asynchronously flushed to the database later. This gives very fast write performance but risks data loss if the cache crashes before flushing. Only appropriate when some data loss is acceptable (analytics counters, view counts).

**Read-Through:** The cache sits in front of the database and the application only ever talks to the cache. On a miss, the cache itself fetches from the database. This is what Amazon ElastiCache with cache-aware drivers does. It abstracts the cache-miss logic away from application code.

### Cache Eviction Policies

When the cache is full, it must decide which entries to evict to make room for new ones:

- **LRU (Least Recently Used):** Evict the entry that was accessed least recently. This is the most common policy because it reflects the principle of temporal locality — recently accessed data is likely to be accessed again soon. Redis uses an approximation of LRU.
- **LFU (Least Frequently Used):** Evict the entry accessed least often overall. Better for workloads with a stable "hot" dataset — rarely accessed items age out even if they were accessed recently.
- **TTL (Time To Live):** Every entry expires after a fixed time, regardless of access patterns. Simple and predictable. Use for data that goes stale after a known time window.
- **FIFO (First In, First Out):** Evict the oldest entry. Simple but not adaptive to access patterns.

```java
// Caffeine: high-performance in-process cache for Java (used by Spring Boot internally)
// Best for caching within a single JVM — no serialization, no network
Cache<Long, Product> productCache = Caffeine.newBuilder()
    .maximumSize(10_000)           // Evict when more than 10,000 entries (LRU-based)
    .expireAfterWrite(1, TimeUnit.HOURS)  // TTL: expire 1 hour after write
    .expireAfterAccess(30, TimeUnit.MINUTES) // Also expire if not accessed for 30 minutes
    .recordStats()                  // Enable hit rate monitoring
    .build();

// Spring Cache abstraction — declarative caching with @Cacheable annotation
// Works with any cache implementation (Caffeine, Redis, Ehcache) via pluggable providers
@Service
public class UserService {

    @Cacheable(value = "users", key = "#userId")
    // Spring will check the "users" cache for this userId before executing the method.
    // On cache hit, method body is SKIPPED entirely.
    public User getUserById(Long userId) {
        return userRepository.findById(userId).orElseThrow(); // Only called on cache miss
    }

    @CacheEvict(value = "users", key = "#userId")
    // Remove this user's entry from the cache when their profile is updated
    public User updateUser(Long userId, UserUpdate update) {
        return userRepository.save(update.applyTo(getUserById(userId)));
    }

    @CachePut(value = "users", key = "#result.id")
    // Always execute the method AND update the cache with the result (write-through)
    public User createUser(CreateUserRequest request) {
        return userRepository.save(User.from(request));
    }
}
```

### Cache Invalidation

Cache invalidation is famously difficult — one of the "two hard things in computer science." The core challenge: when does a cache entry become stale? Three strategies:

TTL-based invalidation: entries expire automatically after a time window. Simple, self-healing, but introduces a window of inconsistency equal to the TTL. Good default for most data.

Event-based invalidation: when data changes, explicitly delete or update the corresponding cache entry. Zero staleness window, but requires tight coupling between write paths and cache invalidation logic. A missed invalidation causes stale reads until TTL expires.

Write-through: keep the cache synchronized on every write. No staleness, but more complex write paths and higher write latency.

---

## 4. Database Sharding

**Sharding** (horizontal partitioning) splits a single database into multiple smaller databases called **shards**, each holding a subset of the data. Unlike replication (where every copy has all data), sharding divides the data so no single shard has all of it.

**Why sharding:** A single PostgreSQL server can handle a few thousand writes per second and store a few terabytes efficiently. At billions of records and millions of writes per second, even the best single database server is not enough. Sharding distributes both storage and write load across many servers.

### Sharding Strategies

**Range-Based Sharding:** Assign records to shards based on a range of the shard key. Users with IDs 1-1,000,000 go to Shard 1, 1,000,001-2,000,000 go to Shard 2, and so on. Simple to understand, enables range queries, but risks **hot spots** — users added recently all land on the last shard.

**Hash-Based Sharding:** Apply a hash function to the shard key and use the result to choose the shard. This distributes data evenly and eliminates hot spots. But range queries become impossible (you can't ask "give me all users created between date A and date B" efficiently because those users are scattered across shards).

**Directory-Based Sharding:** A lookup table (the "shard directory") maps each record to its shard. Flexible — you can move records between shards by updating the directory. But the directory itself can become a bottleneck and a single point of failure.

```java
// Hash-based sharding in application code
// The application decides which database shard to use based on the shard key
public class ShardedUserRepository {
    private final List<DataSource> shards; // List of database connections, one per shard
    private final int SHARD_COUNT;

    public ShardedUserRepository(List<DataSource> shards) {
        this.shards = shards;
        this.SHARD_COUNT = shards.size();
    }

    // Determine which shard holds data for a given user
    private DataSource getShardForUser(Long userId) {
        // Consistent hash: user 12345 always goes to the same shard
        int shardIndex = (int) (Math.abs(userId % SHARD_COUNT));
        return shards.get(shardIndex);
    }

    // Save user to the correct shard
    public void save(User user) {
        DataSource shard = getShardForUser(user.getId());
        JdbcTemplate jdbc = new JdbcTemplate(shard);
        jdbc.update("INSERT INTO users (id, name, email) VALUES (?, ?, ?)",
            user.getId(), user.getName(), user.getEmail());
    }

    // Read from the correct shard — fast because we know exactly where the data is
    public User findById(Long userId) {
        DataSource shard = getShardForUser(userId);
        JdbcTemplate jdbc = new JdbcTemplate(shard);
        return jdbc.queryForObject(
            "SELECT * FROM users WHERE id = ?",
            new UserRowMapper(),
            userId);
    }

    // Cross-shard query — must query ALL shards and aggregate results
    // This is expensive and should be avoided; it's why choice of shard key matters enormously
    public List<User> findByCity(String city) {
        return shards.parallelStream()
            .flatMap(shard -> {
                JdbcTemplate jdbc = new JdbcTemplate(shard);
                return jdbc.query("SELECT * FROM users WHERE city = ?",
                    new UserRowMapper(), city).stream();
            })
            .collect(Collectors.toList());
    }
}
```

**The shard key is the most critical design decision in a sharded system.** A poor shard key leads to hot spots (one shard gets all the traffic), data skew (one shard stores 90% of the data), or expensive cross-shard queries. Good shard keys have high cardinality (many distinct values), are evenly distributed, and align with your most frequent query patterns.

---

## 5. Database Replication

**Replication** copies data from one database server (the **primary/leader**) to one or more other servers (**replicas/followers**). Unlike sharding, every replica holds a complete copy of all data.

**Primary use cases:** Read scaling (route reads to replicas to offload the primary), high availability (promote a replica if the primary fails), geographic distribution (put replicas near users in different regions).

```java
// Spring configuration for read/write splitting — primary handles writes, replicas handle reads
// This is one of the most impactful performance techniques available with zero schema changes
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    public DataSource primaryDataSource() {
        // Primary: handles all write operations (INSERT, UPDATE, DELETE, DDL)
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://primary-db:5432/appdb");
        ds.setMaximumPoolSize(10);
        return ds;
    }

    @Bean
    public DataSource replicaDataSource() {
        // Replica pool: handles read-only operations (SELECT)
        // In production, this would be a round-robin pool across multiple replicas
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://replica-db:5432/appdb");
        ds.setMaximumPoolSize(30); // More connections since reads dominate
        return ds;
    }

    // Routing DataSource: transparently routes to primary or replica based on transaction type
    @Bean
    public DataSource routingDataSource(
            @Qualifier("primaryDataSource") DataSource primary,
            @Qualifier("replicaDataSource") DataSource replica) {
        AbstractRoutingDataSource routing = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                // If the current transaction is read-only (@Transactional(readOnly=true)),
                // route to the replica; otherwise route to primary
                return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
                    ? "replica" : "primary";
            }
        };
        routing.setTargetDataSources(Map.of("primary", primary, "replica", replica));
        routing.setDefaultTargetDataSource(primary);
        return routing;
    }
}

@Service
public class ProductService {
    // Write operations use the primary — @Transactional defaults to readOnly=false
    @Transactional
    public Product createProduct(ProductRequest req) {
        return productRepository.save(Product.from(req)); // Goes to primary
    }

    // Read operations use the replica — readOnly=true routes to replica
    @Transactional(readOnly = true)
    public Page<Product> getProducts(Pageable pageable) {
        return productRepository.findAll(pageable); // Goes to replica
    }
}
```

**Replication lag is a real concern.** There is always a delay between when a write is committed on the primary and when it appears on the replica — typically milliseconds within a datacenter, but potentially seconds during high load or across regions. If a user creates an order and immediately reads it back, they might read from a replica that hasn't received the write yet and see nothing. This is the same eventual consistency problem discussed in Part 1. Solutions: read your own writes from the primary for a short window after a write, or use synchronous replication for critical data (at the cost of write latency).

---

## 6. Consistent Hashing

**Consistent hashing** solves a critical problem in distributed systems: when you add or remove a server from a cluster, how do you redistribute the data with minimal disruption?

**The problem with naive modulo hashing:** If you have 3 servers and use `server = hash(key) % 3`, then adding a 4th server changes the formula to `% 4`. Now almost every key maps to a different server — you'd need to move nearly all your data. This is catastrophic in a caching system or a distributed database.

**The consistent hashing solution:** Imagine a ring (a circle of hash values from 0 to 2^32). Each server is assigned a position on this ring by hashing its name or IP address. Each key is also hashed to a position on the ring, then assigned to the first server encountered clockwise from that position. When a server is added or removed, only the keys that were assigned to that server need to be redistributed — typically 1/N of all keys (where N is the number of servers).

```java
// Consistent Hashing implementation — used internally by Redis Cluster, Cassandra, DynamoDB
import java.util.SortedMap;
import java.util.TreeMap;

public class ConsistentHashRing<T> {
    // The "ring" — a sorted map from hash position to server
    private final SortedMap<Integer, T> ring = new TreeMap<>();
    private final HashFunction hashFunction;
    // Virtual nodes: each physical server is represented multiple times on the ring
    // This prevents hot spots and achieves more even distribution
    private final int virtualNodesPerServer;

    public ConsistentHashRing(HashFunction hashFunction, int virtualNodesPerServer) {
        this.hashFunction = hashFunction;
        this.virtualNodesPerServer = virtualNodesPerServer;
    }

    // Adding a server: place it (and its virtual nodes) at positions on the ring
    public void addServer(T server) {
        for (int i = 0; i < virtualNodesPerServer; i++) {
            // Each virtual node gets a unique hash: "server-address:1", "server-address:2", etc.
            int hash = hashFunction.hash(server.toString() + ":" + i);
            ring.put(hash, server);
        }
    }

    // Removing a server: remove all its positions from the ring
    // Only keys that mapped to this server need to be redistributed to the next server clockwise
    public void removeServer(T server) {
        for (int i = 0; i < virtualNodesPerServer; i++) {
            int hash = hashFunction.hash(server.toString() + ":" + i);
            ring.remove(hash);
        }
    }

    // Finding which server holds a given key: clockwise lookup from the key's hash position
    public T getServer(String key) {
        if (ring.isEmpty()) throw new RuntimeException("No servers in ring");
        int hash = hashFunction.hash(key);
        // Find the smallest position on the ring >= hash (first server clockwise)
        SortedMap<Integer, T> tailMap = ring.tailMap(hash);
        // If we're past all servers, wrap around to the first server on the ring
        int position = tailMap.isEmpty() ? ring.firstKey() : tailMap.firstKey();
        return ring.get(position);
    }
}

// Practical usage — consistent hashing for a distributed cache
ConsistentHashRing<String> cacheRing = new ConsistentHashRing<>(
    key -> Math.abs(key.hashCode()), // Hash function
    150 // 150 virtual nodes per physical server for even distribution
);

cacheRing.addServer("cache-server-1:6379");
cacheRing.addServer("cache-server-2:6379");
cacheRing.addServer("cache-server-3:6379");

// Route cache operations to the correct server
String key = "user:12345";
String server = cacheRing.getServer(key); // Always returns the same server for the same key
// → "cache-server-2:6379" (deterministic)

// Adding a 4th server: only ~25% of keys need to be remapped (1/4)
// Compare to naive % hashing: ~75% of keys would need remapping when going from 3 to 4 servers
cacheRing.addServer("cache-server-4:6379");
```

---

## 7. Rate Limiting

**Rate limiting** controls how many requests a client can make within a time window. It protects services from abuse, prevents resource exhaustion, and enforces fair usage in multi-tenant systems.

### Rate Limiting Algorithms

**Fixed Window Counter:** Count requests in fixed time windows (e.g., 100 requests per minute). At the start of each minute, the counter resets. Simple but has a "boundary burst" problem: a client can make 100 requests in the last second of minute 1 and 100 requests in the first second of minute 2 — 200 requests in 2 seconds while technically staying within the limit.

**Sliding Window Log:** Store the timestamp of every request in a log. For each new request, count how many timestamps are within the last time window. Precise but memory-intensive — you store a timestamp per request.

**Sliding Window Counter:** A practical middle ground. Divide the time window into smaller buckets. The current window's count is estimated using a weighted combination of the current and previous buckets. Uses O(1) memory per key.

**Token Bucket:** A bucket holds tokens. Tokens accumulate at a fixed rate (e.g., 10 tokens per second), up to a maximum capacity (e.g., 100 tokens). Each request consumes one token. When the bucket is empty, requests are rejected. This naturally handles bursts: if traffic is light for a minute, the bucket fills up and then allows a burst. This is the most common algorithm used in practice.

**Leaky Bucket:** Like Token Bucket but smooths output. Requests enter a queue; they're processed at a constant rate regardless of how fast they arrive. Prevents bursts — a flood of requests is queued and processed steadily. Good for protecting a slow downstream service.

```java
// Token Bucket Rate Limiter using Guava's RateLimiter
import com.google.common.util.concurrent.RateLimiter;

@Component
public class RateLimiterService {
    // Each API key gets its own rate limiter — 10 requests per second
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();

    // Returns true if the request is allowed, false if rate-limited
    public boolean isAllowed(String apiKey) {
        // computeIfAbsent: create a new RateLimiter if this apiKey hasn't been seen
        RateLimiter limiter = limiters.computeIfAbsent(apiKey,
            key -> RateLimiter.create(10.0)); // 10 permits per second
        return limiter.tryAcquire(); // Returns immediately — no waiting
    }
}

// Rate limiting middleware — Spring interceptor that runs before every controller
@Component
public class RateLimitInterceptor implements HandlerInterceptor {
    private final RateLimiterService rateLimiterService;

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) throws Exception {
        String apiKey = request.getHeader("X-API-Key");
        if (apiKey == null) {
            response.sendError(HttpStatus.UNAUTHORIZED.value(), "API key required");
            return false;
        }
        if (!rateLimiterService.isAllowed(apiKey)) {
            response.setHeader("X-RateLimit-Limit", "10");
            response.setHeader("Retry-After", "1"); // Retry after 1 second
            response.sendError(HttpStatus.TOO_MANY_REQUESTS.value(), "Rate limit exceeded");
            return false; // Block the request — do not call the controller
        }
        return true; // Allow the request to proceed
    }
}

// Redis-based distributed rate limiting — works across multiple server instances
// Token Bucket using Redis atomic operations (Lua script ensures atomicity)
@Service
public class DistributedRateLimiter {
    private final StringRedisTemplate redisTemplate;
    private static final int MAX_REQUESTS = 100;    // Max requests per window
    private static final int WINDOW_SECONDS = 60;   // 60-second window

    // Sliding window counter using Redis — works correctly with multiple app servers
    public boolean isAllowed(String clientId) {
        String key = "ratelimit:" + clientId;
        long now = System.currentTimeMillis();
        long windowStart = now - (WINDOW_SECONDS * 1000L);

        // Redis transaction: atomically count requests in the current window
        List<Object> results = redisTemplate.execute(new SessionCallback<List<Object>>() {
            @Override
            public List<Object> execute(RedisOperations operations) {
                operations.multi(); // Begin transaction
                // Remove timestamps older than the window
                operations.opsForZSet().removeRangeByScore(key, 0, windowStart);
                // Add current request's timestamp
                operations.opsForZSet().add(key, UUID.randomUUID().toString(), now);
                // Count requests in window
                operations.opsForZSet().count(key, windowStart, now);
                // Set expiry to clean up old keys
                operations.expire(key, WINDOW_SECONDS * 2, TimeUnit.SECONDS);
                return operations.exec(); // Execute all atomically
            }
        });

        Long count = (Long) results.get(2); // Result of the count operation
        return count != null && count <= MAX_REQUESTS;
    }
}
```

---

## 8. API Pagination

Pagination prevents a client from fetecting an entire table in one request (which would be slow for both the database and the client) and instead returns data in manageable pages.

**Offset-based pagination:** `SELECT * FROM orders LIMIT 20 OFFSET 100` fetches records 101-120. Simple to implement and supports random page access ("jump to page 5"), but has a critical problem: it performs poorly on large offsets because the database must scan and discard the first N rows before returning the page you want.

**Cursor-based pagination (keyset pagination):** Instead of an offset, you give the client a "cursor" — a pointer to the last item they saw. The next page query is `WHERE id > :lastSeenId LIMIT 20`. This is O(log n) with an index (no wasted scans), handles concurrent inserts correctly (no "phantom rows"), and is the correct approach for feeds, infinite scroll, and large datasets.

```java
// Offset-based pagination — simple but degrades at high offsets
@GetMapping("/products")
public Page<ProductDTO> getProductsOffset(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size) {

    Pageable pageable = PageRequest.of(page, size, Sort.by("id"));
    return productRepository.findAll(pageable) // Spring Data handles the SQL
        .map(ProductDTO::from);
    // Generated SQL: SELECT * FROM products ORDER BY id LIMIT 20 OFFSET (page * 20)
    // At page=1000: database scans 20,000 rows and discards 19,980 — very wasteful
}

// Cursor-based pagination — efficient at any scale
@GetMapping("/products/stream")
public CursorPage<ProductDTO> getProductsCursor(
        @RequestParam(required = false) Long cursor, // null means "start from beginning"
        @RequestParam(defaultValue = "20") int size) {

    List<Product> products;
    if (cursor == null) {
        // First page: fetch the newest products
        products = productRepository.findTopByOrderByIdDesc(size + 1); // Fetch one extra
    } else {
        // Subsequent pages: fetch products with ID less than the cursor
        products = productRepository.findByIdLessThanOrderByIdDesc(cursor, size + 1);
        // Generated SQL: SELECT * FROM products WHERE id < :cursor ORDER BY id DESC LIMIT 21
        // With an index on 'id', this is O(log n) regardless of how deep in the dataset we are
    }

    boolean hasMore = products.size() > size;
    List<ProductDTO> page = products.stream()
        .limit(size)         // Only return 'size' items (not the extra one)
        .map(ProductDTO::from)
        .collect(Collectors.toList());

    // The cursor for the next page is the ID of the last item returned
    Long nextCursor = hasMore ? page.get(page.size() - 1).getId() : null;

    return new CursorPage<>(page, nextCursor, hasMore);
}
```

---

## 9. Database Indexing Strategies

An **index** is an auxiliary data structure (typically a B-tree or hash table) that the database maintains to speed up reads at the cost of slower writes and extra storage. Without an index, a query like `WHERE email = 'alice@example.com'` requires a full table scan — reading every row. With an index on `email`, the database finds the matching row in O(log n) time.

```java
// Defining indexes in Spring Data JPA
@Entity
@Table(name = "orders",
    indexes = {
        // Single-column index: speeds up queries filtering by customer_id
        @Index(name = "idx_orders_customer_id", columnList = "customer_id"),
        // Composite index: optimal for queries that filter by BOTH status AND created_at
        // Order of columns matters! This index helps: WHERE status = 'PENDING' AND created_at > ?
        // But does NOT help: WHERE created_at > ? (only right-prefix queries don't use the index)
        @Index(name = "idx_orders_status_created", columnList = "status, created_at"),
        // Unique index: enforces uniqueness AND speeds up lookups
        @Index(name = "idx_orders_reference_unique", columnList = "reference_number", unique = true)
    })
public class Order {
    @Id
    private Long id;

    @Column(name = "customer_id", nullable = false)
    private Long customerId; // Index on this: O(log n) lookup for "orders by customer"

    @Column(name = "status", nullable = false)
    private String status;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    @Column(name = "reference_number", unique = true)
    private String referenceNumber;
}

// Query that benefits from composite index idx_orders_status_created:
// "Find all PENDING orders created in the last 7 days, sorted by creation time"
@Query("SELECT o FROM Order o WHERE o.status = :status AND o.createdAt > :since ORDER BY o.createdAt DESC")
List<Order> findByStatusAndCreatedAfter(@Param("status") String status, @Param("since") Instant since);
// With the composite index: database finds the PENDING orders with a range scan — very fast
// Without index: full table scan through potentially millions of orders — very slow
```

**Index types you should know:**

A **B-tree index** (the default in PostgreSQL, MySQL) supports equality queries (`WHERE id = 5`), range queries (`WHERE price BETWEEN 10 AND 100`), and `ORDER BY`. It's the right choice for most queries.

A **Hash index** supports only equality queries and is slightly faster for them. In PostgreSQL, hash indexes are rarely used because B-tree indexes are nearly as fast and support range queries.

A **Partial index** only indexes rows matching a condition. `CREATE INDEX ON orders (created_at) WHERE status = 'PENDING'` builds a small, fast index specifically for pending orders — much more efficient than a full-table index when you consistently query a small subset.

A **Covering index** includes all columns needed to answer a query, so the database can answer it entirely from the index without touching the main table ("index-only scan"). Extremely fast for read-heavy queries on large tables.

---

## 10. Important Points & Best Practices

**On caching:** "There are only two hard things in computer science: cache invalidation and naming things" (Phil Karlton). Take cache invalidation seriously. Every time you add a cache, document exactly when and how that cache entry will be invalidated. Stale cache entries that persist indefinitely cause bugs that are incredibly hard to diagnose in production.

**On sharding:** Sharding is a last resort, not a first design choice. Exhaust these options first: add indexes, add read replicas, upgrade hardware, denormalize data, use connection pooling. Sharding adds enormous operational complexity — cross-shard queries, cross-shard transactions, and rebalancing when shard counts change are all genuinely hard problems.

**On rate limiting:** Always return informative headers with rate limit responses. `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, and `Retry-After` headers tell clients exactly what happened and when they can try again — enabling well-behaved clients to adapt without hammering your service.

**On indexes:** Index the columns you query by, not the columns you store. Adding an index to every column "just in case" is harmful — each index slows down every INSERT, UPDATE, and DELETE because the database must update the index as well as the table. A table with 15 indexes can write 10x slower than the same table with 3 focused indexes. Run `EXPLAIN ANALYZE` in PostgreSQL on your slow queries to see exactly which indexes are and aren't being used.

**On pagination:** Never expose offset-based pagination to end users for large datasets. The moment a user can request "page 10,000," they can (accidentally or maliciously) bring your database to its knees. Cursor-based pagination is the production-correct approach for any dataset that grows over time.
