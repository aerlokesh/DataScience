# 🎯 Topic 19: SQL with Python

> **Data Science Interview — SQL + Python Integration Deep Dive**  
> Master the bridge between SQL and Python for analytics roles at Google, Meta, Amazon, Netflix, Spotify, and Uber. Covers sqlite3, pandas integration, side-by-side SQL/Pandas translations, window functions, classic interview problems, and the decision framework interviewers expect from a 5-YOE candidate.

---

## Table of Contents

1. [sqlite3 Module Fundamentals](#sqlite3-module-fundamentals)
2. [Pandas SQL Integration](#pandas-sql-integration)
3. [SQL-to-Pandas Translation: Core Patterns](#sql-to-pandas-translation-core-patterns)
4. [JOINs — SQL vs Pandas Side-by-Side](#joins--sql-vs-pandas-side-by-side)
5. [Window Functions in SQL and Pandas](#window-functions-in-sql-and-pandas)
6. [CTEs and Self-Joins](#ctes-and-self-joins)
7. [Classic Interview Problems with Solutions](#classic-interview-problems-with-solutions)
8. [Decision Framework: SQL vs Pandas](#decision-framework-sql-vs-pandas)
9. [Interview Talking Points & Scripts](#interview-talking-points--scripts)
10. [Common Interview Mistakes](#common-interview-mistakes)
11. [Rapid-Fire Q&A (Top 12)](#rapid-fire-qa-top-12)
12. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## sqlite3 Module Fundamentals

```python
import sqlite3

conn = sqlite3.connect(":memory:")  # or "file.db"
cursor = conn.cursor()

cursor.execute("""CREATE TABLE orders (
    order_id INTEGER PRIMARY KEY, user_id INTEGER,
    amount REAL, order_date TEXT)""")

data = [(1,101,49.99,'2024-01-15'),(2,102,89.50,'2024-01-16'),
        (3,101,23.00,'2024-01-17'),(4,103,150.0,'2024-01-17')]
cursor.executemany("INSERT INTO orders VALUES (?,?,?,?)", data)
conn.commit()

cursor.execute("SELECT user_id, SUM(amount) FROM orders GROUP BY user_id")
results = cursor.fetchall()  # list of tuples
conn.close()
```

> **Critical Insight:** Always use parameterized queries (`?` placeholders) — never f-strings. Interviewers will flag SQL injection even in prototype code.

| Method | Returns | Use Case |
| ------ | ------- | -------- |
| `cursor.execute(sql, params)` | cursor | Single statement |
| `cursor.executemany(sql, seq)` | cursor | Bulk insert |
| `cursor.fetchone()` | tuple/None | Next row |
| `cursor.fetchall()` | list[tuple] | All results |
| `conn.commit()` | None | Persist changes |

```python
# Production pattern: context manager + named columns
with sqlite3.connect("analytics.db") as conn:
    conn.row_factory = sqlite3.Row
    cur = conn.execute("SELECT * FROM orders WHERE amount > ?", (50.0,))
    for row in cur:
        print(row["user_id"], row["amount"])
```

---

## Pandas SQL Integration

```python
import pandas as pd

# Read query into DataFrame
df = pd.read_sql("SELECT user_id, SUM(amount) as total FROM orders GROUP BY user_id", conn)

# Read with date parsing
df = pd.read_sql("SELECT * FROM orders", conn, parse_dates=["order_date"])

# Write DataFrame to SQL
df.to_sql("summary", conn, if_exists="replace", index=False)
```

| `if_exists` | Behavior |
| ----------- | -------- |
| `"fail"` | Error if table exists (default) |
| `"replace"` | Drop + recreate table |
| `"append"` | Insert into existing table |

> **Critical Insight:** For large datasets, use `chunksize` to avoid OOM: `for chunk in pd.read_sql(q, conn, chunksize=10000): process(chunk)`

---

## SQL-to-Pandas Translation: Core Patterns

```python
# SQL
q = """SELECT department, AVG(salary) as avg_sal, COUNT(*) as n
       FROM employees WHERE hire_date >= '2020-01-01'
       GROUP BY department HAVING COUNT(*) >= 5
       ORDER BY avg_sal DESC"""

# Pandas equivalent
result = (
    df[df["hire_date"] >= "2020-01-01"]
    .groupby("department")
    .agg(avg_sal=("salary","mean"), n=("salary","count"))
    .query("n >= 5")
    .sort_values("avg_sal", ascending=False)
    .reset_index()
)
```

| SQL Clause | Pandas Method |
| ---------- | ------------- |
| `SELECT col1, col2` | `df[["col1","col2"]]` |
| `SELECT DISTINCT` | `df.drop_duplicates()` |
| `WHERE cond` | `df[mask]` or `df.query("cond")` |
| `GROUP BY` | `df.groupby()` |
| `HAVING` | `.query()` after agg |
| `ORDER BY` | `.sort_values()` |
| `LIMIT n` | `.head(n)` |
| `CASE WHEN` | `np.where()` / `np.select()` |
| `COALESCE` | `.fillna()` |

---

## JOINs — SQL vs Pandas Side-by-Side

```python
# Setup
users = pd.DataFrame({"user_id":[1,2,3,4], "name":["Alice","Bob","Carol","Dave"]})
orders = pd.DataFrame({"order_id":[101,102,103,104], "user_id":[1,2,2,5], "amount":[50,30,80,120]})
```

```python
# INNER JOIN
sql:  "SELECT * FROM users u INNER JOIN orders o ON u.user_id = o.user_id"
pd:   users.merge(orders, on="user_id", how="inner")

# LEFT JOIN
sql:  "SELECT * FROM users u LEFT JOIN orders o ON u.user_id = o.user_id"
pd:   users.merge(orders, on="user_id", how="left")

# RIGHT JOIN
sql:  "SELECT * FROM users u RIGHT JOIN orders o ON u.user_id = o.user_id"
pd:   users.merge(orders, on="user_id", how="right")

# FULL OUTER JOIN
sql:  "SELECT * FROM users u FULL OUTER JOIN orders o ON u.user_id = o.user_id"
pd:   users.merge(orders, on="user_id", how="outer")
```

| Join Type | Pandas `how=` | Keeps |
| --------- | ------------- | ----- |
| Inner | `"inner"` | Matching rows only |
| Left | `"left"` | All left + matching right |
| Right | `"right"` | All right + matching left |
| Full Outer | `"outer"` | All rows from both |
| Cross | `"cross"` | Cartesian product |

> **Critical Insight:** `merge()` joins on columns; `join()` defaults to index. Always use `merge()` in interviews — it maps directly to SQL semantics.

---

## Window Functions in SQL and Pandas

### ROW_NUMBER / RANK / DENSE_RANK

```python
# SQL
"""SELECT *, ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) as rn,
           RANK() OVER (PARTITION BY dept ORDER BY salary DESC) as rnk,
           DENSE_RANK() OVER (PARTITION BY dept ORDER BY salary DESC) as drnk
   FROM employees"""

# Pandas
df["rn"]   = df.groupby("dept")["salary"].rank(method="first", ascending=False).astype(int)
df["rnk"]  = df.groupby("dept")["salary"].rank(method="min", ascending=False).astype(int)
df["drnk"] = df.groupby("dept")["salary"].rank(method="dense", ascending=False).astype(int)
```

| SQL Function | Pandas `method=` | Tie Behavior |
| ------------ | ---------------- | ------------ |
| `ROW_NUMBER()` | `"first"` | Arbitrary for ties |
| `RANK()` | `"min"` | Skip (1,2,2,4) |
| `DENSE_RANK()` | `"dense"` | No skip (1,2,2,3) |

### LAG / LEAD and Running Totals

```python
# SQL: LAG / LEAD
"""SELECT *, LAG(amount,1) OVER (PARTITION BY user_id ORDER BY date) as prev,
           LEAD(amount,1) OVER (PARTITION BY user_id ORDER BY date) as next
   FROM orders"""

# Pandas
df["prev"] = df.groupby("user_id")["amount"].shift(1)   # LAG
df["next"] = df.groupby("user_id")["amount"].shift(-1)  # LEAD

# SQL: Running total
"""SUM(amount) OVER (PARTITION BY user_id ORDER BY date ROWS UNBOUNDED PRECEDING)"""

# Pandas
df["running_total"] = df.groupby("user_id")["amount"].cumsum()
```

> **Critical Insight:** Default SQL window frame with ORDER BY is `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. Without ORDER BY, it's the entire partition. Classic trick question.

---

## CTEs and Self-Joins

```python
# SQL with CTE (each CTE = one logical step)
"""WITH user_totals AS (
    SELECT user_id, SUM(amount) as total FROM orders GROUP BY user_id
)
SELECT u.name, ut.total FROM user_totals ut JOIN users u ON ut.user_id = u.user_id
WHERE ut.total > 100"""

# Pandas: each CTE becomes an intermediate DataFrame
user_totals = orders.groupby("user_id")["amount"].sum().reset_index(name="total")
result = user_totals[user_totals["total"] > 100].merge(users, on="user_id")

# Self-Join: employees earning more than their manager
"""SELECT e.name, e.salary, m.name as mgr FROM employees e
   JOIN employees m ON e.manager_id = m.id WHERE e.salary > m.salary"""

result = (employees.merge(employees, left_on="manager_id", right_on="id", suffixes=("","_mgr"))
          .query("salary > salary_mgr")[["name","salary","name_mgr"]])
```

---

## Classic Interview Problems with Solutions

### Problem 1: Top-N Per Group

```python
# SQL: Top 2 paid per department
"""WITH ranked AS (
    SELECT *, DENSE_RANK() OVER (PARTITION BY dept ORDER BY salary DESC) as rnk
    FROM employees)
SELECT * FROM ranked WHERE rnk <= 2"""

# Pandas
df["rnk"] = df.groupby("dept")["salary"].rank(method="dense", ascending=False)
top2 = df[df["rnk"] <= 2]
```

### Problem 2: Consecutive Days (Streaks)

```python
# SQL — "islands and gaps" technique
"""WITH numbered AS (
    SELECT *, activity_date - ROW_NUMBER() OVER(PARTITION BY user_id ORDER BY activity_date) as grp
    FROM activity)
SELECT user_id, COUNT(*) as streak FROM numbered GROUP BY user_id, grp HAVING COUNT(*) >= 3"""

# Pandas
df = df.sort_values(["user_id","activity_date"])
df["day_num"] = df.groupby("user_id").cumcount()
df["grp"] = df["activity_date"] - pd.to_timedelta(df["day_num"], unit="D")
streaks = df.groupby(["user_id","grp"]).size().reset_index(name="streak").query("streak >= 3")
```

> **Critical Insight:** The islands-and-gaps technique (subtracting row_number from date) appears in ~30% of Meta/Google SQL screens.

### Problem 3: Year-over-Year Growth

```python
# SQL
"""WITH monthly AS (SELECT strftime('%Y-%m',order_date) as mo, SUM(amount) as rev FROM orders GROUP BY 1)
SELECT *, ROUND((rev - LAG(rev) OVER(ORDER BY mo))*100.0/LAG(rev) OVER(ORDER BY mo),2) as growth
FROM monthly"""

# Pandas
monthly = df.groupby(df["order_date"].dt.to_period("M"))["amount"].sum().reset_index(name="rev")
monthly["growth"] = monthly["rev"].pct_change() * 100
```

### Problem 4: Funnel Analysis

```python
# SQL
"""SELECT COUNT(DISTINCT CASE WHEN event='visit' THEN user_id END) as visits,
       COUNT(DISTINCT CASE WHEN event='signup' THEN user_id END) as signups,
       COUNT(DISTINCT CASE WHEN event='purchase' THEN user_id END) as purchases
FROM events WHERE event_date BETWEEN '2024-01-01' AND '2024-01-31'"""

# Pandas
f = df.query("'2024-01-01' <= event_date <= '2024-01-31'")
funnel = {stage: f[f["event"]==stage]["user_id"].nunique() for stage in ["visit","signup","purchase"]}
```

---

## Decision Framework: SQL vs Pandas

| Use SQL When | Use Pandas When |
| ------------ | --------------- |
| Data lives in a database | Data from APIs/files/mixed sources |
| Dataset exceeds memory | Small-medium data (< 5GB in-memory) |
| Complex multi-table JOINs | Complex transforms (regex, UDFs) |
| Shared logic across teams (dbt) | ML feature engineering |
| Production ETL pipelines | Prototyping / visualization |

> **Critical Insight:** The 5-YOE answer is "both." SQL handles extraction + heavy aggregation (pushdown to DB), pandas handles enrichment + ML prep. This is the **ELT** pattern.

---

## Interview Talking Points & Scripts

### "Walk me through how you'd solve this SQL problem"

> *"I start by clarifying the expected output — columns, granularity, edge cases like NULLs. Then I decompose into CTEs: each handles one logical step. If the question says 'for each group' with row-level detail, that signals a window function over GROUP BY. I validate by tracing a sample row through the logic mentally before declaring done."*

### "When would you use SQL vs Pandas in production?"

> *"SQL for anything pushable to the database — aggregations, joins, filtering on large tables. The optimizer handles billions of rows efficiently. Pandas for Python-specific transforms: regex parsing, custom business logic, statistical computations, or ML features. Typical production pipeline: SQL extracts and aggregates, pandas enriches and prepares for models or dashboards."*

### "How do you optimize a slow query?"

> *"I run EXPLAIN ANALYZE to spot full scans and expensive sorts. Common fixes: indexes on JOIN/WHERE columns, reducing data early with subqueries, avoiding SELECT star, checking for accidental Cartesian products. In pandas, I use chunksize processing or push heavy work to the database."*

---

## Common Interview Mistakes

| # | Mistake | Correction |
| - | ------- | ---------- |
| 1 | Using GROUP BY when needing row-level detail | Use window functions with OVER(PARTITION BY ...) |
| 2 | Writing `WHERE col = NULL` | Use `WHERE col IS NULL` — NULL is not a value |
| 3 | Confusing HAVING vs WHERE | WHERE filters before grouping; HAVING filters after |
| 4 | Using `merge()` without explicit `how=` | Default is inner — always specify join type |
| 5 | Confusing RANK vs DENSE_RANK | RANK: 1,2,2,4 (skips); DENSE_RANK: 1,2,2,3 |
| 6 | f-strings for SQL parameters | Use `?` placeholders to prevent injection |
| 7 | Using `.apply()` when `.transform()` works | transform preserves shape; apply is slower |
| 8 | Forgetting `reset_index()` after groupby | GroupBy sets grouped col as index |
| 9 | Not checking cardinality before JOINs | Many-to-many creates Cartesian explosion |
| 10 | One massive monolithic query | Break into CTEs — readability matters |

---

## Rapid-Fire Q&A (Top 12)

**Q1: `WHERE` vs `HAVING`?**  
WHERE filters rows before aggregation; HAVING filters groups after GROUP BY.

**Q2: `read_sql()` vs `read_sql_query()`?**  
`read_sql()` accepts both table names and SQL; `read_sql_query()` only SQL strings.

**Q3: Default window frame with ORDER BY?**  
`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.

**Q4: Second-highest salary per department?**  
`DENSE_RANK() OVER(PARTITION BY dept ORDER BY salary DESC)` then filter rank = 2.

**Q5: NULL behavior in LEFT JOIN non-matches?**  
Right-side columns become NULL for unmatched left rows.

**Q6: Pandas duplicate column names after merge?**  
Adds `_x`/`_y` suffixes. Customize with `suffixes=("_left","_right")`.

**Q7: Window function in WHERE clause?**  
Not allowed — evaluated after WHERE. Wrap in CTE/subquery to filter.

**Q8: Pandas equivalent of COALESCE(a,b,c)?**  
`df["a"].fillna(df["b"]).fillna(df["c"])`.

**Q9: Prevent SQL injection with sqlite3?**  
Parameterized queries: `cursor.execute("... WHERE id=?", (val,))`.

**Q10: What is "islands and gaps"?**  
Subtract row_number from sequential value — consecutive items get same group ID.

**Q11: `to_sql(if_exists='replace')` danger?**  
Drops table, losing indexes/constraints/permissions. Use `'append'` in production.

**Q12: Large result sets with read_sql?**  
Use `chunksize=N` for an iterator of DataFrames processed incrementally.

---

## Summary Cheat Sheet

```
+----------------------------------------------------------------------+
|                   SQL WITH PYTHON — CHEAT SHEET                      |
+----------------------------------------------------------------------+
|                                                                      |
|  SQLITE3:  conn = sqlite3.connect(":memory:")                        |
|            cur.execute("SELECT ... WHERE id=?", (val,))              |
|            results = cur.fetchall()                                   |
|                                                                      |
|  PANDAS:   df = pd.read_sql(query, conn, parse_dates=["d"])          |
|            df.to_sql("tbl", conn, if_exists="append", index=False)   |
|                                                                      |
|  JOINS:    INNER -> how="inner"   LEFT -> how="left"                 |
|            RIGHT -> how="right"   FULL -> how="outer"                |
|                                                                      |
|  WINDOWS:  ROW_NUMBER -> rank(method="first")                        |
|            RANK -> rank(method="min")                                 |
|            DENSE_RANK -> rank(method="dense")                         |
|            LAG(1) -> shift(1)    LEAD(1) -> shift(-1)                |
|            running SUM -> cumsum()                                    |
|                                                                      |
|  PATTERNS: Top-N = DENSE_RANK + filter rnk <= N                      |
|            Streaks = date - ROW_NUMBER() groups consecutive           |
|            YoY = LAG() then pct_change                               |
|            Funnel = COUNT(DISTINCT CASE WHEN...) per stage            |
|                                                                      |
|  WHEN:     SQL -> DB-resident, large, shared, ETL                    |
|            Pandas -> files/APIs, UDFs, ML, visualization             |
|                                                                      |
+----------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 19 of 25*
