# 📖 Chapter 4: Intermediate SQL

> **Textbook:** *Database System Concepts (7th Edition)*  
> **Authors:** Abraham Silberschatz, Henry F. Korth, S. Sudarshan  
> **Chapter Focus:** Accessing SQL from Applications, Functions & Procedures, Triggers, Integrity Constraints, Data Types & Schemas, Indexes, and Authorization.

---

## 1. Accessing SQL from a Programming Language

SQL is powerful for data manipulation, but applications often need to combine SQL with a host language (C, Java, Python, etc.).

| Approach | Description |
| :--- | :--- |
| **Embedded SQL** | SQL statements written directly inside program code |
| **Dynamic SQL** | SQL statements constructed and executed at runtime |
| **JDBC / ODBC** | APIs that allow programs to communicate with a database |

```sql
-- Example: a program might execute this and process rows in the host language
SELECT name
FROM instructor
WHERE dept_name = 'Physics';
```

```text
Application → Programming Language → SQL → Database → Result → Application
```

---

## 2. Functions and Procedures

Instead of writing the same SQL repeatedly, we can **package the operation**.

| Concept | Description |
| :--- | :--- |
| **Function** | Performs an operation and **returns a value** |
| **Procedure** | Performs an operation, potentially modifying data; does **not** have to return a value |

```sql
-- Conceptual example
FUNCTION get_salary(id)
    → return instructor salary

SELECT get_salary('101');
```

```text
Repeated operation → Function / Procedure → Reusable database logic
```

---

## 3. Triggers

A **trigger** is a rule that automatically executes when a specified event occurs (`INSERT`, `UPDATE`, `DELETE`).

```text
UPDATE instructor
       ↓
Trigger fires
       ↓
Insert old/new salary into salary_history
```

### Trigger Timing:

| Timing | Purpose |
| :--- | :--- |
| `BEFORE` | Execute before the triggering event |
| `AFTER` | Execute after the triggering event |
| `INSTEAD OF` | Replace the triggering event (useful with views) |

```text
User modifies view → INSTEAD OF trigger → Custom operation on underlying tables
```

---

## 4. Integrity Constraints

An **integrity constraint** is a rule that ensures the database remains valid and consistent.

### `NOT NULL`

Prevents a column from containing `NULL`.

```sql
name VARCHAR(20) NOT NULL
```

```text
name = NULL  → ❌
name = 'Ali' → ✓
```

### `UNIQUE`

Prevents duplicate values/combinations.

```sql
email VARCHAR(100) UNIQUE
```

```text
a@gmail.com → ✓ (first insert)
a@gmail.com → ❌ (duplicate)
```

### `CHECK`

Requires a condition not to evaluate to `FALSE`.

```sql
CHECK (budget > 0)
```

```text
budget = 5000  → ✓
budget = -5000 → ❌
```

### Foreign Key

A **foreign key** requires values to correspond to a key in another relation.

```sql
FOREIGN KEY (dept_name)
REFERENCES department(dept_name)
```

```text
course.dept_name → department.dept_name
```

### Foreign Key Actions

| Action | On Delete Behavior |
| :--- | :--- |
| `CASCADE` | Delete referencing rows |
| `SET NULL` | Set FK to `NULL` |
| `SET DEFAULT` | Set FK to default value |

### Assertions

An **assertion** is a general condition that should always hold for the database.

> **Note:** Most widely used DBMSs do not currently support `CREATE ASSERTION`.

### Integrity Constraints Mental Model:

```text
             Integrity
                │
    ┌───────────┼───────────┐
    ↓           ↓           ↓
NOT NULL      UNIQUE       CHECK
    │           │           │
 no NULL    no duplicate  condition
                │
                ↓
          FOREIGN KEY
                │
                ↓
       relationship between
            tables
```

---

## 5. SQL Data Types and Schemas

### Date and Time

```text
DATE       → '2026-08-27'
TIME       → '14:30:00'
TIMESTAMP  → '2026-08-27 14:30:00'
INTERVAL   → '1' DAY
```

```sql
DATE '2026-08-27' + INTERVAL '1' DAY
```

### `CAST`

Converts an expression to a specified type.

```sql
CAST(ID AS NUMERIC(5))
```

### `COALESCE`

Returns the first non-`NULL` value.

```sql
COALESCE(salary, 0)
```

```text
salary = 50000 → 50000
salary = NULL  → 0
```

### `DEFAULT`

Specifies the value automatically used when no value is provided.

```sql
tot_cred NUMERIC(3,0) DEFAULT 0
```

### Large Objects

| Type | Description |
| :--- | :--- |
| `BLOB` | Binary Large Object (images, videos) |
| `CLOB` | Character Large Object (documents) |

### User-Defined Types

Creates a **distinct type** to prevent accidental mixing of semantically different values.

