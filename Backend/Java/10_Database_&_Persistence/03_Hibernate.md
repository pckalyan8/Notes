# 📙 10.3 — Hibernate ORM (Deep Dive)

Hibernate is the most widely used ORM framework in the Java world. While the previous chapter covered JPA as a specification, Hibernate is what actually runs — it is the engine behind the scenes. When you use JPA annotations, Hibernate interprets them and generates the SQL. Understanding Hibernate's internal mechanics, especially its caching system and session behaviour, is what separates developers who "use Hibernate" from those who truly master it.

---

## 🧠 Hibernate's Place in the Stack

Hibernate implements the JPA specification, meaning every JPA annotation and API you learned works with Hibernate. But Hibernate also offers its own additional annotations and APIs that go beyond the JPA standard, giving you more control when you need it.

```
Your Code
    │ uses
    ▼
JPA API (interfaces, annotations)   ← Standard; portable
    │ implemented by
    ▼
Hibernate ORM                       ← The actual engine; vendor-specific extensions available
    │ calls
    ▼
JDBC API
    │ uses
    ▼
Database Driver (e.g., PostgreSQL)
    │ connects to
    ▼
PostgreSQL / MySQL / Oracle / ...
```

---

## ⚙️ Configuration

Hibernate can be configured either through `persistence.xml` (standard JPA approach) or through `hibernate.cfg.xml` (Hibernate-native approach). In modern Spring Boot applications you rarely deal with either file directly — Spring Boot auto-configures everything from `application.properties`. But understanding the raw configuration is important when debugging problems or building non-Spring applications.

### hibernate.cfg.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE hibernate-configuration PUBLIC
    "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
    "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">

<hibernate-configuration>
    <session-factory>
        <!-- Database connection settings -->
        <property name="hibernate.connection.driver_class">org.postgresql.Driver</property>
        <property name="hibernate.connection.url">jdbc:postgresql://localhost:5432/school_db</property>
        <property name="hibernate.connection.username">postgres</property>
        <property name="hibernate.connection.password">secret</property>

        <!-- SQL dialect — tells Hibernate which SQL variant to generate -->
        <property name="hibernate.dialect">org.hibernate.dialect.PostgreSQLDialect</property>

        <!-- Schema management -->
        <property name="hibernate.hbm2ddl.auto">create-drop</property>

        <!-- Debugging: print SQL and format it nicely -->
        <property name="hibernate.show_sql">true</property>
        <property name="hibernate.format_sql">true</property>

        <!-- Built-in connection pool (NOT for production — use HikariCP instead) -->
        <property name="hibernate.connection.pool_size">10</property>

        <!-- HikariCP integration (production-ready) -->
        <property name="hibernate.connection.provider_class">
            org.hibernate.hikaricp.internal.HikariCPConnectionProvider
        </property>
        <property name="hibernate.hikari.maximumPoolSize">10</property>
        <property name="hibernate.hikari.minimumIdle">2</property>

        <!-- Entity classes to register -->
        <mapping class="com.example.entity.Student"/>
        <mapping class="com.example.entity.Department"/>
        <mapping class="com.example.entity.Course"/>
    </session-factory>
</hibernate-configuration>
```

### Building a SessionFactory

```java
import org.hibernate.SessionFactory;
import org.hibernate.boot.MetadataSources;
import org.hibernate.boot.registry.StandardServiceRegistry;
import org.hibernate.boot.registry.StandardServiceRegistryBuilder;

public class HibernateUtil {

    // SessionFactory is expensive to create — it is thread-safe and should be a singleton
    private static SessionFactory sessionFactory;

    static {
        try {
            // Reads hibernate.cfg.xml from the classpath automatically
            StandardServiceRegistry registry = new StandardServiceRegistryBuilder()
                .configure()
                .build();

            sessionFactory = new MetadataSources(registry)
                .buildMetadata()
                .buildSessionFactory();

        } catch (Exception e) {
            throw new ExceptionInInitializerError("Failed to create SessionFactory: " + e);
        }
    }

