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

### DROP and TRUNCATE
```sql
DROP TABLE IF EXISTS employees;
TRUNCATE TABLE employees; -- Faster than DELETE, resets auto-increment
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

-- Insert from SELECT
INSERT INTO employees_backup
SELECT * FROM employees WHERE hire_date < '2024-01-01';
```

### UPDATE
```sql
-- Update single record
UPDATE employees 
SET salary = 85000 
WHERE employee_id = 1;

-- Update with JOIN
UPDATE employees e
JOIN departments d ON e.department_id = d.department_id
SET e.salary = e.salary * 1.05
WHERE d.department_name = 'Sales';
```

### DELETE
```sql
DELETE FROM employees WHERE employee_id = 10;
DELETE FROM employees WHERE hire_date < '2020-01-01';
```

## 2.3 Data Query Language (DQL)

### SELECT Basics
```sql
SELECT * FROM employees;
SELECT first_name, last_name, salary FROM employees;
SELECT first_name AS "First Name", salary AS "Annual Salary" FROM employees;
SELECT DISTINCT department_id FROM employees;
```

### WHERE Clause
```sql
-- Comparison
SELECT * FROM employees WHERE salary > 75000;

-- Logical operators
SELECT * FROM employees 
WHERE salary > 70000 AND department_id = 5;

-- BETWEEN
SELECT * FROM employees WHERE salary BETWEEN 60000 AND 80000;

-- IN
SELECT * FROM employees WHERE department_id IN (1, 3, 5);

-- LIKE
SELECT * FROM employees WHERE last_name LIKE 'Sm%'; -- Starts with Sm
SELECT * FROM employees WHERE email LIKE '%@gmail.com'; -- Ends with
SELECT * FROM employees WHERE first_name LIKE '_ohn'; -- Matches John

-- NULL checks
SELECT * FROM employees WHERE manager_id IS NULL;
```

### ORDER BY and LIMIT
```sql
SELECT * FROM employees ORDER BY salary DESC;
SELECT * FROM employees ORDER BY department_id ASC, salary DESC;
SELECT * FROM employees LIMIT 10;
SELECT * FROM employees LIMIT 10 OFFSET 10; -- Pagination
```

## 2.4 Aggregate Functions

```sql
-- COUNT, SUM, AVG, MIN, MAX
SELECT COUNT(*) FROM employees;
SELECT SUM(salary) AS total_payroll FROM employees;
SELECT AVG(salary) AS average_salary FROM employees;
SELECT MIN(salary), MAX(salary) FROM employees;

-- Combined
SELECT 
    COUNT(*) AS total_employees,
    AVG(salary) AS avg_salary,
    MIN(salary) AS min_salary,
    MAX(salary) AS max_salary
FROM employees;
```

### GROUP BY
```sql
SELECT department_id, COUNT(*) AS employee_count
FROM employees
GROUP BY department_id;

SELECT department_id, job_title, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id, job_title
ORDER BY avg_salary DESC;
```

### HAVING
```sql
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 75000;

SELECT department_id, COUNT(*) AS count
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 5 AND AVG(salary) > 70000;
```

## 2.5 JOINS

### INNER JOIN
```sql
SELECT e.first_name, e.last_name, d.department_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.department_id;

-- Multiple joins
SELECT e.first_name, d.department_name, p.project_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.department_id
INNER JOIN projects p ON d.department_id = p.department_id;
```

### LEFT JOIN
```sql
SELECT e.first_name, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id;
```

### RIGHT JOIN
```sql
SELECT e.first_name, d.department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.department_id;
```

### FULL OUTER JOIN (MySQL alternative)
```sql
SELECT e.first_name, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id
UNION
SELECT e.first_name, d.department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.department_id;
```

### SELF JOIN
```sql
-- Employee and their manager
SELECT e1.first_name AS employee, e2.first_name AS manager
FROM employees e1
LEFT JOIN employees e2 ON e1.manager_id = e2.employee_id;
```

## 2.6 Subqueries

### WHERE Subquery
```sql
-- Employees earning more than average
SELECT first_name, last_name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

-- IN with subquery
SELECT first_name, last_name
FROM employees
WHERE department_id IN (
    SELECT department_id FROM departments 
    WHERE department_name IN ('Sales', 'Marketing')
);
```

### SELECT Subquery
```sql
SELECT 
    first_name,
    salary,
    (SELECT AVG(salary) FROM employees) AS avg_salary,
    salary - (SELECT AVG(salary) FROM employees) AS diff_from_avg
FROM employees;
```

