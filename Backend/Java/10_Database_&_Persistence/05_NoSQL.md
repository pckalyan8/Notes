# 📕 10.5 — NoSQL with Java

NoSQL databases challenge the assumption that all data belongs in rows and columns. Once you understand when and why to reach for them, they become powerful tools that complement — not replace — your relational database skills.

---

## 🧠 Why NoSQL?

Relational databases are excellent for structured, consistent, relational data. But the real world throws data at us in shapes that don't fit neatly into tables:

A user's social media profile might have zero posts, or ten thousand. A product in an e-commerce catalogue might have ten attributes or a hundred, depending on its category. An IoT sensor streams millions of timestamped readings per day. Storing and querying these patterns efficiently in a relational database requires complex schema gymnastics.

NoSQL databases trade some of the consistency and query flexibility of relational databases for **flexibility of data structure**, **horizontal scalability**, and **optimised performance for specific access patterns**.

There are four primary NoSQL database families, each solving a different problem:

**Document stores** (MongoDB) store JSON-like documents, allowing each record to have a different structure. Perfect for content management, user profiles, and product catalogues. **Key-value stores** (Redis) are essentially a dictionary — store and retrieve values by key with microsecond latency. Perfect for caching, sessions, and leaderboards. **Column-family stores** (Cassandra) distribute data across nodes optimised for heavy write loads and time-series queries. Perfect for IoT telemetry, analytics, and audit logs. **Search engines** (Elasticsearch) invert the index — instead of looking up a document by ID, you search full text and the engine tells you which documents contain it. Perfect for free-text search, logs, and metrics.

---

## 🍃 Part 1 — MongoDB

MongoDB stores data as **BSON documents** (Binary JSON) in **collections** (analogous to tables). The key difference from a relational database is that documents in the same collection can have completely different fields — there's no schema enforcement by default.

```
Relational Database     MongoDB
──────────────────────  ────────────────────────
Database                Database
Table                   Collection
Row                     Document (JSON/BSON)
Column                  Field
Primary Key             _id field (ObjectId)
JOIN query              $lookup aggregation stage
```

### Maven Dependencies

```xml
<!-- For a plain Java (non-Spring) app -->
<dependency>
    <groupId>org.mongodb</groupId>
    <artifactId>mongodb-driver-sync</artifactId>
    <version>5.0.0</version>
</dependency>

<!-- For Spring Boot — includes Spring Data MongoDB + driver -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

### 1.1 Raw MongoDB Java Driver

```java
import com.mongodb.client.*;
import com.mongodb.client.model.*;
import com.mongodb.client.result.*;
import org.bson.Document;
import org.bson.types.ObjectId;

public class MongoRawDemo {