    public static SessionFactory getSessionFactory() {
        return sessionFactory;
    }

    public static void shutdown() {
        if (sessionFactory != null) {
            sessionFactory.close();
        }
    }
}
```

---

## 💼 Session API — Hibernate's Native Interface

The Hibernate `Session` interface is the Hibernate-native equivalent of the JPA `EntityManager`. In fact, Hibernate's `EntityManager` implementation wraps a `Session` internally — you can always get the `Session` from an `EntityManager` by calling `em.unwrap(Session.class)`.

The operations are nearly identical to JPA's `EntityManager`, but the Session adds some Hibernate-specific methods.

```java
import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.Transaction;
import org.hibernate.query.Query;

public class StudentDao {

    private final SessionFactory sf = HibernateUtil.getSessionFactory();

    // ────────────── CREATE ──────────────

    public Student save(Student student) {
        // try-with-resources closes the session automatically
        try (Session session = sf.openSession()) {
            Transaction tx = session.beginTransaction();
            try {
                // persist() is the JPA-style method; save() is the Hibernate-native equivalent
                // Use persist() — it is the standardised JPA method
                session.persist(student);
                tx.commit();
                return student;
            } catch (Exception e) {
                tx.rollback();
                throw e;
            }
        }
    }

    // ────────────── READ ──────────────

    public Student findById(Long id) {
        try (Session session = sf.openSession()) {
            // get() is the Hibernate-native equivalent of JPA's find()
            // Returns null if not found (does NOT throw)
            return session.get(Student.class, id);
        }
    }

    // ────────────── UPDATE (Dirty Checking) ──────────────

    public void updateScore(Long id, double newScore) {
        try (Session session = sf.openSession()) {
            Transaction tx = session.beginTransaction();
            try {
                Student student = session.get(Student.class, id);
                if (student != null) {
                    student.setScore(newScore); // Just modify — no explicit update() call needed!
                    // Hibernate compares the original snapshot to the current state at flush time
                    // and generates an UPDATE statement for changed fields automatically
                }
                tx.commit();
            } catch (Exception e) {
                tx.rollback();
                throw e;
            }
        }
    }

    // ────────────── DELETE ──────────────

    public void delete(Long id) {
        try (Session session = sf.openSession()) {
            Transaction tx = session.beginTransaction();
            try {
                Student student = session.get(Student.class, id);
                if (student != null) {
                    session.remove(student);
                }
                tx.commit();
            } catch (Exception e) {
                tx.rollback();
                throw e;
            }
        }
    }

    // ────────────── HQL QUERY ──────────────

    public List<Student> findByGrade(String grade) {
        try (Session session = sf.openSession()) {
            // HQL (Hibernate Query Language) is Hibernate's flavour of JPQL
            // The syntax is identical to JPQL — they are effectively the same
            Query<Student> query = session.createQuery(
                "FROM Student s WHERE s.grade = :grade ORDER BY s.score DESC",
                Student.class
            );
            query.setParameter("grade", grade);
            return query.list();
        }
    }

    // ────────────── NATIVE SQL ──────────────

