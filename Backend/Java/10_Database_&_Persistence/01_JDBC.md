# 📗 10.1 — JDBC: Java Database Connectivity

JDBC is the foundation of all database interaction in Java. Before you can appreciate JPA or Hibernate, you must understand what they are abstracting away. Every ORM framework ultimately generates JDBC calls under the hood — so understanding JDBC means you understand the true cost of every database operation.

---

## 🧠 What is JDBC?

JDBC (Java Database Connectivity) is a **Java API** that defines how Java programs connect to and interact with relational databases. It is part of the Java SE standard library under `java.sql` and `javax.sql` packages.

The key design principle of JDBC is **vendor neutrality**: your code stays the same whether you're talking to MySQL, PostgreSQL, Oracle, or SQLite. The database-specific translation is handled by a **JDBC Driver** — a library that the database vendor provides (or the community maintains).

Think of JDBC as an electrical socket standard. Your appliance (Java app) always uses the same plug (JDBC API). The socket's internal wiring (driver) differs by country (database), but the interface you interact with never changes.

---

## 🏗️ JDBC Architecture

The JDBC architecture has four main layers:

```
Java Application
      ↓
  JDBC API (java.sql.*)          ← You write against this
      ↓
 JDBC Driver Manager             ← Selects the right driver
      ↓
  JDBC Driver (vendor-specific)  ← e.g., mysql-connector-java
      ↓
    Database
```

There are four types of JDBC drivers, but in modern applications you will almost always use **Type 4 — Pure Java Driver** (also called Thin Driver). This driver communicates with the database directly using the database's native network protocol, with no native code required. Examples include `mysql-connector-java` and `postgresql`.

---

## 📦 Setting Up JDBC

### Maven Dependencies

Add the appropriate driver to your `pom.xml`. The application code itself stays identical regardless of which driver you use.

```xml
<!-- PostgreSQL -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.7.3</version>
</dependency>

<!-- MySQL -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.3.0</version>
</dependency>

<!-- H2 (in-memory, great for testing) -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>2.2.224</version>
</dependency>
```

---

## 🔗 1. Establishing a Connection

The entry point into JDBC is a `Connection` object. You obtain it from `DriverManager.getConnection()` by passing a **JDBC URL**, a username, and a password.

### JDBC URL Formats

The URL tells the driver which database type to connect to and where it is:

```
jdbc:<subprotocol>://<host>:<port>/<database>?<optional-params>

PostgreSQL:  jdbc:postgresql://localhost:5432/mydb
MySQL:       jdbc:mysql://localhost:3306/mydb?useSSL=false&serverTimezone=UTC
H2 (file):   jdbc:h2:./data/mydb
H2 (memory): jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
Oracle:      jdbc:oracle:thin:@localhost:1521:orcl
SQL Server:  jdbc:sqlserver://localhost:1433;databaseName=mydb
```

### Basic Connection Example

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class JdbcConnectionDemo {

    // Store connection details as constants or load from config/environment
    private static final String URL      = "jdbc:postgresql://localhost:5432/school_db";
    private static final String USERNAME = "postgres";
    private static final String PASSWORD = "secret";

    public static void main(String[] args) {
        // try-with-resources ensures the connection is ALWAYS closed,
        // even if an exception occurs — this is critical to avoid connection leaks!
        try (Connection connection = DriverManager.getConnection(URL, USERNAME, PASSWORD)) {

            System.out.println("Connected to database: " + connection.getMetaData().getDatabaseProductName());
            System.out.println("Driver version: "        + connection.getMetaData().getDriverVersion());

        } catch (SQLException e) {
            // SQLException contains a SQLState code and vendor error code for diagnosis
            System.err.println("Connection failed!");
            System.err.println("SQL State : " + e.getSQLState());
            System.err.println("Error Code: " + e.getErrorCode());
            System.err.println("Message   : " + e.getMessage());
        }
    }
}
```

> ⚠️ **Important:** In older JDBC code you'll see `Class.forName("com.mysql.cj.jdbc.Driver")` before `getConnection()`. This was required in JDBC 3.x to manually register the driver. Since JDBC 4.0 (Java 6+), drivers are auto-registered via the **ServiceLoader** mechanism — you no longer need `Class.forName()`. Modern code should omit it entirely.

---

## 📝 2. Statement — Executing SQL

Once you have a `Connection`, you use it to create a `Statement` to execute SQL. There are three types: `Statement`, `PreparedStatement`, and `CallableStatement`.

### 2.1 Statement — Simple SQL Execution

`Statement` is used for **static SQL** — queries with no parameters.

```java
import java.sql.*;

