# 🎯 Topic 13: Data Warehouse Architecture

> *A data warehouse is not just a database — it's a layered system of ingestion, storage, transformation, and serving that turns raw operational chaos into trusted, query-ready analytics. Understanding the architecture is what separates a data engineer from someone who just writes SQL.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [The Layered Architecture](#the-layered-architecture)
3. [Raw/Staging Layer](#rawstaging-layer)
4. [Transform Layer](#transform-layer)
5. [Data Marts Layer](#data-marts-layer)
6. [BigQuery Architecture](#bigquery-architecture)
7. [Snowflake Architecture](#snowflake-architecture)
8. [Redshift Architecture](#redshift-architecture)
9. [Cost Models Comparison](#cost-models-comparison)
10. [Platform Comparison Table](#platform-comparison-table)
11. [Interview Talking Points](#interview-talking-points)
12. [Common Mistakes](#common-mistakes)
13. [Rapid-Fire Q&A](#rapid-fire-qa)
14. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Introduction

A data warehouse is a **subject-oriented, integrated, time-variant, non-volatile** collection of data supporting management decisions (Bill Inmon's definition). Modern cloud data warehouses have evolved far beyond this — they now support semi-structured data, real-time streaming, ML workloads, and even data sharing across organizations.

The architecture must answer:
- How does data get in? (Ingestion)
- How is it organized? (Storage layers)
- How is it transformed? (Processing)
- How is it consumed? (Serving/access)
- How much does it cost? (Compute/storage economics)

> **Critical Insight:** The most impactful architectural decision isn't which platform to choose — it's how you layer your data. A well-designed Raw -> Staging -> Transform -> Mart architecture works on any platform. A poorly designed single-layer approach fails on every platform.

---

## The Layered Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATA WAREHOUSE LAYERS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  DATA SOURCES                                                ││
│  │  PostgreSQL, MySQL, APIs, S3, Kafka, SaaS tools (Stripe...)  ││
│  └──────────────────────────┬──────────────────────────────────┘│
│                             │ Ingestion (Fivetran, Airbyte, CDC) │
│                             ▼                                    │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  RAW / STAGING LAYER (Bronze)                                ││
│  │  • Exact copy of source data                                 ││
│  │  • Append-only, immutable                                    ││
│  │  • Schema matches source                                     ││
│  │  • Full load + incremental                                   ││
│  └──────────────────────────┬──────────────────────────────────┘│
│                             │ dbt / Dataform / Stored Procs      │
│                             ▼                                    │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  TRANSFORM LAYER (Silver)                                    ││
│  │  • Cleaned, deduplicated, typed                              ││
│  │  • Business logic applied                                    ││
│  │  • Conformed dimensions built                                ││
│  │  • Incremental models                                        ││
│  └──────────────────────────┬──────────────────────────────────┘│
│                             │ Aggregation + domain modeling      │
│                             ▼                                    │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  DATA MARTS (Gold)                                           ││
│  │  • Business-domain specific                                  ││
│  │  • Pre-aggregated for performance                            ││
│  │  • Star/snowflake schema                                     ││
│  │  • BI tool optimized                                         ││
│  └─────────────────────────────────────────────────────────────┘│
│                             │                                    │
│                             ▼                                    │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  CONSUMERS                                                   ││
│  │  BI Tools (Tableau, Looker) │ ML Models │ Reverse ETL │ APIs ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Raw/Staging Layer

### Purpose

The raw layer is an **exact replica** of source systems within the warehouse. It serves as the single source of truth that all downstream transformations reference.

### Design Principles

| Principle | Implementation |
|-----------|---------------|
| Immutable | Never modify raw data; append-only or full snapshot |
| Source-faithful | Column names, types, and values match source exactly |
| Timestamped | Every load has a metadata column (_loaded_at, _batch_id) |
| Complete | Load all columns, even if not immediately needed |
| Partitioned | By ingestion date for cost control and reprocessing |

### Example: Raw Layer Tables

```sql
-- Raw table: exact copy from source PostgreSQL
CREATE TABLE raw.orders (
    -- Source columns (exactly as they appear in source)
    id              BIGINT,
    customer_id     BIGINT,
    order_date      TIMESTAMP,
    status          VARCHAR(50),
    total_amount    DECIMAL(12,2),
    shipping_address TEXT,
    
    -- Metadata columns (added by ingestion)
    _loaded_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    _source         VARCHAR(50) DEFAULT 'production_postgres',
    _batch_id       VARCHAR(100)
);

-- Handling schema evolution: use JSON for volatile sources
CREATE TABLE raw.api_events (
    event_id        VARCHAR(100),
    received_at     TIMESTAMP,
    raw_payload     JSONB,              -- Entire API response
    _loaded_at      TIMESTAMP,
    _source         VARCHAR(50)
);
```

### Ingestion Patterns

| Pattern | When to Use | Tools |
|---------|------------|-------|
| Full refresh | Small tables, reference data | Fivetran, Airbyte |
| Incremental (append) | Event data, logs | CDC, Kafka Connect |
| Incremental (merge) | Mutable source records | Fivetran (incremental + dedupe) |
| Change Data Capture | Real-time needs, large tables | Debezium, AWS DMS |

---

## Transform Layer

### Purpose

The transform layer applies **business logic**, cleans data, and creates reusable intermediate models. This is where dbt/Dataform models live.

### Common Transformations

```sql
-- 1. Deduplication
CREATE TABLE transform.orders AS
SELECT DISTINCT ON (order_id) *
FROM raw.orders
ORDER BY order_id, _loaded_at DESC;

-- 2. Type casting and cleaning
CREATE TABLE transform.customers AS
SELECT
    id AS customer_id,
    TRIM(LOWER(email)) AS email,
    INITCAP(first_name) AS first_name,
    INITCAP(last_name) AS last_name,
    CAST(created_at AS DATE) AS signup_date,
    CASE WHEN status IN ('active', 'ACTIVE', 'Active') THEN 'active'
         WHEN status IN ('churned', 'CHURNED', 'cancelled') THEN 'churned'
         ELSE 'unknown' END AS status_clean
FROM raw.customers;

-- 3. Business logic (enrichment)
CREATE TABLE transform.orders_enriched AS
SELECT
    o.*,
    c.segment AS customer_segment,
    p.category AS product_category,
    o.total_amount - o.discount_amount AS net_revenue,
    DATEDIFF(day, c.signup_date, o.order_date) AS days_since_signup,
    ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY o.order_date) 
        AS order_sequence_number
FROM transform.orders o
JOIN transform.customers c ON o.customer_id = c.customer_id
JOIN transform.products p ON o.product_id = p.product_id;
```

### dbt Model Organization

```
models/
├── staging/          ← 1:1 with raw tables, light cleaning
│   ├── stg_orders.sql
│   ├── stg_customers.sql
│   └── stg_products.sql
├── intermediate/     ← Business logic, joins, enrichment
│   ├── int_orders_enriched.sql
│   └── int_customer_lifetime.sql
└── marts/            ← Final consumption models
    ├── finance/
    │   └── fct_revenue.sql
    ├── marketing/
    │   └── fct_campaign_performance.sql
    └── core/
        ├── dim_customers.sql
        └── dim_products.sql
```

> **Critical Insight:** The transform layer should be IDEMPOTENT — running it twice produces the same result. This means using MERGE/UPSERT patterns for incremental models and avoiding side effects. If your pipeline breaks, you should be able to re-run from raw without manual intervention.

---

## Data Marts Layer

### Purpose

Data marts are **domain-specific**, pre-aggregated views of the warehouse designed for specific business teams or use cases.

### Design Principles

| Principle | Rationale |
|-----------|-----------|
| Domain-focused | One mart per business function (finance, marketing, product) |
| Pre-aggregated | Common calculations done upfront, not at query time |
| Star schema | Optimized for BI tools (fact + dimension tables) |
| SLA-bound | Guaranteed freshness and availability for stakeholders |
| Governed | Access controls, data quality tests, documentation |

### Example: Marketing Analytics Mart

```sql
-- Mart: marketing campaign performance (pre-aggregated)
CREATE TABLE marts.marketing.fct_campaign_daily AS
SELECT
    d.date_key,
    c.campaign_id,
    c.campaign_name,
    c.channel,
    c.target_segment,
    COUNT(DISTINCT e.user_id) AS users_reached,
    SUM(e.impressions) AS total_impressions,
    SUM(e.clicks) AS total_clicks,
    SUM(e.conversions) AS total_conversions,
    SUM(e.spend) AS total_spend,
    SAFE_DIVIDE(SUM(e.clicks), SUM(e.impressions)) AS ctr,
    SAFE_DIVIDE(SUM(e.spend), SUM(e.conversions)) AS cost_per_conversion,
    SAFE_DIVIDE(SUM(e.revenue), SUM(e.spend)) AS roas
FROM transform.campaign_events e
JOIN transform.campaigns c ON e.campaign_id = c.campaign_id
JOIN transform.dim_date d ON CAST(e.event_date AS DATE) = d.calendar_date
GROUP BY 1, 2, 3, 4, 5;
```

---

## BigQuery Architecture

### Core Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    BigQuery                               │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  DREMEL (Compute Layer)                           │   │
│  │  • Distributed query execution engine             │   │
│  │  • Tree architecture (root → mixer → leaf)        │   │
│  │  • Auto-scales slots per query                    │   │
│  │  • No cluster management needed                   │   │
│  └──────────────────────────────────────────────────┘   │
│                         │                                │
│                         ▼                                │
│  ┌──────────────────────────────────────────────────┐   │
│  │  COLOSSUS (Storage Layer)                         │   │
│  │  • Google's distributed filesystem                │   │
│  │  • Capacitor columnar format                      │   │
│  │  • Automatic compression and encryption           │   │
│  │  • Separated from compute (pay independently)     │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  JUPITER (Network)                                │   │
│  │  • Petabit-scale network between compute/storage  │   │
│  │  • Enables separation without performance loss    │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Key BigQuery Features

| Feature | Benefit |
|---------|---------|
| Serverless | No cluster management, auto-scales |
| Partitioning | Date/integer partitions reduce scan cost |
| Clustering | Sorts data within partitions for filter efficiency |
| Materialized views | Auto-maintained pre-aggregations |
| BI Engine | In-memory acceleration for dashboards |
| Streaming inserts | Real-time data availability |
| ML integration | BQML — train models with SQL |
| Slots | Unit of compute; shared pool or reserved |

### BigQuery Optimization

```sql
-- Partition by date (reduces bytes scanned = reduces cost)
CREATE TABLE project.analytics.fact_orders
PARTITION BY DATE(order_date)
CLUSTER BY customer_segment, product_category
AS SELECT * FROM transform.orders_enriched;

-- Query with partition filter (CRITICAL for cost)
SELECT SUM(revenue) 
FROM analytics.fact_orders
WHERE order_date >= '2024-01-01'   -- Partition pruning
  AND customer_segment = 'enterprise';  -- Cluster filtering
```

---

## Snowflake Architecture

### Core Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Snowflake                              │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  CLOUD SERVICES LAYER                             │   │
│  │  • Query optimization and compilation             │   │
│  │  • Metadata management                            │   │
│  │  • Access control and authentication              │   │
│  │  • Infrastructure management                      │   │
│  └──────────────────────────────────────────────────┘   │
│                         │                                │
│                         ▼                                │
│  ┌──────────────────────────────────────────────────┐   │
│  │  VIRTUAL WAREHOUSES (Compute Layer)               │   │
│  │  • Independent compute clusters                   │   │
│  │  • Scale up (bigger) or out (more clusters)       │   │
│  │  • Auto-suspend and auto-resume                   │   │
│  │  • Concurrent workload isolation                  │   │
│  │                                                    │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐          │   │
│  │  │ ETL_WH  │  │ANALYTICS│  │  ML_WH  │          │   │
│  │  │ X-Large │  │  Medium │  │  Large  │          │   │
│  │  └─────────┘  └─────────┘  └─────────┘          │   │
│  └──────────────────────────────────────────────────┘   │
│                         │                                │
│                         ▼                                │
│  ┌──────────────────────────────────────────────────┐   │
│  │  STORAGE LAYER                                    │   │
│  │  • Micro-partitions (50-500 MB compressed)        │   │
│  │  • Automatic clustering                           │   │
│  │  • Time Travel (1-90 days)                        │   │
│  │  • Fail-safe (7 days after Time Travel)           │   │
│  │  • Zero-copy cloning                              │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Key Snowflake Features

| Feature | Benefit |
|---------|---------|
| Virtual Warehouses | Isolated compute per workload (ETL vs BI vs ML) |
| Zero-copy cloning | Instant dev/test environments without data duplication |
| Time Travel | Query data as it was 1-90 days ago (UNDROP!) |
| Micro-partitions | Automatic, no manual partitioning needed |
| Data Sharing | Share live data across accounts without copying |
| Streams & Tasks | Native CDC and scheduling |
| Semi-structured | Native JSON/Avro/Parquet support (VARIANT type) |
| Multi-cloud | Same product on AWS, GCP, Azure |

### Snowflake Optimization

```sql
-- Size warehouses appropriately
ALTER WAREHOUSE analytics_wh SET 
    WAREHOUSE_SIZE = 'MEDIUM',
    AUTO_SUSPEND = 120,       -- Suspend after 2 min idle
    AUTO_RESUME = TRUE,
    MIN_CLUSTER_COUNT = 1,
    MAX_CLUSTER_COUNT = 3;    -- Auto-scale out for concurrency

-- Zero-copy clone (instant, free until changes are made)
CREATE DATABASE staging_db CLONE production_db;
CREATE TABLE test_orders CLONE production.orders;

-- Time Travel
SELECT * FROM orders AT(TIMESTAMP => '2024-03-01 09:00:00');
SELECT * FROM orders BEFORE(STATEMENT => 'query-id-here');
```

> **Critical Insight:** Snowflake's killer feature for cost management is workload isolation via virtual warehouses. Your ETL pipeline uses a large warehouse at 2 AM (auto-suspend after), while analysts use a medium warehouse during business hours. Neither workload impacts the other, and you pay per-second only when active.

---

## Redshift Architecture

### Core Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Redshift                               │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  LEADER NODE                                      │   │
│  │  • Query planning and optimization                │   │
│  │  • Coordinates compute nodes                      │   │
│  │  • Stores metadata and statistics                 │   │
│  │  • Returns results to client                      │   │
│  └──────────────────────────┬───────────────────────┘   │
│                             │                            │
│              ┌──────────────┼──────────────┐            │
│              ▼              ▼              ▼            │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │
│  │ Compute Node │ │ Compute Node │ │ Compute Node │   │
│  │   (Slices)   │ │   (Slices)   │ │   (Slices)   │   │
│  │  ┌───┬───┐  │ │  ┌───┬───┐  │ │  ┌───┬───┐  │   │
│  │  │ S │ S │  │ │  │ S │ S │  │ │  │ S │ S │  │   │
│  │  └───┴───┘  │ │  └───┴───┘  │ │  └───┴───┘  │   │
│  └──────────────┘ └──────────────┘ └──────────────┘   │
│                                                          │
│  Key Concepts:                                           │
│  • Distribution Key: determines which node holds a row   │
│  • Sort Key: determines physical ordering on disk        │
│  • AQUA: hardware-accelerated cache layer               │
│  • Redshift Spectrum: query S3 directly                  │
│  • Serverless: auto-scaling alternative                  │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Distribution Strategies

| Strategy | Keyword | When to Use |
|----------|---------|-------------|
| KEY | `DISTKEY(col)` | Large tables frequently joined on this column |
| EVEN | `DISTSTYLE EVEN` | No clear join key; want balanced distribution |
| ALL | `DISTSTYLE ALL` | Small dimension tables (< 3M rows); replicated to all nodes |
| AUTO | `DISTSTYLE AUTO` | Let Redshift choose (default in Serverless) |

### Sort Key Strategies

```sql
-- Compound sort key (optimizes queries that filter on prefix columns)
CREATE TABLE fact_sales (
    sale_date       DATE,
    region          VARCHAR(20),
    product_id      INT,
    revenue         DECIMAL(12,2)
)
COMPOUND SORTKEY (sale_date, region);
-- Fast: WHERE sale_date = '2024-01-01' AND region = 'US'
-- Slow: WHERE region = 'US' (must use leftmost prefix first)

-- Interleaved sort key (optimizes any combination of sort columns)
CREATE TABLE fact_sales (...)
INTERLEAVED SORTKEY (sale_date, region, product_id);
-- Equally fast regardless of which columns are filtered
-- Trade-off: VACUUM is much slower
```

---

## Cost Models Comparison

| Dimension | BigQuery | Snowflake | Redshift (Provisioned) | Redshift Serverless |
|-----------|----------|-----------|----------------------|---------------------|
| **Pricing model** | Per-byte scanned | Per-second compute | Per-hour cluster | Per-RPU-second |
| **Storage cost** | ~$0.02/GB/month | ~$0.023/GB/month | Included in node cost | ~$0.024/GB/month |
| **Compute cost** | $5/TB scanned (on-demand) | ~$2-4/credit (~$30-60/hr for Medium) | ~$0.25-13/hr per node | ~$0.375/RPU-hour |
| **Idle cost** | Zero | Zero (auto-suspend) | Full cluster cost | Zero |
| **Scaling** | Automatic | Manual/auto resize | Manual resize | Automatic |
| **Reservation option** | Flat-rate slots ($2000/mo for 100 slots) | Prepaid credits (discount) | Reserved instances (1-3yr) | N/A |
| **Best for** | Bursty workloads, pay-per-use | Concurrent workloads, multi-team | Predictable, always-on workloads | Variable workloads on AWS |

> **Critical Insight:** The cost model should match your usage pattern. If you run queries 24/7 with predictable load, provisioned Redshift is cheapest. If your usage is bursty (heavy during business hours, idle at night), BigQuery on-demand or Snowflake with auto-suspend saves 60-80% over always-on clusters.

### Cost Optimization Strategies

| Platform | Strategy |
|----------|----------|
| BigQuery | Partition by date + clustering; use authorized views; set custom cost controls |
| Snowflake | Right-size warehouses; aggressive auto-suspend (60s); use resource monitors |
| Redshift | Choose correct node type; use Spectrum for cold data on S3; pause during off-hours |
| All | Archive old data to cold storage; build aggregation tables for repeated queries |

---

## Platform Comparison Table

| Feature | BigQuery | Snowflake | Redshift |
|---------|----------|-----------|----------|
| **Deployment** | GCP only | Multi-cloud (AWS, GCP, Azure) | AWS only |
| **Architecture** | Serverless | Shared-data, multi-cluster | Shared-nothing MPP |
| **Storage/Compute separation** | Full | Full | Partial (RA3 nodes + S3) |
| **Concurrency handling** | Slots auto-allocated | Multi-cluster warehouses | WLM queues |
| **Semi-structured data** | Native JSON, nested/repeated | VARIANT type | SUPER type (limited) |
| **Streaming ingestion** | Streaming inserts API | Snowpipe | Kinesis integration |
| **ML integration** | BQML (SQL-based) | Snowpark (Python/Scala) | Redshift ML (SageMaker) |
| **Data sharing** | Analytics Hub | Secure Data Sharing | Datashares |
| **Time travel** | 7 days (configurable) | 1-90 days | N/A (use snapshots) |
| **Ecosystem** | Looker, Dataform | dbt, Fivetran, Sigma | dbt, AQUA, Spectrum |
| **Best for** | GCP shops, serverless fans | Multi-cloud, diverse workloads | AWS-native, predictable load |

---

## Interview Talking Points

> "I architect our warehouse in three distinct layers: raw, transform, and marts. The raw layer is a faithful copy of source systems — we never modify it. The transform layer applies business logic using dbt models (deduplication, type casting, joins, business rules). The marts layer serves specific teams with pre-aggregated, governed data. This separation means we can always re-derive any downstream table from raw data."

> "We chose Snowflake because we needed workload isolation without infrastructure complexity. Our ETL runs on a dedicated X-Large warehouse at 2 AM that auto-suspends after 2 minutes of inactivity. Analysts use a shared Medium warehouse during the day. Data science has their own Large warehouse for ML training. None of these workloads interfere with each other, and we only pay when compute is active."

> "The biggest cost optimization win I implemented was adding partition pruning to our BigQuery fact tables. Before partitioning, analysts were scanning 2 TB per query (costing $10/query). After partitioning by date and clustering by customer_segment, the same queries scan 50 GB ($0.25/query) — a 40x cost reduction with zero change to query logic."

> "I always advocate for the ELT pattern over ETL in modern warehouses. Loading raw data first means we never lose information during transformation. When business logic changes — and it always does — we re-run dbt models against the raw layer to produce updated mart tables. If we had transformed before loading, we'd need to re-extract from source systems."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Single layer (raw data directly exposed to BI tools) | Three layers: raw -> transform -> marts |
| Modifying raw layer tables | Raw is immutable; transformations create new tables |
| No partitioning strategy (full table scans) | Partition by date; cluster by common filter columns |
| One giant warehouse/cluster for all workloads | Separate compute for ETL, analytics, ML (Snowflake/BQ) |
| Not monitoring query costs | Set up budget alerts, cost dashboards, query governance |
| Over-provisioned always-on clusters | Use auto-suspend, serverless, or right-size reservations |
| Skipping data quality tests in transform layer | Add dbt tests: unique, not_null, relationships, custom |
| No documentation of mart table semantics | Use dbt docs, data catalogs, or column descriptions |
| Loading all historical data in every batch | Incremental models: only process new/changed records |
| Ignoring distribution/sort keys in Redshift | Profile query patterns; set DISTKEY on join columns |

---

## Rapid-Fire Q&A

**Q1: What's the difference between a data warehouse and a data mart?**
A warehouse is the enterprise-wide integrated store. A mart is a subset focused on one domain (marketing, finance). Marts are derived from the warehouse, serving specific team needs with pre-aggregated data.

**Q2: What's zero-copy cloning in Snowflake?**
Creating a logical copy of a database/table that shares the same underlying storage. No data is physically copied until modifications are made (copy-on-write). Makes instant dev/test environments free until they diverge.

**Q3: How does BigQuery pricing work for on-demand vs flat-rate?**
On-demand: pay $5 per TB scanned per query. Flat-rate: buy reserved slots (compute units) for a fixed monthly fee (~$2000/100 slots). Flat-rate wins when monthly scanning exceeds 400 TB.

**Q4: What's a Redshift distribution key mistake that kills performance?**
Choosing a low-cardinality column (like status or country) as DISTKEY causes data skew — one node gets most of the rows while others sit idle. Always choose high-cardinality columns used in joins.

**Q5: What's the difference between Snowflake Time Travel and Fail-Safe?**
Time Travel: user-accessible, query historical data, undrop objects (1-90 days configurable). Fail-safe: Snowflake-internal disaster recovery (additional 7 days after Time Travel expires). You can't query Fail-safe data yourself.

**Q6: How do you handle late-arriving data in the warehouse?**
Use incremental models that process based on _loaded_at (arrival time), not event_time. Partition by event_date but allow backfill. Set up reconciliation checks comparing source counts to warehouse counts.

**Q7: What's Redshift Spectrum?**
Enables Redshift to query data directly in S3 (Parquet, CSV, JSON) without loading it. Useful for rarely-queried historical data — keep hot data in Redshift, cold data in S3, query both with same SQL.

**Q8: When should you use materialized views vs tables?**
Materialized views: auto-refreshed, query optimizer aware, limited SQL support. Tables: full SQL flexibility, manual refresh scheduling, more control. Use mat views for simple aggregations; tables for complex transforms.

**Q9: What's the WLM in Redshift?**
Workload Management — assigns queries to queues with different memory/concurrency allocations. Priority queues ensure critical dashboards get resources even when long-running queries are active.

**Q10: How do you migrate between warehouse platforms?**
Phase approach: (1) replicate data to target, (2) port transformations (dbt makes this easier — just change adapter), (3) dual-run and compare results, (4) migrate consumers, (5) decommission old system.

---

## ASCII Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────┐
│            DATA WAREHOUSE ARCHITECTURE GUIDE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  THE THREE LAYERS                                                │
│  ════════════════                                                │
│                                                                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │    RAW      │    │  TRANSFORM  │    │    MARTS    │         │
│  │  (Bronze)   │───▶│  (Silver)   │───▶│   (Gold)    │         │
│  │             │    │             │    │             │         │
│  │ • Immutable │    │ • Cleaned   │    │ • Aggregated│         │
│  │ • Source    │    │ • Typed     │    │ • Domain    │         │
│  │   faithful  │    │ • Joined    │    │   specific  │         │
│  │ • Append    │    │ • Tested    │    │ • Star      │         │
│  │   only      │    │ • Idempotent│    │   schema    │         │
│  └─────────────┘    └─────────────┘    └─────────────┘         │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  PLATFORM DECISION MATRIX                                        │
│  ════════════════════════                                        │
│                                                                  │
│  "Which warehouse?"                                              │
│                                                                  │
│  Already on GCP? ──────────────────── BigQuery                   │
│  Already on AWS? (predictable load) ─ Redshift                   │
│  Multi-cloud or variable load? ────── Snowflake                  │
│  Real-time + open source? ──────────── ClickHouse                │
│  Local/embedded analytics? ─────────── DuckDB                    │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  COST MODEL MATCH                                                │
│  ════════════════                                                │
│                                                                  │
│  Usage Pattern          │ Best Cost Model                        │
│  ───────────────────────┼────────────────────                    │
│  Bursty (9-5 analysts)  │ BigQuery on-demand / Snowflake auto    │
│  Constant 24/7          │ Redshift reserved / BQ flat-rate       │
│  Unpredictable spikes   │ Snowflake multi-cluster / BQ on-demand │
│  Dev/test environments  │ Snowflake zero-copy clone              │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  SNOWFLAKE: VIRTUAL WAREHOUSES                                   │
│  ═════════════════════════════                                   │
│                                                                  │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐                      │
│  │ ETL_WH  │   │BI_WH    │   │ ML_WH   │  ← Isolated compute  │
│  │ XL      │   │ Medium  │   │ Large   │                       │
│  │ 2AM-4AM │   │ 9AM-6PM │   │ On-demand│                      │
│  └────┬────┘   └────┬────┘   └────┬────┘                      │
│       │              │              │                            │
│       └──────────────┼──────────────┘                            │
│                      ▼                                           │
│              ┌───────────────┐                                   │
│              │ SHARED STORAGE │  ← One copy of data              │
│              │ (micro-parts)  │                                   │
│              └───────────────┘                                   │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  REDSHIFT: DISTRIBUTION + SORT                                   │
│  ═════════════════════════════                                   │
│                                                                  │
│  DISTKEY: Which NODE holds the row?                              │
│    • KEY(customer_id) → co-locate for JOINs                     │
│    • EVEN → balanced, no join optimization                       │
│    • ALL → small dims replicated everywhere                      │
│                                                                  │
│  SORTKEY: Physical ORDER on disk (within node)                   │
│    • COMPOUND(date, region) → prefix filtering                   │
│    • INTERLEAVED(date, region, product) → any-column filtering  │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  KEY PRINCIPLES                                                  │
│  ══════════════                                                  │
│                                                                  │
│  1. Layer your data: Raw → Transform → Marts                    │
│  2. Raw is IMMUTABLE — never modify source copies               │
│  3. Transforms must be IDEMPOTENT — safe to re-run              │
│  4. Separate compute from storage (modern architectures)         │
│  5. Match cost model to usage pattern                            │
│  6. Partition + cluster = cost control in cloud DWs             │
│  7. Isolate workloads (ETL vs BI vs ML)                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 13 of 45*