    public static void main(String[] args) {

        // MongoClient is thread-safe and expensive to create — use as a singleton
        try (MongoClient client = MongoClients.create("mongodb://localhost:27017")) {

            // Get a database handle (created lazily — no I/O yet)
            MongoDatabase db = client.getDatabase("school_db");

            // Get a collection handle
            MongoCollection<Document> students = db.getCollection("students");

            // ── INSERT ──
            Document alice = new Document()
                .append("name", "Alice")
                .append("email", "alice@example.com")
                .append("score", 95.5)
                .append("status", "ENROLLED")
                .append("courses", List.of("Java", "Algorithms")) // Embedded array
                .append("address", new Document()                  // Embedded document
                    .append("city", "New York")
                    .append("zip", "10001"));

            InsertOneResult insertResult = students.insertOne(alice);
            // MongoDB auto-generates an ObjectId if _id is not provided
            ObjectId newId = insertResult.getInsertedId().asObjectId().getValue();
            System.out.println("Inserted ID: " + newId);

            // Bulk insert
            List<Document> batch = List.of(
                new Document("name", "Bob").append("score", 82.0).append("status", "ENROLLED"),
                new Document("name", "Carol").append("score", 97.0).append("status", "ENROLLED")
            );
            students.insertMany(batch);

            // ── FIND ──
            // Filters.eq is a type-safe way to build query conditions
            Document found = students.find(Filters.eq("name", "Alice")).first();
            System.out.println("Found: " + found.getString("name"));

            // More complex filters
            FindIterable<Document> highScorers = students.find(
                Filters.and(
                    Filters.gte("score", 90.0),         // score >= 90
                    Filters.eq("status", "ENROLLED")     // status == "ENROLLED"
                )
            ).sort(Sorts.descending("score"))            // ORDER BY score DESC
             .limit(10)                                  // LIMIT 10
             .projection(Projections.include("name", "score")); // SELECT name, score only

            for (Document doc : highScorers) {
                System.out.println(doc.getString("name") + ": " + doc.getDouble("score"));
            }

            // Querying nested fields using dot notation
            FindIterable<Document> newYorkers = students.find(
                Filters.eq("address.city", "New York") // Access embedded document field
            );

            // Querying array fields — "courses" array contains "Java"
            FindIterable<Document> javaCourseStudents = students.find(
                Filters.in("courses", "Java")
            );

            // ── UPDATE ──
            // updateOne: updates the first matching document
            UpdateResult updateResult = students.updateOne(
                Filters.eq("name", "Alice"),
                Updates.combine(
                    Updates.set("score", 98.0),     // SET score = 98
                    Updates.set("status", "GRADUATED"), // SET status = "GRADUATED"
                    Updates.push("courses", "Spring Boot") // Append to array
                )
            );
            System.out.println("Modified count: " + updateResult.getModifiedCount());

            // updateMany: updates all matching documents
            students.updateMany(
                Filters.lt("score", 50.0),           // WHERE score < 50
                Updates.set("status", "AT_RISK")     // SET status = "AT_RISK"
            );

            // Upsert: update if exists, insert if not
            students.updateOne(
                Filters.eq("email", "dave@example.com"),
                Updates.set("name", "Dave"),
                new UpdateOptions().upsert(true)
            );

            // ── DELETE ──
            DeleteResult deleteResult = students.deleteOne(
                Filters.eq("name", "Bob")
            );
            System.out.println("Deleted: " + deleteResult.getDeletedCount());

            // ── AGGREGATION PIPELINE ──
            // MongoDB's aggregation framework is its equivalent of SQL's GROUP BY, JOIN, etc.
            // Each stage transforms the documents and passes them to the next stage
            List<Document> stats = students.aggregate(List.of(
                // Stage 1: Filter to only enrolled students
                Aggregates.match(Filters.eq("status", "ENROLLED")),
                // Stage 2: Group by status and compute avg score + count
                Aggregates.group(
                    "$status",
                    Accumulators.avg("averageScore", "$score"),
                    Accumulators.sum("count", 1)
                ),
                // Stage 3: Sort by average score descending
                Aggregates.sort(Sorts.descending("averageScore"))
            )).into(new ArrayList<>());

            stats.forEach(System.out::println);

            // ── CREATE INDEX ──
            // Indexes dramatically speed up queries — without them, MongoDB does a full scan
            students.createIndex(Indexes.ascending("email"),
                new IndexOptions().unique(true)); // Unique index

            students.createIndex(Indexes.compoundIndex(
                Indexes.ascending("status"),
                Indexes.descending("score")
            ));

            // Text index for full-text search
            students.createIndex(Indexes.text("name"));
            students.find(Filters.text("Alice")).forEach(System.out::println);
        }
    }
}
```

### 1.2 Spring Data MongoDB

Spring Data MongoDB provides the same conveniences as Spring Data JPA but for MongoDB — `@Document` replaces `@Entity`, and `MongoRepository` replaces `JpaRepository`.

```java
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import org.springframework.data.mongodb.core.mapping.Field;
import org.springframework.data.mongodb.core.index.Indexed;

// @Document maps this class to a MongoDB collection
// If name is omitted, the collection name defaults to the class name in camelCase
@Document(collection = "students")
public class Student {

    @Id // Maps to MongoDB's "_id" field; Spring Data handles ObjectId ↔ String conversion
    private String id;

    @Field("full_name") // Store as "full_name" in MongoDB, but access as "name" in Java
    private String name;