public class StatementDemo {

    public static void main(String[] args) throws SQLException {
        String url = "jdbc:h2:mem:demo;DB_CLOSE_DELAY=-1";

        try (Connection conn = DriverManager.getConnection(url, "sa", "");
             Statement stmt = conn.createStatement()) {

            // DDL — Create a table
            stmt.execute("""
                CREATE TABLE students (
                    id      INT PRIMARY KEY AUTO_INCREMENT,
                    name    VARCHAR(100) NOT NULL,
                    grade   CHAR(1),
                    score   DOUBLE
                )
            """);

            // DML — Insert rows
            // executeUpdate() returns the number of rows affected
            int rowsInserted = stmt.executeUpdate(
                "INSERT INTO students (name, grade, score) VALUES ('Alice', 'A', 95.5)"
            );
            stmt.executeUpdate("INSERT INTO students (name, grade, score) VALUES ('Bob',   'B', 82.0)");
            stmt.executeUpdate("INSERT INTO students (name, grade, score) VALUES ('Carol', 'A', 97.0)");
            System.out.println("Rows inserted in first statement: " + rowsInserted);

            // DQL — Query rows
            // executeQuery() returns a ResultSet
            ResultSet rs = stmt.executeQuery("SELECT * FROM students ORDER BY score DESC");
            while (rs.next()) {
                // Column access by name is preferred over index — more readable
                int    id    = rs.getInt("id");
                String name  = rs.getString("name");
                String grade = rs.getString("grade");
                double score = rs.getDouble("score");
                System.out.printf("ID: %d | Name: %-10s | Grade: %s | Score: %.1f%n",
                                  id, name, grade, score);
            }
        }
    }
}
```

**Output:**
```
Rows inserted in first statement: 1
ID: 3 | Name: Carol      | Grade: A | Score: 97.0
ID: 1 | Name: Alice      | Grade: A | Score: 95.5
ID: 2 | Name: Bob        | Grade: B | Score: 82.0
```

The three key execution methods are `execute()` (works for any SQL, returns boolean), `executeUpdate()` (for INSERT/UPDATE/DELETE/DDL, returns row count), and `executeQuery()` (for SELECT, returns `ResultSet`).

---

### 2.2 PreparedStatement — Parameterized SQL (Use This Always)

`PreparedStatement` is a **pre-compiled** SQL template where you replace values with `?` placeholders. This is the **most important JDBC class** you should use in real applications.

It has two major advantages over plain `Statement`:

**Security — Prevents SQL Injection:** With `Statement`, if a user inputs `Alice' OR '1'='1`, the query logic can be destroyed. `PreparedStatement` treats parameters as pure data, never as executable SQL — making injection attacks impossible.

**Performance — Pre-compilation:** The database parses and compiles the SQL once (the first time). Subsequent executions with different parameters skip the compilation step, making repeated executions significantly faster.

```java
import java.sql.*;
import java.util.List;

public class PreparedStatementDemo {

    // ❌ VULNERABLE to SQL injection — NEVER do this
    public static void findStudentUnsafe(Connection conn, String name) throws SQLException {
        String sql = "SELECT * FROM students WHERE name = '" + name + "'";
        // If name is: Alice' OR '1'='1' --
        // the query becomes: SELECT * FROM students WHERE name = 'Alice' OR '1'='1' --'
        // This returns ALL students! A classic SQL injection attack.
        try (Statement stmt = conn.createStatement();
             ResultSet rs   = stmt.executeQuery(sql)) {
            while (rs.next()) {
                System.out.println("Found: " + rs.getString("name"));
            }
        }
    }

