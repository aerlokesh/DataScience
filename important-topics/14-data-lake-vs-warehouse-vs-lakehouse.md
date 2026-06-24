# 🎯 Topic 14: Data Lake vs Data Warehouse vs Data Lakehouse

> *Three paradigms for storing and processing analytical data — each born from the limitations of its predecessor. Understanding when to use which (and how the lakehouse unifies them) is essential for modern data architecture decisions.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [Data Lake](#data-lake)
3. [Data Warehouse](#data-warehouse)
4. [Data Lakehouse](#data-lakehouse)
5. [Comprehensive Comparison Table](#comprehensive-comparison-table)
6. [The Medallion Architecture](#the-medallion-architecture)
7. [Lakehouse Table Formats](#lakehouse-table-formats)
8. [When to Use Which](#when-to-use-which)
9. [Architecture Patterns in Practice](#architecture-patterns-in-practice)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Introduction

The evolution of analytical data storage follows a clear trajectory:

1. **Data Warehouses (1990s-2010s):** Structured, schema-on-write, expensive storage, fast queries
2. **Data Lakes (2010s):** Any format, schema-on-read, cheap storage, slow/complex queries
3. **Data Lakehouses (2020s):** Best of both — cheap storage with warehouse-like performance and governance

Each paradigm emerged because the previous one couldn't handle evolving data needs:
- Warehouses couldn't handle unstructured data or scale economically
- Lakes became ungoverned "data swamps" with unreliable quality
- Lakehouses bring structure and performance to lake-scale storage

> **Critical Insight:** The lakehouse isn't just a marketing term — it's a technical architecture enabled by open table formats (Delta Lake, Iceberg, Hudi) that add ACID transactions, time travel, and schema enforcement to files sitting in object storage. This eliminates the fundamental reason organizations needed two separate systems.

---

## Data Lake

### Definition

A data lake is a centralized repository that stores data in its **raw, native format** — structured, semi-structured, and unstructured — at any scale, using cheap object storage.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         DATA LAKE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  STORAGE: Object Storage (S3, GCS, ADLS)                         │
│  ═════════════════════════════════════════                        │
│                                                                  │
│  s3://data-lake/                                                 │
│  ├── raw/                                                        │
│  │   ├── clickstream/        (JSON, 500GB/day)                   │
│  │   ├── transactions/       (CSV from OLTP exports)             │
│  │   ├── logs/               (Unstructured text)                 │
│  │   ├── images/             (Binary files)                      │
│  │   └── iot-sensors/        (Avro from Kafka)                   │
│  ├── processed/                                                  │
│  │   ├── clickstream_parquet/ (Converted to columnar)            │
│  │   └── aggregated/          (Pre-computed metrics)             │
│  └── curated/                                                    │
│      └── feature_store/       (ML-ready features)                │
│                                                                  │
│  COMPUTE: Separate engines query the storage                     │
│  ═══════════════════════════════════════════                      │
│  • Spark (batch processing)                                      │
│  • Presto/Trino (interactive SQL)                                │
│  • Athena (serverless SQL on S3)                                 │
│  • EMR (managed Hadoop/Spark)                                    │
│                                                                  │
│  CATALOG: External metadata management                           │
│  ═════════════════════════════════════                            │
│  • AWS Glue Catalog                                              │
│  • Hive Metastore                                                │
│  • Unity Catalog (Databricks)                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Characteristics

| Property | Description |
|----------|-------------|
| **Schema** | Schema-on-read (interpret data at query time) |
| **Storage format** | Any: JSON, CSV, Parquet, Avro, ORC, images, video |
| **Storage cost** | Very low (~$0.023/GB/month for S3 Standard) |
| **Data types** | Structured + semi-structured + unstructured |
| **Processing** | External engines (Spark, Presto, Athena) |
| **ACID transactions** | Not natively (files are just files) |
| **Governance** | External tools needed (catalogs, access policies) |
| **Query performance** | Variable; depends on format and partitioning |
| **Primary users** | Data engineers, data scientists, ML engineers |
| **Scalability** | Virtually unlimited (object storage is infinite) |

### Strengths

- Store ANY data type at massive scale for pennies per GB
- Decouple storage from compute (pay for each independently)
- Multiple processing engines can read the same data
- Preserve raw data for future, unforeseen use cases
- Excellent for ML training data (images, text, embeddings)

### Weaknesses

- No transaction support (concurrent writes can corrupt data)
- No schema enforcement (data quality issues accumulate)
- "Data swamp" risk — without governance, data becomes unusable
- Query performance unpredictable without careful partitioning
- No UPDATE/DELETE — must rewrite entire partitions
- Consistency challenges (eventual consistency in S3)

> **Critical Insight:** The data lake's greatest strength (accept anything) is also its greatest weakness. Without active governance — data catalogs, quality checks, access controls, retention policies — a lake degrades into a swamp within 6-12 months. The technology is easy; the discipline is hard.

---

## Data Warehouse

### Definition

A data warehouse is an **optimized analytical database** with schema-on-write, ACID transactions, and purpose-built query engines for structured, governed data.

### Key Characteristics

| Property | Description |
|----------|-------------|
| **Schema** | Schema-on-write (define before loading) |
| **Storage format** | Proprietary columnar (optimized by vendor) |
| **Storage cost** | Higher (~$23-40/TB/month depending on vendor) |
| **Data types** | Structured primarily (some semi-structured support) |
| **Processing** | Integrated query engine (optimized, managed) |
| **ACID transactions** | Full support |
| **Governance** | Built-in (roles, row/column security, masking) |
| **Query performance** | Consistently fast (optimized engine + format) |
| **Primary users** | Analysts, BI developers, business users |
| **Scalability** | TB to low PB (vendor-dependent) |

### Strengths

- Blazing fast queries on structured data (seconds, not minutes)
- Full ACID transactions (reliable concurrent access)
- Built-in governance (access control, audit logging)
- Mature BI tool integration (Tableau, Looker, Power BI)
- Schema enforcement prevents bad data from entering
- SQL-first interface accessible to non-engineers

### Weaknesses

- Expensive storage for large datasets ($$$)
- Cannot efficiently store unstructured data (images, PDFs, logs)
- Schema-on-write means upfront design work (less flexible)
- Vendor lock-in (proprietary formats and engines)
- ETL required before data is queryable (latency)
- Scaling compute and storage together (older architectures)

### Modern Warehouses (BigQuery, Snowflake, Redshift)

Modern cloud warehouses have addressed SOME traditional limitations:
- Separated compute/storage (no longer coupled)
- Semi-structured data support (JSON, VARIANT)
- Near-real-time ingestion (streaming inserts)
- External table support (query S3/GCS data)
- But still primarily optimized for structured, governed analytics

---

## Data Lakehouse

### Definition

A data lakehouse combines the **low-cost, scalable storage of a data lake** with the **reliability, performance, and governance of a data warehouse**. It's enabled by open table formats that add database-like capabilities to file-based storage.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        DATA LAKEHOUSE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  QUERY ENGINES (Multiple can read same data)              │   │
│  │  Spark SQL │ Trino │ Databricks SQL │ Snowflake │ Athena  │   │
│  └─────────────────────────┬────────────────────────────────┘   │
│                            │                                     │
│  ┌─────────────────────────┼────────────────────────────────┐   │
│  │  TABLE FORMAT LAYER     ▼                                 │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Delta Lake / Apache Iceberg / Apache Hudi          │  │   │
│  │  │                                                      │  │   │
│  │  │  Provides:                                           │  │   │
│  │  │  • ACID transactions (atomic writes)                 │  │   │
│  │  │  • Schema enforcement + evolution                    │  │   │
│  │  │  • Time travel (version history)                     │  │   │
│  │  │  • UPDATE / DELETE / MERGE support                   │  │   │
│  │  │  • Partition evolution                               │  │   │
│  │  │  • File-level statistics (for pruning)               │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            │                                     │
│  ┌─────────────────────────┼────────────────────────────────┐   │
│  │  STORAGE LAYER          ▼                                 │   │
│  │  Object Storage (S3, GCS, ADLS)                           │   │
│  │  • Parquet files (columnar, compressed)                   │   │
│  │  • Metadata files (JSON/Avro manifests)                   │   │
│  │  • Transaction logs                                        │   │
│  │  • $0.023/GB/month (lake pricing!)                        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Characteristics

| Property | Description |
|----------|-------------|
| **Schema** | Schema-on-write WITH evolution support |
| **Storage format** | Open formats (Parquet + metadata layer) |
| **Storage cost** | Low (object storage pricing) |
| **Data types** | All types (structured primary, unstructured supported) |
| **Processing** | Multi-engine (Spark, Trino, vendor engines) |
| **ACID transactions** | Yes (via table format layer) |
| **Governance** | Catalogs + table format (Unity Catalog, etc.) |
| **Query performance** | Fast (statistics, caching, optimized reads) |
| **Primary users** | Everyone: engineers, analysts, scientists |
| **Scalability** | Unlimited (object storage) |

### What Table Formats Add to Object Storage

```
Without table format (plain data lake):
  s3://bucket/data/part-00001.parquet
  s3://bucket/data/part-00002.parquet
  → No transactions, no schema, no versioning, no UPDATE

With table format (lakehouse):
  s3://bucket/data/part-00001.parquet
  s3://bucket/data/part-00002.parquet
  s3://bucket/data/_delta_log/        ← Transaction log
  s3://bucket/data/_delta_log/00000.json  ← Commit metadata
  → ACID, schema, time travel, UPDATE/DELETE, statistics
```

> **Critical Insight:** The lakehouse doesn't replace the lake's storage — it ADDS a metadata layer on top. Your data still lives in S3/GCS as Parquet files. The table format (Delta/Iceberg/Hudi) is just metadata files that track which Parquet files are part of the "current" table state, enabling transactions and versioning.

---

## Comprehensive Comparison Table

| Dimension | Data Lake | Data Warehouse | Data Lakehouse |
|-----------|-----------|----------------|----------------|
| **Schema** | Schema-on-read | Schema-on-write | Schema-on-write + evolution |
| **Storage** | Object store (S3/GCS) | Proprietary columnar | Object store + table format |
| **Cost (storage)** | $0.023/GB/month | $23-40/TB/month | $0.023/GB/month |
| **Cost (compute)** | Separate engines | Integrated | Flexible (multi-engine) |
| **Data types** | Any (structured to unstructured) | Structured only | Primarily structured + semi |
| **ACID transactions** | No | Yes | Yes |
| **UPDATE/DELETE** | Rewrite partition | Native | Native (via merge-on-read/CoW) |
| **Time travel** | Manual (keep old files) | Platform-specific | Built-in (versioned commits) |
| **Query performance** | Variable, often slow | Consistently fast | Fast (with optimization) |
| **Governance** | External (catalog + IAM) | Built-in (RBAC, masking) | Improving (Unity Catalog etc.) |
| **BI tool support** | Limited (need SQL engine) | Excellent (native) | Good and improving |
| **ML/AI workloads** | Excellent (native Spark/Python) | Limited | Excellent |
| **Vendor lock-in** | Low (open formats) | High (proprietary) | Low (open table formats) |
| **Maturity** | Mature (10+ years) | Very mature (30+ years) | Emerging (3-5 years) |
| **Best for** | Raw storage, ML, unstructured | BI, reporting, governed analytics | Unified analytics + ML |
| **Risk** | Data swamp | Cost at scale | Complexity, ecosystem maturity |

---

## The Medallion Architecture

The medallion architecture (bronze/silver/gold) is the standard layering pattern for lakehouses, analogous to raw/staging/marts in warehouses.

### Bronze Layer (Raw)

```
Purpose: Land data exactly as received from sources
Properties:
  • Append-only (never modify)
  • Schema matches source
  • May contain duplicates
  • Full history preserved
  • Partitioned by ingestion date
```

```sql
-- Bronze: raw ingestion from source
CREATE TABLE bronze.raw_orders (
    -- Source columns
    order_id        STRING,
    customer_id     STRING,
    order_date      STRING,     -- Keep as string (don't cast yet)
    total           STRING,     -- Keep as string
    status          STRING,
    items_json      STRING,     -- Raw JSON blob
    
    -- Ingestion metadata
    _ingested_at    TIMESTAMP,
    _source_file    STRING,
    _batch_id       STRING
)
USING DELTA
PARTITIONED BY (_ingested_at::DATE);
```

### Silver Layer (Cleaned/Conformed)

```
Purpose: Clean, deduplicate, type-cast, and conform data
Properties:
  • Deduplicated
  • Correct data types
  • Business keys standardized
  • Quality validated
  • Conformed dimensions
  • Incremental processing
```

```sql
-- Silver: cleaned and typed
CREATE TABLE silver.orders (
    order_id        BIGINT,
    customer_id     BIGINT,
    order_date      DATE,
    total_amount    DECIMAL(12,2),
    status          STRING,     -- Standardized: 'active','cancelled','completed'
    items           ARRAY<STRUCT<
                        product_id: BIGINT,
                        quantity: INT,
                        price: DECIMAL(10,2)
                    >>,
    _processed_at   TIMESTAMP,
    _source_batch   STRING
)
USING DELTA
PARTITIONED BY (order_date);

-- Silver processing: MERGE for deduplication + type casting
MERGE INTO silver.orders AS target
USING (
    SELECT 
        CAST(order_id AS BIGINT) AS order_id,
        CAST(customer_id AS BIGINT) AS customer_id,
        CAST(order_date AS DATE) AS order_date,
        CAST(total AS DECIMAL(12,2)) AS total_amount,
        LOWER(TRIM(status)) AS status,
        FROM_JSON(items_json, schema) AS items,
        current_timestamp() AS _processed_at,
        _batch_id AS _source_batch
    FROM bronze.raw_orders
    WHERE _ingested_at > (SELECT MAX(_processed_at) FROM silver.orders)
) AS source
ON target.order_id = source.order_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

### Gold Layer (Business-Level Aggregates)

```
Purpose: Business-ready, aggregated, domain-specific tables
Properties:
  • Star/snowflake schema
  • Pre-aggregated metrics
  • Business-friendly naming
  • SLA-governed freshness
  • Optimized for BI consumption
```

```sql
-- Gold: business-ready mart
CREATE TABLE gold.daily_sales_summary (
    date_key            DATE,
    product_category    STRING,
    customer_segment    STRING,
    region              STRING,
    total_orders        BIGINT,
    total_revenue       DECIMAL(14,2),
    avg_order_value     DECIMAL(10,2),
    unique_customers    BIGINT,
    return_rate         DECIMAL(5,4)
)
USING DELTA
PARTITIONED BY (date_key);
```

### Medallion Flow Diagram

```
Sources          Bronze              Silver              Gold
━━━━━━━          ━━━━━━              ━━━━━━              ━━━━
                 
PostgreSQL ─┐    ┌──────────┐       ┌──────────┐       ┌──────────┐
            ├──▶ │ Raw copy  │ ───▶  │ Cleaned  │ ───▶  │ Revenue  │
Kafka      ─┤    │ As-is     │       │ Typed    │       │ Mart     │
            │    │ Append    │       │ Deduped  │       │          │
APIs       ─┤    └──────────┘       └──────────┘       └──────────┘
            │                                           
S3 files   ─┘    Immutable           Conformed          Aggregated
                 Schema-on-read      Schema enforced     Star schema
                 Raw types           Correct types       Business KPIs
```

> **Critical Insight:** The medallion architecture's power is that each layer has a clear contract. Bronze accepts anything (resilient ingestion). Silver guarantees quality (tests, schema enforcement). Gold guarantees business relevance (documented metrics, SLAs). A failure in silver doesn't corrupt bronze; you can always re-derive.

---

## Lakehouse Table Formats

### Delta Lake (Databricks)

| Feature | Details |
|---------|---------|
| Creator | Databricks (open-sourced) |
| Transaction log | JSON files in `_delta_log/` directory |
| Write pattern | Copy-on-write (default) or merge-on-read |
| Time travel | Version numbers or timestamps |
| Ecosystem | Best with Spark/Databricks; growing multi-engine support |
| Key command | `OPTIMIZE` (compaction), `VACUUM` (cleanup), `Z-ORDER` (clustering) |

```sql
-- Delta Lake operations
-- Time travel
SELECT * FROM orders VERSION AS OF 5;
SELECT * FROM orders TIMESTAMP AS OF '2024-01-15';

-- MERGE (upsert)
MERGE INTO silver.customers AS target
USING updates AS source
ON target.customer_id = source.customer_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;

-- Optimize file sizes
OPTIMIZE silver.orders ZORDER BY (customer_id);
```

### Apache Iceberg (Netflix/Apple/AWS)

| Feature | Details |
|---------|---------|
| Creator | Netflix (now Apache project) |
| Transaction log | Manifest files (Avro) + metadata (JSON) |
| Write pattern | Copy-on-write or merge-on-read (configurable per table) |
| Time travel | Snapshot IDs or timestamps |
| Ecosystem | Strongest multi-engine: Spark, Trino, Flink, Athena, Snowflake |
| Key feature | Partition evolution (change partitioning without rewriting) |

```sql
-- Iceberg operations
-- Partition evolution (no data rewrite!)
ALTER TABLE orders SET PARTITION SPEC (
    PARTITION BY days(order_date), bucket(16, customer_id)
);

-- Time travel
SELECT * FROM orders FOR SYSTEM_TIME AS OF '2024-01-15 10:00:00';

-- Expire old snapshots (cleanup)
CALL system.expire_snapshots('db.orders', TIMESTAMP '2024-01-01');
```

### Apache Hudi (Uber)

| Feature | Details |
|---------|---------|
| Creator | Uber (now Apache project) |
| Transaction log | Timeline (commit instants) |
| Write pattern | Copy-on-write OR merge-on-read (table-level choice) |
| Time travel | Instant timestamps |
| Ecosystem | Strong Spark support; good for streaming upserts |
| Key feature | Incremental queries (only read new/changed data since last read) |

### Table Format Comparison

| Aspect | Delta Lake | Apache Iceberg | Apache Hudi |
|--------|-----------|----------------|-------------|
| Best engine support | Spark/Databricks | Multi-engine (broadest) | Spark |
| Partition evolution | Limited | Excellent (hidden partitioning) | Limited |
| Schema evolution | Good | Excellent | Good |
| Streaming support | Structured Streaming | Flink integration | Native (upsert-heavy) |
| Adoption momentum | Databricks shops | Cross-platform/cloud | Uber + AWS-focused |
| Cloud vendor support | Databricks, Azure | AWS (Athena/EMR), Snowflake | AWS (EMR, Glue) |
| Open governance | Delta Sharing | REST Catalog | Timeline server |

---

## When to Use Which

### Decision Framework

```
START HERE
    │
    ▼
Do you have unstructured data (images, video, PDFs)?
    │
    ├── YES ──▶ You need at MINIMUM a data lake for storage
    │           (lakehouse can serve structured portion)
    │
    └── NO
        │
        ▼
    Is your primary use case BI/reporting with governed metrics?
        │
        ├── YES, and data < 10 TB ──▶ Data Warehouse (simplest)
        │
        └── NO, or data > 10 TB
            │
            ▼
        Do you need both ML/AI AND BI on the same data?
            │
            ├── YES ──▶ Data Lakehouse (unified platform)
            │
            └── NO
                │
                ▼
            Is cost the primary concern (PB-scale data)?
                │
                ├── YES ──▶ Data Lake + query engine (Athena/Trino)
                │
                └── NO ──▶ Warehouse for BI, Lake for ML (two-system)
```

### Use Case Mapping

| Use Case | Best Architecture | Why |
|----------|------------------|-----|
| Executive dashboards, KPI reporting | Warehouse | Governance, speed, BI integration |
| ML training on large datasets | Lake or Lakehouse | Cost, flexible formats, Spark native |
| Real-time analytics on streaming data | Lakehouse | ACID + streaming ingestion |
| Compliance/audit (full history) | Lakehouse | Time travel, immutable bronze layer |
| Small team, < 5 TB, BI-focused | Warehouse | Simplest, fastest to value |
| Multi-team, mixed workloads | Lakehouse | One platform for all personas |
| IoT/sensor data at PB scale | Lake | Cheapest storage, schema flexibility |
| Ad-hoc exploration by data scientists | Lake or Lakehouse | Schema-on-read flexibility |

---

## Architecture Patterns in Practice

### Pattern 1: Warehouse-Centric (Traditional)

```
Sources → ETL → Data Warehouse → BI Tools
                      ↓
               (Export to S3 for ML — awkward)
```
Best for: Small-medium orgs, BI-focused, < 10 TB.

### Pattern 2: Two-Tier (Lake + Warehouse)

```
Sources → Data Lake (S3) → ETL → Data Warehouse → BI Tools
               ↓
          ML/Spark jobs (data science)
```
Best for: Orgs with both BI and ML needs, budget for two systems.

### Pattern 3: Lakehouse-Unified

```
Sources → Data Lakehouse (Delta/Iceberg on S3)
               ↓                    ↓
          ML (Spark/Python)    BI (SQL engine/Databricks SQL)
```
Best for: Modern orgs wanting one platform, strong engineering team.

### Pattern 4: Hybrid (Lakehouse + Warehouse)

```
Sources → Data Lake (bronze) → Lakehouse (silver) → Warehouse (gold/marts)
               ↓                      ↓                     ↓
          Raw archive           ML training            BI/Reporting
```
Best for: Enterprises with existing warehouse investments; gradual migration.

---

## Interview Talking Points

> "I recommend the lakehouse architecture for most greenfield projects today because it eliminates the two-system problem. Previously, we maintained a data lake for our ML engineers AND a Snowflake warehouse for our analysts — same data copied into two systems with different freshness guarantees. With Iceberg tables on S3, both teams query the same governed, transactional data using their preferred engine."

> "The medallion architecture (bronze/silver/gold) maps perfectly to our data quality journey. Bronze is our safety net — we never lose raw data. Silver is where we invest in quality: deduplication, schema enforcement, dbt tests. Gold is what the business trusts for decisions. When a stakeholder questions a number, I can trace it from gold through silver back to the exact bronze record."

> "We chose Apache Iceberg over Delta Lake because we needed multi-engine compatibility. Our data engineers use Spark for heavy ETL, our analysts use Trino for interactive queries, and our streaming team uses Flink. Iceberg's engine-agnostic design means all three teams work on the same tables without format conversion. Delta Lake's tighter Spark coupling would have forced our Trino users into a separate copy."

> "The cost argument for lakehouses is compelling. Our 50 TB analytical dataset in Snowflake cost us approximately $1,150/month in storage alone. The same data in S3 with Iceberg format costs $1,150/month... for the ENTIRE year. That's a 12x storage cost reduction. We still use Snowflake for our gold-layer marts where query performance matters most, but the bulk of our data lives in the lakehouse."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Dumping everything into a lake with no governance | Implement catalogs, quality checks, and access controls from day one |
| Using a warehouse for ML training data (expensive) | Store training data in lake/lakehouse; only put serving data in warehouse |
| Choosing lakehouse without engineering capability | Lakehouses require more DIY; consider managed warehouse if team is small |
| Treating lake as "just S3 with folders" | Use table formats (Iceberg/Delta) even on the lake — get ACID for free |
| Copying data between lake and warehouse constantly | Use external tables or lakehouse to avoid dual copies |
| Ignoring file size optimization in lakehouses | Run OPTIMIZE/compaction regularly; small files kill query performance |
| No retention policy on lake data | Set lifecycle rules; archive to Glacier/Coldline after retention period |
| Choosing table format based on hype, not ecosystem | Pick based on your engine(s): Delta for Databricks, Iceberg for multi-engine |
| Skipping the silver layer ("bronze straight to gold") | Silver quality enforcement prevents bad data from reaching business users |
| Building lakehouse without a catalog | Without a catalog (Unity, Glue, Polaris), you've just built another swamp |

---

## Rapid-Fire Q&A

**Q1: What's the difference between schema-on-read and schema-on-write?**
Schema-on-write: define structure before loading (warehouse). Schema-on-read: store raw, interpret structure at query time (lake). Lakehouse uses schema-on-write with evolution support — enforced but flexible.

**Q2: Can you UPDATE a record in a data lake?**
Not natively — files are immutable in object storage. You must rewrite the entire partition containing that record. Table formats (Delta/Iceberg) abstract this by tracking which files are "current" via metadata, making UPDATE appear seamless.

**Q3: What's the "data swamp" problem?**
When a lake accumulates data without governance — no documentation, no quality checks, no ownership, no access controls. Data exists but nobody trusts or can find it. Prevention: catalog everything, enforce naming conventions, assign owners.

**Q4: How does time travel work in a lakehouse?**
The table format maintains a log of all commits (file additions/removals). To read a past version, the engine reads the metadata for that version's commit, which references the Parquet files that existed at that point. Old files are retained until explicitly vacuumed.

**Q5: What makes Apache Iceberg's partition evolution special?**
You can change how a table is partitioned (e.g., from daily to hourly, or add a new partition column) WITHOUT rewriting existing data. The metadata layer handles the transition transparently across old and new partition schemes.

**Q6: Is Databricks a data lakehouse?**
Databricks is a lakehouse PLATFORM built on Delta Lake. The lakehouse is the architecture pattern; Databricks is one implementation. You can also build a lakehouse with Iceberg + Trino + S3 without Databricks.

**Q7: Can Snowflake read Iceberg tables?**
Yes — Snowflake supports external Iceberg tables, querying data that lives in your S3/GCS with Iceberg format. This enables a hybrid: lakehouse storage with Snowflake query engine for gold-layer tables.

**Q8: What's the difference between Delta Lake and Delta Sharing?**
Delta Lake is the table format (ACID on files). Delta Sharing is a protocol for sharing data between organizations without copying — live, governed read access to Delta tables across accounts.

**Q9: How do you handle streaming data in a lakehouse?**
Use structured streaming (Spark) or Flink to write micro-batches to Delta/Iceberg tables. The table format's ACID properties ensure readers see consistent snapshots even while writers are appending. Bronze layer handles the raw stream.

**Q10: What's the total cost of ownership comparison?**
At 100 TB: Lake (~$2.3K/mo storage + compute) < Lakehouse (~$5K/mo with optimization) < Warehouse (~$50K/mo with queries). But factor in engineering time: Warehouse needs least, Lake needs most, Lakehouse is middle.

---

## ASCII Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────┐
│         DATA LAKE vs WAREHOUSE vs LAKEHOUSE GUIDE                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  EVOLUTION                                                       │
│  ═════════                                                       │
│                                                                  │
│  1990s─────────▶ 2010s─────────▶ 2020s                          │
│  WAREHOUSE        DATA LAKE        LAKEHOUSE                     │
│  Structured       Anything          Structured +                 │
│  Expensive        Cheap             Cheap                        │
│  Governed         Ungoverned        Governed                     │
│  Fast queries     Slow queries      Fast queries                 │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  ARCHITECTURE COMPARISON                                         │
│  ═══════════════════════                                         │
│                                                                  │
│  DATA LAKE:                                                      │
│  ┌──────────────────────────┐                                   │
│  │  S3/GCS (any format)     │ ← Cheap, no structure             │
│  │  [JSON][CSV][Parquet]    │                                    │
│  │  [images][logs][avro]    │                                    │
│  └──────────────────────────┘                                   │
│         ↑ No ACID, no schema, no governance                      │
│                                                                  │
│  DATA WAREHOUSE:                                                 │
│  ┌──────────────────────────┐                                   │
│  │  Proprietary Engine      │ ← Fast, expensive, governed       │
│  │  ┌────────────────────┐  │                                   │
│  │  │ Columnar + indexes │  │                                   │
│  │  │ ACID + governance  │  │                                   │
│  │  └────────────────────┘  │                                   │
│  └──────────────────────────┘                                   │
│                                                                  │
│  LAKEHOUSE:                                                      │
│  ┌──────────────────────────┐                                   │
│  │  Query Engines (any)     │ ← Fast, cheap, governed           │
│  │  ┌────────────────────┐  │                                   │
│  │  │ Table Format Layer │  │  ← ACID + schema + versioning     │
│  │  │ (Delta/Iceberg)    │  │                                   │
│  │  └────────────────────┘  │                                   │
│  │  ┌────────────────────┐  │                                   │
│  │  │ Object Storage     │  │  ← Parquet files on S3/GCS       │
│  │  │ ($0.023/GB/month)  │  │                                   │
│  │  └────────────────────┘  │                                   │
│  └──────────────────────────┘                                   │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  MEDALLION ARCHITECTURE                                          │
│  ══════════════════════                                          │
│                                                                  │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐                   │
│  │ BRONZE  │────▶│ SILVER  │────▶│  GOLD   │                   │
│  │ (Raw)   │     │(Cleaned)│     │ (Marts) │                   │
│  └─────────┘     └─────────┘     └─────────┘                   │
│  • As-is from     • Deduped       • Star schema                  │
│    source         • Typed          • Pre-aggregated              │
│  • Append-only    • Validated      • Business KPIs               │
│  • All data       • Conformed      • SLA-governed                │
│  • Immutable      • Incremental    • BI-optimized                │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  TABLE FORMAT CHOICE                                             │
│  ═══════════════════                                             │
│                                                                  │
│  Using Databricks? ─────────────── Delta Lake                    │
│  Multi-engine (Spark+Trino+Flink)? Apache Iceberg               │
│  Heavy streaming upserts? ──────── Apache Hudi                   │
│  AWS-native? ───────────────────── Iceberg (Athena/EMR support) │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  DECISION QUICK-REFERENCE                                        │
│  ════════════════════════                                        │
│                                                                  │
│  Small team + BI focus + < 10TB → Warehouse                     │
│  ML-heavy + unstructured + PB scale → Lake                      │
│  Mixed workloads + cost conscious → Lakehouse                    │
│  Existing warehouse + adding ML → Hybrid (Lake + WH)            │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  COST AT SCALE (100 TB storage)                                  │
│  ══════════════════════════════                                  │
│                                                                  │
│  Lake (S3):        ~$2,300/month                                │
│  Lakehouse (S3+fmt): ~$2,300/month (same storage!)              │
│  Warehouse:        ~$2,300-4,000/month (separated arch)         │
│                    or $25,000+/month (older provisioned)         │
│                                                                  │
│  Compute costs depend on query volume — warehouse often          │
│  cheaper per-query but lake/lakehouse cheaper at scale            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 14 of 45*