    @Indexed(unique = true) // Create a unique index on this field at startup
    private String email;

    private Double score;
    private String status;

    private List<String> courses = new ArrayList<>();

    // Embedded document — stored inline in the same MongoDB document (not a JOIN)
    private Address address;

    // Constructors, getters, setters...
    public record Address(String city, String zip) {}
}
```

```java
// MongoRepository gives you all the same derived query methods as JpaRepository
public interface StudentRepository extends MongoRepository<Student, String> {

    List<Student> findByStatus(String status);
    List<Student> findByScoreGreaterThan(double minScore);
    Optional<Student> findByEmail(String email);
    List<Student> findByCoursesContaining(String course); // Array contains an element
    List<Student> findByAddressCity(String city);         // Nested field query

    // Text search (requires a text index)
    @TextIndexed // Or configure via @Document
    List<Student> findByNameContaining(String text);
}
```

```java
// MongoTemplate gives you lower-level access for complex queries
@Service
public class StudentMongoService {

    private final MongoTemplate mongoTemplate;

    public StudentMongoService(MongoTemplate mongoTemplate) {
        this.mongoTemplate = mongoTemplate;
    }

    public List<Student> findHighScorersInDepartment(String status, double minScore) {
        Query query = new Query();
        query.addCriteria(Criteria.where("status").is(status)
                                  .and("score").gte(minScore));
        query.with(Sort.by(Sort.Direction.DESC, "score"));
        query.limit(20);
        return mongoTemplate.find(query, Student.class);
    }

    public UpdateResult promoteTopStudents(double minScore) {
        Query query = Query.query(Criteria.where("score").gte(minScore));
        Update update = new Update().set("status", "GRADUATED");
        return mongoTemplate.updateMulti(query, update, Student.class);
    }
}
```

---

## ⚡ Part 2 — Redis

Redis is an **in-memory data structure store**. It holds all data in RAM, which is why reads and writes happen in microseconds. It supports a rich set of data structures: strings, hashes, lists, sets, sorted sets, streams, and more.

The most common Java use cases for Redis are session storage, caching expensive computations, rate limiting, distributed locks, and real-time leaderboards.

### Maven Dependencies

```xml
<!-- Lettuce is the modern, reactive-capable Redis client (used by Spring by default) -->
<dependency>
    <groupId>io.lettuce</groupId>
    <artifactId>lettuce-core</artifactId>
    <version>6.3.2.RELEASE</version>
</dependency>

<!-- Or, for Spring Boot (includes Spring Data Redis + Lettuce) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### 2.1 Raw Lettuce Client