### FROM Subquery (Derived Table)
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
SELECT e1.first_name, e1.salary
FROM employees e1
WHERE salary > (
    SELECT AVG(salary)
    FROM employees e2
    WHERE e2.department_id = e1.department_id
);
```

## 2.7 Set Operations

### UNION and UNION ALL
```sql
-- UNION (removes duplicates)
SELECT first_name FROM employees WHERE department_id = 1
UNION
SELECT first_name FROM employees WHERE department_id = 2;

-- UNION ALL (keeps duplicates)
SELECT first_name FROM employees WHERE salary > 70000
UNION ALL
SELECT first_name FROM employees WHERE hire_date > '2024-01-01';
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

-- REPLACE
SELECT REPLACE(email, '@company.com', '@newdomain.com') FROM employees;

-- TRIM
SELECT TRIM('  hello  ') AS trimmed;
```

## 2.9 Date and Time Functions

```sql
-- Current date/time
SELECT NOW(), CURDATE(), CURTIME();

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
    TIMESTAMPDIFF(YEAR, hire_date, NOW()) AS years_employed
FROM employees;
```

## 2.10 Conditional Functions

### CASE Statement
```sql
SELECT 
    first_name,
    salary,
    CASE 
        WHEN salary >= 100000 THEN 'High'
        WHEN salary >= 70000 THEN 'Medium'
        ELSE 'Low'
    END AS salary_grade
FROM employees;
```

### IF and COALESCE
```sql
-- IF
SELECT first_name, IF(salary > 75000, 'High', 'Low') AS category
FROM employees;

-- COALESCE (returns first non-NULL)
SELECT first_name, COALESCE(manager_id, 0) AS manager_id
FROM employees;
```

## 2.11 Window Functions

### ROW_NUMBER, RANK, DENSE_RANK
```sql
SELECT 
    first_name,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num,
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
    salary,
    LAG(salary) OVER (ORDER BY hire_date) AS previous_salary,
    LEAD(salary) OVER (ORDER BY hire_date) AS next_salary
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

### Recursive CTE
```sql
-- Employee hierarchy
WITH RECURSIVE employee_hierarchy AS (
    SELECT employee_id, first_name, manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    SELECT e.employee_id, e.first_name, e.manager_id, eh.level + 1
    FROM employees e
    INNER JOIN employee_hierarchy eh ON e.manager_id = eh.employee_id
)
SELECT * FROM employee_hierarchy ORDER BY level;
```

## 2.13 Views

```sql
-- Create view
CREATE VIEW employee_details AS
SELECT 
    e.employee_id,
    CONCAT(e.first_name, ' ', e.last_name) AS full_name,
    d.department_name,
    e.salary
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id;

-- Use view
SELECT * FROM employee_details WHERE salary > 75000;

-- Drop view
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
- Redirect short URL to original
- Custom aliases (optional)
- Analytics

**Non-Functional:**
- High availability
- Low latency
- Scalable (billions of URLs)

### Solution

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
    INDEX idx_short_code (short_code)
);
```

#### Architecture
```
Client → Load Balancer → App Servers → Cache (Redis) + Database (MySQL)
                                     → Message Queue (Analytics)
```

#### Key Design Decisions

**1. Short Code Generation:**
- Base62 encoding [a-zA-Z0-9]
- 7 characters = 62^7 = 3.5 trillion URLs
- Convert auto-increment ID to base62

```python
def id_to_base62(id):
    chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    result = []
    while id > 0:
        result.append(chars[id % 62])
        id //= 62
    return ''.join(reversed(result))
```

**2. Caching:**
- Cache popular URLs in Redis
- LRU eviction policy

**3. Analytics:**
- Async processing via message queue
- Batch inserts

**4. Scaling:**
- Database sharding by hash of short_code
- Read replicas for scaling reads

---

## 3.2 Design Twitter

### Requirements
**Functional:**
- Post tweets (280 chars)
- Follow/unfollow users
- Timeline (home feed)
- Search tweets

**Non-Functional:**
- High availability
- Low latency
- Millions of users

### Database Schema
```sql
CREATE TABLE users (
    user_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
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
    PRIMARY KEY (follower_id, followee_id)
);

CREATE TABLE likes (
    user_id BIGINT,
    tweet_id BIGINT,
    PRIMARY KEY (user_id, tweet_id)
);
```

### Architecture
```
Client → Load Balancer → Tweet Service, Timeline Service, User Service
                      → Cache (Redis) + Database (Sharded)
```

### Timeline Generation Strategies

**1. Fanout on Write (Push):**
- When user tweets, push to all followers' timelines
- Fast reads, slow writes
- Good for users with few followers

**2. Fanout on Read (Pull):**
- Fetch tweets from followees on request
- Fast writes, slower reads
- Good for celebrity accounts

**3. Hybrid:**
- Fanout on write for most users
- Fanout on read for celebrities

