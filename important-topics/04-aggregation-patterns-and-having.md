# Topic 4: Aggregation Patterns & HAVING

> **Data Science Interview -- Deep Dive**  
> Mastering GROUP BY mechanics, multi-level aggregation, advanced grouping sets, conditional aggregation with CASE, pivot patterns, and the subtle distinction between WHERE and HAVING that trips up even experienced candidates.

---

## Table of Contents

1. [GROUP BY Mechanics -- How It Actually Works](#1-group-by-mechanics--how-it-actually-works)
2. [WHERE vs HAVING -- The Critical Distinction](#2-where-vs-having--the-critical-distinction)
3. [Aggregate Functions -- Beyond COUNT and SUM](#3-aggregate-functions--beyond-count-and-sum)
4. [Multi-Level Aggregation](#4-multi-level-aggregation)
5. [GROUPING SETS, CUBE, and ROLLUP](#5-grouping-sets-cube-and-rollup)
6. [Conditional Aggregation -- CASE Inside Aggregates](#6-conditional-aggregation--case-inside-aggregates)
7. [Pivot and Unpivot Patterns](#7-pivot-and-unpivot-patterns)
8. [Common Aggregation Pitfalls](#8-common-aggregation-pitfalls)
9. [Interview Problems with Solutions](#9-interview-problems-with-solutions)
10. [Interview Talking Points](#10-interview-talking-points)
11. [Common Interview Mistakes](#11-common-interview-mistakes)
12. [Rapid-Fire Q&A](#12-rapid-fire-qa)
13. [Summary Cheat Sheet](#13-summary-cheat-sheet)

---

## 1. GROUP BY Mechanics -- How It Actually Works

GROUP BY conceptually does three things in sequence:

1. **Partitions** rows into groups based on the grouping columns
2. **Collapses** each group into a single output row
3. **Applies** aggregate functions across rows within each group

### SQL Logical Execution Order

```
FROM / JOIN       -- Build the working set
WHERE             -- Filter individual rows (before grouping)
GROUP BY          -- Partition into groups
HAVING            -- Filter groups (after grouping)
SELECT            -- Evaluate expressions and aggregates
DISTINCT          -- Remove duplicate result rows
ORDER BY          -- Sort final output
LIMIT / OFFSET    -- Restrict row count
```

> **Critical Insight:** The execution order means you CANNOT reference an alias defined in SELECT inside your WHERE or HAVING clause in standard SQL. Some engines (MySQL, BigQuery) allow this as a convenience, but it is non-standard.

### What Must Appear in GROUP BY

Every column in SELECT that is NOT inside an aggregate function MUST appear in GROUP BY (in standard SQL).

```sql
-- CORRECT
SELECT department, job_title, AVG(salary)
FROM employees
GROUP BY department, job_title;

-- WRONG (job_title not in GROUP BY)
SELECT department, job_title, AVG(salary)
FROM employees
GROUP BY department;
-- Error: column "job_title" must appear in GROUP BY clause
```

**Exception:** MySQL's permissive mode allows non-aggregated columns not in GROUP BY, but it returns indeterminate values. Never rely on this.

---

## 2. WHERE vs HAVING -- The Critical Distinction

| Aspect | WHERE | HAVING |
|--------|-------|--------|
| **Timing** | Before GROUP BY | After GROUP BY |
| **Operates on** | Individual rows | Aggregated groups |
| **Can use aggregates** | No | Yes |
| **Performance** | Reduces data before grouping (faster) | Filters after full aggregation |
| **Use case** | Pre-filter raw data | Filter based on aggregate results |

```sql
-- WHERE: Filter to only 2024 orders BEFORE aggregating
-- HAVING: Then keep only users with more than 5 orders
SELECT 
    user_id,
    COUNT(*) AS order_count,
    SUM(amount) AS total_spend
FROM orders
WHERE order_date >= '2024-01-01'   -- row-level filter (reduces work)
GROUP BY user_id
HAVING COUNT(*) > 5;              -- group-level filter (post-aggregation)
```

> **Critical Insight:** A common interview question is "Can you replace HAVING with WHERE?" The answer: Only if the filter condition does not involve an aggregate. HAVING is ONLY necessary when you need to filter on the result of COUNT, SUM, AVG, MAX, MIN, etc. Put as much filtering in WHERE as possible for performance.

### Anti-Pattern: HAVING without Aggregation

```sql
-- BAD: Using HAVING for non-aggregate filter
SELECT department, AVG(salary)
FROM employees
GROUP BY department
HAVING department != 'Intern';  -- This works but is wasteful

-- GOOD: Use WHERE for non-aggregate filter
SELECT department, AVG(salary)
FROM employees
WHERE department != 'Intern'    -- Filters BEFORE grouping = less work
GROUP BY department;
```

---

## 3. Aggregate Functions -- Beyond COUNT and SUM

### Standard Aggregates

| Function | Behavior with NULLs | Notes |
|----------|---------------------|-------|
| `COUNT(*)` | Counts all rows including NULLs | Row count |
| `COUNT(column)` | Ignores NULLs | Non-null count |
| `COUNT(DISTINCT col)` | Ignores NULLs, deduplicates | Cardinality |
| `SUM(column)` | Ignores NULLs | Returns NULL if all values NULL |
| `AVG(column)` | Ignores NULLs | Does NOT count NULLs in denominator |
| `MIN / MAX` | Ignores NULLs | Works on strings too (lexicographic) |

### Statistical Aggregates

```sql
SELECT 
    category,
    COUNT(*) AS n,
    AVG(price) AS mean_price,
    STDDEV_SAMP(price) AS std_price,       -- sample standard deviation
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY price) AS median_price,
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY price) AS p25,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY price) AS p75,
    CORR(price, quantity) AS price_qty_correlation
FROM products
GROUP BY category;
```

### Array and String Aggregates

```sql
-- Collect all tags for each product
SELECT 
    product_id,
    ARRAY_AGG(tag ORDER BY tag) AS all_tags,               -- PostgreSQL
    STRING_AGG(tag, ', ' ORDER BY tag) AS tags_csv         -- PostgreSQL
FROM product_tags
GROUP BY product_id;
```

---

## 4. Multi-Level Aggregation

Many interview questions require aggregating at one level, then re-aggregating at a higher level.

```sql
-- Problem: What's the average number of daily orders per user per month?
-- Step 1: Count orders per user per day
-- Step 2: Average across days within each month

WITH daily_counts AS (
    SELECT 
        user_id,
        DATE_TRUNC('month', order_date) AS month,
        order_date::DATE AS day,
        COUNT(*) AS orders_that_day
    FROM orders
    GROUP BY user_id, DATE_TRUNC('month', order_date), order_date::DATE
)
SELECT 
    month,
    AVG(orders_that_day) AS avg_daily_orders_per_user,
    STDDEV(orders_that_day) AS std_daily_orders
FROM daily_counts
GROUP BY month
ORDER BY month;
```

```sql
-- Problem: For each category, what percentage of total revenue does the top product represent?
WITH product_revenue AS (
    SELECT 
        p.category,
        p.product_id,
        p.product_name,
        SUM(o.amount) AS revenue
    FROM orders o
    JOIN products p ON o.product_id = p.product_id
    GROUP BY p.category, p.product_id, p.product_name
),
category_totals AS (
    SELECT 
        category,
        SUM(revenue) AS total_category_revenue,
        MAX(revenue) AS top_product_revenue
    FROM product_revenue
    GROUP BY category
)
SELECT 
    category,
    total_category_revenue,
    top_product_revenue,
    ROUND(100.0 * top_product_revenue / total_category_revenue, 1) AS top_product_pct
FROM category_totals
ORDER BY top_product_pct DESC;
```

---

## 5. GROUPING SETS, CUBE, and ROLLUP

These generate multiple levels of aggregation in a single query -- essential for reporting and dashboards.

### GROUPING SETS -- Explicit Control

```sql
-- Revenue by (country, category), by country alone, and grand total
SELECT 
    COALESCE(country, 'ALL') AS country,
    COALESCE(category, 'ALL') AS category,
    SUM(revenue) AS total_revenue,
    COUNT(DISTINCT user_id) AS unique_buyers
FROM sales
GROUP BY GROUPING SETS (
    (country, category),   -- detailed level
    (country),             -- country subtotal
    ()                     -- grand total
)
ORDER BY country, category;
```

### ROLLUP -- Hierarchical Subtotals

```sql
-- ROLLUP(a, b, c) generates: (a,b,c), (a,b), (a), ()
SELECT 
    region,
    country,
    city,
    SUM(sales) AS total_sales
FROM store_sales
GROUP BY ROLLUP(region, country, city);
```

### CUBE -- All Combinations

```sql
-- CUBE(a, b) generates: (a,b), (a), (b), ()
SELECT 
    COALESCE(platform, 'ALL') AS platform,
    COALESCE(device_type, 'ALL') AS device_type,
    COUNT(DISTINCT user_id) AS users,
    SUM(revenue) AS revenue
FROM sessions
GROUP BY CUBE(platform, device_type);
```

### GROUPING() Function -- Disambiguate NULLs

```sql
SELECT 
    CASE WHEN GROUPING(country) = 1 THEN 'ALL' ELSE country END AS country,
    CASE WHEN GROUPING(category) = 1 THEN 'ALL' ELSE category END AS category,
    SUM(revenue) AS revenue
FROM sales
GROUP BY ROLLUP(country, category);
```

> **Critical Insight:** Without `GROUPING()`, you cannot distinguish between a NULL in the data and a NULL introduced by ROLLUP/CUBE. Always use GROUPING() in production queries with these operators.

---

## 6. Conditional Aggregation -- CASE Inside Aggregates

This is one of the most powerful patterns for data scientists. It lets you create multiple filtered metrics in a single pass.

```sql
-- Compute multiple conversion rates in one query
SELECT 
    acquisition_channel,
    COUNT(*) AS total_users,
    COUNT(CASE WHEN first_purchase_date IS NOT NULL THEN 1 END) AS converted_users,
    COUNT(CASE WHEN first_purchase_date <= signup_date + INTERVAL '7 days' THEN 1 END) AS converted_7d,
    COUNT(CASE WHEN first_purchase_date <= signup_date + INTERVAL '30 days' THEN 1 END) AS converted_30d,
    ROUND(100.0 * COUNT(CASE WHEN first_purchase_date IS NOT NULL THEN 1 END) / COUNT(*), 2) AS conversion_rate_pct
FROM users
GROUP BY acquisition_channel;
```

```sql
-- Pivot-like behavior: revenue by quarter in columns
SELECT 
    product_category,
    SUM(CASE WHEN quarter = 'Q1' THEN revenue ELSE 0 END) AS q1_revenue,
    SUM(CASE WHEN quarter = 'Q2' THEN revenue ELSE 0 END) AS q2_revenue,
    SUM(CASE WHEN quarter = 'Q3' THEN revenue ELSE 0 END) AS q3_revenue,
    SUM(CASE WHEN quarter = 'Q4' THEN revenue ELSE 0 END) AS q4_revenue,
    SUM(revenue) AS annual_revenue
FROM quarterly_sales
GROUP BY product_category;
```

```sql
-- Boolean aggregation with FILTER clause (Postgres-specific, cleaner)
SELECT 
    department,
    COUNT(*) AS total_employees,
    COUNT(*) FILTER (WHERE salary > 100000) AS high_earners,
    AVG(salary) FILTER (WHERE tenure_years >= 3) AS avg_salary_tenured
FROM employees
GROUP BY department;
```

---

## 7. Pivot and Unpivot Patterns

### Manual Pivot (Cross-tab) Using Conditional Aggregation

```sql
-- Transform: one row per (user, month) -> one row per user with month columns
SELECT 
    user_id,
    MAX(CASE WHEN month = '2024-01' THEN spend END) AS jan_spend,
    MAX(CASE WHEN month = '2024-02' THEN spend END) AS feb_spend,
    MAX(CASE WHEN month = '2024-03' THEN spend END) AS mar_spend
FROM (
    SELECT user_id, TO_CHAR(order_date, 'YYYY-MM') AS month, SUM(amount) AS spend
    FROM orders
    WHERE order_date >= '2024-01-01' AND order_date < '2024-04-01'
    GROUP BY user_id, TO_CHAR(order_date, 'YYYY-MM')
) monthly
GROUP BY user_id;
```

### Unpivot (Columns to Rows)

```sql
-- Transform wide format to long format using UNION ALL
SELECT user_id, 'jan' AS month, jan_spend AS spend FROM user_monthly WHERE jan_spend IS NOT NULL
UNION ALL
SELECT user_id, 'feb' AS month, feb_spend AS spend FROM user_monthly WHERE feb_spend IS NOT NULL
UNION ALL
SELECT user_id, 'mar' AS month, mar_spend AS spend FROM user_monthly WHERE mar_spend IS NOT NULL;

-- Or using LATERAL/CROSS JOIN UNNEST (Postgres)
SELECT u.user_id, x.month, x.spend
FROM user_monthly u
CROSS JOIN LATERAL (
    VALUES ('jan', u.jan_spend), ('feb', u.feb_spend), ('mar', u.mar_spend)
) AS x(month, spend)
WHERE x.spend IS NOT NULL;
```

---

## 8. Common Aggregation Pitfalls

### The Double-Counting Trap

```sql
-- BAD: Joining orders to payments creates a fan-out
-- If one order has 3 payments, the order amount is counted 3 times
SELECT user_id, SUM(o.amount) AS total_orders  -- WRONG! Inflated
FROM orders o
JOIN payments p ON o.order_id = p.order_id
GROUP BY user_id;

-- GOOD: Aggregate BEFORE joining, or use DISTINCT
SELECT user_id, SUM(DISTINCT o.amount) AS total_orders  -- If amounts are unique
FROM orders o
JOIN payments p ON o.order_id = p.order_id
GROUP BY user_id;

-- BETTER: Pre-aggregate in a CTE
WITH order_totals AS (
    SELECT user_id, SUM(amount) AS total_orders
    FROM orders GROUP BY user_id
)
SELECT ot.user_id, ot.total_orders, COUNT(p.payment_id) AS payment_count
FROM order_totals ot
JOIN orders o ON ot.user_id = o.user_id
JOIN payments p ON o.order_id = p.order_id
GROUP BY ot.user_id, ot.total_orders;
```

### AVG Ignores NULLs (Silently)

```sql
-- These give DIFFERENT results:
SELECT AVG(rating) FROM reviews;                    -- ignores NULLs
SELECT SUM(rating) / COUNT(*) FROM reviews;         -- includes NULLs in denominator (lower)
SELECT SUM(rating) / COUNT(rating) FROM reviews;    -- same as AVG (ignores NULLs)

-- If you want NULLs treated as 0:
SELECT AVG(COALESCE(rating, 0)) FROM reviews;
```

---

## 9. Interview Problems with Solutions

### Problem 1: Retention by Cohort Week

**Task:** For each weekly signup cohort, what percentage of users were active in their second week?

```sql
WITH cohorts AS (
    SELECT 
        user_id,
        DATE_TRUNC('week', signup_date) AS cohort_week
    FROM users
),
week2_active AS (
    SELECT DISTINCT 
        c.user_id,
        c.cohort_week
    FROM cohorts c
    JOIN user_activity a ON c.user_id = a.user_id
    WHERE a.activity_date >= c.cohort_week + INTERVAL '7 days'
      AND a.activity_date < c.cohort_week + INTERVAL '14 days'
)
SELECT 
    c.cohort_week,
    COUNT(DISTINCT c.user_id) AS cohort_size,
    COUNT(DISTINCT w.user_id) AS retained_week2,
    ROUND(100.0 * COUNT(DISTINCT w.user_id) / COUNT(DISTINCT c.user_id), 2) AS retention_pct
FROM cohorts c
LEFT JOIN week2_active w ON c.user_id = w.user_id AND c.cohort_week = w.cohort_week
GROUP BY c.cohort_week
ORDER BY c.cohort_week;
```

### Problem 2: Top N Per Group

**Task:** Find the top 3 products by revenue in each category.

```sql
WITH product_revenue AS (
    SELECT 
        p.category,
        p.product_name,
        SUM(o.quantity * o.unit_price) AS revenue,
        ROW_NUMBER() OVER (PARTITION BY p.category ORDER BY SUM(o.quantity * o.unit_price) DESC) AS rn
    FROM order_items o
    JOIN products p ON o.product_id = p.product_id
    GROUP BY p.category, p.product_name
)
SELECT category, product_name, revenue
FROM product_revenue
WHERE rn <= 3
ORDER BY category, rn;
```

### Problem 3: Consecutive Days Streak

**Task:** Find each user's longest consecutive active days streak.

```sql
WITH numbered AS (
    SELECT 
        user_id,
        activity_date,
        activity_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY activity_date))::INT * INTERVAL '1 day' AS grp
    FROM (SELECT DISTINCT user_id, activity_date::DATE AS activity_date FROM user_activity) d
),
streaks AS (
    SELECT 
        user_id,
        grp,
        COUNT(*) AS streak_length,
        MIN(activity_date) AS streak_start,
        MAX(activity_date) AS streak_end
    FROM numbered
    GROUP BY user_id, grp
)
SELECT 
    user_id,
    MAX(streak_length) AS longest_streak
FROM streaks
GROUP BY user_id
ORDER BY longest_streak DESC;
```

---

## 10. Interview Talking Points

> "I always put row-level filters in WHERE and reserve HAVING exclusively for conditions on aggregated values. This is not just a style choice -- it is a performance choice because WHERE reduces the data before the expensive GROUP BY operation."

> "When I see a question that needs subtotals and grand totals alongside detailed rows, I reach for GROUPING SETS or ROLLUP rather than writing separate queries with UNION ALL. It is a single scan of the data versus multiple scans."

> "Conditional aggregation with CASE is my go-to for building feature matrices in a single pass. Rather than joining multiple subqueries that each filter differently, I compute all the variants simultaneously inside SUM or COUNT."

> "A common mistake I've seen in production is double-counting after a join fan-out. I always verify the grain of my intermediate result. If joining a fact table to a dimension table inflates rows, I pre-aggregate in a CTE before the join."

---

## 11. Common Interview Mistakes

| Mistake | Why It's Wrong | Correct Approach |
|---------|---------------|-----------------|
| Filtering on aggregate in WHERE | WHERE cannot access aggregates | Use HAVING for aggregate filters |
| Non-aggregate column not in GROUP BY | Undefined behavior (error in strict engines) | Include all non-aggregate columns in GROUP BY |
| Using HAVING for row-level filters | Works but wastes resources (groups all data first) | Move non-aggregate conditions to WHERE |
| COUNT(*) when you mean COUNT(column) | COUNT(*) counts NULLs; COUNT(col) does not | Be explicit about NULL handling |
| Forgetting AVG ignores NULLs | Denominator only includes non-NULL rows | Use COALESCE if NULLs should be zero |
| Double-counting from fan-out joins | Revenue inflated by payment/shipment rows | Pre-aggregate or use DISTINCT |
| DISTINCT inside COUNT on large data | Expensive operation; often unnecessary | Verify you actually need distinct count |
| Missing GROUPING() with ROLLUP/CUBE | Cannot distinguish data NULLs from grouping NULLs | Always use GROUPING() function |
| Aggregating before filtering | Processing unnecessary rows | Filter in WHERE first, then GROUP BY |
| Using ORDER BY in subquery | No guaranteed effect on outer query | Apply ORDER BY only in the outermost query |

---

## 12. Rapid-Fire Q&A

**Q1: Can HAVING reference a SELECT alias?**  
A: In standard SQL, no. In MySQL and BigQuery, yes (as a convenience). Always use the full expression in HAVING for portability.

**Q2: What does GROUP BY 1, 2 mean?**  
A: Group by the first and second columns in the SELECT list (positional reference). Common in ad-hoc queries but discouraged in production for readability.

**Q3: Can you GROUP BY a column not in SELECT?**  
A: Yes. You can group by a column and only show aggregates without displaying the grouping column itself.

**Q4: What is the difference between COUNT(1) and COUNT(*)?**  
A: Functionally identical in all modern engines. Both count rows including NULLs. Use COUNT(*) for clarity.

**Q5: How does GROUP BY handle NULLs?**  
A: All NULLs are treated as a single group. NULL = NULL for grouping purposes (unlike in WHERE comparisons).

**Q6: Can you use window functions with GROUP BY?**  
A: Yes. Window functions execute AFTER GROUP BY, so they operate on the grouped result set.

**Q7: What does ROLLUP(a, b, c) produce vs CUBE(a, b, c)?**  
A: ROLLUP produces (a,b,c), (a,b), (a), () -- 4 grouping sets (hierarchical). CUBE produces all 2^3 = 8 combinations.

**Q8: How do you compute a running percentage of total?**  
A: Use `SUM(amount) / SUM(SUM(amount)) OVER ()` -- the nested SUM is the grouped aggregate and the outer SUM OVER() is the window total.

**Q9: Can HAVING reference a different table via subquery?**  
A: Yes. `HAVING COUNT(*) > (SELECT AVG(cnt) FROM some_table)` is valid.

**Q10: What is the performance impact of COUNT(DISTINCT) on large data?**  
A: Significant. It requires tracking all unique values (hash set). For approximate distinct counts at scale, use HyperLogLog (APPROX_COUNT_DISTINCT in BigQuery, HLL in Redshift).

---

## 13. Summary Cheat Sheet

```
+------------------------------------------------------------------+
|  AGGREGATION PATTERNS & HAVING -- CHEAT SHEET                    |
+------------------------------------------------------------------+
|                                                                    |
|  EXECUTION ORDER:                                                  |
|    FROM -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY       |
|    (Filter rows first, group second, filter groups third)          |
|                                                                    |
|  WHERE vs HAVING:                                                  |
|    WHERE  = row-level filter (before grouping) -- no aggregates    |
|    HAVING = group-level filter (after grouping) -- uses aggregates |
|    Rule: Put everything possible in WHERE for performance          |
|                                                                    |
|  CONDITIONAL AGGREGATION:                                          |
|    SUM(CASE WHEN condition THEN value ELSE 0 END)                  |
|    COUNT(CASE WHEN condition THEN 1 END)  -- NULLs not counted     |
|    Postgres: COUNT(*) FILTER (WHERE condition)                     |
|                                                                    |
|  GROUPING SETS HIERARCHY:                                          |
|    GROUPING SETS((a,b), (a), ())  -- explicit                      |
|    ROLLUP(a, b)                   -- hierarchical subtotals        |
|    CUBE(a, b)                     -- all 2^n combinations          |
|    GROUPING(col) = 1 means "this is a subtotal row"                |
|                                                                    |
|  CRITICAL PITFALLS:                                                |
|    - AVG() silently ignores NULLs (denominator shrinks)            |
|    - JOIN fan-out inflates SUM/COUNT (pre-aggregate or DISTINCT)   |
|    - COUNT(*) includes NULLs; COUNT(col) excludes them             |
|    - Non-aggregate columns MUST be in GROUP BY (strict mode)       |
|                                                                    |
|  INTERVIEW STRATEGY:                                               |
|    1. State the grain of your GROUP BY                             |
|    2. Use conditional aggregation for multi-metric queries          |
|    3. Call out NULL handling explicitly                             |
|    4. Mention GROUPING SETS for subtotal requirements              |
|                                                                    |
+------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Topics](README.md) collection -- Topic 4 of 45*