```java
import io.lettuce.core.*;
import io.lettuce.core.api.StatefulRedisConnection;
import io.lettuce.core.api.sync.RedisCommands;

public class RedisDemo {

    public static void main(String[] args) {
        RedisClient client = RedisClient.create("redis://localhost:6379");

        try (StatefulRedisConnection<String, String> conn = client.connect()) {
            RedisCommands<String, String> redis = conn.sync();

            // ── STRING ── (most basic type)
            redis.set("greeting", "Hello, Redis!");
            String val = redis.get("greeting");
            System.out.println(val); // Hello, Redis!

            // Set with expiry (TTL — Time To Live)
            redis.setex("session:user123", 3600, "{ userId: 123, role: 'ADMIN' }");
            Long ttl = redis.ttl("session:user123"); // Seconds remaining
            System.out.println("TTL: " + ttl + "s");

            // Atomic counter (thread-safe — great for rate limiting or counting)
            redis.set("page:views", "0");
            redis.incr("page:views");     // Increment by 1
            redis.incrby("page:views", 5); // Increment by 5
            System.out.println("Views: " + redis.get("page:views")); // 6

            // ── HASH ── (like a Java Map; perfect for storing object fields)
            redis.hset("student:1", "name",  "Alice");
            redis.hset("student:1", "score", "95.5");
            redis.hset("student:1", "email", "alice@example.com");

            String name = redis.hget("student:1", "name"); // Get one field
            Map<String, String> student = redis.hgetall("student:1"); // Get all fields
            System.out.println("All fields: " + student);

            // ── LIST ── (ordered, allows duplicates; great for queues and activity feeds)
            redis.rpush("activity:user1", "logged_in");    // Push to right (enqueue)
            redis.rpush("activity:user1", "viewed:course:42");
            redis.rpush("activity:user1", "submitted:quiz:7");
            redis.lpush("activity:user1", "session_start"); // Push to left

            List<String> activities = redis.lrange("activity:user1", 0, -1); // All items
            System.out.println("Activities: " + activities);

            String next = redis.lpop("activity:user1"); // Pop from left (dequeue)

            // ── SET ── (unordered, no duplicates; great for tags, unique visitors)
            redis.sadd("tags:course:1", "java", "programming", "backend");
            redis.sadd("tags:course:2", "java", "spring", "backend");

            // Set operations
            Set<String> commonTags = redis.sinter("tags:course:1", "tags:course:2"); // Intersection
            Set<String> allTags    = redis.sunion("tags:course:1", "tags:course:2"); // Union
            System.out.println("Common tags: " + commonTags); // [java, backend]

            // ── SORTED SET ── (set with a numeric score; elements auto-sorted by score)
            // Perfect for leaderboards, priority queues, time-series ranking
            redis.zadd("leaderboard", 95.5, "alice");
            redis.zadd("leaderboard", 82.0, "bob");
            redis.zadd("leaderboard", 97.0, "carol");

            // Get top 3 with scores (highest first — use ZREVRANGE for descending order)
            List<ScoredValue<String>> top3 = redis.zrevrangeWithScores(
                "leaderboard", 0, 2 // Top 3 (0-based)
            );
            top3.forEach(sv -> System.out.printf("%s: %.1f%n", sv.getValue(), sv.getScore()));

            // Get rank of a specific member (0 = top rank)
            Long carolRank = redis.zrevrank("leaderboard", "carol");
            System.out.println("Carol's rank: " + carolRank); // 0

            // ── KEY MANAGEMENT ──
            boolean exists = redis.exists("greeting") > 0;
            redis.del("greeting");                   // Delete a key
            redis.expire("session:user123", 7200L);  // Update TTL
            Set<String> allKeys = redis.keys("student:*"); // Wildcard pattern (use SCAN in production!)
        }

        client.shutdown();
    }
}
```

> ⚠️ **Never use `redis.keys("*")` in production.** It blocks the single-threaded Redis server while scanning all keys, which can cause seconds of latency for all other clients. Use `redis.scan()` instead — it iterates incrementally and is safe for production.

### 2.2 Spring Data Redis & Spring Cache

Spring Data Redis provides templates for Redis operations, and Spring Cache gives you a declarative caching abstraction that works with Redis (or any other cache) using annotations.

```yaml
# application.yml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: ""          # Set if Redis requires authentication
      timeout: 2000ms       # Connection timeout
  cache:
    type: redis
    redis:
      time-to-live: 600000  # 10 minutes default TTL for all caches
```

```java
@SpringBootApplication
@EnableCaching // Activates Spring's caching annotation support
public class SchoolApplication { }
```

```java
@Service
public class DepartmentService {

    private final DepartmentRepository departmentRepo;

    public DepartmentService(DepartmentRepository departmentRepo) {
        this.departmentRepo = departmentRepo;
    }

    // @Cacheable: before executing, check if result is already in cache
    // If yes, return the cached value. If no, execute the method and cache the result.
    // The cache key is derived from the method parameter ("depts::" + id)
    @Cacheable(value = "departments", key = "#id")
    public Department findById(Long id) {
        // This database call is SKIPPED if the result is in cache
        System.out.println("Loading from database: " + id);
        return departmentRepo.findById(id).orElseThrow();
    }

    // @CachePut: ALWAYS executes the method AND updates the cache
    // Use this when you update an entity — keeps the cache in sync
    @CachePut(value = "departments", key = "#department.id")
    public Department update(Department department) {
        Department saved = departmentRepo.save(department);
        return saved; // The returned value is stored in the cache
    }

    // @CacheEvict: removes entries from the cache
    // Use this when you delete an entity — prevents stale data
    @CacheEvict(value = "departments", key = "#id")
    public void delete(Long id) {
        departmentRepo.deleteById(id);
    }

    // Clear the entire "departments" cache at once
    @CacheEvict(value = "departments", allEntries = true)
    public void clearDepartmentCache() {}

    // @Caching: apply multiple cache annotations to one method
    @Caching(evict = {
        @CacheEvict(value = "departments", allEntries = true),
        @CacheEvict(value = "departmentStats", allEntries = true)
    })
    public void importDepartments(List<Department> departments) {
        departmentRepo.saveAll(departments);
    }
}
```