### Key Components
- **Caching**: Timeline cache in Redis
- **Database Sharding**: By user_id
- **Search**: Elasticsearch for full-text search
- **Notifications**: Message queue + WebSockets

---

## 3.3 Design Instagram

### Requirements
**Functional:**
- Upload photos/videos
- Follow users
- Feed generation
- Likes, comments
- Stories (24-hour)

**Non-Functional:**
- High availability
- Low latency
- Handle large media files

### Database Schema
```sql
CREATE TABLE users (
    user_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    bio TEXT,
    profile_pic_url VARCHAR(500)
);

CREATE TABLE posts (
    post_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    media_url VARCHAR(500) NOT NULL,
    media_type ENUM('image', 'video'),
    caption TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    likes_count INT DEFAULT 0,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    INDEX idx_user_created (user_id, created_at)
);

CREATE TABLE follows (
    follower_id BIGINT,
    followee_id BIGINT,
    PRIMARY KEY (follower_id, followee_id)
);

CREATE TABLE stories (
    story_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    media_url VARCHAR(500) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    INDEX idx_expires (expires_at)
);
```

### Architecture
```
Client → CDN → Load Balancer → Upload Service, Feed Service, User Service
                             → Object Storage (S3) + Database + Cache
```

### Key Components

**1. Media Upload:**
- Client uploads to Upload Service
- Generate pre-signed S3 URL
- Direct upload to S3
- Async processing (thumbnails, compression)

**2. Feed Generation:**
- Hybrid approach (like Twitter)
- Pre-compute for most users
- On-demand for celebrity followers

**3. Stories:**
- Auto-delete after 24 hours (background job)
- Higher priority in feed

**4. Optimization:**
- Multiple image resolutions
- WebP format
- CDN for delivery
- Lazy loading

---

## 3.4 Design Uber

### Requirements
**Functional:**
- Rider requests ride
- Match with nearby driver
- Real-time tracking
- Fare calculation
- Ratings

**Non-Functional:**
- Low latency matching (<5s)
- Real-time updates
- Millions of concurrent users

### Database Schema
```sql
CREATE TABLE users (
    user_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    user_type ENUM('rider', 'driver')
);

CREATE TABLE drivers (
    driver_id BIGINT PRIMARY KEY,
    is_available BOOLEAN DEFAULT FALSE,
    current_lat DECIMAL(10, 8),
    current_lng DECIMAL(11, 8),
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
    FOREIGN KEY (rider_id) REFERENCES users(user_id),
    FOREIGN KEY (driver_id) REFERENCES users(user_id)
);
```

### Architecture
```
Client → Load Balancer → Rider Service, Driver Service, Matching Service
                      → WebSocket Server → Redis (Location Cache) → Database
```

### Key Components

**1. Location Tracking:**
```python
# Redis Geospatial
GEOADD drivers:locations <lng> <lat> <driver_id>
GEORADIUS drivers:locations <user_lng> <user_lat> 5 km
```

**2. Matching Algorithm:**
```python
def find_driver(rider_lat, rider_lng):
    nearby = get_nearby_drivers(rider_lat, rider_lng, radius=5)
    available = [d for d in nearby if d.is_available]
    
    best = sorted(available, 
                  key=lambda d: (
                      distance(rider_lat, rider_lng, d.lat, d.lng),
                      -d.rating
                  ))[0]
    return best
```

**3. Fare Calculation:**
```python
def calculate_fare(distance_km, duration_min, surge=1.0):
    BASE = 2.50
    PER_KM = 1.50
    PER_MIN = 0.30
    fare = (BASE + distance_km * PER_KM + duration_min * PER_MIN) * surge
    return max(fare, 5.00)  # Minimum fare
```

**4. Real-time Updates:**
- WebSocket for live location
- Push notifications

### Scaling Considerations
- Geospatial indexing (QuadTree, S2)
- Sharding by geographic region
- Surge pricing (supply/demand)
- Driver location caching

---

## 3.5 Design Netflix

### Requirements
**Functional:**
- Upload videos
- Stream videos
- Search content
- Recommendations
- Resume playback

**Non-Functional:**
- 99.99% availability
- Low latency streaming
- Millions of concurrent streams
- Global distribution

### Database Schema
```sql
CREATE TABLE users (
    user_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(100) UNIQUE NOT NULL,
    subscription_type ENUM('basic', 'standard', 'premium')
);

CREATE TABLE videos (
    video_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    description TEXT,
    duration_seconds INT,
    release_year INT,
    INDEX idx_title (title)
);

CREATE TABLE video_files (
    file_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    video_id BIGINT NOT NULL,
    quality ENUM('360p', '720p', '1080p', '4K'),
    file_url VARCHAR(500) NOT NULL,
    FOREIGN KEY (video_id) REFERENCES videos(video_id)
);

CREATE TABLE watch_history (
    profile_id BIGINT,
    video_id BIGINT,
    watched_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    progress_seconds INT,
    completed BOOLEAN DEFAULT FALSE,
    INDEX idx_profile_watched (profile_id, watched_at)
);
```

