# 🎯 Topic 20: Data Formats — Parquet, ORC, Avro, JSON

> *Choosing the right file format is one of the highest-leverage decisions in data engineering — it affects query speed, storage cost, schema evolution, and interoperability across your entire platform*

---

## 📋 Table of Contents

1. [Core Concepts](#core-concepts)
2. [Row vs Columnar Storage](#row-vs-columnar-storage)
3. [Apache Parquet](#apache-parquet)
4. [Apache ORC](#apache-orc)
5. [Apache Avro](#apache-avro)
6. [JSON and CSV](#json-and-csv)
7. [Compression Codecs](#compression-codecs)
8. [Format Comparison Table](#format-comparison-table)
9. [When to Use Which](#when-to-use-which)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Core Concepts

### Why File Format Matters

The file format determines:
- **Query performance** — How fast can you read specific columns?
- **Storage efficiency** — How much space does your data consume?
- **Schema evolution** — Can you add columns without breaking consumers?
- **Interoperability** — Can different tools read this format?
- **Write performance** — How fast can you produce the data?

> **Critical Insight:** Choosing the wrong format can make queries 100x slower and storage 10x more expensive. A columnar format (Parquet) reading 3 columns from a 200-column table reads ~1.5% of the data. A row format (JSON) must read 100% of the data to extract those same 3 columns. At petabyte scale, this translates to minutes vs hours and dollars vs thousands of dollars.

### The Two Fundamental Storage Layouts

| Layout | Stores | Best For | Example Formats |
|--------|--------|----------|-----------------|
| **Row-oriented** | Complete rows together | OLTP, full-row reads, streaming | Avro, JSON, CSV |
| **Column-oriented** | Complete columns together | OLAP, analytical queries, aggregations | Parquet, ORC |

---

## Row vs Columnar Storage

### How Data is Stored

```
Original Table:
┌────┬────────┬─────┬────────┐
│ id │  name  │ age │ salary │
├────┼────────┼─────┼────────┤
│  1 │ Alice  │  30 │  90000 │
│  2 │ Bob    │  25 │  75000 │
│  3 │ Carol  │  35 │ 110000 │
└────┴────────┴─────┴────────┘

ROW STORAGE (how it's stored on disk):
[1, Alice, 30, 90000] [2, Bob, 25, 75000] [3, Carol, 35, 110000]
 └─── Row 1 ──────┘   └─── Row 2 ──────┘  └─── Row 3 ─────────┘

COLUMNAR STORAGE (how it's stored on disk):
[1, 2, 3] [Alice, Bob, Carol] [30, 25, 35] [90000, 75000, 110000]
 └─ id ──┘ └──── name ──────┘ └── age ───┘ └──── salary ────────┘
```

### Why Columnar is Faster for Analytics

```sql
-- Analytical query: "What's the average salary?"
SELECT AVG(salary) FROM employees;
```

| Storage | Data Read | Explanation |
|---------|-----------|-------------|
| **Row** | All 4 columns x all rows | Must scan entire rows to extract salary |
| **Columnar** | Only salary column | Reads just the salary bytes, skips everything else |

For a table with 200 columns and a query touching 3 columns:
- Row format reads: **100% of data**
- Columnar format reads: **~1.5% of data**

### Comparison

| Dimension | Row-Oriented | Column-Oriented |
|-----------|-------------|-----------------|
| **Best for** | Full-row operations | Analytical aggregations |
| **Write speed** | Fast (append whole rows) | Slower (must organize by column) |
| **Read (full row)** | Fast (data is co-located) | Slower (must assemble from columns) |
| **Read (few columns)** | Slow (reads all columns) | Fast (reads only needed columns) |
| **Compression** | Moderate (mixed data types per row) | Excellent (same type = better compression) |
| **Use cases** | Streaming, message queues, OLTP | Data warehouses, analytics, ML features |

> **Critical Insight:** Columnar formats compress dramatically better because adjacent values share the same data type. A column of timestamps compresses far better than alternating strings, integers, and timestamps in a row format. This is why Parquet files are typically 3-5x smaller than equivalent JSON or CSV.

---

## Apache Parquet

### Overview

- **Type:** Columnar
- **Created by:** Cloudera + Twitter (2013), Apache project
- **Ecosystem:** Universal — Spark, Hive, Presto, BigQuery, Athena, Pandas, Polars
- **Strengths:** Predicate pushdown, column pruning, rich statistics, excellent compression

### File Structure

```
Parquet File:
┌─────────────────────────────────┐
│         Magic Number             │
├─────────────────────────────────┤
│  Row Group 1                     │
│  ┌────────────────────────────┐ │
│  │ Column Chunk: id           │ │
│  │  └─ Page 1 (data)         │ │
│  │  └─ Page 2 (data)         │ │
│  │ Column Chunk: name         │ │
│  │  └─ Page 1 (data)         │ │
│  │ Column Chunk: salary       │ │
│  │  └─ Page 1 (data)         │ │
│  └────────────────────────────┘ │
├─────────────────────────────────┤
│  Row Group 2                     │
│  └─ (same structure)            │
├─────────────────────────────────┤
│  Footer                          │
│  ├─ Schema                       │
│  ├─ Row group metadata           │
│  ├─ Column statistics            │
│  │   (min, max, null_count)      │
│  └─ Column chunk offsets         │
├─────────────────────────────────┤
│  Footer Length                    │
│  Magic Number                    │
└─────────────────────────────────┘
```

### Key Features

**Predicate Pushdown:**
```sql
-- Query: WHERE salary > 100000
-- Parquet checks column statistics in footer:
-- Row Group 1: salary min=50000, max=95000 → SKIP (no rows match)
-- Row Group 2: salary min=80000, max=150000 → READ (may contain matches)
```

**Column Pruning:**
```sql
-- Query: SELECT name, salary FROM employees
-- Only reads the 'name' and 'salary' column chunks
-- Skips 'id', 'age', and all other columns entirely
```

**Nested Data Support:**
```python
# Parquet supports nested structures (maps, arrays, structs)
import pyarrow as pa
import pyarrow.parquet as pq

schema = pa.schema([
    ('user_id', pa.int64()),
    ('name', pa.string()),
    ('addresses', pa.list_(pa.struct([
        ('street', pa.string()),
        ('city', pa.string()),
        ('zip', pa.string()),
    ]))),
])
```

### Writing Parquet (Python)

```python
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq

# From Pandas
df = pd.DataFrame({'id': [1,2,3], 'name': ['a','b','c'], 'value': [10,20,30]})
df.to_parquet('output.parquet', engine='pyarrow', compression='snappy')

# With fine-grained control
table = pa.Table.from_pandas(df)
pq.write_table(
    table,
    'output.parquet',
    compression='zstd',
    row_group_size=1_000_000,  # Rows per row group
    use_dictionary=True,       # Dictionary encoding for low-cardinality columns
)

# Reading specific columns (column pruning)
df = pd.read_parquet('output.parquet', columns=['name', 'value'])

# Reading with filter (predicate pushdown)
df = pd.read_parquet('output.parquet', filters=[('value', '>', 15)])
```

---

## Apache ORC

### Overview

- **Type:** Columnar
- **Created by:** Hortonworks (2013), for Hive ecosystem
- **Ecosystem:** Hive, Presto, Spark (with adapter), Flink
- **Strengths:** Lightweight indexes, ACID support in Hive, excellent compression

### ORC vs Parquet

| Dimension | ORC | Parquet |
|-----------|-----|---------|
| **Origin** | Hive-specific | Language/engine agnostic |
| **Ecosystem** | Best in Hive/Presto | Universal (Spark, BigQuery, Athena, etc.) |
| **Indexes** | Built-in bloom filters + stripe-level stats | Row group statistics (similar) |
| **ACID** | Supports ACID in Hive | No native ACID (use Delta/Iceberg) |
| **Nested types** | Supported | Better nested type support |
| **Compression** | Slightly better for some workloads | Comparable |
| **Industry adoption** | Declining (Hive declining) | Growing (universal standard) |

### ORC File Structure

```
ORC File:
┌──────────────────────────────┐
│  Stripe 1 (≈250MB)           │
│  ├─ Index Data (per column)  │
│  │   min, max, bloom filter  │
│  ├─ Row Data (columnar)      │
│  └─ Stripe Footer            │
├──────────────────────────────┤
│  Stripe 2                     │
│  └─ (same structure)         │
├──────────────────────────────┤
│  File Footer                  │
│  ├─ Schema                   │
│  ├─ Stripe locations         │
│  └─ Column statistics        │
├──────────────────────────────┤
│  Postscript                   │
│  └─ Compression, version     │
└──────────────────────────────┘
```

> **Critical Insight:** If you're building a new data platform today, Parquet is the safe default. ORC is still strong in Hive-heavy environments, but Parquet has won the ecosystem war — it's supported by every major tool (Spark, BigQuery, Snowflake, Athena, Pandas, Polars, DuckDB, Redshift Spectrum). Choose ORC only if you're deeply invested in Hive with ACID requirements.

---

## Apache Avro

### Overview

- **Type:** Row-oriented
- **Created by:** Doug Cutting (Hadoop creator), 2009
- **Ecosystem:** Kafka, Hadoop, Spark, schema registry
- **Strengths:** Schema evolution, compact binary encoding, schema registry integration

### Key Characteristics

| Feature | Description |
|---------|-------------|
| **Row-oriented** | Stores complete records together |
| **Schema in file** | Schema embedded in header (self-describing) |
| **Rich schema** | Supports complex types, defaults, documentation |
| **Schema evolution** | Forward and backward compatible changes |
| **Compact binary** | Smaller than JSON, fast serialization |
| **Splittable** | Can be split for parallel processing |

### Schema Definition

```json
{
  "type": "record",
  "name": "UserEvent",
  "namespace": "com.company.events",
  "fields": [
    {"name": "user_id", "type": "long"},
    {"name": "event_type", "type": "string"},
    {"name": "timestamp", "type": "long", "logicalType": "timestamp-millis"},
    {"name": "properties", "type": {"type": "map", "values": "string"}},
    {"name": "device_type", "type": ["null", "string"], "default": null},
    {"name": "app_version", "type": "string", "default": "unknown"}
  ]
}
```

### Schema Evolution Rules

| Change | Forward Compatible | Backward Compatible |
|--------|-------------------|---------------------|
| Add field with default | Yes | Yes |
| Remove field with default | Yes | Yes |
| Add field without default | Yes | No |
| Remove field without default | No | Yes |
| Rename field | No | No |
| Change field type | Sometimes (promotions only) | Sometimes |

```json
// Version 1: Original schema
{"name": "user_id", "type": "long"}
{"name": "name", "type": "string"}

// Version 2: Add optional field (backward + forward compatible)
{"name": "user_id", "type": "long"}
{"name": "name", "type": "string"}
{"name": "email", "type": ["null", "string"], "default": null}  // NEW

// Old readers can read new data (ignore unknown fields)
// New readers can read old data (use default for missing fields)
```

### Avro with Kafka (Schema Registry)

```python
from confluent_kafka import SerializingProducer
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer

# Schema Registry manages schema versions
schema_registry = SchemaRegistryClient({'url': 'http://schema-registry:8081'})

# Producer with Avro serialization
producer = SerializingProducer({
    'bootstrap.servers': 'kafka:9092',
    'value.serializer': AvroSerializer(schema_registry, schema_str),
})

# Benefits:
# 1. Schema validated at produce time (no bad data in Kafka)
# 2. Consumers can evolve independently
# 3. Schema changes tracked and versioned
```

> **Critical Insight:** Avro is the standard for streaming/messaging (Kafka) because of schema evolution and compact encoding. Parquet is the standard for analytics storage. A common pattern: produce events as Avro into Kafka, then convert to Parquet when landing in the data lake for analytical queries.

---

## JSON and CSV

### JSON

| Aspect | Details |
|--------|---------|
| **Type** | Row-oriented, text-based |
| **Schema** | Schema-less (self-describing per record) |
| **Compression** | Poor (verbose syntax — keys repeated in every record) |
| **Human-readable** | Yes |
| **Nested data** | Native support |
| **Use cases** | APIs, configs, small datasets, debugging |
| **Limitations** | Large, slow to parse, no type enforcement |

```json
{"user_id": 1, "name": "Alice", "age": 30, "tags": ["premium", "active"]}
{"user_id": 2, "name": "Bob", "age": 25, "tags": ["free"]}
```

### JSON Lines (JSONL / NDJSON)

```
# One JSON object per line — splittable, streamable
{"user_id": 1, "name": "Alice", "event": "login"}
{"user_id": 2, "name": "Bob", "event": "purchase"}
{"user_id": 1, "name": "Alice", "event": "logout"}
```

### CSV

| Aspect | Details |
|--------|---------|
| **Type** | Row-oriented, text-based |
| **Schema** | Header row only (no types) |
| **Compression** | Moderate (less verbose than JSON) |
| **Human-readable** | Yes |
| **Nested data** | Not supported |
| **Use cases** | Spreadsheets, simple exports, seeds |
| **Limitations** | No type info, delimiter conflicts, no nested data |

### When to Use Text Formats

| Format | Use When |
|--------|----------|
| **JSON** | API responses, configs, schema flexibility needed, debugging |
| **JSONL** | Streaming ingestion, log files, line-by-line processing |
| **CSV** | Excel/spreadsheet interop, simple flat data, dbt seeds, quick exports |

### Converting to Efficient Formats

```python
import pandas as pd

# Read messy formats
df = pd.read_json('events.jsonl', lines=True)
df = pd.read_csv('export.csv')

# Write to efficient format for analytics
df.to_parquet('events.parquet', compression='zstd')

# Spark: Convert JSON landing zone to Parquet
spark.read.json("s3://raw/events/").write.parquet("s3://lake/events/")
```

---

## Compression Codecs

### Comparison

| Codec | Compression Ratio | Speed (Compress) | Speed (Decompress) | Splittable | Best For |
|-------|-------------------|-------------------|--------------------|------------|----------|
| **None** | 1x | N/A | N/A | Yes | When CPU is bottleneck |
| **Snappy** | ~2-4x | Very fast | Very fast | Yes (in Parquet) | Default for hot data |
| **LZ4** | ~2-4x | Very fast | Very fast | Yes (in Parquet) | Low-latency reads |
| **ZSTD** | ~4-8x | Fast | Fast | Yes (in Parquet) | Best balance (ratio + speed) |
| **Gzip** | ~4-7x | Slow | Moderate | No (raw), Yes (in Parquet) | Archival, maximum compatibility |
| **Brotli** | ~5-8x | Very slow | Moderate | No | Web assets, not data |

### Choosing a Codec

```
                    Compression Ratio
                    ▲
                    │
            ZSTD ●  │  ● Gzip
                    │
                    │
          Snappy ●  │  ● LZ4
                    │
                    └──────────────────────▶ Speed
                    Slow                   Fast
```

> **Critical Insight:** ZSTD (Zstandard) has emerged as the optimal choice for most data engineering workloads. It offers compression ratios close to Gzip but with decompression speeds approaching Snappy. Unless you have a specific reason (legacy compatibility = Gzip, extreme latency sensitivity = Snappy/LZ4), default to ZSTD.

### Codec in Practice

```python
# Spark: Set compression for Parquet output
df.write.option("compression", "zstd").parquet("s3://lake/data/")

# dbt (BigQuery): Compression is automatic
# dbt (Snowflake): Compression is automatic (auto-compressed)

# PyArrow: Explicit codec selection
import pyarrow.parquet as pq
pq.write_table(table, 'data.parquet', compression='zstd')

# Pandas
df.to_parquet('data.parquet', compression='zstd')
```

---

## Format Comparison Table

| Dimension | Parquet | ORC | Avro | JSON | CSV |
|-----------|---------|-----|------|------|-----|
| **Layout** | Columnar | Columnar | Row | Row | Row |
| **Encoding** | Binary | Binary | Binary | Text | Text |
| **Human-readable** | No | No | No | Yes | Yes |
| **Schema** | In footer | In footer | In header | None | Header row |
| **Schema evolution** | Limited | Limited | Excellent | N/A | N/A |
| **Nested types** | Yes | Yes | Yes | Yes | No |
| **Compression** | Excellent | Excellent | Good | Poor | Moderate |
| **Column pruning** | Yes | Yes | No | No | No |
| **Predicate pushdown** | Yes | Yes | No | No | No |
| **Splittable** | Yes | Yes | Yes | JSONL only | With headers |
| **Best for** | Analytics/DW | Hive/Presto | Kafka/streaming | APIs/debug | Spreadsheets |
| **Ecosystem** | Universal | Hive-centric | Kafka-centric | Universal | Universal |
| **Typical size (vs JSON)** | 10-20% | 10-20% | 30-50% | 100% | 60-80% |

---

## When to Use Which

### Decision Framework

| Scenario | Recommended Format | Why |
|----------|-------------------|-----|
| Data lake / analytical storage | **Parquet + ZSTD** | Column pruning, predicate pushdown, compression |
| Kafka / event streaming | **Avro** | Schema evolution, compact, schema registry |
| Hive warehouse (existing) | **ORC** | Optimized for Hive, ACID support |
| API request/response | **JSON** | Human-readable, flexible schema |
| Quick data exchange | **CSV** | Universal, spreadsheet-compatible |
| ML feature store | **Parquet** | Fast column reads for feature vectors |
| Log ingestion | **JSONL** | Streamable, line-by-line, schema flexible |
| Data lake table format | **Parquet** (via Delta/Iceberg) | Standard underlayer for table formats |
| Archival (long-term) | **Parquet + ZSTD** | Compact, self-describing, widely supported |
| dbt seeds (lookup tables) | **CSV** | dbt natively supports CSV seeds |

### The Common Architecture

```
Sources          Landing Zone       Data Lake           Warehouse
─────────        ────────────       ─────────           ─────────
APIs → JSON      → Convert to      → Parquet + ZSTD    → Native
Kafka → Avro       Parquet            (partitioned)       (columnar)
DBs → CDC/Avro
CSVs → CSV       
```

---

## Interview Talking Points

> **"Why would you choose Parquet over CSV for a data lake?"**
>
> "Parquet is columnar, so analytical queries that touch a few columns out of many read dramatically less data — often 10-50x less I/O. It has built-in compression (typically 5-10x smaller than CSV), predicate pushdown via column statistics (skip entire row groups that don't match filters), and embedded schema (no guessing types). At scale, this means queries run in seconds instead of minutes and storage costs are a fraction of CSV."

> **"When would you use Avro instead of Parquet?"**
>
> "Avro for streaming, Parquet for analytics. Avro is row-oriented and optimized for write-heavy, record-at-a-time workloads like Kafka events. Its killer feature is schema evolution — you can add fields with defaults and both old and new consumers continue working. It also integrates with schema registries for governance. Once data lands in the lake for analytical queries, I convert it to Parquet for columnar efficiency."

> **"Explain predicate pushdown and column pruning."**
>
> "Column pruning means the engine reads only the columns referenced in the query — in Parquet, columns are stored separately, so unneeded ones are never loaded from disk. Predicate pushdown means filter conditions are evaluated against column statistics (min/max per row group) before reading the data — entire row groups are skipped if their statistics prove no rows match the filter. Together, these can reduce I/O by 90%+ on wide tables with selective queries."

---

## Common Mistakes

| ❌ Mistake | ✅ Best Practice |
|-----------|-----------------|
| Storing analytics data as JSON in the lake | Convert to Parquet on landing for 10x better performance |
| Using CSV for large datasets (millions of rows) | Use Parquet — type safety, compression, column pruning |
| Choosing ORC for a new platform (not Hive-based) | Use Parquet — it's the universal standard |
| No compression on Parquet files | Always compress (ZSTD default, Snappy for speed) |
| Extremely small Parquet files (1 MB each) | Target 128MB-1GB files for optimal performance |
| Extremely large Parquet files (10+ GB) | Keep under 1GB for parallelism and fault tolerance |
| Using Parquet for Kafka messages | Use Avro for streaming — row-oriented, schema evolution |
| Not setting row group size appropriately | 128MB row groups balance column pruning vs overhead |
| Ignoring schema evolution in Avro | Define defaults on all optional fields for compatibility |
| Storing timestamps as strings in any format | Use native timestamp types — saves space, enables pushdown |

---

## Rapid-Fire Q&A

**Q1: What is the fundamental difference between row and columnar formats?**
> Row formats store entire records together (good for full-row reads). Columnar formats store each column separately (good for reading specific columns in analytical queries).

**Q2: Why does Parquet compress better than JSON?**
> Columnar storage groups values of the same type together. A column of integers compresses far better than alternating integers, strings, and booleans in a row format. Plus, Parquet uses dictionary encoding for repeated values.

**Q3: What is predicate pushdown?**
> Using column statistics (min/max per row group) to skip row groups that cannot contain matching rows, avoiding reading data that will be filtered out anyway.

**Q4: When would you choose Avro over Parquet?**
> For streaming/messaging (Kafka) where schema evolution, compact row serialization, and schema registry integration matter more than columnar query performance.

**Q5: What compression codec should be the default for Parquet?**
> ZSTD — it offers the best balance of compression ratio and speed. Use Snappy only if decompression latency is critical.

**Q6: What is column pruning?**
> Reading only the columns referenced in a query. In Parquet, unused columns are never loaded from storage, dramatically reducing I/O for queries on wide tables.

**Q7: Why is ORC less popular than Parquet today?**
> ORC was optimized for Hive. Parquet was designed to be engine-agnostic and is supported by every major tool (Spark, BigQuery, Athena, Snowflake, Pandas, DuckDB). Ecosystem breadth won.

**Q8: What is a row group in Parquet?**
> A horizontal partition of the file containing a subset of rows. Each row group stores its columns independently with per-column statistics. Typical size: 128MB.

**Q9: How does schema evolution work in Avro?**
> New fields with defaults can be added without breaking existing consumers. Old consumers ignore unknown fields. New consumers use defaults for missing fields. The schema registry tracks versions.

**Q10: What's the ideal Parquet file size?**
> 128MB to 1GB. Too small (< 10MB) creates excessive metadata overhead and reduces parallelism efficiency. Too large (> 5GB) hurts fault tolerance and parallelism.

---

## ASCII Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════╗
║              DATA FORMATS CHEAT SHEET                             ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  FORMAT SELECTION:                                               ║
║  ┌──────────────────────────────────────────────────┐            ║
║  │ Analytics / Data Lake?  → PARQUET                │            ║
║  │ Kafka / Streaming?      → AVRO                   │            ║
║  │ Hive + ACID needed?     → ORC                    │            ║
║  │ API / human-readable?   → JSON                   │            ║
║  │ Spreadsheet / simple?   → CSV                    │            ║
║  └──────────────────────────────────────────────────┘            ║
║                                                                  ║
║  ROW vs COLUMNAR:                                                ║
║  Row:     [R1: all cols] [R2: all cols] [R3: all cols]           ║
║  Columnar:[Col1: all rows] [Col2: all rows] [Col3: all rows]    ║
║                                                                  ║
║  PARQUET OPTIMIZATIONS:                                          ║
║  ┌────────────────────────────────────────────────┐              ║
║  │ Column Pruning:  Read only referenced columns  │              ║
║  │ Predicate Pushdown: Skip row groups via stats  │              ║
║  │ Dictionary Encoding: Compress repeated values  │              ║
║  │ Page-level stats: Fine-grained filtering       │              ║
║  └────────────────────────────────────────────────┘              ║
║                                                                  ║
║  COMPRESSION PICK:                                               ║
║  Default:        ZSTD (best balance)                             ║
║  Max speed:      Snappy or LZ4                                   ║
║  Max ratio:      GZIP (but slow compress)                        ║
║  Legacy compat:  GZIP                                            ║
║                                                                  ║
║  TYPICAL SIZES (same data):                                      ║
║  JSON (raw):     1000 MB                                         ║
║  CSV:            600-800 MB                                      ║
║  Avro:           300-500 MB                                      ║
║  Parquet+ZSTD:   100-200 MB   ← 5-10x smaller than JSON         ║
║                                                                  ║
║  COMMON PATTERN:                                                 ║
║  Source → Avro (Kafka) → Parquet (Lake) → DW (native columnar)  ║
║                                                                  ║
║  FILE SIZE GUIDELINES (Parquet):                                 ║
║  Too small: < 10 MB  (overhead, no parallelism benefit)          ║
║  Ideal:     128 MB - 1 GB                                        ║
║  Too large: > 5 GB   (fault tolerance, memory issues)            ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 20 of 45*
