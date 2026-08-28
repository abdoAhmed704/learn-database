# Chapter 5: Advanced SQL

> **Textbook:** *Database System Concepts (7th Edition)*  
> **Authors:** Abraham Silberschatz, Henry F. Korth, S. Sudarshan  
> **Chapter Focus:** Accessing SQL from Applications, Functions & Procedures, Triggers, Recursive Queries, and Advanced Aggregation (Ranking, Windowing, Pivot, ROLLUP, CUBE).

The chapter is essentially about extending SQL beyond basic querying:

```text
Programming languages ↔ SQL/DBMS
Functions & Procedures
Triggers
Recursive Queries
Advanced Aggregation
Data Analysis
```

---

# 5.1 Accessing SQL from a Programming Language

A normal programming language such as Java works with **variables/objects**, while SQL works with **relations (tables)**.

The problem is:

```text
Java                         SQL
----                         ---
variable                     relation
object                       tuple/row
ResultSet                    query result
```

You need a mechanism to let the two worlds communicate.

There are two approaches:

### Dynamic SQL

The program constructs/submits SQL at runtime.

```text
Java
 ↓
build SQL
 ↓
DBMS
 ↓
ResultSet
```

### Embedded SQL

SQL is embedded into the source code and processed by a **preprocessor before compilation**.

```text
C + SQL
   ↓
preprocessor
   ↓
C code
   ↓
compiler
```

The key distinction is **when SQL is integrated**: runtime for dynamic SQL versus preprocessing/compile time for embedded SQL.

---

## JDBC

**JDBC = Java API for communicating with databases.**

Typical flow:

```text
Connection
    ↓
Statement / PreparedStatement
    ↓
executeQuery / executeUpdate
    ↓
ResultSet
```

Example:

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

Instead of:

```java
"SELECT * FROM instructor WHERE name = '" + name + "'"
```

use:

```java
"SELECT * FROM instructor WHERE name = ?"
```

and bind:

```java
ps.setString(1, name);
```

**Why?**

Because concatenating user input can cause **SQL injection**.

```text
User input
   ↓
SQL string concatenation
   ↓
Input becomes SQL code
   ↓
SQL Injection
```

Prepared statements treat the input as **data**, not SQL code. They can also improve efficiency when the same query is executed repeatedly.

**Rule:**

> Never concatenate user-controlled values into SQL.

---

## ODBC

**ODBC = another API for applications to communicate with DBMSs**, originally designed around C and later used by other languages.

Conceptually:

```text
C/C++/C#/...
      ↓
    ODBC
      ↓
    DBMS
```

The application uses calls such as:

```text
SQLConnect()
SQLExecDirect()
SQLFetch()
SQLDisconnect()
```

ODBC also provides mechanisms for metadata, prepared statements, transactions, etc.

---

## Metadata

**Metadata = information about the database/result itself.**

For example:

```text
What columns exist?
What are their names?
What are their types?
How many columns?
What tables exist?
```

This lets an application work with a database without hard-coding the schema.

For example, a generic database browser can inspect an arbitrary table and display its columns.

---

# 5.2 Functions and Procedures

The idea is to put **reusable logic/business logic inside the database**.

Instead of every application implementing:

```text
"How many instructors are in Physics?"
```

you can define:

```text
dept_count('Physics')
       ↓
       5
```

This gives multiple applications a **single implementation of the rule**.

---

## Function

Conceptually:

```text
input → function → result
```

Example:

```text
dept_count('Physics') → 5
```

A function can also return a **table**.

### Table function

```text
instructor_of('Physics')
```

could return:

```text
ID     name       dept       salary
101    Ahmed      Physics    80000
102    Ali        Physics    85000
```

Think of a table function as a:

> **Parameterized view**

because you give it a parameter and it produces a relation based on that parameter.

---

## Procedure

A procedure is more about **performing an operation**, and can have:

```text
IN  → input
OUT → output
```

Conceptually:

