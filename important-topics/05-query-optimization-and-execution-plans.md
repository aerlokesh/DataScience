# Topic 5: Query Optimization & Execution Plans

> **Data Science Interview -- Deep Dive**  
> Reading EXPLAIN plans, understanding index types, knowing join algorithms, spotting performance killers, and applying optimization strategies that prove you can work with production-scale data -- not just toy datasets.

---

## Table of Contents

1. [Why Optimization Matters for Data Scientists](#1-why-optimization-matters-for-data-scientists)
2. [EXPLAIN Plan Anatomy](#2-explain-plan-anatomy)
3. [Index Types and When to Use Them](#3-index-types-and-when-to-use-them)
4. [Scan Operators -- How Data Is Accessed](#4-scan-operators--how-data-is-accessed)
5. [Join Algorithms -- The Big Three](#5-join-algorithms--the-big-three)
6. [Common Performance Killers](#6-common-performance-killers)
7. [Optimization Strategies](#7-optimization-strategies)
8. [Partitioning and Partition Pruning](#8-partitioning-and-partition-pruning)
9. [Materialized Views](#9-materialized-views)
10. [Statistics and the Query Planner](#10-statistics-and-the-query-planner)
11. [Interview Problems with Solutions](#11-interview-problems-with-solutions)
12. [Interview Talking Points](#12-interview-talking-points)
13. [Common Interview Mistakes](#13-common-interview-mistakes)
14. [Rapid-Fire Q&A](#14-rapid-fire-qa)
15. [Summary Cheat Sheet](#15-summary-cheat-sheet)

---

## 1. Why Optimization Matters for Data Scientists

Data scientists at 5 YOE are expected to:

- Write queries that finish in minutes, not hours, on billion-row tables
- Diagnose why a colleague's pipeline query is slow
- Understand cost/benefit of adding indexes vs. restructuring queries
- Know when to escalate to a data engineer vs. fixing it themselves

> **Critical Insight:** In interviews, mentioning "I'd check the execution plan" when asked to optimize a query immediately signals senior-level thinking. Even if you can't recite every operator, showing you know the tool exists sets you apart.

---

## 2. EXPLAIN Plan Anatomy

### Running EXPLAIN

```sql
-- Basic plan (estimated costs only, does NOT execute)
EXPLAIN SELECT * FROM orders WHERE user_id = 42;

-- Actual execution stats (runs the query)
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42;

-- Verbose with buffers (PostgreSQL)
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) 
SELECT * FROM orders WHERE user_id = 42;
```

### Reading a Plan (PostgreSQL Example)

```
Nested Loop  (cost=0.43..16.50 rows=1 width=64) (actual time=0.05..0.06 rows=1 loops=1)
  -> Index Scan using idx_orders_user on orders  (cost=0.43..8.45 rows=1 width=32)
       Index Cond: (user_id = 42)
  -> Index Scan using idx_order_items_order on order_items  (cost=0.29..8.03 rows=3 width=32)
       Index Cond: (order_id = orders.order_id)
Planning Time: 0.12 ms
Execution Time: 0.08 ms
```

### Key Metrics to Read

| Metric | Meaning |
|--------|---------|
| **cost** | Estimated startup cost..total cost (in arbitrary units) |
| **rows** | Estimated number of rows output |
| **width** | Average row size in bytes |
| **actual time** | Real wall-clock time (startup..total) in ms |
| **actual rows** | Real rows returned |
| **loops** | Number of times this node executed |
| **Buffers: shared hit** | Pages read from cache |
| **Buffers: shared read** | Pages read from disk |

> **Critical Insight:** The most important diagnostic is comparing **estimated rows** vs **actual rows**. A large mismatch (e.g., estimated 1 row, actual 100,000 rows) means stale statistics or a bad cardinality estimate, which leads the planner to choose the wrong join algorithm or scan type.

---

## 3. Index Types and When to Use Them

### B-Tree Index (Default)

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_date_user ON orders(order_date, user_id);  -- composite
```

| Property | B-Tree |
|----------|--------|
| Best for | Equality (=), range (<, >, BETWEEN), ORDER BY, prefix LIKE |
| Supports | Multi-column composite indexes |
| Limitations | Not useful for IS NULL (older engines), LIKE '%suffix' |
| Interview note | Default choice; mention it first |

### Hash Index

```sql
CREATE INDEX idx_users_email_hash ON users USING hash(email);
```

| Property | Hash |
|----------|------|
| Best for | Exact equality only (=) |
| Cannot do | Range queries, ORDER BY, partial matches |
| Advantage | O(1) lookup vs O(log n) for B-Tree on equality |
| Limitation | Not crash-safe in older Postgres (pre-10); not WAL-logged |

### Bitmap Index

```sql
-- Oracle/Greenplum syntax (not native PostgreSQL -- Postgres uses bitmap scans internally)
CREATE BITMAP INDEX idx_orders_status ON orders(status);
```

| Property | Bitmap |
|----------|--------|
| Best for | Low-cardinality columns (status, gender, region) |
| Advantage | Very compact; efficient for AND/OR on multiple conditions |
| Limitation | Expensive to update; bad for OLTP with frequent writes |
| Interview note | Common in data warehouses (Redshift, Oracle) |

### GIN Index (Generalized Inverted)

```sql
CREATE INDEX idx_products_tags ON products USING gin(tags);  -- array/jsonb
```

| Property | GIN |
|----------|-----|
| Best for | Full-text search, array containment, JSONB queries |
| Advantage | Handles multi-valued attributes efficiently |
| Limitation | Slow to build and update; larger storage |

### Composite Index Column Order

```sql
-- Index on (a, b, c) supports:
-- WHERE a = ?                    (yes -- leftmost prefix)
-- WHERE a = ? AND b = ?          (yes)
-- WHERE a = ? AND b = ? AND c = ? (yes -- full match)
-- WHERE b = ?                    (NO -- skips leftmost column)
-- WHERE a = ? AND c = ?          (partial -- uses a, cannot skip b for c range)
```

> **Critical Insight:** The "leftmost prefix rule" for composite indexes is one of the most commonly tested optimization concepts. Always design composite indexes with the most selective (or most filtered) column first, and range columns last.

---

## 4. Scan Operators -- How Data Is Accessed

| Operator | Description | When Used | Cost |
|----------|-------------|-----------|------|
| **Seq Scan** | Full table scan, reads every row | No usable index; small table; large % of rows needed | O(n) |
| **Index Scan** | Traverse B-Tree, fetch heap pages | Selective filter with matching index | O(log n + k) |
| **Index Only Scan** | Reads only index, no heap fetch | All needed columns are in the index (covering) | Fastest |
| **Bitmap Index Scan** | Build bitmap from index, then scan heap | Multiple conditions OR'd; moderate selectivity | Between seq and index |
| **Bitmap Heap Scan** | Fetch heap pages using bitmap | Follows bitmap index scan | Ordered I/O |

```sql
-- Force a sequential scan (for testing/comparison)
SET enable_indexscan = off;
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42;
SET enable_indexscan = on;
```

### When the Planner Chooses Seq Scan Despite an Index

- Selecting a large fraction of the table (>10-20% of rows)
- Table is very small (fits in a few pages)
- Statistics are stale (ANALYZE not run)
- Function applied to indexed column (see sargability below)

---

## 5. Join Algorithms -- The Big Three

### Nested Loop Join

```
For each row in outer table:
    For each row in inner table:
        If join condition matches, emit combined row
```

| Property | Value |
|----------|-------|
| Complexity | O(n * m) worst case |
| Best when | Small outer table; inner table has good index |
| Supports | All join types including inequality joins |
| Interview note | Default for small datasets or when index is available on inner |

### Hash Join

```
Phase 1 (Build): Hash the smaller table into memory
Phase 2 (Probe): Scan larger table, probe hash table for matches
```

| Property | Value |
|----------|-------|
| Complexity | O(n + m) -- linear |
| Best when | Large tables, no useful index, equality join |
| Limitation | Requires equality condition; memory for hash table |
| Spill | If hash table exceeds work_mem, spills to disk (slow) |
| Interview note | Most common for analytical queries on large tables |

### Merge Join (Sort-Merge)

```
Phase 1: Sort both tables on join key
Phase 2: Walk through both sorted lists simultaneously
```

| Property | Value |
|----------|-------|
| Complexity | O(n log n + m log m) for sort, then O(n + m) for merge |
| Best when | Both inputs already sorted (from index or prior sort) |
| Advantage | Memory-efficient; handles large data gracefully |
| Limitation | Requires equality or range condition; sort cost |
| Interview note | Preferred in data warehouses with pre-sorted data |

### Comparison Summary

| Algorithm | Best For | Memory | Equality Only | Pre-sorted Benefit |
|-----------|----------|--------|---------------|-------------------|
| Nested Loop | Small outer, indexed inner | Low | No | No |
| Hash Join | Large unsorted tables, equi-join | High | Yes | No |
| Merge Join | Large pre-sorted data | Medium | No (supports >=, etc.) | Yes |

---

## 6. Common Performance Killers

### 1. Non-Sargable Predicates (Functions on Indexed Columns)

```sql
-- BAD: Function on indexed column -- index cannot be used
SELECT * FROM orders WHERE YEAR(order_date) = 2024;
SELECT * FROM users WHERE UPPER(email) = 'JOHN@EXAMPLE.COM';

-- GOOD: Sargable rewrites
SELECT * FROM orders WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01';
SELECT * FROM users WHERE email = LOWER('JOHN@EXAMPLE.COM');  -- or functional index
```

### 2. Implicit Type Casting

```sql
-- BAD: user_id is INTEGER but compared to string -- forces cast on every row
SELECT * FROM orders WHERE user_id = '42';

-- GOOD: Match the type
SELECT * FROM orders WHERE user_id = 42;
```

### 3. SELECT * (Over-fetching)

```sql
-- BAD: Fetches all columns; prevents index-only scans
SELECT * FROM orders WHERE user_id = 42;

-- GOOD: Select only needed columns (enables covering index)
SELECT order_id, order_date, amount FROM orders WHERE user_id = 42;
```

### 4. Missing Statistics

```sql
-- After bulk loading, ALWAYS update statistics
ANALYZE orders;                    -- PostgreSQL
ANALYZE TABLE orders COMPUTE STATISTICS;  -- Oracle/Hive
```

### 5. Correlated Subqueries on Large Tables

```sql
-- BAD: Executes inner query once per outer row
SELECT *, (SELECT MAX(amount) FROM orders o2 WHERE o2.user_id = o1.user_id)
FROM orders o1;

-- GOOD: Window function (single pass)
SELECT *, MAX(amount) OVER (PARTITION BY user_id) AS max_user_amount
FROM orders;
```

### 6. OR Conditions That Prevent Index Use

```sql
-- BAD: OR across different columns -- single index cannot cover both
SELECT * FROM orders WHERE user_id = 42 OR product_id = 99;

-- GOOD: UNION ALL with separate index scans
SELECT * FROM orders WHERE user_id = 42
UNION ALL
SELECT * FROM orders WHERE product_id = 99 AND user_id != 42;
```

---

## 7. Optimization Strategies

### Strategy 1: Cover Your Queries (Covering Indexes)

```sql
-- If you frequently run:
SELECT user_id, order_date, amount FROM orders WHERE user_id = ? ORDER BY order_date;

-- Create a covering index:
CREATE INDEX idx_orders_covering ON orders(user_id, order_date, amount);
-- Now the engine never needs to read the heap -- index-only scan
```

### Strategy 2: Predicate Pushdown

```sql
-- BAD: Filter after expensive join
SELECT * FROM (
    SELECT o.*, u.name 
    FROM orders o 
    JOIN users u ON o.user_id = u.user_id
) sub
WHERE sub.order_date >= '2024-01-01';

-- GOOD: Filter early (most optimizers do this automatically, but be explicit)
SELECT o.*, u.name
FROM orders o
JOIN users u ON o.user_id = u.user_id
WHERE o.order_date >= '2024-01-01';  -- planner pushes this to the orders scan
```

### Strategy 3: Aggregate Before Join

```sql
-- BAD: Join creates fan-out, then aggregate
SELECT u.user_id, u.name, SUM(o.amount)
FROM users u
JOIN orders o ON u.user_id = o.user_id
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY u.user_id, u.name;

-- GOOD: Pre-aggregate at lowest grain
WITH order_totals AS (
    SELECT user_id, SUM(amount) AS total_amount
    FROM orders
    GROUP BY user_id
)
SELECT u.user_id, u.name, ot.total_amount
FROM users u
JOIN order_totals ot ON u.user_id = ot.user_id;
```

### Strategy 4: Use EXISTS Instead of COUNT for Existence Checks

```sql
-- BAD: Count all matching rows just to check if any exist
SELECT user_id FROM users u
WHERE (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.user_id) > 0;

-- GOOD: Short-circuit after finding first match
SELECT user_id FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.user_id);
```

---

## 8. Partitioning and Partition Pruning

### Partition Types

| Type | Splits By | Best For |
|------|-----------|----------|
| Range | Value ranges (dates, IDs) | Time-series data, logs |
| List | Discrete values | Region, status, category |
| Hash | Hash of column value | Even distribution when no natural range |

### Range Partitioning Example

```sql
-- PostgreSQL declarative partitioning
CREATE TABLE events (
    event_id BIGINT,
    event_date DATE,
    user_id INT,
    event_type TEXT
) PARTITION BY RANGE (event_date);

CREATE TABLE events_2024_q1 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE events_2024_q2 PARTITION OF events
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
```

### Partition Pruning in Action

```sql
-- This query only scans events_2024_q1 (pruning eliminates other partitions)
EXPLAIN SELECT * FROM events WHERE event_date = '2024-02-15';
-- Plan output: "Append -> Seq Scan on events_2024_q1"

-- WARNING: Function on partition key disables pruning
EXPLAIN SELECT * FROM events WHERE DATE_TRUNC('month', event_date) = '2024-02-01';
-- Plan output: "Append -> Seq Scan on events_2024_q1, events_2024_q2, ..."  (ALL partitions)
```

> **Critical Insight:** Partition pruning only works when the WHERE clause directly filters on the partition key with a sargable expression. Wrapping the partition key in a function defeats pruning -- the same principle as index sargability.

---

## 9. Materialized Views

Materialized views store pre-computed results. They trade storage and staleness for query speed.

```sql
-- Create a materialized view for an expensive dashboard query
CREATE MATERIALIZED VIEW mv_daily_revenue AS
SELECT 
    order_date::DATE AS day,
    product_category,
    SUM(amount) AS daily_revenue,
    COUNT(DISTINCT user_id) AS unique_buyers
FROM orders o
JOIN products p ON o.product_id = p.product_id
WHERE order_date >= '2024-01-01'
GROUP BY order_date::DATE, product_category;

-- Create index on the materialized view
CREATE INDEX idx_mv_daily_rev ON mv_daily_revenue(day, product_category);

-- Refresh (manual -- blocks reads during refresh)
REFRESH MATERIALIZED VIEW mv_daily_revenue;

-- Concurrent refresh (does not block reads; requires unique index)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_revenue;
```

### When to Use Materialized Views

| Use Case | Benefit |
|----------|---------|
| Dashboard queries run frequently | Sub-second response vs. minutes |
| Aggregations over huge fact tables | Pre-computed once, read many times |
| Cross-database joins not possible at query time | Materialized copy of external data |

### Trade-offs

| Advantage | Disadvantage |
|-----------|--------------|
| Fast reads | Stale data (until refresh) |
| Indexable | Storage cost |
| Simplifies downstream queries | Refresh maintenance overhead |

---

## 10. Statistics and the Query Planner

The query planner makes decisions based on **statistics** about table size, column distributions, and correlation.

```sql
-- PostgreSQL: View statistics for a column
SELECT 
    attname,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    correlation
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

### When Statistics Go Wrong

- After bulk INSERT/DELETE without ANALYZE
- Highly skewed data (one value = 90% of rows)
- Correlated columns that planner assumes are independent

```sql
-- Force statistics refresh
ANALYZE orders;

-- Increase statistics target for important columns (Postgres)
ALTER TABLE orders ALTER COLUMN user_id SET STATISTICS 1000;  -- default is 100
ANALYZE orders;
```

---

## 11. Interview Problems with Solutions

### Problem 1: "This query is slow. How would you diagnose it?"

**Answer framework:**

```sql
-- Step 1: Run EXPLAIN ANALYZE
EXPLAIN ANALYZE
SELECT u.name, COUNT(o.order_id)
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
WHERE u.signup_date >= '2024-01-01'
GROUP BY u.name;

-- Step 2: Look for red flags
-- - Seq Scans on large tables (missing index?)
-- - Estimated rows vs actual rows mismatch (stale stats?)
-- - Sort or Hash spilling to disk (need more memory?)
-- - Nested Loop with large outer set (wrong join algorithm?)

-- Step 3: Potential fixes
-- a) Add index: CREATE INDEX idx_users_signup ON users(signup_date);
-- b) Update stats: ANALYZE users; ANALYZE orders;
-- c) Rewrite: Pre-filter users, then join
-- d) Check if SELECT u.name forces a non-covering scan
```

### Problem 2: "Design an index strategy for this table"

**Given:** A 500M-row `events` table with queries filtering on `(user_id, event_type, event_date)`.

```sql
-- Most common query patterns:
-- 1. WHERE user_id = ? AND event_date BETWEEN ? AND ?
-- 2. WHERE user_id = ? AND event_type = ? ORDER BY event_date DESC LIMIT 10
-- 3. WHERE event_type = ? AND event_date = ?

-- Recommended indexes:
CREATE INDEX idx_events_user_date ON events(user_id, event_date);        -- Pattern 1
CREATE INDEX idx_events_user_type_date ON events(user_id, event_type, event_date DESC);  -- Pattern 2
CREATE INDEX idx_events_type_date ON events(event_type, event_date);     -- Pattern 3

-- Consider partitioning by event_date (range) for pruning
```

---

## 12. Interview Talking Points

> "My first step when optimizing a slow query is always EXPLAIN ANALYZE. I look for three things: sequential scans on large tables, large discrepancies between estimated and actual row counts, and sort or hash operations spilling to disk."

> "For index design, I follow the equality-sort-range (ESR) rule: put equality predicates first in the composite index, then sort columns, then range predicates last. This maximizes the prefix that the planner can use."

> "I'm aware that adding indexes is not free -- they slow down writes, consume storage, and require maintenance. For write-heavy OLTP tables, I'm selective about indexes. For analytical workloads that are read-heavy, I index more aggressively."

> "In warehouse environments, I rely heavily on partition pruning. I ensure that WHERE clauses are sargable on partition keys and that I never wrap partition columns in functions."

---

## 13. Common Interview Mistakes

| Mistake | Why It's Wrong | Correct Approach |
|---------|---------------|-----------------|
| "Just add an index on every column" | Indexes cost write performance and storage | Design indexes based on actual query patterns |
| Wrapping indexed columns in functions | Prevents index usage (non-sargable) | Rewrite predicate to keep column bare |
| Ignoring estimated vs actual row mismatch | Causes wrong join algorithm / scan choice | Run ANALYZE; increase statistics target |
| Not knowing join algorithm trade-offs | Cannot explain why a query is slow | Know nested loop vs hash vs merge characteristics |
| Assuming the optimizer is always right | Stale stats or parameter sniffing can mislead it | Verify with EXPLAIN ANALYZE |
| Forgetting about covering indexes | Extra heap fetches for columns not in index | Include frequently selected columns in index |
| Adding index without checking write impact | Heavy INSERT workload suffers | Measure write throughput before/after |
| Ignoring partition pruning requirements | Full partition scan when pruning fails | Keep partition key filters sargable |
| Not considering data distribution | Uniform assumption fails on skewed data | Check histogram; increase stats target |
| Over-relying on SELECT * | Prevents index-only scans; wastes I/O | Select only needed columns |

---

## 14. Rapid-Fire Q&A

**Q1: What does "cost" mean in an EXPLAIN plan?**  
A: An abstract unit representing estimated I/O and CPU. The absolute number is meaningless; compare relative costs between plan alternatives.

**Q2: When would a Seq Scan be faster than an Index Scan?**  
A: When selecting >10-20% of the table. Sequential I/O (one disk sweep) is much faster than random I/O (many index-to-heap jumps).

**Q3: What is "index-only scan" and when does it happen?**  
A: When all columns needed by the query are in the index. The heap is never touched. Requires the visibility map to be up-to-date (run VACUUM).

**Q4: How does VACUUM relate to performance?**  
A: VACUUM reclaims dead tuples and updates the visibility map (enabling index-only scans). Without it, bloat increases and performance degrades over time.

**Q5: What is work_mem and why does it matter?**  
A: The memory allocated per sort/hash operation. If the data exceeds work_mem, it spills to disk (temp files), dramatically slowing the query.

**Q6: How do you optimize a query with multiple OR conditions?**  
A: Rewrite as UNION ALL of separate queries, each using its own index. Or use a bitmap OR scan if available.

**Q7: What is the difference between a clustered and non-clustered index?**  
A: A clustered index (SQL Server) / CLUSTER (Postgres) physically reorders the table data to match the index. Only one per table. Dramatically helps range scans.

**Q8: What is "selectivity" in the context of indexes?**  
A: The fraction of rows an index lookup returns. High selectivity (few rows) makes indexes worthwhile; low selectivity (many rows) makes seq scan preferable.

**Q9: How do you know if a temp file spill occurred?**  
A: In EXPLAIN (ANALYZE, BUFFERS), look for "Sort Method: external merge Disk: XXkB" or "Batches: N" in hash operations.

**Q10: What is predicate pushdown?**  
A: The optimizer pushing WHERE filters down to the earliest possible point in the plan (e.g., into a table scan rather than after a join). Most modern engines do this automatically.

---

## 15. Summary Cheat Sheet

```
+------------------------------------------------------------------+
|  QUERY OPTIMIZATION & EXECUTION PLANS -- CHEAT SHEET             |
+------------------------------------------------------------------+
|                                                                    |
|  DIAGNOSIS WORKFLOW:                                               |
|    1. EXPLAIN ANALYZE (actual times + row counts)                  |
|    2. Compare estimated vs actual rows (stale stats?)              |
|    3. Identify seq scans on large tables (missing index?)          |
|    4. Check for disk spills (increase work_mem?)                   |
|    5. Verify sargability of predicates                             |
|                                                                    |
|  INDEX TYPES:                                                      |
|    B-Tree  = default; equality + range + ORDER BY                  |
|    Hash    = equality only; O(1) lookup                            |
|    Bitmap  = low cardinality; warehouse workloads                  |
|    GIN     = arrays, JSONB, full-text search                       |
|                                                                    |
|  JOIN ALGORITHMS:                                                  |
|    Nested Loop = small outer + indexed inner                       |
|    Hash Join   = large tables + equality + no index                |
|    Merge Join  = pre-sorted inputs + any comparison                |
|                                                                    |
|  PERFORMANCE KILLERS:                                              |
|    - Functions on indexed columns (non-sargable)                   |
|    - Implicit type casting                                         |
|    - SELECT * preventing index-only scan                           |
|    - OR conditions spanning multiple columns                       |
|    - Stale statistics after bulk loads                              |
|    - Correlated subqueries on large tables                         |
|                                                                    |
|  PARTITIONING:                                                     |
|    - Range (dates), List (categories), Hash (even distribution)    |
|    - Pruning only works with sargable filters on partition key     |
|                                                                    |
|  MATERIALIZED VIEWS:                                               |
|    - Trade staleness for speed                                     |
|    - Index them; REFRESH CONCURRENTLY for zero-downtime            |
|                                                                    |
+------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Topics](README.md) collection -- Topic 5 of 45*