    // ✅ SAFE — always use PreparedStatement with parameters
    public static void findStudentSafe(Connection conn, String name) throws SQLException {
        // The ? is a placeholder — parameters are never interpreted as SQL
        String sql = "SELECT * FROM students WHERE name = ?";

        try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
            pstmt.setString(1, name); // Parameters are 1-indexed, not 0-indexed!
            try (ResultSet rs = pstmt.executeQuery()) {
                while (rs.next()) {
                    System.out.println("Found: " + rs.getString("name"));
                }
            }
        }
    }

    // Inserting a new student with multiple parameters
    public static void insertStudent(Connection conn, String name, char grade, double score) throws SQLException {
        String sql = "INSERT INTO students (name, grade, score) VALUES (?, ?, ?)";

        // RETURN_GENERATED_KEYS tells the driver to give us back auto-generated IDs
        try (PreparedStatement pstmt = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            pstmt.setString(1, name);
            pstmt.setString(2, String.valueOf(grade));
            pstmt.setDouble(3, score);
            pstmt.executeUpdate();

            // Retrieve the auto-generated primary key
            try (ResultSet generatedKeys = pstmt.getGeneratedKeys()) {
                if (generatedKeys.next()) {
                    long newId = generatedKeys.getLong(1);
                    System.out.println("Inserted student with ID: " + newId);
                }
            }
        }
    }

    // Batch insertion — insert many rows efficiently in a single round-trip
    public static void insertStudentsBatch(Connection conn, List<String> names) throws SQLException {
        String sql = "INSERT INTO students (name, grade, score) VALUES (?, ?, ?)";

        conn.setAutoCommit(false); // Disable auto-commit for batch operations

        try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
            for (int i = 0; i < names.size(); i++) {
                pstmt.setString(1, names.get(i));
                pstmt.setString(2, "C");
                pstmt.setDouble(3, 70.0);
                pstmt.addBatch(); // Queue this set of parameters

                // Flush every 500 rows to avoid memory buildup in very large batches
                if (i % 500 == 0 && i > 0) {
                    pstmt.executeBatch();
                }
            }

            int[] results = pstmt.executeBatch(); // Execute all queued statements
            conn.commit(); // Commit the transaction
            System.out.println("Batch inserted " + results.length + " rows");

        } catch (SQLException e) {
            conn.rollback(); // Roll back on failure
            throw e;
        } finally {
            conn.setAutoCommit(true); // Always restore auto-commit
        }
    }
}
```

### Setter Methods Reference

Every Java type has a corresponding setter in `PreparedStatement`:

```java
pstmt.setInt(1, 42);
pstmt.setLong(2, 100L);
pstmt.setDouble(3, 3.14);
pstmt.setString(4, "hello");
pstmt.setBoolean(5, true);
pstmt.setDate(6, java.sql.Date.valueOf(LocalDate.now()));
pstmt.setTimestamp(7, Timestamp.valueOf(LocalDateTime.now()));
pstmt.setBigDecimal(8, new BigDecimal("99.99"));
pstmt.setNull(9, Types.VARCHAR); // Explicitly set a parameter to NULL
pstmt.setObject(10, someObject); // Generic setter — JDBC figures out the type
```

---

### 2.3 CallableStatement — Stored Procedures

`CallableStatement` is used to call **stored procedures** defined in the database. The syntax uses `{call procedure_name(?, ?)}`.

```java
// Assume a PostgreSQL stored procedure:
// CREATE FUNCTION get_top_students(min_score DOUBLE PRECISION)
//   RETURNS TABLE(id INT, name VARCHAR, score DOUBLE PRECISION) AS ...

public static void callStoredProcedure(Connection conn, double minScore) throws SQLException {
    // {call ...} is the JDBC escape syntax for stored procedure calls
    String sql = "{call get_top_students(?)}";

    try (CallableStatement cstmt = conn.prepareCall(sql)) {
        cstmt.setDouble(1, minScore);
        try (ResultSet rs = cstmt.executeQuery()) {
            while (rs.next()) {
                System.out.printf("ID: %d | Name: %s | Score: %.1f%n",
                    rs.getInt("id"), rs.getString("name"), rs.getDouble("score"));
            }
        }
    }
}

