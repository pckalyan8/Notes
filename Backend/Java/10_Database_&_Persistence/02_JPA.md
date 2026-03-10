# 📘 10.2 — JPA: Java Persistence API

JPA is the bridge between the object-oriented world of Java and the relational world of databases. Before JPA, developers either wrote raw JDBC (tedious, error-prone) or used proprietary ORMs like Hibernate directly (vendor lock-in). JPA standardises the ORM approach into a specification that any vendor can implement, giving you the best of both worlds: high-level abstraction with the freedom to switch implementations.

---

## 🧠 What is JPA?

**JPA (Java Persistence API)** — now officially called **Jakarta Persistence** — is a *specification* (not a library) that defines how Java objects map to relational database tables. The specification is just a set of interfaces and annotations. You need a *provider* (an actual implementation) to do the real work.

Think of JPA as a USB standard. Any USB device works in any USB port because they both follow the same specification. Similarly, any JPA provider works with any JPA-compliant code. The three main JPA providers are **Hibernate** (by far the most popular), **EclipseLink** (the reference implementation), and **OpenJPA**.

The core problem JPA solves is called the **Object-Relational Impedance Mismatch**: Java models data as objects with inheritance, relationships, and behaviour; relational databases model data as flat tables with rows, foreign keys, and joins. These two models don't naturally align. JPA provides the mapping layer that translates between them.

```
Java World          JPA Mapping         Database World
──────────────────  ────────────────    ─────────────────
Class               @Entity             Table
Field               @Column             Column
Object              Row                 Row
Reference (has-a)   @ManyToOne          Foreign Key
Collection          @OneToMany          JOIN / Junction Table
Inheritance         @Inheritance        Multiple strategies
```

---

## 📦 Maven Setup

```xml
<!-- JPA API (specification only — just interfaces and annotations) -->
<dependency>
    <groupId>jakarta.persistence</groupId>
    <artifactId>jakarta.persistence-api</artifactId>
    <version>3.1.0</version>
</dependency>

<!-- Hibernate (the implementation of the JPA specification) -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>6.4.4.Final</version>
</dependency>

<!-- PostgreSQL driver -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.7.3</version>
</dependency>
```

---

## ⚙️ Configuration — `persistence.xml`

JPA is configured via a file called `persistence.xml` located at `src/main/resources/META-INF/persistence.xml`. This file defines one or more **persistence units** — named configurations that bundle together your entity classes, database connection settings, and provider options.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence xmlns="https://jakarta.ee/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="https://jakarta.ee/xml/ns/persistence
                 https://jakarta.ee/xml/ns/persistence/persistence_3_1.xsd"
             version="3.1">

    <!-- A persistence unit is like a named JPA configuration profile -->
    <persistence-unit name="schoolPU" transaction-type="RESOURCE_LOCAL">

        <!-- The JPA provider — Hibernate in this case -->
        <provider>org.hibernate.jpa.HibernatePersistenceProvider</provider>

        <!-- List all @Entity classes here (or let Hibernate scan automatically) -->
        <class>com.example.entity.Student</class>
        <class>com.example.entity.Course</class>

        <properties>
            <!-- Database connection -->
            <property name="jakarta.persistence.jdbc.url"      value="jdbc:postgresql://localhost:5432/school_db"/>
            <property name="jakarta.persistence.jdbc.user"     value="postgres"/>
            <property name="jakarta.persistence.jdbc.password" value="secret"/>
            <property name="jakarta.persistence.jdbc.driver"   value="org.postgresql.Driver"/>

            <!-- Schema management:
                 none     — do nothing (production default)
                 validate — verify the schema matches your entities (good for CI)
                 update   — add missing columns/tables (risky in production)
                 create   — recreate schema every start (development only)
                 create-drop — create on start, drop on stop (testing only)     -->
            <property name="jakarta.persistence.schema-generation.database.action" value="create-drop"/>

            <!-- Hibernate-specific options -->
            <property name="hibernate.show_sql"     value="true"/>   <!-- Print SQL to console -->
            <property name="hibernate.format_sql"   value="true"/>   <!-- Pretty-print the SQL -->
            <property name="hibernate.use_sql_comments" value="true"/> <!-- Add comments to SQL -->
            <property name="hibernate.dialect"      value="org.hibernate.dialect.PostgreSQLDialect"/>
        </properties>
    </persistence-unit>