    public List<Object[]> getScoreStatsByDepartment() {
        try (Session session = sf.openSession()) {
            // Sometimes you need native SQL for database-specific features
            // or complex queries that JPQL can't express efficiently
            return session.createNativeQuery(
                "SELECT d.name, AVG(s.score), MAX(s.score), MIN(s.score) " +
                "FROM students s JOIN departments d ON s.department_id = d.id " +
                "GROUP BY d.name",
                Object[].class
            ).list();
        }
    }
}
```

---

## 🗄️ Hibernate Caching — The Key to Performance

Caching is Hibernate's most powerful performance feature, and the most misunderstood. Hibernate has two levels of cache, and they serve completely different purposes.

### First-Level Cache (Session Cache) — Always Active

The first-level cache is a **per-session identity map** that lives inside a single `Session`. Within one session, Hibernate will never load the same entity twice — it returns the already-loaded object from the cache on the second call. This is automatic and cannot be disabled.

```java
try (Session session = sf.openSession()) {
    // First call: Hibernate fires: SELECT * FROM students WHERE id = 1
    Student s1 = session.get(Student.class, 1L);

    // Second call in the SAME session: NO SQL fired — returns cached object
    Student s2 = session.get(Student.class, 1L);

    // Both variables point to the EXACT SAME object in memory
    System.out.println(s1 == s2); // true

    // The cache also handles cycles: if A references B and B references A,
    // Hibernate won't infinite-loop; it returns the already-cached object
}
// When the session closes, the first-level cache is DISCARDED
// The next session has an empty cache and will re-query the database
```

You can explicitly control the first-level cache with `session.evict(entity)` to remove one entity or `session.clear()` to wipe the entire cache. This is useful in long batch-processing operations to prevent the cache from consuming too much memory.

### Second-Level Cache — Shared, Optional, Configured

The second-level cache is a **shared cache** that survives beyond individual `Session` lifetimes. It is scoped to the `SessionFactory` and shared across all sessions in the application. Unlike the first-level cache, it is **opt-in** — you must configure it and annotate each entity you want cached.

The most common second-level cache provider for Hibernate is **Ehcache** (or its modern replacement, **Caffeine via JCacheCachingProvider**).

```xml
<!-- Enable second-level cache in hibernate.cfg.xml or persistence.xml -->
<property name="hibernate.cache.use_second_level_cache">true</property>
<property name="hibernate.cache.use_query_cache">true</property>
<property name="hibernate.cache.region.factory_class">
    org.hibernate.cache.jcache.JCacheCachingProvider
</property>
```

```java
import org.hibernate.annotations.Cache;
import org.hibernate.annotations.CacheConcurrencyStrategy;

@Entity
@Table(name = "departments")
// Mark this entity as cacheable in the second-level cache
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Department {
    // Entities that are frequently read but rarely updated are great candidates
    // for second-level caching (lookup tables, reference data, etc.)

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
}
```

The four cache concurrency strategies are `READ_ONLY` (for immutable data — fastest), `NONSTRICT_READ_WRITE` (allows brief stale reads; no locking overhead), `READ_WRITE` (uses soft locks to ensure consistency; good for most cases), and `TRANSACTIONAL` (full transaction support; only for JTA environments).

```java
// Query cache — caches the result set of a specific query
// Only useful if the query is run many times with the same parameters
try (Session session = sf.openSession()) {
    List<Department> departments = session.createQuery(
        "FROM Department ORDER BY name", Department.class
    )
    .setCacheable(true)           // Enable query caching for this query
    .setCacheRegion("allDepts")   // Named cache region (configured separately)
    .list();
}
```

---

## 📝 HQL (Hibernate Query Language)

HQL is Hibernate's own query language. It is syntactically almost identical to JPQL (which was actually based on HQL). In modern Hibernate + JPA code, you use JPQL (via `em.createQuery()`) and they are interchangeable. The key additional HQL feature is that you can sometimes access Hibernate-specific functions and extensions.

```java
try (Session session = sf.openSession()) {

    // Standard select
    List<Student> students = session.createQuery(
        "FROM Student WHERE score > 90", Student.class
    ).list();

    // Join fetch to solve N+1
    List<Student> withDept = session.createQuery(
        "SELECT s FROM Student s JOIN FETCH s.department WHERE s.score > :min",
        Student.class
    ).setParameter("min", 85.0).list();

    // Aggregate with GROUP BY and HAVING
    List<Object[]> stats = session.createQuery(
        "SELECT s.department.name, COUNT(s), AVG(s.score) " +
        "FROM Student s " +
        "GROUP BY s.department.name " +
        "HAVING COUNT(s) > 5",
        Object[].class
    ).list();

    // Sub-query
    List<Student> topStudents = session.createQuery(
        "FROM Student s WHERE s.score > " +
        "  (SELECT AVG(s2.score) FROM Student s2)",
        Student.class
    ).list();
}
```

---

## 🏷️ Important Hibernate-Specific Annotations

Beyond the standard JPA annotations, Hibernate provides additional annotations for fine-grained control.

### @Formula — Computed Columns

```java
@Entity
@Table(name = "students")
public class Student {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "first_name") private String firstName;
    @Column(name = "last_name")  private String lastName;

    // @Formula maps a field to a SQL expression, not a column
    // The value is calculated by the database on each SELECT — it is NOT stored
    @Formula("(first_name || ' ' || last_name)")
    private String fullName; // Will be populated from the SQL expression
}
```

### @Where — Soft Delete Filter

Soft delete means marking records as deleted without actually removing them from the database. The `@Where` annotation adds a permanent WHERE clause to every query for that entity.

```java
@Entity
@Table(name = "students")
@Where(clause = "is_deleted = false") // This clause is appended to EVERY query
public class Student {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "is_deleted")
    private boolean deleted = false;

    // To "delete" a student: set deleted = true
    // All queries will automatically exclude deleted students
    public void softDelete() {
        this.deleted = true;
    }
}
```

### @BatchSize — Reducing N+1 Queries

When you have a collection that's lazily loaded, the N+1 problem occurs: 1 query to load the parent entities, then N queries to load each child collection individually. `@BatchSize` batches the child collection queries together.

```java
@Entity
public class Department {

