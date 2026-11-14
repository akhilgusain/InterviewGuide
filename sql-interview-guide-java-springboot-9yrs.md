# SQL Interview Guide — Java / Spring Boot Developer (9 Years Experience)

> **How you'll be graded at 9 YOE:** nobody cares if you can recite the syntax of `LEFT JOIN`.
> They care whether you can (a) write a correct window-function query without hesitating,
> (b) explain *why* a query is slow by reading a plan, (c) reason about transactions,
> isolation and locking under concurrency, and (d) connect all of that to Hibernate/JPA
> behaviour in a real Spring Boot service.
>
> This guide is ordered from fundamentals → senior topics → scenario/debugging → puzzles.
> Every answer has a **Memory hook** (one line to recall under pressure) and, where useful,
> a **Follow-up** (the question they will ask right after).

---

## Table of Contents

1. [Query Fundamentals & Logical Execution Order](#1-query-fundamentals--logical-execution-order)
2. [Joins](#2-joins)
3. [NULL Handling — the classic trap](#3-null-handling--the-classic-trap)
4. [Aggregation, GROUP BY, HAVING](#4-aggregation-group-by-having)
5. [Set Operations](#5-set-operations)
6. [Subqueries, CTEs & Recursive CTEs](#6-subqueries-ctes--recursive-ctes)
7. [Window / Analytic Functions](#7-window--analytic-functions)
8. [Keys, Constraints & Schema Design](#8-keys-constraints--schema-design)
9. [Normalization & Denormalization](#9-normalization--denormalization)
10. [Indexes — How They Actually Work](#10-indexes--how-they-actually-work)
11. [Query Optimization & Reading Execution Plans](#11-query-optimization--reading-execution-plans)
12. [Transactions, ACID & Isolation Levels](#12-transactions-acid--isolation-levels)
13. [Locking, Deadlocks & Concurrency Patterns](#13-locking-deadlocks--concurrency-patterns)
14. [Views, Materialized Views, Procedures, Triggers](#14-views-materialized-views-procedures-triggers)
15. [Partitioning, Sharding, Replication](#15-partitioning-sharding-replication)
16. [SQL from Spring Boot — JDBC, JPA, Hibernate](#16-sql-from-spring-boot--jdbc-jpa-hibernate)
17. [Spring `@Transactional` Deep Dive](#17-spring-transactional-deep-dive)
18. [Connection Pooling (HikariCP)](#18-connection-pooling-hikaricp)
19. [Batch Operations & Bulk Data](#19-batch-operations--bulk-data)
20. [Schema Migrations (Flyway / Liquibase)](#20-schema-migrations-flyway--liquibase)
21. [Security — SQL Injection & Data Protection](#21-security--sql-injection--data-protection)
22. [Scenario-Based Questions (the real interview)](#22-scenario-based-questions-the-real-interview)
23. [Query-Writing Puzzles You Must Be Able to Type Instantly](#23-query-writing-puzzles-you-must-be-able-to-type-instantly)
24. [Rapid-Fire One-Liners](#24-rapid-fire-one-liners)
25. [Cheat Sheets](#25-cheat-sheets)

---

## 1. Query Fundamentals & Logical Execution Order

### Q1.1 — What is the logical order of execution of a SELECT statement?

**Answer.** SQL is *declarative*: you write it in one order, the engine evaluates it in another.
The logical order is:

```
1. FROM        -- pick source tables
2. JOIN / ON   -- combine rows
3. WHERE       -- filter individual rows (no aggregates allowed)
4. GROUP BY    -- collapse rows into groups
5. HAVING      -- filter groups (aggregates allowed)
6. SELECT      -- compute expressions, assign aliases
7. DISTINCT    -- dedupe the result set
8. ORDER BY    -- sort (aliases from SELECT are visible here)
9. LIMIT/OFFSET / FETCH FIRST  -- slice
```

Two consequences interviewers fish for:

- A `SELECT` alias **cannot** be used in `WHERE`, `GROUP BY` or `HAVING` (evaluated earlier),
  but **can** be used in `ORDER BY` (evaluated later). *(MySQL is lenient and allows aliases in
  `GROUP BY`/`HAVING`; standard SQL, Oracle and SQL Server are not. Never rely on it.)*
- Window functions run **after** `GROUP BY`/`HAVING` and **during** `SELECT`, which is why you
  cannot filter on a window function in `WHERE` — you must wrap it in a subquery or CTE
  (or use `QUALIFY` in Snowflake/Teradata/DuckDB).

**Memory hook:** **F**rom → **W**here → **G**roup → **H**aving → **S**elect → **O**rder → **L**imit.
*"**F**riends **W**ho **G**ossip **H**ave **S**erious **O**pinions **L**ately."*

**Follow-up:** *"Then why can I sometimes filter on an alias in MySQL?"* → MySQL extends the
standard; portability and predictability say don't.

---

### Q1.2 — `WHERE` vs `HAVING`?

**Answer.** `WHERE` filters **rows before grouping**; `HAVING` filters **groups after
aggregation**. Because `WHERE` runs first and reduces the number of rows fed into `GROUP BY`,
you should always push every non-aggregate predicate into `WHERE`. Putting a row-level filter
in `HAVING` is functionally equivalent in some engines but forces aggregation over rows you'll
throw away — and it can't use an index the same way.

```sql
-- GOOD: filter rows first, then aggregate, then filter groups
SELECT department_id, AVG(salary) AS avg_sal
FROM   employees
WHERE  status = 'ACTIVE'          -- row filter, index-friendly
GROUP  BY department_id
HAVING COUNT(*) >= 5              -- group filter, needs aggregation
   AND AVG(salary) > 50000;
```

**Why the split exists:** an aggregate such as `AVG(salary)` only has a value *after* the rows
are grouped, so a condition on it cannot be checked while individual rows are being filtered.
That's why `WHERE AVG(salary) > 50000` is a syntax error, while `HAVING AVG(salary) > 50000` is
fine. Conversely, `HAVING status = 'ACTIVE'` only works if `status` is also in the `GROUP BY`
(PostgreSQL rejects it otherwise) — a hint that the condition belonged in `WHERE` all along.

**Worked example:** 1M employees, 10k of them `ACTIVE`. With the filter in `WHERE`, the engine
groups just 10k rows. With it in `HAVING`, it groups all 1M rows and then discards most of the
groups — same answer, ~100× more work.

**Memory hook:** *WHERE guards the door, HAVING inspects the finished groups.*

---

### Q1.3 — `DELETE` vs `TRUNCATE` vs `DROP`

| | `DELETE` | `TRUNCATE` | `DROP` |
|---|---|---|---|
| Type | DML | DDL | DDL |
| `WHERE` clause | Yes | No | No |
| Rollback-able | Yes (fully logged) | Usually not (MySQL/Oracle auto-commit; **PostgreSQL and SQL Server can roll it back**) | No (MySQL); yes in PostgreSQL |
| Speed | Slow (row-by-row, triggers, undo/WAL per row) | Very fast (deallocates pages) | Very fast |
| Fires triggers | Yes | No | No |
| Resets identity/auto-increment | No | Yes | N/A |
| Table & structure survive | Yes | Yes (structure only) | No — table gone |
| FK-referenced table | Allowed | Blocked (unless cascade) | Blocked |
| Reclaims space | Not immediately (needs `VACUUM`/`OPTIMIZE`) | Yes | Yes |

**Memory hook:** *DELETE = erase rows with a pencil. TRUNCATE = throw away the paper, keep the
notebook. DROP = burn the notebook.*

**Follow-up:** *"You need to delete 50M of 60M rows — which do you use?"* → Neither naively.
Best pattern: `CREATE TABLE new AS SELECT ... WHERE keep_condition`, then swap names — or
delete in batches of 10–50k with a commit per batch to avoid a giant transaction, lock
escalation and replication lag.

---

### Q1.4 — `CHAR` vs `VARCHAR` vs `TEXT`; `DECIMAL` vs `FLOAT`

**Answer.**
- `CHAR(n)` is fixed-length, right-padded with spaces — fast for genuinely fixed data
  (country codes, `Y`/`N`, hashes). `VARCHAR(n)` is variable-length + a length prefix.
  `TEXT`/`CLOB` is for large blobs of text; in MySQL it may be stored off-page and can't be
  fully indexed (needs a prefix index).
- `DECIMAL/NUMERIC(p,s)` is **exact** base-10 — mandatory for money. `FLOAT/DOUBLE` is binary
  floating point and **cannot** represent 0.1 exactly, so `0.1 + 0.2 != 0.3`. Never store
  currency in a float. In Java: `BigDecimal` ↔ `DECIMAL`, and always use
  `new BigDecimal("10.50")` (String constructor), never `new BigDecimal(10.50)`.

**Plain-language detail:**
- `CHAR(3)` storing `'IN'` actually holds `'IN '` (padded to 3). `VARCHAR(255)` storing `'IN'` holds
  just 2 characters plus a small length prefix — so `VARCHAR` saves space whenever lengths vary.
  The `(n)` is a *maximum*, not a reservation; it exists to reject bad data, not to allocate space.
- `DECIMAL(10,2)` means *precision 10, scale 2*: up to 10 digits in total, 2 of them after the
  decimal point (so max `99999999.99`). Values with more decimals are rounded/rejected.
- Why floats go wrong: computers store `FLOAT` in binary, and 0.1 has no finite binary
  representation (like 1/3 in decimal). In Java, `0.1 + 0.2` prints `0.30000000000000004`; in
  PostgreSQL `SELECT 0.1::float8 + 0.2::float8 = 0.3::float8` returns `false`. Summing a million
  float prices drifts by real money; `DECIMAL` does not.
- `TEXT` vs `VARCHAR`: in PostgreSQL they perform identically (`TEXT` = unlimited `VARCHAR`);
  choose `VARCHAR(n)` when you want the database to enforce a maximum length.

**Memory hook:** *Money → DECIMAL/BigDecimal. Science → DOUBLE. Never mix them up in a
banking interview.*

---

## 2. Joins

### Q2.1 — Explain every join type.

**Answer.**

```sql
-- INNER JOIN: only matching rows on both sides
SELECT e.name, d.name FROM employees e INNER JOIN departments d ON e.dept_id = d.id;

-- LEFT (OUTER) JOIN: all left rows; NULLs where no right match
SELECT e.name, d.name FROM employees e LEFT JOIN departments d ON e.dept_id = d.id;

-- RIGHT JOIN: mirror of LEFT (rarely used — just flip the tables and use LEFT)
-- FULL OUTER JOIN: all rows from both sides, NULL-padded
--   (MySQL has no FULL OUTER JOIN → emulate with LEFT JOIN UNION RIGHT JOIN)

-- CROSS JOIN: cartesian product, m × n rows. Legit use: generating date/number series.
SELECT d.date, p.product_id FROM calendar d CROSS JOIN products p;

-- SELF JOIN: table joined to itself, for hierarchies
SELECT e.name AS emp, m.name AS mgr
FROM   employees e LEFT JOIN employees m ON e.manager_id = m.id;

-- NATURAL JOIN: auto-joins on all same-named columns. NEVER USE IT.
--   Adding a column later (e.g. created_at to both tables) silently changes the join.
```

**Memory hook:** INNER = intersection, LEFT = left circle whole, FULL = both circles,
CROSS = every-with-every.

**Tiny worked example:** `employees = {Asha(dept 1), Ben(dept 2), Cara(dept NULL)}`,
`departments = {1 IT, 2 HR, 3 Legal}`.
- `INNER JOIN` → Asha-IT, Ben-HR (Cara has no department, Legal has no employee: both dropped).
- `LEFT JOIN` → Asha-IT, Ben-HR, **Cara-NULL** (every employee kept).
- `FULL OUTER JOIN` → the above plus **NULL-Legal** (every row from both sides kept).
- `CROSS JOIN` → 3 × 3 = 9 rows, no `ON` condition.

Why `NATURAL JOIN` is banned: it silently joins on *every* same-named column, so adding a column
like `created_at` to both tables changes the join condition without any code change.
Join choice rule of thumb: start with `INNER`; switch to `LEFT` the moment the question says
"including those without…" (e.g. "all customers, even with no orders").

---

### Q2.2 — Difference between a filter in `ON` vs in `WHERE` for a LEFT JOIN?

**Answer.** This is *the* most common senior join question and it trips up a lot of people.

```sql
-- (A) predicate in ON — applied BEFORE the join is padded.
--     Keeps ALL employees; only 'IT' departments get matched, others show NULL.
SELECT e.name, d.name
FROM   employees e
LEFT   JOIN departments d ON e.dept_id = d.id AND d.name = 'IT';

-- (B) predicate in WHERE — applied AFTER padding.
--     The NULL-padded rows fail `d.name = 'IT'` and are removed,
--     silently turning your LEFT JOIN into an INNER JOIN.
SELECT e.name, d.name
FROM   employees e
LEFT   JOIN departments d ON e.dept_id = d.id
WHERE  d.name = 'IT';
```

Rule: for the **outer/optional** table, filters belong in `ON`. For the **preserved** table,
filters belong in `WHERE`. The one legitimate `WHERE` predicate on the outer table is
`WHERE d.id IS NULL` — the anti-join idiom for "employees with no department".

**Memory hook:** *ON filters before padding, WHERE filters after. A WHERE on the outer table
kills the OUTER.*

---

### Q2.3 — How do you find rows in A that are not in B? (Anti-join, 3 ways)

```sql
-- 1. NOT EXISTS  ← safest and usually fastest; NULL-safe
SELECT c.* FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);

-- 2. LEFT JOIN / IS NULL  ← very readable, good plans everywhere
SELECT c.* FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL;

-- 3. NOT IN  ← DANGEROUS: if the subquery returns even ONE NULL, the whole
--    result is empty, because `x NOT IN (1, NULL)` evaluates to UNKNOWN, not TRUE.
SELECT c.* FROM customers c
WHERE c.id NOT IN (SELECT customer_id FROM orders);   -- broken if customer_id nullable
```

**Why `NOT IN` breaks (step by step):** `5 NOT IN (1, 2, NULL)` is rewritten as
`5 <> 1 AND 5 <> 2 AND 5 <> NULL`. The last term is `UNKNOWN` (anything compared to NULL is
unknown, see Q3.1), and `TRUE AND TRUE AND UNKNOWN` is `UNKNOWN` — not `TRUE` — so the row is
filtered out. One NULL in the list therefore makes *every* row fail, and you get an empty result
with no error, which is why it's such a nasty production bug.

**Which one to pick:** `NOT EXISTS` and `LEFT JOIN ... IS NULL` give the same rows. The
optimizer usually turns both into an *anti-join* (a join that keeps only non-matching rows), so
use whichever reads best — just make sure the join column is indexed (`orders.customer_id`).
This pattern answers business questions like "customers who never ordered" or "products never sold".

**Memory hook:** *`NOT IN` + one `NULL` = zero rows. Use `NOT EXISTS`.*

---

### Q2.4 — What physical join algorithms does the optimizer choose from?

**Answer.**

| Algorithm | How it works | Best when | Cost |
|---|---|---|---|
| **Nested Loop** | For each outer row, probe inner (ideally via index) | Small outer set + indexed inner | O(n × log m) with index; O(n×m) without |
| **Hash Join** | Build hash table on smaller input, probe with larger | Large, unsorted, equality join, no useful index | O(n+m), needs memory; spills to disk if `work_mem` too small |
| **Merge Join** | Sort both inputs, walk them in lockstep | Both already sorted (index order) or huge sets | O(n log n + m log m) |

MySQL historically only had nested loop + Block Nested Loop; **hash join arrived in MySQL 8.0.18**.
PostgreSQL, Oracle and SQL Server have all three.

Practical takeaway: if you see a **Nested Loop over a large table with no index** in a plan,
that's your bug. If you see a hash join spilling to disk, raise memory or reduce the row set.

**Memory hook:** *Loop = small+indexed. Hash = big+equality. Merge = already sorted.*

---

## 3. NULL Handling — the classic trap

### Q3.1 — Explain NULL semantics.

**Answer.** `NULL` means *unknown*, not zero and not empty string. SQL uses three-valued logic
(TRUE / FALSE / UNKNOWN), and only rows evaluating to **TRUE** are returned.

```sql
NULL = NULL            -- UNKNOWN (not true!) → use IS NULL / IS NOT NULL
NULL <> 5              -- UNKNOWN
NULL + 5               -- NULL (arithmetic propagates)
'abc' || NULL          -- NULL in Oracle/Postgres; Oracle CONCAT treats it as ''
COUNT(*)               -- counts rows, includes NULLs
COUNT(col)             -- counts NON-NULL values of col
AVG(col)               -- ignores NULLs → denominator is non-null count, NOT row count
SUM(col)               -- ignores NULLs; returns NULL (not 0) if all rows are NULL
```

Key gotchas:
- **Aggregates skip NULLs** — `AVG(bonus)` over 10 rows where 5 are NULL divides by 5.
  If you want zeros counted: `AVG(COALESCE(bonus, 0))`.
- **UNIQUE constraints allow multiple NULLs** in most engines (Oracle, PostgreSQL, MySQL),
  because two NULLs aren't "equal". SQL Server allows only one.
- **`GROUP BY` and `DISTINCT` treat all NULLs as one group** — inconsistent with `=`, but that's
  the standard.
- **`ORDER BY`**: PostgreSQL/Oracle sort NULLs last ascending by default; MySQL/SQL Server
  first. Use `ORDER BY col ASC NULLS LAST` where supported.

Functions: `COALESCE(a,b,c)` returns first non-NULL (standard, n-ary);
`NULLIF(a,b)` returns NULL if a=b (great for divide-by-zero: `x / NULLIF(y,0)`);
`IFNULL` (MySQL), `NVL`/`NVL2` (Oracle), `ISNULL` (SQL Server) are vendor-specific two-arg forms.

**Memory hook:** *NULL is a ghost — you can't compare it, it infects arithmetic, and aggregates
pretend it isn't there.*

---

### Q3.2 — What does this return and why?

```sql
SELECT COUNT(*), COUNT(commission), AVG(commission), SUM(commission)
FROM employees;   -- 10 rows, 6 of them have commission = NULL
```

**Answer.** `COUNT(*) = 10`, `COUNT(commission) = 4`, `AVG` = sum of the 4 divided by **4**,
`SUM` = sum of the 4. If all 10 were NULL: `COUNT(commission) = 0`, `AVG = NULL`, `SUM = NULL`
(not 0 — a common cause of NPEs when mapping to a Java primitive `long`; map to `Long` or wrap
in `COALESCE(SUM(x), 0)`).

**Memory hook:** *`COUNT(*)` counts rows; everything else counts values.*

---

## 4. Aggregation, GROUP BY, HAVING

### Q4.1 — What must appear in `GROUP BY`?

**Answer.** Every non-aggregated column in the `SELECT` list must be in the `GROUP BY`
(or be functionally dependent on it, e.g. grouped by primary key). MySQL historically allowed
you to select ungrouped columns and returned an arbitrary value — disabled by default since
5.7 via the `ONLY_FULL_GROUP_BY` sql_mode. If you want "one row per group with the full row's
other columns", that's not `GROUP BY` — that's a **window function** (`ROW_NUMBER`) or a
lateral/correlated top-1 lookup.

**Why the rule exists:** once rows are collapsed into groups, a column like `name` that is *not*
a group key has several different values per group (every employee in department 10 has a
different name). The database cannot know which one you want, so standard SQL refuses the query.

```sql
-- Rejected (PostgreSQL / MySQL 5.7+ default): 'name' is neither grouped nor aggregated
SELECT department_id, name, MAX(salary) FROM employees GROUP BY department_id;

-- Fixes depend on what you actually meant:
SELECT department_id, MAX(salary) FROM employees GROUP BY department_id;     -- just the aggregate
SELECT * FROM (SELECT e.*, ROW_NUMBER() OVER (PARTITION BY department_id
                                              ORDER BY salary DESC) rn
               FROM employees e) t WHERE rn = 1;                              -- the top earner's full row
```

Exception: if you `GROUP BY` the primary key, every other column of that table is determined by
it ("functional dependence"), so PostgreSQL lets you select them without listing them.

---

### Q4.2 — `GROUPING SETS`, `ROLLUP`, `CUBE`?

**Answer.** Ways to produce multiple aggregation levels in **one pass** instead of `UNION ALL`-ing
several queries — big win for reporting.

```sql
-- Subtotals per region, per region+product, and a grand total, in one scan
SELECT region, product, SUM(amount)
FROM   sales
GROUP  BY ROLLUP (region, product);
-- ROLLUP(a,b) => (a,b), (a), ()          -- hierarchical, n+1 levels
-- CUBE(a,b)   => (a,b), (a), (b), ()     -- all combinations, 2^n levels
-- GROUPING SETS((a,b),(b))               -- exactly the levels you list

-- Distinguish a real NULL from a subtotal NULL:
SELECT region, GROUPING(region) AS is_subtotal, SUM(amount) FROM sales GROUP BY ROLLUP(region);
```

**What the output looks like** (sample `sales` data) — each extra level is a row where the
rolled-up column is `NULL`:

| region | product | SUM(amount) | what this row is |
|---|---|---|---|
| EU | Laptop | 100 | detail (region + product) |
| EU | Phone | 50 | detail |
| EU | *NULL* | 150 | subtotal for EU |
| US | Laptop | 200 | detail |
| US | *NULL* | 200 | subtotal for US |
| *NULL* | *NULL* | 350 | grand total |

Why it matters: without it you'd write three `GROUP BY` queries glued with `UNION ALL`, scanning
the table three times. The `NULL`s are placeholders for "all values", which is why `GROUPING(col)`
(returns 1 for a rolled-up row, 0 otherwise) is needed to tell them from real NULLs in the data.
MySQL syntax is `GROUP BY region, product WITH ROLLUP`; MySQL has no `CUBE`/`GROUPING SETS`.

**Memory hook:** *ROLLUP = drill-down hierarchy. CUBE = every combination. GROUPING SETS = à la carte.*

---

### Q4.3 — Conditional aggregation (pivoting without PIVOT)

**Answer.** The single most useful reporting trick. `CASE` inside an aggregate.

```sql
SELECT
  department_id,
  COUNT(*)                                           AS total,
  SUM(CASE WHEN status = 'ACTIVE'   THEN 1 ELSE 0 END) AS active,
  COUNT(CASE WHEN status = 'ACTIVE'   THEN 1 END)      AS active_alt,  -- same thing
  AVG(CASE WHEN gender = 'F' THEN salary END)          AS avg_female_sal,
  SUM(CASE WHEN order_date >= DATE '2026-01-01' THEN amount ELSE 0 END) AS ytd
FROM   employees
GROUP  BY department_id;
```

Also: `COUNT(DISTINCT x)`, and filtered aggregates in PostgreSQL:
`COUNT(*) FILTER (WHERE status='ACTIVE')` — cleaner and standard SQL.

**Memory hook:** *One scan, many columns: CASE inside SUM/COUNT.*

---

## 5. Set Operations

### Q5.1 — `UNION` vs `UNION ALL` vs `INTERSECT` vs `EXCEPT`/`MINUS`

**Answer.**

| Op | Meaning | Dedupes? | Notes |
|---|---|---|---|
| `UNION` | A ∪ B | **Yes** — implies a sort/hash distinct | Slower; only use when you truly need dedupe |
| `UNION ALL` | A + B, keep duplicates | No | **Default choice** — no sort, streams |
| `INTERSECT` | A ∩ B | Yes | MySQL added it in 8.0.31; before that use `INNER JOIN`/`EXISTS` |
| `EXCEPT` (Oracle: `MINUS`) | A − B | Yes | Before MySQL 8.0.31 use `NOT EXISTS` |

Rules for all set ops: same number of columns, compatible types, column names come from the
first query, and a single `ORDER BY` applies to the whole result (goes at the very end).
Set operators treat `NULL`s as equal for dedupe purposes.

**Tiny example** — `A = {1, 2, 2}`, `B = {2, 3}`:
`A UNION B` → `1, 2, 3` (dupes removed, extra sort/hash step); `A UNION ALL B` → `1, 2, 2, 2, 3`
(just appended); `A INTERSECT B` → `2`; `A EXCEPT B` → `1`.

Typical use: combining the same shape of data from different sources, e.g. current and archived
orders: `SELECT id, total FROM orders UNION ALL SELECT id, total FROM orders_archive`. If the two
sets can never overlap (different date ranges), `UNION` only adds cost for nothing.

**Memory hook:** *If you can't prove you need dedupe, write `UNION ALL`.*

**Follow-up:** *"How do you know if a `UNION` is hurting you?"* → the plan shows an extra
`HashAggregate`/`Unique`/`Sort` node above the append; on large sets it's the dominant cost.

---

## 6. Subqueries, CTEs & Recursive CTEs

### Q6.1 — Scalar vs row vs table subquery; correlated vs non-correlated?

**Answer.**
- **Non-correlated**: independent of the outer query, evaluated **once**.
  `WHERE salary > (SELECT AVG(salary) FROM employees)`
- **Correlated**: references an outer column, conceptually evaluated **per outer row**.
  `WHERE salary > (SELECT AVG(salary) FROM employees e2 WHERE e2.dept_id = e1.dept_id)`
  Modern optimizers often *decorrelate* these into joins, but not always — a correlated
  subquery over an unindexed column is a classic O(n²) production incident.
- **Scalar** returns 1 row/1 col (usable in `SELECT`); **row** returns one tuple;
  **table** (a.k.a. derived table / inline view) returns a result set used in `FROM`.

```sql
-- Derived table (inline view) — must be aliased
SELECT d.name, s.avg_sal
FROM   departments d
JOIN  (SELECT dept_id, AVG(salary) avg_sal FROM employees GROUP BY dept_id) s
       ON s.dept_id = d.id;
```

**Memory hook:** *Correlated = "for each row, go look again." Great for `EXISTS`, dangerous for
big scans.*

---

### Q6.2 — `IN` vs `EXISTS` vs `JOIN` — which is faster?

**Answer.** In modern optimizers (PostgreSQL, Oracle, SQL Server, MySQL 8) they are frequently
transformed into the same semi-join plan, so "it depends on the plan, not the syntax." The
meaningful differences:

- **`EXISTS`** short-circuits on the first match; ideal when the subquery is large and indexed
  on the correlation column.
- **`IN`** materializes/hashes the list; ideal when the subquery result is small.
- **`JOIN`** can **duplicate** outer rows if the right side has multiple matches — `EXISTS`
  and `IN` never do. This is a *correctness* difference, not a performance one, and it's the
  answer they're looking for.
- **`NOT IN`** is NULL-unsafe (see Q2.3). `NOT EXISTS` is safe.

**Memory hook:** *EXISTS = "does at least one exist?" (no duplication). JOIN = "give me the
matches" (can duplicate).*

---

### Q6.3 — What is a CTE and why use it? Is it a performance optimization?

**Answer.** A Common Table Expression (`WITH`) is a named, scoped subquery. Benefits:
readability, reuse of a name within one statement, and it's the only way to write a **recursive**
query. It is **not inherently faster**.

Critical vendor difference — expect this question:
- **PostgreSQL ≤ 11**: CTEs were always **materialized** — an optimization *fence*. Predicates
  couldn't be pushed in, which sometimes made CTEs much slower than an inline subquery.
- **PostgreSQL 12+**: CTEs referenced once and side-effect-free are **inlined** by default;
  you can force behaviour with `WITH x AS MATERIALIZED (...)` / `AS NOT MATERIALIZED (...)`.
- **MySQL 8+ / SQL Server / Oracle**: generally inlined/merged into the outer query.

Also: a CTE referenced twice may be **executed twice** (when inlined). If the CTE is expensive
and reused, materialize it explicitly or use a temp table.

**Memory hook:** *CTE = readability, not speed. Postgres 12 inlines; older Postgres fenced.*

---

### Q6.4 — Write a recursive CTE for an employee hierarchy.

```sql
WITH RECURSIVE org_chart AS (
    -- Anchor: the root(s)
    SELECT id, name, manager_id, 1 AS level,
           CAST(name AS VARCHAR(1000)) AS path
    FROM   employees
    WHERE  manager_id IS NULL

    UNION ALL

    -- Recursive member: joins back to the CTE itself
    SELECT e.id, e.name, e.manager_id, oc.level + 1,
           CAST(oc.path || ' > ' || e.name AS VARCHAR(1000))
    FROM   employees e
    JOIN   org_chart oc ON e.manager_id = oc.id
    WHERE  oc.level < 20                   -- ALWAYS bound the recursion
)
SELECT level, path, name FROM org_chart ORDER BY path;
```

Notes: `RECURSIVE` keyword is required in PostgreSQL/MySQL, not in SQL Server/Oracle.
Oracle also has the legacy `CONNECT BY PRIOR ... START WITH`.
Guard against cycles with a depth limit or by tracking visited ids in an array
(`WHERE NOT e.id = ANY(oc.visited)`); MySQL has `cte_max_recursion_depth` (default 1000).

**Other classic recursive uses:** generating a date series to fill report gaps, exploding a
bill-of-materials, transitive closure of a graph, splitting a delimited string.

**Memory hook:** *Anchor `UNION ALL` recursive-member, joined back to itself, with a depth guard.*

---

### Q6.5 — What is a LATERAL join / `CROSS APPLY`?

**Answer.** A `LATERAL` subquery (PostgreSQL/Oracle 12c; `CROSS APPLY`/`OUTER APPLY` in SQL Server)
can reference columns from tables **to its left** in the same `FROM` clause. It's the clean way
to do "top-N per group."

```sql
-- 3 most recent orders per customer
SELECT c.id, c.name, o.order_date, o.amount
FROM   customers c
CROSS  JOIN LATERAL (
         SELECT o.order_date, o.amount
         FROM   orders o
         WHERE  o.customer_id = c.id      -- <-- references c, only legal with LATERAL
         ORDER  BY o.order_date DESC
         LIMIT  3
       ) o;
```

Versus `ROW_NUMBER()`: the window-function version scans **all** orders and then filters;
`LATERAL` + index on `(customer_id, order_date DESC)` does 3 index seeks per customer.
For few customers × many orders, `LATERAL` wins dramatically. That comparison is a great
senior answer. MySQL has no `LATERAL` — use window functions there.

**Memory hook:** *LATERAL = a correlated subquery you can put in FROM. Best tool for top-N-per-group.*

---

## 7. Window / Analytic Functions

> If you only memorize one section for a 9-YOE interview, make it this one. Window functions
> come up in nearly every senior SQL screen.

### Q7.1 — What is a window function and how does it differ from `GROUP BY`?

**Answer.** A window function computes a value **across a set of rows related to the current
row**, but **without collapsing the rows**. `GROUP BY` returns one row per group; a window
function returns every input row plus the computed value.

```sql
SELECT
  name, department_id, salary,
  AVG(salary) OVER (PARTITION BY department_id)                    AS dept_avg,
  salary - AVG(salary) OVER (PARTITION BY department_id)           AS diff_from_avg,
  ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rn
FROM employees;
```

Anatomy: `function() OVER (PARTITION BY ... ORDER BY ... ROWS/RANGE frame)`
- `PARTITION BY` = reset the calculation per group (like GROUP BY, but non-collapsing)
- `ORDER BY` = ordering *within* the partition (required for ranking & running totals)
- **frame** = which rows within the partition are visible

**Memory hook:** *GROUP BY squashes rows; OVER() keeps them.*

---

### Q7.2 — `ROW_NUMBER` vs `RANK` vs `DENSE_RANK` vs `NTILE`

**Answer.** Given salaries 100, 90, 90, 80:

| salary | `ROW_NUMBER` | `RANK` | `DENSE_RANK` |
|---|---|---|---|
| 100 | 1 | 1 | 1 |
| 90 | 2 | 2 | 2 |
| 90 | 3 | **2** | **2** |
| 80 | 4 | **4** (gap) | **3** (no gap) |

- `ROW_NUMBER()` — always unique, ties broken arbitrarily (add a tiebreaker column for
  determinism!). Use for dedupe and pagination.
- `RANK()` — ties share a rank, then **skips** numbers (Olympic ranking).
- `DENSE_RANK()` — ties share a rank, **no gaps**. Use for "top 3 distinct salaries".
- `NTILE(n)` — splits the partition into n roughly equal buckets (quartiles, deciles).
  Example: `NTILE(4) OVER (ORDER BY salary)` labels each employee 1–4 by salary quartile; with 10
  rows and `NTILE(4)` the buckets get 3, 3, 2, 2 rows (earlier buckets take the remainder).
- `PERCENT_RANK()`, `CUME_DIST()` — relative position as a fraction.

**Memory hook:** *ROW_NUMBER = 1,2,3,4. RANK = 1,2,2,4 (gaps). DENSE_RANK = 1,2,2,3 (dense).*

---

### Q7.3 — `LAG` / `LEAD` and their use cases

```sql
SELECT
  order_date, revenue,
  LAG(revenue, 1)  OVER (ORDER BY order_date)                   AS prev_day,
  LEAD(revenue, 1) OVER (ORDER BY order_date)                   AS next_day,
  revenue - LAG(revenue) OVER (ORDER BY order_date)             AS delta,
  ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY order_date))
        / NULLIF(LAG(revenue) OVER (ORDER BY order_date), 0), 2) AS pct_change,
  LAG(revenue, 1, 0) OVER (ORDER BY order_date)                 AS prev_with_default
FROM daily_revenue;
```

Uses: day-over-day / month-over-month growth, detecting state transitions in an audit table,
computing time between consecutive events (session gaps), finding gaps in sequences.
`FIRST_VALUE`, `LAST_VALUE` and `NTH_VALUE` are related — note `LAST_VALUE` needs an explicit
frame (`ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`) or it returns the current row.

**Memory hook:** *LAG looks back, LEAD looks forward. Always `NULLIF` the denominator.*

---

### Q7.4 — Explain window frames: `ROWS` vs `RANGE`.

**Answer.** The frame defines the visible rows *within* the partition.

```sql
-- Default when ORDER BY is present:  RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
-- Default when no ORDER BY:          entire partition

SUM(amt) OVER (ORDER BY dt ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) -- running total
AVG(amt) OVER (ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)         -- 7-row moving avg
SUM(amt) OVER (ORDER BY dt RANGE BETWEEN INTERVAL '7' DAY PRECEDING
                             AND CURRENT ROW)                                -- 7-DAY window
SUM(amt) OVER (PARTITION BY cust ORDER BY dt ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING)
```

**`ROWS` counts physical rows. `RANGE` counts logical values** — with `RANGE`, all peer rows
with the same `ORDER BY` value are included together. So on a day with 3 orders, a `ROWS 0
PRECEDING` running total differs from `RANGE` (which lumps the 3 peers in). **This is the #1
subtle window-function bug:** a "running total" using the default `RANGE` frame jumps in steps
across duplicate dates. If you want row-by-row, say `ROWS`.

**Memory hook:** *ROWS = physical count. RANGE = value peers. Duplicate keys? Say ROWS.*

---

### Q7.5 — How do you filter on a window function?

**Answer.** You can't do it in `WHERE` (windows are computed at `SELECT`, after `WHERE`).
Wrap it:

```sql
WITH ranked AS (
  SELECT e.*, DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dr
  FROM   employees e
)
SELECT * FROM ranked WHERE dr <= 3;   -- top 3 salaries per department
```

**Memory hook:** *Window → CTE → filter. (Or `QUALIFY` if you're lucky enough to be on Snowflake.)*

---

### Q7.6 — Can you use `DISTINCT` or a `WHERE` inside a window function?

**Answer.** No `DISTINCT` inside `OVER` in most engines (PostgreSQL doesn't allow
`COUNT(DISTINCT x) OVER (...)`) — workaround is `DENSE_RANK` tricks or a pre-aggregated CTE.
For conditional windows, use `CASE`:
`SUM(CASE WHEN status='PAID' THEN amt ELSE 0 END) OVER (PARTITION BY cust)`,
or in PostgreSQL, `COUNT(*) FILTER (WHERE ...) OVER (...)`.

**Why `DISTINCT` is disallowed:** a window function is evaluated once per row against a
*moving* set of rows (the frame), and de-duplicating a different set for each row is hard for
the engine to do efficiently, so many databases (including PostgreSQL) don't support it. Typical workaround for
"distinct customers seen so far": compute a flag for the *first* occurrence and sum it —

```sql
SELECT order_date,
       SUM(is_first) OVER (ORDER BY order_date) AS distinct_customers_so_far
FROM (SELECT order_date,
             CASE WHEN ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) = 1
                  THEN 1 ELSE 0 END AS is_first
      FROM orders) t;
```

**Why `WHERE` isn't allowed inside:** the filter in `WHERE` would change which rows *exist*; the
`OVER` clause decides which rows are *visible to the calculation* for each row. `CASE`/`FILTER`
express "ignore this row for this one calculation" while keeping the row in the output.

Also useful: define the window once with a `WINDOW` clause.

```sql
SELECT name, SUM(amt) OVER w, AVG(amt) OVER w
FROM   sales
WINDOW w AS (PARTITION BY region ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW);
```

---
## 8. Keys, Constraints & Schema Design

### Q8.1 — Explain all key types.

**Answer.**

- **Super key** — any set of columns that uniquely identifies a row.
- **Candidate key** — a minimal super key (no redundant column).
- **Primary key** — the chosen candidate key. Unique + `NOT NULL`, one per table. In InnoDB
  and SQL Server it typically defines the **clustered** physical order.
- **Alternate key** — the candidate keys not chosen as PK (implemented as `UNIQUE`).
- **Composite key** — a key made of 2+ columns.
- **Foreign key** — references a PK/unique key in another (or the same) table; enforces
  referential integrity.
- **Surrogate key** — a system-generated meaningless id (auto-increment / sequence / UUID).
- **Natural key** — a business-meaningful key (email, ISBN, PAN). Downside: business values
  change, and they're wide (bad for InnoDB secondary indexes).

**Worked example** — table `employees(emp_id, email, phone, name, dept_id)`:
- `{emp_id}` is a key; so is `{email}` (if emails are unique). Both are **candidate keys**.
- `{emp_id, name}` is still a **super key** (unique) but *not* a candidate key, because `name`
  is redundant — drop it and uniqueness remains.
- Choose `emp_id` as the **primary key**; `{email}` becomes an **alternate key**
  (`UNIQUE (email)`); `dept_id` is a **foreign key** to `departments(id)`.
- In an `order_items(order_id, product_id, qty)` table the PK is the **composite key**
  `(order_id, product_id)` — neither column alone is unique.

**Memory hook:** *Super ⊇ Candidate ⊇ Primary. Surrogate = made-up id; Natural = real-world id.*

**Follow-up (very common):** *"Surrogate or natural key — which do you pick?"*
→ Surrogate `BIGINT` PK for stability and narrow indexes, **plus** a `UNIQUE` constraint on the
natural key so the database still enforces the business rule. Best of both.

---

### Q8.2 — `PRIMARY KEY` vs `UNIQUE`

| | PRIMARY KEY | UNIQUE |
|---|---|---|
| NULLs | Not allowed | Allowed (multiple, in most engines) |
| Count per table | One | Many |
| Index created | Yes (clustered by default in InnoDB/SQL Server) | Yes (non-clustered) |
| FK target | Standard target | Also valid |

**In plain words:** the primary key is the row's *identity* — what everything else points to —
so it can never be missing (`NOT NULL`) and there's only one. A `UNIQUE` constraint is a *business
rule* ("no two users share an email") and you can have several. Example:

```sql
CREATE TABLE users (
  id    BIGINT PRIMARY KEY,         -- identity, referenced by FKs
  email VARCHAR(255) NOT NULL UNIQUE,  -- business rule; add NOT NULL if it's mandatory
  phone VARCHAR(20)  UNIQUE            -- optional: many rows may have NULL phone
);
```

Pitfalls: (1) `UNIQUE` on a nullable column lets many NULL rows through (SQL Server allows only
one) — add `NOT NULL` if "missing" shouldn't be allowed. (2) Both are enforced by an index, so
the unique key also speeds lookups by that column. (3) Prefer a *composite* `UNIQUE` when the
rule is about a combination (`UNIQUE (tenant_id, email)`).

---

### Q8.3 — Foreign key `ON DELETE` / `ON UPDATE` options?

**Answer.** `RESTRICT` (reject, checked immediately), `NO ACTION` (reject, checked at end of
statement/transaction — deferrable in PostgreSQL), `CASCADE` (propagate the delete/update),
`SET NULL`, `SET DEFAULT`.

**What each does** — given `orders.customer_id REFERENCES customers(id)` and you delete a customer:
- `RESTRICT` / `NO ACTION` → the delete fails with an FK-violation error while orders exist.
  (Difference: `NO ACTION` can be postponed to commit via `DEFERRABLE INITIALLY DEFERRED`,
  which lets you insert parent and child in any order inside one transaction.)
- `CASCADE` → the customer's orders are deleted too.
- `SET NULL` → orders stay, but `customer_id` becomes `NULL` (column must be nullable).
- `SET DEFAULT` → `customer_id` becomes the column's default value.

```sql
ALTER TABLE orders
  ADD CONSTRAINT fk_orders_customer FOREIGN KEY (customer_id)
  REFERENCES customers (id) ON DELETE RESTRICT ON UPDATE CASCADE;
```

Why FKs exist at all: they stop "orphans" (an order pointing to a customer that doesn't exist),
which application code alone can't guarantee under concurrency.

Senior nuance: **`ON DELETE CASCADE` is dangerous in production** — a single delete can silently
remove millions of child rows, blow up the transaction log, hold locks for minutes, and it
bypasses your application's audit/soft-delete logic. Prefer explicit application-level deletion
or `RESTRICT`, and remember Hibernate's `CascadeType.REMOVE` is a *different, application-level*
cascade (loads children into memory and deletes them one by one) — don't confuse the two in the
interview.

Also: **an unindexed FK column is a classic performance bug.** MySQL auto-creates an index on
FK columns; **PostgreSQL and Oracle do not.** Deleting a parent row then requires a full scan of
the child table to check references, and in Oracle it can take a table-level lock.

**Memory hook:** *MySQL indexes FKs for you. Postgres/Oracle don't — do it yourself.*

---

### Q8.4 — What constraint types exist?

`NOT NULL`, `UNIQUE`, `PRIMARY KEY`, `FOREIGN KEY`, `CHECK` (e.g. `CHECK (salary > 0)`,
`CHECK (end_date >= start_date)`), `DEFAULT`, and in PostgreSQL `EXCLUDE` (e.g. prevent
overlapping booking ranges via `EXCLUDE USING gist (room WITH =, during WITH &&)`).

```sql
CREATE TABLE employees (
  id         BIGINT PRIMARY KEY,
  email      VARCHAR(255) NOT NULL UNIQUE,            -- must be present, no duplicates
  salary     NUMERIC(12,2) NOT NULL CHECK (salary > 0), -- rejects 0 / negative values
  status     VARCHAR(10)  NOT NULL DEFAULT 'ACTIVE',  -- used when INSERT omits the column
  start_date DATE NOT NULL,
  end_date   DATE,
  dept_id    BIGINT REFERENCES departments(id),       -- FK
  CHECK (end_date IS NULL OR end_date >= start_date)  -- table-level, spans two columns
);
```

What each guarantees: `NOT NULL` = value must be present; `UNIQUE` = no duplicates; `CHECK` = any
boolean rule on the row (a row is rejected only if the check is *false* — an `UNKNOWN` result from
a NULL passes, hence the explicit `NOT NULL`); `DEFAULT` = fills in a value when omitted;
`EXCLUDE` = "no two rows may conflict" for ranges (e.g. two bookings for the same room at
overlapping times), which a plain `UNIQUE` can't express.

**Interview stance to take:** put invariants in the database, not only in Java. Bean Validation
(`@NotNull`, `@Size`) protects one application; a `CHECK` constraint protects the data from
every application, every script and every manual `UPDATE` someone runs at 2am. Defence in depth.

---

## 9. Normalization & Denormalization

### Q9.1 — Explain the normal forms with an example.

**Answer.**

- **1NF** — atomic values, no repeating groups/arrays in a column, each row unique.
  *Violation:* `phone_numbers = '99999,88888'`.
- **2NF** — 1NF **+ no partial dependency**: every non-key attribute depends on the *whole*
  composite key.
  *Violation:* table `(order_id, product_id, qty, product_name)` — `product_name` depends only
  on `product_id`, half the key. Fix: move it to `products`.
- **3NF** — 2NF **+ no transitive dependency**: non-key attributes must not depend on other
  non-key attributes.
  *Violation:* `employees(id, dept_id, dept_name)` — `dept_name` depends on `dept_id`, which
  depends on `id`. Fix: `departments` table.
- **BCNF** — stricter 3NF: **every** determinant must be a candidate key. Handles the edge case
  of overlapping candidate keys.
- **4NF** — no multi-valued dependencies (an employee's independent skills and independent
  languages in one table produce a spurious cross product).
- **5NF** — no lossless join dependencies remaining.

**BCNF example:** `course_teacher(student, course, teacher)` where each teacher teaches exactly
one course (`teacher → course`) and a student has one teacher per course. `teacher` determines
`course` but isn't a candidate key, so it violates BCNF. Fix: split into
`teacher_course(teacher, course)` and `student_teacher(student, teacher)`.

**4NF example:** `employee_skill_language(emp, skill, language)` — skills and languages are
independent facts about an employee, so storing both forces every skill × every language row
(3 skills × 2 languages = 6 rows, and adding one skill means adding 2 rows). Split into
`employee_skill` and `employee_language`.

**Practical stance:** in real systems, aim for **3NF** (usually BCNF) for transactional tables.
4NF/5NF come up rarely, and you should say so — knowing *why* each normal form exists
(preventing update, insert and delete anomalies: the same fact stored in many places and drifting
apart) matters more than reciting definitions.

**Memory hook:**
*1NF = **atomic** values. 2NF = kill **partial** deps. 3NF = kill **transitive** deps.
BCNF = every determinant is a key.*
Or: *"The key, the whole key, and nothing but the key — so help me Codd."*

---

### Q9.2 — When would you deliberately denormalize?

**Answer.** Normalization optimizes for **write integrity**; denormalization optimizes for
**read latency**. Denormalize when:

1. **Read-heavy reporting/analytics** where the join count is the bottleneck (star schema:
   fact table + denormalized dimensions).
2. **Expensive derived aggregates** — store `order.total_amount` instead of summing line items
   on every page load; store `post.comment_count`.
3. **Historical/point-in-time snapshots** — an invoice must keep the price and address *as of*
   the sale, not follow the current product/customer row. This isn't really denormalization,
   it's correct temporal modelling, and saying so scores points.
4. **Cross-service boundaries in microservices** — you can't join across databases, so you
   replicate a read model (CQRS).

The cost is **write amplification and drift risk**, so you must pick a maintenance strategy:
triggers, application-level updates in the same transaction, a materialized view with scheduled
refresh, or event-driven projection. Always add a reconciliation job that detects drift.

**Memory hook:** *Normalize until it hurts, denormalize until it works — and only with a plan
to keep copies in sync.*

---

### Q9.3 — How do you model a soft delete? What are the pitfalls?

```sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMP NULL;
```

Pitfalls, in order of how often they bite:
1. **Unique constraints break** — you can't re-register a deleted email. Fixes: a partial/filtered
   index `CREATE UNIQUE INDEX ... ON users(email) WHERE deleted_at IS NULL` (PostgreSQL,
   SQL Server) or include a nullable discriminator in the unique key (MySQL trick:
   `UNIQUE(email, deleted_at)` works because multiple NULLs are allowed).
2. **Every query needs the filter** — one forgotten `WHERE deleted_at IS NULL` leaks deleted
   data. In Hibernate use `@Where(clause="deleted_at is null")` / `@SQLRestriction` (Hibernate 6.3+),
   or better, a database **view** of live rows so raw SQL is safe too.
3. **Indexes and table bloat** grow forever; add `deleted_at IS NULL` as a partial-index
   predicate and archive old rows.
4. **FK integrity is a lie** — children can reference a "deleted" parent.

Alternative: move rows to an `*_archive` table in the same transaction. Cleaner reads, more
work on delete.

**Memory hook:** *Soft delete = broken uniqueness + a filter you will forget. Use a partial
unique index and a live-rows view.*

---

## 10. Indexes — How They Actually Work

### Q10.1 — What is an index and what data structure backs it?

**Answer.** An index is a separate, ordered data structure that maps key values to row
locations, turning an O(n) scan into an O(log n) lookup. The default in every mainstream RDBMS
is a **B+ tree**: balanced, all keys in leaf nodes, leaves linked together in sorted order.
That last property is why a B+ tree serves not just equality lookups but also **range scans**,
`ORDER BY`, `GROUP BY`, `MIN`/`MAX` and prefix `LIKE 'abc%'`.

Typical depth is 3–4 levels even for hundreds of millions of rows (high fanout, ~hundreds of
keys per page), so a lookup is ~3–4 page reads with the upper levels almost always cached.

Other index types worth naming:
- **Hash** — O(1) equality only, no ranges (PostgreSQL `USING hash`, MySQL MEMORY engine).
- **Bitmap** — Oracle; excellent for low-cardinality columns in data warehouses, terrible for OLTP
  (lock contention on write).
- **GIN / GiST** (PostgreSQL) — inverted index for `jsonb`, arrays, full-text; GiST for
  geometric/range types and exclusion constraints.
- **Full-text** — inverted index for search (`MATCH ... AGAINST`, `tsvector`).
- **BRIN** (PostgreSQL) — tiny min/max summary per block range; great for huge naturally-ordered
  data like append-only timestamps.

**Memory hook:** *B+ tree: sorted leaves in a linked list → equality AND ranges AND ORDER BY.*

---

### Q10.2 — Clustered vs non-clustered index?

**Answer.**
- **Clustered index** = the table *is* the index; leaf nodes contain the full rows in key order.
  One per table. In **InnoDB** it's the primary key (if none, a unique NOT NULL index, else a
  hidden 6-byte rowid). In **SQL Server** you choose it.
- **Non-clustered / secondary index** = separate structure, leaves contain the key plus a
  **pointer to the row**. In InnoDB that pointer is the **primary key value**; in SQL Server it's
  a RID (heap) or the clustered key.

Two crucial consequences for InnoDB:
1. **Every secondary index lookup that needs non-indexed columns does a second lookup into the
   clustered index** ("bookmark lookup"). Hence covering indexes matter.
2. **A wide PK bloats every secondary index**, because the PK is duplicated in all of them.
   A 36-char `UUID` string PK on a table with 5 secondary indexes is a serious sin.
3. **A random PK (UUIDv4) causes page splits and fragmentation** on insert, because rows must be
   inserted in key order. Use auto-increment/sequence, or a time-ordered UUID (UUIDv7 /
   ULID / "ordered UUID" byte-swapped form). This is a great, very senior thing to volunteer.

PostgreSQL has **no clustered indexes** — all indexes are secondary over a heap, and the
`CLUSTER` command is a one-time physical reorder. Its equivalent concern is HOT updates and
`fillfactor`.

**Memory hook:** *Clustered = the table sorted. Secondary in InnoDB stores the PK → keep PKs
narrow and sequential.*

---

### Q10.3 — Composite index: explain the leftmost-prefix rule.

**Answer.** Given `INDEX idx (a, b, c)`, the index is sorted by `a`, then `b` within equal `a`,
then `c`. You can use it for any **leftmost prefix** of the columns:

| Predicate | Uses index? |
|---|---|
| `WHERE a = 1` | Yes |
| `WHERE a = 1 AND b = 2` | Yes |
| `WHERE a = 1 AND b = 2 AND c = 3` | Yes (full) |
| `WHERE a = 1 AND c = 3` | Partially — seeks on `a`, then filters `c` (can still "skip scan" in Oracle/MySQL 8.0.13+) |
| `WHERE b = 2` | **No seek** (may do an index-only scan if the index is narrow, but no efficient seek) |
| `WHERE b = 2 AND c = 3` | **No** |
| `WHERE a = 1 ORDER BY b` | Yes — sort is free |
| `WHERE a > 1 AND b = 2` | Seeks on `a` range; `b` **cannot** be used for seeking after a range |

**The range rule:** columns after the first inequality/range predicate stop contributing to the
seek. So **order columns as: equality predicates first, then the column you sort by, then range
predicates last.** That ordering rule is exactly what a senior candidate is expected to state.

Also consider **cardinality**: for equality columns, putting the more selective one first helps
if only some queries filter on it, but the real driver is your query patterns, not cardinality
alone.

**Memory hook:** *Left to right, no skipping, stop at the first range. "**E**quality, **S**ort,
**R**ange" = ESR.*

---

### Q10.4 — What is a covering index / index-only scan?

**Answer.** An index that contains **all columns the query needs**, so the engine never touches
the table. In the plan: `Using index` (MySQL) or `Index Only Scan` (PostgreSQL).

```sql
-- Query
SELECT customer_id, status FROM orders WHERE customer_id = 42 AND created_at > ?;

-- Covering index: filter columns + selected columns
CREATE INDEX idx_orders_cover ON orders (customer_id, created_at, status);

-- SQL Server / PostgreSQL 11+ can put payload columns in the leaf only:
CREATE INDEX idx ON orders (customer_id, created_at) INCLUDE (status);
```

`INCLUDE` columns are not part of the sort key (can't be seeked on) but avoid the table lookup
while keeping the key narrow. In PostgreSQL, index-only scans additionally require the
visibility map to say the page is all-visible — so an un-`VACUUM`ed table may still hit the heap.

**Memory hook:** *Covering index = "the answer is in the index, don't open the book."*

---

### Q10.5 — When does an index NOT help, or actively hurt?

**Answer.**

**Won't be used (non-sargable predicates):**
```sql
WHERE YEAR(created_at) = 2026            -- function on the column → rewrite as a range
  --> WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'
WHERE UPPER(email) = 'A@B.COM'           -- → store normalized, or create a functional index
  --> CREATE INDEX ON users (UPPER(email));   -- functional/expression index
WHERE amount * 1.18 > 1000               -- → WHERE amount > 1000/1.18
WHERE name LIKE '%smith%'                -- leading wildcard → full-text or trigram index (pg_trgm)
WHERE varchar_col = 12345                -- implicit type conversion; MySQL converts the COLUMN
                                         --   to a number, killing the index. Match your types!
WHERE status <> 'DONE'                   -- negation is rarely selective
WHERE col IS NOT NULL                    -- often unselective
```

**Won't help even if usable:**
- **Low selectivity** — a boolean/gender column where each value is 50% of rows; the optimizer
  correctly prefers a seq scan. (Rule of thumb: if a predicate matches more than ~10–25% of the
  table, a full scan with sequential I/O beats random index lookups.)
- **Small tables** — the whole table is one or two pages.
- **Stale statistics** → the optimizer mis-estimates and picks the wrong plan.

**Actively hurts:**
- Every `INSERT`/`UPDATE`/`DELETE` must maintain every index → write amplification, more WAL/redo,
  more lock contention, more disk.
- Redundant indexes: `(a)` is redundant if `(a,b)` exists. Audit and drop them.
- More indexes = a bigger plan search space and more chance of a bad choice.

**Memory hook:** *Don't wrap the column in a function, don't lead with a wildcard, don't mismatch
types, don't index a coin flip.*

---

### Q10.6 — What are cardinality and selectivity?

**Answer.** **Cardinality** = number of distinct values in a column. **Selectivity** =
`distinct values / total rows`, i.e. how well a predicate narrows the set (high selectivity =
few rows returned = index-friendly). The optimizer uses histograms and n-distinct statistics to
estimate this. When the estimate is wrong — skewed data, correlated columns, stale stats — you
get a bad plan. Remedies: `ANALYZE` / `ANALYZE TABLE` / `DBMS_STATS`, create extended statistics
on correlated column pairs (PostgreSQL `CREATE STATISTICS`), or increase the histogram target
on skewed columns.

**Example:** `orders` has 10M rows. `status` has 4 distinct values → cardinality 4, selectivity
`4 / 10M` ≈ 0 (poor): `WHERE status = 'SHIPPED'` may return millions of rows, so an index is
pointless. `order_id` has 10M distinct values → selectivity 1.0 (perfect): `WHERE order_id = 42`
returns one row. *Skew* is the catch — if 99% of rows are `'SHIPPED'` and 0.01% are `'FAILED'`,
an index on `status` is excellent for `'FAILED'` and useless for `'SHIPPED'`, yet a naive
"4 distinct values" statistic would treat both the same. Histograms exist precisely to capture
that, and "stale statistics" means the optimizer is working from an outdated picture of the data.

**Memory hook:** *Selectivity high = few rows = use the index. The optimizer only knows what
statistics tell it.*

---

## 11. Query Optimization & Reading Execution Plans

### Q11.1 — Walk me through how you'd diagnose a slow query.

**Answer — say this as a repeatable runbook, not a list of tricks:**

1. **Confirm and measure.** Get the actual SQL and real parameter values (from `pg_stat_statements`,
   MySQL slow query log / `performance_schema.events_statements_summary_by_digest`, or APM).
   Know the p95 latency and the row counts involved.
2. **Get the plan with real numbers.** `EXPLAIN (ANALYZE, BUFFERS)` in PostgreSQL,
   `EXPLAIN ANALYZE` in MySQL 8.0.18+, actual execution plan in SQL Server. `EXPLAIN` alone gives
   estimates; `ANALYZE` gives estimates **and** actuals.
3. **Find the biggest gap between estimated and actual rows.** That node is where the optimizer
   was lied to — usually stale stats, a correlated predicate, or a non-sargable expression.
4. **Look for the classic red flags** (table below).
5. **Fix in order of leverage:** rewrite the query so it's sargable → add/adjust an index →
   refresh statistics → restructure the schema (denormalize/partition/materialize) → last resort,
   optimizer hints.
6. **Verify** on production-like data volume, and check you didn't slow down writes.

**Red flags in a plan:**

| Symptom | Likely cause / fix |
|---|---|
| `Seq Scan` / `type: ALL` on a big table | Missing index, or non-sargable predicate |
| Rows estimated 1, actual 2,000,000 | Stale statistics or correlated columns → `ANALYZE`, extended stats |
| Nested Loop with huge outer row count | Missing index on inner join column; force/allow hash join |
| `Using filesort`, `Sort` node spilling to disk | No index matching `ORDER BY`; raise `work_mem`; add composite index |
| `Using temporary` (MySQL) | Temp table for `GROUP BY`/`DISTINCT`/`UNION` |
| Hash join `Batches: 8` (Postgres) | Hash spilled to disk; increase `work_mem` |
| High `Buffers: read=` vs `hit=` | Cold cache / poor locality / table bloat |
| `Rows Removed by Filter: 900000` | Index found rows the filter then threw away → widen the index |
| Nested Loop **executed 50,000 times** | Correlated subquery or N+1 from the ORM |

**Memory hook:** *Measure → plan with actuals → find the estimate/actual gap → make it sargable →
index → verify.*

---

### Q11.2 — How do you read `EXPLAIN ANALYZE` output?

**Answer.** Read the tree **inside-out and bottom-up**: the deepest, most-indented nodes execute
first and feed their parents.

```
Nested Loop  (cost=0.29..8.34 rows=1 width=68) (actual time=0.03..142.7 rows=1 loops=1)
  ->  Seq Scan on orders o  (cost=... rows=1 ...) (actual time=... rows=50000 loops=1)
        Filter: (status = 'NEW')
        Rows Removed by Filter: 950000
  ->  Index Scan using customers_pkey on customers c (actual time=0.002..0.002 rows=1 loops=50000)
```

What to extract:
- **`cost=startup..total`** — arbitrary optimizer units, only comparable within a plan.
- **`rows=` (estimated) vs `actual rows=`** — the single most important comparison.
- **`loops=`** — for inner nodes, `actual time` is **per loop**; multiply by `loops` for total.
  Here 50,000 loops × 0.002ms is where the time went.
- **`Buffers: shared hit=X read=Y`** — cached vs from disk (needs `BUFFERS`).
- **Node types** worth knowing: `Seq Scan`, `Index Scan`, `Index Only Scan`, `Bitmap Heap Scan`
  + `Bitmap Index Scan` (many scattered rows), `Nested Loop`, `Hash Join`, `Merge Join`,
  `Sort`, `HashAggregate`, `GroupAggregate`, `Gather`/`Gather Merge` (parallelism).

MySQL: `EXPLAIN` columns to call out — `type` (best→worst: `system, const, eq_ref, ref, range,
index, ALL`), `key` (index chosen), `key_len`, `rows`, `filtered`, and `Extra`
(`Using index` = covering & good; `Using where`; `Using temporary`; `Using filesort` = bad).
`EXPLAIN FORMAT=JSON` gives cost details; `EXPLAIN ANALYZE` gives actuals.

**Memory hook:** *Bottom-up, estimate vs actual, and always multiply inner time by `loops`.*

---

### Q11.3 — Give me 10 concrete query optimization techniques.

1. **Select only the columns you need** — no `SELECT *`. Enables covering indexes, cuts network
   and (with Hibernate) avoids loading LOBs and triggering lazy loads.
2. **Make predicates sargable** — no functions/arithmetic/casts on indexed columns (Q10.5).
3. **Right index, right order** — equality → sort → range (ESR), consider covering/`INCLUDE`.
4. **Replace `OR` across different columns with `UNION ALL`** — `OR` often defeats index usage;
   two indexed branches unioned can be far faster. (Or rely on a Bitmap `OR` in PostgreSQL.)
5. **Filter early, aggregate late** — push predicates into subqueries/CTEs, reduce rows before
   joining; join on indexed keys.
6. **`EXISTS` instead of `IN` for large subqueries; `UNION ALL` instead of `UNION`.**
7. **Keyset (seek) pagination instead of `OFFSET`** for deep pages (Q11.4).
8. **Batch instead of row-at-a-time** — one `INSERT ... VALUES (...),(...),(...)`, `IN (...)`
   lookups instead of N single-row queries, JDBC batching.
9. **Pre-aggregate** — summary tables or materialized views refreshed on a schedule for
   dashboards; don't re-scan a billion rows per page view.
10. **Partition huge tables** on the column you always filter by (usually a date) so the planner
    prunes whole partitions.
11. **Keep statistics fresh, remove bloat** — `ANALYZE`/`VACUUM`, `OPTIMIZE TABLE`, rebuild
    fragmented indexes.
12. **Avoid `DISTINCT` used to paper over a bad join** — it hides duplicate-row bugs and forces
    a sort. Fix the join instead.

**Two of these, shown concretely:**

```sql
-- #4: OR across two different columns often can't use either index well
SELECT * FROM users WHERE email = ? OR phone = ?;
-- rewrite: each branch uses its own index; UNION (not ALL) removes a row matching both
SELECT * FROM users WHERE email = ?
UNION
SELECT * FROM users WHERE phone = ?;

-- #8: N single-row lookups vs one batched lookup (N network round trips -> 1)
-- slow:  for (id : ids) SELECT * FROM products WHERE id = ?;
SELECT * FROM products WHERE id IN (?, ?, ?, ...);
```

Why the order of attack matters: techniques 1–3 and 12 cost nothing and fix most problems;
9–10 (summary tables, partitioning) add maintenance burden, so reach for them only when
measurement (Q11.1) shows the simple fixes aren't enough.

---

### Q11.4 — Why is `LIMIT ... OFFSET 500000` slow, and what's the fix?

**Answer.** `OFFSET n` still **fetches and discards** the first n rows — so page 10,000 costs
10,000 pages of work. It also gives **inconsistent results** if rows are inserted or deleted
between page requests (rows shift, users see duplicates or skips).

**Fix: keyset / seek pagination** — remember the last row's sort key and use a `WHERE`:

```sql
-- Page 1
SELECT id, created_at, ...
FROM   orders
WHERE  customer_id = ?
ORDER  BY created_at DESC, id DESC
LIMIT  20;

-- Page N: pass the last row's (created_at, id) as the cursor
SELECT id, created_at, ...
FROM   orders
WHERE  customer_id = ?
  AND  (created_at, id) < (?, ?)         -- row-value comparison; tuple compare
ORDER  BY created_at DESC, id DESC
LIMIT  20;
-- Index: (customer_id, created_at DESC, id DESC)  → O(log n) regardless of page depth
```

Always include a **unique tiebreaker** (`id`) in the sort key, or rows with equal timestamps
appear on two pages. In Spring Data, this means returning a `Slice`/custom cursor rather than a
`Page`. Bonus point: a `Page` also runs a `SELECT COUNT(*)` which on a big table is itself a
full scan — use `Slice` when you don't need the total, or an approximate count
(`reltuples` in PostgreSQL).

**Memory hook:** *OFFSET counts rows it throws away. Keyset pagination remembers where it
stopped. Always add a unique tiebreaker.*

---

### Q11.5 — `COUNT(*)` on a 500M-row table takes 40 seconds. Options?

**Answer.**
1. **Ask if exact is required.** Usually "About 12M results" is fine.
2. **Approximate count** — PostgreSQL: `SELECT reltuples::bigint FROM pg_class WHERE relname='t'`
   (updated by `ANALYZE`/autovacuum); MySQL InnoDB: `SHOW TABLE STATUS` / `information_schema.TABLES.TABLE_ROWS`
   (an estimate). PostgreSQL `EXPLAIN` row estimate for a filtered count.
3. **Maintained counter table** — incremented by trigger or by the application, ideally
   sharded across N rows to avoid a single hot row, then summed.
4. **Materialized view / summary table** refreshed periodically.
5. **Covering index** — `COUNT(*)` with a narrow index can do an index-only scan, much cheaper
   than a heap scan (still O(n) though).
6. **Bound the count** — `SELECT COUNT(*) FROM (SELECT 1 FROM t WHERE ... LIMIT 1001) x`, and
   display "1000+".
7. Note: **MyISAM stores an exact row count** so `COUNT(*)` is O(1); InnoDB cannot, because of MVCC —
   each transaction may see a different number of rows. That MVCC explanation is the answer they want.

**Memory hook:** *Exact counts are O(n) under MVCC. Estimate, cache, or cap.*

---

## 12. Transactions, ACID & Isolation Levels

### Q12.1 — Explain ACID.

**Answer.**
- **Atomicity** — all-or-nothing. Implemented via undo log / rollback segments / WAL.
- **Consistency** — the transaction moves the DB from one valid state to another, respecting
  all constraints, triggers and cascades.
- **Isolation** — concurrent transactions don't corrupt each other's view; controlled by
  isolation levels, implemented with locking and/or MVCC.
- **Durability** — once committed, it survives a crash. Implemented via write-ahead logging
  (`fsync` of WAL/redo before commit ack) plus checkpoints; `innodb_flush_log_at_trx_commit=1`
  and `synchronous_commit=on` are the settings that actually guarantee it.

**Classic example — transferring ₹500 from account A to B** (two `UPDATE`s):
- *Atomicity:* if the debit succeeds and the credit fails (crash, constraint error), the debit is
  undone — you never lose money in between. The database keeps an *undo log*: a record of how to
  reverse each change.
- *Consistency:* a `CHECK (balance >= 0)` rejects a transfer that would overdraw, so the data
  always satisfies its rules. (Note: C relies on the constraints *you* define — the database
  can't enforce business rules it doesn't know about.)
- *Isolation:* another transaction summing all balances mid-transfer shouldn't see ₹500
  "vanished" (debited but not yet credited).
- *Durability:* the *write-ahead log* (WAL / redo log) is a sequential file where every change is
  written and flushed to disk **before** the commit is acknowledged; after a crash the database
  replays it to restore committed work.

Interview probe: *"Is a crash mid-commit safe?"* → yes; uncommitted changes are rolled back from
the undo log, committed ones are re-applied from the WAL on restart.

**Memory hook:** *A = all-or-nothing, C = rules hold, I = no peeking, D = survives the crash.*

---

### Q12.2 — What are the read phenomena?

**Answer.**

| Phenomenon | What happens |
|---|---|
| **Dirty read** | You read another transaction's **uncommitted** data; it then rolls back |
| **Non-repeatable read** | You read the same **row** twice and get different values (someone committed an `UPDATE` in between) |
| **Phantom read** | You run the same **range query** twice and get a different **set of rows** (someone committed an `INSERT`/`DELETE`) |
| **Lost update** | Two transactions read-modify-write the same row; the second silently overwrites the first |
| **Write skew** | Two transactions each read a set, verify a constraint, and write disjoint rows — individually fine, together they violate the invariant (e.g. two doctors both go off-call because each sees the other as on-call). **Allowed under snapshot isolation / Postgres REPEATABLE READ; blocked only by SERIALIZABLE.** |

**Timeline examples** (T1 and T2 are two concurrent transactions, balance of account 1 = 100):

```
Dirty read          T1: UPDATE bal=50 (uncommitted)   T2: SELECT bal -> 50   T1: ROLLBACK   (T2 used a value that never existed)
Non-repeatable read T1: SELECT bal -> 100             T2: UPDATE bal=50; COMMIT   T1: SELECT bal -> 50   (same row, new value)
Phantom read        T1: SELECT count(*) WHERE amt>10 -> 5   T2: INSERT row amt=20; COMMIT   T1: same query -> 6
Lost update         T1: read 100   T2: read 100   T1: write 100+10   T2: write 100+20 -> final 120, T1's +10 is gone
```

Why they matter: each phenomenon is a way two correct-in-isolation transactions produce a wrong
result when interleaved. Isolation levels (Q12.3) are simply a menu of *which* of these you
are willing to tolerate in exchange for more concurrency.

**Memory hook:** *Dirty = uncommitted. Non-repeatable = **row** changed. Phantom = **rows**
appeared. Write skew = the sneaky one that needs SERIALIZABLE.*

---

### Q12.3 — The four isolation levels and what each prevents.

| Level | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| READ UNCOMMITTED | Possible | Possible | Possible |
| READ COMMITTED | Prevented | Possible | Possible |
| REPEATABLE READ | Prevented | Prevented | Possible *(per the standard)* |
| SERIALIZABLE | Prevented | Prevented | Prevented |

**Real-world deviations you should mention — this is what separates a senior answer:**
- **Defaults:** MySQL/InnoDB = **REPEATABLE READ**. PostgreSQL, Oracle, SQL Server = **READ
  COMMITTED**. (Oracle doesn't implement READ UNCOMMITTED at all; its "READ ONLY" and
  SERIALIZABLE are snapshot-based.)
- **InnoDB's REPEATABLE READ prevents phantoms in practice** for locking reads, because
  next-key locking (record + gap locks) blocks inserts into scanned ranges.
- **PostgreSQL's REPEATABLE READ is snapshot isolation**: it prevents phantoms but allows
  **write skew**; conflicting concurrent updates raise a serialization failure you must retry.
- **PostgreSQL SERIALIZABLE uses SSI** (Serializable Snapshot Isolation) — optimistic, no extra
  locks, but transactions can abort with `40001 could not serialize access`; **your application
  must implement retry**.
- **READ UNCOMMITTED is meaningless in PostgreSQL** (treated as READ COMMITTED) because MVCC
  never exposes uncommitted rows.

**Memory hook:** Levels ladder — *U → C → R → S*, each rung removing one anomaly.
*"**U**gly **C**ats **R**arely **S**leep."*
And remember the two defaults: **MySQL = RR, Postgres = RC**.

---

### Q12.4 — What is MVCC?

**Answer.** Multi-Version Concurrency Control: instead of readers taking locks, each write
creates a **new version** of the row, and each transaction reads the version consistent with its
snapshot. Therefore **readers never block writers and writers never block readers** — only
writer-vs-writer on the same row conflicts.

- **PostgreSQL** stores old versions in the heap itself (`xmin`/`xmax` per tuple), so dead
  tuples accumulate → `VACUUM` reclaims them; long-running transactions prevent vacuuming and
  cause **bloat**, plus the risk of transaction-ID wraparound.
- **InnoDB** keeps old versions in the **undo log** and rolls back the row on read, with a
  hidden `DB_TRX_ID`/`DB_ROLL_PTR` per row. Long transactions grow the undo/history list.
- **Oracle** uses undo segments; a too-old snapshot gives "ORA-01555 snapshot too old".

Practical consequence for a Spring Boot dev: **keep transactions short**. A `@Transactional`
method that calls a slow external REST API holds a snapshot and a connection open, causing
bloat, blocked DDL, replication lag and pool exhaustion.

**Memory hook:** *MVCC = versions, not locks, for readers. The price is garbage (VACUUM/undo),
so short transactions.*

---

### Q12.5 — What is a savepoint / nested transaction?

```sql
BEGIN;
  INSERT INTO orders ...;
  SAVEPOINT sp_items;
  INSERT INTO order_items ...;        -- suppose this fails
  ROLLBACK TO SAVEPOINT sp_items;     -- undo only the item, keep the order
  RELEASE SAVEPOINT sp_items;
COMMIT;
```

True nested transactions don't exist in most engines; savepoints are the mechanism, and they are
exactly what Spring's `PROPAGATION_NESTED` uses under the hood. Note: in PostgreSQL, **any error
aborts the whole transaction** unless you had a savepoint — so `@Transactional(propagation = NESTED)`
or explicit savepoints matter more there than in MySQL.

**Memory hook:** *Savepoint = partial rollback. Spring's NESTED = savepoint.*

---

### Q12.6 — Two-phase commit / distributed transactions — and what you'd do instead.

**Answer.** **2PC / XA**: a coordinator asks all participants to *prepare* (durably promise they
can commit), then, once all vote yes, tells them to *commit*. Guarantees atomicity across
resources but is **blocking** — if the coordinator dies after prepare, participants hold locks
indefinitely; it's slow and doesn't scale, and cloud-managed databases often disable it.

In microservices, prefer:
- **Saga pattern** — a sequence of local transactions with compensating actions
  (choreographed via events, or orchestrated by a state machine). Eventual consistency.
- **Transactional Outbox** — write the business row and an `outbox` event row **in the same local
  transaction**, then a separate poller/CDC (Debezium) publishes to Kafka. This is the standard
  answer to "how do you atomically update the DB and publish an event?" and it's what
  interviewers want to hear instead of `@Transactional` around a Kafka producer.
- **Idempotent consumers** with a `processed_message_id` unique key, so retries are safe.

**Memory hook:** *2PC blocks. Real systems use Outbox + Saga + idempotency.*

---

## 13. Locking, Deadlocks & Concurrency Patterns

### Q13.1 — What kinds of locks exist?

**Answer.**
- **By mode:** **Shared (S)** — many readers; **Exclusive (X)** — one writer, blocks everything;
  **Update (U)** (SQL Server) — prevents conversion deadlocks; **Intent locks (IS/IX/SIX)** —
  declared at a higher granularity so the engine doesn't have to check every row.
- **By granularity:** row → page → extent → table → schema/metadata → database.
  **Lock escalation** (SQL Server) converts many row locks into a table lock at ~5,000 locks,
  which suddenly serializes everything — a real production surprise when you delete in big batches.
- **InnoDB specifics:** **record lock** (one index record), **gap lock** (the space between
  records, stops inserts), **next-key lock** (record + preceding gap — how RR prevents phantoms),
  **insert intention lock**, and table-level **metadata locks (MDL)** that make an
  `ALTER TABLE` wait behind a long-running transaction and then block everyone behind it.

**Plain-language walk-through:**
- *Why modes exist:* many sessions may read the same row safely at once (**shared**), but a
  writer must be alone (**exclusive**). Compatibility: S+S OK; S+X and X+X block.
- *Why intent locks exist:* if T1 holds a row lock and T2 wants to lock the **whole table**, the
  engine would have to scan every row to check. Instead T1 first places an **intent lock**
  (IX = "I'm locking some rows inside") on the table, so T2 sees it instantly and waits.
- *Gap lock example (InnoDB, REPEATABLE READ):* `SELECT * FROM t WHERE id BETWEEN 10 AND 20 FOR
  UPDATE` locks existing rows **and the empty gaps** between them, so another transaction can't
  `INSERT id=15` — which is how phantoms are prevented. The flip side: unrelated inserts block,
  and two sessions can deadlock on gaps.
- *Metadata lock (MDL) outage:* a forgotten open transaction that read `orders` holds an MDL;
  your `ALTER TABLE orders ...` queues behind it; every later query on `orders` queues behind the
  `ALTER` — the site appears frozen though nothing is "slow".

**Memory hook:** *S shares, X excludes. Row → page → table. InnoDB's gap locks are why RR has
no phantoms.*

---

### Q13.2 — Optimistic vs pessimistic locking. Which do you use and how, in Spring?

**Answer.**

**Pessimistic** — lock the row up front; others wait.
```sql
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;         -- X lock, others block
SELECT * FROM accounts WHERE id = 1 FOR SHARE;          -- S lock (MySQL: LOCK IN SHARE MODE)
SELECT ... FOR UPDATE NOWAIT;                            -- fail immediately instead of waiting
SELECT ... FOR UPDATE SKIP LOCKED;                       -- skip locked rows (job queues!)
```
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
Optional<Account> findById(Long id);
```
Use when: contention is high, the conflict cost is high (money movement, inventory decrement,
seat booking), and the transaction is short. Risk: blocking, lock timeouts, deadlocks.

**Optimistic** — don't lock; detect conflict at write time using a version column.
```java
@Entity
class Account {
  @Id Long id;
  @Version Long version;      // Hibernate adds "AND version = ?" to every UPDATE
  BigDecimal balance;
}
// UPDATE accounts SET balance=?, version=version+1 WHERE id=? AND version=?
// 0 rows updated -> OptimisticLockException / ObjectOptimisticLockingFailureException
```
Use when: conflicts are rare, and especially across **long/"conversational" transactions** —
a user edits a form for 5 minutes; you cannot hold a DB lock that long. `@Version` also protects
against the **lost update** across separate HTTP requests (send the version to the client, e.g. as
an HTTP `ETag`, and receive it back). Combine with `@Retryable` or a manual retry loop.

**Memory hook:** *Pessimistic = "get out of my way" (lock now, block others). Optimistic =
"I'll check at the end" (`@Version`, retry on failure). Long user think-time → optimistic.*

**Follow-up:** *"Optimistic locking with a `@Version` field — what if two requests both retry
forever?"* → bounded retries with jitter, then surface a 409 Conflict.

---

### Q13.3 — What is a deadlock? How do you find and prevent them?

**Answer.** A deadlock is a cycle of lock waits: T1 holds A and wants B, T2 holds B and wants A.
The engine detects the cycle and **kills one transaction as the victim** (MySQL error 1213
`ER_LOCK_DEADLOCK`; PostgreSQL SQLSTATE `40P01`; Oracle ORA-00060). Distinguish this from a
**lock wait timeout** (MySQL 1205), which is one transaction simply waiting too long — no cycle.

**Diagnose:**
- MySQL: `SHOW ENGINE INNODB STATUS` → `LATEST DETECTED DEADLOCK` section; enable
  `innodb_print_all_deadlocks=ON` to get them all in the error log; `performance_schema.data_locks`.
- PostgreSQL: the deadlock is logged with both statements; `pg_locks` joined to
  `pg_stat_activity` shows current blocking; `log_lock_waits=on`.
- SQL Server: deadlock graph via Extended Events / trace flag 1222.

**Prevent (say these in order):**
1. **Access resources in a consistent order** everywhere in the codebase — e.g. always lock
   account ids ascending. This eliminates most cycles. *(In a transfer: `Math.min`/`Math.max`
   on the two account ids.)*
2. **Keep transactions short and small** — no external HTTP/messaging calls, no user think-time,
   no reading files inside a transaction.
3. **Do the locking early** in the transaction and touch the hot rows last-and-briefly.
4. **Reduce isolation** where safe (RR → RC removes gap locks in MySQL, a very common deadlock
   source on range updates).
5. **Use a single atomic statement** instead of read-then-write:
   `UPDATE inventory SET qty = qty - 1 WHERE id = ? AND qty >= 1` — check `rowsAffected`.
6. **Add the right indexes.** An unindexed `UPDATE ... WHERE` locks far more rows than you think
   (InnoDB locks every row it *examines*), turning row locks into near-table locks.
7. **Expect and retry.** Deadlocks are normal at scale; catch
   `CannotAcquireLockException`/`DeadlockLoserDataAccessException` and retry with exponential
   backoff + jitter (Spring Retry), ideally making the operation idempotent first.

**Memory hook:** *Same order, short transactions, fewer locked rows, and always retry the victim.*

---

### Q13.4 — How would you implement a reliable job queue in SQL?

**Answer.** `SELECT ... FOR UPDATE SKIP LOCKED` — the modern, correct answer (PostgreSQL 9.5+,
MySQL 8.0+, Oracle).

```sql
-- Each worker claims its own batch; no two workers get the same row, no worker blocks.
WITH claimed AS (
  SELECT id
  FROM   jobs
  WHERE  status = 'PENDING' AND run_after <= now()
  ORDER  BY priority DESC, id
  FOR UPDATE SKIP LOCKED
  LIMIT  10
)
UPDATE jobs j
SET    status = 'RUNNING', locked_by = :worker, locked_at = now(), attempts = attempts + 1
FROM   claimed c
WHERE  j.id = c.id
RETURNING j.*;
```

Without `SKIP LOCKED`, every worker queues behind the same first row (`FOR UPDATE` blocks) or
you get duplicate processing (no lock). Add: an index on `(status, run_after)`, a
`locked_at` reaper for crashed workers, an `attempts` cap into a dead-letter status, and
idempotent handlers. If throughput exceeds a few thousand jobs/sec, move to a real broker
(Kafka/SQS) — knowing *when to stop using the database as a queue* is itself a senior answer.

**Memory hook:** *`FOR UPDATE SKIP LOCKED` = the SQL job-queue primitive.*

---

## 14. Views, Materialized Views, Procedures, Triggers

### Q14.1 — View vs materialized view?

**Answer.**

| | View | Materialized View |
|---|---|---|
| Storage | None — a stored query, expanded at runtime | Physically stores the result set |
| Freshness | Always current | Stale until refreshed |
| Read cost | Same as the underlying query | Cheap (it's a table) |
| Indexable | No (indexes live on base tables) | **Yes** |
| Updatable | Sometimes (simple, single-table views) | No (refresh only) |

```sql
CREATE VIEW active_users AS SELECT * FROM users WHERE deleted_at IS NULL;

CREATE MATERIALIZED VIEW daily_sales AS
  SELECT sale_date, region, SUM(amount) AS total FROM sales GROUP BY sale_date, region;
CREATE UNIQUE INDEX ON daily_sales (sale_date, region);   -- required for CONCURRENTLY
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales;       -- doesn't block readers
```

MySQL has **no** materialized views — emulate with a summary table plus a scheduled event or
application job. SQL Server calls them **indexed views** (auto-maintained synchronously, with
heavy restrictions). Oracle supports incremental "fast refresh" via materialized view logs.

**Why use views at all?** Encapsulate complex joins, provide a stable API over a changing schema,
enforce row/column-level security (grant on the view, not the table), and hide soft-delete
filters so raw SQL can't leak deleted rows.

**Memory hook:** *View = saved query (fresh, slow). Materialized view = saved answer (fast, stale).*

---

### Q14.2 — Stored procedure vs function?

| | Procedure | Function |
|---|---|---|
| Return | Zero or many values via `OUT` params / result sets | Must return a value |
| Callable in `SELECT` | No | Yes |
| DML allowed | Yes | Restricted (PostgreSQL: only `VOLATILE`; Oracle: not in a query) |
| Transactions | Can `COMMIT`/`ROLLBACK` | Generally cannot |
| Invocation | `CALL p(...)` / `EXEC` | `SELECT f(...)` |

**Where a 9-YOE opinion is expected:** procedures reduce network round-trips and can be faster
for set-based batch work, but they fragment business logic across two languages, are painful to
version-control, unit-test, debug and deploy, and they lock you to a vendor. Modern stance:
keep business logic in the Spring service layer; use procedures narrowly for
data-intensive bulk operations where moving the data to the JVM would dominate the cost.

Calling one from Spring:
```java
@Procedure(procedureName = "calc_bonus")            // Spring Data JPA
Integer calcBonus(@Param("emp_id") Long empId);
// or
SimpleJdbcCall call = new SimpleJdbcCall(jdbcTemplate).withProcedureName("calc_bonus");
```

---

### Q14.3 — Triggers: when are they justified, and what are the risks?

**Answer.** A trigger is code that fires automatically on `INSERT`/`UPDATE`/`DELETE`
(`BEFORE`/`AFTER`, `FOR EACH ROW` or statement-level; `INSTEAD OF` on views).

**Justified for:** audit trails that must capture *every* change regardless of which application
or DBA made it; maintaining a denormalized counter atomically; enforcing an invariant too complex
for a `CHECK`.

**Risks:** invisible side effects (a developer reads the Java code and can't explain the data
change), they run inside your transaction so they extend lock duration, they can cascade into
other triggers, they break bulk loads, and they're a nightmare to debug.

**Interview stance:** "For audit I'd usually prefer Hibernate Envers or an
`@EntityListeners`/`AuditingEntityListener` (`@CreatedDate`, `@LastModifiedBy`) so it's testable
and versioned with the app — but if the compliance requirement is *'no change may ever escape
the audit log, including manual SQL'*, then a database trigger is the only correct answer."
That trade-off framing is what they're probing for.

**Example (PostgreSQL)** — audit every salary change, no matter who made it:

```sql
CREATE FUNCTION log_salary_change() RETURNS trigger AS $$
BEGIN
  IF NEW.salary <> OLD.salary THEN                 -- OLD/NEW = the row before/after the UPDATE
    INSERT INTO salary_audit (emp_id, old_salary, new_salary, changed_at)
    VALUES (OLD.id, OLD.salary, NEW.salary, now());
  END IF;
  RETURN NEW;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_salary_audit AFTER UPDATE ON employees
FOR EACH ROW EXECUTE FUNCTION log_salary_change();
```

Reading it: `BEFORE` triggers can modify or reject the row (validation, defaulting); `AFTER`
triggers react to a change that already happened (auditing). Because the trigger runs inside the
caller's transaction, if the audit insert fails the original `UPDATE` rolls back too — good for
integrity, bad for latency.

---

## 15. Partitioning, Sharding, Replication

### Q15.1 — Partitioning vs sharding?

**Answer.** **Partitioning** splits one table into physical pieces **within one database**;
**sharding** splits data across **multiple database servers**.

Partitioning types:
- **Range** — by date, the most common (`PARTITION BY RANGE (created_at)`).
- **List** — by discrete values (country, tenant).
- **Hash** — even distribution when there's no natural range key.
- **Composite** — range then hash sub-partitions.

```sql
-- PostgreSQL declarative range partitioning by month
CREATE TABLE orders (id BIGINT, order_date DATE NOT NULL, total NUMERIC(12,2))
  PARTITION BY RANGE (order_date);
CREATE TABLE orders_2026_01 PARTITION OF orders
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- Query with the key -> planner touches only the matching partition ("pruning"):
SELECT * FROM orders WHERE order_date >= '2026-01-10' AND order_date < '2026-01-20';
-- Retention: instant, versus DELETE of millions of rows
ALTER TABLE orders DETACH PARTITION orders_2025_01;  -- then DROP TABLE orders_2025_01;
```

Benefits: **partition pruning** (the planner skips irrelevant partitions), and
**`DROP PARTITION` for instant data retention** — dropping last year's partition is a metadata
operation, versus a `DELETE` of 200M rows. That retention argument is usually the real reason to
partition.

Costs: the partition key must be in your `WHERE` clause or you scan everything; **unique
constraints must include the partition key**; too many partitions slows planning; cross-partition
joins and global secondary indexes are awkward.

Sharding brings distributed-systems pain: cross-shard joins and transactions, rebalancing,
hot shards, and a routing layer (`AbstractRoutingDataSource`, Vitess, Citus, ShardingSphere).
**Say clearly that sharding is a last resort** — first exhaust vertical scaling, read replicas,
caching, archiving and partitioning.

**Memory hook:** *Partition = one server, many pieces (pruning + DROP PARTITION).
Shard = many servers (last resort).*

---

### Q15.2 — Read replicas: what breaks, and how do you handle it in Spring Boot?

**Answer.** Replication is **asynchronous** by default, so replicas lag (milliseconds normally,
seconds-to-minutes under load or long transactions). That produces the classic bug:
**read-your-own-writes violation** — the user saves a profile, is redirected to a read that goes
to a lagging replica, and sees stale data.

Mitigations:
1. Route reads that must be fresh to the **primary** — e.g. anything within N seconds of a write
   for that user ("sticky primary" window in the session).
2. Use `@Transactional(readOnly = true)` and an `AbstractRoutingDataSource` keyed off
   `TransactionSynchronizationManager.isCurrentTransactionReadOnly()` to route to replicas.
   Bonus: `readOnly = true` also sets the Hibernate flush mode to `MANUAL`, skipping dirty
   checking — a real performance win even on a single database.
3. **Monitor lag** (`pg_stat_replication` / `SHOW REPLICA STATUS` → `Seconds_Behind_Source`) and
   pull a replica out of rotation when it exceeds a threshold.
4. Use **synchronous replication** for the critical subset if the latency cost is acceptable
   (`synchronous_commit = remote_apply`, MySQL semi-sync / Group Replication).
5. Never write through a replica connection; make the replica datasource genuinely read-only so
   mistakes fail loudly.

**Memory hook:** *Replicas lag → read-your-writes breaks. `readOnly=true` routes reads and
disables dirty checking.*

---

## 16. SQL from Spring Boot — JDBC, JPA, Hibernate

### Q16.1 — `Statement` vs `PreparedStatement` vs `CallableStatement`

**Answer.**
- `Statement` — plain SQL string. Vulnerable to injection, no plan reuse. Avoid.
- `PreparedStatement` — pre-compiled with `?` placeholders. **Prevents SQL injection** (values
  are sent separately and never parsed as SQL), enables **server-side plan caching**, supports
  **batching**, and handles type conversion/escaping properly. Default choice.
- `CallableStatement` — invokes stored procedures/functions, supports `OUT`/`INOUT` params.

Caveat worth mentioning: you cannot parameterize **identifiers** (table/column names) or the
`ORDER BY` direction with a `PreparedStatement` — those must be whitelisted in code.
Also note MySQL's `useServerPrepStmts` and `cachePrepStmts` connection properties: without them,
"prepared" statements are emulated client-side and you lose the plan-cache benefit.

**Memory hook:** *PreparedStatement = safe + cached + batchable. Identifiers can never be
parameters.*

**How injection-safety actually works:** the driver/database receives the SQL *template*
(`SELECT * FROM users WHERE email = ?`) and the *values* separately. The template is parsed
once; the value is then bound as data and can never change the statement's structure — so
`x' OR '1'='1` is just a strange email address. Plan caching follows from the same idea: the
`?` version of the statement is identical every call, so the database can reuse its parsed/planned
form instead of re-parsing a new string per value.

```java
// Whitelisting an identifier (cannot be a ? parameter)
Set<String> allowed = Set.of("created_at", "total");
String col = allowed.contains(sortParam) ? sortParam : "created_at";
String sql = "SELECT * FROM orders WHERE customer_id = ? ORDER BY " + col + " DESC";
```

---

### Q16.2 — What is the N+1 select problem? Show all the fixes.

**Answer.** You fetch N parent rows with one query, then the ORM issues **one extra query per
parent** to load its association — 1 + N queries. It is the single most common performance bug in
Spring Boot applications, and interviewers ask it constantly.

```java
List<Order> orders = orderRepo.findAll();          // 1 query
for (Order o : orders) o.getItems().size();        // N queries!
```

**Fixes, in the order you should present them:**

1. **`JOIN FETCH` in JPQL** — one query, eager for this use case only:
```java
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items WHERE o.status = :s")
List<Order> findWithItems(@Param("s") Status s);
```
2. **`@EntityGraph`** — declarative, keeps derived query methods:
```java
@EntityGraph(attributePaths = {"items", "customer"})
List<Order> findByStatus(Status status);
```
3. **`@BatchSize(size = 50)`** on the collection/entity — turns N queries into N/50 queries using
   `WHERE parent_id IN (?, ?, ...)`. Best when you can't restructure the query, and it's the
   safest global default (`hibernate.default_batch_fetch_size=50`).
4. **DTO projection** — fetch exactly the columns you need, no entities, no lazy loading at all.
   Usually the fastest and my default for read endpoints:
```java
@Query("SELECT new com.x.OrderSummary(o.id, o.total, c.name) FROM Order o JOIN o.customer c")
List<OrderSummary> findSummaries();
```
5. **Subselect / `FetchMode.SUBSELECT`** — two queries total, the second re-running the parent
   query as a subquery.
6. **Native query or `JdbcTemplate`** for complex reporting.

**Critical trap — never fix N+1 with `FetchType.EAGER`.** It makes the cost unconditional for
every query everywhere, it can't be turned off, and multiple eager collections produce a
cartesian product. Correct answer: **map everything `LAZY`, then fetch eagerly per use case.**

**Second trap — `JOIN FETCH` + pagination.** `JOIN FETCH` on a collection with
`setFirstResult/setMaxResults` makes Hibernate fetch **all** rows and paginate **in memory**
(older versions logged `HHH000104: firstResult/maxResults specified with collection fetch; applying in memory`).
Fix: two queries — page the parent IDs first, then fetch the collections
`WHERE id IN :ids` — or use `@BatchSize`.

**Memory hook:** *Lazy by default + fetch on purpose. `JOIN FETCH` / `@EntityGraph` for one
query, `@BatchSize` for N/50, DTOs for reads. EAGER is never the fix.*

---

### Q16.3 — `LAZY` vs `EAGER`, and what causes `LazyInitializationException`?

**Answer.** Lazy associations are proxies (Hibernate `ByteBuddy` proxy or
`PersistentBag`/`PersistentSet` for collections) populated on first access. If the
`EntityManager`/session is already closed — typically because you returned the entity to the
controller/serializer after the `@Transactional` service method ended — you get
`LazyInitializationException: could not initialize proxy - no Session`.

Defaults to know: `@ManyToOne` and `@OneToOne` are **EAGER by default**;
`@OneToMany` and `@ManyToMany` are **LAZY by default**. So you must explicitly write
`@ManyToOne(fetch = FetchType.LAZY)` — an extremely common interview question.

**Proper fixes:** initialize what you need inside the transaction (`JOIN FETCH`, `@EntityGraph`,
`Hibernate.initialize()`), or return a **DTO** from the service so no proxy ever escapes.

**Anti-fixes to name and reject:** `spring.jpa.open-in-view=true` (on by default in Spring Boot,
and you should set it to `false`) keeps the session open for the whole request — it hides N+1
problems, holds a database connection during view rendering, and pushes queries into the
serialization layer. Also reject `hibernate.enable_lazy_load_no_trans=true`, which silently opens
a new session per lazy access.

**Memory hook:** *`@ManyToOne` is EAGER by default — fix it. Never let a proxy leave the
transaction; return DTOs. Turn `open-in-view` off.*

---

### Q16.4 — JPQL vs Criteria API vs native query vs QueryDSL — when do you use each?

| Approach | Use when |
|---|---|
| **Derived query methods** (`findByStatusAndCreatedAtAfter`) | Simple, 1–2 predicates |
| **JPQL `@Query`** | Most cases; portable, type-checked at startup, supports joins/DTOs |
| **Criteria API / Specifications** | **Dynamic** predicates built at runtime (search filters); verbose but composable and injection-safe |
| **QueryDSL** | Same as Criteria but with fluent, compile-time-safe syntax; needs code generation |
| **Native `@Query(nativeQuery = true)`** | Window functions, CTEs, recursive queries, vendor features (`ON CONFLICT`, hints), bulk DML |
| **`JdbcTemplate` / `JdbcClient`** | Reporting, bulk loads, when you want no ORM at all |

Point to make: **JPQL has no window functions or CTEs** (Hibernate 6 added window-function
support via HQL, but portability is limited). For analytical SQL, drop to native SQL or
`JdbcTemplate` and map to a DTO — don't torture JPQL.

**What "dynamic predicates" means, with an example:** a search screen where the user may fill any
combination of name / status / date. Derived methods and fixed `@Query`s would need one method per
combination; a `Specification` builds the `WHERE` from only the filters provided, using bound
parameters (so it's injection-safe):

```java
Specification<Order> spec = Specification.where(null);
if (status != null) spec = spec.and((r, q, cb) -> cb.equal(r.get("status"), status));
if (from != null)   spec = spec.and((r, q, cb) -> cb.greaterThanOrEqualTo(r.get("createdAt"), from));
Page<Order> page = orderRepo.findAll(spec, pageable);   // repo extends JpaSpecificationExecutor<Order>
```

Rule of thumb for choosing: start with the simplest option in the table that works; move down only
when you hit its limit (dynamic filters → Specifications; vendor-specific SQL → native).

---

### Q16.5 — Pagination in Spring Data: `Page` vs `Slice` vs `Stream`?

**Answer.**
- `Page<T>` — content + **total count**, so it fires a **second `SELECT COUNT(*)`** query. On a
  large table that count can cost more than the page itself.
- `Slice<T>` — content + `hasNext()` only (fetches `limit + 1` rows). No count query. Use this
  for infinite scroll.
- `Stream<T>` / `Window`/keyset via `ScrollPosition` (Spring Data 3.1+) — for large result
  streaming and keyset pagination.

```java
Page<Order>  p  = repo.findByStatus(status, PageRequest.of(0, 20, sort));   // SELECT ... LIMIT 20  + SELECT COUNT(*)
Slice<Order> sl = repo.findByStatus(status, PageRequest.of(0, 20, sort));   // SELECT ... LIMIT 21 (extra row => hasNext)
```

Reading it: `Page` lets a UI show "page 3 of 47" because it knows the total; the price is the extra
count query. `Slice` only knows "is there another page?" (it asks for one extra row), which is all
an infinite-scroll or "Next" button needs. `Stream` keeps a database cursor open — it must be
consumed inside a transaction and closed (try-with-resources), or it leaks a connection.

Always pass a **deterministic sort** (`Sort.by("createdAt").descending().and(Sort.by("id"))`),
otherwise page boundaries are undefined and rows repeat across pages. And remember Q11.4:
deep `OFFSET` is a scaling problem — keyset pagination is the real fix.

**Memory hook:** *`Page` costs a COUNT. `Slice` doesn't. Sort must include a unique tiebreaker.*

---

### Q16.6 — Explain the JPA entity lifecycle and the first-level cache.

**Answer.** States: **transient** (new, no id, unmanaged) → **managed/persistent** (in the
persistence context, tracked for dirty checking) → **detached** (session closed, no longer
tracked) → **removed** (scheduled for delete).

The **persistence context is the first-level cache**, always on and scoped to the
`EntityManager`/transaction. Consequences:
- `findById` twice in one transaction issues **one** query; the second is served from the cache
  and returns the **identical Java object** (`==` is true).
- **Dirty checking + automatic flush:** changing a managed entity's field results in an `UPDATE`
  at flush time with **no explicit `save()` call needed**. This surprises people, and interviewers
  love it. Flush happens before query execution, on commit, or on explicit `flush()`.
- **`save()` on a detached entity with an id → `merge()`**, which does a `SELECT` then an
  `UPDATE`. Two queries where you expected one; use `@Version` or an explicit update query if
  that matters.
- For bulk loops, `flush()` + `clear()` periodically or the persistence context grows unbounded
  (memory + O(n) dirty checking).

**Second-level cache** (Ehcache/Hazelcast/Infinispan via `@Cacheable`, plus the query cache) is
shared across transactions and off by default. Caveat: it's bypassed by native/bulk DML, so
`@Query("delete from ...")` leaves it stale unless you evict.

**Memory hook:** *Transient → Managed → Detached/Removed. The persistence context is a
write-behind cache with automatic dirty checking — managed entities save themselves.*

---

## 17. Spring `@Transactional` Deep Dive

### Q17.1 — How does `@Transactional` actually work?

**Answer.** Spring creates a **proxy** (JDK dynamic proxy for interfaces, CGLIB subclass
otherwise). The proxy's interceptor obtains a transaction from the `PlatformTransactionManager`,
binds the connection to the thread via `TransactionSynchronizationManager`, invokes your method,
then commits or rolls back.

Everything about that mechanism produces the classic gotchas:

1. **Self-invocation doesn't work.** Calling `this.otherTransactionalMethod()` bypasses the
   proxy entirely, so the annotation is ignored. Fix: move the method to another bean, inject
   self, use `TransactionTemplate`, or switch to AspectJ weaving.
2. **Only `public` methods are advised** (with the default proxy mode). `private`, `protected`,
   package-private and `final` methods (CGLIB can't override `final`) are silently ignored.
3. **Rollback rules:** by default Spring rolls back on **unchecked** exceptions
   (`RuntimeException`, `Error`) and **commits on checked exceptions**. If you catch an exception
   and don't rethrow, there's **no rollback**. Fix with
   `@Transactional(rollbackFor = Exception.class)`.
4. **Marking rollback-only:** if an inner `REQUIRED` method throws and is caught by the outer
   method, the shared transaction is already marked rollback-only → commit fails with
   `UnexpectedRollbackException: Transaction silently rolled back`. That exact error is a
   favourite interview question; the fix is `REQUIRES_NEW` for the inner call or don't swallow.
5. **`@Transactional` on a `@Async` or new thread does not propagate** — the transaction is
   thread-bound.

**Memory hook:** *Proxy ⇒ public, non-self-invoked, unchecked-exception rollback. Swallowing an
exception from a REQUIRED inner call gives you `UnexpectedRollbackException`.*

---

### Q17.2 — Explain all propagation levels.

| Propagation | Existing transaction | No transaction |
|---|---|---|
| **REQUIRED** (default) | Join it | Create a new one |
| **REQUIRES_NEW** | **Suspend** it, start an independent one (own commit/rollback, own connection) | Create new |
| **SUPPORTS** | Join it | Run non-transactionally |
| **NOT_SUPPORTED** | Suspend it, run non-transactionally | Run non-transactionally |
| **MANDATORY** | Join it | Throw `IllegalTransactionStateException` |
| **NEVER** | Throw | Run non-transactionally |
| **NESTED** | Create a **savepoint**; inner rollback keeps outer alive | Behaves like REQUIRED |

Practical notes: **`REQUIRES_NEW` uses a second connection** — nest a few of these under load and
you exhaust the Hikari pool and deadlock the application. Its legitimate use is "this must be
committed regardless of the outcome of the main work" — most commonly **audit logging or a
failure record**. `NESTED` requires JDBC savepoint support and doesn't work with JTA.

**Example — "audit must survive even if the main work fails":**

```java
@Service class OrderService {
  @Transactional                                  // REQUIRED: the main transaction
  public void placeOrder(Order o) {
    orderRepo.save(o);
    auditService.record("ORDER_ATTEMPT", o.getId());   // separate bean => goes through the proxy
    paymentService.charge(o);                          // may throw -> main tx rolls back
  }
}
@Service class AuditService {
  @Transactional(propagation = Propagation.REQUIRES_NEW)   // own connection, commits immediately
  public void record(String event, Long id) { auditRepo.save(new Audit(event, id)); }
}
```

If `charge()` throws, the order insert is rolled back but the audit row stays committed.
Contrast: with plain `REQUIRED` the audit row would join the main transaction and vanish on
rollback. Remember the self-invocation rule (Q17.1): `record` must live in a different bean,
otherwise the annotation is ignored.

**Memory hook:** *REQUIRED joins, REQUIRES_NEW suspends (new connection!), NESTED savepoints,
MANDATORY demands one, NEVER forbids one.*

---

### Q17.3 — Isolation, timeout and `readOnly` in `@Transactional`?

```java
@Transactional(
    isolation  = Isolation.REPEATABLE_READ,   // DEFAULT = the database's default
    timeout    = 10,                          // seconds; rolls back if exceeded
    readOnly   = true,                        // hint: Hibernate FlushMode.MANUAL, no dirty check
    propagation = Propagation.REQUIRED,
    rollbackFor = Exception.class,
    noRollbackFor = NotFoundException.class)
```

- `Isolation.DEFAULT` means "whatever the database does" — so the same code behaves differently
  on MySQL (RR) and PostgreSQL (RC). Be explicit when correctness depends on it.
- `readOnly = true` is a **hint**, not enforcement — it doesn't prevent writes on all drivers
  (PostgreSQL does set the transaction read-only; MySQL sets it too, but some setups ignore it).
  Its real value: skips Hibernate dirty checking and flushing, and lets you route to replicas.
- `timeout` is essential defence against a runaway query holding locks.

**Expanded:**
- **Isolation:** the table in Q12.3 applies. If you don't set it, the database default applies, so
  a method that's safe on PostgreSQL (READ COMMITTED) can show different anomalies on MySQL (REPEATABLE
  READ) — and vice versa. Set it explicitly only where you *rely* on a guarantee, e.g.
  `SERIALIZABLE` for a write-skew-prone check (and then be ready to retry on `40001`).
  Note it only takes effect when this method *starts* the transaction; if it merely joins an
  existing one, the outer settings win.
- **`readOnly = true`:** tells Spring/Hibernate "this method won't write", so Hibernate skips
  dirty checking (it doesn't need to compare every loaded entity against its snapshot at commit)
  and the routing layer can send it to a replica. Use it on query-only service methods — it's
  free performance. It is not a security guard: don't rely on it to stop writes.
- **`timeout`:** counted in seconds from transaction start; once exceeded, the *next* statement
  fails and the transaction rolls back. It bounds how long locks/snapshots can be held, but
  doesn't interrupt Java code that isn't talking to the database (a slow HTTP call).
- **`noRollbackFor`:** whitelists exceptions that should still commit — e.g. a "not found"
  business exception you catch and handle upstream — saving you from accidental rollbacks.

---

### Q17.4 — Where should `@Transactional` go, and how long should it last?

**Answer.** On the **service layer** (the unit of business work), not the repository (too fine
grained — you lose atomicity across repository calls) and not the controller (couples web
concerns to persistence and lengthens the transaction across serialization).

Golden rules:
- **Nothing slow inside a transaction:** no REST/HTTP calls, no S3 uploads, no Kafka publish
  awaiting ack, no user interaction, no `Thread.sleep`. Each of those holds a connection, a
  snapshot and possibly row locks for the duration — causing pool exhaustion, MVCC bloat,
  replication lag and lock timeouts.
- Pattern: **read/validate → (short transaction: write) → then do the external call**, with the
  outbox pattern if the external call must be atomic with the write.
- Fail fast with a `timeout`, and keep the number of statements small.

**Concrete before/after:**

```java
// BAD: connection + locks held while waiting on a remote service (could be seconds)
@Transactional
public void checkout(Cart c) {
  Order o = orderRepo.save(new Order(c));
  PaymentResult r = paymentClient.charge(o);      // slow HTTP call INSIDE the transaction
  o.setStatus(r.ok() ? PAID : FAILED);
}

// BETTER: short transaction for the write; external call outside; outbox if it must be atomic
public void checkout(Cart c) {
  Order o = txTemplate.execute(s -> orderRepo.save(new Order(c)));   // tx 1: ms
  PaymentResult r = paymentClient.charge(o);                          // no connection held
  txTemplate.executeWithoutResult(s -> orderRepo.updateStatus(o.getId(), r.ok() ? PAID : FAILED)); // tx 2
}
```

Why the service layer: it's where one *business operation* is defined, so "all repository calls
for this use case commit or roll back together" maps to one annotation.

**Memory hook:** *Transactions are for the database, not for the network. Service layer, short,
timed out.*

---

## 18. Connection Pooling (HikariCP)

### Q18.1 — Why pool connections, and what do the key HikariCP settings mean?

**Answer.** Opening a TCP connection + TLS + authentication + server-side session setup costs
tens of milliseconds and real memory on the database (PostgreSQL forks a process per connection).
A pool amortizes that and, critically, **bounds** concurrency against the database.

```yaml
spring.datasource.hikari:
  maximum-pool-size: 20      # HARD ceiling on concurrent DB work. Default 10.
  minimum-idle: 20           # usually = max, to avoid churn
  connection-timeout: 3000   # ms to wait for a connection before throwing
                             #   SQLTransientConnectionException: pool exhausted
  idle-timeout: 600000       # ms before an idle (above minimum-idle) connection is retired
  max-lifetime: 1800000      # ms; MUST be shorter than the DB/proxy idle kill time
  validation-timeout: 5000
  leak-detection-threshold: 20000   # log a stack trace if a connection is held > 20s
```

**Sizing** — the counter-intuitive part interviewers like: **bigger is not better.** A database
can only do so much concurrent work; excess connections just add context switching and lock
contention. HikariCP's own guidance is roughly
`connections = ((core_count * 2) + effective_spindle_count)` — for many services a pool of
**10–20 is right**, and the correct fix for saturation is usually faster queries or fewer
round-trips, not a bigger pool. Also budget: `pool_size × number_of_instances` must stay well
under the database's `max_connections`, leaving headroom for migrations, admin and replicas.
For very high instance counts, front the database with **PgBouncer** in transaction-pooling mode.

**Memory hook:** *Pool size caps DB concurrency — small and fast beats big and blocked.
`max-lifetime` < the DB's idle timeout.*

---

### Q18.2 — "Pool exhausted / connection is not available, request timed out after 30000ms" — causes?

**Answer.** In diagnosis order:
1. **Long transactions** — external HTTP calls or slow queries inside `@Transactional` (the #1 cause).
2. **Connection leaks** — a manually obtained `Connection`/`EntityManager` never closed. Turn on
   `leak-detection-threshold` and read the stack trace it prints.
3. **`REQUIRES_NEW` nesting** or a transaction calling `@Async` work that also needs a connection —
   each level holds one, so a pool of 10 deadlocks at 5 concurrent nested requests.
4. **Streaming a large `ResultSet`** while doing per-row work (holds the connection for the duration).
5. **Pool too small for genuine throughput** — verify with Hikari metrics
   (`hikaricp.connections.pending`, `.usage`, `.acquire`) before enlarging.
6. **Database-side blocking** — a lock wait or a runaway query makes every borrowed connection
   sit idle-in-transaction. Check `pg_stat_activity` / `SHOW PROCESSLIST` before touching the pool.

**Memory hook:** *Pool exhaustion is almost always a duration problem, not a size problem.*

**Why the arithmetic in #3 deadlocks:** with a pool of 10, suppose 10 requests each hold one
connection in an outer `@Transactional` and then call a `REQUIRES_NEW` method, which needs a
*second* connection. All 10 now wait for a connection that only frees up when one of the 10
finishes — a circular wait, so every request times out after `connection-timeout`. Hint to
confirm: Hikari shows `active = max` and `pending > 0` while the database itself is nearly idle.

**Quick triage rule:** if the database is busy → fix the queries; if the database is idle but the
pool is full → the application is holding connections without using them (long transactions,
leaks, nested connections).

---

## 19. Batch Operations & Bulk Data

### Q19.1 — How do you insert 1 million rows efficiently from Spring Boot?

**Answer, best to worst:**

1. **Native bulk load** — PostgreSQL `COPY` (via `CopyManager`), MySQL `LOAD DATA INFILE`.
   Orders of magnitude faster than any ORM path. Use for real ETL.
2. **`JdbcTemplate.batchUpdate`** with a tuned batch size (500–1000) and
   `rewriteBatchedStatements=true` on MySQL (without it, the driver still sends statements
   one-by-one — a very commonly asked detail).
```java
jdbcTemplate.batchUpdate(
    "INSERT INTO users (name, email) VALUES (?, ?)",
    users, 1000,
    (ps, u) -> { ps.setString(1, u.getName()); ps.setString(2, u.getEmail()); });
```
3. **Hibernate batching**, if you must use entities:
```yaml
spring.jpa.properties:
  hibernate.jdbc.batch_size: 50
  hibernate.order_inserts: true      # group same-table inserts so they can batch
  hibernate.order_updates: true
  hibernate.batch_versioned_data: true
```
```java
for (int i = 0; i < list.size(); i++) {
    em.persist(list.get(i));
    if (i % 50 == 0) { em.flush(); em.clear(); }   // else the persistence context OOMs
}
```
   **Critical gotcha:** `GenerationType.IDENTITY` **disables JDBC batching entirely**, because
   Hibernate must execute each insert to learn the generated id. Use `GenerationType.SEQUENCE`
   with a pooled optimizer (`allocationSize = 50`) — this is a favourite senior question.
4. **`saveAll()` in a loop** — the naive approach; fine for tens of rows, not thousands.

Also: drop/disable non-essential indexes and FK checks during a one-off massive load and rebuild
after (`SET FOREIGN_KEY_CHECKS=0`, `ALTER INDEX ... UNUSABLE`), commit in chunks so the undo/WAL
doesn't explode, and consider a staging table + a single set-based `INSERT ... SELECT`.

**Memory hook:** *COPY/LOAD DATA > `batchUpdate` > Hibernate batch (SEQUENCE, never IDENTITY) >
`saveAll` in a loop. `rewriteBatchedStatements=true` on MySQL.*

---

### Q19.2 — How do you write an UPSERT?

```sql
-- PostgreSQL / SQLite
INSERT INTO inventory (sku, qty) VALUES ('A1', 5)
ON CONFLICT (sku) DO UPDATE SET qty = inventory.qty + EXCLUDED.qty;
ON CONFLICT (sku) DO NOTHING;                       -- idempotent insert

-- MySQL
INSERT INTO inventory (sku, qty) VALUES ('A1', 5)
ON DUPLICATE KEY UPDATE qty = qty + VALUES(qty);    -- MySQL 8.0.20+: use an alias instead

-- Oracle / SQL Server / standard
MERGE INTO inventory t
USING (SELECT 'A1' AS sku, 5 AS qty FROM dual) s ON (t.sku = s.sku)
WHEN MATCHED THEN UPDATE SET t.qty = t.qty + s.qty
WHEN NOT MATCHED THEN INSERT (sku, qty) VALUES (s.sku, s.qty);
```

**Why not `SELECT` then `INSERT`/`UPDATE` in Java?** It's a race: two threads both see "not
exists" and both insert → duplicate-key error or duplicate rows. An upsert is a single atomic
statement. (And SQL Server's `MERGE` has well-known concurrency bugs unless you add
`HOLDLOCK`/`SERIALIZABLE` hints — a nice detail to drop.)

**Memory hook:** *Never read-then-write across the network. `ON CONFLICT` / `ON DUPLICATE KEY` /
`MERGE` — one atomic statement.*

---

## 20. Schema Migrations (Flyway / Liquibase)

### Q20.1 — Flyway vs Liquibase?

| | Flyway | Liquibase |
|---|---|---|
| Format | Plain SQL (`V1__init.sql`), plus Java migrations | XML/YAML/JSON/SQL changelogs |
| Style | Imperative, versioned, SQL-first | Declarative, database-agnostic abstraction |
| Rollback | Undo scripts (paid tier) — usually forward-only | Built-in `rollback` tags |
| Learning curve | Very low | Higher |
| Tracking table | `flyway_schema_history` (with a checksum per script) | `DATABASECHANGELOG` + `DATABASECHANGELOGLOCK` |

Both: run on startup (`spring-boot-starter` auto-configures), take a lock so concurrent instances
don't collide, and record a **checksum** — editing an already-applied migration causes a
validation failure (fix: `repair`, never edit history).

**How it works (Flyway example):** you add files named `V<version>__<description>.sql` to
`src/main/resources/db/migration`. On startup Flyway compares them with `flyway_schema_history`,
runs any new ones in version order, and stores each file's checksum.

```
db/migration/V1__create_users.sql      -- CREATE TABLE users (...)
db/migration/V2__add_users_phone.sql   -- ALTER TABLE users ADD COLUMN phone VARCHAR(20)
db/migration/V3__index_users_email.sql
```
```yaml
spring.flyway.enabled: true          # default when flyway-core is on the classpath
spring.flyway.baseline-on-migrate: true   # adopt an existing non-empty database
```

Why it exists: schema changes become versioned, reviewable code that runs identically in every
environment, instead of someone hand-running SQL on production. Why the checksum matters: it
proves production's history matches the repo; changing an applied file means environments have
silently diverged, so the app refuses to start. Fix mistakes with a *new* migration (V4) that
corrects V2, never by editing V2.

**Golden rule to state:** `spring.jpa.hibernate.ddl-auto` must be `validate` or `none` in any
environment that matters. `update` is non-deterministic, never drops anything, can't do data
migrations, and `create-drop` will eat your data.

---

### Q20.2 — How do you do a zero-downtime schema change? (Very common at 9 YOE)

**Answer.** During a rolling deploy, old and new application versions run **simultaneously**, so
every migration must be **backward compatible**. Use the **expand–migrate–contract** pattern.

Example: renaming `full_name` → `name`:
1. **Expand** — add the new nullable column `name`. Deploy code that **writes both** columns and
   reads `full_name`.
2. **Backfill** — copy data in **batches** (`UPDATE ... WHERE id BETWEEN ? AND ?`), not one giant
   statement.
3. **Switch reads** — deploy code that reads `name`, still writes both.
4. **Contract** — deploy code that only uses `name`, then drop `full_name` in a later release.

Rules for each change type:
- **Adding a column:** always **nullable or with a default**. `NOT NULL` without a default breaks
  the old version's inserts. (Note: PostgreSQL 11+ and MySQL 8 add a defaulted column as a
  metadata-only operation; earlier versions rewrite the whole table.)
- **Adding an index on a big table:** `CREATE INDEX CONCURRENTLY` (PostgreSQL — no table lock,
  but can't run inside a transaction, so mark the Flyway migration
  `-- flyway:executeInTransaction=false`); MySQL 5.6+ does online DDL (`ALGORITHM=INPLACE,
  LOCK=NONE`); for very large MySQL tables use `gh-ost` or `pt-online-schema-change`.
- **Dropping a column/table:** only after **no deployed code references it**, and remember
  `SELECT *` in a view or an ORM entity still references it.
- **Changing a type:** add new column → dual-write → backfill → switch → drop. Never
  `ALTER COLUMN TYPE` in place on a hot table.
- **Beware long-running DDL behind a long transaction** — in MySQL an `ALTER TABLE` waits on a
  metadata lock and then blocks *every* subsequent query on that table, taking the site down.
  Check for long transactions before DDL and set `lock_wait_timeout`.

**Memory hook:** *Expand → Migrate/Backfill → Contract. Never make a breaking change in one
deploy; two versions of your app are always live.*

---

## 21. Security — SQL Injection & Data Protection

### Q21.1 — What is SQL injection and how do you prevent it in Spring Boot?

**Answer.** Untrusted input is concatenated into SQL and reinterpreted as code:

```java
// VULNERABLE
String sql = "SELECT * FROM users WHERE email = '" + email + "'";
// email = "x' OR '1'='1' --"  → returns every user
// email = "x'; DROP TABLE users; --"
```

**Prevention, in layers:**
1. **Parameterized queries / `PreparedStatement`** — the actual fix. Values travel outside the
   SQL text and are never parsed as SQL.
```java
jdbcTemplate.queryForObject("SELECT * FROM users WHERE email = ?", mapper, email);
@Query("SELECT u FROM User u WHERE u.email = :email")   // named parameter, safe
```
2. **Never concatenate into a JPQL/native `@Query` string** — `"... WHERE email = '" + x + "'"`
   is just as vulnerable inside JPA as in raw JDBC. `SpEL` in queries and
   `String.format` on native queries are the usual culprits.
3. **Whitelist identifiers.** Sort columns, table names and `ASC`/`DESC` cannot be bound as
   parameters — map user input through an enum/allow-list. Spring Data's `Sort` is a common
   injection vector if you pass raw strings into a native query.
4. **Use Criteria API / Specifications / QueryDSL** for dynamic filters instead of building SQL
   strings.
5. **Least privilege** — the application's database user should not own the schema or have
   `DROP`/`GRANT`; separate the migration user (DDL) from the runtime user (DML only).
6. **Defence in depth** — input validation, WAF, and **never** return raw SQL errors to the client
   (error-based injection and information leakage).
7. **`LIKE` escaping** — user input containing `%` or `_` should be escaped even in a
   parameterized `LIKE`, or you get unintended full scans/matches.

**Memory hook:** *Bind values, whitelist identifiers, least-privilege user. Parameterization is
the only real fix; everything else is defence in depth.*

---

### Q21.2 — How do you handle sensitive data in the database?

**Answer.** Encryption at rest (TDE / encrypted volumes) for the whole database; **column-level
encryption** or a JPA `AttributeConverter` for individual PII fields, with keys in a KMS/Vault —
never in the repo. Hash passwords with **bcrypt/argon2/PBKDF2 + per-user salt** (`BCryptPasswordEncoder`),
never a plain SHA. Tokenize card data to stay out of PCI scope. **TLS in transit**
(`sslmode=require`). **Row-level security** (PostgreSQL RLS) or views for multi-tenant isolation.
Mask/anonymize in non-production copies. Audit access to sensitive tables. And keep secrets out
of logs — Hibernate's `show-sql` with parameter logging will happily print PII, so disable it in
production.

Trade-off to acknowledge: an encrypted column **can't be indexed or range-queried** usefully;
for equality lookups store a separate deterministic HMAC of the value and index that.

**Glossary and how the pieces fit:**
- **TDE (Transparent Data Encryption)** — the database/disk files are encrypted, so a stolen
  disk or backup is unreadable. It does *not* protect against someone who can log in and run
  `SELECT`.
- **Column-level encryption** — only chosen columns (e.g. national ID) are encrypted, by the
  app or DB. A JPA `AttributeConverter` encrypts on write and decrypts on read automatically:
  ```java
  @Convert(converter = AesGcmConverter.class)   // encrypt(String)->String / decrypt(String)->String
  private String nationalId;
  ```
- **KMS/Vault** — a key-management service that holds the encryption keys separately from the
  data, so leaking the database alone doesn't leak the keys.
- **Salted hash (bcrypt/argon2)** — passwords are never stored or decrypted; you store a slow,
  salted one-way hash and compare hashes at login. The *salt* is random per user so identical
  passwords produce different hashes and precomputed ("rainbow table") attacks fail. Plain SHA is
  fast, which helps attackers brute-force.
- **Tokenization** — replace a card number with a random token; the real number lives in a
  specialised vault, keeping your database out of PCI-DSS (card-industry compliance) scope.
- **RLS (Row-Level Security)** — the database appends a per-user/tenant filter to every query:
  `CREATE POLICY t ON invoices USING (tenant_id = current_setting('app.tenant')::bigint);`
- **HMAC lookup column** — encryption is randomized, so `WHERE ssn = ?` can't use an index; store
  `HMAC(key, ssn)` alongside and search on that (equality only).

---
## 22. Scenario-Based Questions (the real interview)

> At 9 YOE, most of the interview is here. The pattern for answering every one of these:
> **(1) clarify/quantify, (2) diagnose with data, (3) fix in order of leverage,
> (4) prevent recurrence.** Never jump straight to "add an index."

---

### S1 — "An endpoint that returned in 200ms now takes 30 seconds. Nothing was deployed. Walk me through it."

**Answer.**

*Clarify first:* is it all requests or a subset (specific tenant / date range)? When did it
start? Is the database CPU/IO up, or is the app just waiting?

*Diagnose:*
1. Check APM/thread dumps — are threads blocked in JDBC, or in the app? If it's Hikari
   `pending` connections, it's contention, not query cost.
2. `pg_stat_activity` / `SHOW FULL PROCESSLIST` — is there a **long-running transaction** or a
   **lock wait** chain? One `idle in transaction` session can stall everything.
3. Pull the query from `pg_stat_statements` / slow log and run `EXPLAIN (ANALYZE, BUFFERS)`.
4. Compare the plan to what it *should* be. Nothing deployed ⇒ the plan changed or the data did.

*Most likely root causes, in order of real-world frequency:*
- **Data growth crossed a threshold** and the optimizer flipped from index scan to seq scan
  (or the index no longer fits in cache).
- **Stale statistics** after a bulk load → bad row estimates → bad plan. `ANALYZE` fixes it in
  seconds; also check autovacuum isn't starved.
- **Table/index bloat** (many updates/deletes, autovacuum blocked by a long transaction).
- **Lock contention** from a new batch job or a long report running concurrently.
- **A hot partition/skew** — one tenant's data grew 100×, so a plan tuned for the average tenant
  is now wrong (parameter sniffing / plan caching on a skewed value).
- **Cache cold** after a failover/restart, or the replica you're reading fell behind.
- **An index got dropped or invalidated** by a migration or a failed `CREATE INDEX CONCURRENTLY`
  (leaves an `INVALID` index in PostgreSQL).

*Fix + prevent:* correct the immediate cause, then add: slow-query alerting, plan-regression
tests on production-sized data, autovacuum/statistics monitoring, and a `timeout` on the
transaction so one bad query can't hold the pool.

**Memory hook:** *Nothing deployed ⇒ the data changed, the stats changed, or something is
holding a lock.*

---

### S2 — "Two users hit 'Book' on the last available seat at the same instant. Both get a confirmation. Fix it."

**Answer.** Classic **lost update / race condition** from read-then-write:

```java
// BROKEN
Seat s = repo.findById(id);          // both read available = true
if (s.isAvailable()) { s.setAvailable(false); repo.save(s); }   // both write
```

Four valid fixes — present them with trade-offs:

1. **Single atomic conditional UPDATE (best, no locks held across round trips):**
```sql
UPDATE seats SET booked_by = :user, available = false
WHERE id = :id AND available = true;
```
```java
if (rowsAffected == 0) throw new SeatAlreadyBookedException();
```
The database evaluates the condition and the write in one statement, so only one wins.

2. **Optimistic locking** — `@Version` on the entity; the loser gets
   `ObjectOptimisticLockingFailureException` → return 409 or retry.

3. **Pessimistic locking** — `SELECT ... FOR UPDATE` (`@Lock(PESSIMISTIC_WRITE)`) with a short
   lock timeout. Correct, but serializes bookings for that row; fine for low-contention rows,
   risky for a single popular event.

4. **Unique constraint as the ultimate arbiter** — `UNIQUE (event_id, seat_number)` on a
   `bookings` table. Even if application logic is buggy, the database refuses the duplicate. Catch
   `DataIntegrityViolationException`. **Always add this as a backstop regardless of which
   approach you choose** — that's the senior instinct: make the invariant a database constraint,
   not a code path.

For **inventory counts** rather than a single seat:
`UPDATE inventory SET qty = qty - :n WHERE sku = :sku AND qty >= :n` and check the row count.
For high-contention hot rows, shard the counter into N rows and decrement a random one.

**Memory hook:** *Never read-then-write across the network. One atomic conditional UPDATE, plus a
UNIQUE constraint as the referee.*

---

### S3 — "Transfer money between two accounts. Design it."

**Answer.**

```java
@Transactional  // isolation: READ COMMITTED is enough with the conditional update below
public void transfer(long fromId, long toId, BigDecimal amount) {
    if (amount.signum() <= 0) throw new IllegalArgumentException();

    // 1. LOCK IN A CONSISTENT ORDER to prevent deadlocks (A→B and B→A concurrently)
    long first = Math.min(fromId, toId), second = Math.max(fromId, toId);
    accountRepo.lockById(first);      // SELECT ... FOR UPDATE
    accountRepo.lockById(second);

    // 2. Conditional debit — the balance check and the write are one atomic statement
    int debited = accountRepo.debit(fromId, amount);   // ... WHERE id=? AND balance >= ?
    if (debited == 0) throw new InsufficientFundsException();

    accountRepo.credit(toId, amount);

    // 3. Double-entry ledger rows, in the same transaction
    ledgerRepo.save(new LedgerEntry(txnId, fromId, amount.negate()));
    ledgerRepo.save(new LedgerEntry(txnId, toId,   amount));
}
```

Points to make:
- **Consistent lock ordering** is the deadlock fix.
- **`balance >= amount` in the `WHERE`**, not in Java — otherwise a concurrent transfer can
  overdraw.
- **Idempotency**: the caller supplies a `transfer_id`; a `UNIQUE` index on it means a retried
  request (network timeout, Kafka redelivery) can't double-spend. Catch the duplicate-key error
  and return the original result.
- **Double-entry ledger** (append-only rows, balance derived or maintained) rather than only
  mutating a balance column — gives you auditability and reconciliation.
- **Never call an external payment API inside the transaction.** Write an outbox row, commit,
  then publish.
- **Cross-database transfer?** No 2PC — use a Saga with compensating entries and idempotent steps.

**Memory hook:** *Ordered locks, conditional debit, idempotency key, double-entry, outbox for
external calls.*

---

### S4 — "Production is down with `Lock wait timeout exceeded` / everything is blocked. What do you do?"

**Answer.**

*Immediate triage (minutes matter):*
1. Find the blocker, not the blocked. PostgreSQL:
```sql
SELECT pid, state, wait_event_type, now() - xact_start AS txn_age,
       left(query, 120), pg_blocking_pids(pid)
FROM   pg_stat_activity
WHERE  state <> 'idle'
ORDER  BY xact_start;
```
   MySQL: `performance_schema.data_lock_waits` joined to `threads`, or
   `SHOW ENGINE INNODB STATUS`.
2. Identify the **root blocker** (the session with `pg_blocking_pids` empty but others waiting).
   Typically: an `idle in transaction` session (an app that opened a transaction and then made a
   slow HTTP call, or a developer's uncommitted `BEGIN` in a GUI), an unfinished `ALTER TABLE`
   holding a metadata lock, or a giant `UPDATE`/`DELETE` without a `WHERE` on an index.
3. **Kill it** — `SELECT pg_terminate_backend(pid)` / `KILL <id>` — after checking what it is.
4. Confirm recovery; if it recurs, throttle or disable the offending job.

*Root-cause fixes:*
- Set `idle_in_transaction_session_timeout`, `statement_timeout`, `lock_wait_timeout`,
  `innodb_lock_wait_timeout` so no session can ever hang forever.
- Remove external calls from inside `@Transactional`; add `timeout` to the annotation.
- Batch large DML into chunks with commits.
- Add the missing index so `UPDATE ... WHERE` locks the rows it means to, not the whole table.
- Alert on transaction age > 30s and on lock waits.

**Memory hook:** *Find the blocker with `pg_blocking_pids` / `data_lock_waits`, kill it, then set
timeouts so it can never happen again.*

---

### S5 — "A report page loads 500 rows and fires 3,000 queries. Explain and fix."

**Answer.** N+1, twice over — 500 parents × two lazy associations, plus the initial query.

*Diagnose:* enable `spring.jpa.show-sql` in a test, or better, use **datasource-proxy /
p6spy / Hibernate statistics** to count queries per request, and add an assertion in an
integration test that this endpoint executes ≤ N queries. (Making it a *test* is the part that
impresses.)

*Fix:*
1. This is a **read-only report** → don't load entities at all. Write one SQL/JPQL query
   projecting straight into a DTO with the joins done in the database.
2. If entities are needed: `@EntityGraph`/`JOIN FETCH` for the to-one associations,
   `@BatchSize`/`default_batch_fetch_size` for collections.
3. Set `spring.jpa.open-in-view=false` so lazy loads can't silently happen during JSON
   serialization.
4. Paginate (`Slice`) — 500 rows on a page is usually a product problem too.
5. Consider a materialized view / summary table if the aggregation is heavy and near-real-time
   is acceptable.

**Memory hook:** *Reports don't need entities. Project into DTOs, one query, and assert the query
count in a test.*

---

### S6 — "You need to add a `NOT NULL` column with a default to a 200M-row table, in production, with no downtime."

**Answer.**
1. **Check your version's behaviour.** PostgreSQL 11+ and MySQL 8.0 (INSTANT algorithm) can add a
   defaulted column as a **metadata-only** change — near-instant. Older PostgreSQL rewrites the
   entire table under an `ACCESS EXCLUSIVE` lock: unacceptable.
2. **If a rewrite is required, do it in steps:**
   - `ALTER TABLE t ADD COLUMN status VARCHAR(20) NULL;` (fast, metadata only)
   - Deploy application code that **writes** the column for all new/updated rows.
   - **Backfill in batches** with commits and a sleep between them, keyed on the PK, monitoring
     replication lag: `UPDATE t SET status='X' WHERE status IS NULL AND id BETWEEN ? AND ?`.
   - Add a `CHECK (status IS NOT NULL) NOT VALID`, then `VALIDATE CONSTRAINT` (PostgreSQL takes
     only a `SHARE UPDATE EXCLUSIVE` lock for validation), then set `NOT NULL` cheaply.
   - Or for MySQL, use `gh-ost`/`pt-online-schema-change` which builds a shadow table and swaps.
3. **Guard the DDL:** set a short `lock_timeout` before the `ALTER` so it fails fast instead of
   queueing behind a long transaction and blocking every query on the table (the classic
   MySQL metadata-lock outage). Retry in a loop.
4. Do it in a low-traffic window anyway, with a tested rollback.

**Memory hook:** *Nullable column → dual write → batched backfill → validate → NOT NULL.
Always set `lock_timeout` before DDL.*

---

### S7 — "The same query is fast for most customers and times out for one. Why?"

**Answer.** **Data skew.** Most customers have 50 orders; one has 4 million.

Causes and fixes:
- **Plan caching / parameter sniffing** — the cached plan was built for a typical parameter and
  uses a nested loop that's disastrous for the outlier. Fixes: `OPTION (RECOMPILE)` in SQL Server,
  `plan_cache_mode = force_custom_plan` in PostgreSQL, or restructure so the query is
  selective regardless (e.g. always bound by date).
- **Missing composite index** so the query relies on the customer filter alone being selective.
  Add `(customer_id, created_at DESC)` so even the big customer gets a bounded index range.
- **No upper bound on the result** — enforce pagination/date limits at the API level; a query
  that can return 4M rows will eventually be asked to.
- **Histogram/statistics** don't capture the skew — increase the statistics target on that column
  so the optimizer knows the value is a 4M-row outlier.
- **Long term:** partition by customer/date, or move that tenant's heavy reporting to a
  replica/warehouse.

**Memory hook:** *"Fast for most, dead for one" = skew + a cached plan built for the average.*

---

### S8 — "Design the schema for an e-commerce order system, then tell me the indexes."

**Answer.**

```sql
customers(id PK, email UNIQUE, name, created_at)
addresses(id PK, customer_id FK, line1, city, pincode, type)
products(id PK, sku UNIQUE, name, active)
product_prices(id PK, product_id FK, price NUMERIC(12,2), valid_from, valid_to)  -- price history
inventory(product_id PK/FK, qty_available, qty_reserved, version)                -- @Version
orders(id PK, customer_id FK, status, order_date, total_amount NUMERIC(12,2),
       shipping_address_snapshot JSONB, idempotency_key UNIQUE)
order_items(id PK, order_id FK, product_id FK, qty,
            unit_price NUMERIC(12,2),        -- SNAPSHOT of price at purchase time
            UNIQUE(order_id, product_id))
payments(id PK, order_id FK, status, amount, provider_ref UNIQUE)
order_status_history(id PK, order_id FK, from_status, to_status, changed_at, changed_by)
```

Design decisions to call out — these are the marks:
- **Price and address are snapshotted** onto the order, not joined live. An invoice must not change
  when the product price changes. (Temporal correctness, see Q9.2.)
- **`idempotency_key UNIQUE`** on orders so a retried checkout can't create two orders.
- **`NUMERIC`, never `FLOAT`,** for money; store the currency too if multi-currency.
- **Status history table** instead of just a mutable `status` column, for audit and analytics.
- **Inventory with `@Version`** or a conditional decrement, plus a `qty_reserved` concept so a
  cart can hold stock without committing it.

Indexes, driven by the actual access patterns:
```sql
CREATE INDEX ON orders (customer_id, order_date DESC);      -- "my orders", newest first
CREATE INDEX ON orders (status, order_date)                 -- ops dashboard for open orders
       WHERE status IN ('PENDING','PAID');                  -- partial: small & hot
CREATE INDEX ON order_items (order_id);                     -- FK lookup (Postgres won't auto-create)
CREATE INDEX ON order_items (product_id);                   -- "who bought this product"
CREATE INDEX ON payments (order_id);
-- products(sku) and customers(email) already covered by UNIQUE
```
Then: partition `orders` by `order_date` (monthly) once it exceeds a few hundred million rows,
so retention is a `DROP PARTITION` and dashboards prune to recent months.

---

### S9 — "How do you find and delete duplicate rows, keeping the earliest?"

```sql
-- 1. FIND duplicates
SELECT email, COUNT(*) AS cnt, MIN(id) AS keep_id
FROM   users
GROUP  BY email
HAVING COUNT(*) > 1;

-- 2. DELETE, keeping the lowest id (portable, window-function version)
DELETE FROM users
WHERE id IN (
  SELECT id FROM (
    SELECT id, ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_at, id) AS rn
    FROM users
  ) t
  WHERE rn > 1
);

-- PostgreSQL, using the physical row id — fast and neat
DELETE FROM users a USING users b
WHERE a.email = b.email AND a.ctid > b.ctid;

-- MySQL note: you cannot select from the table you're deleting from directly,
-- so wrap the subquery in a derived table (as above) — that extra nesting is the trick.
```

**Then prevent recurrence:** `CREATE UNIQUE INDEX ON users (lower(email));` — and be ready for
the follow-up *"how do you add a unique index to a table that currently has duplicates, live?"*
→ clean in batches first, then `CREATE UNIQUE INDEX CONCURRENTLY`, then validate.

Also mention: dedupe on a **case/whitespace-normalized** key, or you'll "fix" the data and still
have `A@b.com` and `a@b.com`.

**Memory hook:** *`ROW_NUMBER() OVER (PARTITION BY dup_key ORDER BY keep_rule)`, delete `rn > 1`,
then add the unique index so it can't happen again.*

---

### S10 — "How do you audit every change to a table?"

**Answer.** Four options, with trade-offs:

| Approach | Captures manual SQL? | Effort | Notes |
|---|---|---|---|
| **JPA auditing** (`@CreatedDate`, `@LastModifiedBy`, `AuditingEntityListener`) | No | Tiny | Only who/when on the row itself, no history |
| **Hibernate Envers** (`@Audited`) | No | Low | Auto-creates `_AUD` tables + revision entity; queryable history, works with `@Transactional`; bypassed by bulk JPQL/native DML |
| **Database triggers** → audit table | **Yes** | Medium | The only option if the requirement is "nothing escapes"; invisible to app devs, extends lock time |
| **CDC** (Debezium reading the WAL/binlog) | **Yes** | Higher infra | Zero write overhead on the transaction, streams to Kafka/warehouse, best for analytics and event sourcing |

Schema for a generic audit table: `(id, table_name, row_pk, operation, changed_by, changed_at,
old_values JSONB, new_values JSONB, txn_id)`. Add the **user identity** — the DB user is the
connection-pool user, so the application must propagate the real user
(`SET LOCAL app.current_user = ?` per transaction, read by the trigger).

**Interview line:** *"Envers if the application is the only writer; triggers or CDC if the
compliance requirement is that no change may escape, including DBA hotfixes."*

---

### S11 — "How would you migrate a 500GB table to a new structure with zero downtime?"

**Answer.** Dual-write + backfill + verify + cutover:
1. Create the new table/structure.
2. **Dual write** from the application (or a trigger, or CDC) so both stay current.
3. **Backfill historical data in batches**, throttled, monitoring replication lag and IO.
4. **Verify**: row counts, checksums per id range, and a shadow-read phase where you read from
   both and compare/log differences without serving the new result.
5. **Cutover reads** behind a feature flag, per-tenant or percentage-based, so rollback is
   instant.
6. **Stop dual writes**, drop the old table after a retention period.
Keep every step independently reversible, and never delete the old data in the same release
as the cutover.

**Why each step exists:**
- *Dual write* guarantees changes made **during** the long backfill aren't lost — otherwise the
  new table is stale the moment the copy finishes. Prefer a single mechanism (CDC or
  application) so the two writes can't disagree; if the app does both writes, do them in the
  same transaction when both tables are in one database.
- *Backfill throttling* (e.g. 10k rows per batch, sleep between, keyed by primary-key ranges)
  keeps IO and replication lag low so production traffic isn't starved. Make it idempotent
  (`INSERT ... ON CONFLICT DO NOTHING`/upsert) so a rerun is safe.
- *Verification* catches silent bugs: compare `COUNT(*)` and a hash per id range
  (`md5(string_agg(...))`), and run shadow reads that compare old vs new results.
- *Feature-flagged cutover* (1% → 10% → 100%) turns a risky big-bang into something you can
  undo by flipping a flag. Keep dual-writing until you're confident, so rollback loses nothing.

**Memory hook:** *Dual write → backfill → verify → flagged cutover → decommission.*

---

### S12 — "Should this go in the database, the cache, or the application? Give me your reasoning."

**Answer.** Say the decision criteria out loud:
- **Database:** anything that must be **correct under concurrency** — uniqueness, referential
  integrity, monetary invariants, atomic decrements. The database is the only place with real
  transactions and constraints. *"If correctness depends on it, it belongs in the database."*
- **Cache (Redis/Caffeine):** expensive, read-heavy, tolerant of staleness — reference data,
  computed aggregates, session data. Must have a defined TTL and invalidation strategy;
  cache-aside is the default pattern; watch for stampedes (use a lock or probabilistic early
  refresh).
- **Application:** orchestration, business rules that change often, anything needing to be
  unit-tested and code-reviewed easily.

Anti-patterns to name: enforcing uniqueness with a `SELECT` in Java (race), caching data that
must be transactional, and putting business logic in triggers where no one can find it.

**Worked examples of the reasoning:**
- *"Username must be unique"* → database (`UNIQUE` index). Two simultaneous sign-ups both pass a
  Java `existsByUsername` check; only the constraint is race-proof.
- *"Product catalogue shown on every page"* → cache. It changes rarely and a few minutes of
  staleness is acceptable; e.g. Spring's `@Cacheable("products")` with a 5-minute TTL, evicted on edit.
- *"Account balance"* → database, never cache — a stale balance means a wrong decision.
- *"Discount rules that marketing changes weekly"* → application (code or config), where it can be
  reviewed, tested and deployed, not buried in a stored procedure.

**Cache-aside, defined:** the app checks the cache first; on a miss it reads the database, stores
the result in the cache with a TTL, and returns it. On writes it updates the database and then
evicts/updates the cache entry. A *stampede* is when a hot key expires and hundreds of requests
hit the database at once to rebuild it.

---

### S13 — "How do you decide between adding an index and rewriting the query?"

**Answer.** Rewrite first if the query is **non-sargable**, returns more data than needed, or has
a structural problem (`OR` across columns, `SELECT *`, deep `OFFSET`, correlated subquery per
row) — no index rescues a query that wraps the indexed column in a function.
Add an index when the query is already well formed and the plan shows a scan over a large table
with a selective predicate. Then always ask: **what does this index cost?** Write amplification,
disk, and maintenance. Check `pg_stat_user_indexes.idx_scan` / `sys.schema_unused_indexes` a week
later and drop what isn't used. Consolidate rather than accumulate — extend an existing
composite index if the leftmost prefix already matches.

**Example:** `SELECT * FROM orders WHERE DATE(created_at) = '2026-03-01'` is slow even with an index
on `created_at`, because `DATE(created_at)` wraps the column in a function (non-sargable, Q10.5).
Rewriting to `WHERE created_at >= '2026-03-01' AND created_at < '2026-03-02'` makes the existing
index usable — zero new indexes. Compare: `WHERE customer_id = 42 AND status = 'OPEN'` is already
well-formed but scans the table because only `customer_id` is indexed → extending the index to
`(customer_id, status)` is the right fix. The decision question is *"is the query shaped so an
index could help?"* — if no, rewrite; if yes and no suitable index exists, add one.

**Memory hook:** *Fix the query's shape first; index second; then audit and drop unused indexes.*

---

### S14 — "Your service publishes an event to Kafka after saving to the DB. Sometimes the event is lost, sometimes it's published but the DB rolled back. Fix it."

**Answer.** This is the **dual-write problem** — you cannot atomically commit to two systems
without a distributed transaction.

**Transactional Outbox:**
1. In **one local transaction**, insert the business row **and** an `outbox(id, aggregate_id,
   type, payload, created_at, published_at NULL)` row.
2. A separate poller (`SELECT ... WHERE published_at IS NULL FOR UPDATE SKIP LOCKED LIMIT 100`)
   or **CDC via Debezium** reads the outbox and publishes to Kafka, marking rows published.
3. This gives **at-least-once** delivery → **consumers must be idempotent**
   (a `processed_events(event_id)` table with a unique key, or an idempotent upsert).
4. Ordering: partition Kafka by `aggregate_id` and publish outbox rows in id order per aggregate.

Also mention what *not* to do: `@TransactionalEventListener(phase = AFTER_COMMIT)` improves
things (it won't publish on rollback) but still loses the event if the process dies right after
commit — so it's better than a naive publish, but only the outbox is durable.

**Memory hook:** *Two systems, one transaction = impossible. Outbox row in the same commit +
idempotent consumer.*

---

### S15 — "A nightly batch job has started overrunning into business hours and slowing the site."

**Answer.**
1. **Measure** where the time goes: per-step timings, rows processed/sec, and whether it's
   database-bound (locks/IO) or app-bound.
2. **Common causes:** row-by-row processing (N round trips), no batching, a growing table with an
   O(n²) query, giant transactions holding locks and bloating undo/WAL, no index on the "unprocessed"
   predicate, and contention with online traffic.
3. **Fixes:**
   - Convert row-at-a-time logic into **set-based SQL** (`INSERT ... SELECT`, `UPDATE ... FROM`) —
     usually the 10–100× win.
   - **Chunk with commits** (e.g. 5,000 rows) so locks are short and progress is resumable;
     track a high-water mark so a restart doesn't redo everything.
   - Index the driving predicate (`WHERE processed = false` → **partial index**).
   - **Throttle** — sleep between chunks, cap concurrency, and watch replication lag.
   - Run heavy **reads on a replica**.
   - Make it **idempotent and restartable** so it can be killed at 7am safely.
   - Long term: incremental/CDC processing instead of full re-scan, or move analytics to a
     warehouse.
4. **Guardrails:** a hard deadline that stops the job at 06:00, alerting on duration, and
   `statement_timeout` so a single query can't run away.

**Memory hook:** *Set-based, chunked, resumable, throttled, deadlined.*

---

### S16 — "How would you find the queries hurting you most, without an APM?"

```sql
-- PostgreSQL (requires the pg_stat_statements extension)
SELECT calls, round(total_exec_time)  AS total_ms,
       round(mean_exec_time, 2)       AS mean_ms,
       rows, query
FROM   pg_stat_statements
ORDER  BY total_exec_time DESC        -- total, not mean: 1ms × 10M calls beats 5s × 3 calls
LIMIT  20;

-- Missing/unused indexes
SELECT relname, seq_scan, seq_tup_read, idx_scan FROM pg_stat_user_tables ORDER BY seq_tup_read DESC;
SELECT relname, indexrelname, idx_scan FROM pg_stat_user_indexes WHERE idx_scan = 0;

-- MySQL
SELECT digest_text, count_star, round(sum_timer_wait/1e12,2) AS total_s,
       round(avg_timer_wait/1e9,2) AS avg_ms, sum_rows_examined/count_star AS avg_examined
FROM   performance_schema.events_statements_summary_by_digest
ORDER  BY sum_timer_wait DESC LIMIT 20;
-- plus the slow query log with long_query_time=1, and sys.schema_unused_indexes
```

**The insight to state:** sort by **total** time, not average. The endpoint called 10 million
times at 20ms is a bigger problem than the nightly report that takes 8 seconds. Also watch
`rows_examined / rows_sent` — a high ratio means the index isn't doing its job.

---

### S17 — "Multi-tenant SaaS: shared table, schema per tenant, or database per tenant?"

| Model | Isolation | Cost / density | Ops | When |
|---|---|---|---|---|
| **Shared table + `tenant_id`** | Weakest (a missing `WHERE` leaks data) | Best | Simplest migrations (one schema) | Many small tenants, SaaS default |
| **Schema per tenant** | Medium | Good | Migrations × N schemas; connection routing | Tens–hundreds of tenants, some isolation needed |
| **Database per tenant** | Strongest | Worst | N databases to back up/migrate/monitor | Enterprise/regulated customers, noisy-neighbour risk |

For the shared model, the safety mechanisms are the answer: `tenant_id` as the **leading column
of every index and every PK**, PostgreSQL **row-level security** as a backstop, a Hibernate filter
or `@TenantId` (Hibernate 6) so no query can omit it, and a `TenantContext` bound to the request
(cleared in a filter — a `ThreadLocal` leak across pooled threads is a real data-leak bug).
Mention **hybrid**: shared by default, dedicated database for the top few enterprise customers.

---

### S18 — "You're told to move a heavily-used query from the primary to a read replica. What could go wrong?"

**Answer.** Replication lag (stale reads, read-your-writes violations — see Q15.2), a replica
with different hardware/cache warmth giving different plans, long-running queries on a replica
**blocking replay** (PostgreSQL: conflict with recovery, `max_standby_streaming_delay`, or bloat
from `hot_standby_feedback`), no writes allowed (so any accidental write, including a Hibernate
flush of a dirtied entity, fails), sequences/`nextval` unavailable, and failover semantics — if
the replica is promoted, is your routing still correct? Mitigation: explicit `readOnly = true`
transactions, a lag-aware router, a "must be fresh" bypass to the primary, and monitoring.

**Unpacking the PostgreSQL replay conflict:** a replica continuously *replays* the primary's
changes. If a long `SELECT` on the replica needs old row versions that the primary has already
vacuumed away, replay must either wait (query keeps running, replica falls further behind) or
cancel your query (`ERROR: canceling statement due to conflict with recovery`). Settings trade
this off: `max_standby_streaming_delay` (how long replay may wait) and `hot_standby_feedback`
(replica tells the primary "don't vacuum rows I still need" — fewer cancellations, but the
primary can bloat). Typical symptom you can recognise: reports on the replica randomly fail with
that "conflict with recovery" error. Also, anything that quietly writes — a Hibernate flush from a
dirtied entity, `nextval()` on a sequence — throws `cannot execute ... in a read-only transaction`;
that's why `readOnly = true` plus no entity mutation matters.

---

## 23. Query-Writing Puzzles You Must Be Able to Type Instantly

### P1 — Second-highest salary (and Nth)

```sql
-- Window function (best; handles ties explicitly, works for any N)
SELECT DISTINCT salary FROM (
  SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS dr FROM employees
) t WHERE dr = 2;

-- Correlated subquery (the classic "clever" answer)
SELECT MAX(salary) FROM employees WHERE salary < (SELECT MAX(salary) FROM employees);

-- LIMIT/OFFSET (careful: this returns the 2nd ROW, not the 2nd distinct salary)
SELECT DISTINCT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET 1;
```
**Say this out loud:** the interviewer is checking whether you handle **ties** and the **NULL
case when there is no second salary** (the `MAX` version returns `NULL`, `LIMIT` returns no
rows — wrap in a subquery if you need `NULL`).

---

### P2 — Top N per group (top 3 earners per department)

```sql
WITH r AS (
  SELECT e.*, DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dr
  FROM   employees e
)
SELECT department_id, name, salary FROM r WHERE dr <= 3 ORDER BY department_id, salary DESC;
```
Mention the `LATERAL`/`CROSS APPLY` alternative (Q6.5) as the faster option when the group count
is small and the per-group table is huge and indexed.

---

### P3 — Employees earning more than their manager

```sql
SELECT e.name AS employee, e.salary, m.name AS manager, m.salary AS mgr_salary
FROM   employees e
JOIN   employees m ON e.manager_id = m.id
WHERE  e.salary > m.salary;
```
**How it works:** the same `employees` table plays two roles, so alias it twice — `e` is the
employee row and `m` is the *manager's* row, found by matching `e.manager_id` to `m.id`. Then it's
an ordinary row filter comparing two columns. Use an `INNER JOIN` here: the CEO (no manager,
`manager_id IS NULL`) has nothing to compare with and should drop out anyway.

---

### P4 — Departments with no employees (and the reverse)

```sql
SELECT d.* FROM departments d
WHERE NOT EXISTS (SELECT 1 FROM employees e WHERE e.department_id = d.id);
```
**How it works:** for each department, the subquery asks "is there at least one employee pointing
here?"; `NOT EXISTS` keeps the department only when the answer is no. (`SELECT 1` is just a
placeholder — `EXISTS` only cares whether any row comes back.) This is the anti-join from Q2.3.

**The reverse** — employees whose department is missing or invalid (e.g. `department_id` is NULL
or points at a deleted department):
```sql
SELECT e.* FROM employees e
LEFT JOIN departments d ON d.id = e.department_id
WHERE d.id IS NULL;
```

---

### P5 — Running total, moving average, and percent of total

```sql
SELECT
  order_date, amount,
  SUM(amount) OVER (ORDER BY order_date ROWS UNBOUNDED PRECEDING)              AS running_total,
  AVG(amount) OVER (ORDER BY order_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma7,
  ROUND(100.0 * amount / SUM(amount) OVER (), 2)                               AS pct_of_total,
  ROUND(100.0 * amount / SUM(amount) OVER (PARTITION BY region), 2)            AS pct_of_region
FROM sales;
```
Remember Q7.4: use **`ROWS`**, not the default `RANGE`, or duplicate dates lump together.

---

### P6 — Month-over-month growth

```sql
WITH m AS (
  SELECT DATE_TRUNC('month', order_date) AS mon, SUM(amount) AS revenue
  FROM sales GROUP BY 1
)
SELECT mon, revenue,
       LAG(revenue) OVER (ORDER BY mon) AS prev_revenue,
       ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY mon))
             / NULLIF(LAG(revenue) OVER (ORDER BY mon), 0), 2) AS mom_pct
FROM m ORDER BY mon;
```
**How it works:** first collapse daily rows into one revenue number per month (`DATE_TRUNC` snaps
any date to the 1st of its month). Then `LAG(revenue)` pulls the *previous month's* revenue onto
the current row, so growth % = (this − last) ÷ last × 100. The first month has no previous row,
so `LAG` gives `NULL` and the percentage is `NULL` — correct, not a bug. `NULLIF(..., 0)` avoids
a divide-by-zero when last month's revenue was 0. Caveat: if a month has no sales at all it won't
appear in `m`, so `LAG` will compare against the wrong month — join to a month series (P13) first.

---

### P7 — Pivot: monthly sales as columns

```sql
SELECT product_id,
       SUM(CASE WHEN EXTRACT(MONTH FROM sale_date) = 1 THEN amount ELSE 0 END) AS jan,
       SUM(CASE WHEN EXTRACT(MONTH FROM sale_date) = 2 THEN amount ELSE 0 END) AS feb,
       SUM(CASE WHEN EXTRACT(MONTH FROM sale_date) = 3 THEN amount ELSE 0 END) AS mar
FROM   sales WHERE sale_date >= DATE '2026-01-01'
GROUP  BY product_id;
```
Note: SQL has no dynamic pivot — column lists must be known at query time (or generated).

**How it works:** "pivot" = turn row values (months) into column headers. For each output column,
the `CASE` inside `SUM` contributes `amount` only for rows of that month and `0` otherwise, so
each column sums just its own month. `GROUP BY product_id` gives one row per product. Input
`(P1, Jan, 10), (P1, Feb, 5), (P1, Jan, 3)` → output `P1 | jan=13 | feb=5 | mar=0`. SQL Server
and Oracle also have a `PIVOT` keyword, but this `CASE` version works everywhere (Q4.3).

---

### P8 — Median (and percentiles)

```sql
-- Standard SQL (PostgreSQL, Oracle, SQL Server via WITHIN GROUP where supported)
SELECT department_id,
       PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_interpolated,
       PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY salary) AS median_actual_value
FROM   employees GROUP BY department_id;

-- Portable / MySQL (no percentile functions before 8.0 window tricks)
WITH r AS (
  SELECT salary,
         ROW_NUMBER() OVER (ORDER BY salary) AS rn,
         COUNT(*)     OVER ()                AS cnt
  FROM employees
)
SELECT AVG(salary) AS median FROM r WHERE rn IN ((cnt+1)/2, (cnt+2)/2);
```
`CONT` interpolates between the two middle values; `DISC` returns an actual value from the set —
knowing that distinction is the point of the question.

---

### P9 — Gaps and islands: consecutive login streaks

```sql
-- The trick: (row_number - the date) is CONSTANT for consecutive dates → group by it
WITH d AS (
  SELECT DISTINCT user_id, login_date FROM logins
), g AS (
  SELECT user_id, login_date,
         login_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date))::int AS grp
  FROM d
)
SELECT user_id, MIN(login_date) AS streak_start, MAX(login_date) AS streak_end, COUNT(*) AS days
FROM   g
GROUP  BY user_id, grp
HAVING COUNT(*) >= 3
ORDER  BY user_id, streak_start;
```
**Why the trick works — worked example** for one user:

| login_date | row_number | date − row_number (`grp`) |
|---|---|---|
| Jan 1 | 1 | Dec 31 |
| Jan 2 | 2 | Dec 31 |
| Jan 3 | 3 | Dec 31 |
| Jan 7 | 4 | Jan 3 |
| Jan 8 | 5 | Jan 3 |

While dates go up by 1 each row, `row_number` also goes up by 1, so their difference stays the
same. A gap in dates (Jan 3 → Jan 7) makes the date jump more than the row number, so `grp`
changes. Grouping by `grp` therefore yields one group per unbroken streak (here: Jan 1–3 = 3 days,
Jan 7–8 = 2 days). `DISTINCT` in the first CTE matters: two logins on the same day would break
the arithmetic. (Date subtraction syntax is PostgreSQL; in MySQL use `DATE_SUB(login_date,
INTERVAL rn DAY)`.)

**Memory hook:** *date − row_number = a constant "island id". This one trick answers every
"consecutive"/"streak"/"gap" question.*

---

### P10 — Find missing sequence numbers / gaps in IDs

```sql
SELECT id + 1 AS gap_start, next_id - 1 AS gap_end
FROM (SELECT id, LEAD(id) OVER (ORDER BY id) AS next_id FROM invoices) t
WHERE next_id - id > 1;
```
**How it works:** `LEAD(id)` puts the next id onto each row. If ids are 1, 2, 5, 6, then the row
for `2` sees `next_id = 5`; `5 − 2 = 3 > 1` means something is missing, and the missing numbers
run from `id + 1 = 3` to `next_id − 1 = 4`. Consecutive rows (difference 1) are ignored. Limits:
it finds gaps *between* existing ids, not before the first or after the last; and note that
gaps in auto-increment ids are normal (rolled-back inserts consume sequence values), so only
treat them as bugs for legally-sequential numbers such as invoice numbers.

---

### P11 — Customers who bought ALL products in a category (relational division)

```sql
SELECT o.customer_id
FROM   order_items oi
JOIN   orders o  ON o.id = oi.order_id
JOIN   products p ON p.id = oi.product_id
WHERE  p.category = 'LAPTOP'
GROUP  BY o.customer_id
HAVING COUNT(DISTINCT oi.product_id) =
       (SELECT COUNT(*) FROM products WHERE category = 'LAPTOP');
```
**Memory hook:** *"Bought all X" = `COUNT(DISTINCT ...) = (total count of X)`.*

---

### P12 — Duplicate detection on a composite key + first/last event per entity

```sql
-- Latest status row per order (very common in real work)
SELECT * FROM (
  SELECT h.*, ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY changed_at DESC, id DESC) AS rn
  FROM order_status_history h
) t WHERE rn = 1;
```
The `id DESC` tiebreaker matters — without it, two rows with the same timestamp make the result
non-deterministic.

---

### P13 — Fill report gaps with a generated date series

```sql
-- PostgreSQL
SELECT d.day, COALESCE(SUM(s.amount), 0) AS revenue
FROM   generate_series(DATE '2026-01-01', DATE '2026-01-31', INTERVAL '1 day') AS d(day)
LEFT   JOIN sales s ON s.sale_date = d.day
GROUP  BY d.day ORDER BY d.day;
-- MySQL: recursive CTE to generate the series, or a permanent calendar/dim_date table
```
**Memory hook:** *Days with zero sales don't exist in the fact table — LEFT JOIN a date series
(or a `dim_date` table) so the report doesn't silently skip them.*

---

### P14 — Self-join for pairs (departments with equal average salary, no duplicate pairs)

```sql
WITH a AS (SELECT department_id, AVG(salary) avg_sal FROM employees GROUP BY department_id)
SELECT a1.department_id, a2.department_id, a1.avg_sal
FROM   a a1 JOIN a a2
       ON a1.avg_sal = a2.avg_sal
      AND a1.department_id < a2.department_id;    -- '<' avoids self-pairs and mirror duplicates
```
**Memory hook:** *Pair queries: join the table to itself with `a.id < b.id`.*

---

## 24. Rapid-Fire One-Liners

- **`DISTINCT` vs `GROUP BY`?** Same result for plain dedupe and usually the same plan; `GROUP BY`
  additionally supports aggregates and `HAVING`. E.g. `SELECT DISTINCT city FROM users` and
  `SELECT city FROM users GROUP BY city` are equivalent; only `GROUP BY` can add `COUNT(*)` per city.
- **`CHAR(3)` = `'IN '` vs `'IN'`?** Comparison ignores trailing spaces in most engines, but
  `LENGTH()` doesn't — a real bug source. *Why:* `CHAR` pads with spaces, and the standard says
  padded comparison ignores them, but functions measuring length see the stored value (behaviour
  varies by vendor — test on yours).
- **`COALESCE` vs `ISNULL`/`NVL`?** `COALESCE` is standard and n-ary; the others are vendor-specific
  and two-arg. `COALESCE`'s return type is the highest-precedence argument type.
- **`UNION` column names?** Taken from the first `SELECT`.
- **`ORDER BY 1, 2`?** Ordinal positions. Fine in ad-hoc queries, fragile in application code.
- **Can you `ORDER BY` a column not in `SELECT`?** Yes — unless `DISTINCT` or a set operator is
  involved.
- **`TRUNCATE` inside `@Transactional`?** MySQL/Oracle implicitly commit (DDL) — your rollback
  won't undo it. PostgreSQL and SQL Server can roll it back.
- **Can you index a `NULL`?** PostgreSQL B-trees index NULLs (so `IS NULL` can use an index);
  Oracle B-trees do **not** store all-NULL keys — hence `IS NULL` can't use a single-column index
  there. *Workaround in Oracle:* make the index composite with a constant
  (`CREATE INDEX i ON t (col, 0)`) so entries are never entirely NULL.
- **`WHERE 1=1`?** A no-op used to simplify dynamically appended `AND` clauses. Harmless; the
  optimizer removes it. But prefer Criteria/QueryDSL over string building.
- **Difference between `DROP`, `DELETE`, `TRUNCATE` privileges?** DDL vs DML — a least-privilege
  runtime user should have neither `DROP` nor `TRUNCATE`.
- **What's a "wide" vs "long" table?** Wide = many columns (pivoted); long = key/value rows
  (EAV). EAV is flexible but destroys type safety, constraints and query performance — prefer
  `JSONB` with a GIN index if you truly need schemaless attributes.
- **`JSON` vs `JSONB` (PostgreSQL)?** `JSON` stores text verbatim; `JSONB` stores a parsed binary
  form — indexable (GIN), faster to query, loses key order and duplicate keys. Use `JSONB`.
- **Should you store enums as `VARCHAR` or `int`?** `VARCHAR` (`@Enumerated(EnumType.STRING)`)
  — readable in the database and immune to reordering the Java enum. **Never use
  `EnumType.ORDINAL`**; inserting a new enum constant in the middle silently corrupts data.
  *Example:* `enum Status {NEW, PAID}` stores 0 and 1; add `PENDING` between them and every
  existing `PAID` row (stored as 1) now reads back as `PENDING`.
- **`java.util.Date` vs `LocalDateTime` vs `Instant`?** Use `java.time`: `LocalDate` ↔ `DATE`,
  `LocalDateTime` ↔ `TIMESTAMP`, `Instant`/`OffsetDateTime` ↔ `TIMESTAMPTZ`. **Store UTC**
  (`TIMESTAMP WITH TIME ZONE`), convert at the edge, and set the JVM/session timezone explicitly
  (`spring.jpa.properties.hibernate.jdbc.time_zone=UTC`) — otherwise timestamps shift on
  deployment to a differently-configured server.
- **Why not `Serializable` isolation everywhere?** Throughput collapses and PostgreSQL will abort
  transactions with `40001` that you must retry. *Why:* SERIALIZABLE makes concurrent
  transactions behave as if run one-at-a-time, so the database must block or abort anything that
  could conflict; most of your reads don't need that guarantee, so you'd pay for it needlessly.
- **What is a "hot row"?** A single row updated by many concurrent transactions (a counter,
  a leaderboard, an account) — serializes everything. Fix: shard the counter, or aggregate
  asynchronously. *Why:* every writer needs the row's exclusive lock, so they queue one by one
  no matter how many app servers you add. *Sharded counter:* keep N rows
  (`counter_id, slot 0..9, value`), update a random slot, and `SUM` them when reading.
- **What is connection "idle in transaction"?** A borrowed connection inside an open transaction
  doing nothing — the number-one cause of lock pile-ups and vacuum starvation. *How it happens:*
  code starts a transaction (`BEGIN`), runs a statement, then waits on something else (an HTTP
  call, user input, a forgotten commit). It still holds its locks and snapshot the whole time.
- **`@Transactional` on a `@Repository` method plus one on the service — which isolation wins?**
  The outer one; `REQUIRED` joins the existing transaction and **the inner isolation/timeout
  attributes are ignored** (Spring throws if you set a conflicting isolation with
  `validateExistingTransaction=true`).
- **Does `@Transactional(readOnly = true)` prevent writes?** It's a hint; on PostgreSQL the
  transaction really is read-only, but don't rely on it as a security control.

---

## 25. Cheat Sheets

### 25.1 Isolation levels ↔ anomalies

```
                    Dirty  Non-repeat  Phantom  WriteSkew
READ UNCOMMITTED      ✗         ✗          ✗        ✗
READ COMMITTED        ✓         ✗          ✗        ✗      <- Postgres/Oracle/SQLServer default
REPEATABLE READ       ✓         ✓          ✗*       ✗      <- MySQL default (*InnoDB prevents it)
SERIALIZABLE          ✓         ✓          ✓        ✓
(✓ = prevented)
```

*How to read it:* each row down the table prevents one more problem than the row above. Pick the
**lowest** level that still protects your invariant — higher levels cost throughput. Example: a
balance-display query is fine at READ COMMITTED; "check two doctors on call, then go off call"
needs SERIALIZABLE (or an explicit lock) because of write skew. The `*` means the SQL standard
allows phantoms at REPEATABLE READ, but InnoDB's gap locks block them in practice.

### 25.2 Index decision tree

```
Is the predicate sargable?            no  -> rewrite the query (remove functions/casts/leading %)
  yes
Does an index's leftmost prefix match? no -> create/extend a composite index (Equality, Sort, Range)
  yes
Does the query need extra columns?    yes -> make it covering (add columns or INCLUDE)
  no
Is the predicate selective (<~10%)?    no -> a scan is correct; consider pre-aggregation/partitioning
  yes
Are statistics fresh?                  no -> ANALYZE / extended statistics
  yes -> you're done; verify with EXPLAIN ANALYZE and check write cost
```

*How to use it:* walk top-to-bottom for a slow query and stop at the first "no". Example:
`WHERE LOWER(email) = ?` fails the first test (function on the column) → rewrite or add an
expression index before anything else. A query returning 40% of the table fails the selectivity
test → no index will help, so a scan is the correct plan. "Sargable" = the predicate is written
so an index seek is possible (column left bare; Q10.5).

### 25.3 Spring `@Transactional` failure modes

| Symptom | Cause | Fix |
|---|---|---|
| Annotation has no effect | Self-invocation, or non-public/`final` method | Separate bean / `TransactionTemplate` |
| No rollback on a checked exception | Spring's default | `rollbackFor = Exception.class` |
| `UnexpectedRollbackException` | Inner `REQUIRED` failed and was swallowed | Don't swallow, or `REQUIRES_NEW` |
| `LazyInitializationException` | Proxy accessed after commit | `JOIN FETCH` / `@EntityGraph` / DTO |
| Pool exhausted | Long transaction or `REQUIRES_NEW` nesting | Shorten; remove external calls |
| Data saved without `save()` | Dirty checking on a managed entity | Expected — that's JPA |
| Stale data after a bulk JPQL update | Bulk DML bypasses the persistence context | `clearAutomatically`/`flushAutomatically`, or clear the context |

### 25.4 Vendor differences worth naming in an interview

| Topic | MySQL/InnoDB | PostgreSQL | Oracle | SQL Server |
|---|---|---|---|---|
| Default isolation | REPEATABLE READ | READ COMMITTED | READ COMMITTED | READ COMMITTED |
| Clustered PK | Yes (always) | No (heap) | No (unless IOT) | Optional |
| MVCC storage | Undo log | Heap + VACUUM | Undo segments | tempdb version store (with RCSI) |
| Materialized views | No | Yes | Yes (fast refresh) | Indexed views |
| Auto-index on FK | Yes | No | No | No |
| `FULL OUTER JOIN` | No | Yes | Yes | Yes |
| `LATERAL` | No | Yes | Yes (12c+) | `CROSS APPLY` |
| Sequence-friendly IDs | `AUTO_INCREMENT` | `SEQUENCE`/`IDENTITY` | `SEQUENCE` | `IDENTITY`/`SEQUENCE` |
| Upsert | `ON DUPLICATE KEY UPDATE` | `ON CONFLICT` | `MERGE` | `MERGE` |

### 25.5 Twelve things to say that signal seniority

1. "Let me confirm the access patterns before I design the indexes."
2. "That's a read-then-write race — I'd make it one atomic conditional `UPDATE`."
3. "I'd put that invariant in a database constraint, not only in the service layer."
4. "`EAGER` isn't the fix for N+1 — lazy everywhere, fetch per use case."
5. "How big is this table going to get in two years?"
6. "`OFFSET` won't scale past a few thousand pages; keyset pagination will."
7. "No network calls inside a transaction."
8. "This needs an idempotency key so retries can't double-charge."
9. "Two systems can't share one transaction — outbox plus an idempotent consumer."
10. "Every migration has to be backward compatible; two app versions run during a rolling deploy."
11. "Let me look at the estimate-versus-actual rows in the plan before changing anything."
12. "Snapshot the price on the order — an invoice must not change when the catalogue does."

---

### Final revision plan (the night before)

- **Must be automatic:** window functions (§7), join `ON` vs `WHERE` (§2.2), leftmost prefix (§10.3),
  isolation levels + the two defaults (§12.3), N+1 and its fixes (§16.2), `@Transactional` gotchas (§17.1).
- **Must have a story for:** a slow query you fixed, a deadlock you diagnosed, a zero-downtime
  migration you ran, and a time you chose the database over the application (or vice versa).
- **Practise typing without help:** top-N per group, gaps-and-islands, dedupe with `ROW_NUMBER`,
  running total with the correct `ROWS` frame, keyset pagination.

Good luck. Talk about trade-offs, not trivia — that's what nine years is supposed to buy.
