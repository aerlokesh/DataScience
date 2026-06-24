# 🎯 Topic 10: Slowly Changing Dimensions (SCDs)

> *When business attributes change over time — address updates, price adjustments, status transitions — how you handle history determines whether your warehouse can answer "what was true THEN?" versus only "what is true NOW?"*

---

## Table of Contents

1. [Introduction](#introduction)
2. [Why Dimensions Change](#why-dimensions-change)
3. [SCD Type 0: Fixed/Retain Original](#scd-type-0-fixretain-original)
4. [SCD Type 1: Overwrite](#scd-type-1-overwrite)
5. [SCD Type 2: Add New Row (Full History)](#scd-type-2-add-new-row)
6. [SCD Type 3: Add Previous Value Column](#scd-type-3-add-previous-value-column)
7. [SCD Type 4: Mini-Dimension](#scd-type-4-mini-dimension)
8. [SCD Type 6: Hybrid (1+2+3)](#scd-type-6-hybrid)
9. [Comparison Matrix](#comparison-matrix)
10. [SQL Implementation Patterns](#sql-implementation-patterns)
11. [Querying SCD Type 2 Tables](#querying-scd-type-2-tables)
12. [Storage and Performance Tradeoffs](#storage-and-performance-tradeoffs)
13. [Interview Talking Points](#interview-talking-points)
14. [Common Mistakes](#common-mistakes)
15. [Rapid-Fire Q&A](#rapid-fire-qa)
16. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Introduction

A **Slowly Changing Dimension** (SCD) is a dimension whose attribute values change infrequently and unpredictably — unlike rapidly changing attributes (stock price every second) or static attributes (date of birth). The "slowly" means changes happen at business-relevant intervals: a customer moves cities, a product gets reclassified, an employee changes departments.

> **Critical Insight:** The SCD strategy is a business decision, not a technical one. Ask stakeholders: "When a customer moves from NY to CA, should historical sales still show NY (what was true then) or CA (what is true now)?" Different attributes within the SAME dimension can use different SCD types.

---

## Why Dimensions Change

| Change Type | Example | Frequency |
|-------------|---------|-----------|
| Correction | Fix misspelled customer name | Occasional |
| Reclassification | Product moved to different category | Infrequent |
| Status change | Customer upgraded from Basic to Premium | Periodic |
| Physical change | Customer changes address | Infrequent |
| Organizational | Employee transfers departments | Periodic |
| Business rule | Pricing tier thresholds redefined | Rare |

---

## SCD Type 0: Fixed/Retain Original

**Strategy:** Never change the value. Keep the original value forever.

**When to use:** Attributes that should reflect the original state regardless of current reality — customer's original sign-up channel, first product category assignment.

```sql
-- Type 0: original_signup_source never changes
-- even if customer later re-registers through a different channel
CREATE TABLE dim_customer (
    customer_key        INT PRIMARY KEY,
    customer_id         VARCHAR(20),
    original_signup_source  VARCHAR(50),  -- Type 0: NEVER updated
    current_email       VARCHAR(200),     -- Type 1: overwritten
    ...
);
```

---

## SCD Type 1: Overwrite

**Strategy:** Simply overwrite the old value with the new value. No history is preserved.

**When to use:**
- Corrections (typo fixes)
- Attributes where history is irrelevant
- When storage/complexity constraints are tight

```sql
-- Before: customer had wrong zip code
-- customer_key=42, name='Jane Doe', city='Boston', zip='02101'

-- After Type 1 update: zip corrected
UPDATE dim_customer
SET zip_code = '02102'
WHERE customer_key = 42;

-- Result: customer_key=42, name='Jane Doe', city='Boston', zip='02102'
-- History of '02101' is LOST
```

| Pros | Cons |
|------|------|
| Simplest to implement | No history tracking |
| No storage overhead | Cannot answer "what was true then?" |
| No query complexity | Retroactively changes historical analysis |
| Fast updates | Audit trail lost |

---

## SCD Type 2: Add New Row (Full History)

**Strategy:** Insert a new row for each change, creating a full version history. Mark rows with effective/expiration dates and a current flag.

**When to use:**
- When historical accuracy is critical
- Regulatory/compliance requirements
- When you need to analyze trends "as of" a point in time

```sql
CREATE TABLE dim_customer (
    customer_key        INT PRIMARY KEY,         -- Surrogate key (new per version)
    customer_id         VARCHAR(20),             -- Natural key (same across versions)
    customer_name       VARCHAR(200),
    city                VARCHAR(100),
    state               VARCHAR(50),
    segment             VARCHAR(50),
    effective_date      DATE NOT NULL,
    expiration_date     DATE NOT NULL,           -- '9999-12-31' for current
    is_current          BOOLEAN NOT NULL         -- TRUE for active row
);
```

### Example: Customer Moves from NY to CA

```sql
-- BEFORE (1 row):
-- customer_key=42, customer_id='C001', city='New York', state='NY',
--   effective='2020-01-15', expiration='9999-12-31', is_current=TRUE

-- AFTER the move (2 rows):
-- customer_key=42, customer_id='C001', city='New York', state='NY',
--   effective='2020-01-15', expiration='2024-03-01', is_current=FALSE

-- customer_key=87, customer_id='C001', city='Los Angeles', state='CA',
--   effective='2024-03-01', expiration='9999-12-31', is_current=TRUE
```

> **Critical Insight:** In SCD Type 2, the **surrogate key** (customer_key) changes with each version, but the **natural key** (customer_id) stays the same. Fact table rows from 2020-2024 still point to customer_key=42 (NY version), preserving history perfectly.

| Pros | Cons |
|------|------|
| Full history preserved | Table grows with each change |
| Accurate point-in-time analysis | Queries more complex (date ranges) |
| Compliance-friendly | ETL logic more complex |
| Surrogate key per version enables precise joins | Must manage effective/expiration dates |

---

## SCD Type 3: Add Previous Value Column

**Strategy:** Add a column to store the previous value. Only tracks one level of history (current + previous).

**When to use:**
- When you only need "before and after" comparison
- When changes are rare and only the last change matters
- Quick comparison reporting (old vs new territory assignment)

```sql
CREATE TABLE dim_customer (
    customer_key        INT PRIMARY KEY,
    customer_id         VARCHAR(20),
    customer_name       VARCHAR(200),
    current_city        VARCHAR(100),
    previous_city       VARCHAR(100),       -- Only 1 level of history
    city_change_date    DATE,
    current_segment     VARCHAR(50),
    previous_segment    VARCHAR(50),
    segment_change_date DATE
);
```

### Example

```sql
-- BEFORE move:
-- customer_key=42, current_city='New York', previous_city=NULL

-- AFTER move:
UPDATE dim_customer
SET previous_city = current_city,
    current_city = 'Los Angeles',
    city_change_date = '2024-03-01'
WHERE customer_key = 42;

-- Result:
-- customer_key=42, current_city='Los Angeles', previous_city='New York'
```

| Pros | Cons |
|------|------|
| Simple schema | Only 1 level of history |
| No row explosion | Multiple changes lose intermediate states |
| Easy "before/after" queries | Schema changes needed for each tracked attribute |
| Single row per entity | Cannot do point-in-time analysis |

---

## SCD Type 4: Mini-Dimension

**Strategy:** Split rapidly changing attributes into a separate "mini-dimension" table. The fact table has FKs to both the base dimension and the mini-dimension.

**When to use:**
- When a subset of attributes changes frequently (age band, income bracket)
- When the dimension is very large and frequent Type 2 rows would be prohibitive
- Demographics or behavioral segments that shift often

```sql
-- Base dimension (rarely changes)
CREATE TABLE dim_customer (
    customer_key        INT PRIMARY KEY,
    customer_id         VARCHAR(20),
    customer_name       VARCHAR(200),
    signup_date         DATE,
    email               VARCHAR(200)
);

-- Mini-dimension (frequently changing demographics)
CREATE TABLE dim_customer_demographics (
    demo_key            INT PRIMARY KEY,
    age_band            VARCHAR(20),        -- '18-24', '25-34', '35-44'...
    income_bracket      VARCHAR(20),        -- 'low', 'medium', 'high'
    loyalty_tier        VARCHAR(20),        -- 'bronze', 'silver', 'gold'
    urban_rural         VARCHAR(10)
);

-- Fact table references BOTH
CREATE TABLE fact_purchases (
    date_key            INT,
    customer_key        INT,            -- FK to base dimension
    demo_key            INT,            -- FK to mini-dimension (current at time of fact)
    product_key         INT,
    amount              DECIMAL(10,2)
);
```

> **Critical Insight:** The mini-dimension captures the customer's demographic profile AT THE TIME of each transaction. This means different fact rows for the same customer might point to different demo_keys — that's the feature, not a bug.

| Pros | Cons |
|------|------|
| Avoids bloating the base dimension | Two FKs in fact table |
| Efficient for frequently-changing attributes | More joins in queries |
| Mini-dim is small (combinatorial, not per-customer) | Harder to explain to business users |
| Historical accuracy via fact-level FK | Maintenance of two dimensions |

---

## SCD Type 6: Hybrid (1+2+3)

**Strategy:** Combines Type 1 (overwrite), Type 2 (new row), and Type 3 (previous column). Adds a new row for history AND maintains current + previous columns on ALL rows.

**When to use:**
- When you need full history AND easy access to current values
- When users frequently need to group historical facts by current attributes
- Most complex but most flexible

```sql
CREATE TABLE dim_customer (
    customer_key        INT PRIMARY KEY,         -- New per version (Type 2)
    customer_id         VARCHAR(20),             -- Natural key
    customer_name       VARCHAR(200),
    historical_city     VARCHAR(100),            -- Value at this version's time
    current_city        VARCHAR(100),            -- Always the latest (Type 1 overwrite)
    previous_city       VARCHAR(100),            -- One-back reference (Type 3)
    effective_date      DATE,
    expiration_date     DATE,
    is_current          BOOLEAN
);
```

### Example: Customer moves NY -> CA -> TX

```sql
-- Row 1 (expired):
-- key=42, hist_city='New York', curr_city='Texas', prev_city='California',
--   eff='2020-01-15', exp='2024-03-01', is_current=FALSE

-- Row 2 (expired):
-- key=87, hist_city='California', curr_city='Texas', prev_city='California',
--   eff='2024-03-01', exp='2025-08-10', is_current=FALSE

-- Row 3 (current):
-- key=103, hist_city='Texas', curr_city='Texas', prev_city='California',
--   eff='2025-08-10', exp='9999-12-31', is_current=TRUE

-- Note: current_city='Texas' on ALL rows (Type 1 overwrites ALL versions)
```

| Pros | Cons |
|------|------|
| Full history + current state | Most complex ETL |
| Flexible querying | Must update ALL rows on change (Type 1 on current_city) |
| Supports both historical and current reporting | Schema is wider |
| Best of all worlds | Harder to maintain and explain |

---

## Comparison Matrix

| Aspect | Type 0 | Type 1 | Type 2 | Type 3 | Type 4 | Type 6 |
|--------|--------|--------|--------|--------|--------|--------|
| History | None | None | Full | 1 level | Via fact FK | Full + current |
| Rows per entity | 1 | 1 | Many | 1 | 1 + mini-dim | Many |
| Storage growth | None | None | High | Low | Moderate | High |
| ETL complexity | Trivial | Low | Medium | Low | Medium | High |
| Query complexity | Trivial | Trivial | Medium | Low | Medium | Low-Medium |
| Point-in-time | No | No | Yes | No | Partial | Yes |
| Surrogate key change | No | No | Yes | No | No | Yes |
| Best for | Static attrs | Corrections | Compliance | Before/After | Fast-change attrs | Flexible reporting |

---

## SQL Implementation Patterns

### Type 2: Full Implementation with MERGE

```sql
-- Step 1: Expire existing current rows that have changes
UPDATE dim_customer
SET expiration_date = CURRENT_DATE - INTERVAL '1 day',
    is_current = FALSE
WHERE customer_id IN (
    SELECT customer_id FROM staging_customers s
    JOIN dim_customer d ON s.customer_id = d.customer_id
    WHERE d.is_current = TRUE
    AND (s.city != d.city OR s.segment != d.segment)
);

-- Step 2: Insert new version rows
INSERT INTO dim_customer (
    customer_key, customer_id, customer_name, city, segment,
    effective_date, expiration_date, is_current
)
SELECT 
    NEXTVAL('seq_customer_key'),
    s.customer_id,
    s.customer_name,
    s.city,
    s.segment,
    CURRENT_DATE,
    '9999-12-31',
    TRUE
FROM staging_customers s
JOIN dim_customer d ON s.customer_id = d.customer_id
WHERE d.is_current = FALSE
AND d.expiration_date = CURRENT_DATE - INTERVAL '1 day';  -- Just expired
```

### Type 2: Using MERGE (SQL Server / Snowflake)

```sql
MERGE INTO dim_customer AS target
USING staging_customers AS source
ON target.customer_id = source.customer_id AND target.is_current = TRUE
WHEN MATCHED AND (
    target.city != source.city OR target.segment != source.segment
) THEN
    UPDATE SET 
        expiration_date = CURRENT_DATE,
        is_current = FALSE
WHEN NOT MATCHED THEN
    INSERT (customer_key, customer_id, customer_name, city, segment,
            effective_date, expiration_date, is_current)
    VALUES (seq_customer_key.NEXTVAL, source.customer_id, source.customer_name,
            source.city, source.segment, CURRENT_DATE, '9999-12-31', TRUE);

-- Then insert new versions for the just-expired rows (separate statement)
```

### Type 1: Simple Overwrite

```sql
MERGE INTO dim_customer AS target
USING staging_customers AS source
ON target.customer_id = source.customer_id
WHEN MATCHED THEN
    UPDATE SET
        customer_name = source.customer_name,
        email = source.email,
        phone = source.phone
WHEN NOT MATCHED THEN
    INSERT (customer_key, customer_id, customer_name, email, phone)
    VALUES (seq.NEXTVAL, source.customer_id, source.customer_name,
            source.email, source.phone);
```

---

## Querying SCD Type 2 Tables

### Get Current State

```sql
-- Simple: use the is_current flag
SELECT * FROM dim_customer WHERE is_current = TRUE;

-- Alternative: use expiration_date
SELECT * FROM dim_customer WHERE expiration_date = '9999-12-31';
```

### Point-in-Time Query (As-Of)

```sql
-- What was the customer's state on 2023-06-15?
SELECT *
FROM dim_customer
WHERE customer_id = 'C001'
  AND effective_date <= '2023-06-15'
  AND expiration_date > '2023-06-15';
```

### Join Fact to Dimension at Time of Transaction

```sql
-- Each sale joins to the customer version that was active at that time
SELECT 
    f.sale_date,
    d.city AS customer_city_at_time_of_sale,
    f.revenue
FROM fact_sales f
JOIN dim_customer d 
    ON f.customer_key = d.customer_key;  -- Surrogate key handles it!

-- Note: Because the fact stores the surrogate key assigned at insert time,
-- this naturally gives you the correct historical version.
```

> **Critical Insight:** The beauty of SCD Type 2 + surrogate keys is that the fact table's FK *already* points to the correct historical version. You don't need date-range joins on the fact table — that complexity is handled at ETL time when the surrogate key is looked up.

### Find All Versions of an Entity

```sql
SELECT 
    customer_key,
    city,
    segment,
    effective_date,
    expiration_date,
    is_current
FROM dim_customer
WHERE customer_id = 'C001'
ORDER BY effective_date;
```

### Count Changes Per Customer

```sql
SELECT 
    customer_id,
    COUNT(*) - 1 AS num_changes
FROM dim_customer
GROUP BY customer_id
HAVING COUNT(*) > 1
ORDER BY num_changes DESC;
```

---

## Storage and Performance Tradeoffs

| SCD Type | Storage Impact | Write Performance | Read Performance |
|----------|---------------|-------------------|------------------|
| Type 1 | O(n) — fixed | Fast (single UPDATE) | Fast (no filtering) |
| Type 2 | O(n * changes) | Medium (INSERT + UPDATE) | Medium (need date filter) |
| Type 3 | O(n) — fixed | Fast (single UPDATE) | Fast (flat structure) |
| Type 4 | O(n) + O(combos) | Medium (two lookups) | Medium (extra JOIN) |
| Type 6 | O(n * changes) | Slow (UPDATE ALL rows) | Fast (current on all rows) |

### When Type 2 Gets Expensive

For a dimension with 10M entities and an average of 5 changes each:
- Type 1: 10M rows
- Type 2: 50M rows (5x storage)
- Type 6: 50M rows + UPDATE all 50M when current_city changes

> **Critical Insight:** If a "slowly" changing dimension actually changes frequently (daily loyalty scores, weekly price tiers), it's not an SCD problem — use a Type 4 mini-dimension or model it as a fact instead.

---

## Interview Talking Points

> "In our customer dimension, we used a hybrid approach: Type 1 for corrections like name typos, Type 2 for strategically important changes like segment upgrades and city moves, and Type 0 for original_signup_channel which should always reflect the acquisition source regardless of later behavior."

> "The key insight with SCD Type 2 is that the complexity is front-loaded in ETL. At query time, because fact tables store surrogate keys, you automatically get the correct historical version without any date-range join logic. The surrogate key IS the temporal join."

> "We considered Type 6 for our product dimension but rejected it because updating current_category on ALL historical rows every time a product gets reclassified was too expensive for our 200M-row dimension. Instead, we used Type 2 and built a current-state view for reports that need the latest category."

> "When our analytics team needed to group 2023 sales by the customer's CURRENT segment (not historical), I explained that's actually a Type 1 use case in disguise. We created a view that joined fact_sales to the is_current=TRUE row — effectively a Type 1 overlay on Type 2 history."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Using only effective_date without expiration_date | Always store both; use '9999-12-31' sentinel for current |
| Forgetting is_current flag (relying only on dates) | Include is_current BOOLEAN for simple current-state queries |
| Storing timestamps with timezone ambiguity | Use DATE or TIMESTAMP WITH TIME ZONE consistently |
| Not handling same-day multiple changes | Use effective_timestamp (not just date) or apply last-change-wins |
| Using Type 2 for rapidly changing attributes | Use Type 4 mini-dimension or model as fact |
| Expiring old row and inserting new row in wrong order | Expire first, then insert — or use a transaction |
| Joining fact to dimension on natural key | Always join on surrogate key — natural key loses history |
| Overlapping effective/expiration date ranges | Use half-open intervals: [effective, expiration) with no gaps |
| Making ALL attributes Type 2 | Mix types: Type 1 for corrections, Type 2 for tracked changes |
| Not indexing (customer_id, is_current) | Add composite index for common lookup pattern |

---

## Rapid-Fire Q&A

**Q1: Why is it called Type "6" and not Type "5"?**
Type 6 = 1 + 2 + 3 (the sum of the three types it combines). Type 5 exists but is less common — it adds a "current profile" outrigger to a Type 2 dimension.

**Q2: How do you handle deletes in SCD Type 2?**
Soft delete: set is_current=FALSE and expiration_date=today, but don't insert a new row. Or create a final row with a "deleted" status attribute.

**Q3: What's the sentinel value for expiration_date?**
Commonly '9999-12-31' or '2199-12-31'. Some prefer NULL for current rows, but sentinel values simplify BETWEEN queries.

**Q4: Can you combine SCD types within one table?**
Yes, and you should. A customer dim might have Type 0 (signup_source), Type 1 (email), and Type 2 (city, segment) columns in the same table.

**Q5: How does SCD Type 2 interact with aggregate tables?**
Aggregates typically use the current version (is_current=TRUE) or roll up using the version that was active at the grain's time period. Document which approach your aggregate uses.

**Q6: What's the performance impact of SCD Type 2 on large dimensions?**
Main impact is on ETL (change detection + expire/insert logic). Query impact is minimal if you index (natural_key, is_current) and most queries filter on is_current=TRUE.

**Q7: How do you detect changes for Type 2 processing?**
Compare staging data to current dimension rows using hash comparison (MD5/SHA of tracked columns) or column-by-column comparison. Hash is faster for wide tables.

**Q8: What's a "late-arriving dimension" in SCD context?**
When a fact arrives before its dimension. You insert a placeholder dimension row, then back-fill attributes later — potentially creating a new Type 2 version.

**Q9: How does SCD work with streaming/real-time data?**
Micro-batch SCD updates (every 5-15 minutes) or use change data capture (CDC) streams to trigger Type 2 inserts in near-real-time. Full real-time SCD is complex.

**Q10: When would you NOT use any SCD approach?**
When the "dimension" changes every second (sensor readings, stock prices) — model it as a fact table instead, or use a snapshot approach.

---

## ASCII Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────┐
│              SLOWLY CHANGING DIMENSIONS GUIDE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  TYPE 0: RETAIN ORIGINAL                                         │
│  ════════════════════════                                        │
│  [Value A] ──────────────────────────────── (never changes)      │
│                                                                  │
│  TYPE 1: OVERWRITE                                               │
│  ═════════════════                                               │
│  [Value A] ──── X ──── [Value B] ───── (history lost)            │
│                                                                  │
│  TYPE 2: NEW ROW                                                 │
│  ═══════════════                                                 │
│  Row 1: [Value A] ─────────|                                     │
│  Row 2:                    |── [Value B] ─────|                   │
│  Row 3:                                      |── [Value C] ──▶   │
│          eff_date ─────── exp_date                                │
│                                                                  │
│  TYPE 3: PREVIOUS COLUMN                                         │
│  ═══════════════════════                                         │
│  ┌─────────────────────────────────────┐                         │
│  │ current_val = C │ previous_val = B  │  (only 1 back)          │
│  └─────────────────────────────────────┘                         │
│                                                                  │
│  TYPE 4: MINI-DIMENSION                                          │
│  ══════════════════════                                           │
│  dim_customer (stable)──┐                                        │
│                         ├──▶ fact_table                           │
│  dim_customer_demo (fast)┘                                       │
│                                                                  │
│  TYPE 6: HYBRID (1+2+3)                                          │
│  ═══════════════════════                                          │
│  Row 1: hist='A', curr='C', prev='B', eff=..., exp=...          │
│  Row 2: hist='B', curr='C', prev='B', eff=..., exp=...          │
│  Row 3: hist='C', curr='C', prev='B', eff=..., exp=9999         │
│         ▲ Type2    ▲ Type1 overwrite   ▲ Type3                   │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  DECISION FLOWCHART                                              │
│  ══════════════════                                              │
│                                                                  │
│  Need history? ─── NO ──── Is it a correction? ─── YES → Type 1 │
│       │                         │                                │
│      YES                       NO → Type 0 (keep original)       │
│       │                                                          │
│  Changes frequently? ─── YES ──── Type 4 (mini-dim)             │
│       │                                                          │
│      NO                                                          │
│       │                                                          │
│  Need full history? ─── YES ─── Need current on all rows?       │
│       │                              │              │            │
│      NO                            YES            NO             │
│       │                              │              │            │
│    Type 3                         Type 6         Type 2          │
│  (1-level back)                  (hybrid)     (standard)         │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  SCD TYPE 2 QUERY PATTERNS                                       │
│  ═════════════════════════                                        │
│                                                                  │
│  Current state:    WHERE is_current = TRUE                       │
│  Point-in-time:    WHERE eff_date <= @date AND exp_date > @date  │
│  All versions:     WHERE customer_id = 'X' ORDER BY eff_date     │
│  Fact join:        ON fact.customer_key = dim.customer_key        │
│                    (surrogate key handles temporality!)           │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  GOLDEN RULES                                                    │
│  ════════════                                                    │
│                                                                  │
│  1. Mix SCD types WITHIN a single dimension table                │
│  2. Type 2 = surrogate key changes; Type 1 = same key           │
│  3. Fact FK → surrogate key = automatic history join             │
│  4. Use half-open intervals: [effective, expiration)             │
│  5. Index on (natural_key, is_current) for lookups              │
│  6. Hash change detection for wide tables                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 10 of 45*