// Stored procedure with OUT parameter (returns a scalar value)
public static int countStudentsAbove(Connection conn, double minScore) throws SQLException {
    String sql = "{? = call count_students_above(?)}";  // First ? is OUT, second is IN

    try (CallableStatement cstmt = conn.prepareCall(sql)) {
        // Register the OUT parameter by position and SQL type
        cstmt.registerOutParameter(1, Types.INTEGER);
        cstmt.setDouble(2, minScore);
        cstmt.execute();
        return cstmt.getInt(1); // Read the OUT parameter after execution
    }
}
```

---

## 📊 3. ResultSet — Reading Query Results

`ResultSet` is a cursor that starts **before** the first row. You call `rs.next()` to advance it row by row. The cursor returns `false` when there are no more rows.

```java
public static void exploreResultSet(Connection conn) throws SQLException {
    String sql = "SELECT id, name, grade, score FROM students";

    try (Statement stmt = conn.createStatement(
             ResultSet.TYPE_SCROLL_INSENSITIVE, // Can scroll forward and backward
             ResultSet.CONCUR_READ_ONLY)) {     // Read-only (most common)

        ResultSet rs = stmt.executeQuery(sql);

        // --- ResultSet Metadata ---
        ResultSetMetaData meta = rs.getMetaData();
        int colCount = meta.getColumnCount();
        System.out.println("Columns: " + colCount);
        for (int i = 1; i <= colCount; i++) {
            System.out.printf("  Col %d: %-15s | Type: %s%n",
                i, meta.getColumnName(i), meta.getColumnTypeName(i));
        }

        // --- Forward iteration (standard) ---
        System.out.println("\n--- All rows (forward) ---");
        while (rs.next()) {
            System.out.println(rs.getInt("id") + " | " + rs.getString("name"));

            // Check for NULL before reading to avoid unexpected 0 or "" values
            if (rs.wasNull()) {
                System.out.println("  (previous value was NULL in DB)");
            }
        }

        // --- Scrollable ResultSet navigation ---
        rs.beforeFirst();       // Jump back to start
        rs.last();              // Jump to last row
        rs.first();             // Jump to first row
        rs.absolute(2);         // Jump to row 2 (1-indexed)
        rs.relative(1);         // Move 1 row forward from current position

        System.out.println("\nRow at absolute(2): " + rs.getString("name"));
    }
}
```

### ResultSet Types

The `TYPE_FORWARD_ONLY` (default) cursor is the most efficient since it only moves forward. `TYPE_SCROLL_INSENSITIVE` allows navigation but does not reflect database changes made after the query. `TYPE_SCROLL_SENSITIVE` reflects changes but has higher overhead and not all drivers support it. For the vast majority of use cases, stick with `TYPE_FORWARD_ONLY`.

---

## 🔄 4. Transactions

A **transaction** is a sequence of SQL operations treated as a single atomic unit. If any step fails, the entire transaction is rolled back — as if none of it happened. This guarantees **ACID** properties (Atomicity, Consistency, Isolation, Durability).

By default, JDBC operates in **auto-commit mode**: every SQL statement is its own transaction and is committed immediately. For multi-step operations, you must manage transactions manually.

```java
public static void transferMoney(Connection conn, int fromAccount, int toAccount, double amount) {
    try {
        // 1. Disable auto-commit to begin a manual transaction
        conn.setAutoCommit(false);

        try (PreparedStatement debit  = conn.prepareStatement(
                 "UPDATE accounts SET balance = balance - ? WHERE id = ?");
             PreparedStatement credit = conn.prepareStatement(
                 "UPDATE accounts SET balance = balance + ? WHERE id = ?")) {

            // 2. Debit the source account
            debit.setDouble(1, amount);
            debit.setInt(2, fromAccount);
            int debitedRows = debit.executeUpdate();
            if (debitedRows == 0) throw new SQLException("Source account not found: " + fromAccount);

            // 3. Credit the destination account
            credit.setDouble(1, amount);
            credit.setInt(2, toAccount);
            int creditedRows = credit.executeUpdate();
            if (creditedRows == 0) throw new SQLException("Target account not found: " + toAccount);

            // 4. All steps succeeded — commit the transaction
            conn.commit();
            System.out.printf("Transferred %.2f from account %d to %d%n", amount, fromAccount, toAccount);

        } catch (SQLException e) {
            // 5. Something went wrong — undo all changes in this transaction
            conn.rollback();
            System.err.println("Transfer failed, rolled back: " + e.getMessage());
            throw e; // Re-throw so caller knows it failed
        }

    } catch (SQLException e) {
        System.err.println("Transaction error: " + e.getMessage());
    } finally {
        // 6. Always restore auto-commit to avoid surprising the next caller
        try { conn.setAutoCommit(true); }
        catch (SQLException ignored) {}
    }
}
```

### Savepoints

Savepoints let you create **rollback points within a transaction** — you can undo back to the savepoint without discarding the entire transaction.

```java
public static void demoSavepoints(Connection conn) throws SQLException {
    conn.setAutoCommit(false);

    try (Statement stmt = conn.createStatement()) {
        stmt.executeUpdate("INSERT INTO log (msg) VALUES ('Step 1')");

        // Create a savepoint after step 1
        Savepoint sp1 = conn.setSavepoint("after_step1");

        stmt.executeUpdate("INSERT INTO log (msg) VALUES ('Step 2')");

        // Suppose step 2 had a problem — roll back to the savepoint
        // Step 1's insert is retained; step 2's insert is undone
        conn.rollback(sp1);

        stmt.executeUpdate("INSERT INTO log (msg) VALUES ('Step 2 (retry)')");

        conn.commit(); // Commits: "Step 1" and "Step 2 (retry)"
    }
}
```

### Isolation Levels

Isolation defines how much concurrent transactions can see each other's uncommitted changes. Higher isolation = fewer concurrency anomalies but lower throughput.

```java
// Set isolation level BEFORE starting the transaction
conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);

