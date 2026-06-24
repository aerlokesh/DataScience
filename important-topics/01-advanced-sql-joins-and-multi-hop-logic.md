# Topic 1: Advanced SQL Joins & Multi-Hop Logic

> **Data Science Interview -- Deep Dive**  
> Mastering every join type, multi-table traversal patterns, anti/semi-join idioms, and performance considerations that separate senior candidates from the rest.

---

## Table of Contents

1. [Join Fundamentals Refresher](#1-join-fundamentals-refresher)
2. [INNER JOIN -- The Workhorse](#2-inner-join--the-workhorse)
3. [LEFT / RIGHT OUTER JOIN -- Preserving One Side](#3-left--right-outer-join--preserving-one-side)
4. [FULL OUTER JOIN -- Preserving Both Sides](#4-full-outer-join--preserving-both-sides)
5. [CROSS JOIN -- Cartesian Products Done Right](#5-cross-join--cartesian-products-done-right)
6. [Self-Joins -- Comparing Rows Within a Table](#6-self-joins--comparing-rows-within-a-table)
7. [Multi-Hop Joins -- Traversing Entity Graphs](#7-multi-hop-joins--traversing-entity-graphs)
8. [Anti-Joins and Semi-Joins](#8-anti-joins-and-semi-joins)
9. [Join Performance & Optimization](#9-join-performance--optimization)
10. [Common Traps & Pitfalls](#10-common-traps--pitfalls)
11. [Interview Problems with Solutions](#11-interview-problems-with-solutions)
12. [Real-World Query Design Exercise](#12-real-world-query-design-exercise)
13. [Interview Talking Points](#13-interview-talking-points)
14. [Common Interview Mistakes](#14-common-interview-mistakes)
15. [Rapid-Fire Q&A](#15-rapid-fire-qa)
16. [Summary Cheat Sheet](#16-summary-cheat-sheet)

---

## 1. Join Fundamentals Refresher

A join combines rows from two or more tables based on a related column. The key mental model:

| Join Type | Rows Returned |
|-----------|--------------|
| INNER | Only matching rows from both tables |
| LEFT OUTER | All rows from the left table + matches from right (NULLs where no match) |
| RIGHT OUTER | All rows from the right table + matches from left (NULLs where no match) |
| FULL OUTER | All rows from both tables (NULLs on the side with no match) |
| CROSS | Every combination of rows (Cartesian product) |

**Key principle for interviews:** Always state which table is the "driver" (left side) and why you chose it.

---

## 2. INNER JOIN -- The Workhorse

Returns only rows where the join condition matches in both tables.

```sql
-- DS Example: Find users who have made at least one purchase
SELECT 
    u.user_id,
    u.signup_date,
    COUNT(o.order_id) AS total_orders,
    SUM(o.amount) AS lifetime_value
FROM users u
INNER JOIN orders o ON u.user_id = o.user_id
WHERE u.signup_date >= '2024-01-01'
GROUP BY u.user_id, u.signup_date;
```

**When to use in interviews:**
- You only care about entities that exist in BOTH tables
- Building cohort analyses where users must have an event
- Feature engineering where NULL features are unacceptable

**Data Science context:** INNER JOIN implicitly filters your dataset. If 30% of users never ordered, they silently disappear. Always call this out in interviews.

---

## 3. LEFT / RIGHT OUTER JOIN -- Preserving One Side

```sql
-- DS Example: User retention -- include users who NEVER converted
SELECT 
    u.user_id,
    u.signup_date,
    u.acquisition_channel,
    COALESCE(COUNT(o.order_id), 0) AS total_orders,
    CASE WHEN MIN(o.order_date) IS NOT NULL THEN 1 ELSE 0 END AS converted
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
GROUP BY u.user_id, u.signup_date, u.acquisition_channel;
```

```sql
-- DS Example: Find all experiments and their results (some still running)
SELECT 
    e.experiment_id,
    e.experiment_name,
    e.start_date,
    r.metric_name,
    r.lift_percent,
    r.p_value
FROM experiments e
LEFT JOIN experiment_results r 
    ON e.experiment_id = r.experiment_id
ORDER BY e.start_date DESC;
```

**Interview tip:** RIGHT JOIN is logically identical to LEFT JOIN with swapped table order. In practice, always rewrite as LEFT JOIN for readability. Interviewers notice this.

---

## 4. FULL OUTER JOIN -- Preserving Both Sides

```sql
-- DS Example: Reconcile two data sources (app events vs server logs)
SELECT 
    COALESCE(a.event_id, s.event_id) AS event_id,
    a.client_timestamp,
    s.server_timestamp,
    CASE 
        WHEN a.event_id IS NULL THEN 'server_only'
        WHEN s.event_id IS NULL THEN 'client_only'
        ELSE 'matched'
    END AS reconciliation_status
FROM app_events a
FULL OUTER JOIN server_logs s 
    ON a.event_id = s.event_id
    AND ABS(DATEDIFF(second, a.client_timestamp, s.server_timestamp)) <= 5;
```

**When this appears in interviews:**
- Data quality audits
- Comparing two scoring models side-by-side
- Finding gaps between two datasets (e.g., predicted vs actual)

---

## 5. CROSS JOIN -- Cartesian Products Done Right

```sql
-- DS Example: Generate a complete date x product grid for time-series analysis
SELECT 
    d.date_day,
    p.product_id,
    COALESCE(s.units_sold, 0) AS units_sold
FROM date_spine d
CROSS JOIN products p
LEFT JOIN daily_sales s 
    ON d.date_day = s.sale_date 
    AND p.product_id = s.product_id
WHERE d.date_day BETWEEN '2024-01-01' AND '2024-12-31';
```

```sql
-- DS Example: All pairwise user comparisons for similarity scoring
SELECT 
    a.user_id AS user_a,
    b.user_id AS user_b
FROM users a
CROSS JOIN users b
WHERE a.user_id < b.user_id;  -- avoid duplicates and self-pairs
```

**Interview warning:** Always mention cardinality. CROSS JOIN of 1M x 1M = 1 trillion rows. Show awareness of this.

---

## 6. Self-Joins -- Comparing Rows Within a Table

### Pattern A: Employee-Manager Hierarchy

```sql
-- Find each employee's manager name and their manager's manager
SELECT 
    e.employee_id,
    e.name AS employee_name,
    m.name AS manager_name,
    mm.name AS skip_level_manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id
LEFT JOIN employees mm ON m.manager_id = mm.employee_id;
```

### Pattern B: Sequential Events (Session Analysis)

```sql
-- Find consecutive page views and compute time between them
SELECT 
    curr.user_id,
    curr.page_name AS current_page,
    next_evt.page_name AS next_page,
    DATEDIFF(second, curr.event_time, next_evt.event_time) AS seconds_between
FROM page_views curr
INNER JOIN page_views next_evt
    ON curr.user_id = next_evt.user_id
    AND curr.event_sequence = next_evt.event_sequence - 1
    AND curr.session_id = next_evt.session_id;
```

### Pattern C: Comparing Rows (Month-over-Month Growth)

```sql
-- Month-over-month revenue comparison per product
SELECT 
    curr.product_id,
    curr.month,
    curr.revenue AS current_revenue,
    prev.revenue AS prev_revenue,
    ROUND(100.0 * (curr.revenue - prev.revenue) / NULLIF(prev.revenue, 0), 2) 
        AS mom_growth_pct
FROM monthly_revenue curr
LEFT JOIN monthly_revenue prev
    ON curr.product_id = prev.product_id
    AND curr.month = prev.month + INTERVAL '1 month';
```

### Pattern D: Finding Duplicate or Near-Duplicate Records

```sql
-- Find potential duplicate users (same email domain + similar name)
SELECT 
    a.user_id AS user_a,
    b.user_id AS user_b,
    a.email,
    b.email,
    a.full_name,
    b.full_name
FROM users a
INNER JOIN users b
    ON a.user_id < b.user_id
    AND SPLIT_PART(a.email, '@', 2) = SPLIT_PART(b.email, '@', 2)
    AND LEVENSHTEIN(a.full_name, b.full_name) <= 3;
```

---

## 7. Multi-Hop Joins -- Traversing Entity Graphs

### The Classic 4-Table Chain: User -> Order -> Product -> Category

```sql
-- What product categories does each user segment prefer?
SELECT 
    u.user_segment,
    c.category_name,
    COUNT(DISTINCT o.order_id) AS order_count,
    SUM(oi.quantity * oi.unit_price) AS total_spend,
    AVG(oi.quantity * oi.unit_price) AS avg_order_value
FROM users u
INNER JOIN orders o ON u.user_id = o.user_id
INNER JOIN order_items oi ON o.order_id = oi.order_id
INNER JOIN products p ON oi.product_id = p.product_id
INNER JOIN categories c ON p.category_id = c.category_id
WHERE o.order_date >= CURRENT_DATE - INTERVAL '90 days'
GROUP BY u.user_segment, c.category_name
ORDER BY u.user_segment, total_spend DESC;
```

### Graph Traversal: Social Network (Friends of Friends)

```sql
-- Find 2nd-degree connections (friends of friends, excluding direct friends)
SELECT DISTINCT
    f2.friend_id AS suggested_user
FROM friendships f1
INNER JOIN friendships f2 
    ON f1.friend_id = f2.user_id
WHERE f1.user_id = 12345
    AND f2.friend_id != 12345
    AND f2.friend_id NOT IN (
        SELECT friend_id FROM friendships WHERE user_id = 12345
    );
```

### Multi-Hop with Aggregations at Each Level

```sql
-- Attribution: campaign -> click -> session -> conversion
SELECT 
    camp.campaign_name,
    camp.channel,
    COUNT(DISTINCT cl.click_id) AS total_clicks,
    COUNT(DISTINCT s.session_id) AS sessions_started,
    COUNT(DISTINCT cv.conversion_id) AS conversions,
    ROUND(100.0 * COUNT(DISTINCT cv.conversion_id) / 
        NULLIF(COUNT(DISTINCT cl.click_id), 0), 2) AS conversion_rate
FROM campaigns camp
INNER JOIN clicks cl ON camp.campaign_id = cl.campaign_id
LEFT JOIN sessions s ON cl.click_id = s.entry_click_id
LEFT JOIN conversions cv ON s.session_id = cv.session_id
WHERE camp.start_date >= '2024-01-01'
GROUP BY camp.campaign_name, camp.channel
ORDER BY conversions DESC;
```

**Interview insight:** Notice the shift from INNER to LEFT as we move down the funnel. Clicks always have a campaign (INNER), but not all clicks start sessions (LEFT), and not all sessions convert (LEFT).

---

## 8. Anti-Joins and Semi-Joins

### Anti-Join: Find Rows That Do NOT Have a Match

**Pattern 1: LEFT JOIN + WHERE NULL (most common in interviews)**

```sql
-- Users who signed up but never placed an order
SELECT u.user_id, u.email, u.signup_date
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
WHERE o.order_id IS NULL;
```

**Pattern 2: NOT EXISTS (often more performant)**

```sql
-- Products that were never purchased in Q4
SELECT p.product_id, p.product_name, p.category_id
FROM products p
WHERE NOT EXISTS (
    SELECT 1 
    FROM order_items oi
    INNER JOIN orders o ON oi.order_id = o.order_id
    WHERE oi.product_id = p.product_id
    AND o.order_date BETWEEN '2024-10-01' AND '2024-12-31'
);
```

**Pattern 3: NOT IN (use with caution)**

```sql
-- Careful: NOT IN fails silently if subquery returns NULLs
SELECT user_id FROM users
WHERE user_id NOT IN (
    SELECT user_id FROM orders WHERE user_id IS NOT NULL  -- must filter NULLs!
);
```

### Semi-Join: Find Rows That HAVE a Match (Without Duplicating)

```sql
-- Users who purchased at least one premium product (no row duplication)
SELECT u.user_id, u.name, u.signup_date
FROM users u
WHERE EXISTS (
    SELECT 1
    FROM orders o
    INNER JOIN order_items oi ON o.order_id = oi.order_id
    INNER JOIN products p ON oi.product_id = p.product_id
    WHERE o.user_id = u.user_id
    AND p.tier = 'premium'
);
```

**Why EXISTS over INNER JOIN here?** If a user bought 5 premium products, INNER JOIN returns 5 rows. EXISTS returns exactly 1 row per user -- no deduplication needed.

---

## 9. Join Performance & Optimization

### Index Usage

```sql
-- Ensure join columns are indexed
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);

-- Composite index for common filtered joins
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);
```

### Join Order Matters

| Strategy | When to Use |
|----------|-------------|
| Small table first (build side) | Hash joins -- smaller table fits in memory |
| Most selective filter first | Reduce row count before expensive joins |
| Star schema: fact table as driver | Dimension tables are small, join them in |

### Broadcast vs. Shuffle (Distributed Systems -- Spark/BigQuery)

```sql
-- Spark hint: broadcast small dimension table
SELECT /*+ BROADCAST(dim_country) */
    f.user_id,
    f.revenue,
    d.country_name
FROM fact_transactions f
INNER JOIN dim_country d ON f.country_code = d.country_code;
```

| Join Strategy | How It Works | Best When |
|---------------|-------------|-----------|
| Broadcast | Small table sent to all nodes | One table < 10MB (configurable) |
| Shuffle/Sort-Merge | Both tables redistributed by join key | Both tables are large |
| Bucket Join | Pre-partitioned tables joined locally | Repeated joins on same key |

### EXPLAIN ANALYZE Interpretation

```sql
EXPLAIN ANALYZE
SELECT u.user_id, COUNT(o.order_id)
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
GROUP BY u.user_id;

-- Look for:
-- Hash Join vs Nested Loop vs Merge Join
-- Seq Scan vs Index Scan
-- Estimated vs Actual row counts (large gaps = stale statistics)
```

---

## 10. Common Traps & Pitfalls

### Trap 1: Many-to-Many Join Explosion

```sql
-- BAD: user_tags is many-to-many with both users and tags
-- If user has 5 tags and 3 orders, you get 15 rows per user
SELECT u.user_id, t.tag_name, o.order_id
FROM users u
INNER JOIN user_tags t ON u.user_id = t.user_id
INNER JOIN orders o ON u.user_id = o.user_id;
-- Result: |users| x |tags_per_user| x |orders_per_user| rows!

-- FIX: Aggregate before joining
WITH user_tag_summary AS (
    SELECT user_id, STRING_AGG(tag_name, ', ') AS tags
    FROM user_tags
    GROUP BY user_id
)
SELECT u.user_id, uts.tags, COUNT(o.order_id) AS order_count
FROM users u
LEFT JOIN user_tag_summary uts ON u.user_id = uts.user_id
LEFT JOIN orders o ON u.user_id = o.user_id
GROUP BY u.user_id, uts.tags;
```

### Trap 2: NULL Behavior in Joins

```sql
-- NULLs NEVER match in join conditions
-- If orders.user_id has NULLs, those rows are dropped in INNER JOIN

-- This returns NOTHING for NULL user_ids:
SELECT * FROM orders o INNER JOIN users u ON o.user_id = u.user_id;

-- To find orphan orders with NULL user_id:
SELECT * FROM orders WHERE user_id IS NULL;
```

### Trap 3: Filtering in ON vs WHERE (LEFT JOIN)

```sql
-- WRONG: This converts LEFT JOIN to INNER JOIN behavior
SELECT u.user_id, o.order_id
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
WHERE o.order_date >= '2024-01-01';  -- NULLs filtered out!

-- CORRECT: Put the filter in the ON clause
SELECT u.user_id, o.order_id
FROM users u
LEFT JOIN orders o 
    ON u.user_id = o.user_id
    AND o.order_date >= '2024-01-01';  -- preserves non-matching users
```

### Trap 4: Unintentional Cross Join

```sql
-- Missing join condition creates a cross join
SELECT u.name, o.amount
FROM users u, orders o;  -- implicit cross join! 1M users x 5M orders = 5 trillion rows

-- Always use explicit JOIN syntax
SELECT u.name, o.amount
FROM users u
INNER JOIN orders o ON u.user_id = o.user_id;
```

---

## 11. Interview Problems with Solutions

### Problem 1: User Retention Cohort Analysis

**Question:** "Given a `users` table (user_id, signup_date) and a `logins` table (user_id, login_date), calculate the Day-7 retention rate for each weekly signup cohort."

```sql
WITH signup_cohorts AS (
    SELECT 
        user_id,
        signup_date,
        DATE_TRUNC('week', signup_date) AS cohort_week
    FROM users
),
day7_logins AS (
    SELECT DISTINCT
        l.user_id,
        l.login_date
    FROM logins l
    INNER JOIN signup_cohorts sc ON l.user_id = sc.user_id
    WHERE l.login_date = sc.signup_date + INTERVAL '7 days'
)
SELECT 
    sc.cohort_week,
    COUNT(DISTINCT sc.user_id) AS cohort_size,
    COUNT(DISTINCT d7.user_id) AS retained_day7,
    ROUND(100.0 * COUNT(DISTINCT d7.user_id) / COUNT(DISTINCT sc.user_id), 2) 
        AS retention_rate_pct
FROM signup_cohorts sc
LEFT JOIN day7_logins d7 ON sc.user_id = d7.user_id
GROUP BY sc.cohort_week
ORDER BY sc.cohort_week;
```

**Why LEFT JOIN?** We need ALL users in the denominator, even those who did not return on Day 7.

---

### Problem 2: Products Never Ordered Together

**Question:** "Find pairs of products that have NEVER appeared in the same order, but are in the same category."

```sql
WITH same_category_pairs AS (
    -- All possible product pairs within same category
    SELECT 
        a.product_id AS prod_a,
        b.product_id AS prod_b,
        a.category_id
    FROM products a
    INNER JOIN products b 
        ON a.category_id = b.category_id
        AND a.product_id < b.product_id
),
co_purchased AS (
    -- Product pairs that HAVE appeared in the same order
    SELECT DISTINCT
        oi1.product_id AS prod_a,
        oi2.product_id AS prod_b
    FROM order_items oi1
    INNER JOIN order_items oi2
        ON oi1.order_id = oi2.order_id
        AND oi1.product_id < oi2.product_id
)
SELECT 
    scp.prod_a,
    scp.prod_b,
    scp.category_id
FROM same_category_pairs scp
LEFT JOIN co_purchased cp
    ON scp.prod_a = cp.prod_a
    AND scp.prod_b = cp.prod_b
WHERE cp.prod_a IS NULL;  -- Anti-join: never co-purchased
```

---

### Problem 3: Identify Churned High-Value Users

**Question:** "Find users who spent more than $500 in their first 30 days but have not made any purchase in the last 90 days."

```sql
WITH first_30_day_spend AS (
    SELECT 
        u.user_id,
        SUM(o.amount) AS early_spend
    FROM users u
    INNER JOIN orders o 
        ON u.user_id = o.user_id
        AND o.order_date BETWEEN u.signup_date AND u.signup_date + INTERVAL '30 days'
    GROUP BY u.user_id
    HAVING SUM(o.amount) > 500
),
recent_activity AS (
    SELECT DISTINCT user_id
    FROM orders
    WHERE order_date >= CURRENT_DATE - INTERVAL '90 days'
)
SELECT 
    f.user_id,
    f.early_spend,
    u.signup_date,
    u.email
FROM first_30_day_spend f
INNER JOIN users u ON f.user_id = u.user_id
LEFT JOIN recent_activity ra ON f.user_id = ra.user_id
WHERE ra.user_id IS NULL  -- Anti-join: no recent activity
ORDER BY f.early_spend DESC;
```

---

### Problem 4: Multi-Hop Attribution with Deduplication

**Question:** "For each marketing campaign, find the number of unique users who clicked an ad, started a trial, AND converted to paid -- all within 14 days of the click."

```sql
SELECT 
    c.campaign_id,
    c.campaign_name,
    c.channel,
    COUNT(DISTINCT cl.user_id) AS clickers,
    COUNT(DISTINCT t.user_id) AS trial_starters,
    COUNT(DISTINCT p.user_id) AS paid_converters,
    ROUND(100.0 * COUNT(DISTINCT p.user_id) / 
        NULLIF(COUNT(DISTINCT cl.user_id), 0), 2) AS click_to_paid_rate
FROM campaigns c
INNER JOIN ad_clicks cl ON c.campaign_id = cl.campaign_id
LEFT JOIN trials t 
    ON cl.user_id = t.user_id
    AND t.trial_start BETWEEN cl.click_time AND cl.click_time + INTERVAL '14 days'
LEFT JOIN paid_conversions p
    ON t.user_id = p.user_id
    AND p.conversion_date BETWEEN t.trial_start AND t.trial_start + INTERVAL '14 days'
WHERE cl.click_time >= '2024-01-01'
GROUP BY c.campaign_id, c.campaign_name, c.channel
ORDER BY paid_converters DESC;
```

**Key insight to mention:** The temporal constraints (BETWEEN ... AND ...) in the ON clause prevent incorrect attribution while preserving the LEFT JOIN semantics.

---

## 12. Real-World Query Design Exercise

**Prompt:** "Design the query for a daily user activity report that joins Users, Sessions, PageViews, and Purchases to show per-user daily engagement metrics."

### Step-by-Step Approach (talk through this in interviews):

**Step 1: Identify the grain** -- One row per user per day.

**Step 2: Identify the driver table** -- A date spine crossed with active users ensures no gaps.

**Step 3: Build incrementally:**

```sql
-- Final query: Daily user activity report
WITH date_spine AS (
    SELECT generate_series(
        '2024-01-01'::date, 
        CURRENT_DATE, 
        '1 day'::interval
    )::date AS report_date
),
active_users AS (
    SELECT DISTINCT user_id
    FROM sessions
    WHERE session_start >= '2024-01-01'
),
user_dates AS (
    SELECT au.user_id, ds.report_date
    FROM active_users au
    CROSS JOIN date_spine ds
),
daily_sessions AS (
    SELECT 
        user_id,
        DATE(session_start) AS activity_date,
        COUNT(*) AS session_count,
        SUM(EXTRACT(EPOCH FROM session_end - session_start)) / 60.0 AS total_minutes
    FROM sessions
    GROUP BY user_id, DATE(session_start)
),
daily_pageviews AS (
    SELECT 
        user_id,
        DATE(view_time) AS activity_date,
        COUNT(*) AS pageview_count,
        COUNT(DISTINCT page_url) AS unique_pages
    FROM pageviews
    GROUP BY user_id, DATE(view_time)
),
daily_purchases AS (
    SELECT 
        user_id,
        DATE(purchase_time) AS activity_date,
        COUNT(*) AS purchase_count,
        SUM(amount) AS daily_spend
    FROM purchases
    GROUP BY user_id, DATE(purchase_time)
)
SELECT 
    ud.user_id,
    ud.report_date,
    u.user_segment,
    COALESCE(ds.session_count, 0) AS sessions,
    COALESCE(ds.total_minutes, 0) AS minutes_active,
    COALESCE(dp.pageview_count, 0) AS pageviews,
    COALESCE(dp.unique_pages, 0) AS unique_pages,
    COALESCE(dpur.purchase_count, 0) AS purchases,
    COALESCE(dpur.daily_spend, 0) AS spend
FROM user_dates ud
INNER JOIN users u ON ud.user_id = u.user_id
LEFT JOIN daily_sessions ds 
    ON ud.user_id = ds.user_id AND ud.report_date = ds.activity_date
LEFT JOIN daily_pageviews dp 
    ON ud.user_id = dp.user_id AND ud.report_date = dp.activity_date
LEFT JOIN daily_purchases dpur 
    ON ud.user_id = dpur.user_id AND ud.report_date = dpur.activity_date
ORDER BY ud.user_id, ud.report_date;
```

**Why aggregate-then-join?** Pre-aggregating each fact table to the desired grain (user x day) before joining prevents the many-to-many explosion that would occur if you joined raw tables.

---

## 13. Interview Talking Points

> "I always start by clarifying the grain of the output -- what does one row represent? This determines which table drives the query and whether I need LEFT or INNER joins."

> "When joining multiple fact tables to a shared dimension, I pre-aggregate each fact table to the target grain first, then join. This avoids the fan-out problem where a many-to-many relationship inflates row counts."

> "For anti-join patterns, I prefer NOT EXISTS over LEFT JOIN WHERE NULL in production because the optimizer can short-circuit evaluation once a match is found. But in interviews I use LEFT JOIN WHERE NULL because it is more visually intuitive."

> "In distributed systems like Spark or BigQuery, I am mindful of shuffle costs. If one table is small enough, I will broadcast it. If both tables are large, I ensure they are partitioned on the join key to enable co-located joins."

> "I always check for NULL values in my join keys. NULLs never match other NULLs in standard SQL join conditions, which can silently drop rows. I filter or COALESCE them explicitly."

> "When I see a self-join requirement, I ask: can this be solved with a window function instead? LAG/LEAD for sequential comparisons is usually more efficient than self-joining on sequence numbers."

---

## 14. Common Interview Mistakes

| Mistake | Correct Approach |
|---------|-----------------|
| Using INNER JOIN when the question says "all users including those without orders" | Use LEFT JOIN to preserve the complete user set |
| Filtering on the right table in WHERE clause after LEFT JOIN | Move filter into the ON clause to preserve LEFT JOIN semantics |
| Using NOT IN with a subquery that might return NULLs | Use NOT EXISTS or add WHERE col IS NOT NULL in subquery |
| Joining raw fact tables without pre-aggregation | Aggregate each fact to target grain BEFORE joining |
| Forgetting to handle NULLs in aggregations after LEFT JOIN | Wrap with COALESCE(metric, 0) for counts/sums |
| Writing `SELECT *` in multi-table joins | Always specify columns; avoids ambiguity and shows intent |
| Assuming join order does not matter | In large datasets, filter early, join small tables first |
| Using DISTINCT to fix duplicated rows from a bad join | Fix the root cause -- add missing join conditions or pre-aggregate |
| Not mentioning indexes when discussing performance | Always mention that join columns should be indexed |
| Using comma-separated FROM (implicit cross join) | Always use explicit JOIN ... ON syntax |

---

## 15. Rapid-Fire Q&A

**Q1: What is the difference between WHERE and ON in a LEFT JOIN?**  
A: ON determines which rows match. WHERE filters AFTER the join, potentially eliminating NULLs from non-matches and converting the LEFT JOIN to an INNER JOIN.

**Q2: Can you join on inequality conditions?**  
A: Yes. Example: `ON a.date BETWEEN b.start_date AND b.end_date`. This is common for SCD Type 2 lookups and event attribution windows.

**Q3: What happens if you INNER JOIN on a column with NULLs?**  
A: Rows with NULL join keys are excluded because NULL = NULL evaluates to UNKNOWN, not TRUE.

**Q4: How do you avoid duplicate rows when joining a one-to-many table?**  
A: Either aggregate the many-side first, use EXISTS for a semi-join, or apply DISTINCT on the primary key.

**Q5: When would you use FULL OUTER JOIN in practice?**  
A: Data reconciliation between two sources, union-style comparisons (before/after migration), or finding mismatches between predicted and actual datasets.

**Q6: What is a hash join vs. a merge join?**  
A: Hash join builds a hash table from the smaller table and probes it with the larger. Merge join sorts both tables on the join key and merges in order. Merge is better when data is already sorted.

**Q7: How do you handle slowly changing dimensions (SCD Type 2) in joins?**  
A: Join on business key AND date range: `ON fact.dim_key = dim.dim_key AND fact.event_date BETWEEN dim.effective_start AND dim.effective_end`.

**Q8: What is a semi-join?**  
A: Returns rows from the left table where a match EXISTS in the right table, without duplicating rows. Implemented via EXISTS or IN.

**Q9: How does CROSS JOIN differ from an unintentional Cartesian product?**  
A: Semantically identical, but CROSS JOIN is intentional and explicit. Accidental Cartesian products come from missing ON conditions.

**Q10: In Spark/BigQuery, when should you repartition before a join?**  
A: When both tables are large and not already partitioned on the join key. Repartitioning (bucketing) avoids full shuffles on repeated joins.

---

## 16. Summary Cheat Sheet

```
+------------------------------------------------------------------+
|              ADVANCED SQL JOINS -- CHEAT SHEET                    |
+------------------------------------------------------------------+
|                                                                  |
|  JOIN TYPE SELECTION:                                            |
|    Need ALL left rows?          --> LEFT JOIN                    |
|    Need only matches?           --> INNER JOIN                   |
|    Need rows from BOTH sides?   --> FULL OUTER JOIN              |
|    Need every combination?      --> CROSS JOIN                   |
|    Need to compare within table?--> SELF JOIN                   |
|                                                                  |
|  ANTI/SEMI PATTERNS:                                            |
|    Rows WITHOUT match:  LEFT JOIN ... WHERE right.key IS NULL   |
|    Rows WITH match:     WHERE EXISTS (SELECT 1 FROM ...)        |
|    Avoid:               NOT IN (subquery that may have NULLs)   |
|                                                                  |
|  PERFORMANCE RULES:                                             |
|    1. Index all join columns                                    |
|    2. Filter early (reduce rows before joining)                 |
|    3. Aggregate before joining fact tables                      |
|    4. Broadcast small tables in distributed systems             |
|    5. Use EXPLAIN to verify join strategies                     |
|                                                                  |
|  TRAP AVOIDANCE:                                                |
|    - ON vs WHERE in LEFT JOIN (filter in ON to keep NULLs)     |
|    - NULL != NULL (always filter NULLs from join keys)         |
|    - Many-to-many = row explosion (pre-aggregate!)             |
|    - DISTINCT is a band-aid, not a fix for bad joins           |
|                                                                  |
|  INTERVIEW FRAMEWORK:                                           |
|    1. Clarify the grain (what is one row?)                     |
|    2. Identify the driver table                                 |
|    3. Choose join types based on completeness needs             |
|    4. Handle NULLs explicitly                                   |
|    5. Mention performance considerations                        |
|                                                                  |
+------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Topics](README.md) collection -- Topic 1 of 45*
