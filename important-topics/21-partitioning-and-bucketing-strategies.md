# 🎯 Topic 21: Partitioning and Bucketing Strategies

> *The physical organization of data determines query performance at scale — master partitioning, bucketing, clustering, sort keys, and Z-ordering to make terabyte-scale queries run in seconds*

---

## 📋 Table of Contents

1. [Core Concepts](#core-concepts)
2. [Table Partitioning](#table-partitioning)
3. [Partition Pruning](#partition-pruning)
4. [Over-Partitioning Pitfalls](#over-partitioning-pitfalls)
5. [Bucketing and Clustering](#bucketing-and-clustering)
6. [Sort Keys (Redshift)](#sort-keys-redshift)
7. [Z-Ordering](#z-ordering)
8. [Practical Guidelines](#practical-guidelines)
9. [Platform-Specific Strategies](#platform-specific-strategies)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Core Concepts

### Why Physical Data Organization Matters

At petabyte scale, reading all data for every query is impossibly expensive. Physical organization strategies enable **data skipping** — the ability to read only the relevant subset of data.

```
Without partitioning:  Scan 10 TB to answer "revenue for January 2024"
With date partitioning: Scan 300 GB (January partition only) → 33x faster
With clustering:        Scan 30 GB (relevant data blocks only) → 333x faster
```

> **Critical Insight:** Partitioning and clustering are not performance "optimizations" — at scale, they're requirements. A 10TB table without partitioning is essentially unusable for interactive queries. The choice of partition key is one of the most impactful decisions you'll make for a table's lifetime.

### Key Terminology

| Term | Definition |
|------|-----------|
| **Partition** | A physical division of a table based on column values |
| **Partition pruning** | Query engine skips partitions that don't match the filter |
| **Bucketing** | Distributing rows into a fixed number of buckets by hash |
| **Clustering** | Sorting data within storage blocks by chosen columns |
| **Sort key** | Column(s) that determine physical row ordering |
| **Z-ordering** | Multi-dimensional clustering that interleaves sort dimensions |
| **Data skipping** | Any technique that avoids reading irrelevant data |

---

## Table Partitioning

### Partitioning Types

| Type | How It Works | Example | Best For |
|------|-------------|---------|----------|
| **Range** | Rows assigned by value range | Date ranges: Jan, Feb, Mar... | Time-series data |
| **List** | Rows assigned by exact value | Country: US, UK, DE, JP... | Categorical dimensions |
| **Hash** | Rows assigned by hash(key) % N | customer_id % 256 | Even distribution |
| **Composite** | Multiple levels of partitioning | Year → Month → Day | Hierarchical access patterns |

### Date-Based Partitioning (Most Common)

```sql
-- BigQuery: Partition by date
CREATE TABLE analytics.fct_events
PARTITION BY DATE(event_timestamp)
CLUSTER BY user_id, event_type
AS SELECT * FROM staging.stg_events;

-- Snowflake: Automatic micro-partitioning (no explicit DDL needed)
-- Snowflake auto-partitions by ingestion order
-- Use CLUSTER BY to influence physical ordering:
ALTER TABLE analytics.fct_events CLUSTER BY (event_date, user_id);

-- Hive: Partition by date and country
CREATE TABLE events (
    event_id STRING,
    user_id STRING,
    event_type STRING,
    event_timestamp TIMESTAMP
)
PARTITIONED BY (event_date DATE, country STRING)
STORED AS PARQUET;

-- Spark: Write with partitioning
df.write \
    .partitionBy("event_date") \
    .mode("overwrite") \
    .parquet("s3://data-lake/events/")

# Creates directory structure:
# s3://data-lake/events/event_date=2024-01-01/
# s3://data-lake/events/event_date=2024-01-02/
# s3://data-lake/events/event_date=2024-01-03/
```

### Range Partitioning

```sql
-- PostgreSQL: Range partitioning
CREATE TABLE orders (
    order_id BIGINT,
    order_date DATE,
    amount DECIMAL(10,2)
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE orders_2024_q2 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- BigQuery: Range partitioning on integer
CREATE TABLE analytics.user_scores
PARTITION BY RANGE_BUCKET(score, GENERATE_ARRAY(0, 100, 10))
AS SELECT * FROM staging.stg_scores;
```

### List Partitioning

```sql
-- PostgreSQL: List partitioning
CREATE TABLE sales (
    sale_id BIGINT,
    region TEXT,
    amount DECIMAL(10,2)
) PARTITION BY LIST (region);

CREATE TABLE sales_na PARTITION OF sales FOR VALUES IN ('US', 'CA', 'MX');
CREATE TABLE sales_eu PARTITION OF sales FOR VALUES IN ('UK', 'DE', 'FR');
CREATE TABLE sales_apac PARTITION OF sales FOR VALUES IN ('JP', 'AU', 'IN');
```

### Hash Partitioning

```sql
-- PostgreSQL: Hash partitioning (even distribution)
CREATE TABLE user_events (
    user_id BIGINT,
    event_data JSONB
) PARTITION BY HASH (user_id);

CREATE TABLE user_events_0 PARTITION OF user_events
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE user_events_1 PARTITION OF user_events
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);
-- ... and so on for 2, 3
```

> **Critical Insight:** Date partitioning is the dominant strategy because most analytical queries filter by time ("last 7 days", "this month", "YoY comparison"). If your queries don't filter by the partition key, partitioning provides zero benefit — the engine still scans all partitions.

---

## Partition Pruning

### How It Works

```sql
-- Query with partition filter:
SELECT SUM(revenue)
FROM fct_daily_revenue
WHERE revenue_date BETWEEN '2024-01-01' AND '2024-01-31';

-- Without partitioning: Full table scan (365 days of data)
-- With date partitioning: Only reads January partition (31 days)
-- Pruning eliminates 334/365 = 91% of data scanned
```

### Pruning Rules

| Query Pattern | Pruning? | Explanation |
|---------------|----------|-------------|
| `WHERE date = '2024-01-15'` | Yes | Direct equality on partition key |
| `WHERE date BETWEEN ... AND ...` | Yes | Range filter on partition key |
| `WHERE date >= '2024-01-01'` | Yes | Open range on partition key |
| `WHERE YEAR(date) = 2024` | Depends | Some engines can optimize, some can't |
| `WHERE user_id = 123` | No | Not the partition key |
| `WHERE date = '2024-01-15' AND user_id = 123` | Yes (date only) | Prunes on date, scans within partition |

### Verifying Partition Pruning

```sql
-- BigQuery: Check bytes scanned
SELECT total_bytes_processed 
FROM `region-us`.INFORMATION_SCHEMA.JOBS
WHERE job_id = 'your-job-id';

-- Snowflake: Query profile shows partitions scanned vs total
-- Look for "Partitions scanned" vs "Partitions total"

-- Spark: Explain plan shows partition filters
df.filter("event_date = '2024-01-15'").explain(True)
-- Look for: PushedFilters: [IsNotNull(event_date), EqualTo(event_date, 2024-01-15)]
-- And: PartitionFilters: [isnotnull(event_date), (event_date = 2024-01-15)]

-- Athena / Presto: EXPLAIN shows partition pruning
EXPLAIN SELECT * FROM events WHERE event_date = '2024-01-15';
```

### Functions That Break Pruning

```sql
-- BREAKS pruning (function on partition column):
WHERE YEAR(event_date) = 2024 AND MONTH(event_date) = 1

-- ENABLES pruning (direct comparison):
WHERE event_date >= '2024-01-01' AND event_date < '2024-02-01'

-- BREAKS pruning (casting partition column):
WHERE CAST(event_date AS STRING) = '2024-01-15'

-- ENABLES pruning (native type comparison):
WHERE event_date = DATE '2024-01-15'
```

> **Critical Insight:** Always filter directly on the partition column without wrapping it in functions. `WHERE YEAR(dt) = 2024` often prevents pruning because the engine can't simplify the expression. Rewrite as `WHERE dt >= '2024-01-01' AND dt < '2025-01-01'` for guaranteed pruning.

---

## Over-Partitioning Pitfalls

### The Small Files Problem

```
Over-partitioned (1000s of tiny partitions):
s3://lake/events/year=2024/month=01/day=15/hour=00/minute=00/  → 50 KB
s3://lake/events/year=2024/month=01/day=15/hour=00/minute=01/  → 45 KB
s3://lake/events/year=2024/month=01/day=15/hour=00/minute=02/  → 52 KB
... (1440 partitions per day × 365 days = 525,600 partitions/year)

Problems:
1. Metadata overhead (catalog must track 500K+ partitions)
2. Tiny files (each < 1MB → poor compression, excessive I/O overhead)
3. Slow listing (S3/HDFS listing scales with partition count)
4. Query planning time exceeds query execution time
```

### Guidelines: When Is It Too Many?

| Partition Granularity | Partitions/Year | Typical File Size | Verdict |
|----------------------|-----------------|-------------------|---------|
| **Yearly** | 1 | Very large | Too coarse (no pruning benefit for daily queries) |
| **Monthly** | 12 | Large | Good for small-medium tables |
| **Daily** | 365 | Medium | Ideal for most event tables |
| **Hourly** | 8,760 | Small | Only if hourly queries are common AND high volume |
| **Per-minute** | 525,600 | Tiny | Almost always too fine |

### The 128MB Rule

Each partition should contain at least **128MB of compressed data** for efficient Parquet reads. If your daily volume for a partition is 10MB, use weekly or monthly partitioning instead.

```python
# Calculate: Is daily partitioning appropriate?
daily_data_volume_gb = 50  # 50 GB per day total
num_subpartitions = 1       # Just date partitioning

partition_size = daily_data_volume_gb / num_subpartitions
# 50 GB per partition → Well above 128 MB threshold ✓

# What about date + country (200 countries)?
partition_size = daily_data_volume_gb / 200
# 250 MB per partition → Marginal, some small countries will be tiny ✗
```

### Solutions for Over-Partitioning

| Problem | Solution |
|---------|----------|
| Too many small files | Compaction job (merge small files into larger ones) |
| Excessive partitions | Coarser granularity (hourly → daily) |
| Uneven partition sizes | Use clustering instead of partitioning for low-cardinality |
| Metadata overhead | Use Delta Lake / Iceberg (better metadata management) |

```python
# Spark: Compaction — merge small files
spark.read.parquet("s3://lake/events/dt=2024-01-15/") \
    .repartition(4) \  # Reduce to 4 files
    .write.mode("overwrite") \
    .parquet("s3://lake/events/dt=2024-01-15/")
```

---

## Bucketing and Clustering

### Bucketing (Hive/Spark)

Bucketing distributes rows into a **fixed number of files** based on a hash of the bucket column. This optimizes joins between tables bucketed on the same key.

```sql
-- Hive: Create bucketed table
CREATE TABLE orders (
    order_id BIGINT,
    customer_id BIGINT,
    amount DECIMAL(10,2),
    order_date DATE
)
PARTITIONED BY (order_date)
CLUSTERED BY (customer_id) INTO 256 BUCKETS
STORED AS PARQUET;

-- Benefit: Joining orders with customers (both bucketed by customer_id)
-- avoids a full shuffle — bucket 42 of orders only joins with bucket 42 of customers
```

```python
# Spark: Write bucketed table
df.write \
    .bucketBy(256, "customer_id") \
    .sortBy("customer_id") \
    .saveAsTable("bucketed_orders")

# Spark: Join two bucketed tables (sort-merge join, no shuffle)
orders = spark.table("bucketed_orders")
customers = spark.table("bucketed_customers")
joined = orders.join(customers, "customer_id")  # No shuffle needed!
```

### BigQuery Clustering

BigQuery clustering sorts data blocks by the clustering columns, enabling efficient data skipping.

```sql
-- BigQuery: Table with partitioning + clustering
CREATE TABLE analytics.fct_events
PARTITION BY DATE(event_timestamp)
CLUSTER BY user_id, event_type
AS SELECT * FROM staging.stg_events;

-- Queries that filter or group by clustering columns benefit:
SELECT event_type, COUNT(*)
FROM analytics.fct_events
WHERE DATE(event_timestamp) = '2024-01-15'
  AND user_id = 'user_abc123'  -- Clustering eliminates blocks
GROUP BY event_type;
```

### Clustering vs Partitioning

| Dimension | Partitioning | Clustering |
|-----------|-------------|-----------|
| **Mechanism** | Physical directory/file separation | Sorting within storage blocks |
| **Cardinality** | Low (< 10,000 distinct values) | High (millions of distinct values) |
| **Granularity** | Entire partition pruned | Block-level data skipping |
| **Columns** | Typically 1 | Up to 4 (BigQuery) |
| **Maintenance** | Manual (partition management) | Automatic (BigQuery re-clusters) |
| **Best for** | Date, region, status | user_id, product_id, session_id |

> **Critical Insight:** Use partitioning for low-cardinality columns that appear in almost every query (date). Use clustering for high-cardinality columns that appear in many queries (user_id). The combination — partition by date, cluster by user_id — handles the most common analytical access pattern: "What did user X do on date Y?"

---

## Sort Keys (Redshift)

### Redshift Sort Key Types

| Type | How It Works | Best For |
|------|-------------|----------|
| **Compound sort key** | Data sorted by col1, then col2, then col3 (hierarchical) | Queries that filter on the leading column(s) |
| **Interleaved sort key** | Equal weight to each column (multi-dimensional) | Queries that filter on any combination of columns |

```sql
-- Compound sort key: optimal for date-first queries
CREATE TABLE fct_events (
    event_id BIGINT,
    event_date DATE,
    user_id BIGINT,
    event_type VARCHAR(50),
    amount DECIMAL(10,2)
)
SORTKEY (event_date, user_id);
-- Optimal for: WHERE event_date = X (or event_date = X AND user_id = Y)
-- Less optimal for: WHERE user_id = Y (without date filter)

-- Interleaved sort key: equal benefit for any column combination
CREATE TABLE fct_events (...)
INTERLEAVED SORTKEY (event_date, user_id, event_type);
-- Equally optimal for:
--   WHERE event_date = X
--   WHERE user_id = Y
--   WHERE event_type = Z
--   WHERE event_date = X AND user_id = Y
-- Trade-off: VACUUM (re-sorting) is more expensive
```

### Sort Key Selection Guidelines

```
COMPOUND SORT KEY when:
- One dominant access pattern (e.g., always filter by date first)
- Queries build filters hierarchically (date → user → event_type)
- Low maintenance preference (simpler VACUUM)

INTERLEAVED SORT KEY when:
- Multiple access patterns with different leading columns
- Ad-hoc queries from analysts (unpredictable filters)
- Willing to accept slower VACUUM operations
```

### Zone Maps (How Sort Keys Enable Skipping)

```
Sorted by event_date:
Block 1: event_date min=2024-01-01, max=2024-01-05
Block 2: event_date min=2024-01-06, max=2024-01-10
Block 3: event_date min=2024-01-11, max=2024-01-15

Query: WHERE event_date = '2024-01-08'
→ Skip Block 1 (max < 08), Read Block 2, Skip Block 3 (min > 08)
→ 67% data skipped via zone maps
```

---

## Z-Ordering

### What is Z-Ordering?

Z-ordering (locality-sensitive hashing) interleaves bits from multiple columns to create a single sort order that respects all dimensions simultaneously. Used in Delta Lake and Databricks.

```
Standard sort (by col_A then col_B):
Good for: WHERE col_A = X
Bad for:  WHERE col_B = Y (col_B not sorted within blocks)

Z-ordering (by col_A, col_B):
Good for: WHERE col_A = X
Good for: WHERE col_B = Y
Good for: WHERE col_A = X AND col_B = Y
```

### Z-Order Visual Intuition

```
2D space (col_A × col_B):

Standard sort by A:        Z-ordered by (A, B):
┌───┬───┬───┬───┐         ┌───┬───┬───┬───┐
│ 1 │ 1 │ 1 │ 1 │ B=4    │ 3 │ 4 │ 3 │ 4 │ B=4
├───┼───┼───┼───┤         ├───┼───┼───┼───┤
│ 1 │ 1 │ 1 │ 1 │ B=3    │ 3 │ 4 │ 3 │ 4 │ B=3
├───┼───┼───┼───┤         ├───┼───┼───┼───┤
│ 1 │ 1 │ 1 │ 1 │ B=2    │ 1 │ 2 │ 1 │ 2 │ B=2
├───┼───┼───┼───┤         ├───┼───┼───┼───┤
│ 1 │ 1 │ 1 │ 1 │ B=1    │ 1 │ 2 │ 1 │ 2 │ B=1
└───┴───┴───┴───┘         └───┴───┴───┴───┘
A=1  A=2  A=3  A=4        A=1  A=2  A=3  A=4

Numbers = block assignment
Standard: Filter on B scans all blocks (B values in every block)
Z-order:  Filter on B=1 or B=2 → only blocks 1,2 (skip blocks 3,4)
```

### Delta Lake Z-Ordering

```python
# Databricks / Delta Lake: Z-order a table
# This physically reorganizes the Parquet files
spark.sql("""
    OPTIMIZE analytics.fct_events
    ZORDER BY (user_id, event_type)
""")

# Combined with partitioning:
# Partition by date (coarse pruning) + Z-order within partitions (fine skipping)
spark.sql("""
    OPTIMIZE analytics.fct_events
    WHERE event_date >= '2024-01-01'
    ZORDER BY (user_id, product_id)
""")
```

### When to Use Z-Ordering

| Scenario | Recommendation |
|----------|---------------|
| Single dominant filter column | Regular sort (compound key) is sufficient |
| Two or more equally important filter columns | Z-ordering provides balanced skipping |
| Queries filter on unpredictable column combinations | Z-ordering handles ad-hoc patterns well |
| Very high cardinality columns only | Z-ordering has diminishing returns (consider bloom filters) |
| Read-heavy, write-light table | Good fit (Z-ordering is expensive to maintain) |

> **Critical Insight:** Z-ordering is not free — it requires an OPTIMIZE operation that rewrites files. This is expensive for large tables and should be run periodically (not on every write). The benefit is purely at read time — queries that filter on any z-ordered column benefit from data skipping, regardless of which column they filter on.

---

## Practical Guidelines

### The Universal Rule

**Partition by date, cluster by high-cardinality columns that appear in filters.**

```sql
-- The pattern that works for 90% of fact tables:
-- BigQuery
CREATE TABLE analytics.fct_orders
PARTITION BY DATE(order_date)
CLUSTER BY customer_id, product_id;

-- Snowflake
CREATE TABLE analytics.fct_orders (...)
CLUSTER BY (order_date, customer_id);

-- Delta Lake
-- Partition by date, Z-order by customer + product
df.write.partitionBy("order_date").format("delta").save(path)
OPTIMIZE analytics.fct_orders ZORDER BY (customer_id, product_id);
```

### Decision Framework

```
Step 1: Is the table > 1 GB?
├── No  → No partitioning needed
└── Yes → Continue

Step 2: Do most queries filter by a date/time column?
├── Yes → Partition by that date column
└── No  → Consider hash or list partitioning

Step 3: After partitioning, are partitions still > 1 GB?
├── Yes → Add clustering on frequently filtered columns
└── No  → Partitioning alone is sufficient

Step 4: Do queries filter on multiple columns unpredictably?
├── Yes → Z-ordering on those columns
└── No  → Simple sort/clustering on the most filtered column
```

### Partition Key Selection Criteria

| Criterion | Good Partition Key | Bad Partition Key |
|-----------|-------------------|-------------------|
| **Cardinality** | Low-medium (100s-1000s) | Very high (millions) or very low (2-3) |
| **Query coverage** | Appears in 80%+ of queries | Rarely filtered |
| **Distribution** | Relatively even across values | Highly skewed (90% in one value) |
| **Growth** | Predictable (new partitions per day) | Unpredictable or explosive |
| **Examples** | date, region, status | user_id, order_id, UUID |

### Anti-Patterns

```sql
-- ANTI-PATTERN: Partitioning by high-cardinality column
PARTITION BY user_id  -- 10M users = 10M partitions (disaster)

-- ANTI-PATTERN: Partitioning by boolean
PARTITION BY is_active  -- Only 2 partitions (no benefit)

-- ANTI-PATTERN: Over-partitioning
PARTITION BY (year, month, day, hour, country, device_type)
-- Combinatorial explosion: 365 × 24 × 200 × 5 = 8.7M partitions

-- CORRECT: Partition by date, cluster by others
PARTITION BY DATE(event_time)
CLUSTER BY country, device_type
```

---

## Platform-Specific Strategies

### BigQuery

```sql
-- Partitioning (required for large tables)
PARTITION BY DATE(timestamp_column)     -- Daily partitions
PARTITION BY TIMESTAMP_TRUNC(ts, MONTH) -- Monthly partitions
PARTITION BY RANGE_BUCKET(int_col, GENERATE_ARRAY(0, 100, 10))  -- Integer range
PARTITION BY _PARTITIONTIME              -- Ingestion-time partitioning

-- Clustering (up to 4 columns, order matters)
CLUSTER BY col1, col2, col3, col4

-- Require partition filter (prevent full scans)
ALTER TABLE dataset.table
SET OPTIONS (require_partition_filter = TRUE);
```

### Snowflake

```sql
-- Automatic micro-partitioning (no explicit partition DDL)
-- Snowflake auto-creates micro-partitions (~16 MB each)
-- Use CLUSTER BY to influence organization:

ALTER TABLE fct_events CLUSTER BY (event_date, customer_id);

-- Check clustering quality:
SELECT SYSTEM$CLUSTERING_INFORMATION('fct_events', '(event_date, customer_id)');

-- Snowflake's Automatic Clustering maintains sort order as data is ingested
-- (enterprise feature, costs credits)
```

### Databricks / Delta Lake

```python
# Partition on write
df.write.format("delta").partitionBy("date").save("/lake/events")

# Z-ordering (run periodically)
spark.sql("OPTIMIZE events ZORDER BY (user_id, event_type)")

# Liquid clustering (newer, replaces partitioning + Z-ordering)
spark.sql("""
    CREATE TABLE events USING DELTA
    CLUSTER BY (event_date, user_id)
""")
# Liquid clustering automatically manages file organization
```

### Redshift

```sql
-- Distribution style + sort key
CREATE TABLE fct_orders (
    order_id BIGINT,
    customer_id BIGINT,
    order_date DATE,
    amount DECIMAL(10,2)
)
DISTSTYLE KEY
DISTKEY (customer_id)          -- Distribute by customer for join colocation
COMPOUND SORTKEY (order_date, customer_id);  -- Sort for range scans
```

---

## Interview Talking Points

> **"How do you decide on a partitioning strategy for a fact table?"**
>
> "I start by analyzing query patterns — what columns appear in WHERE clauses most frequently? For event/transaction tables, it's almost always a date column, so I partition by date. Then I verify the partition size — each partition should be at least 128MB to avoid the small files problem. If the table is queried with additional high-cardinality filters like user_id or product_id, I add clustering on those columns. The combination of date partitioning plus clustering handles 90% of analytical access patterns efficiently."

> **"What is over-partitioning and how do you avoid it?"**
>
> "Over-partitioning creates too many small partitions, leading to excessive metadata, tiny files with poor compression, and query planning overhead that exceeds execution time. The key metric is partition size — if a partition has less than 128MB of data, your granularity is too fine. Solutions: coarsen the partition granularity (hourly to daily), use clustering instead of sub-partitioning, or run periodic compaction to merge small files."

> **"Explain the difference between partitioning, clustering, and Z-ordering."**
>
> "Partitioning physically separates data into directories or segments by a low-cardinality column — entire partitions are skipped if they don't match the query filter. Clustering sorts data within partitions by high-cardinality columns — individual blocks are skipped using min/max statistics. Z-ordering is multi-dimensional clustering that interleaves multiple column dimensions, providing balanced data skipping regardless of which z-ordered column you filter on. I use them in combination: partition by date for coarse pruning, cluster or Z-order by frequently filtered columns for fine-grained skipping."

---

## Common Mistakes

| ❌ Mistake | ✅ Best Practice |
|-----------|-----------------|
| Partitioning by high-cardinality column (user_id) | Partition by low-cardinality (date), cluster by high-cardinality |
| No partitioning on tables > 10 GB | Always partition large tables — date is the safe default |
| Wrapping partition column in functions in queries | Filter directly on the partition column for pruning to work |
| Over-partitioning (minute-level on low-volume tables) | Ensure each partition has at least 128MB of data |
| Partitioning by a column rarely used in filters | Partition by the column that appears in 80%+ of query filters |
| Ignoring data skew across partitions | Monitor partition sizes; rebalance if heavily skewed |
| Z-ordering on too many columns (> 4) | Focus on the 2-3 columns most frequently filtered |
| Never running OPTIMIZE/VACUUM after writes | Schedule periodic maintenance for sorted/z-ordered tables |
| Same strategy for OLTP and OLAP tables | OLTP: hash/range for write distribution. OLAP: date for read pruning |
| Not verifying pruning is working (EXPLAIN) | Regularly check query plans to confirm partitions are being pruned |

---

## Rapid-Fire Q&A

**Q1: What is partition pruning?**
> The query engine eliminates entire partitions from scanning when the query filter doesn't match the partition key values. Only matching partitions are read.

**Q2: What is the best default partition key for fact tables?**
> Date (typically day-level). Most analytical queries filter by time, and dates provide predictable, even-sized partitions.

**Q3: What causes over-partitioning and what's the symptom?**
> Too-fine granularity (hourly/minute) on low-volume data creates thousands of tiny files. Symptoms: slow query planning, poor compression, excessive metadata overhead.

**Q4: What is the difference between partitioning and clustering?**
> Partitioning physically separates data into distinct segments (entire partitions skipped). Clustering sorts data within partitions/blocks (individual blocks skipped via statistics).

**Q5: What is a sort key in Redshift?**
> The column(s) that determine physical row ordering. Queries filtering on the sort key benefit from zone maps that skip blocks where the key range doesn't match.

**Q6: What is Z-ordering?**
> Multi-dimensional clustering that interleaves bits from multiple columns to create balanced data locality across all dimensions simultaneously.

**Q7: When should you use clustering instead of partitioning?**
> For high-cardinality columns (user_id, product_id) that would create too many partitions. Clustering provides block-level skipping without creating separate physical partitions.

**Q8: What happens if you query without a partition filter?**
> The engine scans all partitions (full table scan). BigQuery allows enforcing partition filters with `require_partition_filter = TRUE`.

**Q9: How do you choose between compound and interleaved sort keys in Redshift?**
> Compound when queries consistently filter on the same leading column(s). Interleaved when queries filter on unpredictable combinations of columns.

**Q10: What is the ideal partition size?**
> At least 128MB of compressed data per partition. Below this, the overhead of partition management exceeds the benefit of pruning.

---

## ASCII Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════╗
║        PARTITIONING & BUCKETING CHEAT SHEET                      ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  GOLDEN RULE:                                                    ║
║  Partition by DATE + Cluster by HIGH-CARDINALITY filters         ║
║                                                                  ║
║  DECISION TREE:                                                  ║
║  Table > 1 GB? ─── No ──▶ No partitioning needed                ║
║       │                                                          ║
║      Yes                                                         ║
║       │                                                          ║
║  Queries filter by date? ─── Yes ──▶ PARTITION BY date           ║
║       │                                                          ║
║      No ──▶ Partition by most-filtered low-cardinality col       ║
║       │                                                          ║
║  Partitions > 128MB each?                                        ║
║  ├── Yes ──▶ Add CLUSTERING on other filter columns              ║
║  └── No  ──▶ Coarsen partition granularity                       ║
║                                                                  ║
║  STRATEGIES BY PLATFORM:                                         ║
║  ┌────────────┬───────────────────────────────────────┐          ║
║  │ BigQuery   │ PARTITION BY + CLUSTER BY (up to 4)  │          ║
║  │ Snowflake  │ Auto micro-partition + CLUSTER BY    │          ║
║  │ Redshift   │ DISTKEY + SORTKEY                    │          ║
║  │ Delta Lake │ partitionBy + ZORDER BY              │          ║
║  │ Hive/Spark │ PARTITIONED BY + CLUSTERED BY        │          ║
║  └────────────┴───────────────────────────────────────┘          ║
║                                                                  ║
║  SIZING GUIDE:                                                   ║
║  Min partition size:  128 MB (compressed)                        ║
║  Ideal file size:     128 MB - 1 GB                              ║
║  Max partitions:      ~10,000 (beyond this = metadata overhead)  ║
║                                                                  ║
║  WHAT BREAKS PRUNING:                                            ║
║  ✗ WHERE YEAR(dt) = 2024        (function on partition col)      ║
║  ✓ WHERE dt >= '2024-01-01'     (direct comparison)              ║
║  ✗ WHERE CAST(dt AS STRING)...  (casting partition col)          ║
║  ✓ WHERE dt = DATE '2024-01-15' (native type)                   ║
║                                                                  ║
║  DATA SKIPPING HIERARCHY:                                        ║
║  Level 1: Partition pruning   (skip entire partitions)           ║
║  Level 2: Clustering/sorting  (skip blocks via zone maps)        ║
║  Level 3: Column pruning      (skip unneeded columns)            ║
║  Level 4: Predicate pushdown  (skip rows via page stats)         ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 21 of 45*