    @OneToMany(mappedBy = "department", fetch = FetchType.LAZY)
    @BatchSize(size = 25) // Load 25 departments' student collections at once instead of 1 at a time
    private List<Student> students;
    // Without @BatchSize: 1 query per department to get its students
    // With @BatchSize(25): groups up to 25 departments' IN clause into one query:
    //   SELECT * FROM students WHERE department_id IN (1, 2, 3, ..., 25)
}
```

### @NaturalId — Business Key Mapping

```java
@Entity
public class Student {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id; // Surrogate key (database-generated)

    @NaturalId                    // Business/natural key (stable, meaningful)
    @Column(unique = true, nullable = false)
    private String studentNumber; // e.g., "STU-2024-0042"

    // You can now load by natural ID efficiently (Hibernate caches natural ID lookups)
    // Usage: session.byNaturalId(Student.class).using("studentNumber", "STU-2024-0042").load()
}
```

---

## 🔍 The N+1 Query Problem — Diagnosis and Solutions

The N+1 problem is the most common performance issue in Hibernate applications. It occurs when you load a list of N parent entities and then access a lazy collection on each one, triggering N additional queries.

```java
// ❌ PROBLEM: N+1 queries
try (Session session = sf.openSession()) {
    List<Department> departments = session.createQuery(
        "FROM Department", Department.class
    ).list();
    // SQL 1: SELECT * FROM departments

    for (Department dept : departments) {
        // Each access to getStudents() triggers a new SQL query!
        // SQL 2: SELECT * FROM students WHERE department_id = 1
        // SQL 3: SELECT * FROM students WHERE department_id = 2
        // ...and so on for each department
        System.out.println(dept.getName() + ": " + dept.getStudents().size());
    }
}

// ✅ SOLUTION 1: JOIN FETCH — single query with a SQL JOIN
try (Session session = sf.openSession()) {
    List<Department> departments = session.createQuery(
        "SELECT DISTINCT d FROM Department d JOIN FETCH d.students",
        Department.class
    ).list();
    // Single SQL: SELECT DISTINCT d.*, s.* FROM departments d JOIN students s ON ...
    // All student collections are already loaded — no additional queries
}

// ✅ SOLUTION 2: @BatchSize — groups lazy loads into batches (good for optional loading)
// (configure on the entity as shown in the @BatchSize example above)

// ✅ SOLUTION 3: EntityGraph — JPA standard way to specify eager loading per query
EntityGraph<Department> graph = em.createEntityGraph(Department.class);
graph.addAttributeNodes("students"); // Eagerly load the students collection