### Architecture
```
Client → CDN → Load Balancer → User Service, Streaming Service, Rec Service
                             → Cache (Redis) → Database (Sharded) → S3
```

### Key Components

**1. Video Processing:**
- Upload to S3
- Transcode to multiple formats
- Generate thumbnails
- Create subtitles

**2. Adaptive Bitrate Streaming:**
- HLS or DASH
- Auto quality switching
- Chunked segments (10s)

**3. CDN Strategy:**
- Global edge locations
- Cache popular content
- Prefetch trending shows

**4. Recommendations:**
```python
def get_recommendations(user_id):
    user_videos = get_watched_videos(user_id)
    similar_users = find_similar_users(user_videos)
    candidates = get_videos_from_users(similar_users)
    recommendations = [v for v in candidates if v not in user_videos]
    return rank_videos(recommendations)
```

**5. Resume Playback:**
```python
def update_progress(profile_id, video_id, progress):
    key = f"progress:{profile_id}:{video_id}"
    redis.setex(key, 3600, progress)
    queue.push({'profile_id': profile_id, 'progress': progress})
```

### Optimization
- Predictive caching
- Multiple resolutions
- Thumbnail optimization
- Elasticsearch for search

---

# Part 4: Low-Level Design (LLD) Problems

## 4.1 Design a Parking Lot System

### Requirements
- Multiple floors
- Different vehicle types (bike, car, truck)
- Different spot sizes
- Entry/exit management
- Fee calculation

### Class Diagram
```
ParkingLot (Singleton)
├── Floor[]
│   └── ParkingSpot[]
│       └── Vehicle
├── Ticket[]
└── calculateFee()
```

### Solution (Python)

```python
from enum import Enum
from datetime import datetime

class VehicleType(Enum):
    BIKE = 1
    CAR = 2
    TRUCK = 3

class SpotSize(Enum):
    COMPACT = 1  # Bikes
    MEDIUM = 2   # Cars
    LARGE = 3    # Trucks

class Vehicle:
    def __init__(self, license_plate, vehicle_type):
        self.license_plate = license_plate
        self.vehicle_type = vehicle_type

class ParkingSpot:
    def __init__(self, spot_id, size):
        self.spot_id = spot_id
        self.size = size
        self.vehicle = None
        self.is_occupied = False
    
    def can_fit_vehicle(self, vehicle):
        if self.is_occupied:
            return False
        
        required_size = {
            VehicleType.BIKE: SpotSize.COMPACT,
            VehicleType.CAR: SpotSize.MEDIUM,
            VehicleType.TRUCK: SpotSize.LARGE
        }
        
        return self.size.value >= required_size[vehicle.vehicle_type].value
    
    def park_vehicle(self, vehicle):
        if self.can_fit_vehicle(vehicle):
            self.vehicle = vehicle
            self.is_occupied = True
            return True
        return False
    
    def remove_vehicle(self):
        if self.is_occupied:
            vehicle = self.vehicle
            self.vehicle = None
            self.is_occupied = False
            return vehicle
        return None

class Floor:
    def __init__(self, floor_number):
        self.floor_number = floor_number
        self.spots = []
    
    def add_spot(self, spot):
        self.spots.append(spot)
    
    def find_available_spot(self, vehicle):
        for spot in self.spots:
            if spot.can_fit_vehicle(vehicle):
                return spot
        return None

class Ticket:
    _counter = 0
    
    def __init__(self, vehicle, spot):
        Ticket._counter += 1
        self.ticket_id = Ticket._counter
        self.vehicle = vehicle
        self.spot = spot
        self.entry_time = datetime.now()
        self.exit_time = None
        self.fee = 0.0
    
    def calculate_fee(self):
        if not self.exit_time:
            self.exit_time = datetime.now()
        
        hours = (self.exit_time - self.entry_time).total_seconds() / 3600
        
        rates = {
            VehicleType.BIKE: 10,
            VehicleType.CAR: 20,
            VehicleType.TRUCK: 40
        }
        
        self.fee = hours * rates[self.vehicle.vehicle_type]
        return self.fee

class ParkingLot:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
    
    def __init__(self):
        if not hasattr(self, 'initialized'):
            self.floors = []
            self.active_tickets = {}
            self.initialized = True
    
    def add_floor(self, floor):
        self.floors.append(floor)
    
    def park_vehicle(self, vehicle):
        for floor in self.floors:
            spot = floor.find_available_spot(vehicle)
            if spot:
                if spot.park_vehicle(vehicle):
                    ticket = Ticket(vehicle, spot)
                    self.active_tickets[ticket.ticket_id] = ticket
                    return ticket
        return None
    
    def exit_vehicle(self, ticket_id):
        if ticket_id not in self.active_tickets:
            return 0.0
        
        ticket = self.active_tickets[ticket_id]
        fee = ticket.calculate_fee()
        ticket.spot.remove_vehicle()
        del self.active_tickets[ticket_id]
        return fee
```