</persistence>
```

---

## 🗃️ 1. Entity Mapping — The Core Annotations

An **entity** is a Java class that maps to a database table. Each instance of the entity represents one row. You mark a class as an entity with `@Entity`.

### Basic Entity with Column Mapping

```java
package com.example.entity;

import jakarta.persistence.*;
import java.time.LocalDate;

// @Entity marks this class as a JPA entity — it will map to a database table
@Entity
// @Table lets you customise the table name and add constraints
@Table(
    name = "students",
    uniqueConstraints = {
        @UniqueConstraint(name = "uq_student_email", columnNames = {"email"})
    }
)
public class Student {

    // @Id marks the primary key field
    @Id
    // @GeneratedValue defines the strategy for auto-generating the primary key:
    //   IDENTITY  — database auto-increment (MySQL: AUTO_INCREMENT, PostgreSQL: SERIAL)
    //   SEQUENCE  — uses a database sequence (preferred for PostgreSQL)
    //   TABLE     — simulates a sequence using a separate table (slow, avoid)
    //   AUTO      — JPA picks what it thinks is best (usually SEQUENCE)
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // @Column lets you customise the column name, length, nullability, and precision
    @Column(name = "full_name", length = 150, nullable = false)
    private String name;

    // If you omit @Column, JPA maps the field to a column with the same name
    // Field "email" → Column "email"
    @Column(unique = true, nullable = false)
    private String email;

    @Column(name = "date_of_birth")
    private LocalDate dateOfBirth;

    @Column(precision = 5, scale = 2) // e.g., 100.00 — 5 total digits, 2 decimal places
    private Double score;

    // @Transient tells JPA to completely ignore this field — it won't be persisted
    @Transient
    private String temporaryNote;

    // @Enumerated maps a Java enum to the database
    // EnumType.ORDINAL stores the index (0, 1, 2) — FRAGILE: adding enum values can break it
    // EnumType.STRING stores the name ("ACTIVE", "INACTIVE") — ALWAYS use this
    @Enumerated(EnumType.STRING)
    @Column(length = 20)
    private Status status;

    // @Lob marks a field as a Large Object — stored as TEXT or BLOB/CLOB in the DB
    @Lob
    @Column(name = "bio")
    private String biography;

    public enum Status { ENROLLED, GRADUATED, WITHDRAWN }

    // JPA requires a no-arg constructor (can be protected, doesn't need to be public)
    protected Student() {}

    public Student(String name, String email) {
        this.name   = name;
        this.email  = email;
        this.status = Status.ENROLLED;
    }

    // Getters and setters...
    public Long getId()        { return id; }
    public String getName()    { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail()   { return email; }
    public Double getScore()   { return score; }
    public void setScore(Double score) { this.score = score; }
    public Status getStatus()  { return status; }
    public void setStatus(Status status) { this.status = status; }

    @Override
    public String toString() {
        return "Student{id=" + id + ", name='" + name + "', score=" + score + "}";
    }
}
```

---

## 🔗 2. Relationships Between Entities

Relationships are the most nuanced part of JPA. They map the "has-a" relationships in your Java model to foreign keys and join tables in the database.

### @ManyToOne and @OneToMany — The Most Common Relationship

The "many" side of the relationship owns the foreign key column. Always define `@ManyToOne` on the side that holds the foreign key column.

```java
// Department entity — the "one" side
@Entity
@Table(name = "departments")
public class Department {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String name;

    // @OneToMany — one Department has many Students
    // mappedBy = "department" refers to the FIELD NAME in Student that owns the relationship
    // Without mappedBy, JPA would create a separate junction table — not what we want here
    // cascade = ALL means: when you save/delete a Department, also save/delete its Students
    // orphanRemoval = true means: if a Student is removed from this list, delete it from DB
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Student> students = new ArrayList<>();

    // Helper methods to keep both sides of the bidirectional relationship in sync
    public void addStudent(Student student) {
        students.add(student);
        student.setDepartment(this); // Update the owning (foreign key) side
    }

    public void removeStudent(Student student) {
        students.remove(student);
        student.setDepartment(null);
    }

    protected Department() {}
    public Department(String name) { this.name = name; }
    public Long getId()            { return id; }
    public String getName()        { return name; }
    public List<Student> getStudents() { return Collections.unmodifiableList(students); }
}
```

```java
// Update the Student entity to include the relationship
@Entity
@Table(name = "students")
public class Student {

