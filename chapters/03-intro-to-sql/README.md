# 📖 Chapter 3: Introduction to SQL

> **Textbook:** *Database System Concepts (7th Edition)*  
> **Authors:** Abraham Silberschatz, Henry F. Korth, S. Sudarshan  
> **Chapter Focus:** Basic Query Structure, Filtering, Aggregation, Grouping, Set Operations, Subqueries & Common Mental Models.

---

## 1. Basic Query Structure

```sql
SELECT columns
FROM table
WHERE condition;
```

| Clause | Purpose |
| :--- | :--- |
| `FROM` | Choose table(s) |
| `WHERE` | Filter rows |
| `SELECT` | Choose columns |

---

## 2. `SELECT`, `*`, and `DISTINCT`

```sql
-- Select specific columns
SELECT name, salary
FROM instructor;

-- Select all columns
SELECT *
FROM instructor;

-- Select all columns explicitly from a table
SELECT instructor.*
FROM instructor;

-- Remove duplicate results
SELECT DISTINCT dept_name
FROM instructor;
```

---

## 3. Strings & Formatting

- Strings use **single quotes**: `'Computer Science'`
- A literal single quote inside a string is escaped by writing it twice: `'It''s right'`
- String comparison is standard case-sensitive (dependent on RDBMS collation).

---

## 4. Pattern Matching (`LIKE`)

| Pattern | Meaning |
| :---: | :--- |
| `%` | Match zero or more characters |
| `_` | Match exactly one character |

```sql
LIKE 'A%'       -- starts with A
LIKE '%son'     -- ends with son
LIKE '%son%'    -- contains son
LIKE 'A___'     -- A + exactly 3 characters
LIKE 'A___%'    -- A + at least 3 characters
```

### Opposite (`NOT LIKE`):
```sql
WHERE name NOT LIKE 'A%';
```

### Escaping `%` and `_`:
```sql
-- Escape character specification
WHERE name LIKE 'ab\%cd%' ESCAPE '\';
```

---

## 5. Aliases (`AS`)

```sql
-- Rename a column in output
SELECT name AS instructor_name
FROM instructor;

-- Rename a table (Tuple Variable)
SELECT T.name
FROM instructor AS T;

-- Self-join comparison using aliases
SELECT T.name
FROM instructor AS T, instructor AS S
WHERE T.salary > S.salary;
```

---

## 6. Multiple Tables & Cartesian Product

```sql
FROM instructor, teaches
```
Logically produces the Cartesian product:
$$\text{instructor} \times \text{teaches}$$

Every row of one table is combined with every row of the other ($12 \times 13 = 156$ combinations). A join condition in `WHERE` or `JOIN ... ON` is necessary to connect meaningful related rows.

---

## 7. Joining Tables

```sql
-- Old-style SQL (Theta Join in WHERE clause):
SELECT name, course_id
FROM instructor, teaches
WHERE instructor.ID = teaches.ID;

-- Modern ANSI SQL:
SELECT name, course_id
FROM instructor
JOIN teaches
  ON instructor.ID = teaches.ID;
```

```text
teaches.ID (Foreign Key) ──> instructor.ID (Primary Key)
```

---

## 8. Primary Key vs. Foreign Key

| Key Type | Purpose |
| :--- | :--- |
| **Primary Key** | Uniquely identifies each row in a relation |
| **Foreign Key** | References a primary/candidate key in another relation |

**Composite Primary Key Example:**
```text
(ID, course_id, sec_id, semester, year)
```
The entire composite tuple must be unique.

---

## 9. Ordering Results (`ORDER BY`)

```sql
ORDER BY name;                 -- Default is ASC
ORDER BY salary DESC;          -- Descending
ORDER BY salary DESC, name ASC;-- Multi-column sort
```

---

## 10. Range Queries (`BETWEEN`)

```sql
WHERE salary BETWEEN 90000 AND 100000;
-- Equivalent to: salary >= 90000 AND salary <= 100000 (Inclusive on both ends)

WHERE salary NOT BETWEEN 90000 AND 100000;
```

---

## 11. Row Constructors (Tuples)

Compare multiple attributes simultaneously:

```sql
WHERE (ID, dept_name) = (10101, 'Biology');
-- Equivalent to: WHERE ID = 10101 AND dept_name = 'Biology';
```

---

## 12. Aggregate Functions

| Function | Purpose |
| :--- | :--- |
| `AVG()` | Average of numeric values |
| `SUM()` | Sum of values |
| `MIN()` | Minimum value |
| `MAX()` | Maximum value |
| `COUNT()` | Total number of rows/values |