### Key Design Patterns
- **Singleton**: ParkingLot class
- **Strategy**: Fee calculation based on vehicle type

---

## 4.2 Design an ATM System

### Requirements
- Authentication
- Check balance
- Withdraw cash
- Deposit cash
- Transfer money
- Transaction history

### Class Diagram
```
ATM
├── CashDispenser
├── Card
│   └── Account
└── Transaction[]
    ├── WithdrawTransaction
    ├── DepositTransaction
    └── TransferTransaction
```

### Solution (Python)

```python
from enum import Enum
from datetime import datetime

class TransactionType(Enum):
    WITHDRAW = "WITHDRAW"
    DEPOSIT = "DEPOSIT"
    TRANSFER = "TRANSFER"

class TransactionStatus(Enum):
    SUCCESS = "SUCCESS"
    FAILED = "FAILED"

class Account:
    def __init__(self, account_number, pin, balance=0.0):
        self.account_number = account_number
        self._pin = pin  # Private
        self.balance = balance
        self.transactions = []
    
    def verify_pin(self, pin):
        return self._pin == pin
    
    def debit(self, amount):
        if self.balance >= amount:
            self.balance -= amount
            return True
        return False
    
    def credit(self, amount):
        self.balance += amount
    
    def add_transaction(self, transaction):
        self.transactions.append(transaction)

class Transaction:
    _counter = 0
    
    def __init__(self, account, txn_type, amount):
        Transaction._counter += 1
        self.transaction_id = Transaction._counter
        self.account = account
        self.transaction_type = txn_type
        self.amount = amount
        self.timestamp = datetime.now()
        self.status = TransactionStatus.FAILED
    
    def execute(self):
        pass  # Override in subclasses

class WithdrawTransaction(Transaction):
    def __init__(self, account, amount):
        super().__init__(account, TransactionType.WITHDRAW, amount)
    
    def execute(self):
        if self.account.debit(self.amount):
            self.status = TransactionStatus.SUCCESS
            self.account.add_transaction(self)
            return True
        return False

class DepositTransaction(Transaction):
    def __init__(self, account, amount):
        super().__init__(account, TransactionType.DEPOSIT, amount)
    
    def execute(self):
        self.account.credit(self.amount)
        self.status = TransactionStatus.SUCCESS
        self.account.add_transaction(self)
        return True

class TransferTransaction(Transaction):
    def __init__(self, from_account, to_account, amount):
        super().__init__(from_account, TransactionType.TRANSFER, amount)
        self.to_account = to_account
    
    def execute(self):
        if self.account.debit(self.amount):
            self.to_account.credit(self.amount)
            self.status = TransactionStatus.SUCCESS
            self.account.add_transaction(self)
            self.to_account.add_transaction(self)
            return True
        return False

class Card:
    def __init__(self, card_number, account):
        self.card_number = card_number
        self.account = account
        self.is_blocked = False

class CashDispenser:
    def __init__(self, initial_cash=10000.0):
        self.total_cash = initial_cash
    
    def dispense_cash(self, amount):
        if self.total_cash >= amount:
            self.total_cash -= amount
            return True
        return False

class ATM:
    MAX_PIN_ATTEMPTS = 3
    
    def __init__(self, atm_id, cash_dispenser):
        self.atm_id = atm_id
        self.cash_dispenser = cash_dispenser
        self.current_card = None
        self.current_account = None
        self.pin_attempts = 0
    
    def insert_card(self, card):
        if card.is_blocked:
            return False
        self.current_card = card
        self.pin_attempts = 0
        return True
    
    def authenticate(self, pin):
        if not self.current_card:
            return False
        
        if self.current_card.account.verify_pin(pin):
            self.current_account = self.current_card.account
            return True
        
        self.pin_attempts += 1
        if self.pin_attempts >= self.MAX_PIN_ATTEMPTS:
            self.current_card.is_blocked = True
            self.eject_card()
        return False
    
    def check_balance(self):
        if not self.current_account:
            return None
        return self.current_account.balance
    
    def withdraw(self, amount):
        if not self.current_account:
            return False
        
        if not self.cash_dispenser.dispense_cash(amount):
            return False
        
        txn = WithdrawTransaction(self.current_account, amount)
        return txn.execute()
    
    def deposit(self, amount):
        if not self.current_account:
            return False
        
        txn = DepositTransaction(self.current_account, amount)
        return txn.execute()
    
    def transfer(self, to_account, amount):
        if not self.current_account:
            return False
        
        txn = TransferTransaction(self.current_account, to_account, amount)
        return txn.execute()
    
    def eject_card(self):
        self.current_card = None
        self.current_account = None
        self.pin_attempts = 0
```