// Available levels (from least to most strict):
// TRANSACTION_READ_UNCOMMITTED — can read uncommitted "dirty" data (avoid this)
// TRANSACTION_READ_COMMITTED  — only reads committed data (PostgreSQL default)
// TRANSACTION_REPEATABLE_READ — same reads return same data within a transaction (MySQL default)
// TRANSACTION_SERIALIZABLE    — completely isolated; transactions run as if sequential
```

---

## 🏊 5. Connection Pooling — HikariCP

Opening a new database connection is expensive — it involves TCP handshake, authentication, and session setup. In a web application serving hundreds of requests per second, creating a new connection per request would be catastrophically slow.

**Connection pooling** solves this by maintaining a pool of already-open connections. When your code needs a connection, it borrows one from the pool. When done, the connection is returned to the pool (not actually closed) and made available for the next request.

**HikariCP** is the de facto standard connection pool for Java — it's the default in Spring Boot and is known for being extremely fast and reliable.

```xml
<!-- Maven dependency -->
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>5.1.0</version>
</dependency>
```

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import javax.sql.DataSource;
import java.sql.*;

public class ConnectionPoolDemo {

    // The DataSource is a factory for connections — create it once, share it everywhere
    // This should be a singleton (or a Spring bean) in a real application
    private static DataSource createDataSource() {
        HikariConfig config = new HikariConfig();

        // --- Essential settings ---
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/school_db");
        config.setUsername("postgres");
        config.setPassword("secret");
        config.setDriverClassName("org.postgresql.Driver");

        // --- Pool sizing ---
        // maximumPoolSize: the max number of connections in the pool
        // Rule of thumb for CPU-bound: maximumPoolSize = number of CPU cores * 2
        // For IO-bound (most web apps): start at 10 and tune from there
        config.setMaximumPoolSize(10);
        config.setMinimumIdle(2); // Keep at least 2 connections open always

        // --- Timeouts ---
        config.setConnectionTimeout(30_000);   // Wait up to 30s to get a connection from pool
        config.setIdleTimeout(600_000);         // Return idle connections after 10 minutes
        config.setMaxLifetime(1_800_000);       // Recycle connections after 30 minutes

        // --- Validation ---
        config.setConnectionTestQuery("SELECT 1"); // Query to verify connection is alive
        config.setPoolName("SchoolDBPool");         // Named pool — appears in logs/metrics

        return new HikariDataSource(config);
    }

    private static final DataSource DATA_SOURCE = createDataSource();

    public static void main(String[] args) throws SQLException {
        // Usage is identical to DriverManager — just get a connection from the DataSource
        try (Connection conn = DATA_SOURCE.getConnection()) {
            System.out.println("Got connection from pool: " + conn.isValid(2));
            // ... execute queries ...
        }
        // conn.close() returns the connection to the pool, not truly closing it
    }
}
```

### DataSource vs DriverManager

In any application beyond a simple one-off script, you should always use a `DataSource` instead of `DriverManager.getConnection()`. `DriverManager` creates a brand-new connection every time. `DataSource` serves connections from a pool, manages their lifecycle, validates them, and provides metrics. Frameworks like Spring always work with `DataSource`.

---

## 📦 6. Batch Updates

When you need to execute many similar SQL statements (e.g., bulk-importing data), sending them one by one creates a separate network round-trip per statement. **Batch updates** group multiple statements and send them to the database in one network round-trip — massively improving performance for large datasets.

```java
public static void bulkInsert(DataSource dataSource, List<Student> students) throws SQLException {
    String sql = "INSERT INTO students (name, grade, score) VALUES (?, ?, ?)";
    final int BATCH_SIZE = 1000; // How many rows to send per batch

    try (Connection conn = dataSource.getConnection();
         PreparedStatement pstmt = conn.prepareStatement(sql)) {

        conn.setAutoCommit(false);

        for (int i = 0; i < students.size(); i++) {
            Student s = students.get(i);
            pstmt.setString(1, s.name());
            pstmt.setString(2, s.grade());
            pstmt.setDouble(3, s.score());
            pstmt.addBatch(); // Stage this row for batch execution

            // Execute and clear the batch every BATCH_SIZE rows
            if ((i + 1) % BATCH_SIZE == 0) {
                int[] counts = pstmt.executeBatch();
                System.out.println("Batch executed, rows: " + counts.length);
            }
        }

        // Execute remaining rows that didn't fill a complete batch
        pstmt.executeBatch();
        conn.commit();
    }
}
```

