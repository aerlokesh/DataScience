# 🎯 Topic 2: Window Functions Deep Dive

> **Data Science Interview — Deep Dive**
> The single most differentiating SQL skill in DS interviews. Master ROW_NUMBER, RANK, DENSE_RANK, LAG, LEAD, running totals, moving averages, and percentiles — with the mental models to decompose any problem.

---

## Table of Contents

1. [Window Function Anatomy](#window-function-anatomy)
2. [Ranking Functions](#ranking-functions)
3. [Offset Functions](#offset-functions)
4. [Aggregate Window Functions](#aggregate-window-functions)
5. [Frame Specifications](#frame-specifications)
6. [NTILE for Percentile Bucketing](#ntile-for-percentile-bucketing)
7. [Common Patterns](#common-patterns)
8. [Performance Considerations](#performance-considerations)
9. [Interview Problems with Solutions](#interview-problems-with-solutions)
10. [Real-World Scenarios](#real-world-scenarios)
11. [Interview Talking Points](#interview-talking-points)
12. [Common Interview Mistakes](#common-interview-mistakes)
13. [Rapid-Fire Q&A](#rapid-fire-qa)
14. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## Window Function Anatomy

A window function performs a calculation across a set of rows that are somehow related to the current row — **without collapsing** the result into a single output row (unlike GROUP BY).

### Full Syntax

```sql
function_name(expression) OVER (
    [PARTITION BY partition_expression, ...]
    [ORDER BY sort_expression [ASC|DESC], ...]
    [frame_clause]
)
```

### Component Breakdown

| Component | Purpose | Required? |
|-----------|---------|-----------|
| `PARTITION BY` | Divides rows into groups (like GROUP BY but without collapsing) | Optional |
| `ORDER BY` | Defines the logical ordering within each partition | Depends on function |
| Frame clause | Defines which rows relative to current row to include | Optional (has defaults) |

### Mental Model: The Three Layers

```
┌─────────────────────────────────────────────────────┐
│  Layer 1: PARTITION BY  →  Splits data into groups  │
│  Layer 2: ORDER BY      →  Sorts within each group  │
│  Layer 3: FRAME         →  Slides a "window" across │
└─────────────────────────────────────────────────────┘
```

### Key Insight: Execution Order

Window functions execute **after** WHERE, GROUP BY, and HAVING — but **before** ORDER BY and LIMIT. This means:

```sql
SELECT
    user_id,
    revenue,
    RANK() OVER (ORDER BY revenue DESC) AS revenue_rank
FROM orders
WHERE order_date >= '2024-01-01'
ORDER BY revenue_rank
LIMIT 10;
```

The WHERE filters first, then the window function computes rank, then ORDER BY sorts, then LIMIT caps output.

---

## Ranking Functions

### Comparison Table

| Function | Ties | Gaps | Example (scores: 95, 90, 90, 85) |
|----------|------|------|-----------------------------------|
| `ROW_NUMBER()` | Breaks arbitrarily | No | 1, 2, 3, 4 |
| `RANK()` | Same rank | Yes | 1, 2, 2, 4 |
| `DENSE_RANK()` | Same rank | No | 1, 2, 2, 3 |

### When to Use Which

**ROW_NUMBER()** — Use when you need exactly N rows per group (e.g., "latest order per customer"):
```sql
-- Get the most recent order per customer
WITH ranked AS (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders
)
SELECT * FROM ranked WHERE rn = 1;
```

**RANK()** — Use when ties matter and you want to reflect the "true position" (e.g., competition ranking):
```sql
-- Sales leaderboard where ties skip positions
SELECT
    salesperson,
    total_sales,
    RANK() OVER (ORDER BY total_sales DESC) AS position
FROM sales_summary;
```

**DENSE_RANK()** — Use when you need continuous rank values without gaps (e.g., "top 3 salary levels"):
```sql
-- Find employees in the top 3 salary tiers
WITH salary_ranks AS (
    SELECT *,
        DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_tier
    FROM employees
)
SELECT * FROM salary_ranks WHERE salary_tier <= 3;
```

### Critical Interview Distinction

> If the question says "top N **rows**" → ROW_NUMBER  
> If the question says "top N **values/levels**" → DENSE_RANK  
> If the question says "rank with standard competition ranking" → RANK

---

## Offset Functions

Offset functions let you access values from other rows **without a self-join**.

### LAG and LEAD

```sql
LAG(expression, offset, default)  OVER (PARTITION BY ... ORDER BY ...)
LEAD(expression, offset, default) OVER (PARTITION BY ... ORDER BY ...)
```

| Function | Direction | Use Case |
|----------|-----------|----------|
| `LAG(col, n)` | Looks **back** n rows | Previous value, period-over-period |
| `LEAD(col, n)` | Looks **forward** n rows | Next value, time-to-next-event |

**Example: Month-over-Month Growth**
```sql
SELECT
    month,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY month) AS prev_month_revenue,
    ROUND(100.0 * (revenue - LAG(revenue, 1) OVER (ORDER BY month))
        / LAG(revenue, 1) OVER (ORDER BY month), 2) AS mom_growth_pct
FROM monthly_revenue;
```

### FIRST_VALUE, LAST_VALUE, NTH_VALUE

```sql
FIRST_VALUE(expression) OVER (PARTITION BY ... ORDER BY ... frame)
LAST_VALUE(expression)  OVER (PARTITION BY ... ORDER BY ... frame)
NTH_VALUE(expression, n) OVER (PARTITION BY ... ORDER BY ... frame)
```

**Critical Gotcha with LAST_VALUE:**
```sql
-- WRONG: Default frame is ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
SELECT LAST_VALUE(price) OVER (ORDER BY date) AS last_price  -- Only sees up to current row!

-- CORRECT: Extend frame to end of partition
SELECT LAST_VALUE(price) OVER (
    ORDER BY date
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
) AS last_price
```

**Example: Difference from First Value**
```sql
-- How far is each day's stock price from its opening price that month?
SELECT
    trade_date,
    close_price,
    FIRST_VALUE(close_price) OVER (
        PARTITION BY DATE_TRUNC('month', trade_date)
        ORDER BY trade_date
    ) AS month_open,
    close_price - FIRST_VALUE(close_price) OVER (
        PARTITION BY DATE_TRUNC('month', trade_date)
        ORDER BY trade_date
    ) AS diff_from_open
FROM stock_prices;
```

---

## Aggregate Window Functions

Any standard aggregate can be used as a window function. The key difference: **the rows are not collapsed**.

### Running Total

```sql
SELECT
    order_date,
    amount,
    SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM orders;
```

### Running Count

```sql
SELECT
    user_id,
    login_date,
    COUNT(*) OVER (PARTITION BY user_id ORDER BY login_date) AS cumulative_logins
FROM logins;
```

### Moving Average (7-day)

```sql
SELECT
    date,
    daily_revenue,
    AVG(daily_revenue) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS seven_day_moving_avg
FROM daily_metrics;
```

### Running Max / Min

```sql
SELECT
    date,
    stock_price,
    MAX(stock_price) OVER (ORDER BY date) AS all_time_high,
    stock_price - MAX(stock_price) OVER (ORDER BY date) AS drawdown
FROM prices;
```

### Partition-Level Aggregates (No Frame)

```sql
-- Compare each employee's salary to their department average
SELECT
    employee_name,
    department,
    salary,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg,
    salary - AVG(salary) OVER (PARTITION BY department) AS diff_from_avg
FROM employees;
```

---

## Frame Specifications

The frame clause defines the **sliding window** of rows relative to the current row.

### Syntax

```sql
{ ROWS | RANGE | GROUPS } BETWEEN frame_start AND frame_end
```

### Frame Boundaries

| Boundary | Meaning |
|----------|---------|
| `UNBOUNDED PRECEDING` | First row of partition |
| `n PRECEDING` | n rows/values before current |
| `CURRENT ROW` | The current row |
| `n FOLLOWING` | n rows/values after current |
| `UNBOUNDED FOLLOWING` | Last row of partition |

### ROWS vs RANGE vs GROUPS

| Mode | Unit | Tie Handling |
|------|------|--------------|
| `ROWS` | Physical row positions | Each row is separate |
| `RANGE` | Logical value ranges | Ties treated as same position |
| `GROUPS` | Peer groups (distinct ORDER BY values) | Groups of tied rows |

### Default Frames (Critical for Interviews!)

| Scenario | Default Frame |
|----------|---------------|
| `OVER ()` — no ORDER BY | Entire partition (UNBOUNDED PRECEDING to UNBOUNDED FOLLOWING) |
| `OVER (ORDER BY col)` — with ORDER BY | `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` |

**This default explains why:**
- `SUM(x) OVER (ORDER BY date)` gives a running total (frame grows with each row)
- `SUM(x) OVER (PARTITION BY grp)` gives the partition total (entire partition is the frame)

### Common Frame Patterns

```sql
-- Trailing 7-day window (physical rows)
ROWS BETWEEN 6 PRECEDING AND CURRENT ROW

-- Centered 5-row window
ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING

-- Cumulative from start
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

-- Remaining rows to end
ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING

-- Entire partition (override ORDER BY default)
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
```

---

## NTILE for Percentile Bucketing

`NTILE(n)` divides the ordered partition into `n` roughly equal buckets.

### Basic Usage

```sql
-- Divide customers into quartiles by lifetime value
SELECT
    customer_id,
    lifetime_value,
    NTILE(4) OVER (ORDER BY lifetime_value DESC) AS ltv_quartile
FROM customers;
```

### Bucket Size Distribution

For N rows divided into B buckets:
- First `N % B` buckets get `CEIL(N/B)` rows
- Remaining buckets get `FLOOR(N/B)` rows

Example: 10 rows into 4 buckets → 3, 3, 2, 2

### Percentile Analysis

```sql
-- Find the median (P50) order value
WITH bucketed AS (
    SELECT
        order_value,
        NTILE(100) OVER (ORDER BY order_value) AS percentile
    FROM orders
)
SELECT
    MIN(order_value) AS p50_value
FROM bucketed
WHERE percentile = 50;
```

### Segmentation Use Case

```sql
-- Segment users into engagement tiers
SELECT
    user_id,
    total_sessions,
    CASE NTILE(5) OVER (ORDER BY total_sessions DESC)
        WHEN 1 THEN 'Power User'
        WHEN 2 THEN 'High'
        WHEN 3 THEN 'Medium'
        WHEN 4 THEN 'Low'
        WHEN 5 THEN 'Dormant'
    END AS engagement_tier
FROM user_activity;
```

---

## Common Patterns

### Pattern 1: Top-N per Group

```sql
-- Top 3 products by revenue per category
WITH ranked AS (
    SELECT
        category,
        product_name,
        revenue,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY revenue DESC) AS rn
    FROM products
)
SELECT * FROM ranked WHERE rn <= 3;
```

### Pattern 2: Year-over-Year Comparison

```sql
SELECT
    year,
    quarter,
    revenue,
    LAG(revenue, 4) OVER (ORDER BY year, quarter) AS same_quarter_last_year,
    ROUND(100.0 * (revenue - LAG(revenue, 4) OVER (ORDER BY year, quarter))
        / LAG(revenue, 4) OVER (ORDER BY year, quarter), 1) AS yoy_growth_pct
FROM quarterly_revenue;
```

### Pattern 3: Gap Detection (Islands and Gaps)

```sql
-- Find gaps in sequential IDs
WITH with_prev AS (
    SELECT
        id,
        LAG(id) OVER (ORDER BY id) AS prev_id
    FROM records
)
SELECT
    prev_id + 1 AS gap_start,
    id - 1 AS gap_end
FROM with_prev
WHERE id - prev_id > 1;
```

### Pattern 4: Sessionization

```sql
-- Define sessions: new session if gap > 30 minutes
WITH flagged AS (
    SELECT
        user_id,
        event_time,
        CASE WHEN event_time - LAG(event_time) OVER (
            PARTITION BY user_id ORDER BY event_time
        ) > INTERVAL '30 minutes' THEN 1 ELSE 0 END AS new_session_flag
    FROM events
),
sessioned AS (
    SELECT *,
        SUM(new_session_flag) OVER (
            PARTITION BY user_id ORDER BY event_time
        ) AS session_id
    FROM flagged
)
SELECT * FROM sessioned;
```

### Pattern 5: Consecutive Events Detection

```sql
-- Find users with 3+ consecutive days of activity
WITH daily AS (
    SELECT DISTINCT user_id, DATE(event_time) AS activity_date
    FROM events
),
grouped AS (
    SELECT *,
        activity_date - INTERVAL '1 day' * ROW_NUMBER() OVER (
            PARTITION BY user_id ORDER BY activity_date
        ) AS grp
    FROM daily
)
SELECT
    user_id,
    MIN(activity_date) AS streak_start,
    MAX(activity_date) AS streak_end,
    COUNT(*) AS streak_length
FROM grouped
GROUP BY user_id, grp
HAVING COUNT(*) >= 3;
```

### Pattern 6: Cumulative Distribution

```sql
-- What percentage of orders are below each value?
SELECT
    order_value,
    CUME_DIST() OVER (ORDER BY order_value) AS cumulative_pct,
    PERCENT_RANK() OVER (ORDER BY order_value) AS percent_rank
FROM orders;
```

---

## Performance Considerations

### Execution Order in Query Processing

```
1. FROM / JOIN
2. WHERE
3. GROUP BY
4. HAVING
5. SELECT (window functions computed here)
6. DISTINCT
7. ORDER BY
8. LIMIT / OFFSET
```

### Impact on Query Plans

| Factor | Performance Impact | Mitigation |
|--------|-------------------|------------|
| Large partitions | Memory pressure from sorting | Add filters to reduce partition sizes |
| Multiple window functions | Each may require a separate sort | Use same PARTITION BY + ORDER BY to share sorts |
| No partition clause | Entire table in one partition | Always partition when possible |
| RANGE vs ROWS | RANGE needs peer-group logic | Use ROWS for better performance when ties are irrelevant |

### Optimization Tips

1. **Share window definitions** with `WINDOW` clause:
```sql
SELECT
    SUM(amount) OVER w AS running_sum,
    AVG(amount) OVER w AS running_avg,
    COUNT(*) OVER w AS running_count
FROM orders
WINDOW w AS (PARTITION BY customer_id ORDER BY order_date);
```

2. **Index alignment**: Create indexes matching (PARTITION BY cols, ORDER BY cols) for sort elimination.

3. **Filter before windowing**: Push predicates into CTEs or subqueries since window functions run after WHERE.

4. **Avoid redundant windows**: Combine calculations that share the same OVER clause.

---

## Interview Problems with Solutions

### Problem 1: Second Highest Salary per Department

> "Find the second highest salary in each department. If there's no second highest, return NULL."

```sql
WITH ranked AS (
    SELECT
        department,
        employee_name,
        salary,
        DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank
    FROM employees
)
SELECT department, employee_name, salary
FROM ranked
WHERE salary_rank = 2;
```

**Why DENSE_RANK?** If two people tie for highest, we still want the next distinct salary level.

---

### Problem 2: Running Total with Reset

> "Calculate a running total of purchases per customer, resetting each month."

```sql
SELECT
    customer_id,
    purchase_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY customer_id, DATE_TRUNC('month', purchase_date)
        ORDER BY purchase_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS monthly_running_total
FROM purchases;
```

---

### Problem 3: Median Calculation

> "Find the median session duration per platform."

```sql
WITH ordered AS (
    SELECT
        platform,
        session_duration,
        ROW_NUMBER() OVER (PARTITION BY platform ORDER BY session_duration) AS rn,
        COUNT(*) OVER (PARTITION BY platform) AS total_count
    FROM sessions
)
SELECT
    platform,
    AVG(session_duration) AS median_duration
FROM ordered
WHERE rn IN (FLOOR((total_count + 1) / 2.0), CEIL((total_count + 1) / 2.0))
GROUP BY platform;
```

---

### Problem 4: Time Between Events

> "For each user, calculate the average time between consecutive logins."

```sql
WITH login_gaps AS (
    SELECT
        user_id,
        login_time,
        login_time - LAG(login_time) OVER (
            PARTITION BY user_id ORDER BY login_time
        ) AS time_since_last_login
    FROM logins
)
SELECT
    user_id,
    AVG(time_since_last_login) AS avg_time_between_logins
FROM login_gaps
WHERE time_since_last_login IS NOT NULL
GROUP BY user_id;
```

---

### Problem 5: Funnel Drop-off Analysis

> "Given an events table with event_type (view, cart, purchase), calculate the conversion rate at each stage per user cohort."

```sql
WITH stage_flags AS (
    SELECT
        user_cohort,
        user_id,
        MAX(CASE WHEN event_type = 'view' THEN 1 ELSE 0 END) AS viewed,
        MAX(CASE WHEN event_type = 'cart' THEN 1 ELSE 0 END) AS carted,
        MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS purchased
    FROM events
    GROUP BY user_cohort, user_id
),
funnel AS (
    SELECT
        user_cohort,
        SUM(viewed) AS views,
        SUM(carted) AS carts,
        SUM(purchased) AS purchases
    FROM stage_flags
    GROUP BY user_cohort
)
SELECT
    user_cohort,
    views,
    carts,
    ROUND(100.0 * carts / NULLIF(views, 0), 1) AS view_to_cart_pct,
    purchases,
    ROUND(100.0 * purchases / NULLIF(carts, 0), 1) AS cart_to_purchase_pct
FROM funnel;
```

---

### Problem 6: Identify Churned Users

> "A user is considered churned if they haven't had activity in the last 30 days from their most recent event. Identify churned users."

```sql
WITH last_activity AS (
    SELECT
        user_id,
        MAX(event_date) AS last_event_date,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY event_date DESC) AS rn
    FROM user_events
    GROUP BY user_id
)
SELECT user_id, last_event_date
FROM last_activity
WHERE CURRENT_DATE - last_event_date > 30;
```

---

### Problem 7: Detect Price Changes

> "For each product, find all dates where the price changed from the previous day."

```sql
SELECT
    product_id,
    date,
    price,
    LAG(price) OVER (PARTITION BY product_id ORDER BY date) AS prev_price
FROM product_prices
WHERE price != LAG(price) OVER (PARTITION BY product_id ORDER BY date);

-- Note: The above won't work due to window function in WHERE.
-- Correct approach:
WITH price_changes AS (
    SELECT
        product_id,
        date,
        price,
        LAG(price) OVER (PARTITION BY product_id ORDER BY date) AS prev_price
    FROM product_prices
)
SELECT *
FROM price_changes
WHERE price != prev_price OR prev_price IS NULL;
```

---

## Real-World Scenarios

### Scenario 1: 7-Day Rolling Retention

> "Calculate the 7-day rolling retention rate: of users active on day D, what fraction are active within the next 7 days?"

```sql
WITH daily_active AS (
    SELECT DISTINCT user_id, DATE(event_time) AS active_date
    FROM events
),
retention_pairs AS (
    SELECT
        a.active_date AS cohort_date,
        a.user_id,
        CASE WHEN b.user_id IS NOT NULL THEN 1 ELSE 0 END AS retained_7d
    FROM daily_active a
    LEFT JOIN daily_active b
        ON a.user_id = b.user_id
        AND b.active_date BETWEEN a.active_date + INTERVAL '1 day'
                               AND a.active_date + INTERVAL '7 days'
),
daily_retention AS (
    SELECT
        cohort_date,
        COUNT(DISTINCT user_id) AS active_users,
        COUNT(DISTINCT CASE WHEN retained_7d = 1 THEN user_id END) AS retained_users
    FROM retention_pairs
    GROUP BY cohort_date
)
SELECT
    cohort_date,
    active_users,
    retained_users,
    ROUND(100.0 * retained_users / active_users, 2) AS retention_rate_pct,
    AVG(ROUND(100.0 * retained_users / active_users, 2)) OVER (
        ORDER BY cohort_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7d_avg_retention
FROM daily_retention
ORDER BY cohort_date;
```

### Scenario 2: Find Consecutive Days of Revenue Decline

> "Identify periods where revenue declined for 3 or more consecutive days."

```sql
WITH daily_rev AS (
    SELECT
        date,
        revenue,
        CASE WHEN revenue < LAG(revenue) OVER (ORDER BY date)
             THEN 1 ELSE 0 END AS is_decline
    FROM daily_revenue
),
decline_groups AS (
    SELECT *,
        SUM(CASE WHEN is_decline = 0 THEN 1 ELSE 0 END) OVER (
            ORDER BY date
        ) AS grp
    FROM daily_rev
)
SELECT
    MIN(date) AS decline_start,
    MAX(date) AS decline_end,
    COUNT(*) AS consecutive_decline_days,
    FIRST_VALUE(revenue) OVER (PARTITION BY grp ORDER BY date) AS starting_revenue,
    MIN(revenue) AS lowest_revenue
FROM decline_groups
WHERE is_decline = 1
GROUP BY grp
HAVING COUNT(*) >= 3
ORDER BY decline_start;
```

### Scenario 3: Customer Lifetime Value Segmentation

> "Segment customers into deciles by LTV and show the revenue contribution of each decile."

```sql
WITH customer_ltv AS (
    SELECT
        customer_id,
        SUM(order_amount) AS lifetime_value
    FROM orders
    GROUP BY customer_id
),
deciled AS (
    SELECT *,
        NTILE(10) OVER (ORDER BY lifetime_value DESC) AS ltv_decile
    FROM customer_ltv
)
SELECT
    ltv_decile,
    COUNT(*) AS customer_count,
    ROUND(AVG(lifetime_value), 2) AS avg_ltv,
    ROUND(SUM(lifetime_value), 2) AS total_revenue,
    ROUND(100.0 * SUM(lifetime_value) / SUM(SUM(lifetime_value)) OVER (), 2) AS pct_of_total_revenue
FROM deciled
GROUP BY ltv_decile
ORDER BY ltv_decile;
```

---

## Interview Talking Points

> **On choosing ranking functions:**
> "I choose between ROW_NUMBER, RANK, and DENSE_RANK based on how ties should be handled. If I need exactly one row per group — like the most recent order — I use ROW_NUMBER. If I need competition-style ranking where ties share a position and the next rank is skipped, I use RANK. If I want dense consecutive ranks — like finding the top 3 salary tiers regardless of how many people share a tier — I use DENSE_RANK."

> **On frame specifications:**
> "The frame clause is where most people trip up. The key insight is that the default frame changes depending on whether ORDER BY is present. Without ORDER BY, the frame is the entire partition. With ORDER BY, it defaults to RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW — which is why SUM with ORDER BY gives a running total. I always explicitly specify the frame to make intent clear and avoid subtle bugs, especially with LAST_VALUE."

> **On performance:**
> "Window functions execute after GROUP BY and HAVING, during the SELECT phase. The main cost is sorting — each distinct combination of PARTITION BY and ORDER BY may require a sort. I optimize by reusing the same window definition across multiple functions via the WINDOW clause, and by ensuring indexes align with my partition and order columns."

> **On sessionization:**
> "Sessionization is a classic window function pattern. I use LAG to compute the time gap between consecutive events per user, flag any gap exceeding the threshold as a new session boundary, then use a cumulative SUM over those flags to assign session IDs. This avoids expensive self-joins and is the standard approach in clickstream analytics."

> **On problem decomposition:**
> "When I see a window function problem, I decompose it in three steps: first, what's the partition — what group am I computing within? Second, what's the ordering — how are rows arranged inside that group? Third, what's the computation — am I ranking, comparing to adjacent rows, or aggregating over a sliding frame? This framework handles 90% of problems I see in interviews."

---

## Common Interview Mistakes

| # | Mistake (❌) | Correct Approach (✅) |
|---|---|---|
| 1 | Using window functions in WHERE clause | Wrap in CTE/subquery first, then filter |
| 2 | Forgetting LAST_VALUE default frame stops at current row | Explicitly set `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` |
| 3 | Using ROW_NUMBER when ties matter | Use RANK or DENSE_RANK for tie-aware ranking |
| 4 | Confusing ROWS and RANGE frames | ROWS = physical positions; RANGE = logical value ranges |
| 5 | Not handling NULLs in LAG/LEAD | Use the third argument: `LAG(col, 1, 0)` for default value |
| 6 | Applying NTILE without ORDER BY | Always include ORDER BY — buckets are meaningless without ordering |
| 7 | Multiple sorts from unaligned OVER clauses | Consolidate with WINDOW clause or align PARTITION/ORDER |
| 8 | Using DISTINCT with window functions expecting deduplication | Window functions run before DISTINCT — use CTE to dedup first |
| 9 | Running total with RANGE (default) when duplicates exist | Use ROWS explicitly to avoid treating tied ORDER BY values as same position |
| 10 | Forgetting that ORDER BY in OVER is independent of query ORDER BY | The OVER's ORDER BY defines the window; query's ORDER BY defines output sort |

---

## Rapid-Fire Q&A

**Q1: Can you use a window function in a WHERE clause?**
No. Window functions are evaluated in the SELECT phase. Use a CTE or subquery, then filter in the outer query.

**Q2: What's the difference between `SUM(x) OVER ()` and `SUM(x) OVER (ORDER BY date)`?**
Without ORDER BY: returns the total sum for the entire partition (every row gets the same value). With ORDER BY: returns a running/cumulative sum because the default frame extends from UNBOUNDED PRECEDING to CURRENT ROW.

**Q3: Does PARTITION BY change the output row count?**
No. Unlike GROUP BY, PARTITION BY never reduces the number of rows. It only defines the scope of the window computation.

**Q4: What happens when NTILE(4) has 7 rows?**
Buckets get: 2, 2, 2, 1 rows respectively (first N%B buckets get one extra row).

**Q5: Can you nest window functions?**
No. `RANK() OVER (ORDER BY SUM(x) OVER (...))` is invalid. Use a CTE to compute the inner window function first.

**Q6: What's PERCENT_RANK vs CUME_DIST?**
PERCENT_RANK = (rank - 1) / (total_rows - 1), ranges from 0 to 1. CUME_DIST = rows_with_value_leq / total_rows, ranges from 1/N to 1. CUME_DIST is never 0.

**Q7: Does ORDER BY in OVER guarantee output order?**
No. It only orders rows for the window computation. You still need a query-level ORDER BY for output ordering.

**Q8: When would you use GROUPS instead of ROWS or RANGE?**
GROUPS counts distinct peer groups. Example: "average over the last 3 distinct date values" regardless of how many rows share each date.

**Q9: Can window functions reference columns from outer queries?**
No in standard SQL. Window functions can only reference columns available in the current query's SELECT scope.

**Q10: How do window functions interact with NULL?**
NULLs in PARTITION BY form their own group. NULLs in ORDER BY sort first or last depending on the engine (use NULLS FIRST/LAST to control). Aggregate windows (SUM, AVG) skip NULLs like regular aggregates.

---

## Summary Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    WINDOW FUNCTIONS CHEAT SHEET                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SYNTAX:  func(expr) OVER (PARTITION BY .. ORDER BY .. frame_clause)        │
│                                                                             │
├─────────────────────┬───────────────────────────────────────────────────────┤
│  RANKING            │  ROW_NUMBER: unique sequential, breaks ties            │
│                     │  RANK:       ties share rank, gaps after               │
│                     │  DENSE_RANK: ties share rank, no gaps                  │
│                     │  NTILE(n):   divide into n equal-ish buckets           │
├─────────────────────┼───────────────────────────────────────────────────────┤
│  OFFSET             │  LAG(col, n, def):   n rows back                      │
│                     │  LEAD(col, n, def):  n rows forward                   │
│                     │  FIRST_VALUE(col):   first in frame                   │
│                     │  LAST_VALUE(col):    last in frame (watch the frame!) │
│                     │  NTH_VALUE(col, n):  nth row in frame                 │
├─────────────────────┼───────────────────────────────────────────────────────┤
│  AGGREGATE          │  SUM / AVG / COUNT / MIN / MAX over window            │
│                     │  With ORDER BY → running/cumulative                   │
│                     │  Without ORDER BY → partition-wide total              │
├─────────────────────┼───────────────────────────────────────────────────────┤
│  DISTRIBUTION       │  PERCENT_RANK:  (rank-1)/(n-1)                        │
│                     │  CUME_DIST:     count(<=val)/n                         │
├─────────────────────┼───────────────────────────────────────────────────────┤
│  FRAMES             │                                                       │
│                     │  ROWS BETWEEN x AND y     ← physical positions        │
│                     │  RANGE BETWEEN x AND y    ← logical value range       │
│                     │  GROUPS BETWEEN x AND y   ← peer group count          │
│                     │                                                       │
│                     │  Boundaries: UNBOUNDED PRECEDING | n PRECEDING |      │
│                     │              CURRENT ROW | n FOLLOWING |               │
│                     │              UNBOUNDED FOLLOWING                       │
├─────────────────────┼───────────────────────────────────────────────────────┤
│  KEY RULES          │                                                       │
│                     │  1. Cannot use in WHERE/HAVING → wrap in CTE          │
│                     │  2. Executes after GROUP BY, before ORDER BY          │
│                     │  3. Default frame with ORDER BY = RANGE to CURRENT    │
│                     │  4. Default frame without ORDER BY = entire partition  │
│                     │  5. LAST_VALUE needs explicit UNBOUNDED FOLLOWING     │
│                     │  6. Cannot nest window functions                       │
├─────────────────────┼───────────────────────────────────────────────────────┤
│  PATTERNS           │                                                       │
│                     │  Top-N per group     → ROW_NUMBER + CTE + WHERE rn<=N │
│                     │  Running total       → SUM() OVER (ORDER BY date)     │
│                     │  Moving avg (7d)     → AVG() OVER (ROWS 6 PREC..CUR) │
│                     │  YoY comparison      → LAG(val, 12) or LAG(val, 4)   │
│                     │  Gap detection       → LAG + difference check         │
│                     │  Sessionization      → LAG + flag + cumulative SUM    │
│                     │  Consecutive streaks  → date - ROW_NUMBER → group     │
│                     │  Percentile buckets  → NTILE(100) for percentiles     │
└─────────────────────┴───────────────────────────────────────────────────────┘
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 2 of 45*