```sql
COUNT(*)            -- Count all rows (including NULLs)
COUNT(ID)           -- Count non-NULL IDs
COUNT(DISTINCT ID)  -- Count unique non-NULL IDs
```

---

## 13. `GROUP BY`

Partitions rows into subsets so aggregate functions run separately per group:

```sql
SELECT dept_name, AVG(salary)
FROM instructor
GROUP BY dept_name;
```

```text
GROUP BY ──> Split rows into groups ──> Aggregate (AVG/SUM) separately per group
```

---

## 14. `HAVING` (Group Filtering)

- **`WHERE`**: Filters **rows** *before* grouping.
- **`HAVING`**: Filters **groups** *after* aggregation.

```sql
SELECT dept_name, AVG(salary)
FROM instructor
GROUP BY dept_name
HAVING AVG(salary) > 70000;
```

### SQL Clause Execution Order:
```text
1. FROM       ──> Identify source tables
2. WHERE      ──> Filter raw rows
3. GROUP BY   ──> Form groups
4. HAVING     ──> Filter groups
5. SELECT     ──> Project desired columns & aggregates
6. ORDER BY   ──> Sort final output
```

---

## 15. Handling `NULL` Values

- `NULL` represents **unknown / missing value** (not `0`, `''`, or `false`).
- Comparisons with `NULL` evaluate to `UNKNOWN` in three-valued logic (`TRUE`, `FALSE`, `UNKNOWN`).
- Check using: `IS NULL` or `IS NOT NULL`.

---

## 16. Set Operations

| Operation | Standard (Distinct) | Keep Duplicates |
| :--- | :--- | :--- |
| **Union** | `UNION` | `UNION ALL` |
| **Intersection** | `INTERSECT` | `INTERSECT ALL` |
| **Set Difference** | `EXCEPT` | `EXCEPT ALL` |

```sql
query1
UNION
query2;
```

---

## 17. Subqueries & Set Membership

### `IN` / `NOT IN`:
```sql
WHERE course_id IN (
    SELECT course_id
    FROM section
    WHERE semester = 'Spring'
);
```

### `SOME` / `ANY` (At least one):
```sql
WHERE salary > SOME (
    SELECT salary
    FROM instructor
    WHERE dept_name = 'Biology'
);
-- Note: "= SOME" is equivalent to "IN"
```

### `ALL` (Every one):
```sql
WHERE salary > ALL (
    SELECT salary
    FROM instructor
    WHERE dept_name = 'Biology'
);
-- Note: "<> ALL" is equivalent to "NOT IN"
```

---

## 18. `EXISTS` & `NOT EXISTS`

Tests whether a subquery returns any tuples (evaluates to true if at least 1 row is returned):

```sql
WHERE EXISTS (
    SELECT *
    FROM teaches
    WHERE teaches.ID = instructor.ID
);
```

---

## 19. Correlated Subqueries

A subquery is **correlated** when it references attributes from the outer query:

```sql
SELECT S.course_id
FROM section AS S
WHERE EXISTS (
    SELECT *
    FROM section AS T
    WHERE T.course_id = S.course_id AND T.year = 2025
);
```

---

## 20. Subqueries in the `FROM` Clause

Treat query results as temporary derived tables:

```sql
SELECT dept_name, avg_salary
FROM (
    SELECT dept_name, AVG(salary) AS avg_salary
    FROM instructor
    GROUP BY dept_name
) AS dept_avg
WHERE avg_salary > 42000;
```

---

## 21. Common Table Expressions (`WITH` Clause)

Defines named temporary relations local to the query:

```sql
WITH max_budget(value) AS (
    SELECT MAX(budget)
    FROM department
)
SELECT budget
FROM department, max_budget
WHERE department.budget = max_budget.value;
```

---

## 22. Scalar Subqueries

A subquery that evaluates to **exactly 1 row and 1 column** (a single atomic value):

```sql
SELECT dept_name,
       (
           SELECT COUNT(*)
           FROM instructor
           WHERE instructor.dept_name = department.dept_name
       ) AS num_instructors
FROM department;
```

---

## 🎯 Master Summary Mental Model

```text
FROM
 ↓
WHERE          → Which rows?
 ↓
JOIN           → Which rows connect?
 ↓
GROUP BY       → Which rows group together?
 ↓
HAVING         → Which groups qualify?
 ↓
SELECT         → What to output?
 ↓
ORDER BY       → What order?
```

### Subquery Types:
```text
SUBQUERY
├── IN / NOT IN         → Set membership
├── SOME / ALL          → Set comparison
├── EXISTS / NOT EXISTS → Row existence check
├── FROM (subquery)     → Derived table
├── WITH                → Named temporary CTE
└── Scalar subquery    → Single-value projection
```