```text
CALL register_student(student, course)

        ↓

check capacity

        ↓

if available → INSERT
else          → error
```

The textbook's `registerStudent` example does exactly this: it checks classroom capacity before inserting an enrollment.

---

## Procedural SQL

SQL's procedural extensions can provide:

```text
variables
assignments
BEGIN / END
IF / ELSE
WHILE
REPEAT
FOR
CASE
exceptions
handlers
```

This makes SQL procedural code much more like a programming language.

### Tradeoff

**Good:**

* Centralized business logic
* Multiple applications can reuse it
* Database can enforce rules close to the data

**Bad:**

* DBMS-specific syntax
* Harder to migrate between DBMSs
* More logic inside the database can make systems harder to reason about

The textbook emphasizes that Oracle, SQL Server, PostgreSQL, etc. use different procedural languages/syntax.

---

# 5.3 Triggers

A trigger is different from a function/procedure because **you don't explicitly call it**.

```text
INSERT / UPDATE / DELETE
          ↓
       Trigger
          ↓
    automatic action
```

A trigger consists conceptually of:

```text
Event
  +
Condition
  +
Action
```

For example:

```text
UPDATE inventory
      ↓
Trigger checks level
      ↓
level <= minimum?
      ↓
YES → create reorder
```

---

## Why triggers?

They're useful for:

### 1. Automatic business rules

```text
Student enrolled
      ↓
Trigger
      ↓
Update total credits
```

### 2. Integrity constraints that are difficult to express otherwise

### 3. Automatic actions/auditing

---

## BEFORE vs AFTER

### BEFORE

```text
Trigger
 ↓
INSERT
```

Useful when you want to **modify/reject something before it happens**.

### AFTER

```text
INSERT
 ↓
Trigger
```

Useful when you want to **react to something that happened**.

---

## OLD vs NEW

For an update:

```text
OLD salary = 80,000
NEW salary = 90,000
```

This lets the trigger reason about **what changed**, not merely the current value.

---

## Important trigger tradeoff

Triggers are powerful because they're automatic.

But that is also their biggest weakness.

```text
UPDATE
 ↓
Trigger A
 ↓
UPDATE
 ↓
Trigger B
 ↓
INSERT
 ↓
Trigger C
 ↓
...
```

This creates **implicit behavior** that can be difficult to debug and understand.

Triggers can even cause chains of triggers or, in bad designs, infinite triggering.

Therefore:

> **Use triggers when automatic behavior is genuinely needed, not simply because you can implement something with a trigger.**

If a normal constraint can express the rule, prefer the constraint.

If a materialized view solves the problem, prefer that where appropriate.

---

# 5.4 Recursive Queries

This is one of the most important concepts in the chapter.

## The problem

Some data is naturally **hierarchical** or **graph-like**.

For example:

```text
CS-347
  ↓
CS-319
  ↓
CS-315
  ↓
CS-190
  ↓
CS-101
```

Suppose:

```text
prereq(course, prerequisite)
```

contains:

```text
CS-347 → CS-319
CS-319 → CS-315
CS-315 → CS-190
CS-190 → CS-101
```

If I ask:

> What are **all** prerequisites of CS-347?

A normal join only gets you one/few levels.

You need to repeatedly follow:

```text
course
 ↓
prerequisite
 ↓
prerequisite's prerequisite
 ↓
...
```

This is called **transitive closure**.

---

# Recursive SQL

SQL supports:

```sql
WITH RECURSIVE
```

The important structure is:

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

The recursive definition has **two parts**:

### Base query

The starting information.

```text
Direct prerequisites
```

### Recursive query

Uses the result generated so far to find another level.

```text
Prerequisites of prerequisites
```

The textbook explicitly describes recursive views this way.

---

## How recursion actually works

Suppose:

```text
A → B
B → C
C → D
```

The database conceptually does:

```text
Iteration 1:
A → B

Iteration 2:
A → B
A → C

Iteration 3:
A → B
A → C
A → D

Iteration 4:
No new rows
```

