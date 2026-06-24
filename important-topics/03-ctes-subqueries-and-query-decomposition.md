# Topic 3: CTEs, Subqueries & Query Decomposition

> **Data Science Interview -- Deep Dive**  
> Breaking complex analytical queries into readable, maintainable, and performant components using CTEs, subqueries, temp tables, and recursive patterns that demonstrate senior-level SQL fluency.

---

## Table of Contents

1. [Why Query Decomposition Matters](#1-why-query-decomposition-matters)
2. [Common Table Expressions (CTEs) -- The Foundation](#2-common-table-expressions-ctes--the-foundation)
3. [Subqueries -- Inline Power](#3-subqueries--inline-power)
4. [Correlated Subqueries -- Row-by-Row Logic](#4-correlated-subqueries--row-by-row-logic)
5. [Temp Tables and Table Variables](#5-temp-tables-and-table-variables)
6. [CTE vs Subquery vs Temp Table -- The Decision Matrix](#6-cte-vs-subquery-vs-temp-table--the-decision-matrix)
7. [Recursive CTEs -- Hierarchical Data](#7-recursive-ctes--hierarchical-data)
8. [Multi-CTE Query Architecture](#8-multi-cte-query-architecture)
9. [Performance Implications](#9-performance-implications)
10. [Interview Problems with Solutions](#10-interview-problems-with-solutions)
11. [Interview Talking Points](#11-interview-talking-points)
12. [Common Interview Mistakes](#12-common-interview-mistakes)
13. [Rapid-Fire Q&A](#13-rapid-fire-qa)
14. [Summary Cheat Sheet](#14-summary-cheat-sheet)

---

## 1. Why Query Decomposition Matters

In data science interviews, you are often asked to write queries that solve multi-step analytical problems. The way you decompose your query tells the interviewer:

- Whether you can think in logical steps
- Whether you write production-ready code or spaghetti SQL
- Whether you understand performance trade-offs

> **Critical Insight:** Interviewers at the 5-YOE level expect you to proactively structure your query before writing it. Narrate your decomposition strategy: "I'll first isolate the cohort, then compute their metrics, then rank them."

---

## 2. Common Table Expressions (CTEs) -- The Foundation

A CTE is a named temporary result set defined within a `WITH` clause that exists only for the duration of the query.

```sql
-- DS Example: Find power users (top 10% by order frequency) and their avg basket size
WITH user_orders AS (
    SELECT 
        user_id,
        COUNT(*) AS order_count,
        AVG(order_total) AS avg_basket
    FROM orders
    WHERE order_date >= '2024-01-01'
    GROUP BY user_id
),
percentile_calc AS (
    SELECT 
        PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY order_count) AS p90_threshold
    FROM user_orders
)
SELECT 
    uo.user_id,
    uo.order_count,
    uo.avg_basket
FROM user_orders uo
CROSS JOIN percentile_calc pc
WHERE uo.order_count >= pc.p90_threshold
ORDER BY uo.order_count DESC;
```

**Why CTEs win in interviews:**
- Each step has a name -- self-documenting
- You can explain your logic step-by-step as you write
- Easy to debug (comment out final SELECT and test each CTE)

**CTE Characteristics:**
| Property | Behavior |
|----------|----------|
| Scope | Single statement only |
| Materialization | Engine-dependent (Postgres may inline, BigQuery materializes) |
| Reusability | Can reference earlier CTEs in the same WITH block |
| Recursion | Supported via RECURSIVE keyword |
| Indexing | Cannot create indexes on CTE results |

---

## 3. Subqueries -- Inline Power

Subqueries are queries nested inside another query. They come in three flavors:

### Scalar Subquery (returns one value)

```sql
-- Find users whose lifetime value exceeds the company average
SELECT 
    user_id,
    lifetime_value
FROM users
WHERE lifetime_value > (
    SELECT AVG(lifetime_value) FROM users
);
```

### Table Subquery (returns a result set -- used in FROM)

```sql
-- Average daily active users per week
SELECT 
    week_start,
    AVG(daily_dau) AS avg_dau
FROM (
    SELECT 
        DATE_TRUNC('week', activity_date) AS week_start,
        activity_date,
        COUNT(DISTINCT user_id) AS daily_dau
    FROM user_activity
    GROUP BY DATE_TRUNC('week', activity_date), activity_date
) daily_stats
GROUP BY week_start;
```

### Existential Subquery (used in WHERE with EXISTS/IN)

```sql
-- Users who purchased in January but NOT in February (churn signal)
SELECT user_id, email
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.user_id = u.user_id 
    AND o.order_date BETWEEN '2024-01-01' AND '2024-01-31'
)
AND NOT EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.user_id = u.user_id 
    AND o.order_date BETWEEN '2024-02-01' AND '2024-02-29'
);
```

> **Critical Insight:** `EXISTS` is almost always preferred over `IN` for correlated checks because EXISTS short-circuits once it finds a match, while IN must evaluate the entire list. This is a common interview follow-up question.

---

## 4. Correlated Subqueries -- Row-by-Row Logic

A correlated subquery references the outer query and executes once per row. They are powerful but potentially expensive.

```sql
-- For each user, find their most recent order amount
SELECT 
    u.user_id,
    u.name,
    (
        SELECT o.amount 
        FROM orders o 
        WHERE o.user_id = u.user_id 
        ORDER BY o.order_date DESC 
        LIMIT 1
    ) AS latest_order_amount
FROM users u;
```

```sql
-- Running comparison: each order vs user's average
SELECT 
    o.order_id,
    o.user_id,
    o.amount,
    o.amount - (
        SELECT AVG(o2.amount) 
        FROM orders o2 
        WHERE o2.user_id = o.user_id 
        AND o2.order_date <= o.order_date
    ) AS diff_from_running_avg
FROM orders o;
```

**When correlated subqueries are appropriate:**
- Small outer result set
- Elegant expression of row-relative logic
- Lateral joins are not available

**When to avoid:**
- Large tables (O(n*m) execution risk)
- When a window function or CTE achieves the same result

> **Critical Insight:** In interviews, if you write a correlated subquery, immediately say: "I'm aware this is O(n*m) in the worst case. In production I'd consider a window function or a CTE with a join, but for clarity I'll write it this way first."

---

## 5. Temp Tables and Table Variables

Temp tables persist for the session and can be indexed. They are not typically used in interview whiteboard settings but are critical for production data pipelines.

```sql
-- Create a temp table for expensive intermediate computation
CREATE TEMPORARY TABLE user_features AS
SELECT 
    user_id,
    COUNT(*) AS total_orders,
    SUM(amount) AS total_spend,
    AVG(amount) AS avg_order_value,
    MAX(order_date) AS last_order_date,
    MIN(order_date) AS first_order_date
FROM orders
GROUP BY user_id;

-- Now index it for fast lookups
CREATE INDEX idx_user_features_spend ON user_features(total_spend);

-- Use it in downstream queries
SELECT 
    uf.*,
    DATEDIFF(CURRENT_DATE, uf.last_order_date) AS days_since_last_order
FROM user_features uf
WHERE uf.total_spend > 1000;
```

**When temp tables beat CTEs:**
- Result is used multiple times across separate queries
- You need to add an index on the intermediate result
- Dataset is large and you want to materialize once
- Multi-step ETL pipelines

---

## 6. CTE vs Subquery vs Temp Table -- The Decision Matrix

| Criterion | CTE | Subquery | Temp Table |
|-----------|-----|----------|------------|
| **Readability** | Excellent | Poor (when nested) | Good |
| **Reuse within query** | Yes (multiple references) | No (must duplicate) | Yes (across queries) |
| **Performance** | Engine-dependent | Engine-dependent | Materialized + indexable |
| **Recursion support** | Yes | No | No |
| **Scope** | Single statement | Single statement | Session |
| **Interview preference** | Default choice | Simple one-offs | Mention for production |
| **Materialization** | Maybe | Maybe | Always |
| **Indexable** | No | No | Yes |
| **Debug-friendly** | High | Low | High |

> **Critical Insight:** In an interview, default to CTEs unless you have a specific reason not to. Then mention the alternatives to show depth: "I'd use a CTE here for readability, but in production if this intermediate result is used across multiple downstream queries, I'd materialize it as a temp table."

---

## 7. Recursive CTEs -- Hierarchical Data

Recursive CTEs solve problems involving hierarchical or graph-like data: org charts, category trees, bill of materials, network paths.

### Structure

```sql
WITH RECURSIVE cte_name AS (
    -- Base case (anchor member)
    SELECT ... FROM table WHERE condition
    
    UNION ALL
    
    -- Recursive case (references cte_name)
    SELECT ... FROM table JOIN cte_name ON ...
)
SELECT * FROM cte_name;
```

### Example: Org Chart -- Find All Reports Under a Manager

```sql
WITH RECURSIVE org_tree AS (
    -- Anchor: start with the target manager
    SELECT 
        employee_id,
        name,
        manager_id,
        1 AS depth,
        name::TEXT AS path
    FROM employees
    WHERE employee_id = 101  -- target manager
    
    UNION ALL
    
    -- Recursive: find direct reports of current level
    SELECT 
        e.employee_id,
        e.name,
        e.manager_id,
        ot.depth + 1,
        ot.path || ' > ' || e.name
    FROM employees e
    INNER JOIN org_tree ot ON e.manager_id = ot.employee_id
    WHERE ot.depth < 10  -- safety limit
)
SELECT 
    employee_id,
    name,
    depth,
    path
FROM org_tree
ORDER BY depth, name;
```

### Example: Category Breadcrumbs (E-commerce)

```sql
WITH RECURSIVE category_path AS (
    SELECT 
        category_id,
        category_name,
        parent_id,
        category_name::TEXT AS breadcrumb
    FROM categories
    WHERE parent_id IS NULL  -- root categories
    
    UNION ALL
    
    SELECT 
        c.category_id,
        c.category_name,
        c.parent_id,
        cp.breadcrumb || ' > ' || c.category_name
    FROM categories c
    INNER JOIN category_path cp ON c.parent_id = cp.category_id
)
SELECT category_id, category_name, breadcrumb
FROM category_path
WHERE category_id NOT IN (SELECT DISTINCT parent_id FROM categories WHERE parent_id IS NOT NULL);
```

**Interview safety checks to mention:**
- Always include a depth limit to prevent infinite loops
- Use UNION ALL (not UNION) for performance unless you need deduplication
- Be aware that circular references can cause infinite recursion

---

## 8. Multi-CTE Query Architecture

For complex analytical problems, structure your CTEs like a data pipeline:

```sql
-- Problem: Find the top product category by revenue for each user segment
WITH user_segments AS (
    -- Step 1: Classify users into segments
    SELECT 
        user_id,
        CASE 
            WHEN lifetime_value >= 1000 THEN 'high_value'
            WHEN lifetime_value >= 200 THEN 'medium_value'
            ELSE 'low_value'
        END AS segment
    FROM users
),
category_revenue AS (
    -- Step 2: Calculate revenue by user-category pair
    SELECT 
        o.user_id,
        p.category,
        SUM(o.amount) AS category_spend
    FROM orders o
    JOIN products p ON o.product_id = p.product_id
    WHERE o.order_date >= '2024-01-01'
    GROUP BY o.user_id, p.category
),
segment_category_revenue AS (
    -- Step 3: Aggregate to segment-category level
    SELECT 
        us.segment,
        cr.category,
        SUM(cr.category_spend) AS total_revenue,
        COUNT(DISTINCT cr.user_id) AS user_count
    FROM user_segments us
    JOIN category_revenue cr ON us.user_id = cr.user_id
    GROUP BY us.segment, cr.category
),
ranked AS (
    -- Step 4: Rank categories within each segment
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY segment ORDER BY total_revenue DESC) AS rn
    FROM segment_category_revenue
)
-- Step 5: Final output
SELECT segment, category, total_revenue, user_count
FROM ranked
WHERE rn = 1;
```

> **Critical Insight:** When writing multi-CTE queries in interviews, announce your plan first: "I'll solve this in 5 steps..." This shows structured thinking and lets the interviewer follow along.

---

## 9. Performance Implications

### CTE Materialization Behavior by Engine

| Database | Default CTE Behavior | Control |
|----------|---------------------|---------|
| PostgreSQL 12+ | Inlined (optimized) | `MATERIALIZED` / `NOT MATERIALIZED` hints |
| PostgreSQL <12 | Always materialized | None |
| MySQL 8.0+ | Materialized | Optimizer decides |
| BigQuery | Materialized | No control |
| Snowflake | Inlined | No explicit control |
| SQL Server | Inlined | No explicit control |
| Redshift | Inlined | No explicit control |

### When CTEs Hurt Performance

```sql
-- BAD: CTE referenced multiple times may be computed multiple times (if inlined)
WITH expensive_calc AS (
    SELECT user_id, complex_function(data) AS result
    FROM large_table  -- 100M rows
)
SELECT * FROM expensive_calc WHERE result > 10
UNION ALL
SELECT * FROM expensive_calc WHERE result < -10;
-- In Postgres 12+, force materialization:
-- WITH expensive_calc AS MATERIALIZED (...)
```

### Subquery Performance Patterns

```sql
-- GOOD: Uncorrelated subquery (executes once)
SELECT * FROM orders WHERE amount > (SELECT AVG(amount) FROM orders);

-- POTENTIALLY BAD: Correlated subquery (executes per row)
SELECT *, (SELECT MAX(amount) FROM orders o2 WHERE o2.user_id = o1.user_id) 
FROM orders o1;
-- Better rewrite:
SELECT o1.*, m.max_amount
FROM orders o1
JOIN (SELECT user_id, MAX(amount) AS max_amount FROM orders GROUP BY user_id) m
    ON o1.user_id = m.user_id;
```

---

## 10. Interview Problems with Solutions

### Problem 1: Running Total with Reset

**Task:** Calculate cumulative revenue per user, resetting each month.

```sql
WITH monthly_orders AS (
    SELECT 
        user_id,
        DATE_TRUNC('month', order_date) AS month,
        order_date,
        amount,
        SUM(amount) OVER (
            PARTITION BY user_id, DATE_TRUNC('month', order_date) 
            ORDER BY order_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS monthly_running_total
    FROM orders
)
SELECT * FROM monthly_orders ORDER BY user_id, order_date;
```

### Problem 2: Find Gaps in Sequential Data

**Task:** Find missing dates in a daily metrics table.

```sql
WITH RECURSIVE date_series AS (
    SELECT MIN(metric_date) AS dt FROM daily_metrics
    UNION ALL
    SELECT dt + INTERVAL '1 day'
    FROM date_series
    WHERE dt < (SELECT MAX(metric_date) FROM daily_metrics)
),
actual_dates AS (
    SELECT DISTINCT metric_date FROM daily_metrics
)
SELECT ds.dt AS missing_date
FROM date_series ds
LEFT JOIN actual_dates ad ON ds.dt = ad.metric_date
WHERE ad.metric_date IS NULL
ORDER BY ds.dt;
```

### Problem 3: Hierarchical Aggregation

**Task:** For each manager, calculate the total headcount of all nested reports.

```sql
WITH RECURSIVE full_tree AS (
    SELECT employee_id, manager_id, employee_id AS root_manager
    FROM employees
    WHERE manager_id IS NOT NULL
    
    UNION ALL
    
    SELECT ft.employee_id, e.manager_id, e.manager_id AS root_manager
    FROM full_tree ft
    JOIN employees e ON ft.manager_id = e.employee_id
    WHERE e.manager_id IS NOT NULL
)
SELECT 
    e.employee_id,
    e.name,
    COUNT(ft.employee_id) AS total_reports
FROM employees e
LEFT JOIN full_tree ft ON ft.root_manager = e.employee_id
GROUP BY e.employee_id, e.name
HAVING COUNT(ft.employee_id) > 0
ORDER BY total_reports DESC;
```

### Problem 4: Session Identification Using LAG + CTE

**Task:** Group user events into sessions (30-min inactivity gap).

```sql
WITH time_gaps AS (
    SELECT 
        user_id,
        event_time,
        event_type,
        LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) AS prev_event_time,
        CASE 
            WHEN EXTRACT(EPOCH FROM (event_time - LAG(event_time) OVER (
                PARTITION BY user_id ORDER BY event_time))) > 1800 
            THEN 1 
            ELSE 0 
        END AS new_session_flag
    FROM user_events
),
session_ids AS (
    SELECT 
        *,
        SUM(new_session_flag) OVER (
            PARTITION BY user_id ORDER BY event_time
        ) AS session_id
    FROM time_gaps
)
SELECT 
    user_id,
    session_id,
    MIN(event_time) AS session_start,
    MAX(event_time) AS session_end,
    COUNT(*) AS events_in_session,
    EXTRACT(EPOCH FROM MAX(event_time) - MIN(event_time)) / 60 AS session_duration_min
FROM session_ids
GROUP BY user_id, session_id;
```

---

## 11. Interview Talking Points

> "I default to CTEs for analytical queries because they make the logical flow explicit. Each CTE represents a transformation step -- isolate the cohort, compute metrics, rank, then filter. This mirrors how I think about the problem."

> "For correlated subqueries, I always evaluate whether a window function can replace it. Window functions are set-based and typically O(n log n) versus the potential O(n*m) of a correlated subquery."

> "I'm mindful that CTE materialization varies by engine. In Postgres 12+, CTEs are inlined by default, so if I reference a CTE multiple times and it's expensive, I'll add the MATERIALIZED hint. In production pipelines with very large intermediate results, I'd use temp tables with indexes."

> "Recursive CTEs are my go-to for hierarchical data in interviews. I always include three things: a clear anchor member, the recursive join, and a depth safety limit to prevent infinite loops."

---

## 12. Common Interview Mistakes

| Mistake | Why It's Wrong | Correct Approach |
|---------|---------------|-----------------|
| Deeply nested subqueries (3+ levels) | Unreadable; interviewer cannot follow logic | Refactor into named CTEs |
| Using IN with a large subquery | Performance risk; does not handle NULLs well | Use EXISTS or a JOIN |
| Forgetting RECURSIVE keyword | Syntax error in recursive CTEs | Always specify WITH RECURSIVE |
| No depth limit in recursive CTE | Risk of infinite loop | Add WHERE depth < N |
| Assuming CTEs are always materialized | May be inlined, causing repeated computation | Know your engine's behavior |
| Using UNION instead of UNION ALL in recursion | Unnecessary dedup overhead; can mask bugs | Default to UNION ALL |
| Correlated subquery on large tables | O(n*m) performance disaster | Rewrite as JOIN or window function |
| Not aliasing subqueries in FROM | Syntax error in most engines | Always alias derived tables |
| Referencing CTE in a different statement | CTEs are scoped to one statement only | Use temp table if needed across statements |
| Writing everything in one flat query | Shows lack of structured thinking | Decompose into logical steps |

---

## 13. Rapid-Fire Q&A

**Q1: Can a CTE reference another CTE defined earlier in the same WITH block?**  
A: Yes. CTEs are evaluated in order, and each can reference any previously defined CTE.

**Q2: Can you UPDATE using a CTE?**  
A: Yes, in PostgreSQL and SQL Server. Use `WITH ... UPDATE ... FROM cte_name`. Not supported in MySQL.

**Q3: What happens if a recursive CTE has no termination condition?**  
A: Infinite loop. Most engines have a max recursion depth (e.g., SQL Server defaults to 100, Postgres has no hard limit but will eventually exhaust memory).

**Q4: EXISTS vs IN -- when does IN win?**  
A: When the subquery returns a small, static list of literals. The optimizer can hash a small IN list efficiently.

**Q5: Can you use ORDER BY inside a CTE?**  
A: Syntactically yes in most engines, but it has no guaranteed effect on the final result unless the outer query also specifies ORDER BY.

**Q6: What is a lateral join and how does it relate to correlated subqueries?**  
A: A LATERAL join (Postgres) or CROSS APPLY (SQL Server) is the set-returning equivalent of a correlated subquery. It can return multiple rows per outer row.

**Q7: Can you nest CTEs (CTE within a CTE)?**  
A: No. CTEs cannot contain their own WITH clause (except recursive self-reference). Flatten into a single WITH block with multiple named CTEs.

**Q8: How do you debug a long CTE chain?**  
A: Replace the final SELECT with `SELECT * FROM cte_name_N` to inspect each intermediate step.

**Q9: Are temp tables visible to other sessions?**  
A: No. Temporary tables are session-scoped and invisible to other connections.

**Q10: When would you choose a VIEW over a CTE?**  
A: When the logic needs to be reused across multiple queries or by multiple users. Views persist in the schema; CTEs are one-shot.

---

## 14. Summary Cheat Sheet

```
+------------------------------------------------------------------+
|  CTEs, SUBQUERIES & QUERY DECOMPOSITION -- CHEAT SHEET           |
+------------------------------------------------------------------+
|                                                                    |
|  CTE (WITH ... AS):                                               |
|    - Named, readable, reusable within one statement                |
|    - Use for multi-step analytics (default choice in interviews)   |
|    - Recursive variant solves hierarchical data                    |
|    - Materialization is engine-dependent                           |
|                                                                    |
|  SUBQUERY:                                                         |
|    - Scalar (one value), Table (in FROM), Existential (WHERE)      |
|    - Good for simple one-off filters                               |
|    - Correlated = executes per outer row (watch performance)       |
|    - EXISTS > IN for correlated checks                             |
|                                                                    |
|  TEMP TABLE:                                                       |
|    - Session-scoped, indexable, materialized                       |
|    - Use when intermediate result is reused across queries         |
|    - Appropriate for ETL pipelines and large datasets              |
|                                                                    |
|  RECURSIVE CTE PATTERN:                                            |
|    WITH RECURSIVE cte AS (                                         |
|      <anchor>                                                      |
|      UNION ALL                                                     |
|      <recursive member with depth limit>                           |
|    )                                                               |
|                                                                    |
|  INTERVIEW DEFAULTS:                                               |
|    1. Narrate your decomposition plan before writing               |
|    2. Use CTEs as your primary structuring tool                    |
|    3. Mention alternatives (temp table, subquery) for depth        |
|    4. Always alias derived tables                                  |
|    5. Call out performance trade-offs proactively                   |
|                                                                    |
+------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Topics](README.md) collection -- Topic 3 of 45*