### Key Design Patterns
- **State Pattern**: ATM states
- **Command Pattern**: Transaction types
- **Template Method**: Transaction execution

---

## 4.3 Design a Library Management System

### Requirements
- Book inventory
- Member management
- Checkout/return
- Search books
- Fine calculation

### Class Diagram
```
Library
├── Catalog
│   ├── Book
│   └── BookItem[]
├── Member[]
└── Librarian

BookItem: has status (Available, Loaned, Reserved)
Member: can borrow/return books
```

### Solution (Python)

```python
from enum import Enum
from datetime import datetime, timedelta

class BookStatus(Enum):
    AVAILABLE = "AVAILABLE"
    LOANED = "LOANED"
    RESERVED = "RESERVED"

class Book:
    def __init__(self, isbn, title, authors):
        self.isbn = isbn
        self.title = title
        self.authors = authors

class BookItem:
    def __init__(self, barcode, book, rack_location):
        self.barcode = barcode
        self.book = book
        self.rack_location = rack_location
        self.status = BookStatus.AVAILABLE
        self.borrowed_date = None
        self.due_date = None
    
    def checkout(self, days=14):
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
    
    def is_overdue(self):
        if self.due_date:
            return datetime.now() > self.due_date
        return False
    
    def calculate_fine(self, fine_per_day=1.0):
        if self.is_overdue():
            days = (datetime.now() - self.due_date).days
            return days * fine_per_day
        return 0.0

class Member:
    def __init__(self, member_id, name, max_books=5):
        self.member_id = member_id
        self.name = name
        self.borrowed_books = []
        self.total_fine = 0.0
        self.max_books_allowed = max_books
    
    def can_borrow_more(self):
        return len(self.borrowed_books) < self.max_books_allowed
    
    def borrow_book(self, book_item):
        if self.can_borrow_more():
            self.borrowed_books.append(book_item)
            return True
        return False
    
    def return_book(self, book_item):
        if book_item in self.borrowed_books:
            fine = book_item.calculate_fine()
            self.total_fine += fine
            self.borrowed_books.remove(book_item)
            return fine
        return 0.0

class Catalog:
    def __init__(self):
        self.books = {}  # ISBN -> Book
        self.book_items = {}  # Barcode -> BookItem
        self.title_index = {}
        self.author_index = {}
    
    def add_book(self, book):
        if book.isbn not in self.books:
            self.books[book.isbn] = book
            
            if book.title not in self.title_index:
                self.title_index[book.title] = []
            self.title_index[book.title].append(book.isbn)
            
            for author in book.authors:
                if author not in self.author_index:
                    self.author_index[author] = []
                self.author_index[author].append(book.isbn)
    
    def add_book_item(self, book_item):
        self.add_book(book_item.book)
        self.book_items[book_item.barcode] = book_item
    
    def search_by_title(self, title):
        results = []
        for book_title, isbns in self.title_index.items():
            if title.lower() in book_title.lower():
                for isbn in isbns:
                    for item in self.book_items.values():
                        if item.book.isbn == isbn:
                            results.append(item)
        return results

class Library:
    def __init__(self, name):
        self.name = name
        self.catalog = Catalog()
        self.members = {}
    
    def add_member(self, member):
        self.members[member.member_id] = member
    
    def checkout_book(self, member_id, barcode):
        if member_id not in self.members:
            return False
        if barcode not in self.catalog.book_items:
            return False
        
        member = self.members[member_id]
        book_item = self.catalog.book_items[barcode]
        
        if not member.can_borrow_more():
            return False
        if book_item.status != BookStatus.AVAILABLE:
            return False
        
        if book_item.checkout():
            return member.borrow_book(book_item)
        return False
    
    def return_book(self, member_id, barcode):
        if member_id not in self.members:
            return 0.0
        if barcode not in self.catalog.book_items:
            return 0.0
        
        member = self.members[member_id]
        book_item = self.catalog.book_items[barcode]
        
        fine = member.return_book(book_item)
        book_item.return_book()
        return fine
```

### Key Design Patterns
- **Facade Pattern**: Library class
- **Composite Pattern**: Book and BookItem
- **Strategy Pattern**: Different membership types

---

## 4.4 Design an Elevator System