Stop.

This is called reaching a **fixed point**.

> **Fixed point = another iteration produces no new tuples.**

---

## Why recursion?

Without recursion, you'd need to know the hierarchy depth:

```sql
JOIN ...       -- level 1
JOIN ...       -- level 2
JOIN ...       -- level 3
...
```

What if tomorrow the hierarchy becomes 50 levels deep?

You can't reasonably write 50 joins.

Recursive SQL handles **unknown/unbounded depth**.

This is useful for:

* course prerequisites
* organizational hierarchies
* folder structures
* component/subcomponent relationships
* graph reachability
* flight routes

The textbook explicitly notes that reachability problems such as finding cities connected through flights can be handled similarly.

---

## Recursive query edge case: cycles

Consider:

```text
A → B
B → C
C → A
```

Now recursion could keep going forever:

```text
A → B
A → C
A → A
A → B
A → C
...
```

Therefore recursive queries need to handle cycles appropriately, often by ensuring already-seen tuples aren't repeatedly added.

The textbook discusses this explicitly for prerequisite/reachability graphs.

---

## Monotonicity

The textbook places an important restriction on recursive queries.

A recursive query must be **monotonic**.

Intuitively:

> Adding more tuples to the recursive relation should not cause previously obtained results to disappear.

For example:

```text
More input
   ↓
same results + possibly more results
```

not:

```text
More input
   ↓
some previous results disappear
```

Therefore certain operations are restricted in recursive queries, such as aggregation on the recursive view, certain `NOT EXISTS` uses, and set difference involving the recursive view.

---

# 5.5 Advanced Aggregation Features

Normal:

```sql
GROUP BY
COUNT()
SUM()
AVG()
```

handles many problems.

But analytical queries often ask more sophisticated questions:

```text
Who is #1?
What is the running average?
What is the total for each category AND overall?
Turn rows into columns?
Give me every possible grouping?
```

That's why SQL provides advanced aggregation features.

---

# 5.5.1 Ranking

Suppose:

```text
student    GPA
Ahmed      4.0
Ali        3.8
Omar       3.8
Sara       3.5
```

You want:

```text
Ahmed → 1
Ali   → 2
Omar  → 2
Sara  → 4
```

Use:

```sql
SELECT id,
       RANK() OVER (ORDER BY GPA DESC) AS student_rank
FROM student_grades;
```

### Why `RANK()` instead of just `ORDER BY`?

`ORDER BY` sorts.

`RANK()` **assigns a position** to every row.

---

## `RANK` vs `ROW_NUMBER`

Conceptually:

```text
GPA

100
90
90
80
```

`RANK()`:

```text
100 → 1
90  → 2
90  → 2
80  → 4
```

`ROW_NUMBER()`:

```text
100 → 1
90  → 2
90  → 3
80  → 4
```

So:

> `RANK()` preserves ties; `ROW_NUMBER()` gives every row a unique position.

There is also `DENSE_RANK()`, which preserves ties like `RANK()` but **does not leave gaps**:

```text
100 → 1
90  → 2
90  → 2
80  → 3   ← no gap
```

The textbook also discusses `PERCENT_RANK` and `CUME_DIST` for percentile/distribution analysis.

---

## Ranking within groups: `PARTITION BY`

Suppose you want:

> Rank students **within each department**.

```sql
RANK() OVER (
    PARTITION BY dept_name
    ORDER BY GPA DESC
)
```

Conceptually:

```text
Physics
  Ahmed → 1
  Ali   → 2

Finance
  Omar  → 1
  Sara  → 2
```

`PARTITION BY` means:

> Divide the result into independent groups, then perform the window operation separately inside each group.

---

## Top-N and ties

To get top 5:

```sql
SELECT *
FROM (
    SELECT id,
           RANK() OVER (ORDER BY GPA DESC) AS r
    FROM student_grades
)
WHERE r <= 5;
```

