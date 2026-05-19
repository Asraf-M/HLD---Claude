# 30 Most Asked PostgreSQL Interview Questions

---

## Sample Tables Used Throughout

All examples use these tables:

```sql
-- Employees
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(150) UNIQUE,
    department VARCHAR(50),
    salary NUMERIC(10,2),
    manager_id INT REFERENCES employees(id),
    hire_date DATE DEFAULT CURRENT_DATE,
    is_active BOOLEAN DEFAULT TRUE
);

-- Departments
CREATE TABLE departments (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    location VARCHAR(100),
    budget NUMERIC(12,2)
);

-- Orders
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    employee_id INT REFERENCES employees(id),
    product VARCHAR(100),
    amount NUMERIC(10,2),
    order_date TIMESTAMP DEFAULT NOW(),
    status VARCHAR(20) DEFAULT 'pending'
);

-- Sample Data
INSERT INTO departments (name, location, budget) VALUES
('Engineering', 'Floor 3', 500000),
('HR',          'Floor 1', 200000),
('Sales',       'Floor 2', 300000),
('Marketing',   'Floor 2', 250000);

INSERT INTO employees (name, email, department, salary, manager_id, hire_date, is_active) VALUES
('Alice',   'alice@company.com',   'Engineering', 95000,  NULL, '2020-01-15', TRUE),
('Bob',     'bob@company.com',     'Engineering', 85000,  1,    '2020-06-20', TRUE),
('Carol',   'carol@company.com',   'HR',          70000,  NULL, '2019-03-10', TRUE),
('Dave',    'dave@company.com',    'Sales',       60000,  NULL, '2021-09-01', TRUE),
('Eve',     'eve@company.com',     'Engineering', 92000,  1,    '2021-02-14', TRUE),
('Frank',   'frank@company.com',   'Sales',       55000,  4,    '2022-07-30', FALSE),
('Grace',   'grace@company.com',   'HR',          72000,  3,    '2020-11-05', TRUE),
('Hank',    'hank@company.com',    'Marketing',   65000,  NULL, '2023-01-20', TRUE),
('Ivy',     'ivy@company.com',     'Engineering', 88000,  1,    '2022-04-12', TRUE),
('Jack',    'jack@company.com',    'Sales',       58000,  4,    '2023-06-15', TRUE);

INSERT INTO orders (employee_id, product, amount, order_date, status) VALUES
(1, 'Laptop',     1200.00, '2024-01-10', 'completed'),
(2, 'Monitor',     450.00, '2024-01-15', 'completed'),
(1, 'Keyboard',     80.00, '2024-02-01', 'completed'),
(3, 'Desk Chair',  350.00, '2024-02-10', 'pending'),
(4, 'Headphones',  150.00, '2024-03-05', 'completed'),
(5, 'Laptop',     1200.00, '2024-03-10', 'shipped'),
(NULL, 'Mouse',      30.00, '2024-03-15', 'pending'),
(2, 'Webcam',       90.00, '2024-04-01', 'completed'),
(6, 'Tablet',      600.00, '2024-04-15', 'cancelled'),
(1, 'SSD Drive',   200.00, '2024-05-01', 'completed');
```

### Resulting Tables

**employees**

| id | name  | email              | department  | salary | manager_id | hire_date  | is_active |
|----|-------|--------------------|-------------|--------|------------|------------|-----------|
| 1  | Alice | alice@company.com  | Engineering | 95000  | NULL       | 2020-01-15 | TRUE      |
| 2  | Bob   | bob@company.com    | Engineering | 85000  | 1          | 2020-06-20 | TRUE      |
| 3  | Carol | carol@company.com  | HR          | 70000  | NULL       | 2019-03-10 | TRUE      |
| 4  | Dave  | dave@company.com   | Sales       | 60000  | NULL       | 2021-09-01 | TRUE      |
| 5  | Eve   | eve@company.com    | Engineering | 92000  | 1          | 2021-02-14 | TRUE      |
| 6  | Frank | frank@company.com  | Sales       | 55000  | 4          | 2022-07-30 | FALSE     |
| 7  | Grace | grace@company.com  | HR          | 72000  | 3          | 2020-11-05 | TRUE      |
| 8  | Hank  | hank@company.com   | Marketing   | 65000  | NULL       | 2023-01-20 | TRUE      |
| 9  | Ivy   | ivy@company.com    | Engineering | 88000  | 1          | 2022-04-12 | TRUE      |
| 10 | Jack  | jack@company.com   | Sales       | 58000  | 4          | 2023-06-15 | TRUE      |

**orders**

| id | employee_id | product    | amount  | order_date | status    |
|----|-------------|------------|---------|------------|-----------|
| 1  | 1           | Laptop     | 1200.00 | 2024-01-10 | completed |
| 2  | 2           | Monitor    | 450.00  | 2024-01-15 | completed |
| 3  | 1           | Keyboard   | 80.00   | 2024-02-01 | completed |
| 4  | 3           | Desk Chair | 350.00  | 2024-02-10 | pending   |
| 5  | 4           | Headphones | 150.00  | 2024-03-05 | completed |
| 6  | 5           | Laptop     | 1200.00 | 2024-03-10 | shipped   |
| 7  | NULL        | Mouse      | 30.00   | 2024-03-15 | pending   |
| 8  | 2           | Webcam     | 90.00   | 2024-04-01 | completed |
| 9  | 6           | Tablet     | 600.00  | 2024-04-15 | cancelled |
| 10 | 1           | SSD Drive  | 200.00  | 2024-05-01 | completed |

---

## Table of Contents