### Requirements
- Multiple elevators
- Request from any floor
- Optimize wait time
- Handle up/down directions
- Emergency handling

### Class Diagram
```
ElevatorSystem
├── Elevator[]
│   ├── currentFloor
│   ├── direction
│   └── requests[]
└── ElevatorController
    └── assignElevator()
```

### Solution (Python)

```python
from enum import Enum
from collections import deque

class Direction(Enum):
    UP = 1
    DOWN = -1
    IDLE = 0

class Request:
    def __init__(self, floor, direction):
        self.floor = floor
        self.direction = direction

class Elevator:
    def __init__(self, elevator_id, max_floor):
        self.elevator_id = elevator_id
        self.current_floor = 0
        self.direction = Direction.IDLE
        self.max_floor = max_floor
        self.up_requests = set()
        self.down_requests = set()
    
    def add_request(self, floor):
        if floor > self.current_floor:
            self.up_requests.add(floor)
        elif floor < self.current_floor:
            self.down_requests.add(floor)
    
    def move(self):
        if self.direction == Direction.UP:
            if self.up_requests:
                next_floor = min(self.up_requests)
                self.current_floor = next_floor
                self.up_requests.remove(next_floor)
                
                if not self.up_requests:
                    if self.down_requests:
                        self.direction = Direction.DOWN
                    else:
                        self.direction = Direction.IDLE
        
        elif self.direction == Direction.DOWN:
            if self.down_requests:
                next_floor = max(self.down_requests)
                self.current_floor = next_floor
                self.down_requests.remove(next_floor)
                
                if not self.down_requests:
                    if self.up_requests:
                        self.direction = Direction.UP
                    else:
                        self.direction = Direction.IDLE
        
        else:  # IDLE
            if self.up_requests:
                self.direction = Direction.UP
            elif self.down_requests:
                self.direction = Direction.DOWN

class ElevatorController:
    def __init__(self, num_elevators, max_floor):
        self.elevators = [Elevator(i, max_floor) for i in range(num_elevators)]
    
    def request_elevator(self, floor, direction):
        best_elevator = self._find_best_elevator(floor, direction)
        if best_elevator:
            best_elevator.add_request(floor)
            return best_elevator.elevator_id
        return None
    
    def _find_best_elevator(self, floor, direction):
        best = None
        min_distance = float('inf')
        
        for elevator in self.elevators:
            # Prefer idle elevators
            if elevator.direction == Direction.IDLE:
                distance = abs(elevator.current_floor - floor)
                if distance < min_distance:
                    min_distance = distance
                    best = elevator
            
            # Or elevators moving in same direction
            elif elevator.direction == direction:
                if direction == Direction.UP and elevator.current_floor <= floor:
                    distance = floor - elevator.current_floor
                    if distance < min_distance:
                        min_distance = distance
                        best = elevator
                elif direction == Direction.DOWN and elevator.current_floor >= floor:
                    distance = elevator.current_floor - floor
                    if distance < min_distance:
                        min_distance = distance
                        best = elevator
        
        return best if best else self.elevators[0]

class ElevatorSystem:
    def __init__(self, num_elevators, num_floors):
        self.controller = ElevatorController(num_elevators, num_floors)
        self.num_floors = num_floors
    
    def call_elevator(self, floor, direction):
        if 0 <= floor <= self.num_floors:
            return self.controller.request_elevator(floor, direction)
        return None
    
    def select_floor(self, elevator_id, floor):
        if 0 <= elevator_id < len(self.controller.elevators):
            self.controller.elevators[elevator_id].add_request(floor)
```

### Key Algorithms
- **Elevator Assignment**: Find nearest idle or same-direction elevator
- **SCAN Algorithm**: Move in one direction until no more requests
- **Optimization**: Minimize wait time and travel distance

### Key Design Patterns
- **State Pattern**: Elevator states (IDLE, UP, DOWN)
- **Strategy Pattern**: Different scheduling algorithms

---

## 4.5 Design a Vending Machine

### Requirements
- Product inventory
- Accept coins/bills
- Return change
- Dispense product
- Handle sold out

### Class Diagram
```
VendingMachine
├── Inventory
│   └── Product[]
├── CashHandler
│   └── acceptPayment()
└── State (Idle, HasMoney, Dispense)
```

### Solution (Python)