---

## 🔍 7. DatabaseMetaData — Inspecting the Database

`DatabaseMetaData` provides information about the database itself — its capabilities, version, tables, columns, indexes, and more. This is particularly useful for writing database-agnostic code or building inspection tools.

```java
public static void inspectDatabase(Connection conn) throws SQLException {
    DatabaseMetaData meta = conn.getMetaData();

    System.out.println("Database: " + meta.getDatabaseProductName()
                                    + " " + meta.getDatabaseProductVersion());
    System.out.println("Driver  : " + meta.getDriverName()
                                    + " " + meta.getDriverVersion());
    System.out.println("URL     : " + meta.getURL());
    System.out.println("Max Connections: " + meta.getMaxConnections());
    System.out.println("Supports Transactions: " + meta.supportsTransactions());
    System.out.println("Supports Batch Updates: " + meta.supportsBatchUpdates());

    // List all tables in the current schema
    System.out.println("\n--- Tables ---");
    try (ResultSet tables = meta.getTables(null, null, "%", new String[]{"TABLE"})) {
        while (tables.next()) {
            System.out.println("  " + tables.getString("TABLE_NAME"));
        }
    }

    // List columns of a specific table
    System.out.println("\n--- Columns of 'students' ---");
    try (ResultSet cols = meta.getColumns(null, null, "students", "%")) {
        while (cols.next()) {
            System.out.printf("  %-20s | Type: %-12s | Nullable: %s%n",
                cols.getString("COLUMN_NAME"),
                cols.getString("TYPE_NAME"),
                cols.getString("IS_NULLABLE"));
        }
    }
}
```

---

## ✅ Best Practices & Important Points

**Always use PreparedStatement.** Using plain `Statement` with string concatenation is both a security vulnerability (SQL injection) and a performance issue. There are essentially zero valid reasons to use `Statement` for queries with user-supplied data.

**Always close resources in try-with-resources.** Connections, statements, and result sets all hold resources at the database level (threads, memory, file handles). Failing to close them causes resource leaks that silently accumulate until your application crashes or the database runs out of connections. The `try-with-resources` block guarantees closure even when exceptions occur.

**Never call `getConnection()` from `DriverManager` in production.** Use a connection pool (`HikariCP`, `Apache DBCP`, or `C3P0`). `DriverManager` creates a new connection every time, which is far too expensive for production loads.

**Use `conn.setAutoCommit(false)` for multi-step operations.** Any operation that modifies multiple rows or tables should be wrapped in a manual transaction. If step 3 of 4 fails and you were in auto-commit mode, steps 1 and 2 are already permanently in the database — you now have corrupted data.

**Handle `SQLException` meaningfully.** Log the SQL state (`getSQLState()`) and vendor error code (`getErrorCode()`) — they tell you whether the failure was a constraint violation, a deadlock, a connection timeout, or something else. Never silently swallow SQL exceptions.

**Use `rs.wasNull()` after reading numeric columns.** If a database column contains `NULL` and you call `rs.getInt()`, you'll get `0` — not an exception. You won't know whether the database stored `0` or `NULL` unless you call `rs.wasNull()` immediately after the getter.

**Tune your HikariCP pool size thoughtfully.** A common mistake is setting `maximumPoolSize` very high thinking more connections = more throughput. The opposite is often true. The database has limited resources. Many idle connections waste memory on both sides. Research the "HikariCP pool sizing" guide by the author for a deeper understanding.

**Prefer named columns over index-based access in `ResultSet`.** `rs.getString("name")` is far more readable and less error-prone than `rs.getString(2)`. The index approach breaks silently if a column is added or reordered in the query.

---

## 🔑 Key Summary

```
Connection  → Entry point, represents one session with the database
Statement   → Executes static SQL (no parameters) — avoid for user input
PreparedStatement → Parameterized SQL — use for everything with parameters
CallableStatement → Calls stored procedures
ResultSet   → Cursor over query results, call .next() to iterate
DataSource  → Factory for connections (use with connection pool in production)
Transaction → conn.setAutoCommit(false) → operations → commit() or rollback()
```