    // ... (id, name, email, score fields from before) ...

    // @ManyToOne — many Students belong to one Department
    // This side owns the foreign key column in the database table
    // FetchType.LAZY means: don't load the Department until you access student.getDepartment()
    // This avoids unnecessary database queries when you only need the student's own fields
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id") // Name of the foreign key column in the students table
    private Department department;

    public Department getDepartment() { return department; }
    public void setDepartment(Department department) { this.department = department; }
}
```

### @OneToOne — One-to-One Relationship

```java
@Entity
@Table(name = "student_profiles")
public class StudentProfile {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String address;
    private String phoneNumber;

    // The @OneToOne side that holds the foreign key (owns the relationship)
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "student_id", unique = true)
    private Student student;

    // Getters, setters, constructors...
}
```

### @ManyToMany — Many-to-Many Relationship

```java
@Entity
@Table(name = "courses")
public class Course {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    // @ManyToMany — one Course has many Students; one Student takes many Courses
    // @JoinTable defines the junction table that holds the pairs of foreign keys
    @ManyToMany
    @JoinTable(
        name = "student_course",                         // Junction table name
        joinColumns = @JoinColumn(name = "course_id"),   // FK to this entity (Course)
        inverseJoinColumns = @JoinColumn(name = "student_id") // FK to the other entity (Student)
    )
    private Set<Student> students = new HashSet<>();

    protected Course() {}
    public Course(String title) { this.title = title; }
    public Long getId()         { return id; }
    public String getTitle()    { return title; }
    public Set<Student> getStudents() { return students; }
}
```

> 💡 **Use `Set` for `@ManyToMany` collections, not `List`.** If you use `List` and Hibernate decides to delete and re-insert the join table rows when the collection changes, you can get an unintended full delete followed by re-inserts — a classic Hibernate pitfall. `Set` avoids duplicate semantics and often performs better.

---

## 🔄 3. Fetch Types — LAZY vs EAGER

This is one of the most important concepts in JPA, and getting it wrong causes the dreaded **N+1 query problem**.

```java
// EAGER loading — load the related entity immediately in the same query
// Use EAGER only when you ALWAYS need the related data
@ManyToOne(fetch = FetchType.EAGER)
@JoinColumn(name = "department_id")
private Department department; // SQL: SELECT s.*, d.* FROM students s LEFT JOIN departments d ...

// LAZY loading — load the related entity only when you access it (default for @OneToMany)
// This generates a second SQL query at the moment you call student.getDepartment()
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "department_id")
private Department department; // SQL: SELECT * FROM students WHERE ... (no JOIN yet)
                                // Then: SELECT * FROM departments WHERE id = ? (when accessed)
```

**Default fetch types:** `@ManyToOne` and `@OneToOne` default to `EAGER`. `@OneToMany` and `@ManyToMany` default to `LAZY`. A widely accepted best practice is to **always use `LAZY` everywhere** and use JOIN FETCH in your JPQL queries when you know you need the relationship — this gives you explicit control over when JOINs happen.

---

## 💼 4. EntityManager — The Core JPA API

The `EntityManager` is the primary interface for interacting with JPA. Every CRUD operation goes through it. It represents a JPA session (conceptually similar to a JDBC `Connection`).

```java
public class StudentRepository {

    private EntityManagerFactory emf;

    public StudentRepository() {
        // The persistence unit name must match what's in persistence.xml
        this.emf = Persistence.createEntityManagerFactory("schoolPU");
    }

    // ──────────────────────── CREATE ────────────────────────

    public Student save(Student student) {
        EntityManager em = emf.createEntityManager();
        try {
            em.getTransaction().begin(); // Start a transaction

            // persist() makes the entity MANAGED and schedules an INSERT
            // The INSERT is executed when the transaction commits (or on flush)
            em.persist(student);

            em.getTransaction().commit();
            return student; // The entity now has its generated ID set
        } catch (Exception e) {
            em.getTransaction().rollback();
            throw e;
        } finally {
            em.close();
        }
    }

    // ──────────────────────── READ ────────────────────────

    public Optional<Student> findById(Long id) {
        EntityManager em = emf.createEntityManager();
        try {
            // find() returns null if not found (does NOT throw an exception)
            // It checks the first-level cache before hitting the database
            Student student = em.find(Student.class, id);
            return Optional.ofNullable(student);
        } finally {
            em.close();
        }
    }