```python
from enum import Enum

class VendingMachineState(Enum):
    IDLE = "IDLE"
    HAS_MONEY = "HAS_MONEY"
    DISPENSE = "DISPENSE"

class Product:
    def __init__(self, name, price, quantity):
        self.name = name
        self.price = price
        self.quantity = quantity
    
    def is_available(self):
        return self.quantity > 0
    
    def dispense(self):
        if self.is_available():
            self.quantity -= 1
            return True
        return False

class Inventory:
    def __init__(self):
        self.products = {}  # slot_number -> Product
    
    def add_product(self, slot_number, product):
        self.products[slot_number] = product
    
    def get_product(self, slot_number):
        return self.products.get(slot_number)
    
    def is_available(self, slot_number):
        product = self.get_product(slot_number)
        return product and product.is_available()

class VendingMachine:
    def __init__(self):
        self.inventory = Inventory()
        self.state = VendingMachineState.IDLE
        self.current_amount = 0.0
        self.selected_slot = None
    
    def insert_money(self, amount):
        self.current_amount += amount
        self.state = VendingMachineState.HAS_MONEY
        print(f"Inserted ${amount:.2f}. Total: ${self.current_amount:.2f}")
    
    def select_product(self, slot_number):
        if self.state != VendingMachineState.HAS_MONEY:
            print("Please insert money first")
            return False
        
        product = self.inventory.get_product(slot_number)
        if not product:
            print("Invalid slot")
            return False
        
        if not product.is_available():
            print(f"{product.name} is sold out")
            return False
        
        if self.current_amount < product.price:
            print(f"Insufficient funds. Need ${product.price - self.current_amount:.2f} more")
            return False
        
        self.selected_slot = slot_number
        self.state = VendingMachineState.DISPENSE
        return self._dispense_product()
    
    def _dispense_product(self):
        product = self.inventory.get_product(self.selected_slot)
        
        if product.dispense():
            change = self.current_amount - product.price
            print(f"Dispensing {product.name}")
            
            if change > 0:
                print(f"Returning change: ${change:.2f}")
            
            # Reset
            self.current_amount = 0.0
            self.selected_slot = None
            self.state = VendingMachineState.IDLE
            return True
        
        return False
    
    def cancel(self):
        if self.current_amount > 0:
            print(f"Returning ${self.current_amount:.2f}")
            self.current_amount = 0.0
        
        self.state = VendingMachineState.IDLE
        self.selected_slot = None

# Usage
machine = VendingMachine()
machine.inventory.add_product(1, Product("Coke", 1.50, 10))
machine.inventory.add_product(2, Product("Chips", 1.00, 5))

machine.insert_money(2.00)
machine.select_product(1)  # Buy Coke
```

### Key Design Patterns
- **State Pattern**: Machine states
- **Singleton Pattern**: Single vending machine instance

---

## Additional LLD Problems

### 4.6 Design a Tic-Tac-Toe Game
**Key Concepts:**
- Board representation (2D array)
- Win condition checking
- Player turns
- Game state management

### 4.7 Design a Snake and Ladder Game
**Key Concepts:**
- Board with snakes and ladders
- Dice rolling
- Player movement
- Win condition

### 4.8 Design a Movie Ticket Booking System
**Key Concepts:**
- Theater, Show, Seat management
- Booking and cancellation
- Payment processing
- Seat locking mechanism

### 4.9 Design a Hotel Management System
**Key Concepts:**
- Room types and availability
- Booking system
- Billing
- Housekeeping

### 4.10 Design a File System
**Key Concepts:**
- Directory tree structure
- File operations (create, delete, move)
- Permissions
- Search functionality

---

## Summary

### Database & SQL Key Takeaways
1. Understand normalization and when to denormalize
2. Master JOINS and subqueries
3. Know indexing strategies
4. Understand ACID properties
5. Practice window functions and CTEs

### HDD Key Principles
1. Scalability (horizontal vs vertical)
2. Load balancing
3. Caching strategies
4. Database sharding
5. CAP theorem
6. Eventual consistency

### LLD Key Principles
1. SOLID principles
2. Design patterns (Singleton, Factory, Strategy, Observer)
3. Clean code practices
4. Object-oriented design
5. Error handling
6. Testing considerations

### Interview Preparation Tips
1. **For HDD**: Always ask clarifying questions about scale, traffic, consistency requirements
2. **For LLD**: Focus on extensibility, maintainability, and SOLID principles
3. **Practice**: Implement these systems hands-on
4. **Trade-offs**: Always discuss trade-offs in your design decisions
5. **Start Simple**: Begin with basic design, then add complexity

---

## Practice Resources

### Recommended Platforms
- **LeetCode**: SQL problems
- **HackerRank**: Database challenges
- **System Design Primer** (GitHub)
- **Designing Data-Intensive Applications** (Book)
- **Grokking the System Design Interview** (Course)

### Next Steps
1. Implement each LLD problem in your preferred language
2. Draw diagrams for HDD problems
3. Practice SQL queries daily
4. Study real-world system architectures (Netflix, Uber tech blogs)
5. Mock interviews with peers

---

*End of Notes*