---

## 🪐 Part 3 — Apache Cassandra

Apache Cassandra is a **distributed, wide-column database** designed to handle massive amounts of data across many commodity servers with no single point of failure. Unlike relational databases and MongoDB, Cassandra is write-optimised and built for linear scalability.

The central rule in Cassandra data modelling is: **design your tables around your queries, not around your entities.** You cannot do arbitrary JOINs or ad-hoc queries in Cassandra — you model your data so that each query is a simple primary key lookup.

### Maven Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-cassandra</artifactId>
</dependency>
```

### Cassandra Data Modelling Concepts

```
Cassandra Concept     SQL Equivalent
──────────────────────────────────────────────────
Keyspace              Database / Schema
Table                 Table
Primary Key           Composite key: Partition Key + Clustering Keys
Partition Key         Determines which node stores the data
Clustering Key        Determines the order of rows within a partition
Row                   Row
Column                Column (but sparse — can differ per row)
```

### Spring Data Cassandra Example

```java
import org.springframework.data.cassandra.core.mapping.*;
import java.time.Instant;
import java.util.UUID;

// @Table maps to a Cassandra table
@Table("sensor_readings")
public class SensorReading {

    // The PRIMARY KEY in Cassandra = Partition Key + Clustering Keys
    // Here: sensor_id is the partition key (all readings for a sensor go to the same node)
    //       timestamp is the clustering key (rows within a partition are sorted by timestamp)
    //       reading_id ensures uniqueness

    @PrimaryKeyColumn(name = "sensor_id", type = PrimaryKeyType.PARTITIONED)
    private String sensorId;   // Partition key — routes data to the correct node

    @PrimaryKeyColumn(name = "recorded_at", type = PrimaryKeyType.CLUSTERED,
                      ordering = Ordering.DESCENDING) // Most recent first within a partition
    private Instant recordedAt; // Clustering key — controls sort order

    @PrimaryKeyColumn(name = "reading_id", type = PrimaryKeyType.CLUSTERED)
    private UUID readingId;    // For uniqueness within the same second

    @Column("temperature")
    private Double temperature;

    @Column("humidity")
    private Double humidity;

    // Constructors, getters, setters...
}
```

```java
// CassandraRepository works the same as JpaRepository and MongoRepository
public interface SensorReadingRepository
        extends CassandraRepository<SensorReading, SensorReadingPrimaryKey> {

    // Cassandra queries MUST use the primary key (or indexed columns)
    // This query is efficient: it targets a single partition (sensor_id)
    // and filters by the clustering key range
    List<SensorReading> findBySensorIdAndRecordedAtBetween(
        String sensorId, Instant from, Instant to
    );

    // ❌ This would NOT work efficiently in Cassandra (requires an index or ALLOW FILTERING):
    // List<SensorReading> findByTemperatureGreaterThan(double temp); -- avoid this
}
```

```java
// CassandraTemplate for custom queries
@Service
public class SensorService {

    private final CassandraTemplate cassandraTemplate;

    public SensorService(CassandraTemplate cassandraTemplate) {
        this.cassandraTemplate = cassandraTemplate;
    }

