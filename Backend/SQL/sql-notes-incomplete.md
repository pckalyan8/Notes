# Complete Database, SQL & System Design Notes

## Table of Contents
- [Part 1: Database Fundamentals](#part-1-database-fundamentals)
- [Part 2: SQL Complete Guide](#part-2-sql-complete-guide)
- [Part 3: High-Level Design (HDD) Problems](#part-3-high-level-design-hdd-problems)
- [Part 4: Low-Level Design (LLD) Problems](#part-4-low-level-design-lld-problems)

---

# Part 1: Database Fundamentals

## 1.1 Introduction to Databases

### What is a Database?
A database is an organized collection of structured data stored electronically in a computer system. It's managed by a Database Management System (DBMS).

### Types of Databases

#### 1. Relational Databases (RDBMS)
- Store data in tables with rows and columns
- Use SQL for querying
- Examples: MySQL, PostgreSQL, Oracle, SQL Server
- ACID compliant

#### 2. NoSQL Databases
- **Document Stores**: MongoDB, CouchDB
- **Key-Value Stores**: Redis, DynamoDB
- **Column-Family Stores**: Cassandra, HBase
- **Graph Databases**: Neo4j, Amazon Neptune

#### 3. NewSQL Databases
- Google Spanner, CockroachDB
- Combine benefits of both SQL and NoSQL

## 1.2 RDBMS Concepts

### Tables (Relations)
- Basic unit of storage
- Consists of rows (tuples) and columns (attributes)

### Keys

#### Primary Key
- Uniquely identifies each record
- Cannot be NULL
- Only one per table

```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(50)
);
```

#### Foreign Key
- Links two tables together
- References primary key of another table

```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

#### Unique Key
- Ensures all values in column are unique
- Can have multiple per table
- Can be NULL (unless specified NOT NULL)

#### Composite Key
- Primary key made up of multiple columns

```sql
CREATE TABLE enrollment (
    student_id INT,
    course_id INT,
    PRIMARY KEY (student_id, course_id)
);
```

### Constraints
- **NOT NULL**: Column cannot have NULL value
- **UNIQUE**: All values must be unique
- **CHECK**: Ensures values meet specific condition
- **DEFAULT**: Sets default value

```sql
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) CHECK (price > 0),
    stock INT DEFAULT 0,
    sku VARCHAR(50) UNIQUE
);
```

## 1.3 Normalization

### Why Normalize?
- Eliminate redundancy
- Ensure data integrity
- Improve query performance
- Reduce storage space

### Normal Forms

#### First Normal Form (1NF)
- Each cell contains atomic (indivisible) values
- Each record is unique

**Before 1NF:**
```
| student_id | name  | courses              |
|------------|-------|----------------------|
| 1          | John  | Math, Physics        |
```

**After 1NF:**
```
| student_id | name  | course   |
|------------|-------|----------|
| 1          | John  | Math     |
| 1          | John  | Physics  |
```

#### Second Normal Form (2NF)
- Must be in 1NF
- No partial dependencies (all non-key attributes depend on entire primary key)

**Before 2NF:**
```
| student_id | course_id | student_name | instructor |
|------------|-----------|--------------|------------|
| 1          | 101       | John         | Dr. Smith  |
```

**After 2NF:**
```
Students:
| student_id | student_name |
|------------|--------------|
| 1          | John         |

Courses:
| course_id | instructor |
|-----------|------------|
| 101       | Dr. Smith  |

Enrollment:
| student_id | course_id |
|------------|-----------|
| 1          | 101       |
```

#### Third Normal Form (3NF)
- Must be in 2NF
- No transitive dependencies (non-key attributes should not depend on other non-key attributes)

**Before 3NF:**
```
| employee_id | name | department_id | department_name |
|-------------|------|---------------|-----------------|
| 1           | John | 10            | Sales           |
```

**After 3NF:**
```
Employees:
| employee_id | name | department_id |
|-------------|------|---------------|
| 1           | John | 10            |

Departments:
| department_id | department_name |
|---------------|-----------------|
| 10            | Sales           |
```

#### Boyce-Codd Normal Form (BCNF)
- Stricter version of 3NF
- For every functional dependency X → Y, X must be a super key

## 1.4 Indexes

### What is an Index?
A data structure that improves the speed of data retrieval operations.

### Types of Indexes

#### 1. B-Tree Index (Default)
- Balanced tree structure
- Good for range queries
- Most common type

```sql
CREATE INDEX idx_employee_name ON employees(name);
```

#### 2. Hash Index
- Fast for equality comparisons
- Not suitable for range queries

#### 3. Composite Index
```sql
CREATE INDEX idx_name_dept ON employees(name, department);
```

#### 4. Unique Index
```sql
CREATE UNIQUE INDEX idx_email ON users(email);
```

#### 5. Full-Text Index
```sql
CREATE FULLTEXT INDEX idx_description ON products(description);
```

### When to Use Indexes
- Columns frequently used in WHERE clauses
- Columns used in JOIN conditions
- Columns used in ORDER BY
- Foreign key columns

### When NOT to Use Indexes
- Small tables
- Columns with high update frequency
- Columns with low cardinality (few unique values)

## 1.5 Transactions and ACID Properties

### ACID Properties

#### Atomicity
- All operations in a transaction succeed or all fail
- No partial updates

```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;
COMMIT; -- or ROLLBACK if error occurs
```

#### Consistency
- Database moves from one valid state to another
- All constraints are satisfied

#### Isolation
- Concurrent transactions don't interfere with each other
- **Isolation Levels:**
  - READ UNCOMMITTED
  - READ COMMITTED
  - REPEATABLE READ
  - SERIALIZABLE

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

#### Durability
- Once committed, changes are permanent
- Survive system crashes

### Transaction Commands
```sql
BEGIN TRANSACTION;
-- SQL statements
COMMIT; -- Save changes

-- or

ROLLBACK; -- Undo changes
```

## 1.6 Database Design Best Practices

1. **Use appropriate data types** - Don't use VARCHAR(255) for everything
2. **Normalize appropriately** - Balance between normalization and performance
3. **Name conventions** - Use consistent, descriptive names
4. **Document your schema** - Add comments to tables and columns
5. **Plan for scalability** - Consider future growth
6. **Use constraints** - Enforce data integrity at database level
7. **Index strategically** - Not too many, not too few
8. **Avoid NULLs when possible** - Use default values or NOT NULL constraints

---

# Part 2: SQL Complete Guide

## 2.1 Data Definition Language (DDL)

### CREATE DATABASE
```sql
CREATE DATABASE company_db;
USE company_db;
```

### CREATE TABLE
```sql
CREATE TABLE employees (
    employee_id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    phone VARCHAR(15),
    hire_date DATE NOT NULL,
    job_title VARCHAR(50),
    salary DECIMAL(10,2),
    department_id INT,
    manager_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (department_id) REFERENCES departments(department_id),
    FOREIGN KEY (manager_id) REFERENCES employees(employee_id)
);
```

### ALTER TABLE
```sql
-- Add column
ALTER TABLE employees ADD COLUMN address VARCHAR(200);

-- Modify column
ALTER TABLE employees MODIFY COLUMN salary DECIMAL(12,2);

-- Drop column
ALTER TABLE employees DROP COLUMN phone;

-- Add constraint
ALTER TABLE employees ADD CONSTRAINT chk_salary CHECK (salary > 0);

-- Rename table
ALTER TABLE employees RENAME TO staff;
```

### DROP TABLE
```sql
DROP TABLE IF EXISTS employees;
```

### TRUNCATE TABLE
```sql
TRUNCATE TABLE employees; -- Deletes all data, faster than DELETE
```

## 2.2 Data Manipulation Language (DML)

### INSERT
```sql
-- Single row
INSERT INTO employees (first_name, last_name, email, hire_date, salary)
VALUES ('John', 'Doe', 'john.doe@company.com', '2024-01-15', 75000);

-- Multiple rows
INSERT INTO employees (first_name, last_name, email, hire_date, salary)
VALUES 
    ('Jane', 'Smith', 'jane.smith@company.com', '2024-01-20', 80000),
    ('Bob', 'Johnson', 'bob.j@company.com', '2024-02-01', 72000);

-- Insert from another table
INSERT INTO employees_backup
SELECT * FROM employees WHERE hire_date < '2024-01-01';
```

### UPDATE
```sql
-- Update single record
UPDATE employees 
SET salary = 85000 
WHERE employee_id = 1;

-- Update multiple columns
UPDATE employees 
SET salary = salary * 1.1, 
    job_title = 'Senior Developer'
WHERE employee_id = 5;

-- Update with JOIN
UPDATE employees e
JOIN departments d ON e.department_id = d.department_id
SET e.salary = e.salary * 1.05
WHERE d.department_name = 'Sales';
```

### DELETE
```sql
-- Delete specific records
DELETE FROM employees WHERE employee_id = 10;

-- Delete with condition
DELETE FROM employees WHERE hire_date < '2020-01-01';

-- Delete all records (use TRUNCATE instead for better performance)
DELETE FROM employees;
```

## 2.3 Data Query Language (DQL)

### SELECT Basics
```sql
-- Select all columns
SELECT * FROM employees;

-- Select specific columns
SELECT first_name, last_name, salary FROM employees;

-- Select with alias
SELECT 
    first_name AS "First Name",
    last_name AS "Last Name",
    salary AS "Annual Salary"
FROM employees;

-- Select distinct values
SELECT DISTINCT department_id FROM employees;
```

### WHERE Clause
```sql
-- Comparison operators
SELECT * FROM employees WHERE salary > 75000;
SELECT * FROM employees WHERE hire_date = '2024-01-15';

-- Logical operators
SELECT * FROM employees 
WHERE salary > 70000 AND department_id = 5;

SELECT * FROM employees 
WHERE department_id = 3 OR department_id = 5;

-- BETWEEN
SELECT * FROM employees 
WHERE salary BETWEEN 60000 AND 80000;

-- IN
SELECT * FROM employees 
WHERE department_id IN (1, 3, 5);

-- LIKE (pattern matching)
SELECT * FROM employees WHERE last_name LIKE 'Sm%'; -- Starts with Sm
SELECT * FROM employees WHERE email LIKE '%@gmail.com'; -- Ends with
SELECT * FROM employees WHERE first_name LIKE '_ohn'; -- J_hn matches John

-- IS NULL / IS NOT NULL
SELECT * FROM employees WHERE manager_id IS NULL;
SELECT * FROM employees WHERE email IS NOT NULL;
```

### ORDER BY
```sql
-- Ascending order (default)
SELECT * FROM employees ORDER BY salary;

-- Descending order
SELECT * FROM employees ORDER BY salary DESC;

-- Multiple columns
SELECT * FROM employees 
ORDER BY department_id ASC, salary DESC;
```

### LIMIT and OFFSET
```sql
-- Get first 10 records
SELECT * FROM employees LIMIT 10;

-- Pagination (skip 10, get next 10)
SELECT * FROM employees LIMIT 10 OFFSET 10;

-- Alternative syntax
SELECT * FROM employees LIMIT 10, 10; -- OFFSET 10, LIMIT 10
```

## 2.4 Aggregate Functions

### Common Aggregate Functions
```sql
-- COUNT
SELECT COUNT(*) FROM employees; -- Total records
SELECT COUNT(DISTINCT department_id) FROM employees; -- Unique departments

-- SUM
SELECT SUM(salary) AS total_payroll FROM employees;

-- AVG
SELECT AVG(salary) AS average_salary FROM employees;

-- MIN and MAX
SELECT MIN(salary) AS lowest_salary FROM employees;
SELECT MAX(salary) AS highest_salary FROM employees;

-- Combined
SELECT 
    COUNT(*) AS total_employees,
    AVG(salary) AS avg_salary,
    MIN(salary) AS min_salary,
    MAX(salary) AS max_salary,
    SUM(salary) AS total_payroll
FROM employees;
```

### GROUP BY
```sql
-- Group by single column
SELECT department_id, COUNT(*) AS employee_count
FROM employees
GROUP BY department_id;

-- Group by multiple columns
SELECT department_id, job_title, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id, job_title;

-- With ORDER BY
SELECT department_id, COUNT(*) AS count
FROM employees
GROUP BY department_id
ORDER BY count DESC;
```

### HAVING
```sql
-- Filter after grouping
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 75000;

-- Multiple conditions
SELECT department_id, COUNT(*) AS count, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 5 AND AVG(salary) > 70000;
```

## 2.5 JOINS

### Sample Tables for JOIN Examples
```sql
-- Departments table
CREATE TABLE departments (
    department_id INT PRIMARY KEY,
    department_name VARCHAR(50)
);

-- Projects table
CREATE TABLE projects (
    project_id INT PRIMARY KEY,
    project_name VARCHAR(100),
    department_id INT
);
```

### INNER JOIN
```sql
-- Returns only matching records from both tables
SELECT 
    e.first_name, 
    e.last_name, 
    d.department_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.department_id;

-- Multiple joins
SELECT 
    e.first_name,
    e.last_name,
    d.department_name,
    p.project_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.department_id
INNER JOIN projects p ON d.department_id = p.department_id;
```

### LEFT JOIN (LEFT OUTER JOIN)
```sql
-- Returns all records from left table and matching from right
SELECT 
    e.first_name,
    e.last_name,
    d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id;
```

### RIGHT JOIN (RIGHT OUTER JOIN)
```sql
-- Returns all records from right table and matching from left
SELECT 
    e.first_name,
    e.last_name,
    d.department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.department_id;
```

### FULL OUTER JOIN
```sql
-- Returns all records when there's a match in either table
-- MySQL doesn't support FULL OUTER JOIN directly, use UNION
SELECT 
    e.first_name,
    e.last_name,
    d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id
UNION
SELECT 
    e.first_name,
    e.last_name,
    d.department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.department_id;
```

### CROSS JOIN
```sql
-- Cartesian product of both tables
SELECT e.first_name, d.department_name
FROM employees e
CROSS JOIN departments d;
```

### SELF JOIN
```sql
-- Join table to itself
SELECT 
    e1.first_name AS employee,
    e2.first_name AS manager
FROM employees e1
LEFT JOIN employees e2 ON e1.manager_id = e2.employee_id;
```

## 2.6 Subqueries

### Subquery in WHERE Clause
```sql
-- Find employees earning more than average
SELECT first_name, last_name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

-- IN with subquery
SELECT first_name, last_name
FROM employees
WHERE department_id IN (
    SELECT department_id 
    FROM departments 
    WHERE department_name IN ('Sales', 'Marketing')
);

-- EXISTS
SELECT first_name, last_name
FROM employees e
WHERE EXISTS (
    SELECT 1 FROM projects p 
    WHERE p.department_id = e.department_id
);
```

### Subquery in SELECT Clause
```sql
SELECT 
    first_name,
    last_name,
    salary,
    (SELECT AVG(salary) FROM employees) AS avg_salary,
    salary - (SELECT AVG(salary) FROM employees) AS diff_from_avg
FROM employees;
```

### Subquery in FROM Clause (Derived Table)
```sql
SELECT dept_stats.department_id, dept_stats.avg_salary
FROM (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
) AS dept_stats
WHERE dept_stats.avg_salary > 70000;
```

### Correlated Subquery
```sql
-- Subquery references outer query
SELECT e1.first_name, e1.last_name, e1.salary
FROM employees e1
WHERE salary > (
    SELECT AVG(salary)
    FROM employees e2
    WHERE e2.department_id = e1.department_id
);
```

## 2.7 Set Operations

### UNION
```sql
-- Combines results, removes duplicates
SELECT first_name, last_name FROM employees WHERE department_id = 1
UNION
SELECT first_name, last_name FROM employees WHERE department_id = 2;
```

### UNION ALL
```sql
-- Combines results, keeps duplicates
SELECT first_name FROM employees WHERE salary > 70000
UNION ALL
SELECT first_name FROM employees WHERE hire_date > '2024-01-01';
```

### INTERSECT (MySQL alternative using INNER JOIN)
```sql
-- Records that appear in both queries
SELECT DISTINCT e1.employee_id
FROM employees e1
INNER JOIN (
    SELECT employee_id FROM employees WHERE salary > 70000
) e2 ON e1.employee_id = e2.employee_id
WHERE e1.hire_date > '2024-01-01';
```

### EXCEPT/MINUS (MySQL alternative)
```sql
-- Records in first query but not in second
SELECT employee_id FROM employees WHERE department_id = 1
AND employee_id NOT IN (
    SELECT employee_id FROM employees WHERE salary > 80000
);
```

## 2.8 String Functions

```sql
-- CONCAT
SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM employees;

-- UPPER and LOWER
SELECT UPPER(first_name), LOWER(last_name) FROM employees;

-- LENGTH
SELECT first_name, LENGTH(first_name) AS name_length FROM employees;

-- SUBSTRING
SELECT SUBSTRING(email, 1, 10) FROM employees;

-- TRIM, LTRIM, RTRIM
SELECT TRIM('  hello  ') AS trimmed;

-- REPLACE
SELECT REPLACE(email, '@company.com', '@newdomain.com') FROM employees;

-- LEFT and RIGHT
SELECT LEFT(first_name, 3), RIGHT(last_name, 3) FROM employees;
```

## 2.9 Date and Time Functions

```sql
-- Current date and time
SELECT NOW(), CURDATE(), CURTIME();

-- Date formatting
SELECT DATE_FORMAT(hire_date, '%Y-%m-%d') FROM employees;

-- Date arithmetic
SELECT hire_date, DATE_ADD(hire_date, INTERVAL 90 DAY) AS probation_end 
FROM employees;

SELECT DATEDIFF(NOW(), hire_date) AS days_employed FROM employees;

-- Extract parts
SELECT 
    YEAR(hire_date) AS year,
    MONTH(hire_date) AS month,
    DAY(hire_date) AS day
FROM employees;

-- Age calculation
SELECT 
    first_name,
    hire_date,
    TIMESTAMPDIFF(YEAR, hire_date, NOW()) AS years_employed
FROM employees;
```

## 2.10 Conditional Functions

### CASE Statement
```sql
SELECT 
    first_name,
    last_name,
    salary,
    CASE 
        WHEN salary >= 100000 THEN 'High'
        WHEN salary >= 70000 THEN 'Medium'
        ELSE 'Low'
    END AS salary_grade
FROM employees;

-- CASE in aggregation
SELECT 
    department_id,
    SUM(CASE WHEN salary > 75000 THEN 1 ELSE 0 END) AS high_earners,
    SUM(CASE WHEN salary <= 75000 THEN 1 ELSE 0 END) AS low_earners
FROM employees
GROUP BY department_id;
```

### IF Function
```sql
SELECT 
    first_name,
    salary,
    IF(salary > 75000, 'High', 'Low') AS salary_category
FROM employees;
```

### COALESCE
```sql
-- Returns first non-NULL value
SELECT 
    first_name,
    COALESCE(manager_id, 0) AS manager_id
FROM employees;
```

### NULLIF
```sql
-- Returns NULL if two values are equal
SELECT NULLIF(salary, 0) FROM employees;
```

## 2.11 Window Functions (Advanced)

### ROW_NUMBER
```sql
SELECT 
    first_name,
    last_name,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS salary_rank
FROM employees;
```

### RANK and DENSE_RANK
```sql
SELECT 
    first_name,
    salary,
    RANK() OVER (ORDER BY salary DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM employees;
```

### PARTITION BY
```sql
SELECT 
    first_name,
    department_id,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_rank,
    AVG(salary) OVER (PARTITION BY department_id) AS dept_avg_salary
FROM employees;
```

### LEAD and LAG
```sql
SELECT 
    first_name,
    hire_date,
    salary,
    LAG(salary) OVER (ORDER BY hire_date) AS previous_salary,
    LEAD(salary) OVER (ORDER BY hire_date) AS next_salary
FROM employees;
```

### NTILE
```sql
-- Divide into quartiles
SELECT 
    first_name,
    salary,
    NTILE(4) OVER (ORDER BY salary) AS salary_quartile
FROM employees;
```

## 2.12 Common Table Expressions (CTE)

### Basic CTE
```sql
WITH high_earners AS (
    SELECT * FROM employees WHERE salary > 80000
)
SELECT first_name, last_name, salary 
FROM high_earners
ORDER BY salary DESC;
```

### Multiple CTEs
```sql
WITH 
dept_avg AS (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
),
high_salary_depts AS (
    SELECT department_id 
    FROM dept_avg 
    WHERE avg_salary > 75000
)
SELECT e.first_name, e.last_name, e.salary
FROM employees e
INNER JOIN high_salary_depts h ON e.department_id = h.department_id;
```

### Recursive CTE
```sql
-- Employee hierarchy
WITH RECURSIVE employee_hierarchy AS (
    -- Base case: top-level managers
    SELECT employee_id, first_name, manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: employees reporting to previous level
    SELECT e.employee_id, e.first_name, e.manager_id, eh.level + 1
    FROM employees e
    INNER JOIN employee_hierarchy eh ON e.manager_id = eh.employee_id
)
SELECT * FROM employee_hierarchy ORDER BY level, employee_id;
```

## 2.13 Views

### Create View
```sql
CREATE VIEW employee_details AS
SELECT 
    e.employee_id,
    CONCAT(e.first_name, ' ', e.last_name) AS full_name,
    e.email,
    d.department_name,
    e.salary
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id;

-- Use view
SELECT * FROM employee_details WHERE salary > 75000;
```

### Updatable Views
```sql
-- Create updatable view
CREATE VIEW sales_employees AS
SELECT employee_id, first_name, last_name, salary
FROM employees
WHERE department_id = (SELECT department_id FROM departments WHERE department_name = 'Sales');

-- Update through view
UPDATE sales_employees SET salary = salary * 1.1 WHERE employee_id = 5;
```

### Drop View
```sql
DROP VIEW IF EXISTS employee_details;
```

## 2.14 Stored Procedures

```sql
-- Create procedure
DELIMITER //
CREATE PROCEDURE GetEmployeesByDepartment(IN dept_id INT)
BEGIN
    SELECT * FROM employees WHERE department_id = dept_id;
END //
DELIMITER ;

-- Call procedure
CALL GetEmployeesByDepartment(5);

-- Procedure with OUT parameter
DELIMITER //
CREATE PROCEDURE GetEmployeeCount(IN dept_id INT, OUT emp_count INT)
BEGIN
    SELECT COUNT(*) INTO emp_count 
    FROM employees 
    WHERE department_id = dept_id;
END //
DELIMITER ;

-- Call with OUT parameter
CALL GetEmployeeCount(5, @count);
SELECT @count;
```

## 2.15 Triggers

```sql
-- Create audit table
CREATE TABLE employee_audit (
    audit_id INT AUTO_INCREMENT PRIMARY KEY,
    employee_id INT,
    old_salary DECIMAL(10,2),
    new_salary DECIMAL(10,2),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create trigger
DELIMITER //
CREATE TRIGGER salary_change_audit
AFTER UPDATE ON employees
FOR EACH ROW
BEGIN
    IF OLD.salary != NEW.salary THEN
        INSERT INTO employee_audit (employee_id, old_salary, new_salary)
        VALUES (NEW.employee_id, OLD.salary, NEW.salary);
    END IF;
END //
DELIMITER ;
```

---

# Part 3: High-Level Design (HDD) Problems

## 3.1 Design URL Shortener (like bit.ly)

### Requirements
**Functional:**
- Generate short URL from long URL
- Redirect short URL to original URL
- Custom aliases (optional)
- Analytics (click tracking)

**Non-Functional:**
- High availability
- Low latency for redirects
- Scalable (billions of URLs)

### Solution

#### API Design
```
POST /api/shorten
{
  "long_url": "https://example.com/very/long/url",
  "custom_alias": "my-link" (optional),
  "expiry": "2025-12-31" (optional)
}

Response:
{
  "short_url": "https://short.ly/abc123",
  "long_url": "https://example.com/very/long/url"
}

GET /{short_code} -> Redirects to long URL
```

#### Database Schema
```sql
CREATE TABLE urls (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    user_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expiry_date TIMESTAMP,
    INDEX idx_short_code (short_code)
);

CREATE TABLE analytics (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    short_code VARCHAR(10),
    accessed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address VARCHAR(45),
    user_agent TEXT,
    referer TEXT,
    INDEX idx_short_code (short_code),
    INDEX idx_accessed_at (accessed_at)
);
```

#### High-Level Architecture
```
┌─────────┐
│ Client  │
└────┬────┘
     │
     ▼
┌─────────────────┐
│  Load Balancer  │
└────┬────────────┘
     │
     ▼
┌──────────────────────────────┐
│   Application Servers        │
│  (URL Generation/Redirect)   │
└────┬─────────────────────────┘
     │
     ├──────────────┬──────────────┐
     ▼              ▼              ▼
┌─────────┐   ┌──────────┐   ┌─────────┐
│  Cache  │   │ Database │   │  Queue  │
│ (Redis) │   │ (MySQL)  │   │(Analytics)│
└─────────┘   └──────────┘   └─────────┘
```

#### Key Design Decisions

**1. Short Code Generation:**
- **Base62 Encoding**: Use [a-zA-Z0-9] = 62 characters
- 7 characters = 62^7 = 3.5 trillion URLs
- Convert auto-increment ID to base62

```python
def id_to_base62(id):
    chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    base = len(chars)
    result = []
    while id > 0:
        result.append(chars[id % base])
        id //= base
    return ''.join(reversed(result))
```

**2. Caching Strategy:**
- Cache popular URLs in Redis
- LRU eviction policy
- Cache hit = direct redirect, no DB query

**3. Analytics:**
- Async processing using message queue
- Batch insert to reduce DB load
- Separate analytics database/table

**4. Database Sharding:**
- Shard by hash of short_code
- Consistent hashing for scalability

**5. Rate Limiting:**
- Prevent abuse
- Per user/IP limits

### Scalability Considerations
- **Read-heavy**: 100:1 read-to-write ratio
- **CDN**: Cache static content
- **Database replication**: Master-slave for reads
- **Partitioning**: Hash-based on short_code

---

## 3.2 Design Twitter

### Requirements
**Functional:**
- Post tweets (280 chars)
- Follow/unfollow users
- Timeline (home feed)
- Search tweets
- Likes, retweets

**Non-Functional:**
- High availability
- Low latency for timeline
- Handle millions of users

### Solution

#### Database Schema
```sql
CREATE TABLE users (
    user_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_username (username)
);

CREATE TABLE tweets (
    tweet_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    likes_count INT DEFAULT 0,
    retweets_count INT DEFAULT 0,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    INDEX idx_user_created (user_id, created_at)
);

CREATE TABLE followers (
    follower_id BIGINT,
    followee_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id),
    INDEX idx_followee (followee_id)
);

CREATE TABLE likes (
    user_id BIGINT,
    tweet_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, tweet_id)
);
```

#### High-Level Architecture
```
                    ┌──────────────┐
                    │   CDN        │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │Load Balancer │
                    └──────┬───────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   ┌────▼────┐      ┌──────▼─────┐    ┌──────▼─────┐
   │  Tweet  │      │  Timeline  │    │   User     │
   │ Service │      │  Service   │    │  Service   │
   └────┬────┘      └──────┬─────┘    └──────┬─────┘
        │                  │                  │
        └─────────┬────────┴──────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
   ┌────▼────┐        ┌─────▼──────┐
   │  Cache  │        │  Database  │
   │ (Redis) │        │ (Sharded)  │
   └─────────┘        └────────────┘
```

#### Timeline Generation Strategies

**1. Fanout on Write (Push Model):**
- When user tweets, push to all followers' timelines
- Pre-compute timelines
- Fast reads, slow writes
- Good for users with few followers

```python
def post_tweet(user_id, content):
    tweet_id = create_tweet(user_id, content)
    followers = get_followers(user_id)
    
    for follower_id in followers:
        add_to_timeline_cache(follower_id, tweet_id)
```

**2. Fanout on Read (Pull Model):**
- Fetch tweets from followees on timeline request
- Fast writes, slower reads
- Good for celebrity accounts

```python
def get_timeline(user_id):
    followees = get_followees(user_id)
    tweets = []
    
    for followee_id in followees:
        tweets.extend(get_recent_tweets(followee_id))
    
    return sort_by_timestamp(tweets)[:100]
```

**3. Hybrid Approach:**
- Fanout on write for most users
- Fanout on read for celebrities
- Best of both worlds

#### Key Design Decisions

**1. Caching:**
- Cache timelines in Redis
- Cache hot tweets
- TTL-based expiration

**2. Database Sharding:**
- Shard tweets by user_id
- Shard timeline by user_id
- Consistent hashing

**3. Search:**
- Elasticsearch for full-text search
- Index tweets asynchronously

**4. Notifications:**
- Message queue for async processing
- WebSockets for real-time updates

---

## 3.3 Design Instagram

### Requirements
**Functional:**
- Upload photos/videos
- Follow users
- Feed generation
- Likes, comments
- Stories (24-hour content)

**Non-Functional:**
- High availability
- Low latency for feed
- Handle large media files
- Scalable storage

### Solution

#### Database Schema
```sql
CREATE TABLE users (
    user_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE,
    bio TEXT,
    profile_pic_url VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE posts (
    post_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    media_url VARCHAR(500) NOT NULL,
    media_type ENUM('image', 'video') NOT NULL,
    caption TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    likes_count INT DEFAULT 0,
    comments_count INT DEFAULT 0,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    INDEX idx_user_created (user_id, created_at)
);

CREATE TABLE follows (
    follower_id BIGINT,
    followee_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id)
);

CREATE TABLE likes (
    user_id BIGINT,
    post_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, post_id)
);

CREATE TABLE comments (
    comment_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    post_id BIGINT NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (post_id) REFERENCES posts(post_id),
    INDEX idx_post_created (post_id, created_at)
);

CREATE TABLE stories (
    story_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    media_url VARCHAR(500) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    INDEX idx_expires (expires_at)
);
```

#### High-Level Architecture
```
                        ┌──────────┐
                        │   CDN    │
                        └────┬─────┘
                             │
                      ┌──────▼──────┐
                      │Load Balancer│
                      └──────┬──────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
         ┌────▼───┐    ┌─────▼────┐   ┌────▼────┐
         │ Upload │    │   Feed   │   │  User   │
         │Service │    │ Service  │   │ Service │
         └────┬───┘    └─────┬────┘   └─────┬───┘
              │              │              │
              │         ┌────▼────┐         │
              │         │  Cache  │         │
              │         │ (Redis) │         │
              │         └─────────┘         │
              │                             │
       ┌──────▼──────┐           ┌──────────▼────┐
       │   Object    │           │   Database    │
       │   Storage   │           │   (Sharded)   │
       │ (S3/Blob)   │           └───────────────┘
       └─────────────┘
```

#### Key Components

**1. Media Upload Flow:**
```
1. Client uploads to Upload Service
2. Upload Service generates pre-signed URL for S3
3. Client uploads directly to S3
4. Upload Service saves metadata to DB
5. Async processing: Generate thumbnails, compress
```

**2. Feed Generation:**
- Similar to Twitter's hybrid approach
- Pre-compute feeds for most users
- On-demand for users following celebrities

**3. Stories:**
- Separate from regular posts
- Auto-delete after 24 hours (background job)
- Higher priority in feed

**4. Media Storage:**
- Store originals in S3/Blob storage
- Generate multiple resolutions
- CDN for delivery
- Lazy loading for images

#### Optimization Techniques

**1. Image Processing:**
- Multiple resolutions (thumbnail, medium, full)
- WebP format for compression
- Lazy loading

**2. Database Optimization:**
- Partition posts by date
- Archive old data
- Read replicas for scaling

**3. Caching Strategy:**
- Cache user profiles
- Cache hot posts
- Cache precomputed feeds

---

## 3.4 Design Uber/Ride Sharing

### Requirements
**Functional:**
- Rider requests ride
- Match with nearby driver
- Real-time location tracking
- Fare calculation
- Payment processing
- Ratings

**Non-Functional:**
- Low latency matching (<5 seconds)
- Real-time updates
- High availability
- Handle millions of concurrent users

### Solution

#### Database Schema
```sql
CREATE TABLE users (
    user_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE,
    phone VARCHAR(15) UNIQUE,
    user_type ENUM('rider', 'driver') NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE drivers (
    driver_id BIGINT PRIMARY KEY,
    license_number VARCHAR(50) UNIQUE,
    vehicle_type VARCHAR(50),
    vehicle_number VARCHAR(20),
    is_available BOOLEAN DEFAULT FALSE,
    current_lat DECIMAL(10, 8),
    current_lng DECIMAL(11, 8),
    last_location_update TIMESTAMP,
    FOREIGN KEY (driver_id) REFERENCES users(user_id),
    SPATIAL INDEX idx_location (current_lat, current_lng)
);

CREATE TABLE rides (
    ride_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    rider_id BIGINT NOT NULL,
    driver_id BIGINT,
    pickup_lat DECIMAL(10, 8),
    pickup_lng DECIMAL(11, 8),
    dropoff_lat DECIMAL(10, 8),
    dropoff_lng DECIMAL(11, 8),
    status ENUM('requested', 'accepted', 'started', 'completed', 'cancelled'),
    fare DECIMAL(10, 2),
    requested_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    accepted_at TIMESTAMP,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    FOREIGN KEY (rider_id) REFERENCES users(user_id),
    FOREIGN KEY (driver_id) REFERENCES users(user_id),
    INDEX idx_rider (rider_id),
    INDEX idx_driver (driver_id),
    INDEX idx_status (status)
);

CREATE TABLE ratings (
    rating_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    ride_id BIGINT NOT NULL,
    rater_id BIGINT NOT NULL,
    ratee_id BIGINT NOT NULL,
    rating INT CHECK (rating BETWEEN 1 AND 5),
    comment TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (ride_id) REFERENCES rides(ride_id)
);
```

#### High-Level Architecture
```
                    ┌──────────────┐
                    │Load Balancer │
                    └──────┬───────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
    ┌────▼────┐      ┌─────▼──────┐    ┌────▼────┐
    │  Rider  │      │   Driver   │    │ Matching│
    │ Service │      │  Service   │    │ Service │
    └────┬────┘      └─────┬──────┘    └────┬────┘
         │                 │                 │
         └────────┬────────┴─────────────────┘
                  │
        ┌─────────┴──────────┐
        │                    │
   ┌────▼──────┐      ┌──────▼────────┐
   │  WebSocket│      │  Notification │
   │  Server   │      │    Service    │
   └────┬──────┘      └───────────────┘
        │
   ┌────▼──────────┐
   │   Redis       │
   │(Location Cache)│
   └────┬──────────┘
        │
   ┌────▼──────┐
   │ Database  │
   └───────────┘
```

#### Key Components

**1. Location Tracking:**
```python
# Store driver locations in Redis with Geospatial data
GEOADD drivers:locations   

# Find nearby drivers within 5km
GEORADIUS drivers:locations   5 km
```

**2. Matching Algorithm:**
```python
def find_driver(rider_lat, rider_lng):
    # Get available drivers within 5km
    nearby_drivers = get_nearby_drivers(rider_lat, rider_lng, radius=5)
    
    # Filter available drivers
    available = [d for d in nearby_drivers if d.is_available]
    
    if not available:
        return None
    
    # Sort by distance, rating, acceptance rate
    best_driver = sorted(available, 
                         key=lambda d: (
                             calculate_distance(rider_lat, rider_lng, d.lat, d.lng),
                             -d.rating,
                             -d.acceptance_rate
                         ))[0]
    
    return best_driver
```

**3. Fare Calculation:**
```python
def calculate_fare(distance_km, duration_min, surge_multiplier=1.0):
    BASE_FARE = 2.50
    PER_KM = 1.50
    PER_MIN = 0.30
    MINIMUM_FARE = 5.00
    
    fare = (BASE_FARE + 
            (distance_km * PER_KM) + 
            (duration_min * PER_MIN)) * surge_multiplier
    
    return max(fare, MINIMUM_FARE)
```

**4. Real-time Updates:**
- WebSocket connections for live location
- Push notifications for ride updates
- Heartbeat mechanism for connection health

#### Scalability Considerations

**1. Geospatial Indexing:**
- Use QuadTree or S2 geometry
- Divide world into grids
- Query only relevant grids

**2. Database Sharding:**
- Shard by geographic region
- Hot spots in cities need special handling

**3. Surge Pricing:**
- Monitor supply/demand ratio
- Dynamic pricing based on area and time

**4. Caching:**
- Cache driver locations (Redis)
- Cache surge pricing multipliers
- Cache user profiles

---

## 3.5 Design Netflix

### Requirements
**Functional:**
- Upload videos
- Stream videos
- Search content
- Recommendations
- User profiles
- Resume playback

**Non-Functional:**
- High availability (99.99%)
- Low latency streaming
- Handle millions of concurrent streams
- Global distribution

### Solution

#### Database Schema
```sql
CREATE TABLE users (
    user_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(100) UNIQUE NOT NULL,
    subscription_type ENUM('basic', 'standard', 'premium'),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE profiles (
    profile_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    name VARCHAR(50) NOT NULL,
    is_kids BOOLEAN DEFAULT FALSE,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

CREATE TABLE videos (
    video_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    description TEXT,
    duration_seconds INT,
    release_year INT,
    content_type ENUM('movie', 'series'),
    rating VARCHAR(10),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_title (title),
    INDEX idx_release (release_year)
);

CREATE TABLE video_files (
    file_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    video_id BIGINT NOT NULL,
    quality ENUM('360p', '480p', '720p', '1080p', '4K'),
    file_url VARCHAR(500) NOT NULL,
    file_size_mb INT,
    FOREIGN KEY (video_id) REFERENCES videos(video_id),
    INDEX idx_video_quality (video_id, quality)
);

CREATE TABLE watch_history (
    history_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    profile_id BIGINT NOT NULL,
    video_id BIGINT NOT NULL,
    watched_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    watch_duration_seconds INT,
    progress_seconds INT,
    completed BOOLEAN DEFAULT FALSE,
    FOREIGN KEY (profile_id) REFERENCES profiles(profile_id),
    FOREIGN KEY (video_id) REFERENCES videos(video_id),
    INDEX idx_profile_watched (profile_id, watched_at)
);

CREATE TABLE genres (
    genre_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE video_genres (
    video_id BIGINT,
    genre_id INT,
    PRIMARY KEY (video_id, genre_id),
    FOREIGN KEY (video_id) REFERENCES videos(video_id),
    FOREIGN KEY (genre_id) REFERENCES genres(genre_id)
);
```

#### High-Level Architecture
```
                          ┌─────────┐
                          │   CDN   │
                          └────┬────┘
                               │
                        ┌──────▼──────┐
                        │Load Balancer│
                        └──────┬──────┘
                               │
            ┌──────────────────┼──────────────────┐
            │                  │                  │
       ┌────▼────┐      ┌──────▼─────┐     ┌─────▼──────┐
       │  User   │      │  Streaming │     │    Rec     │
       │ Service │      │  Service   │     │  Service   │
       └────┬────┘      └──────┬─────┘     └─────┬──────┘
            │                  │                  │
            └────────┬─────────┴──────────────────┘
                     │
         ┌───────────┴───────────┐
         │                       │
    ┌────▼────┐           ┌──────▼─────┐
    │  Cache  │           │  Database  │
    │ (Redis) │           │ (Sharded)  │
    └─────────┘           └────────────┘
                                 │
                          ┌──────▼──────┐
                          │   Object    │
                          │   Storage   │
                          │    (S3)     │
                          └─────────────┘
```

#### Key Components

**1. Video Processing Pipeline:**
```
1. Upload original video to S3
2. Transcode to multiple formats (360p, 720p, 1080p, 4K)
3. Generate thumbnails at different timestamps
4. Create subtitle files
5. Store all versions in S3
6. Update metadata in database
```

**2. Adaptive Bitrate Streaming:**
- HLS (HTTP Live Streaming) or DASH
- Client automatically switches quality based on bandwidth
- Chunked video segments (10-second chunks)

**3. CDN Strategy:**
- Distribute content globally
- Edge locations near users
- Cache popular content
- Prefetch trending shows

**4. Recommendation System:**
```python
# Collaborative Filtering
def get_recommendations(user_id):
    # Get user's watch history
    user_videos = get_watched_videos(user_id)
    
    # Find similar users (watched similar content)
    similar_users = find_similar_users(user_videos)
    
    # Get videos watched by similar users
    candidate_videos = get_videos_from_users(similar_users)
    
    # Remove already watched
    recommendations = [v for v in candidate_videos 
                      if v not in user_videos]
    
    # Rank by popularity among similar users
    return rank_videos(recommendations)
```

**5. Resume Playback:**
```python
# Store watch progress in cache
def update_watch_progress(profile_id, video_id, progress_seconds):
    cache_key = f"progress:{profile_id}:{video_id}"
    redis.setex(cache_key, 3600, progress_seconds)  # 1 hour TTL
    
    # Async update to DB
    queue.push({
        'profile_id': profile_id,
        'video_id': video_id,
        'progress': progress_seconds
    })
```

#### Optimization Techniques

**1. Predictive Caching:**
- Predict what user will watch next
- Pre-cache first few segments
- Reduce buffering time

**2. Thumbnail Generation:**
- Generate at multiple resolutions
- WebP format for compression
- Lazy load on scroll

**3. Database Optimization:**
- Partition watch_history by date
- Archive old data
- Use read replicas

**4. Search Optimization:**
- Elasticsearch for full-text search
- Autocomplete suggestions
- Fuzzy matching

---

# Part 4: Low-Level Design (LLD) Problems

## 4.1 Design a Parking Lot System

### Requirements
- Multiple floors
- Different vehicle types (bike, car, truck)
- Different parking spot sizes
- Entry/exit management
- Fee calculation

### Solution

```python
from enum import Enum
from datetime import datetime
from typing import List, Optional

class VehicleType(Enum):
    BIKE = 1
    CAR = 2
    TRUCK = 3

class SpotSize(Enum):
    COMPACT = 1  # Bikes
    MEDIUM = 2   # Cars
    LARGE = 3    # Trucks

class Vehicle:
    def __init__(self, license_plate: str, vehicle_type: VehicleType):
        self.license_plate = license_plate
        self.vehicle_type = vehicle_type

class ParkingSpot:
    def __init__(self, spot_id: int, size: SpotSize):
        self.spot_id = spot_id
        self.size = size
        self.vehicle: Optional[Vehicle] = None
        self.is_occupied = False
    
    def can_fit_vehicle(self, vehicle: Vehicle) -> bool:
        if self.is_occupied:
            return False
        
        # Map vehicle types to required spot sizes
        required_size = {
            VehicleType.BIKE: SpotSize.COMPACT,
            VehicleType.CAR: SpotSize.MEDIUM,
            VehicleType.TRUCK: SpotSize.LARGE
        }
        
        return self.size.value >= required_size[vehicle.vehicle_type].value
    
    def park_vehicle(self, vehicle: Vehicle) -> bool:
        if self.can_fit_vehicle(vehicle):
            self.vehicle = vehicle
            self.is_occupied = True
            return True
        return False
    
    def remove_vehicle(self) -> Optional[Vehicle]:
        if self.is_occupied:
            vehicle = self.vehicle
            self.vehicle = None
            self.is_occupied = False
            return vehicle
        return None

class Floor:
    def __init__(self, floor_number: int):
        self.floor_number = floor_number
        self.spots: List[ParkingSpot] = []
    
    def add_spot(self, spot: ParkingSpot):
        self.spots.append(spot)
    
    def find_available_spot(self, vehicle: Vehicle) -> Optional[ParkingSpot]:
        for spot in self.spots:
            if spot.can_fit_vehicle(vehicle):
                return spot
        return None
    
    def get_available_count(self, size: SpotSize) -> int:
        return sum(1 for spot in self.spots 
                  if not spot.is_occupied and spot.size == size)

class Ticket:
    _ticket_counter = 0
    
    def __init__(self, vehicle: Vehicle, spot: ParkingSpot):
        Ticket._ticket_counter += 1
        self.ticket_id = Ticket._ticket_counter
        self.vehicle = vehicle
        self.spot = spot
        self.entry_time = datetime.now()
        self.exit_time: Optional[datetime] = None
        self.fee = 0.0
    
    def calculate_fee(self) -> float:
        if self.exit_time is None:
            self.exit_time = datetime.now()
        
        duration_hours = (self.exit_time - self.entry_time).total_seconds() / 3600
        
        # Hourly rates based on vehicle type
        rates = {
            VehicleType.BIKE: 10,
            VehicleType.CAR: 20,
            VehicleType.TRUCK: 40
        }
        
        self.fee = duration_hours * rates[self.vehicle.vehicle_type]
        return self.fee

class ParkingLot:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
    
    def __init__(self):
        if not hasattr(self, 'initialized'):
            self.floors: List[Floor] = []
            self.active_tickets: dict = {}  # ticket_id -> Ticket
            self.initialized = True
    
    def add_floor(self, floor: Floor):
        self.floors.append(floor)
    
    def park_vehicle(self, vehicle: Vehicle) -> Optional[Ticket]:
        # Find available spot across all floors
        for floor in self.floors:
            spot = floor.find_available_spot(vehicle)
            if spot:
                if spot.park_vehicle(vehicle):
                    ticket = Ticket(vehicle, spot)
                    self.active_tickets[ticket.ticket_id] = ticket
                    print(f"Vehicle {vehicle.license_plate} parked at Floor {floor.floor_number}, Spot {spot.spot_id}")
                    return ticket
        
        print("No available spot found!")
        return None
    
    def exit_vehicle(self, ticket_id: int) -> float:
        if ticket_id not in self.active_tickets:
            print("Invalid ticket!")
            return 0.0
        
        ticket = self.active_tickets[ticket_id]
        fee = ticket.calculate_fee()
        ticket.spot.remove_vehicle()
        del self.active_tickets[ticket_id]
        
        print(f"Vehicle {ticket.vehicle.license_plate} exited. Fee: ${fee:.2f}")
        return fee
    
    def display_availability(self):
        for floor in self.floors:
            print(f"\nFloor {floor.floor_number}:")
            for size in SpotSize:
                count = floor.get_available_count(size)
                print(f"  {size.name}: {count} spots available")

# Usage Example
if __name__ == "__main__":
    # Initialize parking lot
    parking_lot = ParkingLot()
    
    # Create floors
    for floor_num in range(1, 4):
        floor = Floor(floor_num)
        
        # Add spots to each floor
        for i in range(10):
            if i < 3:
                floor.add_spot(ParkingSpot(i, SpotSize.COMPACT))
            elif i < 8:
                floor.add_spot(ParkingSpot(i, SpotSize.MEDIUM))
            else:
                floor.add_spot(ParkingSpot(i, SpotSize.LARGE))
        
        parking_lot.add_floor(floor)
    
    # Park vehicles
    bike = Vehicle("BIKE-001", VehicleType.BIKE)
    car = Vehicle("CAR-001", VehicleType.CAR)
    truck = Vehicle("TRUCK-001", VehicleType.TRUCK)
    
    ticket1 = parking_lot.park_vehicle(bike)
    ticket2 = parking_lot.park_vehicle(car)
    ticket3 = parking_lot.park_vehicle(truck)
    
    # Display availability
    parking_lot.display_availability()
    
    # Exit vehicles
    if ticket1:
        parking_lot.exit_vehicle(ticket1.ticket_id)
    if ticket2:
        parking_lot.exit_vehicle(ticket2.ticket_id)
```

### Key Design Patterns Used
- **Singleton Pattern**: ParkingLot class
- **Strategy Pattern**: Fee calculation based on vehicle type
- **Factory Pattern**: Can be used for creating different types of spots

---

## 4.2 Design an ATM System

### Requirements
- Account authentication
- Check balance
- Withdraw cash
- Deposit cash
- Transfer money
- Transaction history

### Solution

```python
from enum import Enum
from datetime import datetime
from typing import List, Optional

class TransactionType(Enum):
    WITHDRAW = "WITHDRAW"
    DEPOSIT = "DEPOSIT"
    TRANSFER = "TRANSFER"
    BALANCE_INQUIRY = "BALANCE_INQUIRY"

class TransactionStatus(Enum):
    SUCCESS = "SUCCESS"
    FAILED = "FAILED"
    PENDING = "PENDING"

class Account:
    def __init__(self, account_number: str, pin: str, balance: float = 0.0):
        self.account_number = account_number
        self._pin = pin  # Private
        self.balance = balance
        self.transactions: List[Transaction] = []
    
    def verify_pin(self, pin: str) -> bool:
        return self._pin == pin
    
    def get_balance(self) -> float:
        return self.balance
    
    def debit(self, amount: float) -> bool:
        if self.balance >= amount:
            self.balance -= amount
            return True
        return False
    
    def credit(self, amount: float):
        self.balance += amount
    
    def add_transaction(self, transaction: 'Transaction'):
        self.transactions.append(transaction)
    
    def get_transaction_history(self, limit: int = 10) -> List['Transaction']:
        return self.transactions[-limit:]

class Transaction:
    _transaction_counter = 0
    
    def __init__(self, account: Account, transaction_type: TransactionType, amount: float):
        Transaction._transaction_counter += 1
        self.transaction_id = Transaction._transaction_counter
        self.account = account
        self.transaction_type = transaction_type
        self.amount = amount
        self.timestamp = datetime.now()
        self.status = TransactionStatus.PENDING
    
    def execute(self) -> bool:
        # Implementation depends on transaction type
        pass
    
    def __str__(self):
        return f"[{self.transaction_id}] {self.transaction_type.value} - ${self.amount:.2f} - {self.status.value} - {self.timestamp}"

class WithdrawTransaction(Transaction):
    def __init__(self, account: Account, amount: float):
        super().__init__(account, TransactionType.WITHDRAW, amount)
    
    def execute(self) -> bool:
        if self.account.debit(self.amount):
            self.status = TransactionStatus.SUCCESS
            self.account.add_transaction(self)
            return True
        self.status = TransactionStatus.FAILED
        return False

class DepositTransaction(Transaction):
    def __init__(self, account: Account, amount: float):
        super().__init__(account, TransactionType.DEPOSIT, amount)
    
    def execute(self) -> bool:
        self.account.credit(self.amount)
        self.status = TransactionStatus.SUCCESS
        self.account.add_transaction(self)
        return True

class TransferTransaction(Transaction):
    def __init__(self, from_account: Account, to_account: Account, amount: float):
        super().__init__(from_account, TransactionType.TRANSFER, amount)
        self.to_account = to_account
    
    def execute(self) -> bool:
        if self.account.debit(self.amount):
            self.to_account.credit(self.amount)
            self.status = TransactionStatus.SUCCESS
            self.account.add_transaction(self)
            self.to_account.add_transaction(self)
            return True
        self.status = TransactionStatus.FAILED
        return False

class Card:
    def __init__(self, card_number: str, account: Account):
        self.card_number = card_number
        self.account = account
        self.is_blocked = False
    
    def block(self):
        self.is_blocked = True
    
    def unblock(self):
        self.is_blocked = False

class CashDispenser:
    def __init__(self, initial_cash: float = 10000.0):
        self.total_cash = initial_cash
    
    def dispense_cash(self, amount: float) -> bool:
        if self.can_dispense(amount):
            self.total_cash -= amount
            print(f"Dispensing ${amount:.2f}")
            return True
        print("Insufficient cash in ATM")
        return False
    
    def can_dispense(self, amount: float) -> bool:
        return self.total_cash >= amount
    
    def get_total_cash(self) -> float:
        return self.total_cash

class ATM:
    MAX_PIN_ATTEMPTS = 3
    
    def __init__(self, atm_id: str, cash_dispenser: CashDispenser):
        self.atm_id = atm_id
        self.cash_dispenser = cash_dispenser
        self.current_card: Optional[Card] = None
        self.current_account: Optional[Account] = None
        self.pin_attempts = 0
    
    def insert_card(self, card: Card) -> bool:
        if card.is_blocked:
            print("Card is blocked!")
            return False
        
        self.current_card = card
        self.pin_attempts = 0
        print("Card inserted successfully")
        return True
    
    def authenticate(self, pin: str) -> bool:
        if not self.current_card:
            print("No card inserted")
            return False
        
        if self.current_card.account.verify_pin(pin):
            self.current_account = self.current_card.account
            print("Authentication successful")
            return True
        
        self.pin_attempts += 1
        if self.pin_attempts >= self.MAX_PIN_ATTEMPTS:
            self.current_card.block()
            print("Card blocked due to multiple incorrect PIN attempts")
            self.eject_card()
        else:
            print(f"Incorrect PIN. {self.MAX_PIN_ATTEMPTS - self.pin_attempts} attempts remaining")
        
        return False
    
    def check_balance(self) -> Optional[float]:
        if not self.current_account:
            print("Please authenticate first")
            return None
        
        balance = self.current_account.get_balance()
        print(f"Current balance: ${balance:.2f}")
        return balance
    
    def withdraw(self, amount: float) -> bool:
        if not self.current_account:
            print("Please authenticate first")
            return False
        
        if amount <= 0:
            print("Invalid amount")
            return False
        
        if not self.cash_dispenser.can_dispense(amount):
            print("ATM doesn't have enough cash")
            return False
        
        transaction = WithdrawTransaction(self.current_account, amount)
        if transaction.execute():
            self.cash_dispenser.dispense_cash(amount)
            print(f"Withdrawal successful: ${amount:.2f}")
            return True
        
        print("Insufficient balance")
        return False
    
    def deposit(self, amount: float) -> bool:
        if not self.current_account:
            print("Please authenticate first")
            return False
        
        if amount <= 0:
            print("Invalid amount")
            return False
        
        transaction = DepositTransaction(self.current_account, amount)
        if transaction.execute():
            print(f"Deposit successful: ${amount:.2f}")
            return True
        
        return False
    
    def transfer(self, to_account: Account, amount: float) -> bool:
        if not self.current_account:
            print("Please authenticate first")
            return False
        
        if amount <= 0:
            print("Invalid amount")
            return False
        
        transaction = TransferTransaction(self.current_account, to_account, amount)
        if transaction.execute():
            print(f"Transfer successful: ${amount:.2f} to {to_account.account_number}")
            return True
        
        print("Transfer failed - insufficient balance")
        return False
    
    def print_transaction_history(self):
        if not self.current_account:
            print("Please authenticate first")
            return
        
        print("\n=== Transaction History ===")
        history = self.current_account.get_transaction_history()
        for txn in history:
            print(txn)
    
    def eject_card(self):
        self.current_card = None
        self.current_account = None
        self.pin_attempts = 0
        print("Card ejected")

# Usage Example
if __name__ == "__main__":
    # Create accounts
    account1 = Account("ACC001", "1234", 1000.0)
    account2 = Account("ACC002", "5678", 500.0)
    
    # Create cards
    card1 = Card("CARD001", account1)
    
    # Create ATM
    cash_dispenser = CashDispenser(10000.0)
    atm = ATM("ATM001", cash_dispenser)
    
    # Use ATM
    atm.insert_card(card1)
    atm.authenticate("1234")
    
    atm.check_balance()
    atm.withdraw(200)
    atm.deposit(100)
    atm.transfer(account2, 150)
    atm.check_balance()
    
    atm.print_transaction_history()
    atm.eject_card()
```

### Key Design Patterns Used
- **State Pattern**: ATM states (card inserted, authenticated, etc.)
- **Command Pattern**: Different transaction types
- **Singleton Pattern**: Can be used for CashDispenser
- **Template Method Pattern**: Transaction execution flow

---

## 4.3 Design a Library Management System

### Requirements
- Book inventory management
- Member management
- Book checkout/return
- Search books
- Fine calculation for late returns
- Multiple copies of same book

### Solution

```python
from enum import Enum
from datetime import datetime, timedelta
from typing import List, Optional, Dict

class BookStatus(Enum):
    AVAILABLE = "AVAILABLE"
    RESERVED = "RESERVED"
    LOANED = "LOANED"
    LOST = "LOST"

class MembershipType(Enum):
    BASIC = "BASIC"
    PREMIUM = "PREMIUM"

class Book:
    def __init__(self, isbn: str, title: str, authors: List[str], publisher: str, 
                 publication_year: int):
        self.isbn = isbn
        self.title = title
        self.authors = authors
        self.publisher = publisher
        self.publication_year = publication_year

class BookItem:
    def __init__(self, barcode: str, book: Book, rack_location: str):
        self.barcode = barcode
        self.book = book
        self.rack_location = rack_location
        self.status = BookStatus.AVAILABLE
        self.borrowed_date: Optional[datetime] = None
        self.due_date: Optional[datetime] = None
    
    def checkout(self, days: int = 14):
        if self.status == BookStatus.AVAILABLE:
            self.status = BookStatus.LOANED
            self.borrowed_date = datetime.now()
            self.due_date = self.borrowed_date + timedelta(days=days)
            return True
        return False
    
    def return_book(self):
        self.status = BookStatus.AVAILABLE
        self.borrowed_date = None
        self.due_date = None
    
    def is_overdue(self) -> bool:
        if self.due_date:
            return datetime.now() > self.due_date
        return False
    
    def calculate_fine(self, fine_per_day: float = 1.0) -> float:
        if self.is_overdue() and self.due_date:
            days_overdue = (datetime.now() - self.due_date).days
            return days_overdue * fine_per_day
        return 0.0

class Member:
    def __init__(self, member_id: str, name: str, email: str, 
                 membership_type: MembershipType = MembershipType.BASIC):
        self.member_id = member_id
        self.name = name
        self.email = email
        self.membership_type = membership_type
        self.borrowed_books: List[BookItem] = []
        self.total_fine = 0.0
        self.max_books_allowed = 5 if membership_type == MembershipType.BASIC else 10
    
    def can_borrow_more(self) -> bool:
        return len(self.borrowed_books) < self.max_books_allowed
    
    def borrow_book(self, book_item: BookItem) -> bool:
        if self.can_borrow_more():
            self.borrowed_books.append(book_item)
            return True
        return False
    
    def return_book(self, book_item: BookItem):
        if book_item in self.borrowed_books:
            fine = book_item.calculate_fine()
            self.total_fine += fine
            self.borrowed_books.remove(book_item)
            return fine
        return 0.0

class Librarian:
    def __init__(self, employee_id: str, name: str):
        self.employee_id = employee_id
        self.name = name
    
    def add_book_item(self, catalog: 'Catalog', book_item: BookItem):
        catalog.add_book_item(book_item)
    
    def block_member(self, member: Member):
        # Implementation for blocking a member
        pass

class Catalog:
    def __init__(self):
        self.books: Dict[str, Book] = {}  # ISBN -> Book
        self.book_items: Dict[str, BookItem] = {}  # Barcode -> BookItem
        self.title_index: Dict[str, List[str]] = {}  # Title -> List of ISBNs
        self.author_index: Dict[str, List[str]] = {}  # Author -> List of ISBNs
    
    def add_book(self, book: Book):
        if book.isbn not in self.books:
            self.books[book.isbn] = book
            
            # Update title index
            if book.title not in self.title_index:
                self.title_index[book.title] = []
            self.title_index[book.title].append(book.isbn)
            
            # Update author index
            for author in book.authors:
                if author not in self.author_index:
                    self.author_index[author] = []
                self.author_index[author].append(book.isbn)
    
    def add_book_item(self, book_item: BookItem):
        # Add the book if not already in catalog
        self.add_book(book_item.book)
        
        # Add the book item
        self.book_items[book_item.barcode] = book_item
    
    def search_by_title(self, title: str) -> List[BookItem]:
        results = []
        for book_title, isbns in self.title_index.items():
            if title.lower() in book_title.lower():
                for isbn in isbns:
                    # Get all book items for this ISBN
                    for barcode, item in self.book_items.items():
                        if item.book.isbn == isbn:
                            results.append(item)
        return results
    
    def search_by_author(self, author: str) -> List[BookItem]:
        results = []
        for auth, isbns in self.author_index.items():
            if author.lower() in auth.lower():
                for isbn in isbns:
                    for barcode, item in self.book_items.items():
                        if item.book.isbn == isbn:
                            results.append(item)
        return results
    
    def search_by_isbn(self, isbn: str) -> List[BookItem]:
        results = []
        for barcode, item in self.book_items.items():
            if item.book.isbn == isbn:
                results.append(item)
        return results

class Library:
    def __init__(self, name: str):
        self.name = name
        self.catalog = Catalog()
        self.members: Dict[str, Member] = {}
    
    def add_member(self, member: Member):
        self.members[member.member_id] = member
    
    def checkout_book(self, member_id: str, barcode: str) -> bool:
        if member_id not in self.members:
            print("Member not found")
            return False
        
        if barcode not in self.catalog.book_items:
            print("Book not found")
            return False
        
        member = self.members[member_id]
        book_item = self.catalog.book_items[barcode]
        
        if not member.can_borrow_more():
            print("Maximum book limit reached")
            return False
        
        if book_item.status != BookStatus.AVAILABLE:
            print("Book is not available")
            return False
        
        # Checkout process
        loan_days = 14 if member.membership_type == MembershipType.BASIC else 21
        if book_item.checkout(loan_days):
            if member.borrow_book(book_item):
                print(f"Book '{book_item.book.title}' checked out successfully")
                print(f"Due date: {book_item.due_date}")
                return True
        
        return False
    
    def return_book(self, member_id: str, barcode: str) -> float:
        if member_id not in self.members:
            print("Member not found")
            return 0.0
        
        if barcode not in self.catalog.book_items:
            print("Book not found")
            return 0.0
        
        member = self.members[member_id]
        book_item = self.catalog.book_items[barcode]
        
        fine = member.return_book(book_item)
        book_item.return_book()
        
        if fine > 0:
            print(f"Book returned with fine: ${fine:.2f}")
        else:
            print("Book returned successfully")
        
        return fine
    
    def search_books(self, query: str, search_type: str = "title") -> List[BookItem]:
        if search_type == "title":
            return self.catalog.search_by_title(query)
        elif search_type == "author":
            return self.catalog.search_by_author(query)
        elif search_type == "isbn":
            return self.catalog.search_by_isbn(query)
        return []

# Usage Example
if __name__ == "__main__":
    # Create library
    library = Library("City Central Library")
    
    # Add books
    book1 = Book("978-0-13-468599-1", "Clean Code", ["Robert C. Martin"], 
                 "Prentice Hall", 2008)
    book2 = Book("978-0-13-235088-4", "Clean Architecture", ["Robert C. Martin"], 
                 "Prentice Hall", 2017)
    
    # Add book items (multiple copies)
    item1 = BookItem("BC001", book1, "A1-S2-R3")
    item2 = BookItem("BC002", book1, "A1-S2-R4")
    item3 = BookItem("BC003", book2, "A1-S3-R1")
    
    library.catalog.add_book_item(item1)
    library.catalog.add_book_item(item2)
    library.catalog.add_book_item(item3)
    
    # Add members
    member1 = Member("M001", "John Doe", "john@email.com", MembershipType.BASIC)
    member2 = Member("M002", "Jane Smith", "jane@email.com", MembershipType.PREMIUM)
    
    library.add_member(member1)
    library.add_member(member2)
    
    # Search books
    print("=== Searching for books by author 'Martin' ===")
    results = library.search_books("Martin", "author")
    for item in results:
        print(f"{item.book.title} - {item.barcode} - {item.status.value}")
    
    # Checkout books
    print("\n=== Checkout Operations ===")
    library.checkout_book("M001", "BC001")
    library.checkout_book("M002", "BC003")
    
    # Simulate overdue
    item1.due_date = datetime.now() - timedelta(days=5)
    
    # Return books
    print("\n=== Return Operations ===")
    library.return_book("M001", "BC001")
    library.return_book("M002", "BC003")
    
    print(f"\nMember {member1.name} total fines: ${member1.total_fine:.2f}")
```

### Key Design Patterns Used
- **Facade Pattern**: Library class provides simple interface
- **Composite Pattern**: Book and BookItem relationship
- **Strategy Pattern**: Different membership types with different rules
- **Observer Pattern**: Can be used for notifications

---

## 4.4 Design an Online Chess Game

### Requirements
- Two players
- Move validation
- Check/checkmate detection
- Castling, en passant
- Move history
- Timer (optional)

### Solution

```python
from enum import Enum
from typing import Optional, List, Tuple

class Color(Enum):
    WHITE = "WHITE"
    BLACK = "BLACK"

class PieceType(Enum):
    KING = "KING"
    QUEEN = "QUEEN"
    ROOK = "ROOK"
    BISHOP = "BISHOP"
    KNIGHT = "KNIGHT"
    PAWN = "PAWN"

class Position:
    def __init__(self, row: int, col: int):
        self.row = row
        self.col = col
    
    def is_valid(self) -> bool:
        return 0 <= self.row < 8 and 0 <= self.col < 8
    
    def __eq__(self, other):
        return self.row == other.row and self.col == other.col
    
    def __str__(self):
        return f"({chr(97 + self.col)}{8 - self.row})"

class Piece:
    def __init__(self, color: Color, piece_type: PieceType):
        self.color = color
        self.piece_type = piece_type
        self.has_moved = False
    
    def can_move(self, board: 'Board', start: Position, end: Position) -> bool:
        # Base validation
        if not end.is_valid():
            return False
        
        # Can't capture own piece
        end_piece = board.get_piece(end)
        if end_piece and end_piece.color == self.color:
            return False
        
        # Piece-specific movement logic
        return self._is_valid_move(board, start, end)
    
    def _is_valid_move(self, board: 'Board', start: Position, end: Position) -> bool:
        # Override in subclasses
        return False
    
    def __str__(self):
        symbols = {
            (Color.WHITE, PieceType.KING): '♔',
            (Color.WHITE, PieceType.QUEEN): '♕',
            (Color.WHITE, PieceType.ROOK): '♖',
            (Color.WHITE, PieceType.BISHOP): '♗',
            (Color.WHITE, PieceType.KNIGHT): '♘',
            (Color.WHITE, PieceType.PAWN): '♙',
            (Color.BLACK, PieceType.KING): '♚',
            (Color.BLACK, PieceType.QUEEN): '♛',
            (Color.BLACK, PieceType.ROOK): '♜',
            (Color.BLACK, PieceType.BISHOP): '♝',
            (Color.BLACK, PieceType.KNIGHT): '♞',
            (Color.BLACK, PieceType.PAWN): '♟',
        }
        return symbols.get((self.color, self.piece_type), '?')

class Pawn(Piece):
    def __init__(self, color: Color):
        super().__init__(color, PieceType.PAWN)
    
    def _is_valid_move(self, board: 'Board', start: Position, end: Position) -> bool:
        direction = -1 if self.color == Color.WHITE else 1
        start_row = 6 if self.color == Color.WHITE else 1
        
        # Move forward one square
        if end.col == start.col and end.row == start.row + direction:
            return board.get_piece(end) is None
        
        # Move forward two squares from starting position
        if (end.col == start.col and end.row == start.row + 2 * direction and
            start.row == start_row):
            middle = Position(start.row + direction, start.col)
            return board.get_piece(middle) is None and board.get_piece(end) is None
        
        # Capture diagonally
        if abs(end.col - start.col) == 1 and end.row == start.row + direction:
            return board.get_piece(end) is not None
        
        return False

class Rook(Piece):
    def __init__(self, color: Color):
        super().__init__(color, PieceType.ROOK)
    
    def _is_valid_move(self, board: 'Board', start: Position, end: Position) -> bool:
        # Must move in straight line
        if start.row != end.row and start.col != end.col:
            return False
        
        # Check path is clear
        return board.is_path_clear(start, end)

class Knight(Piece):
    def __init__(self, color: Color):
        super().__init__(color, PieceType.KNIGHT)
    
    def _is_valid_move(self, board: 'Board', start: Position, end: Position) -> bool:
        row_diff = abs(end.row - start.row)
        col_diff = abs(end.col - start.col)
        return (row_diff == 2 and col_diff == 1) or (row_diff == 1 and col_diff == 2)

class Bishop(Piece):
    def __init__(self, color: Color):
        super().__init__(color, PieceType.BISHOP)
    
    def _is_valid_move(self, board: 'Board', start: Position, end: Position) -> bool:
        # Must move diagonally
        if abs(end.row - start.row) != abs(end.col - start.col):
            return False
        
        return board.is_path_clear(start, end)

class Queen(Piece):
    def __init__(self, color: Color):
        super().__init__(color, PieceType.QUEEN)
    
    def _is_valid_move(self, board: 'Board', start: Position, end: Position) -> bool:
        # Moves like rook or bishop
        is_straight = start.row == end.row or start.col == end.col
        is_diagonal = abs(end.row - start.row) == abs(end.col - start.col)
        
        if not (is_straight or is_diagonal):
            return False
        
        return board.is_path_clear(start, end)

class King(Piece):
    def __init__(self, color: Color):
        super().__init__(color, PieceType.KING)
    
    def _is_valid_move(self, board: 'Board', start: Position, end: Position) -> bool:
        row_diff = abs(end.row - start.row)
        col_diff = abs(end.col - start.col)
        
        # Normal king move (one square in any direction)
        return row_diff <= 1 and col_diff <= 1

class Board:
    def __init__(self):
        self.squares = [[None for _ in range(8)] for _ in range(8)]
        self._initialize_board()
    
    def _initialize_board(self):
        # Place pawns
        for col in range(8):
            self.squares[1][col] = Pawn(Color.BLACK)
            self.squares[6][col] = Pawn(Color.WHITE)
        
        # Place other pieces
        piece_order = [Rook, Knight, Bishop, Queen, King, Bishop, Knight, Rook]
        
        for col, piece_class in enumerate(piece_order):
            self.squares[0][col] = piece_class(Color.BLACK)
            self.squares[7][col] = piece_class(Color.WHITE)
    
    def get_piece(self, position: Position) -> Optional[Piece]:
        if position.is_valid():
            return self.squares[position.row][position.col]
        return None
    
    def set_piece(self, position: Position, piece: Optional[Piece]):
        if position.is_valid():
            self.squares[position.row][position.col] = piece
    
    def is_path_clear(self, start: Position, end: Position) -> bool:
        row_direction = 0 if start.row == end.row else (1 if end.row > start.row else -1)
        col_direction = 0 if start.col == end.col else (1 if end.col > start.col else -1)
        
        current = Position(start.row + row_direction, start.col + col_direction)
        
        while current != end:
            if self.get_piece(current) is not None:
                return False
            current.row += row_direction
            current.col += col_direction
        
        return True
    
    def display(self):
        print("  a b c d e f g h")
        for row in range(8):
            print(f"{8 - row} ", end="")
            for col in range(8):
                piece = self.squares[row][col]
                if piece:
                    print(piece, end=" ")
                else:
                    print(".", end=" ")
            print(f"{8 - row}")
        print("  a b c d e f g h")

class Move:
    def __init__(self, start: Position, end: Position, piece: Piece, 
                 captured_piece: Optional[Piece] = None):
        self.start = start
        self.end = end
        self.piece = piece
        self.captured_piece = captured_piece
        self.timestamp = None
    
    def __str__(self):
        capture = "x" if self.captured_piece else "-"
        return f"{self.piece.piece_type.value}{self.start}{capture}{self.end}"

class Player:
    def __init__(self, name: str, color: Color):
        self.name = name
        self.color = color

class Game:
    def __init__(self, white_player: Player, black_player: Player):
        self.board = Board()
        self.white_player = white_player
        self.black_player = black_player
        self.current_turn = Color.WHITE
        self.move_history: List[Move] = []
        self.game_over = False
    
    def make_move(self, start: Position, end: Position) -> bool:
        piece = self.board.get_piece(start)
        
        # Validate piece exists and belongs to current player
        if not piece or piece.color != self.current_turn:
            print("Invalid piece selection")
            return False
        
        # Validate move
        if not piece.can_move(self.board, start, end):
            print("Invalid move")
            return False
        
        # Make the move
        captured_piece = self.board.get_piece(end)
        self.board.set_piece(end, piece)
        self.board.set_piece(start, None)
        piece.has_moved = True
        
        # Record move
        move = Move(start, end, piece, captured_piece)
        self.move_history.append(move)
        
        # Switch turn
        self.current_turn = Color.BLACK if self.current_turn == Color.WHITE else Color.WHITE
        
        print(f"Move: {move}")
        return True
    
    def display_board(self):
        self.board.display()
        print(f"\nCurrent turn: {self.current_turn.value}")
    
    def get_move_history(self) -> List[str]:
        return [str(move) for move in self.move_history]

# Usage Example
if __name__ == "__main__":
    # Create players
    player1 = Player("Alice", Color.WHITE)
    player2 = Player("Bob", Color.BLACK)
    
    # Create game
    game = Game(player1, player2)
    
    # Display initial board
    game.display_board()
    
    # Make some moves
    print("\n=== Game Start ===\n")
    
    # White pawn e2 to e4
    game.make_move(Position(6, 4), Position(4, 4))
    game.display_board()
    
    # Black pawn e7 to e5
    game.make_move(Position(1, 4), Position(3, 4))