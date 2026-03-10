# Database & SQL Notes

This document provides a comprehensive overview of database and SQL concepts, from fundamental to advanced topics.

## 1. SQL Fundamentals

### 1.1 Basic Queries

- **SELECT**: Retrieves data from a database.
  ```sql
  SELECT column1, column2 FROM table_name;
  SELECT * FROM table_name;
  ```

- **INSERT**: Adds new rows to a table.
  ```sql
  INSERT INTO table_name (column1, column2) VALUES (value1, value2);
  ```

- **UPDATE**: Modifies existing rows in a table.
  ```sql
  UPDATE table_name SET column1 = value1 WHERE condition;
  ```

- **DELETE**: Removes existing rows from a table.
  ```sql
  DELETE FROM table_name WHERE condition;
  ```

### 1.2 Filtering and Sorting

- **WHERE**: Filters records based on a condition.
  ```sql
  SELECT * FROM employees WHERE salary > 50000;
  ```

- **ORDER BY**: Sorts the result set in ascending or descending order.
  ```sql
  SELECT * FROM employees ORDER BY last_name ASC;
  ```

- **GROUP BY**: Groups rows that have the same values in specified columns into summary rows.
  ```sql
  SELECT department, COUNT(*) FROM employees GROUP BY department;
  ```

- **HAVING**: Filters groups based on a condition, used with `GROUP BY`.
  ```sql
  SELECT department, AVG(salary) FROM employees GROUP BY department HAVING AVG(salary) > 60000;
  ```

### 1.3 Joins

- **INNER JOIN**: Returns records that have matching values in both tables.
  ```sql
  SELECT orders.order_id, customers.customer_name
  FROM orders
  INNER JOIN customers ON orders.customer_id = customers.customer_id;
  ```

- **LEFT JOIN**: Returns all records from the left table, and the matched records from the right table.
  ```sql
  SELECT employees.name, departments.department_name
  FROM employees
  LEFT JOIN departments ON employees.department_id = departments.id;
  ```

- **RIGHT JOIN**: Returns all records from the right table, and the matched records from the left table.
  ```sql
  SELECT employees.name, departments.department_name
  FROM employees
  RIGHT JOIN departments ON employees.department_id = departments.id;
  ```

- **FULL OUTER JOIN**: Returns all records when there is a match in either left or right table.
  ```sql
  SELECT customers.customer_name, orders.order_id
  FROM customers
  FULL OUTER JOIN orders ON customers.customer_id = orders.customer_id;
  ```

### 1.4 Advanced SQL Constructs

- **Subqueries**: A query nested inside another query.
  ```sql
  SELECT product_name FROM products WHERE product_id IN (SELECT product_id FROM order_items WHERE quantity > 10);
  ```

- **Aggregate Functions**: Perform a calculation on a set of values and return a single value.
  - `COUNT()`: Counts the number of rows.
  - `SUM()`: Calculates the sum of values.
  - `AVG()`: Calculates the average of values.
  - `MIN()`: Returns the minimum value.
  - `MAX()`: Returns the maximum value.

- **Set Operations**:
  - `UNION`: Combines the result sets of two or more SELECT statements (removes duplicates).
  - `UNION ALL`: Combines result sets (includes duplicates).
  - `INTERSECT`: Returns rows that are common to both result sets.
  - `EXCEPT`: Returns rows from the first result set that are not in the second.

- **DISTINCT**: Removes duplicate rows from a result set.
  ```sql
  SELECT DISTINCT department FROM employees;
  ```

- **LIMIT/OFFSET**: Constrains the number of rows returned.
  ```sql
  SELECT * FROM products ORDER BY price DESC LIMIT 10 OFFSET 20;
  ```

## 2. Database Design

### 2.1 Normalization

- **1NF (First Normal Form)**: Ensures that a table has no repeating groups and that each column contains atomic values.
- **2NF (Second Normal Form)**: Must be in 1NF, and all non-key attributes must be fully dependent on the primary key.
- **3NF (Third Normal Form)**: Must be in 2NF, and there should be no transitive dependencies.
- **BCNF (Boyce-Codd Normal Form)**: A stricter form of 3NF. For any dependency A -> B, A must be a superkey.

### 2.2 Keys and Indexes