1. [PRIMARY KEY, FOREIGN KEY, UNIQUE, NOT NULL](#1-primary-key-foreign-key-unique-not-null)
2. [INNER JOIN vs LEFT JOIN vs RIGHT JOIN vs FULL JOIN](#2-inner-join-vs-left-join-vs-right-join-vs-full-join)
3. [GROUP BY & HAVING](#3-group-by--having)
4. [Subqueries](#4-subqueries)
5. [Window Functions (ROW_NUMBER, RANK, DENSE_RANK)](#5-window-functions)
6. [LEAD, LAG, FIRST_VALUE, LAST_VALUE](#6-lead-lag-first_value-last_value)
7. [CTEs (Common Table Expressions)](#7-ctes-common-table-expressions)
8. [Recursive CTE](#8-recursive-cte)
9. [UNION vs UNION ALL vs INTERSECT vs EXCEPT](#9-union-vs-union-all-vs-intersect-vs-except)
10. [Indexes](#10-indexes)
11. [EXPLAIN & Query Plan](#11-explain--query-plan)
12. [Views & Materialized Views](#12-views--materialized-views)
13. [Transactions & ACID](#13-transactions--acid)
14. [Isolation Levels](#14-isolation-levels)
15. [UPSERT (INSERT ON CONFLICT)](#15-upsert-insert-on-conflict)
16. [COALESCE, NULLIF, CASE](#16-coalesce-nullif-case)
17. [String Functions](#17-string-functions)
18. [Date/Time Functions](#18-datetime-functions)
19. [Aggregate Functions](#19-aggregate-functions)
20. [EXISTS vs IN](#20-exists-vs-in)
21. [ANY, ALL, SOME](#21-any-all-some)
22. [LATERAL JOIN](#22-lateral-join)
23. [JSON/JSONB](#23-jsonjsonb)
24. [Array Data Type](#24-array-data-type)
25. [COPY & Bulk Insert](#25-copy--bulk-insert)
26. [Stored Functions (PL/pgSQL)](#26-stored-functions-plpgsql)
27. [Triggers](#27-triggers)
28. [Table Partitioning](#28-table-partitioning)
29. [Normalization (1NF, 2NF, 3NF)](#29-normalization)
30. [Performance Tuning & Best Practices](#30-performance-tuning--best-practices)

---

## 1. PRIMARY KEY, FOREIGN KEY, UNIQUE, NOT NULL

**Q: What are the main constraints in PostgreSQL?**

| Constraint | Purpose | Allows NULL? | Allows Duplicates? |
|---|---|---|---|
| `PRIMARY KEY` | Unique row identifier | ❌ | ❌ |
| `FOREIGN KEY` | Referential integrity to another table | ✅ | ✅ |
| `UNIQUE` | No duplicate values | ✅ (multiple NULLs allowed) | ❌ |
| `NOT NULL` | Value must be provided | ❌ | ✅ |
| `CHECK` | Custom condition | Depends | Depends |
| `DEFAULT` | Auto-fill if not provided | ✅ | ✅ |

```sql
-- SERIAL = auto-increment integer
-- PRIMARY KEY = NOT NULL + UNIQUE
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,                          -- auto-increment PK
    name VARCHAR(100) NOT NULL,                     -- required field
    email VARCHAR(150) UNIQUE,                      -- no duplicate emails
    manager_id INT REFERENCES employees(id),        -- FK → self-referencing
    salary NUMERIC(10,2) CHECK (salary > 0),        -- must be positive
    hire_date DATE DEFAULT CURRENT_DATE             -- auto-fill today
);
```

### Foreign Key Actions

```sql
-- ON DELETE CASCADE: delete child rows when parent is deleted
-- ON DELETE SET NULL: set FK to NULL when parent is deleted
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    employee_id INT REFERENCES employees(id) ON DELETE SET NULL
);
```

| Action | Effect on child row |
|---|---|
| `CASCADE` | Child deleted too |
| `SET NULL` | FK set to NULL |
| `SET DEFAULT` | FK set to default value |
| `RESTRICT` | Prevent parent deletion (default) |
| `NO ACTION` | Same as RESTRICT |

---

## 2. INNER JOIN vs LEFT JOIN vs RIGHT JOIN vs FULL JOIN

**Q: Explain the different types of JOINs.**

```
  INNER JOIN          LEFT JOIN           RIGHT JOIN          FULL OUTER JOIN
  ┌───┬───┐          ┌───┬───┐          ┌───┬───┐          ┌───┬───┐
  │   │███│          │███│███│          │███│███│          │███│███│
  │   │███│          │███│███│          │███│███│          │███│███│
  │   │███│          │███│███│          │███│███│          │███│███│
  └───┴───┘          └───┴───┘          └───┴───┘          └───┴───┘
   Only matching       All left +         All right +        All from both
                       matching right     matching left
```

```sql
-- INNER JOIN: only matching rows from both tables
SELECT e.name, o.product, o.amount
FROM employees e
INNER JOIN orders o ON e.id = o.employee_id;
```

| name  | product    | amount  |
|-------|------------|---------|
| Alice | Laptop     | 1200.00 |
| Bob   | Monitor    | 450.00  |
| Alice | Keyboard   | 80.00   |
| Carol | Desk Chair | 350.00  |
| Dave  | Headphones | 150.00  |
| Eve   | Laptop     | 1200.00 |
| Bob   | Webcam     | 90.00   |
| Frank | Tablet     | 600.00  |
| Alice | SSD Drive  | 200.00  |

```sql
-- LEFT JOIN: all employees + their orders (NULL if no orders)
SELECT e.name, o.product
FROM employees e
LEFT JOIN orders o ON e.id = o.employee_id;
```

| name  | product    |
|-------|------------|
| Alice | Laptop     |
| Alice | Keyboard   |
| Alice | SSD Drive  |
| Bob   | Monitor    |
| Bob   | Webcam     |
| Carol | Desk Chair |
| Dave  | Headphones |
| Eve   | Laptop     |
| Frank | Tablet     |
| Grace | NULL       |
| Hank  | NULL       |
| Ivy   | NULL       |
| Jack  | NULL       |

```sql
-- RIGHT JOIN: all orders + employee info (NULL if no employee)
SELECT e.name, o.product
FROM employees e
RIGHT JOIN orders o ON e.id = o.employee_id;
```

| name  | product    |
|-------|------------|
| Alice | Laptop     |
| Bob   | Monitor    |
| Alice | Keyboard   |
| Carol | Desk Chair |
| Dave  | Headphones |
| Eve   | Laptop     |
| NULL  | Mouse      |
| Bob   | Webcam     |
| Frank | Tablet     |
| Alice | SSD Drive  |

```sql
-- FULL OUTER JOIN: all rows from both (NULLs on both sides if no match)
SELECT e.name, o.product
FROM employees e
FULL OUTER JOIN orders o ON e.id = o.employee_id;
-- Result: all 13 rows from LEFT JOIN + Mouse order with NULL employee
```

### Self Join

```sql
-- Find employee and their manager name
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

| employee | manager |
|----------|---------|
| Alice    | NULL    |
| Bob      | Alice   |
| Carol    | NULL    |
| Dave     | NULL    |
| Eve      | Alice   |
| Frank    | Dave    |
| Grace    | Carol   |
| Hank     | NULL    |
| Ivy      | Alice   |
| Jack     | Dave    |

---

## 3. GROUP BY & HAVING

**Q: What is the difference between WHERE and HAVING?**

| Clause | Filters | Runs |
|---|---|---|
| `WHERE` | Individual rows | **Before** GROUP BY |
| `HAVING` | Groups (aggregated results) | **After** GROUP BY |

```sql
-- GROUP BY: count employees per department
SELECT department, COUNT(*) AS emp_count, AVG(salary)::NUMERIC(10,2) AS avg_salary
FROM employees
GROUP BY department;
```

| department  | emp_count | avg_salary |
|-------------|-----------|------------|
| Engineering | 4         | 90000.00   |
| HR          | 2         | 71000.00   |
| Sales       | 3         | 57666.67   |
| Marketing   | 1         | 65000.00   |

```sql
-- HAVING: only departments with avg salary > 65000
SELECT department, COUNT(*) AS emp_count, AVG(salary)::NUMERIC(10,2) AS avg_salary
FROM employees
WHERE is_active = TRUE              -- row-level filter (before grouping)
GROUP BY department
HAVING AVG(salary) > 65000          -- group-level filter (after grouping)
ORDER BY avg_salary DESC;
```

| department  | emp_count | avg_salary |
|-------------|-----------|------------|
| Engineering | 4         | 90000.00   |
| HR          | 2         | 71000.00   |

### Execution Order

```
  FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

---

## 4. Subqueries

**Q: What are subqueries and where can they be used?**

```sql
-- 1. Subquery in WHERE: employees earning more than average
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

| name  | salary |
|-------|--------|
| Alice | 95000  |
| Bob   | 85000  |
| Eve   | 92000  |
| Ivy   | 88000  |

```sql
-- 2. Subquery in FROM (derived table): department stats
SELECT dept_stats.department, dept_stats.max_sal
FROM (
    SELECT department, MAX(salary) AS max_sal
    FROM employees
    GROUP BY department
) AS dept_stats
WHERE dept_stats.max_sal > 70000;
```

| department  | max_sal |
|-------------|---------|
| Engineering | 95000   |
| HR          | 72000   |

```sql
-- 3. Correlated subquery: employees who have placed orders
SELECT name, department
FROM employees e
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.employee_id = e.id
);
```

| name  | department  |
|-------|-------------|
| Alice | Engineering |
| Bob   | Engineering |
| Carol | HR          |
| Dave  | Sales       |
| Eve   | Engineering |
| Frank | Sales       |

```sql
-- 4. Subquery in SELECT: order count per employee
SELECT name,
       (SELECT COUNT(*) FROM orders o WHERE o.employee_id = e.id) AS order_count
FROM employees e;
```

| name  | order_count |
|-------|-------------|
| Alice | 3           |
| Bob   | 2           |
| Carol | 1           |
| Dave  | 1           |
| Eve   | 1           |
| Frank | 1           |
| Grace | 0           |
| Hank  | 0           |
| Ivy   | 0           |
| Jack  | 0           |

---

## 5. Window Functions

**Q: What are ROW_NUMBER, RANK, and DENSE_RANK?**

Window functions perform calculations across a set of rows **without collapsing** them (unlike GROUP BY).

```sql
SELECT name, department, salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC)                         AS row_num,
    RANK()       OVER (ORDER BY salary DESC)                         AS rank,
    DENSE_RANK() OVER (ORDER BY salary DESC)                         AS dense_rank,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC)  AS dept_rank
FROM employees;
```

| name  | department  | salary | row_num | rank | dense_rank | dept_rank |
|-------|-------------|--------|---------|------|------------|-----------|
| Alice | Engineering | 95000  | 1       | 1    | 1          | 1         |
| Eve   | Engineering | 92000  | 2       | 2    | 2          | 2         |
| Ivy   | Engineering | 88000  | 3       | 3    | 3          | 3         |
| Bob   | Engineering | 85000  | 4       | 4    | 4          | 4         |
| Grace | HR          | 72000  | 5       | 5    | 5          | 1         |
| Carol | HR          | 70000  | 6       | 6    | 6          | 2         |
| Hank  | Marketing   | 65000  | 7       | 7    | 7          | 1         |
| Dave  | Sales       | 60000  | 8       | 8    | 8          | 1         |
| Jack  | Sales       | 58000  | 9       | 9    | 9          | 2         |
| Frank | Sales       | 55000  | 10      | 10   | 10         | 3         |

### Difference with Ties

| Function | Ties (same value) | Gaps after tie? |
|---|---|---|
| `ROW_NUMBER()` | Assigns different numbers | — |
| `RANK()` | Same rank for ties | ✅ Yes (1,1,3 — skips 2) |
| `DENSE_RANK()` | Same rank for ties | ❌ No (1,1,2 — no gap) |

```sql
-- Top 2 earners PER department using window function
SELECT * FROM (
    SELECT name, department, salary,
           ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
) ranked
WHERE rn <= 2;
```

| name  | department  | salary | rn |
|-------|-------------|--------|----|
| Alice | Engineering | 95000  | 1  |
| Eve   | Engineering | 92000  | 2  |
| Grace | HR          | 72000  | 1  |
| Carol | HR          | 70000  | 2  |
| Hank  | Marketing   | 65000  | 1  |
| Dave  | Sales       | 60000  | 1  |
| Jack  | Sales       | 58000  | 2  |

---

## 6. LEAD, LAG, FIRST_VALUE, LAST_VALUE

**Q: What are LEAD and LAG window functions?**

| Function | Returns |
|---|---|
| `LAG(col, n)` | Value from **n rows before** current row |
| `LEAD(col, n)` | Value from **n rows after** current row |
| `FIRST_VALUE(col)` | First value in the window frame |
| `LAST_VALUE(col)` | Last value in the window frame |
| `NTH_VALUE(col, n)` | Nth value in the window frame |

```sql
SELECT name, department, salary,
    LAG(salary, 1)  OVER (PARTITION BY department ORDER BY salary DESC)  AS prev_salary,
    LEAD(salary, 1) OVER (PARTITION BY department ORDER BY salary DESC)  AS next_salary,
    salary - LAG(salary, 1) OVER (PARTITION BY department ORDER BY salary DESC) AS diff
FROM employees
WHERE department = 'Engineering';
```

| name  | department  | salary | prev_salary | next_salary | diff   |
|-------|-------------|--------|-------------|-------------|--------|
| Alice | Engineering | 95000  | NULL        | 92000       | NULL   |
| Eve   | Engineering | 92000  | 95000       | 88000       | -3000  |
| Ivy   | Engineering | 88000  | 92000       | 85000       | -4000  |
| Bob   | Engineering | 85000  | 88000       | NULL        | -3000  |

```sql
-- Running total (cumulative sum)
SELECT name, salary,
    SUM(salary) OVER (ORDER BY hire_date ROWS UNBOUNDED PRECEDING) AS running_total
FROM employees;
```

---

## 7. CTEs (Common Table Expressions)

**Q: What is a CTE and why use it?**

A **CTE** is a temporary named result set defined using `WITH`. It exists only for the duration of the query.

```sql
-- CTE: find departments where average salary > company average
WITH company_avg AS (
    SELECT AVG(salary) AS avg_sal FROM employees
),
dept_avg AS (
    SELECT department, AVG(salary) AS avg_sal, COUNT(*) AS emp_count
    FROM employees
    GROUP BY department
)
SELECT d.department, d.avg_sal::NUMERIC(10,2), d.emp_count
FROM dept_avg d, company_avg c
WHERE d.avg_sal > c.avg_sal;
```

| department  | avg_sal  | emp_count |
|-------------|----------|-----------|
| Engineering | 90000.00 | 4         |

### CTE vs Subquery

| Feature | CTE | Subquery |
|---|---|---|
| Readability | ✅ Named, top-level | ❌ Nested, harder to read |
| Reusability | ✅ Reference multiple times | ❌ Must repeat |
| Recursion | ✅ Supports recursive | ❌ No |
| Performance | Usually same | Usually same |

---

## 8. Recursive CTE

**Q: How does a recursive CTE work?**

Used for **hierarchical data** — org charts, tree structures, parent-child relationships.

```sql
-- Employee hierarchy: find all reports under Alice (id=1)
WITH RECURSIVE org_chart AS (
    -- Base case: start with Alice
    SELECT id, name, manager_id, 1 AS level
    FROM employees
    WHERE id = 1

    UNION ALL

    -- Recursive step: find employees whose manager is in current result
    SELECT e.id, e.name, e.manager_id, oc.level + 1
    FROM employees e
    INNER JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT id, name, manager_id, level
FROM org_chart;
```

| id | name | manager_id | level |
|----|------|------------|-------|
| 1  | Alice| NULL       | 1     |
| 2  | Bob  | 1          | 2     |
| 5  | Eve  | 1          | 2     |
| 9  | Ivy  | 1          | 2     |

```
  Alice (Level 1)
  ├── Bob (Level 2)
  ├── Eve (Level 2)
  └── Ivy (Level 2)
```

---

## 9. UNION vs UNION ALL vs INTERSECT vs EXCEPT

**Q: What is the difference between UNION, UNION ALL, INTERSECT, and EXCEPT?**

| Operator | Returns | Removes Duplicates? |
|---|---|---|
| `UNION` | Rows from both queries | ✅ Yes |
| `UNION ALL` | Rows from both queries | ❌ No (faster) |
| `INTERSECT` | Only rows in **both** queries | ✅ Yes |
| `EXCEPT` | Rows in first but **not** in second | ✅ Yes |

```sql
-- Employees in Engineering
SELECT name FROM employees WHERE department = 'Engineering';
-- Alice, Bob, Eve, Ivy

-- Employees who placed orders
SELECT e.name FROM employees e JOIN orders o ON e.id = o.employee_id;
-- Alice, Bob, Carol, Dave, Eve, Frank (with duplicates for Alice, Bob)

-- UNION: combine without duplicates
SELECT name FROM employees WHERE department = 'Engineering'
UNION
SELECT e.name FROM employees e JOIN orders o ON e.id = o.employee_id;
-- Alice, Bob, Carol, Dave, Eve, Frank, Ivy

-- INTERSECT: Engineering employees who ALSO placed orders
SELECT name FROM employees WHERE department = 'Engineering'
INTERSECT
SELECT e.name FROM employees e JOIN orders o ON e.id = o.employee_id;
```

| name  |
|-------|
| Alice |
| Bob   |
| Eve   |

```sql
-- EXCEPT: Engineering employees who have NOT placed orders
SELECT name FROM employees WHERE department = 'Engineering'
EXCEPT
SELECT e.name FROM employees e JOIN orders o ON e.id = o.employee_id;
```

| name |
|------|
| Ivy  |

---

## 10. Indexes

**Q: What are indexes and what types does PostgreSQL support?**

An **index** is a data structure that speeds up data retrieval at the cost of extra storage and slower writes.

```sql
-- B-Tree index (default) — best for =, <, >, BETWEEN, ORDER BY
CREATE INDEX idx_emp_department ON employees(department);

-- Unique index
CREATE UNIQUE INDEX idx_emp_email ON employees(email);

-- Composite index (multi-column)
CREATE INDEX idx_emp_dept_salary ON employees(department, salary DESC);

-- Partial index (filtered)
CREATE INDEX idx_active_emp ON employees(department) WHERE is_active = TRUE;

-- GIN index — for full-text search, arrays, JSONB
CREATE INDEX idx_orders_product ON orders USING GIN (to_tsvector('english', product));

-- Check existing indexes
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'employees';

-- Drop index
DROP INDEX idx_emp_department;
```

| Index Type | Best For |
|---|---|
| **B-Tree** (default) | `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, `LIKE 'abc%'` |
| **Hash** | Only `=` comparisons |
| **GIN** | Arrays, JSONB, full-text search |
| **GiST** | Geometric, range, full-text search |
| **BRIN** | Very large tables with naturally ordered data |

> **Rule of thumb:** Don't index columns with low cardinality (e.g., boolean), small tables, or columns rarely used in WHERE/JOIN.

---

## 11. EXPLAIN & Query Plan

**Q: How do you analyze query performance?**

```sql
-- EXPLAIN: shows the query plan (estimated)
EXPLAIN SELECT * FROM employees WHERE department = 'Engineering';

-- EXPLAIN ANALYZE: actually runs the query + shows real timing
EXPLAIN ANALYZE SELECT * FROM employees WHERE department = 'Engineering';

-- EXPLAIN with options
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT e.name, o.product
FROM employees e
JOIN orders o ON e.id = o.employee_id
WHERE e.department = 'Engineering';
```

### Key Terms in Query Plan

| Term | Meaning |
|---|---|
| **Seq Scan** | Full table scan — reads every row |
| **Index Scan** | Uses index to find rows |
| **Index Only Scan** | Data read entirely from index (no table access) |
| **Bitmap Index Scan** | Index creates a bitmap → Bitmap Heap Scan reads matching rows |
| **Nested Loop** | For each row in outer, scan inner (small datasets) |
| **Hash Join** | Hash one table, probe with other (medium datasets) |
| **Merge Join** | Both sorted, merge-walk (large sorted datasets) |
| **cost=0.00..35.50** | Estimated startup..total cost |
| **rows=4** | Estimated row count |
| **actual time** | Real execution time (with ANALYZE) |

---

## 12. Views & Materialized Views

**Q: What is the difference between a View and a Materialized View?**

| Feature | View | Materialized View |
|---|---|---|
| Stores data? | ❌ Query re-runs every time | ✅ Stores result on disk |
| Performance | Same as underlying query | Faster (pre-computed) |
| Freshness | Always current | Stale until refreshed |
| Indexable? | ❌ | ✅ |

```sql
-- VIEW: virtual table — re-executes query every time
CREATE VIEW active_eng_employees AS
SELECT name, salary, hire_date
FROM employees
WHERE department = 'Engineering' AND is_active = TRUE;

SELECT * FROM active_eng_employees;
```

| name  | salary | hire_date  |
|-------|--------|------------|
| Alice | 95000  | 2020-01-15 |
| Bob   | 85000  | 2020-06-20 |
| Eve   | 92000  | 2021-02-14 |
| Ivy   | 88000  | 2022-04-12 |

```sql
-- MATERIALIZED VIEW: stores result physically
CREATE MATERIALIZED VIEW dept_summary AS
SELECT department, COUNT(*) AS emp_count, AVG(salary)::NUMERIC(10,2) AS avg_salary
FROM employees
GROUP BY department;

-- Must refresh manually when data changes
REFRESH MATERIALIZED VIEW dept_summary;
REFRESH MATERIALIZED VIEW CONCURRENTLY dept_summary;  -- no lock (needs unique index)

-- Can index materialized views
CREATE UNIQUE INDEX idx_dept_summary ON dept_summary(department);

-- Drop
DROP VIEW active_eng_employees;
DROP MATERIALIZED VIEW dept_summary;
```

---

## 13. Transactions & ACID

**Q: What are ACID properties?**

| Property | Meaning | Example |
|---|---|---|
| **Atomicity** | All or nothing — if any step fails, rollback everything | Transfer money: debit + credit must both succeed |
| **Consistency** | DB goes from one valid state to another | Constraints, triggers are respected |
| **Isolation** | Concurrent transactions don't interfere | Two transfers don't corrupt balance |
| **Durability** | Once committed, data survives crashes | Written to WAL (Write-Ahead Log) |

```sql
-- Transaction with COMMIT
BEGIN;
    UPDATE employees SET salary = salary - 5000 WHERE name = 'Alice';
    UPDATE employees SET salary = salary + 5000 WHERE name = 'Bob';
COMMIT;  -- both changes saved

-- Transaction with ROLLBACK
BEGIN;
    DELETE FROM employees WHERE department = 'Sales';
    -- oops, wrong deletion!
ROLLBACK;  -- nothing deleted

-- SAVEPOINT: partial rollback
BEGIN;
    UPDATE employees SET salary = 100000 WHERE name = 'Alice';
    SAVEPOINT sp1;
    UPDATE employees SET salary = 0 WHERE name = 'Bob';     -- mistake!
    ROLLBACK TO sp1;                                         -- undo only Bob's update
    UPDATE employees SET salary = 90000 WHERE name = 'Bob';  -- correct update
COMMIT;  -- Alice=100000, Bob=90000
```

---

## 14. Isolation Levels

**Q: What are the transaction isolation levels in PostgreSQL?**

| Problem | Description |
|---|---|
| **Dirty Read** | Read uncommitted data from another transaction |
| **Non-Repeatable Read** | Same row read twice gives different values (another tx committed) |
| **Phantom Read** | Same query returns different rows (another tx inserted/deleted) |
| **Serialization Anomaly** | Concurrent transactions produce result impossible if run serially |

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| **Read Uncommitted** | Possible* | Possible | Possible |
| **Read Committed** (default) | ❌ Prevented | Possible | Possible |
| **Repeatable Read** | ❌ Prevented | ❌ Prevented | Possible* |
| **Serializable** | ❌ Prevented | ❌ Prevented | ❌ Prevented |

> *PostgreSQL's Read Uncommitted actually behaves like Read Committed. Repeatable Read also prevents phantom reads in PostgreSQL (stricter than SQL standard).

```sql
-- Set isolation level
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- your queries here
COMMIT;

-- Check current level
SHOW transaction_isolation;  -- default: read committed
```

---

## 15. UPSERT (INSERT ON CONFLICT)

**Q: How does UPSERT work in PostgreSQL?**

Insert a row; if it violates a unique constraint, **update** instead.

```sql
-- Insert or update on conflict
INSERT INTO employees (id, name, email, department, salary)
VALUES (2, 'Bob', 'bob@company.com', 'Engineering', 90000)
ON CONFLICT (id)
DO UPDATE SET salary = EXCLUDED.salary;
-- Bob's salary updated to 90000 (EXCLUDED refers to the rejected new row)

-- Insert or do nothing (skip duplicates)
INSERT INTO employees (id, name, email, department, salary)
VALUES (2, 'Bob', 'bob@company.com', 'Engineering', 85000)
ON CONFLICT (email)
DO NOTHING;

-- Upsert with conditional update
INSERT INTO employees (id, name, email, department, salary)
VALUES (2, 'Bob', 'bob@company.com', 'Engineering', 95000)
ON CONFLICT (id)
DO UPDATE SET salary = EXCLUDED.salary
WHERE EXCLUDED.salary > employees.salary;  -- only update if new salary is higher
```

---

## 16. COALESCE, NULLIF, CASE

**Q: How do you handle NULL values and conditional logic?**

```sql
-- COALESCE: returns first non-NULL value
SELECT name, COALESCE(manager_id, 0) AS manager_id  -- replace NULL with 0
FROM employees;
```

| name  | manager_id |
|-------|------------|
| Alice | 0          |
| Bob   | 1          |
| Carol | 0          |

```sql
-- NULLIF: returns NULL if two values are equal (useful to avoid division by zero)
SELECT 10 / NULLIF(0, 0);  -- returns NULL instead of division-by-zero error

-- CASE: conditional logic (like if-else)
SELECT name, salary,
    CASE
        WHEN salary >= 90000 THEN 'Senior'
        WHEN salary >= 70000 THEN 'Mid'
        WHEN salary >= 55000 THEN 'Junior'
        ELSE 'Intern'
    END AS level
FROM employees;
```

| name  | salary | level  |
|-------|--------|--------|
| Alice | 95000  | Senior |
| Bob   | 85000  | Mid    |
| Carol | 70000  | Mid    |
| Dave  | 60000  | Junior |
| Eve   | 92000  | Senior |
| Frank | 55000  | Junior |
| Grace | 72000  | Mid    |
| Hank  | 65000  | Junior |
| Ivy   | 88000  | Mid    |
| Jack  | 58000  | Junior |

```sql
-- CASE inside aggregate: pivot-style query
SELECT department,
    COUNT(CASE WHEN salary >= 80000 THEN 1 END) AS high_earners,
    COUNT(CASE WHEN salary < 80000 THEN 1 END)  AS others
FROM employees
GROUP BY department;
```

| department  | high_earners | others |
|-------------|--------------|--------|
| Engineering | 4            | 0      |
| HR          | 0            | 2      |
| Sales       | 0            | 3      |
| Marketing   | 0            | 1      |

---

## 17. String Functions

**Q: What are commonly used string functions in PostgreSQL?**

```sql
SELECT
    UPPER('hello'),                     -- 'HELLO'
    LOWER('HELLO'),                     -- 'hello'
    INITCAP('hello world'),             -- 'Hello World'
    LENGTH('PostgreSQL'),               -- 10
    TRIM('  hello  '),                  -- 'hello'
    LTRIM('  hello'),                   -- 'hello'
    RTRIM('hello  '),                   -- 'hello'
    SUBSTRING('PostgreSQL' FROM 1 FOR 8), -- 'PostgreS'
    LEFT('PostgreSQL', 4),              -- 'Post'
    RIGHT('PostgreSQL', 3),             -- 'SQL'
    REPLACE('Hello World', 'World', 'PG'), -- 'Hello PG'
    CONCAT('Hello', ' ', 'World'),      -- 'Hello World'
    'Hello' || ' ' || 'World',          -- 'Hello World' (concat operator)
    POSITION('SQL' IN 'PostgreSQL'),    -- 8
    REPEAT('ab', 3),                    -- 'ababab'
    REVERSE('hello'),                   -- 'olleh'
    SPLIT_PART('a.b.c', '.', 2);        -- 'b'

-- Pattern matching
SELECT name FROM employees WHERE name LIKE 'A%';        -- starts with A
SELECT name FROM employees WHERE name ILIKE '%eve%';     -- case-insensitive
SELECT name FROM employees WHERE name ~ '^[A-E]';       -- regex: starts with A-E
SELECT name FROM employees WHERE name SIMILAR TO '(A|B)%'; -- SQL regex
```

---

## 18. Date/Time Functions

**Q: What are commonly used date/time functions?**

```sql
SELECT
    NOW(),                                      -- 2024-05-01 14:30:00+05:30
    CURRENT_DATE,                               -- 2024-05-01
    CURRENT_TIME,                               -- 14:30:00+05:30
    CURRENT_TIMESTAMP,                          -- same as NOW()
    
    -- Extract parts
    EXTRACT(YEAR FROM NOW()),                   -- 2024
    EXTRACT(MONTH FROM NOW()),                  -- 5
    EXTRACT(DOW FROM NOW()),                    -- 3 (0=Sunday)
    DATE_PART('year', NOW()),                   -- 2024 (same as EXTRACT)
    
    -- Date arithmetic
    NOW() + INTERVAL '7 days',                  -- 7 days from now
    NOW() - INTERVAL '3 months',                -- 3 months ago
    AGE(NOW(), '2020-01-15'::DATE),             -- 4 years 3 mons 16 days
    DATE_TRUNC('month', NOW()),                 -- 2024-05-01 00:00:00
    
    -- Formatting
    TO_CHAR(NOW(), 'DD-Mon-YYYY HH24:MI'),      -- '01-May-2024 14:30'
    TO_DATE('01-05-2024', 'DD-MM-YYYY');         -- 2024-05-01

-- Practical: employees hired in last 2 years
SELECT name, hire_date
FROM employees
WHERE hire_date >= CURRENT_DATE - INTERVAL '2 years';
```

| name  | hire_date  |
|-------|------------|
| Hank  | 2023-01-20 |
| Jack  | 2023-06-15 |
| Ivy   | 2022-04-12 |
| Frank | 2022-07-30 |

```sql
-- Group orders by month
SELECT DATE_TRUNC('month', order_date)::DATE AS month, COUNT(*), SUM(amount)
FROM orders
GROUP BY month
ORDER BY month;
```

| month      | count | sum     |
|------------|-------|---------|
| 2024-01-01 | 2     | 1650.00 |
| 2024-02-01 | 2     | 430.00  |
| 2024-03-01 | 3     | 1380.00 |
| 2024-04-01 | 2     | 690.00  |
| 2024-05-01 | 1     | 200.00  |

---

## 19. Aggregate Functions

**Q: What are the aggregate functions in PostgreSQL?**

```sql
SELECT
    COUNT(*)              AS total_employees,       -- 10
    COUNT(manager_id)     AS with_manager,          -- 6 (skips NULLs)
    COUNT(DISTINCT department) AS dept_count,        -- 4
    SUM(salary)           AS total_salary,          -- 740000
    AVG(salary)::NUMERIC(10,2) AS avg_salary,       -- 74000.00
    MIN(salary)           AS min_salary,            -- 55000
    MAX(salary)           AS max_salary,            -- 95000
    STRING_AGG(name, ', ' ORDER BY name) AS all_names  -- Alice, Bob, Carol...
FROM employees;
```

| total_employees | with_manager | dept_count | total_salary | avg_salary | min_salary | max_salary |
|-----------------|--------------|------------|--------------|------------|------------|------------|
| 10              | 6            | 4          | 740000       | 74000.00   | 55000      | 95000      |

```sql
-- STRING_AGG: concatenate names per department
SELECT department, STRING_AGG(name, ', ' ORDER BY name) AS members
FROM employees
GROUP BY department;
```

| department  | members                  |
|-------------|--------------------------|
| Engineering | Alice, Bob, Eve, Ivy     |
| HR          | Carol, Grace             |
| Marketing   | Hank                     |
| Sales       | Dave, Frank, Jack        |

---

## 20. EXISTS vs IN

**Q: What is the difference between EXISTS and IN?**

| Feature | `IN` | `EXISTS` |
|---|---|---|
| How it works | Compares value against a list | Checks if subquery returns any rows |
| NULL handling | ❌ Fails with NULLs in list | ✅ Handles NULLs properly |
| Performance (large subquery) | Slower (materializes full list) | Faster (short-circuits on first match) |
| Correlated? | Not necessarily | Usually correlated |

```sql
-- IN: employees who placed orders
SELECT name FROM employees
WHERE id IN (SELECT employee_id FROM orders);

-- EXISTS: same result, usually faster for large datasets
SELECT name FROM employees e
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.employee_id = e.id
);

-- NOT IN problem with NULLs!
SELECT name FROM employees
WHERE id NOT IN (SELECT employee_id FROM orders);
-- ❌ Returns EMPTY because employee_id has NULL (order #7)
-- NULL in a NOT IN list makes the entire condition UNKNOWN

-- NOT EXISTS: handles NULLs correctly
SELECT name FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.employee_id = e.id
);
```

| name  |
|-------|
| Grace |
| Hank  |
| Ivy   |
| Jack  |

> **Rule:** Prefer `NOT EXISTS` over `NOT IN` when subquery can contain NULLs.

---

## 21. ANY, ALL, SOME

**Q: What are `ANY`, `ALL`, and `SOME` operators?**

| Operator | Meaning |
|---|---|
| `ANY` / `SOME` | Condition true for **at least one** value |
| `ALL` | Condition true for **every** value |

```sql
-- ANY: salary greater than ANY salary in Sales
SELECT name, salary FROM employees
WHERE salary > ANY (SELECT salary FROM employees WHERE department = 'Sales');
-- salary > 55000 OR salary > 60000 OR salary > 58000
-- effectively: salary > MIN(sales salaries) = salary > 55000
```

| name  | salary |
|-------|--------|
| Alice | 95000  |
| Bob   | 85000  |
| Carol | 70000  |
| Dave  | 60000  |
| Eve   | 92000  |
| Grace | 72000  |
| Hank  | 65000  |
| Ivy   | 88000  |
| Jack  | 58000  |

```sql
-- ALL: salary greater than ALL salaries in Sales
SELECT name, salary FROM employees
WHERE salary > ALL (SELECT salary FROM employees WHERE department = 'Sales');
-- salary > 55000 AND salary > 60000 AND salary > 58000
-- effectively: salary > MAX(sales salaries) = salary > 60000
```

| name  | salary |
|-------|--------|
| Alice | 95000  |
| Bob   | 85000  |
| Carol | 70000  |
| Eve   | 92000  |
| Grace | 72000  |
| Hank  | 65000  |
| Ivy   | 88000  |

---

## 22. LATERAL JOIN

**Q: What is a LATERAL join?**

A `LATERAL` join allows the subquery to **reference columns from preceding tables** in the FROM clause (like a correlated subquery but in FROM).

```sql
-- Top 2 orders per employee using LATERAL
SELECT e.name, top_orders.product, top_orders.amount
FROM employees e
CROSS JOIN LATERAL (
    SELECT product, amount
    FROM orders o
    WHERE o.employee_id = e.id
    ORDER BY amount DESC
    LIMIT 2
) AS top_orders;
```

| name  | product    | amount  |
|-------|------------|---------|
| Alice | Laptop     | 1200.00 |
| Alice | SSD Drive  | 200.00  |
| Bob   | Monitor    | 450.00  |
| Bob   | Webcam     | 90.00   |
| Carol | Desk Chair | 350.00  |
| Dave  | Headphones | 150.00  |
| Eve   | Laptop     | 1200.00 |
| Frank | Tablet     | 600.00  |

> Without LATERAL, the subquery can't reference `e.id`. LATERAL is like `forEach` — runs the subquery once per row from the left side.

---

## 23. JSON/JSONB

**Q: What is the difference between JSON and JSONB?**

| Feature | `JSON` | `JSONB` |
|---|---|---|
| Storage | Text (preserves formatting) | Binary (parsed, compressed) |
| Duplicate keys | ✅ Preserved | ❌ Last value kept |
| Key order | ✅ Preserved | ❌ Not guaranteed |
| Indexing | ❌ | ✅ GIN index |
| Performance | Slower (re-parse each time) | Faster (pre-parsed) |

```sql
-- Creating table with JSONB
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    data JSONB
);

INSERT INTO events (data) VALUES
('{"type": "click", "page": "/home", "user": {"id": 1, "name": "Alice"}}'),
('{"type": "view", "page": "/products", "user": {"id": 2, "name": "Bob"}}');

-- Accessing JSONB data
SELECT
    data->>'type'              AS event_type,        -- text: 'click'
    data->'user'->>'name'      AS user_name,         -- text: 'Alice'
    data->'user'->'id'         AS user_id,            -- jsonb: 1
    data#>>'{user,name}'       AS nested_name         -- text path: 'Alice'
FROM events;

-- Filtering
SELECT * FROM events WHERE data->>'type' = 'click';
SELECT * FROM events WHERE data @> '{"type": "click"}';  -- containment

-- JSONB operators
SELECT '{"a":1}'::JSONB || '{"b":2}'::JSONB;   -- merge: {"a":1,"b":2}
SELECT '{"a":1,"b":2}'::JSONB - 'a';           -- remove key: {"b":2}
SELECT '{"a":1}'::JSONB ? 'a';                 -- has key? true

-- GIN index for fast JSONB queries
CREATE INDEX idx_events_data ON events USING GIN (data);
```

| Operator | Purpose | Example |
|---|---|---|
| `->` | Get JSON object (returns JSONB) | `data->'user'` |
| `->>` | Get JSON value as text | `data->>'type'` |
| `#>` | Get nested JSONB by path | `data#>'{user,name}'` |
| `#>>` | Get nested text by path | `data#>>'{user,name}'` |
| `@>` | Contains | `data @> '{"type":"click"}'` |
| `?` | Has key | `data ? 'type'` |
| `\|\|` | Merge/concat | `obj1 \|\| obj2` |

---

## 24. Array Data Type

**Q: How do arrays work in PostgreSQL?**

```sql
-- Column with array type
CREATE TABLE projects (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    tags TEXT[],
    scores INT[]
);

INSERT INTO projects (name, tags, scores) VALUES
('Web App', ARRAY['react', 'typescript', 'node'], ARRAY[85, 90, 78]),
('API',     '{"go", "grpc", "docker"}',            '{92, 88}'),
('Mobile',  ARRAY['flutter', 'dart'],              ARRAY[75, 80, 95]);

-- Access elements (1-indexed!)
SELECT name, tags[1] AS first_tag FROM projects;      -- 'react', 'go', 'flutter'

-- Array functions
SELECT name FROM projects WHERE 'react' = ANY(tags);   -- contains 'react'
SELECT name FROM projects WHERE tags @> ARRAY['node']; -- contains 'node'
SELECT name, ARRAY_LENGTH(tags, 1) AS tag_count FROM projects;
SELECT UNNEST(ARRAY[1,2,3]);                            -- expands: 1, 2, 3 (one per row)
SELECT ARRAY_AGG(name) FROM employees WHERE department = 'Engineering';
-- {Alice,Bob,Eve,Ivy}

-- Array operators
SELECT ARRAY[1,2,3] || ARRAY[4,5];      -- {1,2,3,4,5} (concatenate)
SELECT ARRAY[1,2,3] || 4;               -- {1,2,3,4} (append)
SELECT ARRAY[1,2,3] @> ARRAY[2,3];      -- true (contains)
SELECT ARRAY[1,2,3] && ARRAY[3,4,5];    -- true (overlap)
```

---

## 25. COPY & Bulk Insert

**Q: How do you import/export data in bulk?**

```sql
-- COPY: fastest way to bulk load (server-side file)
COPY employees FROM '/tmp/employees.csv' WITH (FORMAT CSV, HEADER TRUE);

-- \copy: client-side (works from psql)
\copy employees FROM 'employees.csv' WITH (FORMAT CSV, HEADER TRUE);

-- Export to CSV
COPY (SELECT * FROM employees WHERE is_active = TRUE)
TO '/tmp/active_employees.csv' WITH (FORMAT CSV, HEADER TRUE);

-- Bulk INSERT with multiple VALUES (alternative)
INSERT INTO employees (name, email, department, salary) VALUES
('Person1', 'p1@co.com', 'HR', 60000),
('Person2', 'p2@co.com', 'HR', 62000),
('Person3', 'p3@co.com', 'HR', 58000);

-- INSERT from SELECT
INSERT INTO employees (name, email, department, salary)
SELECT name, email, department, salary * 1.1
FROM employees
WHERE department = 'Sales';
```

| Method | Speed | Use Case |
|---|---|---|
| `COPY` | Fastest | Server has file access |
| `\copy` | Fast | File on client machine |
| `INSERT VALUES` | Medium | Small batches |
| `INSERT SELECT` | Medium | Copy/transform within DB |

---

## 26. Stored Functions (PL/pgSQL)

**Q: How do you create functions in PostgreSQL?**

```sql
-- Simple function: get employee count by department
CREATE OR REPLACE FUNCTION get_dept_count(dept_name VARCHAR)
RETURNS INT AS $$
DECLARE
    emp_count INT;
BEGIN
    SELECT COUNT(*) INTO emp_count
    FROM employees
    WHERE department = dept_name AND is_active = TRUE;
    
    RETURN emp_count;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT get_dept_count('Engineering');  -- 4

-- Function returning a TABLE
CREATE OR REPLACE FUNCTION get_top_earners(min_salary NUMERIC)
RETURNS TABLE(emp_name VARCHAR, emp_salary NUMERIC) AS $$
BEGIN
    RETURN QUERY
    SELECT name, salary
    FROM employees
    WHERE salary >= min_salary
    ORDER BY salary DESC;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM get_top_earners(85000);
```

| emp_name | emp_salary |
|----------|------------|
| Alice    | 95000      |
| Eve      | 92000      |
| Ivy      | 88000      |
| Bob      | 85000      |

```sql
-- Function vs Procedure
-- Function: returns value, can be used in SELECT
-- Procedure: no return, can manage transactions (COMMIT/ROLLBACK inside)

CREATE OR REPLACE PROCEDURE transfer_budget(from_dept INT, to_dept INT, amount NUMERIC)
LANGUAGE plpgsql AS $$
BEGIN
    UPDATE departments SET budget = budget - amount WHERE id = from_dept;
    UPDATE departments SET budget = budget + amount WHERE id = to_dept;
    COMMIT;
END;
$$;

CALL transfer_budget(1, 2, 50000);
```

---

## 27. Triggers

**Q: What is a trigger and how to create one?**

A **trigger** automatically executes a function when a specified event (INSERT, UPDATE, DELETE) occurs on a table.

```sql
-- Audit log table
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    table_name VARCHAR(50),
    action VARCHAR(10),
    old_data JSONB,
    new_data JSONB,
    changed_at TIMESTAMP DEFAULT NOW(),
    changed_by VARCHAR(100) DEFAULT CURRENT_USER
);

-- Trigger function
CREATE OR REPLACE FUNCTION log_employee_changes()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, action, new_data)
        VALUES ('employees', 'INSERT', row_to_json(NEW)::JSONB);
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, action, old_data, new_data)
        VALUES ('employees', 'UPDATE', row_to_json(OLD)::JSONB, row_to_json(NEW)::JSONB);
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, action, old_data)
        VALUES ('employees', 'DELETE', row_to_json(OLD)::JSONB);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Attach trigger to table
CREATE TRIGGER employee_audit
AFTER INSERT OR UPDATE OR DELETE ON employees
FOR EACH ROW EXECUTE FUNCTION log_employee_changes();

-- Now any change to employees is logged automatically
UPDATE employees SET salary = 100000 WHERE name = 'Alice';
SELECT * FROM audit_log;
```

| Timing | Fires |
|---|---|
| `BEFORE` | Before the operation — can modify NEW values |
| `AFTER` | After the operation — for logging, notification |
| `INSTEAD OF` | Replaces the operation (for views) |

---

## 28. Table Partitioning

**Q: What is table partitioning?**

Splits a large table into smaller, manageable pieces (partitions) while keeping a single logical table.

```sql
-- Range partitioning by order_date
CREATE TABLE orders_partitioned (
    id SERIAL,
    employee_id INT,
    product VARCHAR(100),
    amount NUMERIC(10,2),
    order_date DATE NOT NULL,
    status VARCHAR(20)
) PARTITION BY RANGE (order_date);

-- Create partitions
CREATE TABLE orders_2024_q1 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE orders_2024_q2 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

CREATE TABLE orders_2024_q3 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2024-07-01') TO ('2024-10-01');

-- Query works on parent — PostgreSQL routes to correct partition
SELECT * FROM orders_partitioned WHERE order_date = '2024-02-15';
-- Only scans orders_2024_q1 (partition pruning)
```

| Partition Type | Split By | Example |
|---|---|---|
| **RANGE** | Value ranges | Date ranges, ID ranges |
| **LIST** | Specific values | Country, status, department |
| **HASH** | Hash of column | Even distribution |

```sql
-- List partitioning
CREATE TABLE employees_by_dept (
    id INT, name VARCHAR(100), department VARCHAR(50)
) PARTITION BY LIST (department);

CREATE TABLE emp_engineering PARTITION OF employees_by_dept FOR VALUES IN ('Engineering');
CREATE TABLE emp_hr          PARTITION OF employees_by_dept FOR VALUES IN ('HR');
CREATE TABLE emp_sales       PARTITION OF employees_by_dept FOR VALUES IN ('Sales');
```

---

## 29. Normalization

**Q: What is normalization? Explain 1NF, 2NF, 3NF.**

| Normal Form | Rule | Violation Example |
|---|---|---|
| **1NF** | Each column has atomic (single) values, no repeating groups | `skills = 'Java, Python'` in one cell |
| **2NF** | 1NF + no partial dependency (non-key column depends on full PK) | In composite PK (student_id, course_id): student_name depends only on student_id |
| **3NF** | 2NF + no transitive dependency (non-key depends on another non-key) | dept_name depends on dept_id, not directly on employee PK |

### 1NF Violation → Fix

```sql
-- ❌ NOT 1NF: multi-valued column
| id | name  | phone_numbers          |
|----|-------|------------------------|
| 1  | Alice | 9876543210, 9123456789 |

-- ✅ 1NF: atomic values (separate rows or separate table)
CREATE TABLE phones (
    employee_id INT REFERENCES employees(id),
    phone VARCHAR(15)
);
| employee_id | phone      |
|-------------|------------|
| 1           | 9876543210 |
| 1           | 9123456789 |
```

### 2NF Violation → Fix

```sql
-- ❌ NOT 2NF: composite PK, partial dependency
-- PK = (student_id, course_id)
-- student_name depends only on student_id (partial dependency)
| student_id | course_id | student_name | grade |
|------------|-----------|--------------|-------|
| 1          | 101       | Alice        | A     |

-- ✅ 2NF: separate into two tables
-- students(student_id PK, student_name)
-- enrollments(student_id, course_id, grade) → composite PK
```

### 3NF Violation → Fix

```sql
-- ❌ NOT 3NF: transitive dependency
-- employee PK → dept_id → dept_name (dept_name depends on dept_id, not PK)
| emp_id | name  | dept_id | dept_name   |
|--------|-------|---------|-------------|
| 1      | Alice | 10      | Engineering |

-- ✅ 3NF: separate department into its own table
-- employees(emp_id PK, name, dept_id FK)
-- departments(dept_id PK, dept_name)
```

---

## 30. Performance Tuning & Best Practices

**Q: How do you optimize PostgreSQL queries?**

### Indexing Strategy

```sql
-- Index columns used in WHERE, JOIN, ORDER BY
CREATE INDEX idx_emp_dept ON employees(department);

-- Composite index: order matters! (left-to-right)
CREATE INDEX idx_emp_dept_sal ON employees(department, salary);
-- ✅ WHERE department = 'Eng'
-- ✅ WHERE department = 'Eng' AND salary > 80000
-- ❌ WHERE salary > 80000  (can't skip left column)

-- Covering index: include extra columns to avoid table lookup
CREATE INDEX idx_emp_covering ON employees(department) INCLUDE (name, salary);
```

### Query Optimization

```sql
-- ❌ SELECT * — fetches all columns
SELECT * FROM employees;

-- ✅ SELECT only needed columns
SELECT name, salary FROM employees;

-- ❌ Function on indexed column (index not used)
SELECT * FROM employees WHERE UPPER(name) = 'ALICE';

-- ✅ Expression index
CREATE INDEX idx_emp_name_upper ON employees(UPPER(name));

-- ❌ LIKE '%text%' (can't use B-Tree index)
-- ✅ LIKE 'text%' (can use B-Tree index)
-- ✅ Use pg_trgm extension for %text% searches
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_emp_name_trgm ON employees USING GIN (name gin_trgm_ops);
```

### Common Best Practices

| Practice | Why |
|---|---|
| Use `EXPLAIN ANALYZE` | Understand query plan before optimizing |
| Avoid `SELECT *` | Fetch only needed columns |
| Use `EXISTS` over `IN` for subqueries | Short-circuits, handles NULLs |
| Use `LIMIT` with `ORDER BY` | Avoid sorting entire table |
| Use connection pooling (PgBouncer) | Reduce connection overhead |
| Vacuum regularly | Reclaim dead tuples, update statistics |
| Use `COPY` for bulk loads | 10-100x faster than INSERT |
| Partition large tables | Faster queries via partition pruning |
| Use JSONB over JSON | Indexable, faster reads |
| Avoid N+1 queries | Use JOINs or batch queries |

```sql
-- VACUUM: clean up dead rows
VACUUM ANALYZE employees;

-- Check table statistics
SELECT relname, n_live_tup, n_dead_tup, last_vacuum, last_analyze
FROM pg_stat_user_tables
WHERE relname = 'employees';

-- Connection and session settings
SET work_mem = '256MB';              -- memory for sort/hash operations
SET effective_cache_size = '4GB';    -- estimated OS cache
SET random_page_cost = 1.1;         -- for SSD (default 4.0 for HDD)
```

---

## Quick Revision Table

| # | Topic | One-Line Key Point |
|---|---|---|
| 1 | Constraints | PK = NOT NULL + UNIQUE; FK = referential integrity |
| 2 | JOINs | INNER=matching; LEFT=all left; RIGHT=all right; FULL=everything |
| 3 | GROUP BY/HAVING | WHERE filters rows; HAVING filters groups |
| 4 | Subqueries | Nested queries in WHERE, FROM, SELECT, or EXISTS |
| 5 | Window Functions | ROW_NUMBER/RANK/DENSE_RANK over partitions without collapsing |
| 6 | LEAD/LAG | Access previous/next row values in window |
| 7 | CTE | `WITH name AS (...)` — named temporary result sets |
| 8 | Recursive CTE | Hierarchical queries — org charts, trees |
| 9 | Set Operations | UNION (dedupe), UNION ALL (keep), INTERSECT, EXCEPT |
| 10 | Indexes | B-Tree (default), GIN (JSONB/arrays), create on WHERE/JOIN cols |
| 11 | EXPLAIN | Query plan analysis — Seq Scan vs Index Scan |
| 12 | Views | View = virtual; Materialized View = cached on disk |
| 13 | Transactions | BEGIN/COMMIT/ROLLBACK — ACID properties |
| 14 | Isolation Levels | Read Committed (default) → Serializable (strictest) |
| 15 | UPSERT | `INSERT ON CONFLICT DO UPDATE/NOTHING` |
| 16 | COALESCE/CASE | COALESCE = first non-null; CASE = conditional logic |
| 17 | String Functions | UPPER, LOWER, TRIM, SUBSTRING, CONCAT, LIKE, ILIKE |
| 18 | Date Functions | EXTRACT, AGE, DATE_TRUNC, INTERVAL, TO_CHAR |
| 19 | Aggregates | COUNT, SUM, AVG, MIN, MAX, STRING_AGG, ARRAY_AGG |
| 20 | EXISTS vs IN | EXISTS handles NULLs; NOT IN fails with NULLs |
| 21 | ANY/ALL | ANY = at least one; ALL = every value |
| 22 | LATERAL | Correlated subquery in FROM — reference outer columns |
| 23 | JSONB | Binary JSON — indexable, operators: ->, ->>, @>, ? |
| 24 | Arrays | `TEXT[]`, ANY(), @>, UNNEST(), ARRAY_AGG() |
| 25 | COPY | Fastest bulk import/export — CSV, binary formats |
| 26 | Functions | PL/pgSQL — `CREATE FUNCTION`, returns value/table |
| 27 | Triggers | Auto-execute function on INSERT/UPDATE/DELETE |
| 28 | Partitioning | RANGE, LIST, HASH — split large tables for performance |
| 29 | Normalization | 1NF=atomic; 2NF=no partial dep; 3NF=no transitive dep |
| 30 | Performance | Index wisely, EXPLAIN, VACUUM, avoid SELECT *, use EXISTS |
