# 📒 10.4 — Spring Data JPA

Spring Data JPA is the highest-level abstraction in the Java persistence stack. It sits on top of JPA (which sits on top of Hibernate, which sits on top of JDBC). By this point in the roadmap you understand every layer beneath it — which means you are in the best possible position to use Spring Data JPA productively and to debug it effectively when things go wrong.

---

## 🧠 What is Spring Data JPA?

Spring Data JPA is part of the larger **Spring Data** family, which provides consistent repository abstractions for many data stores (SQL, MongoDB, Redis, Cassandra, etc.). The SQL branch — Spring Data JPA — eliminates the repetitive boilerplate of writing DAO (Data Access Object) classes by hand.

Here is the key insight: with raw JPA, you write a `StudentRepository` class with `save()`, `findById()`, `findAll()`, `delete()`, and custom JPQL methods. You write this same pattern for `DepartmentRepository`, `CourseRepository`, and every other entity. Spring Data JPA replaces all of that repetitive code with **a single interface declaration**. You declare the interface, and Spring generates the implementation at startup using dynamic proxy classes.

```
What you write:        What Spring Data generates:
─────────────────────  ──────────────────────────────────────────────
interface StudentRepo   class $StudentRepoProxy implements StudentRepo {
  extends JpaRepository   public Student save(Student s) { em.persist(s); ... }
{ }                        public Optional<Student> findById(Long id) { ... }
                           public List<Student> findAll() { ... }
                           // ...all CRUD operations, automatically
                         }
```

---

## 📦 Maven Setup

Spring Data JPA is most commonly used with Spring Boot, which configures everything automatically:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

### application.properties

```properties
# Database connection
spring.datasource.url=jdbc:postgresql://localhost:5432/school_db
spring.datasource.username=postgres
spring.datasource.password=secret

# JPA / Hibernate settings
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

# HikariCP pool (Spring Boot auto-configures HikariCP — these override defaults)
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=2
spring.datasource.hikari.connection-timeout=30000
```

---

## 🏛️ Repository Hierarchy — Choosing the Right Interface

Spring Data JPA provides a hierarchy of repository interfaces. You pick the one closest to your needs.

```
Repository<T, ID>                     ← Marker interface; no methods
  └── CrudRepository<T, ID>           ← Basic CRUD: save, findById, findAll, delete, count
        └── PagingAndSortingRepository ← Adds: findAll(Pageable), findAll(Sort)
              └── JpaRepository<T, ID> ← Adds: flush, saveAllAndFlush, deleteInBatch
                                          Also extends PagingAndSortingRepository
```

In practice, `JpaRepository` is the most commonly used because it gives you everything from all parent interfaces plus JPA-specific bulk operations. Use `CrudRepository` if you want to explicitly limit your repository's surface area.

---

## 🔧 1. Basic Repository Setup

```java
package com.example.repository;

import com.example.entity.Student;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

// JpaRepository<EntityType, PrimaryKeyType>
// Spring generates the implementation at startup — you don't write a class
@Repository // Optional — Spring Data finds it by the interface hierarchy — but good for clarity
public interface StudentRepository extends JpaRepository<Student, Long> {
    // No body needed for basic CRUD — it's all inherited from JpaRepository
}
```

### What You Get for Free

The moment you declare this interface, you have all of these methods available without writing a single line of implementation:

```java
// Saving
Student saved = studentRepo.save(student);         // INSERT or UPDATE (based on whether ID is set)
List<Student> all = studentRepo.saveAll(students); // Batch save

// Finding
Optional<Student> found = studentRepo.findById(1L); // Returns Optional (not null)
List<Student> all      = studentRepo.findAll();
List<Student> byIds    = studentRepo.findAllById(List.of(1L, 2L, 3L));
boolean exists         = studentRepo.existsById(1L);
long count             = studentRepo.count();

// Deleting
studentRepo.deleteById(1L);
studentRepo.delete(student);
studentRepo.deleteAll(someStudents);
studentRepo.deleteAll();        // Careful with this one!
studentRepo.deleteAllById(ids);

// Paging and sorting (from PagingAndSortingRepository)
Page<Student> page = studentRepo.findAll(PageRequest.of(0, 10, Sort.by("name")));

// JPA-specific (from JpaRepository)
studentRepo.flush();                 // Force sync to database immediately
studentRepo.saveAndFlush(student);   // Save and immediately flush
studentRepo.deleteAllInBatch();      // DELETE FROM students — no object loading, no cascades
```

