# Topic 7: Deduplication & Data Cleaning in SQL

> **Data Science Interview -- Deep Dive**  
> Finding and eliminating duplicates, handling NULLs gracefully, cleaning strings and dates, casting data types safely, and building robust data pipelines that produce trustworthy analytical datasets.

---

## Table of Contents

1. [Why Data Cleaning Matters in Interviews](#1-why-data-cleaning-matters-in-interviews)
2. [Finding Duplicates -- Detection Patterns](#2-finding-duplicates--detection-patterns)
3. [ROW_NUMBER for Deduplication -- Keeping One](#3-row_number-for-deduplication--keeping-one)
4. [Exact vs Fuzzy Deduplication](#4-exact-vs-fuzzy-deduplication)
5. [Handling NULLs -- COALESCE, NULLIF, IS DISTINCT FROM](#5-handling-nulls--coalesce-nullif-is-distinct-from)
6. [Data Type Casting and Conversion](#6-data-type-casting-and-conversion)
7. [String Cleaning Patterns](#7-string-cleaning-patterns)
8. [Date Handling Edge Cases](#8-date-handling-edge-cases)
9. [Data Quality Validation Queries](#9-data-quality-validation-queries)
10. [Interview Problems with Solutions](#10-interview-problems-with-solutions)
11. [Interview Talking Points](#11-interview-talking-points)
12. [Common Interview Mistakes](#12-common-interview-mistakes)
13. [Rapid-Fire Q&A](#13-rapid-fire-qa)
14. [Summary Cheat Sheet](#14-summary-cheat-sheet)

---

## 1. Why Data Cleaning Matters in Interviews

At 5 YOE, interviewers expect you to:

- Recognize that real-world data is messy (duplicates, NULLs, inconsistent formats)
- Proactively mention data quality assumptions when writing queries
- Know idiomatic SQL patterns for cleaning without destroying information
- Understand when deduplication changes analytical results

> **Critical Insight:** In a DS interview, if you start writing an aggregation query without first acknowledging potential duplicates or NULLs, you signal inexperience with production data. Always say: "Before aggregating, I'd check for duplicates on the join key" or "I'll handle NULLs explicitly with COALESCE."

---

## 2. Finding Duplicates -- Detection Patterns

### Pattern 1: GROUP BY + HAVING COUNT > 1

```sql
-- Find email addresses that appear more than once
SELECT 
    email,
    COUNT(*) AS occurrence_count
FROM users
GROUP BY email
HAVING COUNT(*) > 1
ORDER BY occurrence_count DESC;
```

### Pattern 2: Full Row Duplicates

```sql
-- Find completely identical rows (all columns match)
SELECT 
    user_id, email, name, signup_date,
    COUNT(*) AS dup_count
FROM users
GROUP BY user_id, email, name, signup_date
HAVING COUNT(*) > 1;
```

### Pattern 3: Duplicates on Composite Key

```sql
-- Events table: same user, same event type, same timestamp = likely duplicate
SELECT 
    user_id,
    event_type,
    event_timestamp,
    COUNT(*) AS dup_count
FROM events
GROUP BY user_id, event_type, event_timestamp
HAVING COUNT(*) > 1;
```

### Pattern 4: Near-Duplicates (Within Time Window)

```sql
-- Orders placed by same user within 5 seconds (likely double-submit)
SELECT 
    o1.order_id AS order_1,
    o2.order_id AS order_2,
    o1.user_id,
    o1.order_time,
    o2.order_time,
    ABS(EXTRACT(EPOCH FROM o2.order_time - o1.order_time)) AS seconds_apart
FROM orders o1
JOIN orders o2 
    ON o1.user_id = o2.user_id 
    AND o1.order_id < o2.order_id  -- avoid self-match and double-counting
    AND o1.amount = o2.amount
    AND ABS(EXTRACT(EPOCH FROM o2.order_time - o1.order_time)) < 5;
```

### Duplicate Audit Summary

```sql
-- Comprehensive duplicate report
SELECT 
    'users' AS table_name,
    'email' AS dedup_key,
    COUNT(*) AS total_rows,
    COUNT(DISTINCT email) AS unique_keys,
    COUNT(*) - COUNT(DISTINCT email) AS duplicate_rows,
    ROUND(100.0 * (COUNT(*) - COUNT(DISTINCT email)) / COUNT(*), 2) AS dup_pct
FROM users;
```

---

## 3. ROW_NUMBER for Deduplication -- Keeping One

The most common interview deduplication pattern: assign a row number within each duplicate group, then keep only row 1.

### Keep the Most Recent Record

```sql
-- Deduplicate users: keep the most recently updated record per email
WITH ranked AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (
            PARTITION BY email 
            ORDER BY updated_at DESC
        ) AS rn
    FROM users
)
SELECT * FROM ranked WHERE rn = 1;
```

### Keep the First Record (Earliest)

```sql
-- Deduplicate orders: keep the first order per (user, product) pair
WITH ranked AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (
            PARTITION BY user_id, product_id 
            ORDER BY order_date ASC
        ) AS rn
    FROM orders
)
SELECT * FROM ranked WHERE rn = 1;
```

### Delete Duplicates (Mutation)

```sql
-- PostgreSQL: Delete duplicate rows, keeping the one with lowest ctid
DELETE FROM users
WHERE ctid NOT IN (
    SELECT MIN(ctid)
    FROM users
    GROUP BY email
);

-- Alternative using CTE (safer, more readable)
WITH duplicates AS (
    SELECT 
        ctid,
        ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_at DESC) AS rn
    FROM users
)
DELETE FROM users
WHERE ctid IN (SELECT ctid FROM duplicates WHERE rn > 1);
```

### Choosing the "Winner" -- Multi-Criteria Ordering

```sql
-- Complex priority: prefer verified emails, then most recent, then highest ID
WITH ranked AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (
            PARTITION BY email 
            ORDER BY 
                is_verified DESC,       -- verified first
                last_login_date DESC,   -- most recent login
                user_id DESC            -- tiebreaker: highest ID
        ) AS rn
    FROM users
)
SELECT * FROM ranked WHERE rn = 1;
```

> **Critical Insight:** Always explain your ORDER BY logic when deduplicating in an interview. The interviewer wants to know WHY you chose to keep a particular record. Common strategies: most recent (freshest data), most complete (fewest NULLs), or business-specific priority (verified > unverified).

---

## 4. Exact vs Fuzzy Deduplication

### Exact Deduplication

All key columns must match precisely. Use GROUP BY or ROW_NUMBER as shown above.

### Fuzzy Deduplication -- When Values Are "Close But Not Identical"

```sql
-- Fuzzy matching on names: normalize before comparing
WITH normalized AS (
    SELECT 
        *,
        LOWER(TRIM(REGEXP_REPLACE(name, '\s+', ' ', 'g'))) AS clean_name,
        LOWER(TRIM(email)) AS clean_email
    FROM customers
),
fuzzy_matches AS (
    SELECT 
        a.customer_id AS id_a,
        b.customer_id AS id_b,
        a.name AS name_a,
        b.name AS name_b,
        a.clean_email
    FROM normalized a
    JOIN normalized b 
        ON a.clean_email = b.clean_email
        AND a.customer_id < b.customer_id  -- avoid self and double matches
        AND (
            a.clean_name = b.clean_name
            OR SIMILARITY(a.clean_name, b.clean_name) > 0.7  -- pg_trgm
        )
)
SELECT * FROM fuzzy_matches;
```

### Levenshtein Distance for Typo Detection

```sql
-- PostgreSQL with fuzzystrmatch extension
SELECT 
    a.customer_id,
    b.customer_id,
    a.name,
    b.name,
    LEVENSHTEIN(LOWER(a.name), LOWER(b.name)) AS edit_distance
FROM customers a
JOIN customers b 
    ON a.customer_id < b.customer_id
    AND a.email = b.email
WHERE LEVENSHTEIN(LOWER(a.name), LOWER(b.name)) <= 2;  -- max 2 character differences
```

### Comparison Table

| Method | Precision | Recall | Use Case |
|--------|-----------|--------|----------|
| Exact match (GROUP BY) | Perfect | Low (misses typos) | Clean data, system-generated IDs |
| Normalized match (LOWER/TRIM) | High | Medium | User-entered text with case/whitespace issues |
| Similarity / Trigram | Medium | High | Name matching across sources |
| Levenshtein distance | Medium | Medium | Typo detection in short strings |
| Phonetic (Soundex/Metaphone) | Low | High | Names that sound alike but spell differently |

---

## 5. Handling NULLs -- COALESCE, NULLIF, IS DISTINCT FROM

### NULL Behavior Summary

| Operation | Result |
|-----------|--------|
| `NULL = NULL` | NULL (not TRUE) |
| `NULL <> NULL` | NULL (not TRUE) |
| `NULL AND TRUE` | NULL |
| `NULL OR TRUE` | TRUE |
| `SUM(col)` with all NULLs | NULL (not 0) |
| `COUNT(col)` | Excludes NULLs |
| `COUNT(*)` | Includes NULLs |
| `NULL IN (1, 2, NULL)` | NULL (not TRUE) |

### COALESCE -- Replace NULLs with Defaults

```sql
-- Replace NULL values with sensible defaults
SELECT 
    user_id,
    COALESCE(display_name, username, email, 'Anonymous') AS name,
    COALESCE(country, 'Unknown') AS country,
    COALESCE(lifetime_value, 0) AS lifetime_value
FROM users;

-- Useful in aggregations
SELECT 
    COALESCE(region, 'Unassigned') AS region,
    SUM(COALESCE(revenue, 0)) AS total_revenue
FROM sales
GROUP BY COALESCE(region, 'Unassigned');
```

### NULLIF -- Create NULLs from Sentinel Values

```sql
-- Convert empty strings or placeholder values to NULL
SELECT 
    user_id,
    NULLIF(phone, '') AS phone,            -- '' becomes NULL
    NULLIF(country, 'N/A') AS country,     -- 'N/A' becomes NULL
    NULLIF(age, 0) AS age,                 -- 0 (invalid) becomes NULL
    NULLIF(revenue, -1) AS revenue         -- -1 (sentinel) becomes NULL
FROM raw_users;

-- Prevent division by zero
SELECT 
    campaign_name,
    impressions,
    clicks,
    ROUND(100.0 * clicks / NULLIF(impressions, 0), 2) AS ctr_pct  -- returns NULL instead of error
FROM campaigns;
```

### IS DISTINCT FROM -- NULL-Safe Comparison

```sql
-- Standard equality fails with NULLs:
-- WHERE a.status = b.status  --> FALSE when both are NULL

-- IS DISTINCT FROM treats NULLs as equal:
SELECT *
FROM table_a a
FULL OUTER JOIN table_b b ON a.id = b.id
WHERE a.value IS DISTINCT FROM b.value;  -- catches: NULL vs 'X', 'X' vs 'Y', 'X' vs NULL

-- IS NOT DISTINCT FROM (NULL-safe equality):
-- NULL IS NOT DISTINCT FROM NULL --> TRUE
SELECT *
FROM users
WHERE status IS NOT DISTINCT FROM 'active';  -- matches 'active'; does NOT match NULL
```

> **Critical Insight:** IS DISTINCT FROM is the correct way to compare columns that may contain NULLs when you want NULL-to-NULL to be treated as "equal." Use it in CDC (Change Data Capture) logic, data reconciliation, and SCD Type 2 implementations.

### NULL-Aware Aggregation Patterns

```sql
-- Count NULLs explicitly
SELECT 
    COUNT(*) AS total_rows,
    COUNT(email) AS has_email,
    COUNT(*) - COUNT(email) AS missing_email,
    ROUND(100.0 * (COUNT(*) - COUNT(email)) / COUNT(*), 2) AS null_pct
FROM users;

-- Conditional aggregation with NULL handling
SELECT 
    department,
    AVG(salary) AS avg_salary_excl_null,                    -- NULLs ignored
    AVG(COALESCE(salary, 0)) AS avg_salary_null_as_zero,   -- NULLs = 0
    SUM(CASE WHEN salary IS NULL THEN 1 ELSE 0 END) AS null_count
FROM employees
GROUP BY department;
```

---

## 6. Data Type Casting and Conversion

### Safe Casting Patterns

```sql
-- PostgreSQL: CAST vs :: shorthand
SELECT 
    CAST(order_id AS TEXT) AS order_id_str,
    price::NUMERIC(10,2) AS price_decimal,
    event_date::DATE AS date_only,
    CAST(json_field->>'amount' AS NUMERIC) AS amount
FROM orders;

-- Safe cast (returns NULL instead of error) -- PostgreSQL doesn't have TRY_CAST natively
-- Workaround: use regex validation first
SELECT 
    raw_value,
    CASE 
        WHEN raw_value ~ '^\d+\.?\d*$' THEN raw_value::NUMERIC 
        ELSE NULL 
    END AS safe_numeric
FROM raw_data;

-- BigQuery/SQL Server: TRY_CAST or SAFE_CAST
-- SELECT SAFE_CAST(value AS INT64) FROM table;  -- returns NULL on failure
```

### Common Type Issues in Interviews

| Problem | Symptom | Fix |
|---------|---------|-----|
| Joining INT to VARCHAR | Implicit cast prevents index use | Explicitly cast to matching type |
| Storing dates as strings | Date arithmetic fails or gives wrong results | Cast to DATE/TIMESTAMP before operations |
| Decimal precision loss | `1/3 = 0` (integer division) | Cast to NUMERIC before dividing |
| JSON field extraction | Returns TEXT, not numeric | Cast extracted value: `->>'field'::INT` |
| Boolean as 0/1 | Cannot use in boolean expressions | `CASE` or `CAST(col AS BOOLEAN)` |

```sql
-- Integer division trap
SELECT 3 / 4;                    -- Returns 0 (integer division)
SELECT 3.0 / 4;                  -- Returns 0.75
SELECT CAST(3 AS NUMERIC) / 4;   -- Returns 0.75
SELECT 3::FLOAT / 4;             -- Returns 0.75
```

---

## 7. String Cleaning Patterns

### Whitespace and Case Normalization

```sql
SELECT 
    -- Remove leading/trailing whitespace
    TRIM(name) AS trimmed_name,
    
    -- Remove specific characters
    TRIM(BOTH '"' FROM quoted_value) AS unquoted,
    
    -- Normalize internal whitespace (multiple spaces -> single space)
    REGEXP_REPLACE(TRIM(name), '\s+', ' ', 'g') AS normalized_name,
    
    -- Standardize case
    LOWER(email) AS email_lower,
    UPPER(country_code) AS country_upper,
    INITCAP(city) AS city_proper
FROM raw_customers;
```

### Extracting and Parsing

```sql
-- Extract domain from email
SELECT 
    email,
    SPLIT_PART(email, '@', 2) AS domain,
    SPLIT_PART(email, '@', 1) AS local_part
FROM users;

-- Parse structured strings
SELECT 
    full_address,
    SPLIT_PART(full_address, ',', 1) AS street,
    TRIM(SPLIT_PART(full_address, ',', 2)) AS city,
    TRIM(SPLIT_PART(full_address, ',', 3)) AS state_zip
FROM addresses;

-- Regex extraction
SELECT 
    description,
    REGEXP_REPLACE(phone, '[^0-9]', '', 'g') AS digits_only,
    (REGEXP_MATCH(url, 'https?://([^/]+)'))[1] AS hostname
FROM contacts;
```

### Common String Cleaning Tasks

```sql
-- Remove non-printable characters
SELECT REGEXP_REPLACE(raw_text, '[^\x20-\x7E]', '', 'g') AS clean_text FROM raw_data;

-- Standardize phone numbers
SELECT 
    raw_phone,
    REGEXP_REPLACE(raw_phone, '[^0-9+]', '', 'g') AS cleaned,
    CASE 
        WHEN LENGTH(REGEXP_REPLACE(raw_phone, '[^0-9]', '', 'g')) = 10 
        THEN '+1' || REGEXP_REPLACE(raw_phone, '[^0-9]', '', 'g')
        ELSE REGEXP_REPLACE(raw_phone, '[^0-9+]', '', 'g')
    END AS standardized_phone
FROM contacts;

-- Fix encoding issues (common with imported CSV data)
SELECT REPLACE(REPLACE(name, 'é', 'e'), 'ñ', 'n') AS ascii_name
FROM imported_data;
```

---

## 8. Date Handling Edge Cases

### Timezone Traps

```sql
-- Converting between timezones
SELECT 
    event_timestamp,
    event_timestamp AT TIME ZONE 'UTC' AS utc_time,
    event_timestamp AT TIME ZONE 'America/Los_Angeles' AS pacific_time,
    -- CAUTION: timestamp vs timestamptz behave differently
    event_timestamp::DATE AS date_in_session_tz  -- depends on session timezone!
FROM events;

-- Safe date extraction regardless of timezone
SELECT 
    (event_timestamp AT TIME ZONE 'UTC')::DATE AS utc_date
FROM events;
```

### Month Boundary Issues

```sql
-- Adding months: what happens on Jan 31 + 1 month?
SELECT DATE '2024-01-31' + INTERVAL '1 month';  -- 2024-02-29 (leap year handling)
SELECT DATE '2024-01-31' + INTERVAL '1 month' + INTERVAL '1 month';  -- 2024-03-29 (NOT 03-31!)

-- Safer: use DATE_TRUNC for month-start anchoring
SELECT DATE_TRUNC('month', signup_date) + INTERVAL '1 month' - INTERVAL '1 day' AS end_of_signup_month
FROM users;
```

### Leap Year and DST Issues

```sql
-- Leap year: Feb 29 exists in 2024 but not 2023
SELECT DATE '2024-02-29' - INTERVAL '1 year';  -- 2023-02-28 (PostgreSQL adjusts)

-- DST gap: 2024-03-10 02:30 does not exist in America/New_York
-- Interval arithmetic crossing DST can produce unexpected hour shifts
SELECT 
    '2024-03-09 23:00:00'::TIMESTAMP AT TIME ZONE 'America/New_York' + INTERVAL '4 hours'
    -- Expected: 2024-03-10 03:00 (not 04:00 because clock jumps forward)
```

### Date Arithmetic for Analytics

```sql
-- Days between events
SELECT 
    user_id,
    order_date,
    LAG(order_date) OVER (PARTITION BY user_id ORDER BY order_date) AS prev_order,
    order_date - LAG(order_date) OVER (PARTITION BY user_id ORDER BY order_date) AS days_between
FROM orders;

-- Week number (ISO vs US definition)
SELECT 
    order_date,
    EXTRACT(ISODOW FROM order_date) AS iso_day_of_week,  -- 1=Mon, 7=Sun
    EXTRACT(DOW FROM order_date) AS us_day_of_week,      -- 0=Sun, 6=Sat
    EXTRACT(WEEK FROM order_date) AS iso_week_number,
    TO_CHAR(order_date, 'WW') AS week_of_year
FROM orders;

-- Date spine generation (for filling gaps)
SELECT generate_series('2024-01-01'::DATE, '2024-12-31'::DATE, '1 day')::DATE AS calendar_date;
```

---

## 9. Data Quality Validation Queries

### Comprehensive Data Quality Check

```sql
-- Column-level quality report
SELECT 
    'user_id' AS column_name,
    COUNT(*) AS total_rows,
    COUNT(user_id) AS non_null,
    COUNT(*) - COUNT(user_id) AS null_count,
    COUNT(DISTINCT user_id) AS distinct_count,
    CASE WHEN COUNT(*) = COUNT(DISTINCT user_id) THEN 'UNIQUE' ELSE 'HAS DUPES' END AS uniqueness
FROM users

UNION ALL

SELECT 
    'email',
    COUNT(*),
    COUNT(email),
    COUNT(*) - COUNT(email),
    COUNT(DISTINCT email),
    CASE WHEN COUNT(email) = COUNT(DISTINCT email) THEN 'UNIQUE' ELSE 'HAS DUPES' END
FROM users;
```

### Referential Integrity Check

```sql
-- Find orphaned orders (no matching user)
SELECT o.order_id, o.user_id
FROM orders o
LEFT JOIN users u ON o.user_id = u.user_id
WHERE u.user_id IS NULL;

-- Find orders with future dates (data quality issue)
SELECT order_id, order_date
FROM orders
WHERE order_date > CURRENT_DATE;

-- Find impossible values
SELECT *
FROM orders
WHERE amount < 0
   OR quantity <= 0
   OR unit_price < 0;
```

### Schema Drift Detection

```sql
-- Check for unexpected values in categorical columns
SELECT 
    status,
    COUNT(*) AS row_count,
    MIN(created_at) AS first_seen,
    MAX(created_at) AS last_seen
FROM orders
GROUP BY status
ORDER BY row_count DESC;
-- Compare against expected values: 'pending', 'shipped', 'delivered', 'cancelled'
```

---

## 10. Interview Problems with Solutions

### Problem 1: Deduplicate Events with Business Logic

**Task:** An events table has duplicates from double-firing. Keep one event per (user, event_type) per 5-minute window, choosing the one with the most metadata filled in.

```sql
WITH windowed AS (
    SELECT 
        *,
        -- Create 5-minute window buckets
        DATE_TRUNC('hour', event_time) + 
            INTERVAL '5 min' * FLOOR(EXTRACT(MINUTE FROM event_time) / 5) AS time_bucket
    FROM events
),
ranked AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (
            PARTITION BY user_id, event_type, time_bucket
            ORDER BY 
                -- Prefer records with more non-null fields
                (CASE WHEN page_url IS NOT NULL THEN 1 ELSE 0 END +
                 CASE WHEN referrer IS NOT NULL THEN 1 ELSE 0 END +
                 CASE WHEN device_id IS NOT NULL THEN 1 ELSE 0 END) DESC,
                event_time ASC  -- tiebreak: earliest event
        ) AS rn
    FROM windowed
)
SELECT * FROM ranked WHERE rn = 1;
```

### Problem 2: Merge Duplicate Customers

**Task:** Same customer appears multiple times with different completeness. Merge into one golden record with the best available data for each field.

```sql
WITH customer_groups AS (
    -- Group duplicates by normalized email
    SELECT 
        LOWER(TRIM(email)) AS group_key,
        customer_id,
        name,
        phone,
        address,
        created_at
    FROM customers
),
golden_record AS (
    SELECT DISTINCT ON (group_key)
        group_key AS email,
        -- For each field, pick the most recent non-null value
        FIRST_VALUE(name) OVER (
            PARTITION BY group_key 
            ORDER BY CASE WHEN name IS NOT NULL THEN 0 ELSE 1 END, created_at DESC
        ) AS best_name,
        FIRST_VALUE(phone) OVER (
            PARTITION BY group_key 
            ORDER BY CASE WHEN phone IS NOT NULL THEN 0 ELSE 1 END, created_at DESC
        ) AS best_phone,
        FIRST_VALUE(address) OVER (
            PARTITION BY group_key 
            ORDER BY CASE WHEN address IS NOT NULL THEN 0 ELSE 1 END, created_at DESC
        ) AS best_address,
        MIN(created_at) OVER (PARTITION BY group_key) AS first_created
    FROM customer_groups
    ORDER BY group_key, created_at DESC
)
SELECT * FROM golden_record;
```

### Problem 3: Fill Gaps in Time-Series Data

**Task:** A daily metrics table has missing dates. Fill them with the previous day's value (forward-fill).

```sql
WITH date_spine AS (
    SELECT generate_series(
        (SELECT MIN(metric_date) FROM daily_metrics),
        (SELECT MAX(metric_date) FROM daily_metrics),
        '1 day'::INTERVAL
    )::DATE AS dt
),
with_gaps AS (
    SELECT 
        ds.dt,
        dm.metric_value
    FROM date_spine ds
    LEFT JOIN daily_metrics dm ON ds.dt = dm.metric_date
),
forward_filled AS (
    SELECT 
        dt,
        metric_value,
        -- Forward fill: use last non-null value
        COALESCE(
            metric_value,
            LAG(metric_value) IGNORE NULLS OVER (ORDER BY dt)  -- Snowflake/BigQuery syntax
        ) AS filled_value
    FROM with_gaps
)
SELECT * FROM forward_filled;

-- PostgreSQL alternative (no IGNORE NULLS):
WITH date_spine AS (
    SELECT generate_series(MIN(metric_date), MAX(metric_date), '1 day')::DATE AS dt
    FROM daily_metrics
),
with_groups AS (
    SELECT 
        ds.dt,
        dm.metric_value,
        COUNT(dm.metric_value) OVER (ORDER BY ds.dt) AS grp
    FROM date_spine ds
    LEFT JOIN daily_metrics dm ON ds.dt = dm.metric_date
)
SELECT 
    dt,
    FIRST_VALUE(metric_value) OVER (PARTITION BY grp ORDER BY dt) AS filled_value
FROM with_groups;
```

### Problem 4: Identify and Flag Data Quality Issues

**Task:** Create a quality flag for each order record.

```sql
SELECT 
    order_id,
    user_id,
    amount,
    order_date,
    CASE 
        WHEN user_id IS NULL THEN 'MISSING_USER'
        WHEN amount IS NULL OR amount <= 0 THEN 'INVALID_AMOUNT'
        WHEN order_date > CURRENT_DATE THEN 'FUTURE_DATE'
        WHEN order_date < '2020-01-01' THEN 'SUSPICIOUS_OLD_DATE'
        WHEN NOT EXISTS (SELECT 1 FROM users u WHERE u.user_id = orders.user_id) THEN 'ORPHANED'
        ELSE 'CLEAN'
    END AS quality_flag
FROM orders;
```

---

## 11. Interview Talking Points

> "My deduplication approach depends on the business context. I first identify duplicates using GROUP BY HAVING COUNT > 1 to understand the scale. Then I use ROW_NUMBER with a carefully chosen ORDER BY to pick the 'winner' -- usually the most recent record or the one with the fewest NULLs."

> "For NULL handling, I'm explicit about the behavior I want. AVG ignores NULLs by default -- which changes the denominator. If NULLs represent 'zero' in my context (e.g., zero revenue for a product that had no sales), I use COALESCE(col, 0) before aggregating."

> "IS DISTINCT FROM is my go-to for NULL-safe comparisons, especially in data reconciliation queries where I'm comparing two tables. Standard equality (=) treats NULL as 'unknown,' which means two NULL values are not considered equal -- usually not what you want for dedup or CDC."

> "In production pipelines, I build data quality gates: queries that count NULLs, detect orphaned foreign keys, flag impossible values, and check cardinality expectations. These run before downstream aggregations to catch issues early."

---

## 12. Common Interview Mistakes

| Mistake | Why It's Wrong | Correct Approach |
|---------|---------------|-----------------|
| Using DISTINCT instead of proper dedup | DISTINCT removes all columns' duplicates, cannot choose which row to keep | Use ROW_NUMBER with explicit ordering |
| Assuming NULL = NULL is TRUE | It evaluates to NULL (unknown), not TRUE | Use IS NOT DISTINCT FROM for NULL-safe equality |
| Forgetting NULLs in NOT IN subqueries | `WHERE x NOT IN (1, 2, NULL)` returns no rows | Use NOT EXISTS or filter NULLs from subquery |
| COALESCE with wrong default | Using 0 when NULL means "unknown" (not zero) | Choose default that preserves semantics |
| String comparison without normalization | 'John' != 'john' != ' John ' | LOWER(TRIM(col)) before comparing |
| Integer division in percentage calculations | 3/4 = 0 in SQL | Cast numerator to FLOAT or NUMERIC first |
| Ignoring timezone in date comparisons | Comparing DATE to TIMESTAMP crosses timezone boundary | Explicit timezone conversion before comparison |
| Deduplicating without understanding cause | May mask a real bug in the data pipeline | Investigate root cause; dedup is a band-aid |
| Using DELETE without WHERE clause safety | Accidental full table delete | Always test with SELECT first; wrap in transaction |
| Assuming data types from column names | Column named 'date' may be VARCHAR | Check schema; CAST explicitly |

---

## 13. Rapid-Fire Q&A

**Q1: What is the difference between DISTINCT and GROUP BY for deduplication?**  
A: DISTINCT removes duplicate result rows across ALL selected columns. GROUP BY lets you aggregate (COUNT, SUM) the duplicates. For simple dedup they produce the same result, but GROUP BY gives you more control.

**Q2: How does NULL behave in a UNIQUE constraint?**  
A: In PostgreSQL, multiple NULLs are allowed in a UNIQUE column (NULLs are not considered equal). In SQL Server, only one NULL is allowed by default.

**Q3: What does COALESCE(a, b, c) do?**  
A: Returns the first non-NULL value from the list. If all are NULL, returns NULL.

**Q4: When would you use NULLIF?**  
A: To convert known sentinel values (empty string, -1, 0, 'N/A') into NULL so they behave correctly in aggregations and COALESCE chains.

**Q5: How do you deduplicate when there is no natural ordering column?**  
A: Use ctid (PostgreSQL physical row ID) or an arbitrary tiebreaker. In production, this signals a schema problem -- add a created_at timestamp.

**Q6: What happens to NULLs in GROUP BY?**  
A: All NULLs form one group. This is an exception to the rule that NULL != NULL.

**Q7: How do you compare two tables for differences?**  
A: EXCEPT (set difference) for row-level comparison, or FULL OUTER JOIN with IS DISTINCT FROM for column-level difference detection.

**Q8: What is the performance cost of DISTINCT?**  
A: Requires sorting or hashing all rows. On large datasets, it can be expensive. If you know data is unique (e.g., after GROUP BY), omit DISTINCT.

**Q9: How do you handle dates stored as strings in 'MM/DD/YYYY' format?**  
A: `TO_DATE(date_str, 'MM/DD/YYYY')` in PostgreSQL, `STR_TO_DATE(date_str, '%m/%d/%Y')` in MySQL, `PARSE_DATE('%m/%d/%Y', date_str)` in BigQuery.

**Q10: What is idempotent deduplication?**  
A: A dedup operation that produces the same result regardless of how many times you run it. INSERT ... ON CONFLICT DO NOTHING or MERGE/UPSERT patterns are idempotent; DELETE with ROW_NUMBER is idempotent if the ordering is deterministic.

---

## 14. Summary Cheat Sheet

```
+------------------------------------------------------------------+
|  DEDUPLICATION & DATA CLEANING -- CHEAT SHEET                    |
+------------------------------------------------------------------+
|                                                                    |
|  FINDING DUPLICATES:                                               |
|    GROUP BY key_cols HAVING COUNT(*) > 1                           |
|    Self-join with id1 < id2 for pair detection                     |
|                                                                    |
|  REMOVING DUPLICATES:                                              |
|    ROW_NUMBER() OVER (PARTITION BY key ORDER BY priority) = 1      |
|    Always explain your ORDER BY (why this record wins)             |
|                                                                    |
|  NULL HANDLING TOOLKIT:                                            |
|    COALESCE(a, b, c)      -- first non-null                        |
|    NULLIF(val, sentinel)  -- convert sentinels to NULL             |
|    IS DISTINCT FROM       -- NULL-safe != comparison               |
|    IS NOT DISTINCT FROM   -- NULL-safe = comparison                |
|    COUNT(col) vs COUNT(*) -- excludes vs includes NULLs            |
|                                                                    |
|  STRING CLEANING:                                                  |
|    LOWER(TRIM(col))                -- normalize case/whitespace    |
|    REGEXP_REPLACE(col, pattern, '') -- remove unwanted chars        |
|    SPLIT_PART(col, delim, pos)     -- extract substrings           |
|                                                                    |
|  DATE EDGE CASES:                                                  |
|    - Timezone: always convert to UTC before date comparison        |
|    - Month arithmetic: Jan 31 + 1 month != always Feb 28/29       |
|    - Date spine: generate_series to fill gaps                      |
|    - Forward fill: LAG IGNORE NULLS or COUNT()+FIRST_VALUE()       |
|                                                                    |
|  TYPE CASTING:                                                     |
|    - Always cast before arithmetic (avoid integer division)        |
|    - Validate with regex before unsafe casts                       |
|    - Use SAFE_CAST/TRY_CAST where available                        |
|                                                                    |
|  INTERVIEW APPROACH:                                               |
|    1. Acknowledge data quality issues before writing queries        |
|    2. State your dedup strategy and WHY                            |
|    3. Handle NULLs explicitly (never leave to default behavior)    |
|    4. Validate: count before/after dedup to verify correctness     |
|                                                                    |
+------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Topics](README.md) collection -- Topic 7 of 45*