List<Department> departments = em.createQuery("FROM Department", Department.class)
    .setHint("jakarta.persistence.loadgraph", graph) // Apply the graph to this query
    .getResultList();

// ✅ SOLUTION 4: DTO projection — only load the data you actually need
List<DepartmentSummary> summaries = session.createQuery(
    "SELECT new DepartmentSummary(d.name, COUNT(s)) " +
    "FROM Department d LEFT JOIN d.students s " +
    "GROUP BY d.name",
    DepartmentSummary.class
).list();
// Most efficient: aggregated at database level, no full entities loaded
```

---

## 🔎 Detecting N+1 in Development

You should always use one of these tools during development to catch N+1 issues before they reach production.

```xml
<!-- Add to Maven dependencies -->
<dependency>
    <groupId>com.vladmihalcea</groupId>
    <artifactId>hibernate-types-60</artifactId>
    <version>2.21.1</version>
</dependency>

<!-- Or use datasource-proxy to count SQL statements in tests -->
<dependency>
    <groupId>net.ttddyy</groupId>
    <artifactId>datasource-proxy</artifactId>
    <version>1.10</version>
    <scope>test</scope>
</dependency>
```

```java
// In tests — assert on the number of SQL queries executed
// This is the "statement count assertion" pattern
@Test
void findDepartmentsWithStudents_shouldNotTriggerNPlus1() {
    // Using datasource-proxy to count queries
    assertSelectCount(1); // This query should only trigger 1 SELECT
    List<Department> departments = departmentRepo.findAllWithStudents();
    // If this assertion fails, you have an N+1 problem
}
```

---

## 🔐 Optimistic vs Pessimistic Locking

When multiple transactions try to update the same row concurrently, you need a locking strategy to prevent **lost updates** (one transaction overwriting another's changes without knowing).

### Optimistic Locking (Recommended for Most Cases)

Optimistic locking assumes conflicts are rare. It doesn't lock the row in the database. Instead, it adds a version column and checks it at update time. If the version has changed since you read the record (someone else updated it), the update is rejected.

```java
@Entity
@Table(name = "students")
public class Student {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private Double score;

    // @Version adds a version column to the table
    // Hibernate increments this value on every UPDATE
    // On each UPDATE, Hibernate adds: WHERE id = ? AND version = ?
    // If no row is updated (version mismatch), it throws OptimisticLockException
    @Version
    private int version;
}
```

```java
// Transaction A reads student with version = 5
// Transaction B reads student with version = 5
// Transaction A updates score to 90 — version becomes 6, commits
// Transaction B tries to update name — WHERE version = 5 matches 0 rows
//   → Hibernate throws OptimisticLockException
//   → Transaction B must reload the entity and retry

try {
    student.setScore(95.0);
    session.merge(student);
    session.getTransaction().commit();
} catch (OptimisticLockException e) {
    // The entity was modified by someone else since we last read it
    // Reload, apply the change again, and retry
    Student freshStudent = session.get(Student.class, student.getId());
    freshStudent.setScore(95.0);
    session.merge(freshStudent);
    session.getTransaction().commit();
}
```

### Pessimistic Locking (For High-Contention Scenarios)

Pessimistic locking acquires an actual database lock on the row, preventing other transactions from modifying it until you release the lock.

```java
try (Session session = sf.openSession()) {
    session.beginTransaction();

    // PESSIMISTIC_WRITE: locks the row with SELECT ... FOR UPDATE
    // Other transactions that try to read or write this row will BLOCK until you release the lock
    Student student = session.get(Student.class, 1L, LockMode.PESSIMISTIC_WRITE);

    student.setScore(100.0);
    session.getTransaction().commit(); // Lock is released when transaction commits
}
```

Use optimistic locking for the vast majority of web applications (most updates don't conflict). Use pessimistic locking only when you have genuinely high-contention data and can't tolerate retries (e.g., a bank account debit that must succeed atomically with an inventory decrement).

---

## 🧪 Bean Validation (Hibernate Validator)

Hibernate Validator is the reference implementation of the **Bean Validation** (JSR-380) specification. It allows you to declare data constraints directly on entity fields using annotations, then validate them either automatically (JPA validates before persist/update) or manually.

```xml
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>8.0.1.Final</version>
</dependency>
```

```java
import jakarta.validation.constraints.*;