---

## 🔍 2. Derived Query Methods — The Magic Feature

Spring Data's most celebrated feature is **query derivation from method names**. You add a method to your repository interface with a name that follows a specific convention, and Spring Data parses the name and generates the correct JPQL query automatically.

The naming convention follows this pattern: `find(Top/First/Distinct)...By<Condition>...(<OrderBy>...)`

```java
@Repository
public interface StudentRepository extends JpaRepository<Student, Long> {

    // ── Equality ──
    // Spring generates: SELECT s FROM Student s WHERE s.name = :name
    List<Student> findByName(String name);

    // ── Case-insensitive equality ──
    List<Student> findByNameIgnoreCase(String name);

    // ── LIKE ──
    // Spring generates: WHERE s.name LIKE :name (you provide the % characters)
    List<Student> findByNameContaining(String namePart);
    List<Student> findByNameStartingWith(String prefix);
    List<Student> findByNameEndingWith(String suffix);

    // ── Comparison ──
    List<Student> findByScoreGreaterThan(double minScore);
    List<Student> findByScoreGreaterThanEqual(double minScore);
    List<Student> findByScoreLessThan(double maxScore);
    List<Student> findByScoreBetween(double min, double max);

    // ── Null checks ──
    List<Student> findByDepartmentIsNull();
    List<Student> findByDepartmentIsNotNull();

    // ── Boolean ──
    List<Student> findByActiveTrue();
    List<Student> findByActiveFalse();

    // ── Enum / String ──
    List<Student> findByStatus(Student.Status status);

    // ── Combining conditions with AND and OR ──
    List<Student> findByNameAndStatus(String name, Student.Status status);
    List<Student> findByStatusOrGrade(Student.Status status, String grade);

    // ── Navigating relationships (dot notation → generates a JOIN) ──
    List<Student> findByDepartmentName(String departmentName);
    List<Student> findByCoursesTitle(String courseTitle); // For @ManyToMany

    // ── Ordering ──
    List<Student> findByStatusOrderByScoreDesc(Student.Status status);
    List<Student> findByStatusOrderByNameAscScoreDesc(Student.Status status);

    // ── Limiting results ──
    Optional<Student> findFirstByStatusOrderByScoreDesc(Student.Status status); // Top 1
    List<Student> findTop5ByStatusOrderByScoreDesc(Student.Status status);      // Top 5

    // ── Distinct ──
    List<Student> findDistinctByGrade(String grade);

    // ── Existence and count ──
    boolean existsByEmail(String email);
    long countByStatus(Student.Status status);

    // ── Deletion ──
    long deleteByStatus(Student.Status status); // Returns count of deleted rows
    void deleteByDepartmentId(Long departmentId);
}
```

