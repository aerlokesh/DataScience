# 🎯 Topic 8: Star Schema vs Snowflake Schema

> **Data Science Interview — Deep Dive**
> The foundation of dimensional modeling. How to design data warehouses that serve analytics at scale — with the tradeoffs that determine query performance, storage efficiency, and analyst productivity.

---

## Table of Contents

1. [What is Dimensional Modeling?](#what-is-dimensional-modeling)
2. [Kimball vs Inmon Approaches](#kimball-vs-inmon-approaches)
3. [Star Schema — Deep Dive](#star-schema--deep-dive)
4. [Snowflake Schema — Deep Dive](#snowflake-schema--deep-dive)
5. [Galaxy Schema (Fact Constellation)](#galaxy-schema-fact-constellation)
6. [Fact Table Types](#fact-table-types)
7. [Grain Definition](#grain-definition)
8. [Advanced Dimension Concepts](#advanced-dimension-concepts)
9. [Comparison Table: Star vs Snowflake](#comparison-table-star-vs-snowflake)
10. [Real-World Design Examples](#real-world-design-examples)
11. [How Modern Tools Change the Calculus](#how-modern-tools-change-the-calculus)
12. [Interview Design Question Walkthrough](#interview-design-question-walkthrough)
13. [Interview Talking Points](#interview-talking-points)
14. [Common Interview Mistakes](#common-interview-mistakes)
15. [Rapid-Fire Q&A](#rapid-fire-qa)
16. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## What is Dimensional Modeling?

Dimensional modeling is a technique for structuring data in a data warehouse so that it is **intuitive to business users** and **optimized for analytical queries**. It separates data into:

- **Facts** — Measurable, quantitative data (revenue, quantity, duration)
- **Dimensions** — Descriptive context (who, what, where, when, why, how)

### Core Principles

| Principle | Description |
|-----------|-------------|
| **Business-driven** | Modeled around business processes, not source systems |
| **Understandability** | Non-technical users can navigate and query the schema |
| **Performance** | Optimized for aggregation, filtering, and GROUP BY operations |
| **Flexibility** | New questions can be answered without schema redesign |

### Why It Matters for Data Scientists

- You query dimensional models daily (Redshift, BigQuery, Snowflake)
- Interview questions test whether you can *design* them, not just consume them
- Poor schema design leads to slow dashboards, inconsistent metrics, and analyst frustration
- Understanding the model helps you debug data quality issues upstream

---

## Kimball vs Inmon Approaches

### Ralph Kimball — Bottom-Up (Dimensional)

```
Source Systems → ETL → Dimensional Data Marts → Enterprise DW (conformed dims)
```

- Build one **data mart** per business process (sales, inventory, marketing)
- Use **conformed dimensions** to stitch marts together
- Star/snowflake schemas at the center of each mart
- Faster time-to-value; incremental delivery
- The dominant approach in modern cloud warehousing

### Bill Inmon — Top-Down (Normalized)

```
Source Systems → ETL → Enterprise DW (3NF) → Dimensional Data Marts
```

- Build a **single normalized enterprise data warehouse** first (3NF)
- Derive subject-area data marts from the central DW
- Ensures single source of truth; avoids redundancy
- Higher upfront cost; longer time-to-first-value
- Better for organizations with strong data governance mandates

### When Each Approach Wins

| Scenario | Best Approach |
|----------|---------------|
| Startup with 3 data sources | Kimball — fast iteration |
| Fortune 500 with 200+ source systems | Inmon — centralized governance |
| Team of 2 analytics engineers | Kimball — pragmatic delivery |
| Regulatory-heavy industry (banking) | Inmon — auditability and lineage |
| Modern cloud stack (dbt + BigQuery) | Kimball-influenced with medallion layers |

---

## Star Schema — Deep Dive

### Structure

```
                    ┌──────────────┐
                    │  dim_date    │
                    └──────┬───────┘
                           │
┌──────────────┐    ┌──────┴───────┐    ┌──────────────┐
│ dim_product  │────│  fact_sales  │────│ dim_customer │
└──────────────┘    └──────┬───────┘    └──────────────┘
                           │
                    ┌──────┴───────┐
                    │  dim_store   │
                    └──────────────┘
```

- **One fact table** at the center containing foreign keys and measures
- **Denormalized dimension tables** radiating outward (one level of joins)
- Dimension tables contain all hierarchical levels in a single wide table

### Example: Retail Sales Star Schema

```sql
-- Fact Table
CREATE TABLE fact_sales (
    sale_id         BIGINT PRIMARY KEY,
    date_key        INT REFERENCES dim_date(date_key),
    product_key     INT REFERENCES dim_product(product_key),
    customer_key    INT REFERENCES dim_customer(customer_key),
    store_key       INT REFERENCES dim_store(store_key),
    -- Measures (additive facts)
    quantity_sold   INT,
    unit_price      DECIMAL(10,2),
    discount_amount DECIMAL(10,2),
    total_revenue   DECIMAL(12,2),
    cost_of_goods   DECIMAL(12,2)
);

-- Denormalized Dimension (all hierarchy levels in one table)
CREATE TABLE dim_product (
    product_key     INT PRIMARY KEY,
    product_id      VARCHAR(20),        -- Natural key
    product_name    VARCHAR(200),
    brand           VARCHAR(100),
    category        VARCHAR(100),       -- Hierarchy level 2
    department      VARCHAR(100),       -- Hierarchy level 3
    is_organic      BOOLEAN,
    launch_date     DATE
);
```

### Advantages of Star Schema

| Advantage | Why It Matters |
|-----------|---------------|
| **Simple queries** | Single JOIN per dimension; analysts write fewer mistakes |
| **Fast aggregations** | Fewer joins = faster query execution |
| **BI tool friendly** | Tableau, Looker, Power BI auto-detect star schemas |
| **Intuitive** | Business users can browse dimension tables to find filters |
| **Columnar storage benefits** | Wide dimension tables compress well in Redshift/BigQuery |

### Disadvantages of Star Schema

| Disadvantage | Impact |
|--------------|--------|
| **Data redundancy** | "Electronics" repeated millions of times in dim_product |
| **Larger dimension tables** | More storage for repeated hierarchical attributes |
| **Update anomalies** | Renaming a category requires updating many rows |
| **Not normalized** | Violates 3NF purists' sensibilities |

---

## Snowflake Schema — Deep Dive

### Structure

```
┌──────────────┐    ┌──────────────┐
│ dim_category │────│ dim_product  │
└──────────────┘    └──────┬───────┘
                           │
                    ┌──────┴───────┐    ┌──────────────┐
                    │  fact_sales  │────│ dim_customer │
                    └──────┬───────┘    └──────┬───────┘
                           │                    │
                    ┌──────┴───────┐    ┌──────┴───────┐
                    │  dim_date    │    │  dim_city    │
                    └──────────────┘    └──────┬───────┘
                                               │
                                        ┌──────┴───────┐
                                        │ dim_country  │
                                        └──────────────┘
```

- Dimension tables are **normalized** into sub-dimensions
- Multiple levels of joins required to reach leaf attributes
- Resembles a snowflake pattern when drawn as an ERD

### Example: Normalized Product Dimension

```sql
-- Normalized into separate tables
CREATE TABLE dim_department (
    department_key  INT PRIMARY KEY,
    department_name VARCHAR(100)
);

CREATE TABLE dim_category (
    category_key    INT PRIMARY KEY,
    category_name   VARCHAR(100),
    department_key  INT REFERENCES dim_department(department_key)
);

CREATE TABLE dim_product (
    product_key     INT PRIMARY KEY,
    product_id      VARCHAR(20),
    product_name    VARCHAR(200),
    brand           VARCHAR(100),
    category_key    INT REFERENCES dim_category(category_key),
    is_organic      BOOLEAN,
    launch_date     DATE
);
```

### When to Use Snowflake Schema

- Dimension tables are **very large** (millions of rows) and redundancy is costly
- You need **referential integrity** enforced at the database level
- The data warehouse also serves **operational reporting** requiring normalized views
- Storage costs dominate (less relevant with cheap cloud storage)
- Dimension hierarchies change frequently (update one row vs. millions)

### Disadvantages of Snowflake Schema

| Disadvantage | Impact |
|--------------|--------|
| **Complex queries** | Multiple JOINs for a single dimension path |
| **Slower performance** | More joins = more shuffles in distributed systems |
| **BI tool confusion** | Auto-modeling may not detect deep hierarchies |
| **Harder to maintain** | More tables to manage, document, and test |

---

## Galaxy Schema (Fact Constellation)

A galaxy schema contains **multiple fact tables** sharing conformed dimension tables.

```
┌──────────────┐         ┌──────────────┐
│ fact_orders  │─────┐   │ fact_returns │
└──────────────┘     │   └──────┬───────┘
                     │          │
              ┌──────┴──────────┴───────┐
              │      dim_product        │
              └─────────────────────────┘
              │      dim_customer       │
              └─────────────────────────┘
              │      dim_date           │
              └─────────────────────────┘
```

### When to Use

- Multiple business processes share the same dimensions
- You need to correlate facts across processes (orders vs. returns)
- Enterprise-level warehouse with dozens of subject areas

### Design Principles

1. **Conformed dimensions** — same dim_customer used by all fact tables
2. **Conformed facts** — "revenue" means the same thing across all fact tables
3. **Bus architecture** — a matrix mapping business processes to dimensions

---

## Fact Table Types

### 1. Transactional Fact Table

Records **one row per event** at the most atomic grain.

```sql
-- One row per order line item
fact_order_lines (
    order_line_id, date_key, product_key, customer_key,
    quantity, unit_price, discount, line_total
)
```

- **Grain**: One event occurrence
- **Sparsity**: Very sparse (most dim combinations never occur)
- **Additivity**: All measures are fully additive
- **Growth**: Continuously appended; can grow very large

### 2. Periodic Snapshot Fact Table

Records the **state at regular intervals** (daily, weekly, monthly).

```sql
-- One row per account per month
fact_account_monthly_snapshot (
    month_key, account_key,
    balance, num_transactions, avg_transaction_size
)
```

- **Grain**: One time period per entity
- **Sparsity**: Dense (every entity has a row every period)
- **Additivity**: Balance is semi-additive (cannot sum across time)
- **Use case**: Portfolio tracking, inventory levels, account balances

### 3. Accumulating Snapshot Fact Table

Tracks the **lifecycle of a process** with multiple date milestones.

```sql
-- One row per order, updated as order progresses
fact_order_fulfillment (
    order_key, customer_key, product_key,
    order_date_key, ship_date_key, delivery_date_key, return_date_key,
    days_to_ship, days_to_deliver, order_total
)
```

- **Grain**: One row per process instance (updated over time)
- **Multiple date keys**: Each milestone gets its own FK to dim_date
- **Use case**: Order fulfillment, loan processing, claim handling
- **Key distinction**: Rows are *updated* (unlike transactional which only appends)

### Fact Type Selection Guide

| Question | If Yes → |
|----------|----------|
| Does an event happen once and never change? | Transactional |
| Do you need "state at a point in time" regularly? | Periodic Snapshot |
| Does the entity progress through defined stages? | Accumulating Snapshot |
| Do you need all three? | Yes — they answer different questions |

---

## Grain Definition

**Grain** = what a single row in your fact table represents.

### Why Grain is Critical

- It determines **what questions you can answer**
- Getting it wrong means either losing detail or exploding row counts
- It is the **first decision** in dimensional modeling (before choosing facts or dimensions)

### Declaring the Grain — Examples

| Business Process | Grain Statement |
|-----------------|-----------------|
| Retail sales | One row per product per transaction |
| Web clickstream | One row per page view per session |
| Subscription billing | One row per subscription per billing period |
| Call center | One row per customer interaction |
| Inventory | One row per product per warehouse per day |

### Common Grain Mistakes

| Mistake | Consequence |
|---------|-------------|
| Mixing grains in one fact table | Incorrect aggregations (double counting) |
| Too coarse (daily summary) | Cannot drill to hourly patterns |
| Too fine (millisecond events) | Unmanageable table size, slow queries |
| Not documenting the grain | Team members make conflicting assumptions |

### The "One Row Per..." Test

Always express your grain as: **"One row per [entity] per [event/period]"**

If you cannot state this clearly, your model is not yet well-defined.

---

## Advanced Dimension Concepts

### Degenerate Dimensions

A dimension with **no attributes** — just a key stored directly in the fact table.

```sql
-- order_number is a degenerate dimension (no dim_order table needed)
fact_order_lines (
    order_line_id, date_key, product_key, customer_key,
    order_number,  -- Degenerate dimension (grouping key)
    quantity, line_total
)
```

- **Use case**: Transaction numbers, invoice numbers, PO numbers
- Enables grouping fact rows that belong to the same transaction
- No separate dimension table because there are no additional descriptive attributes

### Junk Dimensions

A **catch-all dimension** combining low-cardinality flags and indicators.

```sql
-- Instead of 8 boolean columns in the fact table
CREATE TABLE dim_transaction_flags (
    flag_key        INT PRIMARY KEY,
    is_online       BOOLEAN,
    is_gift_wrapped BOOLEAN,
    is_expedited    BOOLEAN,
    payment_type    VARCHAR(20),  -- 'credit', 'debit', 'cash'
    is_loyalty      BOOLEAN
);
-- Pre-populated with all observed combinations
```

- **Why**: Avoids cluttering the fact table with many low-cardinality columns
- **How**: Create all possible combinations as rows (Cartesian product)
- **Size**: Typically small (dozens to hundreds of rows)

### Role-Playing Dimensions

The **same dimension table used multiple times** with different semantic meanings.

```sql
-- dim_date used 3 times with different roles
fact_order_fulfillment (
    order_key,
    order_date_key    INT REFERENCES dim_date(date_key),  -- Role 1
    ship_date_key     INT REFERENCES dim_date(date_key),  -- Role 2
    delivery_date_key INT REFERENCES dim_date(date_key),  -- Role 3
    ...
);
```

- Implemented as **views** or **aliases** in BI tools:
  - `dim_order_date` (view on dim_date)
  - `dim_ship_date` (view on dim_date)
  - `dim_delivery_date` (view on dim_date)

### Slowly Changing Dimensions (SCDs)

| Type | Strategy | Example |
|------|----------|---------|
| **Type 0** | Never change | Original credit score at account opening |
| **Type 1** | Overwrite | Fix a typo in customer name |
| **Type 2** | Add new row (versioned) | Track address history with effective dates |
| **Type 3** | Add new column | Store current + previous value only |
| **Type 6** | Hybrid (1+2+3) | Current flag + history + previous column |

**Type 2 is most common in interviews** — know it well:

```sql
CREATE TABLE dim_customer (
    customer_key    INT PRIMARY KEY,      -- Surrogate key
    customer_id     VARCHAR(20),          -- Natural key (not unique!)
    name            VARCHAR(200),
    city            VARCHAR(100),
    state           VARCHAR(50),
    effective_date  DATE,
    expiration_date DATE,
    is_current      BOOLEAN
);
```

---

## Comparison Table: Star vs Snowflake

| Criterion | Star Schema | Snowflake Schema |
|-----------|-------------|------------------|
| **Dimension structure** | Denormalized (flat) | Normalized (hierarchical tables) |
| **Number of JOINs** | 1 per dimension | Multiple per dimension path |
| **Query performance** | Faster (fewer joins) | Slower (more joins, more shuffles) |
| **Query complexity** | Simple SQL | Complex multi-join SQL |
| **Storage efficiency** | Higher redundancy | Lower redundancy |
| **ETL complexity** | Simpler (fewer target tables) | More complex (more tables to load) |
| **BI tool compatibility** | Excellent (auto-detected) | Moderate (requires manual config) |
| **Data integrity** | Weaker (denormalized) | Stronger (referential integrity) |
| **Maintenance** | Update many rows for hierarchy changes | Update one row in normalized table |
| **Best for** | Analytics, BI, ad-hoc queries | Regulated environments, large dims |
| **Modern relevance** | Dominant in cloud DWs | Declining (compute is cheap) |
| **Adoption** | 80%+ of analytical warehouses | Niche use cases |

### Performance Reality Check

In modern columnar engines (BigQuery, Snowflake, Redshift):
- **Star schema** still wins because columnar compression handles redundancy well
- The join cost in snowflake schema is real — distributed shuffles are expensive
- Storage savings from normalization are negligible at modern storage prices
- The **cognitive overhead** of snowflake schema hurts analyst productivity

---

## Real-World Design Examples

### Example 1: Design a Data Model for Uber Trips

**Step 1 — Identify the business process**: Ride completion

**Step 2 — Declare the grain**: One row per completed trip

**Step 3 — Choose dimensions**:

```
dim_date          — trip start date (role-playing: start, end, request)
dim_time          — time of day (peak hours analysis)
dim_rider          — rider demographics, account type
dim_driver         — driver info, rating tier, vehicle type
dim_location       — pickup/dropoff (role-playing: origin, destination)
dim_trip_type      — UberX, UberXL, Uber Black, Pool
dim_payment        — payment method, promo applied
```

**Step 4 — Choose facts (measures)**:

```sql
CREATE TABLE fact_trips (
    trip_key            BIGINT,
    -- Dimension FKs
    request_date_key    INT,
    start_date_key      INT,
    end_date_key        INT,
    rider_key           INT,
    driver_key          INT,
    origin_key          INT,
    destination_key     INT,
    trip_type_key       INT,
    payment_key         INT,
    -- Degenerate dimensions
    trip_id             VARCHAR(36),
    -- Measures
    trip_distance_miles DECIMAL(8,2),
    trip_duration_min   DECIMAL(8,2),
    wait_time_min       DECIMAL(8,2),
    surge_multiplier    DECIMAL(4,2),
    fare_amount         DECIMAL(10,2),
    tip_amount          DECIMAL(10,2),
    total_charged       DECIMAL(10,2),
    driver_payout       DECIMAL(10,2)
);
```

**Step 5 — Identify special dimensions**:
- `dim_location` is role-playing (origin vs. destination)
- `dim_date` is role-playing (request, start, end)
- `trip_id` is a degenerate dimension
- `surge_multiplier` could be a factless fact or a degenerate dimension

---

### Example 2: Model Amazon Order Analytics

**Grain**: One row per order line item

```sql
CREATE TABLE fact_order_lines (
    order_line_key      BIGINT,
    -- Dimension FKs
    order_date_key      INT,
    ship_date_key       INT,
    delivery_date_key   INT,
    customer_key        INT,
    product_key         INT,
    seller_key          INT,
    fulfillment_key     INT,    -- FBA vs FBM vs Prime
    shipping_method_key INT,
    promotion_key       INT,
    -- Degenerate dimensions
    order_id            VARCHAR(20),
    -- Measures
    quantity            INT,
    unit_price          DECIMAL(10,2),
    discount_amount     DECIMAL(10,2),
    shipping_cost       DECIMAL(10,2),
    tax_amount          DECIMAL(10,2),
    line_total          DECIMAL(12,2),
    estimated_margin    DECIMAL(10,2)
);
```

**Complementary fact tables (Galaxy schema)**:

| Fact Table | Grain | Purpose |
|-----------|-------|---------|
| `fact_order_lines` | Per line item | Revenue, quantity analysis |
| `fact_returns` | Per returned item | Return rate, reason analysis |
| `fact_reviews` | Per review submitted | Sentiment, rating trends |
| `fact_inventory_daily` | Per product per warehouse per day | Stock levels, turnover |
| `fact_page_views` | Per product page view | Conversion funnel |

---

## How Modern Tools Change the Calculus

### BigQuery / Snowflake / Redshift — The New Reality

| Factor | Traditional RDBMS | Modern Cloud DW |
|--------|-------------------|-----------------|
| Storage cost | Expensive | Cheap ($20-40/TB/month) |
| Compute model | Shared server | Elastic, pay-per-query |
| Join strategy | Nested loops/hash | Distributed shuffle |
| Columnar compression | Limited | Excellent (handles redundancy) |
| Denormalization penalty | High storage cost | Negligible |
| Normalization benefit | Saves $$$ | Minimal savings |

### Key Implications

1. **Star schema wins even harder** — storage is cheap, joins are expensive
2. **Nested/repeated fields** (BigQuery STRUCT/ARRAY) let you embed sub-dimensions
3. **Materialized views** eliminate the need to pre-aggregate
4. **Partitioning + clustering** matter more than schema shape for performance
5. **dbt + version control** makes star schema maintenance manageable
6. **Semi-structured data** (JSON columns) reduces the need for junk dimensions

### The Modern Stack Pattern

```
Raw Layer (Bronze)    → Source-conformed, append-only (Inmon-inspired)
Staging Layer (Silver) → Cleaned, typed, deduped
Marts Layer (Gold)    → Star schemas per business domain (Kimball-inspired)
```

This **medallion architecture** combines the best of both Kimball and Inmon.

---

## Interview Design Question Walkthrough

### The Structured Approach (Use This Framework)

When asked "Design a data model for X":

```
Step 1: Clarify the business process
        → "What are we optimizing for? What decisions does this support?"

Step 2: Declare the grain
        → "One row per [entity] per [event/period]"

Step 3: Identify dimensions (the 5 W's + H)
        → Who, What, Where, When, Why, How

Step 4: Identify measures (facts)
        → Additive, semi-additive, non-additive

Step 5: Address special cases
        → SCDs, role-playing, degenerate, junk dimensions

Step 6: Consider complementary fact tables
        → What other processes share these dimensions?

Step 7: Discuss physical implementation
        → Partitioning strategy, clustering keys, refresh cadence
```

### Worked Example: "Design a data model for a food delivery app"

**Interviewer**: "Design a data model to analyze delivery performance."

**You**: "Let me clarify — are we focused on delivery time optimization, or broader order analytics?"

**Interviewer**: "Delivery time."

**Your answer**:

1. **Grain**: One row per delivery attempt (not per order — an order might have re-deliveries)

2. **Dimensions**:
   - `dim_date` (role-playing: order placed, picked up, delivered)
   - `dim_restaurant` (cuisine, chain vs. independent, prep time tier)
   - `dim_customer` (location zone, membership tier)
   - `dim_driver` (vehicle type, experience tier, rating band)
   - `dim_weather` (temperature bucket, precipitation, conditions)
   - `dim_time_of_day` (hour, peak/off-peak, day of week)

3. **Measures**:
   - `prep_time_minutes` (restaurant → pickup)
   - `transit_time_minutes` (pickup → delivery)
   - `total_time_minutes` (order → delivery)
   - `estimated_time_minutes` (for accuracy analysis)
   - `distance_km`
   - `delivery_fee`

4. **Special considerations**:
   - Weather is a Type 1 SCD (weather at time of order, never changes)
   - If driver re-assigns happen, use accumulating snapshot
   - Partition by `order_date_key` for time-range queries

---

## Interview Talking Points

> "Star schema is my default for analytical workloads. The single-join-per-dimension pattern means analysts can self-serve without writing complex multi-level JOINs, and BI tools like Tableau auto-detect the relationships."

> "I only reach for snowflake schema when a dimension is extremely large — think millions of SKUs with deep category hierarchies — and when storage costs or update frequency make denormalization impractical."

> "In modern cloud warehouses, the calculus has shifted heavily toward star schema. Columnar compression handles the redundancy efficiently, and the distributed join cost of normalized schemas is the real performance bottleneck."

> "The most important decision in dimensional modeling is the grain. Before I pick any dimensions or measures, I state explicitly what one row represents. This prevents double-counting and ensures the model answers the right questions."

> "I think of fact table types as answering different temporal questions: transactional tells me what happened, periodic snapshot tells me what the state was, and accumulating snapshot tells me how long each stage took."

> "For SCDs, I default to Type 2 because it preserves history. The surrogate key pattern — where the natural key is not unique — lets me join to the correct version of a dimension at any point in time."

> "Galaxy schema is just what happens when you have multiple business processes sharing conformed dimensions. The 'bus matrix' is how I communicate this — processes as rows, dimensions as columns, checkmarks where they connect."

---

## Common Interview Mistakes

| ❌ Mistake | ✅ Better Approach |
|-----------|-------------------|
| Jumping to table design without stating the grain | Always declare grain first: "One row per X per Y" |
| Putting measures in dimension tables | Measures live in fact tables; dimensions are descriptive |
| Creating one giant fact table for everything | Separate fact tables per business process, share dimensions |
| Normalizing dimensions in an analytics warehouse | Default to star; snowflake only with explicit justification |
| Forgetting semi-additive facts | Balances/levels cannot be summed across time — use AVG or latest |
| Ignoring slowly changing dimensions | Address SCD strategy for dimensions that change (Type 2 default) |
| Storing derived metrics in fact tables | Store atomic measures; compute ratios in the query layer |
| Using natural keys as primary keys | Use surrogate keys (integers) — enables SCD Type 2 and performs better |
| Not considering query patterns | Design for how analysts will GROUP BY and filter |
| Mixing operational and analytical concerns | OLTP is normalized for writes; OLAP is denormalized for reads |

---

## Rapid-Fire Q&A

**Q1: What is the difference between a fact and a dimension?**
> Facts are numeric measures you aggregate (SUM, AVG, COUNT). Dimensions are the textual/categorical attributes you filter and group by. Facts answer "how much/many"; dimensions answer "of what."

**Q2: Can a fact table have no measures?**
> Yes — a "factless fact table" records events with no numeric measures. Example: student attendance (student_key, class_key, date_key). The measure is implicit: COUNT(*).

**Q3: What is a surrogate key and why use one?**
> An integer primary key generated by the warehouse (not from the source system). It enables SCD Type 2 versioning, improves join performance, and insulates the warehouse from source key changes.

**Q4: When would you choose snowflake over star?**
> When dimension tables have millions of rows with deep hierarchies that change frequently, and the storage or update cost of denormalization is prohibitive. Rare in modern cloud DWs.

**Q5: What is a conformed dimension?**
> A dimension shared across multiple fact tables with identical keys, attributes, and semantics. Example: dim_date used by both fact_sales and fact_inventory. Enables cross-process analysis.

**Q6: How do you handle many-to-many relationships?**
> Use a bridge table (also called a factless fact table or association table). Example: a patient can have multiple diagnoses, and a diagnosis applies to multiple patients — bridge_patient_diagnosis resolves this.

**Q7: What is additivity in fact tables?**
> Additive facts can be summed across all dimensions (revenue). Semi-additive facts cannot be summed across time (account balance). Non-additive facts cannot be summed at all (unit price, ratios).

**Q8: How does partitioning relate to dimensional modeling?**
> Partition fact tables by the most common filter (usually date). This allows the query engine to prune irrelevant data. It is a physical optimization orthogonal to logical schema design.

**Q9: What is a bus matrix?**
> A planning tool: rows = business processes, columns = dimensions. Checkmarks indicate which dimensions participate in which process. It ensures dimension conformity across the warehouse.

**Q10: How do you model hierarchies in star schema?**
> Flatten all levels into the dimension table as separate columns (department, category, subcategory, product). This avoids JOINs and supports drill-down at any level.

---

## Summary Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STAR vs SNOWFLAKE — CHEAT SHEET                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  STAR SCHEMA (Default Choice)                                           │
│  ─────────────────────────────                                          │
│  • Fact table + denormalized dimensions (one join per dim)              │
│  • Optimized for: fast queries, BI tools, analyst self-service          │
│  • Tradeoff: some data redundancy in dimensions                         │
│                                                                         │
│  SNOWFLAKE SCHEMA (Special Cases)                                       │
│  ────────────────────────────────                                       │
│  • Fact table + normalized dimensions (multi-level joins)               │
│  • Optimized for: storage, referential integrity, infrequent queries    │
│  • Tradeoff: query complexity and performance                           │
│                                                                         │
│  INTERVIEW FRAMEWORK                                                    │
│  ───────────────────                                                    │
│  1. Clarify business process → 2. Declare grain                         │
│  3. Identify dimensions (5W+H) → 4. Identify measures                  │
│  5. Special dims (SCD, role-playing, junk, degenerate)                  │
│  6. Complementary fact tables → 7. Physical implementation              │
│                                                                         │
│  FACT TABLE TYPES                                                       │
│  ────────────────                                                       │
│  Transactional  → What happened (one event per row, append-only)        │
│  Periodic Snap  → What the state was (entity × period, semi-additive)   │
│  Accumulating   → How long it took (entity lifecycle, updated rows)     │
│                                                                         │
│  KEY VOCABULARY                                                         │
│  ──────────────                                                         │
│  Grain: what one row represents                                         │
│  Conformed dim: shared across fact tables                               │
│  Surrogate key: warehouse-generated integer PK                          │
│  Degenerate dim: key with no dimension table (e.g., order_number)       │
│  Junk dim: catch-all for low-cardinality flags                          │
│  Role-playing dim: same table used with different meanings               │
│  SCD Type 2: versioned rows with effective/expiration dates             │
│  Bus matrix: processes × dimensions planning grid                       │
│                                                                         │
│  MODERN REALITY                                                         │
│  ──────────────                                                         │
│  Storage is cheap → denormalize freely                                  │
│  Distributed joins are expensive → minimize joins (star wins)           │
│  dbt + medallion architecture → Kimball marts on Inmon-style raw layer  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 8 of 45*