Important:

> This can return **more than 5 rows**.

Why?

Because of ties.

If three students tie at rank 5, all three are included.

---

# 5.5.2 Windowing

This is different from normal `GROUP BY`.

### `GROUP BY`

Collapses rows.

```text
100 rows
   ↓
GROUP BY department
   ↓
5 rows
```

### Window function

**Keeps the original rows** while calculating something using related rows.

```text
100 rows
   ↓
window function
   ↓
100 rows + calculated value
```

This distinction is extremely important.

---

## Example: Running average

Suppose:

```text
year   credits
2022   100
2023   120
2024   150
2025   180
```

You can calculate a running average:

```sql
AVG(credits) OVER (
    ORDER BY year
    ROWS UNBOUNDED PRECEDING
)
```

Conceptually:

```text
2022 → avg(100)              = 100
2023 → avg(100,120)          = 110
2024 → avg(100,120,150)      = 123.3
2025 → avg(100,120,150,180)  = 137.5
```

The window defines **which rows participate in the calculation for the current row**.

---

## Window frame

You can define exactly which rows belong to the window.

For example:

```sql
ROWS BETWEEN 3 PRECEDING AND CURRENT ROW
```

means:

```text
current row
+ previous 3 rows
```

Or:

```sql
ROWS BETWEEN 3 PRECEDING AND 2 FOLLOWING
```

means:

```text
3 rows before
+
current row
+
2 rows after
```

### `PARTITION BY` + window

You can perform the same calculation independently for each department:

```sql
AVG(credits) OVER (
    PARTITION BY dept_name
    ORDER BY year
    ROWS BETWEEN 3 PRECEDING AND CURRENT ROW
)
```

So each department has its **own window**.

---

# 5.5.3 Pivoting

A **pivot** changes the shape of data.

Suppose you have:

```text
item     color    quantity
dress    dark       2
dress    white      5
dress    pastel     4
```

Normal relational representation:

```text
item | color  | quantity
```

A pivot turns the values of `color` into **columns**:

```text
item    dark    pastel    white
dress    2        4         5
```

This is called a:

* cross-tab
* cross-tabulation
* pivot table

The textbook describes exactly this transformation.

### Why?

This format is often much easier for **human analysis/reporting**.

Instead of:

```text
dress dark 2
dress pastel 4
dress white 5
```

you see:

```text
        dark pastel white
dress     2     4     5
```

The values can also be aggregated:

```sql
SUM(quantity)
```

when multiple rows contribute to the same cell.

### Tradeoff

Pivot is mostly about **presentation/analysis**, not changing the underlying relational concept.

It can make reports much easier to read, but pivot syntax is not equally supported by every DBMS.

---

# 5.5.4 `ROLLUP`

Suppose:

```text
sales(item, color, quantity)
```

You want:

1. Total by `item + color`
2. Total by `item`
3. Grand total

Instead of writing three queries and combining them:

```sql
GROUP BY item, color
UNION
GROUP BY item
UNION
grand total
```

you can use:

```sql
GROUP BY ROLLUP(item, color)
```

The textbook explains that:

```text
ROLLUP(item, color)
```

creates:

```text
(item, color)
(item)
()
```

where `()` means the grand total.

Example result:

```text
item    color    total
dress   dark       20
dress   white       5
dress   NULL       25   ← total dress
shirt   dark       14
shirt   white      28
shirt   NULL       42   ← total shirt
NULL    NULL       67   ← grand total
```

### Mental model

> **ROLLUP = hierarchical subtotals + grand total.**

---

# `CUBE`

`CUBE` goes further.

```sql
GROUP BY CUBE(item, color, size)
```

generates **all combinations/subsets** of those attributes.

For 3 attributes:

```text
(item, color, size)
(item, color)
(item, size)
(color, size)
(item)
(color)
(size)
()
```

That's why `CUBE` produces more results than `ROLLUP`.

### Mental model

