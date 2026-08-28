# Chapter 5: Advanced SQL

> **Textbook:** *Database System Concepts (7th Edition)*  
> **Authors:** Abraham Silberschatz, Henry F. Korth, S. Sudarshan  
> **Chapter Focus:** Accessing SQL from Applications, Functions & Procedures, Triggers, Recursive Queries, and Advanced Aggregation (Ranking, Windowing, Pivot, ROLLUP, CUBE).

---

## 1. Accessing SQL from a Programming Language

A programming language works with variables and objects; SQL works with relations. We need a mechanism to bridge the two.

| Approach | When SQL is Identified | Description |
| :--- | :--- | :--- |
| **Dynamic SQL** | At runtime | Program constructs/submits SQL at runtime (JDBC, ODBC) |
| **Embedded SQL** | At compile time | SQL embedded in source code, processed by a preprocessor |

Most current systems use dynamic SQL.

---

### JDBC

JDBC is the standard Java API for communicating with databases.

```text
Connection → Statement / PreparedStatement → executeQuery / executeUpdate → ResultSet
```

```java
Connection conn = DriverManager.getConnection(...);

PreparedStatement ps =
    conn.prepareStatement(
        "SELECT * FROM instructor WHERE dept_name = ?");

ps.setString(1, "Physics");

ResultSet rs = ps.executeQuery();

while (rs.next()) {
    System.out.println(rs.getString("name"));
}
```

### Prepared Statements

Instead of concatenating user input into SQL (which causes **SQL injection**), use parameterized queries:

```java
// DANGEROUS — never do this
"SELECT * FROM instructor WHERE name = '" + name + "'"

// SAFE — use parameterized queries
"SELECT * FROM instructor WHERE name = ?"
```

Prepared statements treat input as **data**, not SQL code. They also improve efficiency when the same query is executed repeatedly.

> **Rule:** Never concatenate user-controlled values into SQL.

### ODBC

ODBC (Open Database Connectivity) is a language-agnostic API, originally designed for C:

```text
Application → ODBC API → ODBC driver → DBMS
```

### Metadata

Metadata is information about the database/result itself (column names, types, tables, keys). This lets applications work with arbitrary schemas without hard-coding structure.

---

## 2. Functions and Procedures

The idea is to put **reusable business logic inside the database**, giving multiple applications a single implementation.

### Function

A function returns a value:

```text
dept_count('Physics') → 5
```

A function can also return an entire **table** — think of it as a **parameterized view**:

```text
instructor_of('Physics') → table of Physics instructors
```

### Procedure

A procedure performs an operation and can have `IN`/`OUT` parameters:

```text
CALL register_student(student, course) → check capacity → INSERT or error
```

### Function vs Procedure

| | Function | Procedure |
| :--- | :--- | :--- |
| **Returns** | A value (or table) | Nothing required |
| **Used in** | SQL expressions / queries | Called using `CALL` |
| **Parameters** | Input only | `IN` / `OUT` / `INOUT` |

### Procedural SQL (PSM)

SQL's procedural extensions provide: `DECLARE`, `SET`, `BEGIN...END`, `IF/ELSE`, `WHILE`, `REPEAT`, `FOR`, `CASE`, exceptions, handlers.

**Tradeoff:** Centralized logic and enforcement close to the data, but DBMS-specific syntax makes migration harder.

---

## 3. Triggers

A trigger is different from a function/procedure because **you don't explicitly call it** — it fires automatically.

```text
Trigger = Event + Condition + Action
```

Events: `INSERT`, `UPDATE`, `DELETE`

### `BEFORE` vs `AFTER`

| Timing | Runs | Useful For |
| :--- | :--- | :--- |
| `BEFORE` | Before the operation | Validation, modifying values, preventing invalid operations |
| `AFTER` | After the event | Logging, updating related tables, maintaining derived data |

### `OLD` and `NEW`

For an update, the trigger can reference:

```text
OLD salary = 80,000
NEW salary = 90,000
```

This lets the trigger reason about **what changed**, not merely the current value.

| Event | Available References |
| :--- | :--- |
| `UPDATE` | `OLD` (before), `NEW` (after) |
| `INSERT` | `NEW` only |
| `DELETE` | `OLD` only |

### Trigger Tradeoff

Triggers are powerful because they're automatic, but that is also their biggest weakness. They create **implicit behavior** that can be difficult to debug, and can cause chains or even infinite triggering.

> Use triggers when automatic behavior is genuinely needed, not simply because you can. If a normal constraint or materialized view can express the rule, prefer those.

---

## 4. Recursive Queries

### The Problem

Some data is naturally **hierarchical** or **graph-like**:

```text
CS-347 → CS-319 → CS-315 → CS-190 → CS-101
```

A normal join only gets you one level. To find **all direct and indirect** prerequisites, you need **transitive closure**.

### Recursive SQL