    public Student getReference(Long id) {
        EntityManager em = emf.createEntityManager();
        try {
            // getReference() returns a proxy — no SQL is fired until a field is accessed
            // Useful when you only need the ID (e.g., for setting a foreign key)
            // Throws EntityNotFoundException if the ID doesn't exist (when accessed)
            return em.getReference(Student.class, id);
        } finally {
            em.close();
        }
    }

    // ──────────────────────── UPDATE ────────────────────────

    public Student update(Student detachedStudent) {
        EntityManager em = emf.createEntityManager();
        try {
            em.getTransaction().begin();

            // merge() handles a "detached" entity (one obtained outside this EntityManager's scope)
            // It copies the state of the detached entity into a new managed entity and returns it
            // The returned entity is managed; the argument is NOT managed after merge()
            Student managedStudent = em.merge(detachedStudent);

            em.getTransaction().commit();
            return managedStudent;
        } catch (Exception e) {
            em.getTransaction().rollback();
            throw e;
        } finally {
            em.close();
        }
    }

    public void updateName(Long id, String newName) {
        EntityManager em = emf.createEntityManager();
        try {
            em.getTransaction().begin();

            // Alternative update pattern: find → modify → let dirty checking handle the UPDATE
            // Hibernate's "dirty checking" detects field changes and generates UPDATE automatically
            Student student = em.find(Student.class, id);
            if (student != null) {
                student.setName(newName); // Just modify the managed entity — no need to call update()
            }

            em.getTransaction().commit(); // Hibernate generates UPDATE at commit time
        } catch (Exception e) {
            em.getTransaction().rollback();
            throw e;
        } finally {
            em.close();
        }
    }

    // ──────────────────────── DELETE ────────────────────────

    public void delete(Long id) {
        EntityManager em = emf.createEntityManager();
        try {
            em.getTransaction().begin();

            Student student = em.find(Student.class, id);
            if (student != null) {
                // remove() schedules a DELETE — the entity must be managed (in this EntityManager)
                em.remove(student);
            }

            em.getTransaction().commit();
        } catch (Exception e) {
            em.getTransaction().rollback();
            throw e;
        } finally {
            em.close();
        }
    }

    public void close() {
        emf.close();
    }
}
```

### Entity Lifecycle States

Understanding the four lifecycle states is crucial to understanding JPA behaviour:

```
            new Student()
                 │
                 ▼
         ┌──────────────┐
         │   TRANSIENT  │  new object, JPA knows nothing about it
         └──────┬───────┘
                │ em.persist()
                ▼
         ┌──────────────┐
         │   MANAGED    │  JPA tracks changes; changes auto-synced to DB on commit
         └──────┬───────┘
                │ em.close() / em.detach() / em.clear()
                ▼
         ┌──────────────┐
         │   DETACHED   │  JPA no longer tracks changes; use em.merge() to re-attach
         └──────┬───────┘
                │ em.remove() (while managed)
                ▼
         ┌──────────────┐
         │    REMOVED   │  scheduled for deletion at commit
         └──────────────┘
```

---

## 📝 5. JPQL — Java Persistence Query Language

JPQL is a query language that looks like SQL but operates on **entity classes and fields** (not table names and column names). Hibernate translates JPQL to the appropriate database-specific SQL at runtime.

```java
public class JpqlExamples {

    private EntityManager em; // Assume injected or created elsewhere

    // Basic JPQL — uses class name "Student" not table name "students"
    public List<Student> findAll() {
        // TypedQuery is the generic, type-safe way to execute queries
        TypedQuery<Student> query = em.createQuery(
            "SELECT s FROM Student s", Student.class
        );
        return query.getResultList();
    }

    // WHERE clause with named parameter (:name)
    public Optional<Student> findByName(String name) {
        TypedQuery<Student> query = em.createQuery(
            "SELECT s FROM Student s WHERE s.name = :studentName", Student.class
        );
        query.setParameter("studentName", name); // Bind the parameter by name
        try {
            return Optional.of(query.getSingleResult()); // Throws if 0 or >1 result
        } catch (NoResultException e) {
            return Optional.empty();
        }
    }

    // WHERE clause with positional parameter (?1)
    public List<Student> findByStatus(Student.Status status) {
        return em.createQuery(
            "SELECT s FROM Student s WHERE s.status = ?1", Student.class
        ).setParameter(1, status).getResultList();
    }