- **Primary Key**: A constraint that uniquely identifies each record in a table.
- **Foreign Key**: A key used to link two tables together.
- **Indexes**: Special lookup tables that the database search engine can use to speed up data retrieval.
  - **Clustered Index**: Determines the physical order of data in a table.
  - **Non-Clustered Index**: Has a separate structure from the data rows.

### 2.3 Constraints

- **NOT NULL**: Ensures that a column cannot have a NULL value.
- **UNIQUE**: Ensures that all values in a column are different.
- **CHECK**: Ensures that all values in a column satisfy a specific condition.
- **DEFAULT**: Sets a default value for a column when no value is specified.

### 2.4 Other Database Objects

- **Views**: A virtual table based on the result-set of an SQL statement.
  ```sql
  CREATE VIEW high_paid_employees AS
  SELECT * FROM employees WHERE salary > 70000;
  ```

- **Stored Procedures**: A prepared SQL code that you can save, so the code can be reused over and over again.
  ```sql
  CREATE PROCEDURE GetEmployeeById @employeeId INT
  AS
  BEGIN
    SELECT * FROM employees WHERE id = @employeeId;
  END;
  ```

- **Triggers**: A special type of stored procedure that automatically runs when an event occurs in the database server (e.g., INSERT, UPDATE, DELETE).

## 3. Advanced SQL

### 3.1 Window Functions

Functions that perform a calculation across a set of table rows that are somehow related to the current row.
- `ROW_NUMBER()`: Assigns a unique number to each row.
- `RANK()`: Assigns a rank to each row, with gaps in case of ties.
- `DENSE_RANK()`: Assigns a rank without gaps.
  ```sql
  SELECT
    employee_name,
    salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rank_in_department
  FROM employees;
  ```

### 3.2 Common Table Expressions (CTEs)

A temporary named result set that you can reference within a SELECT, INSERT, UPDATE, or DELETE statement.
```sql
WITH regional_sales AS (
  SELECT region, SUM(amount) as total_sales
  FROM orders
  GROUP BY region
)
SELECT region, total_sales
FROM regional_sales
WHERE total_sales > (SELECT AVG(total_sales) FROM regional_sales);
```

### 3.3 Transactions

- **ACID Properties**:
  - **Atomicity**: All changes are performed as a single unit, or none are.
  - **Consistency**: The database remains in a consistent state before and after the transaction.
  - **Isolation**: Concurrent transactions do not interfere with each other.
  - **Durability**: Once a transaction is committed, its changes are permanent.

- **Isolation Levels**: Define the degree to which one transaction must be isolated from the data modifications made by any other transaction.
  - `READ UNCOMMITTED`
  - `READ COMMITTED`
  - `REPEATABLE READ`
  - `SERIALIZABLE`

- **Deadlocks**: A situation where two or more transactions are waiting for each other to release locks.

### 3.4 Query Optimization

- **Execution Plan**: A roadmap that the database follows to execute a query. Analyzing the execution plan helps in identifying performance bottlenecks.
- Use `EXPLAIN` or similar commands to view the query plan.
- Ensure indexes are used effectively.
- Avoid querying more data than necessary.

## 4. NoSQL Databases

### 4.1 MongoDB Basics

- **Document-based storage**: Data is stored in JSON-like documents (BSON).
- **Collections**: A group of MongoDB documents, equivalent to a table in relational databases.

### 4.2 CRUD Operations in MongoDB

- **Create**: `insertOne()`, `insertMany()`
  ```javascript
  db.products.insertOne({ name: "Laptop", price: 1200 });
  ```

- **Read**: `find()`, `findOne()`
  ```javascript
  db.products.find({ price: { $gt: 1000 } });
  ```

- **Update**: `updateOne()`, `updateMany()`
  ```javascript
  db.products.updateOne({ name: "Laptop" }, { $set: { price: 1150 } });
  ```

- **Delete**: `deleteOne()`, `deleteMany()`
  ```javascript
  db.products.deleteOne({ name: "Laptop" });
  ```

### 4.3 Aggregation Pipeline

A framework for data aggregation modeled on the concept of data processing pipelines.
```javascript
db.orders.aggregate([
  { $match: { status: "A" } },
  { $group: { _id: "$cust_id", total: { $sum: "$amount" } } }
]);
```

### 4.4 Indexing in NoSQL

- Similar to relational databases, indexes in MongoDB improve query performance.
- Indexes can be created on any field within a document.
  ```javascript
  db.products.createIndex({ name: 1 }); // Create an index on the 'name' field
  ```