A key thing to appreciate: there's no SQL, no JPQL, no `@Query` — just a method name. Spring Data reads the method name when the application starts, validates it against the entity's fields (throwing an error immediately if a field doesn't exist), and generates the query. This means typos are caught at startup rather than at runtime.

---

## 📝 3. @Query — Custom JPQL and Native SQL

When the method name derivation becomes too complex or unreadable, you can write the query explicitly using `@Query`.

```java
@Repository
public interface StudentRepository extends JpaRepository<Student, Long> {

    // ── Custom JPQL query ──
    @Query("SELECT s FROM Student s WHERE s.score BETWEEN :min AND :max ORDER BY s.score DESC")
    List<Student> findInScoreRange(@Param("min") double min, @Param("max") double max);

    // ── JOIN FETCH to solve N+1 when loading students with their departments ──
    @Query("SELECT s FROM Student s JOIN FETCH s.department WHERE s.status = :status")
    List<Student> findByStatusWithDepartment(@Param("status") Student.Status status);

    // ── DTO Projection (constructor expression) ──
    @Query("SELECT new com.example.dto.StudentSummaryDTO(s.id, s.name, s.score) FROM Student s")
    List<StudentSummaryDTO> findAllSummaries();

    // ── Native SQL — use when JPQL can't express the query ──
    // nativeQuery = true tells Spring to send this SQL directly to the database
    @Query(value = "SELECT * FROM students WHERE score > PERCENTILE_CONT(0.9) " +
                   "WITHIN GROUP (ORDER BY score)", nativeQuery = true)
    List<Student> findTopTenPercent();

    // ── Modifying query (UPDATE or DELETE) — must be in a transaction ──
    @Modifying
    @Transactional
    @Query("UPDATE Student s SET s.status = :newStatus WHERE s.score < :minScore")
    int updateStatusForLowScorers(@Param("newStatus") Student.Status newStatus,
                                  @Param("minScore") double minScore);

    // ── Delete with custom condition ──
    @Modifying
    @Transactional
    @Query("DELETE FROM Student s WHERE s.status = :status AND s.score < :score")
    int deleteByStatusAndScoreBelow(@Param("status") Student.Status status,
                                    @Param("score") double score);
}
```

> ⚠️ Always add `@Modifying` to any `@Query` that modifies data (UPDATE, DELETE). If you forget it, Spring Data JPA will throw an exception because it expects a SELECT from `@Query` by default. Also, modifying queries must run within a transaction — either add `@Transactional` to the repository method itself, or ensure the calling service method is transactional.

---

## 📄 4. Pagination and Sorting

Pagination is crucial for any real application that displays lists. Without it, `findAll()` on a table with a million rows would load all million into memory at once. Spring Data JPA makes pagination nearly effortless.

```java
@Service
@Transactional(readOnly = true)
public class StudentService {

    private final StudentRepository studentRepo;

    public StudentService(StudentRepository studentRepo) {
        this.studentRepo = studentRepo;
    }

    public Page<Student> getStudentPage(int pageNumber, int pageSize, String sortField) {
        // PageRequest combines page number (0-based), page size, and sorting
        Pageable pageable = PageRequest.of(
            pageNumber,              // Page 0 = first page
            pageSize,                // How many items per page
            Sort.by(sortField).descending()  // ORDER BY sortField DESC
        );

        // findAll(Pageable) returns a Page<T> object with rich metadata
        Page<Student> page = studentRepo.findAll(pageable);

        // Page<T> gives you everything you need for a pagination UI
        System.out.println("Total students: "  + page.getTotalElements());
        System.out.println("Total pages: "     + page.getTotalPages());
        System.out.println("Current page: "    + page.getNumber());
        System.out.println("Students on page:" + page.getContent().size());
        System.out.println("Has next page: "   + page.hasNext());
        System.out.println("Is first page: "   + page.isFirst());

        return page;
    }

    // Multi-column sorting
    public List<Student> getSortedStudents() {
        Sort sort = Sort.by(
            Sort.Order.desc("score"),
            Sort.Order.asc("name")
        );
        return studentRepo.findAll(sort); // Returns all, but sorted
    }

    // Pagination with a custom query
    // Add a Pageable parameter to any @Query method — Spring handles pagination automatically
    public Page<Student> findTopStudentsPaged(double minScore, Pageable pageable) {
        // The count query runs separately to get total results for page metadata
        return studentRepo.findByScoreGreaterThan(minScore, pageable);
    }
}
```

> 💡 `Slice<T>` is a lighter alternative to `Page<T>`. `Page` runs an extra `COUNT(*)` query to calculate total pages. `Slice` skips the count query — it only tells you if there's a next page. Use `Slice` for infinite-scroll UIs where total page count doesn't matter.

---

## 🏗️ 5. Auditing — @CreatedDate, @LastModifiedDate

Spring Data JPA can automatically populate audit fields (who created a record, when it was last changed) without any manual code in your entities.

```java
// 1. Enable auditing in your Spring Boot main class or configuration
@SpringBootApplication
@EnableJpaAuditing // Activates the auditing infrastructure
public class SchoolApplication { ... }
```

```java
// 2. Create a base entity with audit fields — all entities extend this
@MappedSuperclass // This class is not an entity itself; it contributes fields to subclasses
@EntityListeners(AuditingEntityListener.class) // Hibernate calls this listener on persist/merge
public abstract class AuditableEntity {

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;    // Set automatically when entity is first persisted

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;    // Updated automatically on every merge

    @CreatedBy
    @Column(name = "created_by", updatable = false)
    private String createdBy;          // Set to the current authenticated user

    @LastModifiedBy
    @Column(name = "updated_by")
    private String updatedBy;

    // Getters only — these fields should never be set manually
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public String getCreatedBy()        { return createdBy; }
    public String getUpdatedBy()        { return updatedBy; }
}
```

```java
// 3. For @CreatedBy and @LastModifiedBy, implement AuditorAware
// This tells Spring Data how to determine the "current user"
@Component
public class SecurityAuditorAware implements AuditorAware<String> {

    @Override
    public Optional<String> getCurrentAuditor() {
        // In a real app, get the username from Spring Security:
        // return Optional.ofNullable(SecurityContextHolder.getContext()
        //     .getAuthentication()).map(Authentication::getName);
        return Optional.of("system"); // Placeholder for this example
    }
}
```

```java
// 4. Extend your entities from the base class
@Entity
@Table(name = "students")
public class Student extends AuditableEntity {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    // createdAt, updatedAt, createdBy, updatedBy are inherited — no duplication
}
```

---

## 🔑 6. Specifications — Type-Safe Dynamic Queries

When you have complex, dynamic filter conditions (e.g., a search form with many optional fields), deriving method names becomes unwieldy and `@Query` with many optional parameters gets messy. `Specification` provides a clean, composable API for dynamic queries.

```java
// Extend JpaSpecificationExecutor to get specification support
@Repository
public interface StudentRepository extends JpaRepository<Student, Long>,
                                            JpaSpecificationExecutor<Student> { }
```

```java
// Define reusable Specification building blocks
public class StudentSpecifications {

    // A Specification is a function: (Root, CriteriaQuery, CriteriaBuilder) → Predicate
    public static Specification<Student> hasName(String name) {
        return (root, query, cb) ->
            name == null ? null : cb.like(cb.lower(root.get("name")), "%" + name.toLowerCase() + "%");
    }

    public static Specification<Student> hasStatus(Student.Status status) {
        return (root, query, cb) ->
            status == null ? null : cb.equal(root.get("status"), status);
    }

    public static Specification<Student> scoreAtLeast(Double minScore) {
        return (root, query, cb) ->
            minScore == null ? null : cb.greaterThanOrEqualTo(root.get("score"), minScore);
    }

    public static Specification<Student> inDepartment(String departmentName) {
        return (root, query, cb) -> {
            if (departmentName == null) return null;
            Join<Student, Department> dept = root.join("department", JoinType.INNER);
            return cb.equal(dept.get("name"), departmentName);
        };
    }
}
```

```java
@Service
@Transactional(readOnly = true)
public class StudentSearchService {

    private final StudentRepository studentRepo;

    public StudentSearchService(StudentRepository studentRepo) {
        this.studentRepo = studentRepo;
    }

    // Compose specifications dynamically based on which filters are provided
    public Page<Student> searchStudents(String name, Student.Status status,
                                        Double minScore, String department,
                                        Pageable pageable) {

        // Specifications are combined with .and(), .or(), .not()
        // null specifications are automatically ignored by Spring Data
        Specification<Student> spec = Specification.where(null)
            .and(StudentSpecifications.hasName(name))
            .and(StudentSpecifications.hasStatus(status))
            .and(StudentSpecifications.scoreAtLeast(minScore))
            .and(StudentSpecifications.inDepartment(department));

        return studentRepo.findAll(spec, pageable);
    }
}
```

---

## 🏷️ 7. Projections — Loading Only What You Need

By default, `findAll()` loads every field of every entity. For large entities or complex entity graphs, this is wasteful. Projections let you define a "view" of an entity with only the fields you need — Spring Data generates a query that SELECTs only those columns.

```java
// Interface projection — Spring Data creates a dynamic proxy implementing this interface
public interface StudentNameAndScore {
    String getName();
    Double getScore();
    // Add @Value to compute derived values
    @Value("#{target.name + ' (' + target.score + ')'}")
    String getDisplayLabel();
}

// DTO (class) projection — uses a constructor; simpler and more explicit
public record StudentSummaryDTO(Long id, String name, Double score) {}
```

```java
@Repository
public interface StudentRepository extends JpaRepository<Student, Long> {

    // Return an interface projection — Spring generates the proxy and SELECT name, score FROM ...
    List<StudentNameAndScore> findByStatus(Student.Status status);

    // Return a DTO projection using the constructor expression approach
    @Query("SELECT new com.example.dto.StudentSummaryDTO(s.id, s.name, s.score) FROM Student s")
    List<StudentSummaryDTO> findAllSummaries();

    // Dynamic projections — same method, different return type based on the Class parameter
    <T> List<T> findByGrade(String grade, Class<T> type);
}
```

```java
// Calling with different projection types
List<StudentNameAndScore> compact = repo.findByGrade("A", StudentNameAndScore.class);
List<Student> full               = repo.findByGrade("A", Student.class);
```

---

## 🧩 8. Custom Repository Implementation

Sometimes you need complex logic that doesn't fit neatly into derived queries, `@Query`, or Specifications. You can add custom implementation by creating a fragment interface and implementation class.

```java
// Step 1: Define a custom interface with the extra methods you need
public interface StudentRepositoryCustom {
    List<Student> findStudentsWithComplexLogic(Map<String, Object> filters);
    void bulkUpdateScores(List<Long> ids, double newScore);
}
```

```java
// Step 2: Implement the interface (naming convention: <RepositoryName>Impl is auto-detected)
@Repository
public class StudentRepositoryImpl implements StudentRepositoryCustom {

    // Inject EntityManager for direct JPA access when needed
    @PersistenceContext
    private EntityManager em;

    @Override
    public List<Student> findStudentsWithComplexLogic(Map<String, Object> filters) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Student> query = cb.createQuery(Student.class);
        Root<Student> root = query.from(Student.class);
        // ... build your complex query dynamically ...
        return em.createQuery(query).getResultList();
    }

    @Override
    @Transactional
    public void bulkUpdateScores(List<Long> ids, double newScore) {
        em.createQuery("UPDATE Student s SET s.score = :score WHERE s.id IN :ids")
          .setParameter("score", newScore)
          .setParameter("ids", ids)
          .executeUpdate();
    }
}
```

```java
// Step 3: Extend both interfaces in your main repository
@Repository
public interface StudentRepository extends JpaRepository<Student, Long>,
                                            JpaSpecificationExecutor<Student>,
                                            StudentRepositoryCustom { // Add custom interface
    // Spring Data merges both interfaces — all methods available from one repository
}
```

---

## ✅ Best Practices & Important Points

**Always annotate service methods with `@Transactional`.** Spring Data repository methods are transactional by default, but only for that single repository call. If your service calls the repository multiple times (e.g., save a student, then save their courses), those should be in one transaction. Put `@Transactional` on your service methods.

**Use `@Transactional(readOnly = true)` for queries.** This is a free performance optimisation. For read-only transactions, Hibernate skips the dirty checking pass at flush time, and some databases optimise read-only transactions at the network level. Annotate all query-only service methods or a query-only service class with this.

**Don't call `findAll()` without pagination in production.** A table with 100,000 rows loaded into memory at once will cause memory pressure and very slow responses. Always add a `Pageable` parameter to any method that returns a list of entities in a production service.

**Avoid derived query methods with more than three conditions.** Method names like `findByStatusAndDepartmentNameAndScoreGreaterThanAndNameContaining()` are unreadable and fragile. Switch to `@Query` or `Specification` when conditions get complex.

**Understand what happens with `save(entity)`.** If the entity's ID is `null`, Spring Data calls `em.persist()` (INSERT). If the ID is set, it calls `em.merge()` (may trigger a SELECT before the UPDATE to load the managed entity). This means calling `save()` on a detached entity with an ID set will always fire a SELECT first. Be aware of this in batch scenarios.

**The `@Repository` annotation is optional** when extending Spring Data interfaces, because Spring Data registers the beans automatically. However, adding it provides a useful marker and ensures Spring translates JPA-specific exceptions (like `EntityNotFoundException`) into Spring's `DataAccessException` hierarchy.

---

## 🔑 Key Summary

```
JpaRepository<T,ID>   → Extend this; gets full CRUD + pagination + batch operations for free
Derived queries       → findByNameAndStatusOrderByScore() → Spring generates JPQL from the name
@Query                → Custom JPQL or native SQL when method names aren't expressive enough
@Modifying            → Required for UPDATE/DELETE @Query methods
Page<T> / Pageable    → Built-in pagination support; use PageRequest.of(page, size, sort)
Specification<T>      → Composable, type-safe dynamic queries for complex filter forms
Projections           → Interface or DTO projections to SELECT only needed columns
@EnableJpaAuditing    → Auto-populate @CreatedDate, @LastModifiedDate, @CreatedBy fields
@Transactional        → Put on service methods, not repository methods; use readOnly=true for queries
```