    // Navigating relationships using dot notation
    public List<Student> findByDepartmentName(String deptName) {
        return em.createQuery(
            // Hibernate translates this navigation into a JOIN automatically
            "SELECT s FROM Student s WHERE s.department.name = :deptName",
            Student.class
        ).setParameter("deptName", deptName).getResultList();
    }

    // JOIN FETCH — eagerly load a lazy relationship in a single query (solves N+1)
    public List<Student> findAllWithDepartment() {
        return em.createQuery(
            // Without JOIN FETCH: 1 query for all students + N queries for each department
            // With JOIN FETCH: 1 query with a JOIN — much more efficient
            "SELECT s FROM Student s JOIN FETCH s.department",
            Student.class
        ).getResultList();
    }

    // Projection — select only specific fields using constructor expression
    public List<StudentSummaryDTO> findSummaries() {
        return em.createQuery(
            // Creates StudentSummaryDTO objects using its constructor — avoids loading full entities
            "SELECT new com.example.dto.StudentSummaryDTO(s.id, s.name, s.score) FROM Student s",
            StudentSummaryDTO.class
        ).getResultList();
    }

    // Aggregation functions
    public Double findAverageScore() {
        return em.createQuery(
            "SELECT AVG(s.score) FROM Student s WHERE s.status = :status",
            Double.class
        ).setParameter("status", Student.Status.ENROLLED)
         .getSingleResult();
    }

    // Grouping
    public List<Object[]> countByDepartment() {
        return em.createQuery(
            "SELECT s.department.name, COUNT(s) FROM Student s GROUP BY s.department.name",
            Object[].class
        ).getResultList();
    }

    // Pagination — essential for large result sets
    public List<Student> findPage(int pageNumber, int pageSize) {
        return em.createQuery("SELECT s FROM Student s ORDER BY s.name", Student.class)
            .setFirstResult(pageNumber * pageSize) // Offset (0-based)
            .setMaxResults(pageSize)               // LIMIT
            .getResultList();
    }

    // Bulk UPDATE — more efficient than loading entities one-by-one for mass updates
    public int updateStatusForGraduates(double minScore) {
        // Note: Bulk updates bypass the first-level cache — call em.clear() after
        int updatedCount = em.createQuery(
            "UPDATE Student s SET s.status = :newStatus WHERE s.score >= :minScore"
        ).setParameter("newStatus", Student.Status.GRADUATED)
         .setParameter("minScore", minScore)
         .executeUpdate();
        em.clear(); // Sync the cache with the updated database state
        return updatedCount;
    }

    // Bulk DELETE
    public int deleteWithdrawnStudents() {
        return em.createQuery(
            "DELETE FROM Student s WHERE s.status = :status"
        ).setParameter("status", Student.Status.WITHDRAWN)
         .executeUpdate();
    }
}
```

### Named Queries

Named queries are defined on the entity class at compile time, making them reusable and easy to find. They are also validated when the `EntityManagerFactory` is created — giving you early errors instead of runtime failures.

```java
@Entity
@Table(name = "students")
// Define named queries at the class level
@NamedQuery(
    name = "Student.findByStatus",
    query = "SELECT s FROM Student s WHERE s.status = :status ORDER BY s.name"
)
@NamedQueries({
    @NamedQuery(name = "Student.findAll",        query = "SELECT s FROM Student s"),
    @NamedQuery(name = "Student.countEnrolled",
                query = "SELECT COUNT(s) FROM Student s WHERE s.status = 'ENROLLED'")
})
public class Student {
    // ...
}

// Usage:
List<Student> enrolled = em
    .createNamedQuery("Student.findByStatus", Student.class)
    .setParameter("status", Student.Status.ENROLLED)
    .getResultList();
