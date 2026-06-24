# 🎯 Topic 46: Spark and Distributed Computing

> *"When your data outgrows a single machine, you don't need a bigger machine — you need a smarter framework. Apache Spark is the lingua franca of large-scale data processing, and knowing when and how to use it separates senior data scientists from analysts who hit a wall at 10M rows."*

---

## 📑 Table of Contents

1. [Why Spark? The Distributed Computing Imperative](#why-spark-the-distributed-computing-imperative)
2. [SparkSession and Core Architecture](#sparksession-and-core-architecture)
3. [RDD vs DataFrame vs Dataset](#rdd-vs-dataframe-vs-dataset)
4. [Transformations vs Actions and Lazy Evaluation](#transformations-vs-actions-and-lazy-evaluation)
5. [Partitioning, Shuffles, and Data Locality](#partitioning-shuffles-and-data-locality)
6. [Join Strategies: Broadcast vs Sort-Merge vs Shuffle-Hash](#join-strategies-broadcast-vs-sort-merge-vs-shuffle-hash)
7. [Caching and Persistence Levels](#caching-and-persistence-levels)
8. [Handling Data Skew (Salting and Beyond)](#handling-data-skew-salting-and-beyond)
9. [Spark Optimization Toolkit](#spark-optimization-toolkit)
10. [UDFs: Why to Avoid Them and Alternatives](#udfs-why-to-avoid-them-and-alternatives)
11. [Spark vs Pandas: When to Use Which](#spark-vs-pandas-when-to-use-which)
12. [Real-World Industry Examples](#real-world-industry-examples)
13. [Interview Talking Points](#interview-talking-points)
14. [Common Mistakes](#common-mistakes)
15. [Rapid-Fire Q&A](#rapid-fire-qa)
16. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Why Spark? The Distributed Computing Imperative

### The Single-Machine Ceiling

```
DATA SIZE vs TOOL SELECTION:

  < 1 GB     → pandas / polars (single machine, in-memory)
  1-10 GB    → pandas with chunking / polars / DuckDB
  10-100 GB  → Spark (single-node cluster or small cluster)
  100 GB+    → Spark (multi-node cluster, mandatory)
  1 TB+      → Spark with careful optimization (partitioning, skew handling)

  Rule of thumb: If data fits in RAM of one machine, 
  don't use Spark. The overhead isn't worth it.
```

### Spark's Core Philosophy

Spark distributes computation across a cluster while providing a familiar DataFrame API. Key principles:

1. **Data parallelism** — Split data into partitions, process each independently
2. **Lazy evaluation** — Build a DAG of transformations, execute only when an action is called
3. **In-memory computation** — Intermediate results stay in RAM (not written to disk like MapReduce)
4. **Fault tolerance** — Lineage graph enables recomputation of lost partitions

> **Critical Insight:** Spark is NOT always faster than pandas. For datasets under 1GB, pandas will outperform Spark due to Spark's scheduling overhead, serialization costs, and JVM startup time. The question is not "which is faster" but "which scales."

---

## SparkSession and Core Architecture

### Creating a SparkSession

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("ClickAnalysis") \
    .config("spark.sql.adaptive.enabled", "true") \
    .config("spark.sql.shuffle.partitions", "200") \
    .config("spark.driver.memory", "4g") \
    .config("spark.executor.memory", "8g") \
    .config("spark.executor.cores", "4") \
    .getOrCreate()
```

### Architecture Components

```
┌─────────────────────────────────────────────────────────────┐
│                      DRIVER PROGRAM                         │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ SparkContext│  │  DAG         │  │  Task            │  │
│  │  (entry pt) │  │  Scheduler   │  │  Scheduler       │  │
│  └─────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────┬───────────────────────────────────┘
                          │
            ┌─────────────┼─────────────┐
            │             │             │
    ┌───────▼───────┐ ┌──▼──────────┐ ┌▼──────────────┐
    │  EXECUTOR 1   │ │  EXECUTOR 2 │ │  EXECUTOR 3   │
    │ ┌────┐ ┌────┐ │ │ ┌────┐┌───┐│ │ ┌────┐ ┌────┐ │
    │ │Task│ │Task│ │ │ │Task││Tsk││ │ │Task│ │Task│ │
    │ └────┘ └────┘ │ │ └────┘└───┘│ │ └────┘ └────┘ │
    │   Cache/RAM   │ │  Cache/RAM  │ │   Cache/RAM   │
    └───────────────┘ └─────────────┘ └───────────────┘
```

**Driver:** Orchestrates the job. Holds the SparkSession. Converts your code into a DAG of stages and tasks.

**Executors:** JVM processes on worker nodes. Run tasks, store cached data. Each executor has cores (parallelism) and memory.

**Cluster Manager:** YARN, Kubernetes, or Mesos. Allocates resources to Spark applications.

> **Critical Insight:** The driver is a single point of failure. If you `collect()` a 50GB DataFrame to the driver, you'll OOM the driver. Always keep large data distributed — only bring small results back to the driver.

---

## RDD vs DataFrame vs Dataset

### Comparison Table

| Feature | RDD | DataFrame | Dataset (Scala/Java) |
|---------|-----|-----------|---------------------|
| **API Level** | Low-level | High-level (SQL-like) | High-level + type-safe |
| **Optimization** | No Catalyst/Tungsten | Full Catalyst + Tungsten | Full Catalyst + Tungsten |
| **Type Safety** | Compile-time (Scala) | Runtime only | Compile-time |
| **Schema** | No schema | Named columns + types | Named columns + types |
| **Serialization** | Java serialization (slow) | Tungsten binary (fast) | Tungsten binary (fast) |
| **When to Use** | Custom partitioning, low-level control | 99% of use cases in PySpark | Scala/Java with type safety needs |
| **Performance** | Slowest (no optimization) | Fastest (optimized) | Fastest (optimized) |
| **Python Support** | Yes (slow) | Yes (fast) | No (Scala/Java only) |

### Code Comparison

```python
# RDD approach (AVOID in modern Spark)
rdd = spark.sparkContext.textFile("clicks.csv")
result = rdd.map(lambda line: line.split(",")) \
            .filter(lambda x: x[2] == "purchase") \
            .map(lambda x: (x[1], float(x[3]))) \
            .reduceByKey(lambda a, b: a + b)

# DataFrame approach (PREFERRED)
df = spark.read.csv("clicks.csv", header=True, inferSchema=True)
result = df.filter(df.event_type == "purchase") \
           .groupBy("user_id") \
           .agg(F.sum("amount").alias("total_spend"))

# Spark SQL approach (EQUALLY PREFERRED)
df.createOrReplaceTempView("clicks")
result = spark.sql("""
    SELECT user_id, SUM(amount) as total_spend
    FROM clicks
    WHERE event_type = 'purchase'
    GROUP BY user_id
""")
```

> **Critical Insight:** In PySpark, there is NO Dataset API — you only have RDDs and DataFrames. Always use DataFrames because they run through the Catalyst optimizer and Tungsten execution engine, making them 10-100x faster than RDD operations in Python (which suffer from Python-JVM serialization overhead).

---

## Transformations vs Actions and Lazy Evaluation

### The Fundamental Distinction

```
TRANSFORMATIONS (Lazy — build the plan):
  .filter()      .select()      .groupBy()
  .join()        .withColumn()  .orderBy()
  .distinct()    .repartition() .coalesce()
  .union()       .drop()        .agg()

ACTIONS (Eager — trigger execution):
  .count()       .collect()     .show()
  .first()       .take(n)       .write.*
  .toPandas()    .foreach()     .reduce()
```

### Lazy Evaluation in Practice

```python
# Nothing executes here — Spark just builds the DAG
df = spark.read.parquet("s3://data/events/")       # Read plan
filtered = df.filter(df.country == "US")            # Filter plan
aggregated = filtered.groupBy("city").count()       # Agg plan
sorted_df = aggregated.orderBy(F.desc("count"))     # Sort plan

# NOW execution happens — the entire pipeline runs
sorted_df.show(10)  # <-- This triggers everything above
```

### Why Lazy Evaluation Matters

1. **Optimization:** Catalyst can reorder/push down operations before execution
2. **Efficiency:** Avoid materializing intermediate DataFrames unnecessarily
3. **Pipeline fusion:** Multiple narrow transformations are fused into a single stage

```
WITHOUT lazy evaluation (hypothetical eager mode):
  Read 1TB → Write intermediate → Filter → Write intermediate → Join
  Total I/O: 3TB+

WITH lazy evaluation:
  Plan: Read → Filter → Join
  Optimized: Read (with pushdown filter) → Join
  Total I/O: ~200GB (only matching rows read + shuffled)
```

> **Critical Insight:** Lazy evaluation means errors surface LATE. If you have a bad column reference in line 10 but your action is on line 50, the error only appears at line 50. This makes debugging harder — use `.printSchema()` (lazy but validates) and `.explain()` (shows plan) liberally during development.

---

## Partitioning, Shuffles, and Data Locality

### Partitions: The Unit of Parallelism

```
┌─────────────────────────────────────────────────────┐
│              1TB DataFrame                           │
│                                                     │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐      ┌─────┐   │
│  │ P0  │ │ P1  │ │ P2  │ │ P3  │ ...  │P199 │   │
│  │5 GB │ │5 GB │ │5 GB │ │5 GB │      │5 GB │   │
│  └─────┘ └─────┘ └─────┘ └─────┘      └─────┘   │
│                                                     │
│  200 partitions × 5GB = 1TB                        │
│  Each partition processed by 1 task on 1 core      │
└─────────────────────────────────────────────────────┘

GUIDELINES:
  - Target partition size: 128MB - 512MB (sweet spot)
  - Too few partitions → underutilization, OOM risk
  - Too many partitions → scheduler overhead, small file problem
  - Rule of thumb: 2-4 partitions per available core
```

### Narrow vs Wide Transformations

```
NARROW (no shuffle — fast):                 WIDE (shuffle — expensive):
  .filter()                                   .groupBy().agg()
  .select()                                   .join() (sort-merge)
  .map()                                      .repartition()
  .withColumn()                               .orderBy()
  .coalesce()                                 .distinct()

  Each output partition depends               Output partitions depend on
  on exactly ONE input partition              MULTIPLE input partitions
  → No data movement needed                  → Data must move across network
```

### The Cost of Shuffles

```python
# This triggers a SHUFFLE (all data for same key moves to same partition)
df.groupBy("user_id").agg(F.sum("revenue"))

# Shuffle involves:
# 1. Each executor writes shuffle files to local disk
# 2. Downstream tasks pull data from upstream executors over network
# 3. Data is sorted within each partition by key

# To see shuffles in your job:
df.groupBy("user_id").agg(F.sum("revenue")).explain()
# Look for "Exchange" in the plan — that's a shuffle
```

> **Critical Insight:** Shuffles are the #1 performance bottleneck in Spark. Every shuffle means: all-to-all network transfer + disk I/O + sorting. A 1TB shuffle across 100 nodes means each node sends/receives ~10GB over the network. Design your pipeline to minimize shuffles.

---

## Join Strategies: Broadcast vs Sort-Merge vs Shuffle-Hash

### Decision Framework

```
┌─────────────────────────────────────────────────────────┐
│            JOIN STRATEGY SELECTION                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Is one table small (< 10MB default)?                  │
│     YES → BROADCAST JOIN (no shuffle)                  │
│     NO  ↓                                              │
│                                                         │
│  Are both tables large and pre-sorted/bucketed?        │
│     YES → SORT-MERGE JOIN (most common for large-large)│
│     NO  ↓                                              │
│                                                         │
│  Is one table much smaller (but > broadcast threshold)?│
│     YES → SHUFFLE-HASH JOIN                            │
│     NO  → SORT-MERGE JOIN (default)                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Broadcast Join (Map-Side Join)

```python
from pyspark.sql.functions import broadcast

# Small table (dim_country: 200 rows) joined with large table (1B events)
result = events_df.join(
    broadcast(dim_country_df),  # Force broadcast
    on="country_code",
    how="left"
)

# What happens:
# 1. Driver collects dim_country_df (small)
# 2. Broadcasts it to ALL executors
# 3. Each executor does a local hash join with its partition of events
# 4. NO SHUFFLE of the large table!
```

**Broadcast threshold:** `spark.sql.autoBroadcastJoinThreshold` (default 10MB). Set to `-1` to disable auto-broadcast.

### Sort-Merge Join

```python
# Two large tables — both get shuffled by join key, then sorted and merged
orders_df.join(customers_df, on="customer_id", how="inner")

# What happens:
# 1. Both tables are shuffled (repartitioned) by customer_id
# 2. Each partition is sorted by customer_id
# 3. Sorted partitions are merged (like merge step in merge sort)
# Cost: 2 shuffles + 2 sorts + merge
```

### Performance Comparison

| Strategy | Shuffle | Best For | Limitation |
|----------|---------|----------|-----------|
| Broadcast | None | Small-large joins (dim tables) | Small table must fit in executor memory |
| Sort-Merge | Both tables | Large-large joins (default) | Expensive sort step |
| Shuffle-Hash | Both tables | Medium-large joins, unsorted | Requires one side fits in memory per partition |
| Bucket Join | None (if pre-bucketed) | Repeated joins on same key | Requires pre-bucketing setup |

> **Critical Insight:** If you're joining a fact table (billions of rows) with a dimension table (thousands of rows) and it's slow, the first thing to check is whether a broadcast join is being used. Force it with `broadcast()` hint. This alone can turn a 30-minute job into a 2-minute job.

---

## Caching and Persistence Levels

### When to Cache

```python
# Cache when a DataFrame is reused multiple times
user_features = compute_expensive_features(raw_df)  # 10 min computation

# USED MULTIPLE TIMES:
model_1_input = user_features.filter(user_features.segment == "A")
model_2_input = user_features.filter(user_features.segment == "B")
validation    = user_features.filter(user_features.segment == "C")

# Without cache: compute_expensive_features runs 3 TIMES
# With cache:    compute_expensive_features runs 1 TIME

user_features.cache()       # Mark for caching
user_features.count()       # Trigger materialization into memory
```

### Storage Levels

```python
from pyspark import StorageLevel

df.persist(StorageLevel.MEMORY_ONLY)           # Default for .cache()
df.persist(StorageLevel.MEMORY_AND_DISK)       # Spill to disk if no RAM
df.persist(StorageLevel.DISK_ONLY)             # Only disk (rare)
df.persist(StorageLevel.MEMORY_ONLY_SER)       # Serialized (less RAM, more CPU)
df.persist(StorageLevel.MEMORY_AND_DISK_SER)   # Serialized + disk spillover

# ALWAYS unpersist when done
df.unpersist()
```

### Caching Anti-Patterns

```
DON'T cache if:
  - DataFrame is only used once (adds overhead, no benefit)
  - DataFrame is very large and doesn't fit in cluster memory
  - Downstream operations filter heavily (better to recompute)
  - You're in a streaming job (data changes constantly)

DO cache if:
  - DataFrame is reused 2+ times in the same job
  - Recomputation is expensive (complex joins, aggregations)
  - DataFrame fits in cluster memory
  - You're doing iterative algorithms (ML training)
```

---

## Handling Data Skew (Salting and Beyond)

### What is Skew?

```
EVEN DISTRIBUTION (fast):          SKEWED DISTRIBUTION (slow):
  Partition 1: 1M rows               Partition 1: 100K rows
  Partition 2: 1M rows               Partition 2: 100K rows
  Partition 3: 1M rows               Partition 3: 100K rows
  Partition 4: 1M rows               Partition 4: 50M rows  ← BOTTLENECK
                                      
  Total time: max(P1,P2,P3,P4)       Total time: P4 (50x slower!)
  = 1M row processing time            = 50M row processing time
```

### Detecting Skew

```python
# Check partition sizes
df.groupBy(F.spark_partition_id()).count().orderBy(F.desc("count")).show()

# Check key distribution for joins/groupBys
df.groupBy("join_key").count().orderBy(F.desc("count")).show(20)

# If top key has 1000x more rows than median → you have skew
```

### Solution 1: Salting

```python
import pyspark.sql.functions as F
from pyspark.sql.types import IntegerType
import random

# PROBLEM: user_id = "bot_account" has 50M events, others have ~1K
# GroupBy on user_id will create one massive partition

# SOLUTION: Add random salt to break up the hot key
NUM_SALTS = 100

# Step 1: Add salt to the large (skewed) table
events_salted = events_df.withColumn(
    "salt", (F.rand() * NUM_SALTS).cast(IntegerType())
).withColumn(
    "salted_key", F.concat(F.col("user_id"), F.lit("_"), F.col("salt"))
)

# Step 2: Explode the small table to match all salts
from pyspark.sql.functions import explode, array, lit
salt_array = array([lit(i) for i in range(NUM_SALTS)])

users_exploded = users_df.withColumn("salt", explode(salt_array)) \
    .withColumn("salted_key", F.concat(F.col("user_id"), F.lit("_"), F.col("salt")))

# Step 3: Join on salted key (distributes hot key across 100 partitions)
result = events_salted.join(users_exploded, on="salted_key", how="inner")
```

### Solution 2: Adaptive Query Execution (AQE)

```python
# Spark 3.0+ handles skew automatically with AQE
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256MB")

# AQE detects skewed partitions at runtime and splits them automatically
```

### Solution 3: Isolate and Union

```python
# Separate the hot keys from the rest
hot_keys = ["bot_account", "system_user"]

normal_df = events_df.filter(~F.col("user_id").isin(hot_keys))
skewed_df = events_df.filter(F.col("user_id").isin(hot_keys))

# Process normally distributed data with regular join
normal_result = normal_df.join(users_df, on="user_id")

# Process skewed data with broadcast (since it's one key, small user side)
skewed_result = skewed_df.join(broadcast(users_df.filter(
    F.col("user_id").isin(hot_keys)
)), on="user_id")

# Combine
final = normal_result.union(skewed_result)
```

> **Critical Insight:** Skew is the most common cause of "one task takes forever" in production Spark jobs. Before adding more cluster resources, check if one partition has disproportionate data. Salting or AQE can turn a 2-hour job into a 5-minute job without adding a single node.

---

## Spark Optimization Toolkit

### Understanding the Explain Plan

```python
df.explain(mode="formatted")  # or "simple", "extended", "codegen", "cost"

# Key things to look for:
# 1. "Exchange" → shuffle (expensive)
# 2. "BroadcastExchange" → broadcast join (good for small tables)
# 3. "Filter" pushed into "Scan" → predicate pushdown (good)
# 4. "PartitionFilters" → partition pruning (good)
# 5. "Sort" → expensive, check if necessary
```

### Adaptive Query Execution (AQE) — Spark 3.0+

```python
# AQE optimizes at RUNTIME based on actual data statistics
spark.conf.set("spark.sql.adaptive.enabled", "true")

# Key AQE features:
# 1. Coalescing shuffle partitions (reduces small partitions after shuffle)
# 2. Converting sort-merge join to broadcast join (if runtime size is small)
# 3. Handling skewed joins (splits hot partitions)

spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.localShuffleReader.enabled", "true")
```

### Partition Pruning

```python
# DATA LAYOUT (partitioned by date):
# s3://data/events/date=2024-01-01/
# s3://data/events/date=2024-01-02/
# ...
# s3://data/events/date=2024-12-31/

# This reads ONLY the relevant partitions (not all 365 days):
df = spark.read.parquet("s3://data/events/")
jan_data = df.filter(F.col("date") == "2024-01-15")
# Spark only scans the 2024-01-15 partition folder!
```

### Column Pruning + Predicate Pushdown

```python
# GOOD: Select columns early, filter early
result = spark.read.parquet("s3://data/events/") \
    .select("user_id", "event_type", "timestamp") \  # Column pruning
    .filter(F.col("event_type") == "purchase")        # Predicate pushdown

# Parquet format + Spark = only reads the 3 columns from disk
# and skips row groups where event_type != "purchase"
```

### Repartition vs Coalesce

```python
# REPARTITION: Increases or decreases partitions (triggers shuffle)
df.repartition(200)                    # Hash repartition to 200
df.repartition(200, "user_id")        # Repartition BY user_id (same user → same partition)

# COALESCE: Only DECREASES partitions (no shuffle — merges adjacent)
df.coalesce(10)                        # Merge 200 partitions → 10

# Use coalesce when: writing output files (reduce small files)
# Use repartition when: need even distribution or partition by key
```

> **Critical Insight:** The single biggest optimization is often the simplest: filter early, select only needed columns, and use partitioned data with partition pruning. These three alone can reduce data scanned by 10-100x before any other optimization matters.

---

## UDFs: Why to Avoid Them and Alternatives

### The Performance Problem

```
┌──────────────────────────────────────────────────────────┐
│  NATIVE SPARK FUNCTIONS (fast path):                     │
│  Data stays in Tungsten binary format inside JVM         │
│  → No serialization overhead                             │
│  → Vectorized operations                                 │
│  → Catalyst optimization                                 │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│  PYTHON UDF (slow path):                                 │
│  For EACH ROW:                                           │
│    1. Serialize row from JVM → Python                    │
│    2. Execute Python function                            │
│    3. Serialize result from Python → JVM                 │
│  → 10-100x slower than native functions                  │
│  → Blocks Catalyst optimization                          │
└──────────────────────────────────────────────────────────┘
```

### Alternatives to UDFs

```python
# BAD: Python UDF for string manipulation
@F.udf(returnType=StringType())
def extract_domain(email):
    return email.split("@")[1] if "@" in email else None

df.withColumn("domain", extract_domain(df.email))

# GOOD: Native Spark function (100x faster)
df.withColumn("domain", 
    F.when(F.col("email").contains("@"),
           F.split(F.col("email"), "@")[1])
    .otherwise(None)
)

# If you MUST use UDF, use Pandas UDF (vectorized):
from pyspark.sql.functions import pandas_udf
import pandas as pd

@pandas_udf(StringType())
def extract_domain_vectorized(emails: pd.Series) -> pd.Series:
    return emails.str.split("@").str[1]

# Pandas UDFs process batches (Arrow serialization) — 3-10x faster than row UDFs
df.withColumn("domain", extract_domain_vectorized(df.email))
```

### When UDFs Are Unavoidable

```
ACCEPTABLE UDF use cases:
  - Complex ML model inference (not available in Spark ML)
  - Calling external APIs (geocoding, enrichment)
  - Complex parsing logic (regex that can't map to Spark functions)
  - Custom NLP processing (tokenization, entity extraction)

ALWAYS prefer in this order:
  1. Built-in Spark SQL functions (fastest)
  2. spark.sql.functions expressions (fastest)
  3. Pandas UDFs / Arrow-optimized UDFs (medium)
  4. Row-at-a-time Python UDFs (slowest — last resort)
```

> **Critical Insight:** In interviews, if you mention using a UDF, immediately follow up with "but I'd first check if there's a native Spark function for this." This signals you understand the performance implications. Common trap: writing a UDF for something that `F.when()`, `F.regexp_extract()`, or `F.transform()` can already do.

---

## Spark vs Pandas: When to Use Which

### Decision Matrix

| Criterion | pandas | Spark |
|-----------|--------|-------|
| **Data size** | < 10GB (fits in RAM) | > 10GB or growing |
| **Latency requirement** | Milliseconds-seconds | Seconds-minutes acceptable |
| **Development speed** | Faster iteration | Slower (cluster setup, lazy eval) |
| **Rich ML ecosystem** | scikit-learn, statsmodels | Spark MLlib (less flexible) |
| **Visualization** | matplotlib, seaborn inline | Must collect() first |
| **Ad-hoc exploration** | Excellent (Jupyter) | Clunky (delayed feedback) |
| **Production pipeline** | Fragile at scale | Built for scale |
| **Team skillset** | Every DS knows pandas | Needs Spark experience |
| **Cost** | One laptop/server | Cluster compute costs |

### Pandas API on Spark (formerly Koalas)

```python
import pyspark.pandas as ps

# Use pandas syntax but runs on Spark engine!
pdf = ps.read_parquet("s3://data/large_dataset/")
pdf.groupby("category")["revenue"].mean()
pdf.describe()

# Limitations:
# - Not all pandas functions supported
# - Some operations trigger expensive shuffles silently
# - Ordering not preserved (distributed = no global order)
```

### The Hybrid Approach

```python
# PATTERN: Spark for heavy lifting, pandas for final analysis

# Step 1: Spark processes 1TB of raw data
aggregated = spark.read.parquet("s3://events/") \
    .filter(F.col("date") >= "2024-01-01") \
    .groupBy("user_segment", "week") \
    .agg(F.sum("revenue").alias("weekly_revenue"),
         F.countDistinct("user_id").alias("unique_users"))

# Step 2: Result is small (50 segments × 52 weeks = 2600 rows)
pdf = aggregated.toPandas()

# Step 3: pandas for final analysis + visualization
import matplotlib.pyplot as plt
pdf.pivot(index="week", columns="user_segment", values="weekly_revenue").plot()
```

---

## Real-World Industry Examples

### Uber: Dynamic Pricing Pipeline

```
PROBLEM: Process 100M+ trips/day to compute surge pricing
SOLUTION:
  - Streaming Spark (Structured Streaming) for real-time demand signals
  - Batch Spark for historical pattern aggregation
  - Partitioned by (city, hour) to enable partition pruning
  - Broadcast join with geo-fence definitions (small lookup table)
  - AQE handles variable city sizes (NYC >> small towns)
```

### Netflix: Recommendation Feature Engineering

```
PROBLEM: Compute viewing features for 200M+ users across 10K+ titles
SOLUTION:
  - Daily batch Spark job computing user-item interaction features
  - Window functions for "watch history in last 7/30/90 days"
  - Skew handling: some titles watched by 100M+ users (salting)
  - Bucketed tables by user_id for efficient repeated joins
  - Cache intermediate user profiles (reused by multiple models)
```

### Spotify: Wrapped Year-in-Review

```
PROBLEM: Compute annual listening stats for 500M+ users
SOLUTION:
  - Full year of streaming data (~50PB)
  - Partition by (user_id_prefix, month) for manageable partition sizes
  - Multi-stage aggregation: daily → weekly → monthly → yearly
  - Broadcast artist metadata (small) to enrich listening records
  - Repartition by user_id for final per-user stat computation
  - Coalesce output to optimal file sizes for downstream consumption
```

---

## Interview Talking Points

> "In my role, I built a PySpark pipeline that processed 800GB of daily clickstream data to compute user engagement features. The key challenge was a skewed join — our 'guest' user ID accounted for 30% of all events. I implemented salting with 50 salt buckets, which reduced our join stage from 45 minutes to 3 minutes. I also added broadcast joins for our dimension tables (product catalog: 2M rows, 500MB) which eliminated two shuffles entirely."

> "I'm deliberate about choosing between pandas and Spark. For our A/B test analysis (typically 5M rows after filtering), I use pandas because the iteration speed matters more during exploratory analysis. But for our feature engineering pipeline that processes the full event stream (2TB/day), Spark is non-negotiable. I write the feature logic in Spark, then sample down to pandas for validation and visualization."

> "When optimizing a slow Spark job, my process is: first check the Spark UI for skewed stages (one task taking 10x longer). Then use `.explain()` to verify partition pruning and predicate pushdown are happening. Then check if any UDFs can be replaced with native functions. In one case, replacing a Python UDF that parsed JSON with `F.from_json()` reduced our job from 2 hours to 20 minutes."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Using `collect()` on large DataFrames | Use `.show()`, `.take(n)`, or write to storage |
| Writing Python UDFs for simple operations | Use built-in `pyspark.sql.functions` (100x faster) |
| Not caching DataFrames reused multiple times | Cache + materialize with `.count()` when reused 2+ times |
| Using `repartition()` when `coalesce()` suffices | Coalesce for reducing partitions (avoids shuffle) |
| Ignoring data skew in joins/groupBys | Detect with key distribution analysis, fix with salting or AQE |
| Setting too few shuffle partitions for large data | Scale `spark.sql.shuffle.partitions` with data size (not default 200) |
| Not filtering early in the pipeline | Push filters as early as possible (reduces downstream data volume) |
| Using `orderBy()` unnecessarily (triggers global sort) | Only sort when output order matters; use `sortWithinPartitions()` when possible |
| Reading all columns from wide tables | `select()` only needed columns early (enables column pruning) |
| Treating Spark like pandas (expecting eager evaluation) | Understand lazy evaluation; use `.explain()` and `.printSchema()` for debugging |

---

## Rapid-Fire Q&A

**Q1: What is lazy evaluation in Spark?**
Transformations don't execute immediately — they build a DAG (Directed Acyclic Graph). Execution only happens when an action (count, collect, write) is called. This allows Catalyst optimizer to rearrange and optimize the full pipeline before running it.

**Q2: How would you process 1TB of click data to find top 100 users by revenue?**
Read partitioned parquet from S3, filter to relevant date range (partition pruning), select only user_id and revenue columns (column pruning), groupBy user_id with sum(revenue), orderBy descending, take(100). Key optimizations: partition pruning, column pruning, AQE for shuffle coalescing.

**Q3: What's the difference between `repartition()` and `coalesce()`?**
`repartition(n)` triggers a full shuffle to create exactly n evenly-distributed partitions (can increase or decrease). `coalesce(n)` only decreases partitions by merging adjacent ones without a shuffle. Use coalesce when reducing partition count (e.g., writing output files).

**Q4: When would you use a broadcast join?**
When one side of the join is small enough to fit in executor memory (default threshold: 10MB, often increased to 100-500MB). The small table is sent to all executors, eliminating the shuffle of the large table. Ideal for fact-dimension joins.

**Q5: How do you handle data skew in a join?**
Three approaches: (1) Salting — add random prefix to skewed keys and explode the other table to match. (2) AQE with skew handling enabled — Spark auto-splits hot partitions. (3) Isolate-and-union — process hot keys separately with broadcast join, union with normal join results.

**Q6: Why are Python UDFs slow in PySpark?**
Data must be serialized from JVM (Tungsten format) to Python and back for every row. This cross-process communication + Python's GIL makes row-at-a-time UDFs 10-100x slower than native Spark functions. Pandas UDFs mitigate this by processing Arrow batches.

**Q7: What does the Catalyst optimizer do?**
Catalyst is Spark's query optimizer. It: (1) parses SQL/DataFrame operations into a logical plan, (2) applies rule-based optimizations (predicate pushdown, column pruning, constant folding), (3) generates physical plans and picks the best one (cost-based), (4) generates optimized JVM bytecode via Tungsten.

**Q8: What is a shuffle and why is it expensive?**
A shuffle redistributes data across partitions based on a key (needed for groupBy, join, distinct). It's expensive because: all-to-all network transfer between executors, data written to local disk before transfer, and requires sorting. It's the primary bottleneck in most Spark jobs.

**Q9: When should you NOT use Spark?**
When data fits in memory of a single machine (< 10GB), for interactive exploration (pandas is faster to iterate), for real-time sub-second latency (use Flink or specialized systems), for small datasets where startup overhead dominates, or when your team lacks Spark expertise and the problem doesn't require it.

**Q10: What is AQE (Adaptive Query Execution)?**
AQE (Spark 3.0+) optimizes queries at runtime using actual data statistics collected during shuffles. It can: coalesce small post-shuffle partitions, convert sort-merge joins to broadcast joins if a table turns out small, and automatically split skewed partitions. Enable with `spark.sql.adaptive.enabled=true`.

---

## ASCII Cheat Sheet

```
+============================================================+
|    SPARK & DISTRIBUTED COMPUTING — CHEAT SHEET             |
+============================================================+

WHEN TO USE SPARK:
  < 10 GB  → pandas/polars/DuckDB (single machine)
  > 10 GB  → Spark (distributed cluster)
  > 1 TB   → Spark with careful optimization

ARCHITECTURE:
  Driver → DAG Scheduler → Task Scheduler
  Executors (Workers): run tasks, cache data
  Cluster Manager: YARN / K8s / Mesos

TRANSFORMATION vs ACTION:
  Transformations: filter, select, join, groupBy (LAZY)
  Actions: count, collect, show, write           (EAGER)
  Nothing runs until an Action is called!

JOIN STRATEGIES (by table size):
  Small + Large  → BROADCAST JOIN (no shuffle)
  Large + Large  → SORT-MERGE JOIN (2 shuffles)
  Pre-bucketed   → BUCKET JOIN (no shuffle)

OPTIMIZATION CHECKLIST:
  1. Filter early (reduce data volume)
  2. Select only needed columns (column pruning)
  3. Use partitioned data (partition pruning)
  4. Broadcast small tables in joins
  5. Avoid Python UDFs (use native functions)
  6. Cache DataFrames reused 2+ times
  7. Handle skew (salting or AQE)
  8. Coalesce output (avoid small files)

DATA SKEW SOLUTIONS:
  1. Salting: random prefix on hot key + explode small table
  2. AQE: spark.sql.adaptive.skewJoin.enabled = true
  3. Isolate: separate hot keys, broadcast join, union back

UDF PERFORMANCE HIERARCHY (fastest → slowest):
  1. Built-in Spark functions (F.xxx)
  2. Spark SQL expressions
  3. Pandas UDFs (vectorized, Arrow batches)
  4. Python row-at-a-time UDFs (AVOID)

SHUFFLE TRIGGERS (wide transformations):
  groupBy, join, repartition, orderBy, distinct,
  window functions, intersect, subtract

CACHING RULES:
  Cache when: reused 2+ times, expensive to recompute
  Don't cache: used once, very large, streaming data
  Always: unpersist() when done

PARTITION SIZING:
  Target: 128MB - 512MB per partition
  Rule: 2-4 partitions per available core
  Check: df.rdd.getNumPartitions()

SPARK vs PANDAS DECISION:
  Pandas: exploration, visualization, < 10GB, fast iteration
  Spark:  production, > 10GB, pipelines, fault tolerance
  Hybrid: Spark aggregates → toPandas() → visualize

+============================================================+
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 46 of 51*
