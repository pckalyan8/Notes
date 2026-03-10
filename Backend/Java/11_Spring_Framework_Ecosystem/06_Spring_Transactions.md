# 📕 11.6 — Spring Transaction Management

Transactions are the mechanism that keeps your database consistent. When your business logic requires multiple database operations to succeed or fail as a unit, transactions are what makes that guarantee. Spring's transaction management abstracts away the raw JDBC transaction API and replaces it with a single, elegant annotation: `@Transactional`.

---

## 🧠 Why Transactions Matter

Imagine an online enrollment system where a student clicks "Enroll in Course." This action requires three database operations: deduct one seat from the course's available capacity, create an enrollment record, and create an invoice for the tuition fee. If step two succeeds but step three fails (the invoice table is locked by another process), the seat has been deducted but no invoice was created. The data is now inconsistent — a seat is gone but no student is enrolled and no money will be collected.

A transaction wraps all three operations in a single atomic unit. If any step fails, the entire transaction rolls back — all three operations are undone as if none of them happened. The seat count goes back, no enrollment record exists, and no invoice was attempted. The database is left in a consistent state.

This is the **ACID** guarantee that transactions provide: **A**tomicity (all or nothing), **C**onsistency (data integrity is preserved), **I**solation (concurrent transactions don't interfere), **D**urability (committed data survives crashes).

---

## ⚙️ How Spring Transactions Work

Spring's `@Transactional` is powered by AOP. When you annotate a method with `@Transactional`, Spring generates a proxy around the bean. When a caller invokes that method, they go through the proxy, which:

1. Checks if there is already an active transaction in the current thread (via `TransactionSynchronizationManager`)
2. If not, starts a new one (calls `connection.setAutoCommit(false)` under the hood)
3. Invokes your actual method
4. If the method returns normally: commits the transaction
5. If the method throws a `RuntimeException` or `Error`: rolls back the transaction

```
Proxy (Spring's AOP wrapper)
┌────────────────────────────────────────────────────────────────┐
│  1. Begin transaction (if needed based on propagation)         │
│  2. Bind connection to current thread                          │
│  3. Call your actual method ─────────────────────────────────► │
│  4a. On success → commit; release connection                   │
│  4b. On RuntimeException → rollback; release connection        │
└────────────────────────────────────────────────────────────────┘
```

Because Spring transactions are proxy-based, the **self-invocation problem from the AOP chapter applies here too**. If one method in your service calls another `@Transactional` method in the same service (without going through the proxy), the inner method's transaction annotation is completely ignored.

---

## 🔧 Basic @Transactional Usage

```java
@Service
@Transactional(readOnly = true) // Default for all methods in this class: read-only transaction
public class StudentService {

    private final StudentRepository studentRepo;
    private final EnrollmentRepository enrollmentRepo;
    private final InvoiceRepository invoiceRepo;
    private final CourseRepository courseRepo;

    // Constructor injection omitted for brevity

    // Read methods inherit readOnly = true from the class-level annotation
    // Read-only transactions skip Hibernate's dirty checking pass — performance optimization
    public List<StudentDTO> findAll() {
        return studentRepo.findAll().stream().map(StudentDTO::from).toList();
    }

    public Optional<StudentDTO> findById(Long id) {
        return studentRepo.findById(id).map(StudentDTO::from);
    }

    // WRITE operations MUST override readOnly and use a full (read-write) transaction
    @Transactional // Overrides the class-level readOnly = true
    public EnrollmentResult enrollStudent(Long studentId, Long courseId) {
        Student student = studentRepo.findById(studentId)
            .orElseThrow(() -> new ResourceNotFoundException("Student not found: " + studentId));

        Course course = courseRepo.findById(courseId)
            .orElseThrow(() -> new ResourceNotFoundException("Course not found: " + courseId));

        // Business rule validation
        if (course.getAvailableSeats() <= 0) {
            throw new NoSeatsAvailableException("Course " + courseId + " is full");
        }

        // All three operations are in the same transaction
        // If any throws, ALL changes are rolled back
        course.setAvailableSeats(course.getAvailableSeats() - 1); // Update seat count
        courseRepo.save(course);

        Enrollment enrollment = new Enrollment(student, course, LocalDate.now());
        enrollmentRepo.save(enrollment);

        Invoice invoice = new Invoice(student, course, course.getTuitionFee());
        invoiceRepo.save(invoice); // If this fails, the two saves above also roll back

        return new EnrollmentResult(enrollment.getId(), invoice.getId());
    }

    @Transactional
    public void deleteStudent(Long id) {
        Student student = studentRepo.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Student not found: " + id));
        enrollmentRepo.deleteByStudentId(id);   // Delete related records first
        invoiceRepo.deleteByStudentId(id);
        studentRepo.delete(student);             // Then delete the student
    }
}
```

---

## 🔀 Transaction Propagation — The Most Important Setting

Propagation controls what happens when a transactional method is called from within another transactional method. This is the most nuanced and most important aspect of Spring transactions.

There are seven propagation levels. The two you use in 95% of cases are `REQUIRED` and `REQUIRES_NEW`.

### REQUIRED (The Default)

"Use the existing transaction if one exists; otherwise start a new one."

```java
@Transactional(propagation = Propagation.REQUIRED) // Default — same as @Transactional
public void outerOperation() {
    // A new transaction T1 is started here

    innerOperation(); // Joins T1 — does NOT start a new transaction

    // If innerOperation() throws, T1 rolls back (both operations undone)
    // If this method throws, T1 rolls back
}

@Transactional(propagation = Propagation.REQUIRED)
public void innerOperation() {
    // Runs INSIDE T1 (the caller's transaction)
    // innerOperation cannot independently commit or roll back
    // It rises or falls with the outer transaction
}
```

The practical implication: when `innerOperation()` throws a `RuntimeException`, it marks the existing transaction for rollback. Even if the caller catches that exception, when the outer transaction tries to commit, Spring will throw `UnexpectedRollbackException` because the transaction was already marked as "rollback-only". This surprises many developers.

### REQUIRES_NEW — A Truly Independent Transaction

```java
@Transactional
public void processOrder(Order order) {
    // Transaction T1 starts here

    orderRepository.save(order);
    auditService.logOrderCreation(order); // Uses REQUIRES_NEW — runs in T2

    // Even if T1 rolls back later, the audit log entry in T2 is already committed
}

@Service
public class AuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logOrderCreation(Order order) {
        // T1 is SUSPENDED; a brand new transaction T2 starts here
        // T2 commits independently — audit log is persisted even if T1 rolls back
        // After this method returns, T2 is committed and T1 resumes
        AuditLog log = new AuditLog("ORDER_CREATED", order.getId(), LocalDateTime.now());
        auditLogRepository.save(log);
    }
}
```

`REQUIRES_NEW` is the right choice when you want an operation to persist regardless of what happens in the calling transaction. Audit logging and notification sending are classic examples — you want to record that an attempt was made even if the business operation ultimately fails.

### Other Propagation Levels (Know When to Use Them)

`NESTED` runs a nested transaction within the outer transaction using database savepoints. If the nested transaction rolls back, it rolls back only to the savepoint — the outer transaction can choose to continue. Use this for operations that should be "best effort" within a larger transaction.

`SUPPORTS` participates in an existing transaction if one exists; runs non-transactionally if none exists. Use for read operations that can work with or without a transaction.

`NOT_SUPPORTED` always runs non-transactionally, suspending any existing transaction. Rarely used.

`MANDATORY` requires an existing transaction to be active; throws an exception if none exists. Use this to document that a method must be called within a transaction (architectural enforcement).

`NEVER` refuses to run if a transaction is active; throws an exception if one exists. Extremely rare.

```java
// A reference table of when to use each propagation level
@Service
public class PropagationExamples {

    // Most common: join existing transaction or start a new one
    @Transactional(propagation = Propagation.REQUIRED)
    public void normalBusinessOperation() { }

    // Independent transaction: audit logs, notifications that must persist regardless of outer tx
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void auditOrNotify() { }

    // Nested: best-effort sub-operations within a larger transaction
    @Transactional(propagation = Propagation.NESTED)
    public void bestEffortSubOperation() { }

    // Optional transaction: can run inside or outside a transaction
    @Transactional(propagation = Propagation.SUPPORTS)
    public void optionalTransactionReads() { }

    // Enforce that caller must provide a transaction
    @Transactional(propagation = Propagation.MANDATORY)
    public void mustBeCalledInTransaction() { }
}
```

---

## 🔒 Isolation Levels — Handling Concurrency

Isolation controls how much one transaction can see of another's uncommitted work. Higher isolation prevents more concurrency anomalies but reduces throughput because more locking is required.

The four concurrency problems that isolation levels address are:

**Dirty Read**: Transaction A reads data that Transaction B has written but not yet committed. If B rolls back, A has read data that never truly existed.

**Non-Repeatable Read**: Transaction A reads a row, Transaction B updates and commits that row, then Transaction A reads the same row again and gets different data within the same transaction.

**Phantom Read**: Transaction A queries a range of rows, Transaction B inserts a new row in that range and commits, then Transaction A re-queries and gets an extra row that wasn't there before.

**Lost Update**: Transactions A and B both read the same row, both modify it based on what they read, both write it back — B's write overwrites A's changes.

```java
@Service
public class IsolationExamples {

    // READ_UNCOMMITTED: allows all anomalies — almost never use this
    // Possible: dirty reads, non-repeatable reads, phantom reads
    @Transactional(isolation = Isolation.READ_UNCOMMITTED)
    public List<Product> getLiveInventory() { /* dangerous */ return null; }

    // READ_COMMITTED: default in PostgreSQL
    // Prevents: dirty reads
    // Possible: non-repeatable reads, phantom reads
    @Transactional(isolation = Isolation.READ_COMMITTED)
    public ReportData generateDailyReport() {
        // Two reads of the same row within this transaction might return different values
        // because other transactions can commit between those reads
        return null;
    }

    // REPEATABLE_READ: default in MySQL
    // Prevents: dirty reads, non-repeatable reads
    // Possible: phantom reads
    @Transactional(isolation = Isolation.REPEATABLE_READ)
    public boolean checkAndReserveItem(Long itemId) {
        // This transaction will always see the same value for a row it has already read
        // Another transaction cannot modify that row until this one completes
        return false;
    }

    // SERIALIZABLE: the strongest isolation; prevents all anomalies
    // Transactions execute as if they were serial (one after another)
    // Use for financial calculations, seat reservation, inventory deduction
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void deductInventory(Long productId, int quantity) {
        Product product = productRepo.findById(productId).orElseThrow();
        if (product.getStock() < quantity) throw new InsufficientStockException();
        product.setStock(product.getStock() - quantity);
        productRepo.save(product);
    }

    // DEFAULT: uses the database's configured default (usually READ_COMMITTED)
    @Transactional(isolation = Isolation.DEFAULT)
    public void normalOperation() { /* uses PostgreSQL's READ_COMMITTED by default */ }
}
```

---

## ↩️ Rollback Rules — What Triggers a Rollback

By default, Spring only rolls back on `RuntimeException` (unchecked exceptions) and `Error`. Checked exceptions do NOT trigger a rollback. This is by design — it mirrors the convention that checked exceptions represent expected error conditions that the application can potentially recover from.

```java
@Service
public class RollbackExamples {

    // Default behavior: rolls back on RuntimeException and Error only
    @Transactional
    public void defaultRollback() throws IOException {
        studentRepo.save(new Student("Alice"));
        // If IOException is thrown (a CHECKED exception), the transaction COMMITS!
        // Only RuntimeException and Error cause automatic rollback
        throw new IOException("File not found"); // ← COMMITS the save above!
    }

    // Explicit rollback for a checked exception
    @Transactional(rollbackFor = {IOException.class, ParseException.class})
    public void rollbackOnCheckedException() throws IOException {
        studentRepo.save(new Student("Alice"));
        throw new IOException("This WILL cause a rollback now");
    }

    // Prevent rollback even for RuntimeException (very unusual use case)
    // Use when you want to commit the work done so far even if a non-critical exception occurs
    @Transactional(noRollbackFor = IllegalArgumentException.class)
    public void noRollbackForSpecificException() {
        studentRepo.save(new Student("Alice")); // This WILL be saved even if the below throws
        processOptionalData(); // May throw IllegalArgumentException but we still want to commit
    }

    // Combined: roll back on all exceptions including checked ones
    @Transactional(rollbackFor = Exception.class)
    public void rollbackOnAll() throws Exception {
        // Any exception, checked or unchecked, causes a rollback
    }
}
```

---

## ⚡ Programmatic Transaction Management

Sometimes you need finer-grained control than `@Transactional` provides — for example, if you need to commit partway through a method, or if you're in a context where annotations don't work (e.g., dynamic proxies that bypass AOP). `TransactionTemplate` gives you programmatic transactions.

```java
@Service
public class BulkImportService {

    private final TransactionTemplate transactionTemplate;
    private final StudentRepository studentRepo;

    public BulkImportService(PlatformTransactionManager transactionManager,
                              StudentRepository studentRepo) {
        // Configure a reusable transaction template
        this.transactionTemplate = new TransactionTemplate(transactionManager);
        this.transactionTemplate.setPropagationBehavior(
            TransactionDefinition.PROPAGATION_REQUIRES_NEW);
        this.transactionTemplate.setTimeout(30); // 30 second timeout
        this.studentRepo = studentRepo;
    }

    // Import students in batches; each batch commits independently
    // If batch 3 fails, batches 1 and 2 are already committed — partial progress is kept
    public ImportResult importStudents(List<StudentImportRow> rows) {
        int successCount = 0;
        List<String> errors = new ArrayList<>();
        int batchSize = 100;

        for (int i = 0; i < rows.size(); i += batchSize) {
            List<StudentImportRow> batch = rows.subList(i, Math.min(i + batchSize, rows.size()));
            final int batchIndex = i / batchSize;

            try {
                int saved = transactionTemplate.execute(status -> {
                    // Everything inside execute() runs in a transaction
                    int count = 0;
                    for (StudentImportRow row : batch) {
                        try {
                            studentRepo.save(row.toEntity());
                            count++;
                        } catch (DataIntegrityViolationException e) {
                            // Can programmatically mark for rollback
                            status.setRollbackOnly();
                            return 0;
                        }
                    }
                    return count;
                });
                successCount += saved;
            } catch (Exception e) {
                errors.add("Batch " + batchIndex + " failed: " + e.getMessage());
            }
        }

        return new ImportResult(successCount, errors);
    }
}
```

---

## 🔁 @Transactional on Repository vs Service Layer

A common point of confusion is where `@Transactional` should be placed. The answer is: **always on your service layer, not on your repository layer**.

Spring Data JPA repository methods already have `@Transactional` on each individual method — so `studentRepo.save(s)` is always in a transaction. But if your service calls `studentRepo.save(s)` and then `invoiceRepo.save(i)`, those run in two separate transactions. If the second fails, the first has already committed and cannot be rolled back.

Placing `@Transactional` on your service method ensures that both repository calls participate in the same transaction:

```java
// ❌ Wrong: each repository call is its own transaction
@Service
public class EnrollmentService {
    public void enroll(Long studentId, Long courseId) {
        enrollmentRepo.save(new Enrollment(studentId, courseId)); // T1 commits
        invoiceRepo.save(new Invoice(studentId, courseId));       // T2 — if this fails, T1 is already committed!
    }
}

// ✅ Correct: one transaction wraps both operations
@Service
public class EnrollmentService {
    @Transactional
    public void enroll(Long studentId, Long courseId) {
        enrollmentRepo.save(new Enrollment(studentId, courseId)); // Part of T1
        invoiceRepo.save(new Invoice(studentId, courseId));       // Part of T1 — one commit or one rollback
    }
}
```

---

## ✅ Best Practices & Important Points

**Put `@Transactional` on the service layer, not the controller or repository.** Controllers should not manage transactions — they are for HTTP concerns. Repository methods already have their own per-method transactions. Service methods are where multi-step business operations live, and that's where transactions should be scoped.

**Always use `@Transactional(readOnly = true)` for query-only methods.** This is a free optimization: Hibernate skips the dirty-checking phase at flush time (which can be expensive for large result sets), some databases optimize read-only connections, and it communicates intent clearly in code reviews.

**Understand that `@Transactional` on `private` methods has no effect.** Spring's AOP proxy cannot intercept private method calls because the proxy wraps only the public interface of the bean. Spring silently ignores `@Transactional` on private methods. If you annotate a private method and wonder why it has no transaction, this is why.

**Understand the checked vs unchecked rollback default.** If your service throws a checked exception (like `IOException` or a custom checked exception), Spring will commit the transaction by default. This is counter-intuitive to most developers. If you want rollback on checked exceptions, use `rollbackFor = Exception.class`.

**Be careful with `REQUIRES_NEW` and database connections.** `REQUIRES_NEW` suspends the outer transaction and opens a brand new database connection for the inner transaction. Under heavy load, this can exhaust your connection pool if outer and inner transactions run concurrently in many threads. Use it deliberately, not habitually.

**The self-invocation AOP problem is your biggest transaction debugging tool.** If a `@Transactional` method seems to not be committing or rolling back correctly, the first thing to check is whether you're calling it through the Spring proxy or directly via `this`. If you call `this.saveStudent()` from within the same class, the transaction annotation is invisible.

---

## 🔑 Key Summary

```
@Transactional                → Wraps a method in a database transaction; commits on success, rolls back on RuntimeException
readOnly = true               → Skip dirty checking; use for all query-only service methods
Propagation.REQUIRED          → Default; join existing transaction or start new one
Propagation.REQUIRES_NEW      → Start a completely independent transaction; outer tx is suspended
Propagation.NESTED            → Use database savepoints; can partially roll back
Isolation.READ_COMMITTED      → Default in PostgreSQL; prevents dirty reads
Isolation.SERIALIZABLE        → Strongest; prevents all anomalies; use for financial operations
rollbackFor                   → Also roll back on checked exceptions (e.g., rollbackFor = Exception.class)
noRollbackFor                 → Prevent rollback for specific RuntimeExceptions
TransactionTemplate           → Programmatic transaction control; use when annotations won't work
Service layer                 → Where @Transactional belongs; NOT on controllers or repositories
Self-invocation               → Calling @Transactional method from same class bypasses proxy — no transaction!
```