```

---

## 🔍 6. Criteria API — Type-Safe Dynamic Queries

The Criteria API lets you build queries programmatically using Java code instead of JPQL strings. It is verbose but **type-safe** (compiler catches typos in field names) and ideal for **dynamic queries** where the filter conditions change based on runtime input.

```java
public List<Student> findStudents(String nameFilter, Double minScore, Student.Status status) {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<Student> cq = cb.createQuery(Student.class);
    Root<Student> root = cq.from(Student.class); // FROM Student s

    // Build a list of predicates (WHERE conditions) dynamically
    List<Predicate> predicates = new ArrayList<>();

    // Only add a condition if the parameter is provided
    if (nameFilter != null && !nameFilter.isBlank()) {
        predicates.add(cb.like(
            cb.lower(root.get("name")),     // LOWER(s.name)
            "%" + nameFilter.toLowerCase() + "%" // LIKE '%alice%'
        ));
    }

    if (minScore != null) {
        predicates.add(cb.greaterThanOrEqualTo(root.get("score"), minScore));
    }

    if (status != null) {
        predicates.add(cb.equal(root.get("status"), status));
    }

    // Combine all predicates with AND
    cq.where(cb.and(predicates.toArray(new Predicate[0])));
    cq.orderBy(cb.asc(root.get("name")));

    return em.createQuery(cq).getResultList();
}
```

---

## 🧬 7. Inheritance Mapping

JPA supports mapping Java inheritance hierarchies to the database. There are three strategies, each with different trade-offs.

```java
// Base class (abstract or concrete)
@Entity
@Inheritance(strategy = InheritanceType.JOINED) // See strategy options below
@DiscriminatorColumn(name = "person_type")      // Used with SINGLE_TABLE
public abstract class Person {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String email;
}

@Entity
@DiscriminatorValue("STUDENT")
public class Student extends Person {
    private Double gpa;
}

@Entity
@DiscriminatorValue("TEACHER")
public class Teacher extends Person {
    private String subject;
    private Double salary;
}
```

**SINGLE_TABLE** puts all subclasses in one table with a discriminator column. This is the fastest strategy (no joins) but the table will have nullable columns for fields that belong to only one subclass. Use it when the subclasses are similar or performance is critical.

**JOINED** gives each class its own table, with subclass tables joining back to the parent table via a foreign key. This is the cleanest model (no nullable columns, fully normalised) but requires a JOIN for every query.

**TABLE_PER_CLASS** gives each concrete class its own complete table with all inherited fields duplicated. This avoids joins for single-class queries but makes polymorphic queries (`SELECT p FROM Person p`) expensive because the database must UNION all subclass tables.

---

## ✅ Best Practices & Important Points

**Always use `Long` (wrapper type) for IDs, not `long`.** The primitive `long` cannot be `null`, which makes it impossible to distinguish "entity has no ID yet" from "entity has ID = 0". JPA uses `null` to determine whether `persist()` is needed.

**Implement `equals()` and `hashCode()` correctly.** If you use entities in `Set` collections or as Map keys (required for `@ManyToMany`), these methods must be consistent. A common approach is to base them on the natural key (business identifier like `email`) rather than the database-generated `id`, since the `id` is `null` before the entity is persisted.

**Never expose the mutable collection directly in `@OneToMany`.** Return `Collections.unmodifiableList(students)` from the getter and provide explicit `addStudent()`/`removeStudent()` helper methods. This protects invariants and keeps the bidirectional relationship in sync.

**Use `@Transactional` boundaries wisely.** The `EntityManager` must be open while accessing lazy-loaded relationships. Accessing a lazy collection outside the transaction scope causes a `LazyInitializationException`. The solution is either to use JOIN FETCH in your query or to use Spring's `@Transactional` to keep the session open for the duration of the operation.

**Avoid `CascadeType.ALL` blindly.** Cascade delete is especially dangerous. If you cascade `REMOVE` from a parent to children, deleting a `Department` will delete all its `Student` records. Make sure this is the intended behaviour before enabling it.

**Use DTOs (Data Transfer Objects) for API responses.** Loading full entity graphs and serialising them directly to JSON can cause circular reference issues (A→B→A) and accidentally expose fields that shouldn't be public. Use `SELECT new DTO(...)` in JPQL or a mapping library like MapStruct to create purpose-built response objects.

---

## 🔑 Key Summary

```
@Entity            → Maps a class to a table
@Id                → Marks the primary key
@GeneratedValue    → Defines how the primary key is auto-generated
@Column            → Customises column mapping (name, length, nullable)
@Transient         → Exclude a field from persistence
@ManyToOne         → N:1 relationship; holds the foreign key
@OneToMany         → 1:N relationship; mappedBy points to the owning side
@ManyToMany        → N:M relationship; use @JoinTable for the junction table
FetchType.LAZY     → Load related entity only when accessed (preferred default)
FetchType.EAGER    → Load related entity immediately with a JOIN
EntityManager      → Core API: persist(), find(), merge(), remove()
JPQL               → Object-oriented query language; uses class/field names not table/column names
Criteria API       → Type-safe programmatic queries; ideal for dynamic filters
```
