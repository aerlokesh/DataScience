# 🎯 Topic 17: dbt and the Transformation Layer

> *Analytics engineering with dbt — the tool that brought software engineering best practices (version control, testing, documentation) to SQL transformations in the data warehouse*

---

## 📋 Table of Contents

1. [Core Philosophy](#core-philosophy)
2. [Analytics Engineering](#analytics-engineering)
3. [Model Layers](#model-layers)
4. [ref() and source()](#ref-and-source)
5. [Testing Framework](#testing-framework)
6. [Documentation and Lineage](#documentation-and-lineage)
7. [Incremental Models](#incremental-models)
8. [Snapshots for SCD Type 2](#snapshots-for-scd-type-2)
9. [Packages and Macros](#packages-and-macros)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Core Philosophy

### What is dbt?

**dbt (data build tool)** is a transformation framework that enables analysts and engineers to transform data in the warehouse using SQL, with software engineering practices built in.

> **Critical Insight:** dbt is not an ETL tool — it only handles the "T" (Transform). It assumes data is already loaded into your warehouse (the "EL" is handled by Fivetran, Airbyte, etc.). dbt runs SQL transformations against your warehouse using its native compute — it pushes computation down to Snowflake, BigQuery, Redshift, or Databricks.

### dbt's Key Principles

| Principle | What It Means |
|-----------|--------------|
| **SQL-first** | Analysts write SELECT statements; dbt handles DDL/DML |
| **Version controlled** | All transformations live in Git — reviewable, auditable |
| **Tested** | Data quality assertions run after every build |
| **Documented** | Auto-generated docs with lineage visualization |
| **Modular** | Small, reusable models composed via `ref()` |
| **Idempotent** | Every `dbt run` produces the same result for the same input |

### dbt Core vs dbt Cloud

| Feature | dbt Core | dbt Cloud |
|---------|----------|-----------|
| **Execution** | CLI, self-managed | Managed SaaS |
| **Scheduling** | External (Airflow, cron) | Built-in scheduler |
| **IDE** | VS Code, terminal | Browser-based IDE |
| **CI/CD** | Configure yourself (GitHub Actions) | Built-in CI on PRs |
| **Cost** | Free (open source) | Paid (per seat) |
| **Governance** | Manual | Environment management, permissions |

---

## Analytics Engineering

### The Role

Analytics engineers sit between data engineers and data analysts:

```
Data Engineer        Analytics Engineer       Data Analyst
─────────────        ──────────────────       ────────────
Builds pipelines     Transforms & models      Builds dashboards
Manages infra        Defines business logic   Answers questions
Source → Raw         Raw → Marts              Marts → Insights
Python/Scala         SQL + dbt                SQL + BI tools
```

> **Critical Insight:** Before analytics engineering, data analysts wrote complex SQL with many CTEs and subqueries directly in their BI tools. This created unmaintainable spaghetti. dbt introduced the concept of reusable, tested, documented models — turning ad-hoc SQL into maintainable data products.

### Project Structure

```
my_dbt_project/
├── dbt_project.yml          # Project configuration
├── profiles.yml             # Connection settings (local only)
├── models/
│   ├── staging/             # Light transformations on raw data
│   │   ├── stripe/
│   │   │   ├── _stripe__models.yml   # Tests & docs
│   │   │   ├── stg_stripe__payments.sql
│   │   │   └── stg_stripe__customers.sql
│   │   └── shopify/
│   │       ├── _shopify__models.yml
│   │       └── stg_shopify__orders.sql
│   ├── intermediate/        # Business logic joins
│   │   └── int_orders_enriched.sql
│   └── marts/               # Consumption-ready tables
│       ├── finance/
│       │   └── fct_revenue.sql
│       └── marketing/
│           └── dim_customers.sql
├── tests/                   # Custom singular tests
├── macros/                  # Reusable SQL functions
├── seeds/                   # CSV lookup files
├── snapshots/               # SCD Type 2 tracking
└── packages.yml             # External package dependencies
```

---

## Model Layers

### The Layered Architecture

```
┌─────────────────────────────────────────────────────┐
│  MARTS (fct_, dim_)                                  │
│  Business-ready tables, optimized for consumers      │
├─────────────────────────────────────────────────────┤
│  INTERMEDIATE (int_)                                 │
│  Complex joins, business logic, prep for marts       │
├─────────────────────────────────────────────────────┤
│  STAGING (stg_)                                      │
│  1:1 with source, renamed, retyped, cleaned          │
├─────────────────────────────────────────────────────┤
│  RAW / SOURCE                                        │
│  Loaded by ingestion tools (Fivetran, Airbyte)       │
└─────────────────────────────────────────────────────┘
```

### Staging Models

Rules for staging:
- One staging model per source table
- Only renaming, recasting, basic cleaning
- No joins, no aggregations
- Materialized as **views** (no storage cost)

```sql
-- models/staging/stripe/stg_stripe__payments.sql

WITH source AS (
    SELECT * FROM {{ source('stripe', 'payments') }}
),

renamed AS (
    SELECT
        id AS payment_id,
        customer_id,
        amount / 100.0 AS amount_dollars,  -- cents to dollars
        currency,
        CAST(created AS TIMESTAMP) AS created_at,
        status,
        CASE
            WHEN status = 'succeeded' THEN TRUE
            ELSE FALSE
        END AS is_successful,
        _fivetran_synced AS loaded_at
    FROM source
)

SELECT * FROM renamed
```

### Intermediate Models

Rules for intermediate:
- Complex joins between staging models
- Business logic that serves multiple marts
- Materialized as **views** or **ephemeral** (CTEs)

```sql
-- models/intermediate/int_orders_enriched.sql

WITH orders AS (
    SELECT * FROM {{ ref('stg_shopify__orders') }}
),

payments AS (
    SELECT * FROM {{ ref('stg_stripe__payments') }}
),

customers AS (
    SELECT * FROM {{ ref('stg_stripe__customers') }}
)

SELECT
    o.order_id,
    o.order_date,
    o.customer_id,
    c.customer_name,
    c.customer_segment,
    p.amount_dollars AS payment_amount,
    p.is_successful AS payment_succeeded,
    DATEDIFF('day', c.first_order_date, o.order_date) AS customer_tenure_days
FROM orders o
LEFT JOIN payments p ON o.payment_id = p.payment_id
LEFT JOIN customers c ON o.customer_id = c.customer_id
```

### Mart Models

Rules for marts:
- Organized by business domain (finance, marketing, product)
- Uses `fct_` (fact) and `dim_` (dimension) naming
- Materialized as **tables** (fast queries for BI tools)

```sql
-- models/marts/finance/fct_monthly_revenue.sql

{{ config(materialized='table', schema='finance') }}

WITH enriched_orders AS (
    SELECT * FROM {{ ref('int_orders_enriched') }}
)

SELECT
    DATE_TRUNC('month', order_date) AS revenue_month,
    customer_segment,
    COUNT(DISTINCT order_id) AS total_orders,
    COUNT(DISTINCT customer_id) AS unique_customers,
    SUM(CASE WHEN payment_succeeded THEN payment_amount ELSE 0 END) AS gross_revenue,
    SUM(CASE WHEN NOT payment_succeeded THEN payment_amount ELSE 0 END) AS failed_payments
FROM enriched_orders
GROUP BY 1, 2
```

---

## ref() and source()

### The `ref()` Function

`ref()` is dbt's dependency management system. It:
1. Creates a dependency edge in the DAG
2. Resolves to the correct table/view name in the target environment
3. Enables dbt to determine execution order

```sql
-- This creates a dependency: fct_revenue depends on stg_stripe__payments
SELECT * FROM {{ ref('stg_stripe__payments') }}

-- Compiles to (in production):
-- SELECT * FROM analytics_prod.staging.stg_stripe__payments

-- Compiles to (in development):
-- SELECT * FROM dev_jsmith.staging.stg_stripe__payments
```

### The `source()` Function

`source()` references raw tables loaded by external tools:

```yaml
# models/staging/stripe/_stripe__sources.yml
sources:
  - name: stripe
    database: raw_database
    schema: stripe_data
    tables:
      - name: payments
        loaded_at_field: _fivetran_synced
        freshness:
          warn_after: {count: 12, period: hour}
          error_after: {count: 24, period: hour}
      - name: customers
```

```sql
-- References the source definition above
SELECT * FROM {{ source('stripe', 'payments') }}
-- Compiles to: SELECT * FROM raw_database.stripe_data.payments
```

> **Critical Insight:** Never use hardcoded table references in dbt models. Always use `ref()` for dbt models and `source()` for raw tables. This enables environment-aware compilation, automatic DAG construction, and freshness monitoring.

---

## Testing Framework

### Built-in Generic Tests

```yaml
# models/staging/stripe/_stripe__models.yml
models:
  - name: stg_stripe__payments
    description: "Staged Stripe payments, one row per payment attempt"
    columns:
      - name: payment_id
        description: "Primary key"
        tests:
          - unique
          - not_null
      - name: status
        tests:
          - accepted_values:
              values: ['succeeded', 'failed', 'pending', 'refunded']
      - name: customer_id
        tests:
          - not_null
          - relationships:
              to: ref('stg_stripe__customers')
              field: customer_id
```

### Test Types

| Test | What It Checks | Example |
|------|---------------|---------|
| **unique** | No duplicate values | Primary keys |
| **not_null** | No NULL values | Required fields |
| **accepted_values** | Values in allowed set | Status codes, categories |
| **relationships** | Referential integrity | Foreign key → parent table |

### Custom Singular Tests

```sql
-- tests/assert_revenue_is_positive.sql
-- This test PASSES if the query returns 0 rows

SELECT
    revenue_month,
    gross_revenue
FROM {{ ref('fct_monthly_revenue') }}
WHERE gross_revenue < 0
```

### Custom Generic Tests (Macros)

```sql
-- macros/test_is_positive.sql
{% test is_positive(model, column_name) %}

SELECT {{ column_name }}
FROM {{ model }}
WHERE {{ column_name }} < 0

{% endtest %}
```

```yaml
# Usage in schema.yml
columns:
  - name: amount_dollars
    tests:
      - is_positive
```

### Running Tests

```bash
# Run all tests
dbt test

# Run tests for a specific model
dbt test --select stg_stripe__payments

# Run tests for a model and all its children
dbt test --select stg_stripe__payments+

# Run only schema tests (generic tests from YAML)
dbt test --select test_type:generic

# Run only custom singular tests
dbt test --select test_type:singular
```

---

## Documentation and Lineage

### Auto-Generated Documentation

```bash
# Generate docs site
dbt docs generate

# Serve locally
dbt docs serve --port 8080
```

### Documentation in YAML

```yaml
models:
  - name: fct_monthly_revenue
    description: |
      Monthly revenue aggregation by customer segment.
      Grain: one row per month per segment.
      Source: Stripe payments (only successful).
      Owner: Finance team.
      SLA: Refreshed by 8 AM UTC daily.
    columns:
      - name: revenue_month
        description: "First day of the month (DATE)"
      - name: gross_revenue
        description: "Sum of successful payment amounts in USD"
```

### Lineage Graph

dbt automatically builds a lineage graph from `ref()` and `source()` calls:

```
source(stripe.payments)
        │
        ▼
stg_stripe__payments ──────┐
        │                   │
        ▼                   ▼
int_orders_enriched    dim_customers
        │
        ▼
fct_monthly_revenue
```

> **Critical Insight:** dbt's lineage is not just documentation — it drives execution. `dbt run --select fct_monthly_revenue+` runs a model and all downstream dependents. `dbt run --select +fct_monthly_revenue` runs all ancestors. This graph-aware execution is what makes selective builds possible.

---

## Incremental Models

### Why Incremental?

Full table refreshes become expensive at scale. Incremental models only process new/changed data.

```sql
-- models/marts/fct_events.sql

{{ config(
    materialized='incremental',
    unique_key='event_id',
    incremental_strategy='merge',
    on_schema_change='append_new_columns'
) }}

SELECT
    event_id,
    user_id,
    event_type,
    event_timestamp,
    properties
FROM {{ ref('stg_amplitude__events') }}

{% if is_incremental() %}
    -- Only process events newer than the latest in our table
    WHERE event_timestamp > (SELECT MAX(event_timestamp) FROM {{ this }})
{% endif %}
```

### Incremental Strategies

| Strategy | How It Works | Best For |
|----------|-------------|----------|
| **append** | Insert new rows only | Immutable event data |
| **merge** | MERGE/UPSERT on unique_key | Mutable data (can be updated) |
| **delete+insert** | Delete matching rows, then insert | Partitioned tables |
| **insert_overwrite** | Overwrite entire partitions | BigQuery partitioned tables |

### Full Refresh Override

```bash
# Force a full rebuild of an incremental model
dbt run --select fct_events --full-refresh
```

> **Critical Insight:** Incremental models trade correctness guarantees for performance. If your source data can be updated retroactively (late-arriving data, corrections), a pure "WHERE timestamp > max" watermark will miss those updates. Use `merge` strategy with a lookback window, or implement periodic full refreshes.

---

## Snapshots for SCD Type 2

### What Are Snapshots?

Snapshots capture how a record changes over time — implementing Slowly Changing Dimension Type 2 (SCD2).

```sql
-- snapshots/snap_customers.sql

{% snapshot snap_customers %}

{{
    config(
        target_schema='snapshots',
        unique_key='customer_id',
        strategy='timestamp',
        updated_at='updated_at',
        invalidate_hard_deletes=True,
    )
}}

SELECT
    customer_id,
    customer_name,
    email,
    plan_type,
    mrr,
    updated_at
FROM {{ source('app_db', 'customers') }}

{% endsnapshot %}
```

### Snapshot Output

| customer_id | plan_type | mrr | dbt_valid_from | dbt_valid_to |
|-------------|-----------|-----|----------------|--------------|
| 101 | free | 0 | 2024-01-01 | 2024-03-15 |
| 101 | pro | 49 | 2024-03-15 | 2024-08-01 |
| 101 | enterprise | 199 | 2024-08-01 | NULL |

The row with `dbt_valid_to = NULL` is the current record.

### Snapshot Strategies

| Strategy | Detects Changes Via | Pros | Cons |
|----------|-------------------|------|------|
| **timestamp** | `updated_at` column | Efficient, only checks timestamp | Requires reliable timestamp |
| **check** | Comparing column values | Catches all changes | Expensive (compares all columns) |

```sql
-- Check strategy: snapshot changes to specific columns
{% snapshot snap_products %}
{{
    config(
        strategy='check',
        unique_key='product_id',
        check_cols=['price', 'status', 'category'],
    )
}}
SELECT * FROM {{ source('app_db', 'products') }}
{% endsnapshot %}
```

---

## Packages and Macros

### Installing Packages

```yaml
# packages.yml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.0.0", "<2.0.0"]
  - package: calogica/dbt_expectations
    version: [">=0.8.0", "<1.0.0"]
  - package: dbt-labs/codegen
    version: [">=0.9.0", "<1.0.0"]
```

```bash
dbt deps  # Install packages
```

### Popular Packages

| Package | Purpose |
|---------|---------|
| **dbt_utils** | Surrogate keys, pivots, date spines, union relations |
| **dbt_expectations** | Great Expectations-style tests in dbt |
| **codegen** | Generate model YAML, staging models from sources |
| **dbt_date** | Date dimension generation |
| **audit_helper** | Compare query results (useful for migration validation) |

### Writing Macros

```sql
-- macros/cents_to_dollars.sql
{% macro cents_to_dollars(column_name, precision=2) %}
    ROUND({{ column_name }} / 100.0, {{ precision }})
{% endmacro %}

-- Usage in a model:
SELECT
    payment_id,
    {{ cents_to_dollars('amount_cents') }} AS amount_dollars
FROM {{ ref('stg_stripe__payments') }}
```

### Useful dbt_utils Examples

```sql
-- Generate a surrogate key
SELECT
    {{ dbt_utils.generate_surrogate_key(['order_id', 'line_item_id']) }} AS order_line_key,
    ...

-- Pivot rows to columns
{{ dbt_utils.pivot(
    'status',
    dbt_utils.get_column_values(ref('stg_orders'), 'status')
) }}

-- Union multiple source tables with the same schema
{{ dbt_utils.union_relations(
    relations=[ref('stg_us_orders'), ref('stg_eu_orders'), ref('stg_apac_orders')]
) }}
```

---

## Interview Talking Points

> **"What is dbt and why would you use it?"**
>
> "dbt is a transformation framework that lets you write SELECT statements in SQL and handles the DDL/DML to materialize them as tables or views in your warehouse. The key value isn't the SQL execution — it's the software engineering practices it brings: version control via Git, automated testing of data quality, auto-generated documentation with lineage, modular design through ref(), and environment management for dev/staging/prod. It turns SQL analysts into analytics engineers."

> **"Walk me through how you'd structure a dbt project."**
>
> "I use a layered architecture. Staging models are 1:1 with source tables — just renaming, recasting, and light cleaning, materialized as views. Intermediate models handle complex joins and business logic shared across multiple marts. Mart models are the consumption layer organized by business domain — facts and dimensions optimized for BI tools, materialized as tables. Each layer has tests: unique/not_null on keys, accepted_values for enums, relationships for referential integrity."

> **"How do you handle slowly changing dimensions in dbt?"**
>
> "I use dbt snapshots with SCD Type 2. You define the source table, the unique key, and a change detection strategy — either timestamp-based or check-based. dbt maintains valid_from and valid_to columns, creating a new row each time a tracked change occurs. The current record has a NULL valid_to. This preserves full history for temporal analysis."

---

## Common Mistakes

| ❌ Mistake | ✅ Best Practice |
|-----------|-----------------|
| Hardcoding table names instead of using `ref()` | Always use `ref()` for models, `source()` for raw tables |
| Putting business logic in staging models | Staging = rename + retype only; logic belongs in intermediate/marts |
| Materializing everything as tables | Views for staging, tables for marts, incremental for large fact tables |
| No tests on models | Test every primary key (unique + not_null) and critical fields |
| One massive model with 500 lines of SQL | Break into smaller models joined via `ref()` |
| Using `SELECT *` in staging models | Explicitly list columns — documents the contract, catches schema drift |
| Incremental without a lookback window | Add a lookback (e.g., 3 days) to catch late-arriving data |
| Not running `dbt test` in CI/CD | Tests should block deployment if they fail |
| Ignoring documentation | Write descriptions — future you will thank present you |
| Snapshot without understanding source timestamps | Verify `updated_at` is reliable before using timestamp strategy |

---

## Rapid-Fire Q&A

**Q1: What does `ref()` do in dbt?**
> Creates a dependency between models, enables DAG construction, and resolves to the correct table name based on the target environment (dev/prod).

**Q2: What are the four built-in generic tests?**
> `unique`, `not_null`, `accepted_values`, `relationships`. They cover primary keys, required fields, valid enums, and referential integrity.

**Q3: What is the difference between a view and a table materialization?**
> Views store only the SQL definition (no storage cost, always fresh). Tables store the query result (fast reads, but stale until rebuilt). Use views for staging, tables for marts.

**Q4: How does an incremental model work?**
> On first run, it builds the full table. On subsequent runs, it only processes new rows (using `is_incremental()` filter) and merges them into the existing table.

**Q5: What is `{{ this }}` in dbt?**
> Refers to the current model's already-materialized table. Used in incremental models to reference the existing data (e.g., `SELECT MAX(timestamp) FROM {{ this }}`).

**Q6: How do snapshots implement SCD Type 2?**
> They track changes to source records by comparing timestamps or column values, creating new rows with `dbt_valid_from`/`dbt_valid_to` columns for each change.

**Q7: What is the purpose of the `source()` function?**
> References raw tables loaded by external tools. Enables freshness monitoring, documentation, and lineage from raw data into the dbt DAG.

**Q8: What does `dbt run --select model_name+` do?**
> Runs the specified model AND all of its downstream dependents (children). The `+` suffix means "and everything downstream."

**Q9: What is a macro in dbt?**
> A reusable Jinja template function that generates SQL. Used for DRY patterns like currency conversion, surrogate keys, or custom tests.

**Q10: How do you handle schema changes in incremental models?**
> Use `on_schema_change` config: `ignore` (default), `append_new_columns`, `sync_all_columns`, or `fail`. Choose based on your tolerance for schema drift.

---

## ASCII Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════╗
║                  dbt CHEAT SHEET                                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  MODEL LAYERS:                                                   ║
║  source() ─▶ stg_ (views) ─▶ int_ (views) ─▶ fct_/dim_ (tables)║
║                                                                  ║
║  COMMANDS:                                                       ║
║  dbt run              Build models                               ║
║  dbt test             Run all tests                              ║
║  dbt build            Run + test (in DAG order)                  ║
║  dbt run --select +model_name+   Model + ancestors + children    ║
║  dbt run --full-refresh          Rebuild incremental from scratch║
║  dbt docs generate    Build documentation site                   ║
║  dbt source freshness Check raw data recency                     ║
║  dbt snapshot         Run SCD2 snapshots                         ║
║  dbt deps             Install packages                           ║
║                                                                  ║
║  MATERIALIZATIONS:                                               ║
║  ┌──────────────┬───────────────────────────────────────┐        ║
║  │ view         │ No storage, always fresh, slow reads  │        ║
║  │ table        │ Stored, fast reads, full rebuild      │        ║
║  │ incremental  │ Append/merge new rows only            │        ║
║  │ ephemeral    │ CTE injected into parent model        │        ║
║  └──────────────┴───────────────────────────────────────┘        ║
║                                                                  ║
║  SELECTION SYNTAX:                                               ║
║  model_name      Just this model                                 ║
║  +model_name     Model + all ancestors                           ║
║  model_name+     Model + all descendants                         ║
║  +model_name+    Full lineage (ancestors + model + descendants)  ║
║  tag:finance     All models with this tag                        ║
║  source:stripe+  Source and all downstream models                ║
║                                                                  ║
║  TESTING:                                                        ║
║  unique          │  No duplicate values in column                ║
║  not_null        │  No NULL values                               ║
║  accepted_values │  Values in allowed list                       ║
║  relationships   │  FK exists in parent table                    ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 17 of 45*