```text
ROLLUP → hierarchical combinations

CUBE   → every possible combination
```

### Tradeoff

`CUBE` is extremely useful for analytics, but the number of grouping combinations grows rapidly as you add dimensions.

For `n` dimensions:

```text
2^n
```

possible subsets.

So:

```text
3 dimensions → 8
5 dimensions → 32
10 dimensions → 1024
```

This is powerful but can generate a **large result**.

---

# `GROUPING()`

There is a subtle problem with `ROLLUP/CUBE`.

They use `NULL` to represent:

> "This attribute isn't part of this particular grouping."

But actual data can also contain `NULL`.

So:

```text
NULL
```

could mean either:

```text
real NULL value
```

or:

```text
subtotal/grand-total generated by ROLLUP/CUBE
```

`GROUPING(column)` tells you whether the NULL was generated by the grouping operation.

---

# The entire Advanced Aggregation section in one picture

```text
                 ADVANCED AGGREGATION
                         │
       ┌─────────────────┼──────────────────┐
       │                 │                  │
    Ranking           Windowing           Pivot
       │                 │                  │
   RANK()            OVER()          rows → columns
   ROW_NUMBER()      frame
   PERCENT_RANK      PARTITION
   CUME_DIST
       │
       └──────────────┬────────────────────┘
                      │
                 ROLLUP / CUBE
                      │
             multiple aggregations
                      │
              ROLLUP → hierarchy
              CUBE   → all combinations
```

---

# 5.6 The Big Picture

Chapter 5 is really teaching you **five different ways SQL becomes more powerful**:

### 1. Connect SQL to applications

```text
Java/C/etc.
    ↕
JDBC / ODBC / Embedded SQL
    ↕
DBMS
```

### 2. Put reusable logic in the database

```text
Functions
Procedures
```

### 3. Make the database react automatically

```text
INSERT / UPDATE / DELETE
          ↓
       Trigger
```

### 4. Deal with hierarchical/graph-like data

```text
WITH RECURSIVE
       ↓
Transitive closure
       ↓
Fixed point
```

### 5. Do sophisticated analytics

```text
Ranking
Windowing
Pivot
ROLLUP
CUBE
```

The textbook's own summary groups the chapter around exactly these ideas.

---

## What I would memorize for an exam/interview

| Concept                | Core idea                                              |
| ---------------------- | ------------------------------------------------------ |
| **JDBC**               | Java ↔ DBMS API                                        |
| **ODBC**               | General DB connectivity API                            |
| **Dynamic SQL**        | SQL executed/constructed at runtime                    |
| **Embedded SQL**       | SQL integrated before compilation                      |
| **Prepared Statement** | Parameterized SQL; prevents SQL injection              |
| **Metadata**           | Information about schema/results                       |
| **Function**           | Input → result                                         |
| **Table Function**     | Input → table                                          |
| **Procedure**          | Performs operations; IN/OUT                            |
| **Trigger**            | Event → automatic action                               |
| **OLD/NEW**            | Before/after row values                                |
| **Recursive Query**    | Query that refers to its own result                    |
| **Transitive Closure** | Direct + indirect relationships                        |
| **Fixed Point**        | Recursion stops producing new tuples                   |
| **Monotonic**          | Adding data doesn't remove previous results            |
| **RANK**               | Position while preserving ties                         |
| **ROW_NUMBER**         | Unique position per row                                |
| **PARTITION BY**       | Perform window calculation separately per group        |
| **Window**             | Related rows used for calculation while retaining rows |
| **Pivot**              | Rows/values → columns                                  |
| **ROLLUP**             | Hierarchical subtotals + grand total                   |
| **CUBE**               | All grouping combinations                              |

**The most important conceptual distinction in the whole chapter:** SQL isn't only about `SELECT ... FROM ... WHERE ...` anymore. Chapter 5 shows how SQL can **communicate with applications, encapsulate logic, react automatically to changes, traverse recursive relationships, and perform analytical computations**.