    // CQL (Cassandra Query Language) is similar to SQL but more restricted
    public List<SensorReading> findLatestReadings(String sensorId, int limit) {
        String cql = "SELECT * FROM sensor_readings WHERE sensor_id = ? LIMIT ?";
        return cassandraTemplate.select(cql, SensorReading.class, sensorId, limit);
    }
}
```

---

## 🔍 Part 4 — Elasticsearch

Elasticsearch is a **distributed search and analytics engine** built on Apache Lucene. It is a document store like MongoDB, but its primary strength is full-text search: you can search across millions of documents in milliseconds, with support for fuzzy matching, relevance scoring, aggregations, and complex boolean queries.

Common use cases include application log aggregation and searching (the "E" in the ELK stack), e-commerce product search, autocomplete suggestions, and business analytics dashboards.

### Maven Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  elasticsearch:
    uris: http://localhost:9200
    username: elastic
    password: changeme
```

### Spring Data Elasticsearch Example

```java
import org.springframework.data.elasticsearch.annotations.*;

// @Document maps this class to an Elasticsearch index
@Document(indexName = "products")
@Setting(shards = 1, replicas = 1)
public class Product {

    @Id
    private String id;

    // @Field controls how Elasticsearch indexes the field
    // FieldType.Text: analysed full-text (broken into tokens, lowercased, stemmed)
    // FieldType.Keyword: exact match only (not analysed) — good for filtering and sorting
    @Field(type = FieldType.Text, analyzer = "english")
    private String name;

    @Field(type = FieldType.Text, analyzer = "english")
    private String description;

    @Field(type = FieldType.Keyword) // Exact match (not analyzed)
    private String category;

    @Field(type = FieldType.Double)
    private Double price;

    @Field(type = FieldType.Integer)
    private Integer stockCount;

    @Field(type = FieldType.Date, format = DateFormat.date_time)
    private LocalDateTime createdAt;

    // Constructors, getters, setters...
}
```

```java
public interface ProductRepository extends ElasticsearchRepository<Product, String> {

    // Full text search on the name field
    List<Product> findByName(String name);

    // Range query
    List<Product> findByPriceBetween(double min, double max);

    // Combine full-text and filter
    List<Product> findByNameContainingAndCategory(String nameQuery, String category);
}
```

```java
@Service
public class ProductSearchService {

    private final ElasticsearchOperations elasticsearchOps;

    public ProductSearchService(ElasticsearchOperations elasticsearchOps) {
        this.elasticsearchOps = elasticsearchOps;
    }

    public SearchHits<Product> searchProducts(String searchText, String category,
                                               double minPrice, double maxPrice) {

        // Build a boolean query — combine must (AND), should (OR), filter (AND, no scoring)
        Query query = NativeQuery.builder()
            .withQuery(q -> q
                .bool(b -> b
                    .must(m -> m
                        .multiMatch(mm -> mm      // Search across multiple fields
                            .query(searchText)
                            .fields("name^2", "description") // name weighted 2x more
                        )
                    )
                    .filter(f -> f
                        .term(t -> t.field("category").value(category)) // Exact category match
                    )
                    .filter(f -> f
                        .range(r -> r.field("price").gte(JsonData.of(minPrice))
                                                    .lte(JsonData.of(maxPrice)))
                    )
                )
            )
            .withSort(s -> s.score(sc -> sc))    // Sort by relevance score (default)
            .withPageable(PageRequest.of(0, 20)) // Paginate results
            .withHighlightQuery(                 // Highlight matching text in results
                HighlightQuery.of(h -> h
                    .withFields(Product.class)
                    .withHighlightParameters(hp -> hp.preTags("<em>").postTags("</em>"))
                )
            )
            .build();

        return elasticsearchOps.search(query, Product.class);
    }

    public Map<String, Long> aggregateByCategory() {
        // Aggregations are Elasticsearch's equivalent of GROUP BY
        Query query = NativeQuery.builder()
            .withAggregation("by_category",
                Aggregation.of(a -> a
                    .terms(t -> t.field("category").size(50)) // Top 50 categories by count
                )
            )
            .build();

        SearchHits<Product> hits = elasticsearchOps.search(query, Product.class);
        // Process aggregation results...
        return Map.of(); // Simplified — real code would parse the aggregation response
    }
}
```