```sql
WITH RECURSIVE result(...) AS (

    -- Base case
    SELECT ...

    UNION

    -- Recursive case
    SELECT ...
    FROM result
    JOIN ...

)
SELECT * FROM result;
```

**How it works:**

```text
Iteration 1: A → B
Iteration 2: A → B, A → C
Iteration 3: A → B, A → C, A → D
Iteration 4: No new rows → Stop (Fixed Point)
```

> **Fixed point** = another iteration produces no new tuples.

Without recursion, you'd need to know the hierarchy depth in advance and write that many joins. Recursive SQL handles **unknown/unbounded depth**.

### Edge Case: Cycles

If `A → B → C → A`, recursion could loop forever. Recursive queries handle this by ensuring already-seen tuples aren't repeatedly added.

### Monotonicity

Recursive queries must be **monotonic**: adding more tuples to the recursive relation must never make the result smaller. Therefore certain operations are restricted (aggregation on the recursive view, `NOT EXISTS`, set difference).

---

## 5. Advanced Aggregation Features

### 5.1 Ranking

```sql
SELECT id,
       RANK() OVER (ORDER BY GPA DESC) AS student_rank
FROM student_grades;
```

| Function | Behavior |
| :--- | :--- |
| `RANK()` | Preserves ties, **leaves gaps** (1, 1, 3) |
| `DENSE_RANK()` | Preserves ties, **no gaps** (1, 1, 2) |
| `ROW_NUMBER()` | Unique position per row, even for ties (1, 2, 3) |

#### Ranking Within Groups

```sql
RANK() OVER (
    PARTITION BY dept_name
    ORDER BY GPA DESC
)
```

`PARTITION BY` divides the result into independent groups, then ranks separately inside each group.

#### Top-N Queries

```sql
SELECT * FROM (
    SELECT id, RANK() OVER (ORDER BY GPA DESC) AS r
    FROM student_grades
) WHERE r <= 5;
```

> This can return **more than 5 rows** if multiple students tie at rank 5.

---

### 5.2 Windowing

The key distinction from `GROUP BY`:

| | `GROUP BY` | Window Function |
| :--- | :--- | :--- |
| **Row count** | Collapses rows (100 → 5) | **Keeps all rows** (100 → 100 + computed value) |
| **Purpose** | Aggregate per group | Aggregate over a **sliding window** per row |

```sql
AVG(credits) OVER (
    ORDER BY year
    ROWS UNBOUNDED PRECEDING
)
```

```text
2022 → avg(100)              = 100
2023 → avg(100,120)          = 110
2024 → avg(100,120,150)      = 123.3
2025 → avg(100,120,150,180)  = 137.5
```

#### Window Frame

```sql
ROWS BETWEEN 3 PRECEDING AND CURRENT ROW
ROWS BETWEEN 3 PRECEDING AND 2 FOLLOWING
ROWS UNBOUNDED PRECEDING
```

Combine with `PARTITION BY` so each group gets its own window calculations.

---

### 5.3 Pivot / Cross-Tabulation

A **pivot** turns values of one attribute into columns:

```text
Before:                          After:
item   color   quantity          item    dark  pastel  white
dress  dark      2               dress    2      4       5
dress  pastel    4
dress  white     5
```

Useful for **human analysis and reporting**. Pivot syntax is not equally supported by every DBMS.

---

### 5.4 `ROLLUP`

Produces **hierarchical subtotals + grand total**:

```sql
GROUP BY ROLLUP(item, color)
```

```text
Grouping sets produced:
(item, color)  ← detail
(item)         ← subtotal per item
()             ← grand total
```

### `CUBE`

Produces **all possible grouping combinations**:

```sql
GROUP BY CUBE(item, color, size)
```

For 3 attributes → 2³ = 8 subsets. Powerful for analytics, but result size grows exponentially.

| | Grouping Strategy |
| :--- | :--- |
| `ROLLUP` | Hierarchical / prefix totals |
| `CUBE` | All possible combinations |

### `GROUPING()`

`ROLLUP`/`CUBE` use `NULL` for subtotal rows, but data can also contain `NULL`. `GROUPING(column)` distinguishes the two:

| Return | Meaning |
| :--- | :--- |
| `1` | `NULL` generated by `ROLLUP`/`CUBE` |
| `0` | Real `NULL` from the data |

---

## The Big Picture

Chapter 5 teaches five ways SQL becomes more powerful:

| Capability | Mechanism |
| :--- | :--- |
| Connect SQL to applications | JDBC, ODBC, Embedded SQL |
| Put reusable logic in the database | Functions, Procedures |
| Make the database react automatically | Triggers |
| Deal with hierarchical/graph data | `WITH RECURSIVE`, Transitive Closure, Fixed Point |
| Do sophisticated analytics | Ranking, Windowing, Pivot, ROLLUP, CUBE |

> SQL isn't only about `SELECT ... FROM ... WHERE ...` anymore. Chapter 5 shows how SQL can communicate with applications, encapsulate logic, react automatically to changes, traverse recursive relationships, and perform analytical computations.