@Entity
@Table(name = "students")
public class Student {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank(message = "Name cannot be blank")
    @Size(min = 2, max = 150, message = "Name must be between 2 and 150 characters")
    private String name;

    @NotNull(message = "Email is required")
    @Email(message = "Must be a valid email address")
    private String email;

    @Min(value = 0, message = "Score cannot be negative")
    @Max(value = 100, message = "Score cannot exceed 100")
    private Double score;

    @NotNull
    @Past(message = "Date of birth must be in the past")
    private LocalDate dateOfBirth;

    @Pattern(regexp = "^[A-Z]{3}-\\d{4}-\\d{4}$", message = "Student number format: STU-YEAR-NNNN")
    private String studentNumber;
}
```

```java
// Manual validation (outside of JPA)
import jakarta.validation.*;

Validator validator = Validation.buildDefaultValidatorFactory().getValidator();

Student student = new Student("", "not-an-email");
Set<ConstraintViolation<Student>> violations = validator.validate(student);

for (ConstraintViolation<Student> v : violations) {
    System.out.println(v.getPropertyPath() + ": " + v.getMessage());
}
// Output:
// name: Name cannot be blank
// email: Must be a valid email address
```

When JPA is configured and Hibernate Validator is on the classpath, validation runs automatically before `persist()` and `merge()` — saving you from storing invalid data.

---

## ✅ Best Practices & Important Points

**Always use `try-with-resources` for `Session`.** Forgetting to close sessions leads to connection leaks and eventually an exhausted connection pool.

**Never use Hibernate's built-in connection pool in production.** The `hibernate.connection.pool_size` setting uses a minimal pool suitable only for testing. Always configure HikariCP or another production-grade pool.

**Understand flush mode.** By default, Hibernate flushes (syncs pending changes to the database) before each query that might be affected by in-memory changes, and at transaction commit. You can change this with `session.setFlushMode()`, but the default is almost always correct.

**Use `@BatchSize` for lazy collections in list views, and JOIN FETCH for single-entity detail views.** This mental model helps you pick the right solution: when displaying a list page, `@BatchSize` amortises the cost across the whole list; when displaying a detail page for one entity, JOIN FETCH is clean and simple.

**Do not call `session.flush()` manually unless you have a specific reason.** Hibernate knows the optimal time to flush. Manual flushes can cause issues with ordering and can trigger premature constraint violations.

**The second-level cache is not a magic speed button.** Cache only data that is frequently read, rarely written, and acceptable to be briefly stale. Caching data that changes frequently can lead to stale reads and very hard-to-debug bugs.

**Log and review the generated SQL during development.** With `hibernate.show_sql=true` and `hibernate.format_sql=true`, you can see exactly what Hibernate is sending to the database. If you're seeing more queries than you expect, you've found an N+1 problem.

---

## 🔑 Key Summary

```
SessionFactory  → Thread-safe factory for Sessions; singleton; expensive to create
Session         → Single-threaded unit of work; Hibernate's equivalent of JPA's EntityManager
First-Level Cache  → Per-session identity map; automatic; prevents duplicate loads within a session
Second-Level Cache → Shared across sessions; opt-in; use @Cache annotation on entity
HQL             → Hibernate Query Language; almost identical to JPQL
@Formula        → Map a field to a computed SQL expression
@Where          → Add a permanent filter clause to every query (soft delete pattern)
@BatchSize      → Mitigate N+1 by loading collections in batches
@Version        → Enable optimistic locking; prevents lost updates
Hibernate Validator → Bean Validation annotations (@NotNull, @Email, etc.)
N+1 Problem     → Load N parents + N×1 lazy loads = performance disaster; fix with JOIN FETCH
```
