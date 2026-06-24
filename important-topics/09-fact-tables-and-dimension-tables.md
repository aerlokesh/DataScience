# 🎯 Topic 9: Fact Tables and Dimension Tables

> *The foundational building blocks of dimensional modeling — understanding grain, measures, and dimensions is the difference between a warehouse that answers questions and one that generates confusion.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [Fact Tables Deep Dive](#fact-tables-deep-dive)
3. [Types of Fact Tables](#types-of-fact-tables)
4. [Measures Classification](#measures-classification)
5. [Dimension Tables Deep Dive](#dimension-tables-deep-dive)
6. [Types of Dimensions](#types-of-dimensions)
7. [Surrogate vs Natural Keys](#surrogate-vs-natural-keys)
8. [Fact Table Grain](#fact-table-grain)
9. [Real-World Examples](#real-world-examples)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Introduction

Dimensional modeling, pioneered by Ralph Kimball, organizes data into two primary structures: **fact tables** (what happened — the measurements) and **dimension tables** (the context — who, what, where, when, why, how). Every analytic query is fundamentally a join between facts and dimensions.

> **Critical Insight:** The single most important design decision in dimensional modeling is declaring the **grain** of the fact table. Every column must be consistent with that grain. Get this wrong, and double-counting or under-counting becomes inevitable.

---

## Fact Tables Deep Dive

Fact tables store the **quantitative measurements** of business processes. They are typically:

- **Narrow and deep** — few columns, many rows (billions+)
- **Numeric-dominant** — most columns are measurable quantities
- **Foreign-key-heavy** — link to dimension tables via surrogate keys
- **Append-mostly** — rarely updated once inserted

### Anatomy of a Fact Table

```sql
CREATE TABLE fact_orders (
    order_key           BIGINT PRIMARY KEY,      -- Surrogate key (optional for fact)
    date_key            INT NOT NULL,            -- FK to dim_date
    customer_key        INT NOT NULL,            -- FK to dim_customer
    product_key         INT NOT NULL,            -- FK to dim_product
    store_key           INT NOT NULL,            -- FK to dim_store
    promotion_key       INT NOT NULL,            -- FK to dim_promotion
    order_id            VARCHAR(20),             -- Degenerate dimension
    quantity            INT,                     -- Additive measure
    unit_price          DECIMAL(10,2),           -- Non-additive measure
    discount_amount     DECIMAL(10,2),           -- Additive measure
    net_revenue         DECIMAL(12,2),           -- Additive measure
    tax_amount          DECIMAL(10,2)            -- Additive measure
);
```

---

## Types of Fact Tables

### 1. Transactional Fact Table

Records one row per business event at the most atomic grain.

| Property | Description |
|----------|-------------|
| **Grain** | One row per transaction/event |
| **Example** | Each line item on a receipt |
| **Sparsity** | Very sparse — only rows for events that occur |
| **Update Pattern** | Insert-only (rarely updated) |
| **Size** | Largest of all fact types |

```sql
-- Transactional fact: one row per purchase line item
SELECT 
    d.calendar_date,
    p.product_name,
    SUM(f.quantity) AS units_sold,
    SUM(f.net_revenue) AS total_revenue
FROM fact_sales_transaction f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
GROUP BY d.calendar_date, p.product_name;
```

### 2. Periodic Snapshot Fact Table

Captures the cumulative state at regular intervals (daily, weekly, monthly).

| Property | Description |
|----------|-------------|
| **Grain** | One row per entity per time period |
| **Example** | Daily account balance, monthly inventory level |
| **Sparsity** | Dense — one row exists even if nothing happened |
| **Update Pattern** | Insert new snapshot rows each period |
| **Size** | Medium — predictable growth |

```sql
-- Periodic snapshot: daily inventory levels
CREATE TABLE fact_inventory_daily (
    date_key            INT NOT NULL,
    product_key         INT NOT NULL,
    warehouse_key       INT NOT NULL,
    quantity_on_hand    INT,
    quantity_on_order   INT,
    days_of_supply      DECIMAL(5,1),
    PRIMARY KEY (date_key, product_key, warehouse_key)
);
```

> **Critical Insight:** Periodic snapshots are **dense** — if a product had zero sales on Tuesday, you still have a row showing quantity_on_hand. This is fundamentally different from transactional facts where absence means nothing happened.

### 3. Accumulating Snapshot Fact Table

Tracks the lifecycle of a process with a definite beginning and end.

| Property | Description |
|----------|-------------|
| **Grain** | One row per process instance (e.g., per order) |
| **Example** | Order fulfillment pipeline, loan application lifecycle |
| **Sparsity** | One row, updated as milestones are reached |
| **Update Pattern** | **Updated in place** — unique among fact types |
| **Size** | Smallest — one row per entity |

```sql
CREATE TABLE fact_order_fulfillment (
    order_key               BIGINT PRIMARY KEY,
    order_date_key          INT,
    payment_date_key        INT,         -- NULL until payment
    ship_date_key           INT,         -- NULL until shipped
    delivery_date_key       INT,         -- NULL until delivered
    return_date_key         INT,         -- NULL unless returned
    order_to_payment_days   INT,         -- Lag metric
    payment_to_ship_days    INT,         -- Lag metric
    ship_to_delivery_days   INT,         -- Lag metric
    order_amount            DECIMAL(12,2)
);
```

### Comparison Table: Fact Table Types

| Characteristic | Transactional | Periodic Snapshot | Accumulating Snapshot |
|---------------|---------------|-------------------|----------------------|
| Grain | One event | One period per entity | One lifecycle |
| Updates | Never | Never (new rows) | Yes (milestones) |
| Density | Sparse | Dense | One row per instance |
| Date Keys | 1 (event date) | 1 (period date) | Multiple (milestone dates) |
| Typical Size | Very large | Large | Moderate |
| Use Case | "What happened?" | "What's the current state?" | "How far along?" |

---

## Measures Classification

### Additive Measures

Can be summed across **all** dimensions without restriction.

| Measure | Why Additive |
|---------|-------------|
| Revenue | Sum across time, product, store = valid |
| Quantity sold | Sum across any dimension = valid |
| Discount amount | Sum across any dimension = valid |
| Cost | Sum across any dimension = valid |

### Semi-Additive Measures

Can be summed across **some** dimensions but NOT across time.

| Measure | Why Semi-Additive |
|---------|------------------|
| Account balance | Sum across accounts = valid; Sum across time = WRONG |
| Inventory quantity | Sum across products = valid; Sum across dates = WRONG |
| Headcount | Sum across departments = valid; Sum across months = WRONG |

```sql
-- WRONG: summing balance across time
SELECT SUM(balance) FROM fact_account_daily;  -- Meaningless!

-- CORRECT: use AVG or pick a specific point-in-time
SELECT AVG(balance) FROM fact_account_daily WHERE month = '2024-03';
-- Or: SELECT balance FROM fact_account_daily WHERE date_key = 20240331;
```

> **Critical Insight:** Semi-additive measures are the #1 source of "the numbers don't add up" complaints. Always use AVG, MIN, MAX, or point-in-time filtering for these measures — never SUM across time.

### Non-Additive Measures

Cannot be meaningfully summed across **any** dimension.

| Measure | Why Non-Additive | Correct Aggregation |
|---------|-----------------|---------------------|
| Unit price | Summing prices is meaningless | AVG, weighted avg |
| Ratio/percentage | 30% + 40% ≠ 70% | Recompute from components |
| Temperature | Sum of temperatures is nonsense | AVG, MIN, MAX |
| Distinct count | Not additive by nature | COUNT(DISTINCT) |

---

## Dimension Tables Deep Dive

Dimension tables provide the **context** for facts. They are typically:

- **Wide and shallow** — many columns, fewer rows
- **Text-heavy** — descriptive attributes for filtering/grouping
- **Slowly changing** — updated when business attributes change
- **Denormalized** — flattened hierarchy for query performance

### Anatomy of a Dimension Table

```sql
CREATE TABLE dim_product (
    product_key         INT PRIMARY KEY,         -- Surrogate key
    product_id          VARCHAR(20),             -- Natural/business key
    product_name        VARCHAR(200),
    brand               VARCHAR(100),
    category            VARCHAR(100),
    subcategory         VARCHAR(100),
    department          VARCHAR(100),
    unit_cost           DECIMAL(10,2),
    unit_list_price     DECIMAL(10,2),
    package_size        VARCHAR(50),
    is_active           BOOLEAN,
    effective_date      DATE,
    expiration_date     DATE
);
```

---

## Types of Dimensions

### 1. Conformed Dimensions

Shared across multiple fact tables with identical meaning and keys.

```
dim_date         → Used by fact_sales, fact_inventory, fact_returns
dim_customer     → Used by fact_sales, fact_support_tickets, fact_web_sessions
dim_product      → Used by fact_sales, fact_inventory, fact_returns
```

> **Critical Insight:** Conformed dimensions are what enable cross-process analysis. Without them, you cannot compare sales performance to inventory levels to customer support metrics. They are the "glue" of the enterprise data warehouse.

### 2. Degenerate Dimensions

Dimension attributes that live directly in the fact table (no separate dimension table).

```sql
-- order_id is a degenerate dimension — it provides grouping
-- but has no additional attributes worth a separate table
SELECT 
    f.order_id,           -- Degenerate dimension
    SUM(f.quantity),
    SUM(f.net_revenue)
FROM fact_order_line f
GROUP BY f.order_id;
```

Common examples: invoice number, order number, transaction ID.

### 3. Junk Dimensions

A single dimension table combining miscellaneous low-cardinality flags and indicators.

```sql
CREATE TABLE dim_transaction_flags (
    flag_key            INT PRIMARY KEY,
    payment_type        VARCHAR(20),     -- 'credit','debit','cash','gift_card'
    is_online           BOOLEAN,         -- TRUE/FALSE
    is_gift_wrapped     BOOLEAN,
    shipping_priority   VARCHAR(10)      -- 'standard','express','overnight'
);
-- Instead of 4 separate dimensions with 2-5 values each,
-- one junk dimension with ~120 distinct combinations
```

### 4. Outrigger Dimensions

A dimension table that hangs off another dimension (not directly off the fact table).

```sql
-- dim_product → dim_brand (outrigger)
CREATE TABLE dim_brand (
    brand_key           INT PRIMARY KEY,
    brand_name          VARCHAR(100),
    parent_company      VARCHAR(100),
    country_of_origin   VARCHAR(50),
    founded_year        INT
);

CREATE TABLE dim_product (
    product_key         INT PRIMARY KEY,
    brand_key           INT REFERENCES dim_brand(brand_key),  -- Outrigger link
    product_name        VARCHAR(200),
    ...
);
```

### 5. Role-Playing Dimensions

Same physical dimension table used multiple times in a fact table under different roles.

```sql
-- dim_date plays 3 roles in the accumulating snapshot
SELECT 
    od.calendar_date AS order_date,
    sd.calendar_date AS ship_date,
    dd.calendar_date AS delivery_date,
    f.ship_to_delivery_days
FROM fact_order_fulfillment f
JOIN dim_date od ON f.order_date_key = od.date_key
JOIN dim_date sd ON f.ship_date_key = sd.date_key
JOIN dim_date dd ON f.delivery_date_key = dd.date_key;
```

### Dimension Types Comparison

| Type | Definition | Use Case |
|------|-----------|----------|
| Conformed | Shared across fact tables | Enterprise consistency |
| Degenerate | Lives in fact table (no dim table) | Transaction identifiers |
| Junk | Combines miscellaneous flags | Reduce fact table width |
| Outrigger | Dim hanging off another dim | Shared hierarchical context |
| Role-Playing | Same dim used multiple times | Multiple dates/locations |
| Shrunk | Subset of a larger dimension | Aggregated fact tables |

---

## Surrogate vs Natural Keys

| Aspect | Surrogate Key | Natural Key |
|--------|---------------|-------------|
| Definition | System-generated integer | Business-meaningful identifier |
| Example | 1, 2, 3, ... (auto-increment) | "SKU-A1234", "CUST-98765" |
| Size | 4 bytes (INT) or 8 bytes (BIGINT) | Variable (often 10-50 bytes) |
| Join Performance | Faster (integer comparison) | Slower (string comparison) |
| History Tracking | New key per version (SCD Type 2) | Same key, can't track versions |
| Source Independence | Decoupled from source systems | Tied to source system |
| Null Handling | Can assign key for "unknown" | NULL propagates |
| Change Management | Stable even if business key changes | Breaks if source key changes |

```sql
-- Surrogate key approach enables SCD Type 2
-- Customer "CUST-001" has two rows with different surrogate keys:
-- customer_key=42: John Smith, NY (2020-01-01 to 2023-06-15)
-- customer_key=87: John Smith, CA (2023-06-15 to 9999-12-31)
```

> **Critical Insight:** Always use surrogate keys in dimension tables. Natural keys create fragile joins, prevent history tracking, and couple your warehouse to source system decisions. The natural key should still exist as an attribute for traceability.

---

## Fact Table Grain

The grain declares exactly what one row in the fact table represents. It must be stated explicitly before any design work begins.

### Grain Declaration Examples

| Business Process | Grain Statement |
|-----------------|-----------------|
| Retail Sales | One row per product per transaction |
| Web Clicks | One row per page view per session |
| Inventory | One row per product per warehouse per day |
| Subscriptions | One row per subscription lifecycle |
| Call Center | One row per call |

### The Grain Test

Every column in the fact table must be TRUE at the declared grain:

```
Grain: "One row per order line item"

✅ quantity         → Yes, each line item has a quantity
✅ unit_price       → Yes, each line item has a price
✅ product_key      → Yes, each line item is one product
❌ total_order_amt  → NO! This is at the ORDER grain, not line item
❌ num_items        → NO! This is at the ORDER grain
```

---

## Real-World Examples

### Example 1: E-Commerce Platform

```sql
-- Fact: One row per order line item
CREATE TABLE fact_ecommerce_sales (
    date_key            INT,
    customer_key        INT,
    product_key         INT,
    seller_key          INT,
    promotion_key       INT,
    order_id            VARCHAR(20),    -- Degenerate dimension
    line_number         INT,            -- Part of grain
    quantity            INT,            -- Additive
    unit_price          DECIMAL(10,2),  -- Non-additive
    discount_pct        DECIMAL(5,2),   -- Non-additive
    gross_revenue       DECIMAL(12,2),  -- Additive
    discount_amount     DECIMAL(10,2),  -- Additive
    net_revenue         DECIMAL(12,2),  -- Additive
    shipping_cost       DECIMAL(8,2),   -- Additive
    tax_amount          DECIMAL(10,2)   -- Additive
);
```

### Example 2: Ride-Sharing Platform

```sql
-- Transactional fact: one row per completed ride
CREATE TABLE fact_rides (
    ride_date_key       INT,
    request_time_key    INT,
    pickup_location_key INT,
    dropoff_location_key INT,
    driver_key          INT,
    rider_key           INT,
    vehicle_type_key    INT,
    ride_id             VARCHAR(30),    -- Degenerate dimension
    distance_miles      DECIMAL(6,2),   -- Additive
    duration_minutes    DECIMAL(6,1),   -- Additive
    base_fare           DECIMAL(8,2),   -- Additive
    surge_multiplier    DECIMAL(3,1),   -- Non-additive
    total_fare          DECIMAL(8,2),   -- Additive
    driver_payout       DECIMAL(8,2),   -- Additive
    platform_fee        DECIMAL(8,2),   -- Additive
    rider_rating        DECIMAL(2,1),   -- Non-additive
    driver_rating       DECIMAL(2,1)    -- Non-additive
);

-- Accumulating snapshot: ride lifecycle
CREATE TABLE fact_ride_lifecycle (
    ride_key            BIGINT PRIMARY KEY,
    request_date_key    INT,
    match_date_key      INT,        -- NULL until matched
    pickup_date_key     INT,        -- NULL until picked up
    dropoff_date_key    INT,        -- NULL until completed
    cancel_date_key     INT,        -- NULL unless cancelled
    request_to_match_sec    INT,
    match_to_pickup_sec     INT,
    pickup_to_dropoff_sec   INT,
    final_status        VARCHAR(20) -- 'completed','cancelled','no_driver'
);
```

### Example 3: SaaS Subscription (Periodic Snapshot)

```sql
-- Monthly snapshot of subscription state
CREATE TABLE fact_subscription_monthly (
    month_key           INT,
    customer_key        INT,
    plan_key            INT,
    mrr                 DECIMAL(10,2),  -- Semi-additive (across customers, not time)
    seats_used          INT,            -- Semi-additive
    seats_purchased     INT,            -- Semi-additive
    api_calls_mtd       BIGINT,         -- Additive within month
    storage_gb          DECIMAL(8,2),   -- Semi-additive
    is_churned          BOOLEAN,
    months_active       INT
);
```

---

## Interview Talking Points

> "When I design a fact table, the first thing I establish is the grain — what exactly does one row represent? For our e-commerce platform, we declared the grain as 'one line item per order.' This made it crystal clear that order-level metrics like total_order_amount do NOT belong in the fact table — they'd cause double-counting when aggregated across products."

> "I use surrogate keys in every dimension table. We had a situation where a vendor changed their product ID format from numeric to alphanumeric. Because we used surrogate keys, our fact table was completely unaffected — we just updated the natural_key column in dim_product. If we'd used natural keys as our join keys, every historical fact row would have been orphaned."

> "For our inventory reporting, I chose a periodic snapshot fact table because stakeholders needed to see inventory levels even on days with zero movement. A transactional approach would have required complex running-total calculations. The snapshot made 'what was inventory on March 15th?' a simple filter query."

> "We encountered the classic semi-additive trap when a junior analyst summed account balances across all dates and reported a wildly inflated total AUM figure. I introduced a data quality check that flags any SUM() on semi-additive measures without a date filter, and documented the correct aggregation pattern in our semantic layer."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Declaring grain too loosely ("one row per order") when line items exist | Be atomic: "one row per order line item" |
| Putting non-additive measures and summing them | Use weighted averages or recompute from components |
| Using natural keys as foreign keys in fact tables | Use surrogate integer keys; keep natural key as attribute |
| Storing derived metrics in fact tables (e.g., profit = revenue - cost) | Store atomic measures; compute derived metrics in views |
| Creating one giant fact table for everything | Separate fact tables per business process |
| Normalizing dimension tables for storage savings | Denormalize for query performance; storage is cheap |
| Mixing grains in one fact table | One grain per fact table; use bridge tables if needed |
| Forgetting to handle NULLs in dimension foreign keys | Create "Unknown" or "Not Applicable" dimension rows |
| Updating transactional facts | Transactional facts are immutable; use correction rows |
| Ignoring semi-additive measures in documentation | Clearly label each measure's additivity in metadata |

---

## Rapid-Fire Q&A

**Q1: What is a factless fact table?**
A fact table with no measures — it records the occurrence of events. Example: student attendance (student_key, date_key, class_key) — the mere presence of a row IS the fact.

**Q2: Can a fact table have no dimensions?**
No. At minimum, every fact has a date/time dimension. Without dimensions, you cannot filter, group, or give context to the measures.

**Q3: What's the difference between a periodic snapshot and a daily aggregate?**
A periodic snapshot is dense (rows exist even with no activity). A daily aggregate only has rows where events occurred. The snapshot answers "what was the state?" while the aggregate answers "what happened?"

**Q4: When would you use an accumulating snapshot over a transactional fact?**
When you need to track a well-defined process with multiple milestones (order lifecycle, loan processing). It has multiple date foreign keys and lag calculations between milestones.

**Q5: How do you handle many-to-many relationships in dimensional models?**
Use a bridge table (also called a factless fact table or association table) between the fact and the multi-valued dimension.

**Q6: What makes a dimension "conformed"?**
It shares the same keys, attributes, and meaning across multiple fact tables and subject areas. dim_date and dim_customer are classic conformed dimensions.

**Q7: Why not just use the source system's primary key as the dimension key?**
Source keys can change, be reused, collide across systems, or be non-integer. Surrogate keys provide stability, performance, and SCD support.

**Q8: How do you model a fact with variable number of attributes?**
Either use a junk dimension for low-cardinality flags, or use a key-value pair bridge table for truly variable attributes.

**Q9: What's the difference between a dimension attribute and a fact measure?**
If you can SUM/AVG it meaningfully, it's likely a measure (fact). If you GROUP BY or filter on it, it's likely a dimension attribute.

**Q10: How does the grain affect query performance?**
Finer grain = more rows = more storage and longer scans. But coarser grain loses detail. Optimize with aggregate tables or materialized views on top of atomic-grain fact tables.

---

## ASCII Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────┐
│                DIMENSIONAL MODEL ANATOMY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────────┐     ┌──────────────────────┐     ┌────────┐│
│   │  dim_date    │     │   FACT TABLE          │     │dim_prod││
│   │─────────────-│     │──────────────────────-│     │────────││
│   │ date_key (PK)│────▶│ date_key (FK)         │◀────│prod_key││
│   │ calendar_date│     │ product_key (FK)      │     │name    ││
│   │ month        │     │ customer_key (FK)     │     │brand   ││
│   │ quarter      │     │ store_key (FK)        │     │category││
│   │ year         │     │ order_id (DD)         │     └────────┘│
│   │ day_of_week  │     │ quantity (additive)   │               │
│   └──────────────┘     │ revenue  (additive)   │     ┌────────┐│
│                        │ unit_price(non-add)   │     │dim_cust││
│   ┌──────────────┐     │ balance  (semi-add)   │     │────────││
│   │  dim_store   │     └──────────────────────-┘     │cust_key││
│   │──────────────│              ▲                    │name    ││
│   │ store_key(PK)│──────────────┘                    │city    ││
│   │ store_name   │                                   │segment ││
│   │ city         │                                   └────────┘│
│   │ state        │                                             │
│   └──────────────┘                                             │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  FACT TABLE TYPES                                               │
│  ═══════════════                                                │
│                                                                 │
│  Transactional:  ● ● ● ● ●  (sparse, one per event)           │
│  Periodic Snap:  ■ ■ ■ ■ ■  (dense, one per period)           │
│  Accumulating:   ◆──→──→──→  (one row, updated at milestones)  │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  MEASURE ADDITIVITY                                             │
│  ══════════════════                                             │
│                                                                 │
│  Additive:       SUM across ALL dims     (revenue, qty)        │
│  Semi-Additive:  SUM across SOME dims    (balance, inventory)  │
│  Non-Additive:   Cannot SUM meaningfully (price, ratio, %)     │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  DIMENSION TYPES                                                │
│  ═══════════════                                                │
│                                                                 │
│  Conformed:      Shared across fact tables (dim_date)           │
│  Degenerate:     Lives IN fact table (order_id, invoice_no)     │
│  Junk:           Combines flags (is_online, payment_type)       │
│  Outrigger:      Dim linked to another dim (dim_brand→dim_prod)│
│  Role-Playing:   Same dim, multiple FKs (order_date, ship_date)│
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  KEY PRINCIPLES                                                 │
│  ══════════════                                                 │
│                                                                 │
│  1. Declare grain FIRST — every column must match               │
│  2. Surrogate keys ALWAYS in dimensions                         │
│  3. One business process = one fact table                       │
│  4. Denormalize dimensions for performance                      │
│  5. Label every measure's additivity                            │
│  6. Handle NULLs with "Unknown" dimension rows                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 9 of 45*