---

## 📊 Choosing the Right Database

A question you will be asked in every system design interview and architecture discussion is: "which database should I use for this?" Here is a practical framework for thinking through it:

Use a **relational database** (PostgreSQL, MySQL) as your default for any data with well-defined relationships, when you need ACID transactions across multiple tables, when your data has a known structure, or when you need ad-hoc reporting queries. Postgres in particular is so capable that you should try hard to make it work before reaching for a specialised database.

Use **MongoDB** when your documents have variable schemas (different products have different attributes), when you're storing hierarchical data that would require many JOINs in SQL, or when you're building with evolving schemas in early-stage projects. Avoid it when you have strong relational data or need multi-document ACID transactions (though Mongo does support these now, it's not its strength).

Use **Redis** for caching (reduce load on your primary database), session storage (shared session data across multiple app instances), real-time features (leaderboards, counters, rate limiting), and as a message broker for lightweight pub/sub. Never use Redis as your primary, durable database — it is in-memory first and persistence is optional.

Use **Cassandra** when you have write-heavy workloads at massive scale (millions of writes per second), time-series data (IoT sensor readings, user activity events), and when you need geographic distribution with multi-datacenter replication. The trade-off is the strict data modelling discipline — you must design tables around your queries.

Use **Elasticsearch** for any kind of full-text search, log aggregation and searching (the ELK stack), autocomplete, and faceted search with aggregations. It is rarely the only database in a system — typically it runs alongside a primary database, which pushes data into Elasticsearch asynchronously.

---

## ✅ Best Practices & Important Points

**Model MongoDB documents for your access patterns.** The power of MongoDB is embedding related data in a single document (avoiding joins). But embed only data that is always accessed together and that won't grow unboundedly. If a document could grow to megabytes (e.g., embedding all user activity ever), use references (foreign keys) instead.

**Always create indexes before going to production.** Without indexes, MongoDB and Elasticsearch do full collection/index scans. Use `explain()` in MongoDB to see the query execution plan and confirm your indexes are being used.

**Set TTLs (Time-To-Live) on Redis keys that don't need to live forever.** A Redis instance with no TTLs set will eventually fill up and start evicting data (or fail). Session keys, cache entries, rate limit counters — these should all have TTLs.

**Do not treat Redis as a primary database.** Redis is designed for speed, not durability. While it supports persistence modes (RDB snapshots and AOF logging), it is not designed to replace a durable primary database. Use it as a complementary cache and secondary store.

**In Cassandra, think partition size.** If your partition key is poorly chosen (e.g., using a date that groups all of a day's data in one partition), you can end up with "hot partitions" — a single node receiving all the traffic and others sitting idle. Design partition keys so that data is evenly distributed across nodes.

**Use the sync and async drivers correctly.** Lettuce (the default Spring Redis client) supports both synchronous and reactive modes. Spring WebFlux + reactive Redis is a powerful combination for highly concurrent non-blocking applications. Jedis is synchronous-only and simpler but doesn't scale as well under high concurrency.

---

## 🔑 Key Summary

```
MongoDB      → Document store; flexible JSON schemas; great for variable-structure data
              → MongoDB Java Driver (raw) or Spring Data MongoDB (@Document, MongoRepository)
              → Design: model for access patterns; embed related data; always index

Redis        → In-memory key-value store; microsecond latency; data structures beyond strings
              → Lettuce (raw) or Spring Data Redis (RedisTemplate, @Cacheable)
              → Design: always set TTLs; never use as primary database; use SCAN not KEYS

Cassandra    → Distributed wide-column store; massive write throughput; no single point of failure
              → DataStax Driver or Spring Data Cassandra (@Table, CassandraRepository)
              → Design: model tables per query; choose partition keys for even distribution

Elasticsearch → Distributed search engine; inverted index; full-text search + aggregations
              → Java API Client or Spring Data Elasticsearch (@Document, ElasticsearchRepository)
              → Design: use Text type for full-text, Keyword for filtering; denormalise documents
```