```sql
CREATE TYPE Dollars AS NUMERIC(12,2) FINAL;
CREATE TYPE Pounds AS NUMERIC(12,2) FINAL;
-- Dollars ≠ Pounds (even though both are NUMERIC)
```

### Domain

A type with additional constraints/defaults.

```sql
CREATE DOMAIN YearlySalary
NUMERIC(8,2)
CHECK (VALUE >= 29000);
```

### Identity / Sequence

The DBMS can automatically generate unique IDs.

```text
INSERT new instructor → DBMS generates ID
```

A **sequence** is another mechanism for generating successive values.

### `CREATE TABLE AS`

Creates a table containing the result of a query.

```sql
CREATE TABLE music_instructors AS
SELECT *
FROM instructor
WHERE dept_name = 'Music';
```

```text
VIEW     → query definition → reflects underlying data
TABLE AS → copied result    → does NOT follow future changes
```

### Schemas and Catalogs

SQL organizes objects using **catalogs and schemas** to prevent naming conflicts.

```text
catalog → schema → tables / views / other objects
```

---

## 6. Index Definition in SQL

An **index** is a data structure that allows the DBMS to find tuples efficiently without scanning the entire relation.

```sql
CREATE INDEX dept_index
ON instructor(dept_name);
```

### `UNIQUE INDEX`

Requires unique search-key values.

```sql
CREATE UNIQUE INDEX idx
ON instructor(ID);
```

### Dropping an Index

```sql
DROP INDEX dept_index;
```

### Index Trade-offs:

```text
Index
 ├── ✓ Faster searches
 ├── ✓ Helps constraint enforcement
 ├── ✗ Uses storage
 └── ✗ Makes INSERT/UPDATE/DELETE more expensive (must maintain the index)
```

---

## 7. Authorization

**Authorization** determines what a user is allowed to do with the database.

| Privilege | Action |
| :--- | :--- |
| `SELECT` | Read |
| `INSERT` | Add |
| `UPDATE` | Modify |
| `DELETE` | Remove |

### `GRANT`

Gives a privilege to a user.

```sql
GRANT SELECT
ON department
TO Amit;
```

### `REVOKE`

Removes a privilege from a user.

```sql
REVOKE SELECT
ON department
FROM Amit;
```

### Roles

A **role** is a named collection of privileges that can be assigned to users.

```sql
CREATE ROLE instructor;

GRANT SELECT ON takes
TO instructor;

GRANT instructor TO Amit;
-- Amit now gets the privileges of the instructor role
```

Roles can also be granted to other roles, creating a chain of inherited privileges.

### Authorization on Views

Create a restricted view to give a user access to only part of a table:

```sql
CREATE VIEW geo_instructor AS
SELECT *
FROM instructor
WHERE dept_name = 'Geology';

-- Grant access to the view, not the underlying table
GRANT SELECT ON geo_instructor TO employee;
```

### `WITH GRANT OPTION`

Allows a user to grant received privileges to others.

```sql
GRANT SELECT
ON department
TO Amit
WITH GRANT OPTION;
-- Amit can now grant SELECT on department to other users
```

### Authorization Graph

The graph represents who granted a privilege to whom. A user has the privilege if there is a path from the DBA/root to that user.

```text
DBA
 ├── U1 ──→ U4
 │    └──→ U5
 │
 └── U2 ──→ U5
```

### Cascading Revocation

When a privilege is revoked, dependent grants may also be lost:

```text
U1 → U4       (DBA revokes U1 → U4 loses privilege too)

U1 → U5
U2 → U5       (DBA revokes U1 → U5 keeps it via U2)
```

Use `RESTRICT` to prevent cascading:

```sql
REVOKE SELECT
ON department
FROM Amit
RESTRICT;
-- Fails if cascading revocations would occur
```

### Row-Level Authorization

Restricts access to specific tuples/rows rather than entire tables.

```sql
-- Conceptually: WHERE ID = current_user
-- Ahmed → sees Ahmed's rows
-- Sara  → sees Sara's rows
```

---

## 🎯 Master Summary Mental Model

```text
                    CHAPTER 4
                  Intermediate SQL
                         │
      ┌──────────────────┼──────────────────┐
      ↓                  ↓                  ↓
 Application          Database            Security
   + SQL               Logic                 │
      │                  │            ┌──────┼──────┐
      ↓                  ↓            ↓      ↓      ↓
Functions/          Constraints     Roles  Views  Privileges
Procedures              │                   │
Triggers                ↓                   ↓
                    Keep data          Control access
                    consistent
                         │
        ┌────────────────┼────────────────┐
        ↓                ↓                ↓
     Data Types       Indexes          Schemas
        │                │                │
   How data is       Make access       Organize
   represented        faster           objects
```

### The Chapter's Core Idea

> **Chapter 4 takes basic SQL and shows how to use it in real database systems:** connect SQL to applications, automate database behavior, protect data integrity, improve performance with indexes, and control who can access what.
