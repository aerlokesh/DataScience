# 🎯 Topic 19: Incremental Processing and Idempotency

> *The art of processing only what's changed — designing pipelines that are efficient, correct, and safe to re-run without creating duplicates or corruption*

---

## 📋 Table of Contents

1. [Core Concepts](#core-concepts)
2. [Full Refresh vs Incremental](#full-refresh-vs-incremental)
3. [Watermarks and Change Tracking](#watermarks-and-change-tracking)
4. [Merge and Upsert Patterns](#merge-and-upsert-patterns)
5. [Handling Late-Arriving Data](#handling-late-arriving-data)
6. [Idempotent Writes](#idempotent-writes)
7. [Deduplication Strategies](#deduplication-strategies)
8. [Partition-Level Refreshes](#partition-level-refreshes)
9. [Exactly-Once Semantics](#exactly-once-semantics)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Core Concepts

### Why Incremental Processing?

As data volumes grow, full table refreshes become expensive and slow. Incremental processing handles only new or changed records — dramatically reducing cost, time, and resource usage.

```
Full Refresh:  Process 100M rows every day → 100M rows/day compute
Incremental:   Process 500K new rows/day   → 500K rows/day compute
                                              (200x more efficient)
```

> **Critical Insight:** Incremental processing is an optimization with correctness trade-offs. It's faster and cheaper, but introduces complexity: what if data arrives late? What if records are updated retroactively? What if the watermark is wrong? Every incremental design must answer these questions explicitly.

### Key Terminology

| Term | Definition |
|------|-----------|
| **Watermark** | The timestamp below which all data is assumed to have arrived |
| **High-water mark** | The maximum timestamp processed in the last successful run |
| **Upsert / Merge** | Insert new records, update existing ones (based on key) |
| **Idempotent** | Safe to re-run — same input always produces same output |
| **Late-arriving data** | Records that arrive after their time window was "closed" |
| **Change Data Capture (CDC)** | Detecting and capturing changes in source systems |
| **Lookback window** | Re-processing a buffer of recent data to catch late arrivals |

---

## Full Refresh vs Incremental

### Comparison

| Dimension | Full Refresh | Incremental |
|-----------|-------------|-------------|
| **What's processed** | Entire dataset every run | Only new/changed records |
| **Compute cost** | High (scales with total data) | Low (scales with change volume) |
| **Correctness** | Always correct (latest logic applied to all data) | Correct only if change detection is complete |
| **Late data handling** | Naturally handles (reprocesses everything) | Must explicitly design for |
| **Backfill** | Just re-run | May need special backfill mode |
| **Schema changes** | Applied to all data immediately | Historical data may have old schema |
| **Complexity** | Simple | Higher (watermarks, merge logic, dedup) |
| **Recovery from failure** | Trivial (re-run) | Must track state, handle partial writes |

### When to Use Each

**Full Refresh** (prefer for):
- Small to medium tables (< 10M rows)
- Dimension tables (slow-changing, need latest state)
- When source doesn't provide reliable timestamps
- After logic changes that must apply retroactively
- Reference/lookup tables

**Incremental** (prefer for):
- Large fact tables (billions of rows)
- Append-only event data (logs, clickstreams)
- When compute cost is a primary concern
- High-frequency processing (hourly or more)
- When source provides reliable change indicators

> **Critical Insight:** Start with full refresh. Move to incremental ONLY when full refresh becomes too slow or expensive. Incremental is an optimization, not a starting point. Premature optimization here introduces bugs that are hard to detect (missing data is invisible unless you look for it).

---

## Watermarks and Change Tracking

### High-Water Mark Pattern

```python
# Track the maximum processed timestamp
# On each run, only process records after this mark

def get_high_water_mark(table_name):
    """Retrieve the last successfully processed timestamp."""
    result = db.execute(f"""
        SELECT MAX(processed_through) 
        FROM pipeline_metadata 
        WHERE table_name = '{table_name}'
    """)
    return result or datetime(2000, 1, 1)  # Default: beginning of time

def update_high_water_mark(table_name, new_mark):
    """Update after successful processing."""
    db.execute(f"""
        INSERT INTO pipeline_metadata (table_name, processed_through, updated_at)
        VALUES ('{table_name}', '{new_mark}', CURRENT_TIMESTAMP)
        ON CONFLICT (table_name) DO UPDATE SET 
            processed_through = '{new_mark}',
            updated_at = CURRENT_TIMESTAMP
    """)
```

### SQL Watermark Implementation

```sql
-- dbt incremental model with high-water mark
{{ config(materialized='incremental', unique_key='event_id') }}

SELECT
    event_id,
    user_id,
    event_type,
    event_timestamp,
    properties
FROM {{ source('events', 'raw_events') }}

{% if is_incremental() %}
    WHERE event_timestamp > (
        SELECT MAX(event_timestamp) FROM {{ this }}
    )
{% endif %}
```

### Change Data Capture (CDC) Methods

| Method | How It Works | Pros | Cons |
|--------|-------------|------|------|
| **Timestamp-based** | Filter on `updated_at > last_run` | Simple, widely applicable | Misses deletes, needs reliable timestamp |
| **Log-based CDC** | Read database transaction log | Captures all changes (insert/update/delete) | Complex setup, DB-specific |
| **Trigger-based** | DB triggers write to change table | Captures all changes | Performance impact on source |
| **Diff-based** | Compare current snapshot to previous | Works without timestamps | Expensive for large tables |
| **Sequence/version** | Filter on auto-increment ID or version | Simple for append-only | Misses updates to existing rows |

### Log-Based CDC (Debezium Example)

```json
// Kafka message from Debezium CDC
{
  "op": "u",          // Operation: c=create, u=update, d=delete
  "before": {         // Previous state
    "customer_id": 101,
    "plan": "free",
    "mrr": 0
  },
  "after": {          // New state
    "customer_id": 101,
    "plan": "pro",
    "mrr": 49
  },
  "source": {
    "ts_ms": 1706745600000,
    "table": "customers"
  }
}
```

---

## Merge and Upsert Patterns

### MERGE INTO (SQL Standard)

```sql
-- The gold standard for idempotent incremental loading
MERGE INTO target_table AS target
USING source_table AS source
ON target.id = source.id

WHEN MATCHED AND source.updated_at > target.updated_at THEN
    UPDATE SET
        target.name = source.name,
        target.status = source.status,
        target.amount = source.amount,
        target.updated_at = source.updated_at

WHEN NOT MATCHED THEN
    INSERT (id, name, status, amount, updated_at)
    VALUES (source.id, source.name, source.status, source.amount, source.updated_at);
```

### Warehouse-Specific Merge Syntax

```sql
-- Snowflake MERGE
MERGE INTO analytics.dim_customers AS target
USING staging.stg_customers AS source
ON target.customer_id = source.customer_id
WHEN MATCHED THEN
    UPDATE SET target.name = source.name, target.updated_at = source.updated_at
WHEN NOT MATCHED THEN
    INSERT (customer_id, name, created_at, updated_at)
    VALUES (source.customer_id, source.name, source.created_at, source.updated_at);

-- BigQuery MERGE
MERGE analytics.dim_customers AS target
USING staging.stg_customers AS source
ON target.customer_id = source.customer_id
WHEN MATCHED THEN
    UPDATE SET name = source.name, updated_at = source.updated_at
WHEN NOT MATCHED THEN
    INSERT ROW;

-- Redshift (no native MERGE — use delete + insert)
BEGIN;
DELETE FROM analytics.dim_customers
USING staging.stg_customers
WHERE analytics.dim_customers.customer_id = staging.stg_customers.customer_id;

INSERT INTO analytics.dim_customers
SELECT * FROM staging.stg_customers;
COMMIT;
```

### dbt Incremental Strategies

```sql
-- Strategy: merge (default for Snowflake, BigQuery)
{{ config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge'
) }}

-- Strategy: delete+insert (for Redshift, also works on others)
{{ config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='delete+insert'
) }}

-- Strategy: insert_overwrite (BigQuery partitioned tables)
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={'field': 'order_date', 'data_type': 'date'}
) }}
```

> **Critical Insight:** MERGE is the most versatile pattern but not always the best. For append-only data (events, logs), a simple INSERT with dedup is faster. For partition-aligned data, DELETE partition + INSERT is simpler and avoids merge conflicts. Choose the pattern based on your data's mutation characteristics.

---

## Handling Late-Arriving Data

### The Problem

```
Timeline:
12:00 - Events for hour 11:00 processed ✓
12:05 - Late event with timestamp 10:45 arrives
        → This event was MISSED because watermark already passed 11:00

Result: Incomplete aggregations, inaccurate metrics
```

### Strategies for Late Data

| Strategy | How It Works | Trade-off |
|----------|-------------|-----------|
| **Lookback window** | Re-process last N hours/days | Extra compute, but catches late data |
| **Grace period** | Delay window closing by N minutes | Adds latency to final results |
| **Periodic full refresh** | Weekly/monthly full rebuild | Expensive but catches everything |
| **Late data partition** | Write late data to separate table, reconcile | Complex but precise |
| **Watermark with buffer** | `WHERE timestamp > max - INTERVAL '3 days'` | Re-processes some data unnecessarily |

### Lookback Window Implementation

```sql
-- Instead of: WHERE timestamp > max(timestamp)
-- Use: WHERE timestamp > max(timestamp) - INTERVAL '3 days'

{{ config(materialized='incremental', unique_key='event_id') }}

SELECT
    event_id,
    user_id,
    event_type,
    event_timestamp
FROM {{ source('events', 'raw_events') }}

{% if is_incremental() %}
    -- Lookback window: re-process last 3 days to catch late arrivals
    WHERE event_timestamp >= (
        SELECT MAX(event_timestamp) - INTERVAL '3 days' 
        FROM {{ this }}
    )
{% endif %}
```

### Late Data Detection

```sql
-- Monitor: How much late data are we getting?
SELECT
    DATE(loaded_at) AS load_date,
    DATE(event_timestamp) AS event_date,
    DATEDIFF('day', event_timestamp, loaded_at) AS days_late,
    COUNT(*) AS late_event_count
FROM raw.events
WHERE loaded_at > event_timestamp + INTERVAL '1 day'
GROUP BY 1, 2, 3
ORDER BY days_late DESC;

-- Alert if late data exceeds threshold
-- This helps calibrate your lookback window size
```

> **Critical Insight:** The lookback window size should be data-driven. Analyze your historical late-arriving data distribution. If 99% of late data arrives within 2 days, a 3-day lookback gives you a safety margin. If you use MERGE/UPSERT with the lookback, re-processing is idempotent — it just updates existing records to the same values.

---

## Idempotent Writes

### Definition and Importance

An **idempotent write** produces the same result regardless of how many times it's executed. This is critical for:
- Safe retries after failure
- Safe backfills (re-running historical dates)
- Overlapping schedules (if two runs process the same data)

### Patterns for Idempotency

| Pattern | Implementation | Best For |
|---------|---------------|----------|
| **Truncate + Insert** | Delete all → Insert all | Small tables, full refresh |
| **Partition Overwrite** | Delete partition → Insert partition | Date-partitioned facts |
| **MERGE / UPSERT** | Insert or update based on key | Dimension tables, mutable data |
| **Deterministic output path** | Write to `s3://bucket/date=2024-01-15/` | File-based pipelines |
| **Conditional insert** | `INSERT ... WHERE NOT EXISTS` | Append-only with dedup |

### Idempotent vs Non-Idempotent Examples

```sql
-- NON-IDEMPOTENT: Running twice creates duplicates
INSERT INTO fact_orders
SELECT * FROM staging_orders WHERE order_date = '2024-01-15';

-- IDEMPOTENT Option 1: Delete + Insert (partition-level)
DELETE FROM fact_orders WHERE order_date = '2024-01-15';
INSERT INTO fact_orders
SELECT * FROM staging_orders WHERE order_date = '2024-01-15';

-- IDEMPOTENT Option 2: MERGE on unique key
MERGE INTO fact_orders AS target
USING staging_orders AS source
ON target.order_id = source.order_id
WHEN MATCHED THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT ...;

-- IDEMPOTENT Option 3: INSERT with dedup guard
INSERT INTO fact_orders
SELECT * FROM staging_orders s
WHERE s.order_date = '2024-01-15'
  AND NOT EXISTS (
    SELECT 1 FROM fact_orders f WHERE f.order_id = s.order_id
  );
```

### File-Based Idempotency

```python
import hashlib
from datetime import date

def write_output(df, base_path, partition_date):
    """
    Idempotent file write: same input always writes to same path.
    Re-running overwrites the same file (no duplicates).
    """
    output_path = f"{base_path}/dt={partition_date}/data.parquet"
    
    # Overwrite mode ensures idempotency
    df.write.mode("overwrite").parquet(output_path)
    
    # Alternatively: write to a deterministic temp path, then atomic rename
    temp_path = f"{base_path}/tmp/{partition_date}_{run_id}/data.parquet"
    df.write.mode("overwrite").parquet(temp_path)
    atomic_rename(temp_path, output_path)
```

---

## Deduplication Strategies

### Why Duplicates Happen

1. **At-least-once delivery** — Message brokers deliver duplicates on retry
2. **Pipeline retries** — Failed tasks re-run and re-process some data
3. **Overlapping windows** — Two incremental runs process overlapping time ranges
4. **Source system bugs** — Source emits the same event twice

### Dedup Strategies

```sql
-- Strategy 1: ROW_NUMBER dedup (most common)
WITH ranked AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY event_id 
            ORDER BY _loaded_at DESC  -- Keep the latest version
        ) AS row_num
    FROM raw.events
)
SELECT * FROM ranked WHERE row_num = 1;

-- Strategy 2: QUALIFY (Snowflake, BigQuery)
SELECT *
FROM raw.events
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY event_id ORDER BY _loaded_at DESC
) = 1;

-- Strategy 3: DISTINCT (only for truly identical rows)
SELECT DISTINCT
    event_id, user_id, event_type, event_timestamp
FROM raw.events;

-- Strategy 4: GROUP BY with aggregation
SELECT
    event_id,
    user_id,
    MAX(event_timestamp) AS event_timestamp,
    MAX(_loaded_at) AS latest_load
FROM raw.events
GROUP BY event_id, user_id;
```

### Dedup in the Data Pipeline

```
Source (may contain dupes)
    │
    ▼
Raw Layer (preserve all, including dupes)
    │
    ▼
Staging Layer (dedup here — single source of truth)
    │
    ▼
Marts Layer (inherits deduped data from staging)
```

```sql
-- Staging model with built-in dedup
-- models/staging/stg_events.sql

WITH source AS (
    SELECT * FROM {{ source('app', 'events') }}
),

deduplicated AS (
    SELECT
        event_id,
        user_id,
        event_type,
        event_timestamp,
        properties,
        ROW_NUMBER() OVER (
            PARTITION BY event_id
            ORDER BY _loaded_at DESC
        ) AS _row_num
    FROM source
)

SELECT
    event_id,
    user_id,
    event_type,
    event_timestamp,
    properties
FROM deduplicated
WHERE _row_num = 1
```

> **Critical Insight:** Deduplicate at the LOWEST layer possible (staging), not at the mart layer. This ensures all downstream models benefit from clean data. The dedup key must be a natural/business key (not a synthetic one) — you need to know what constitutes "the same event" from a business perspective.

---

## Partition-Level Refreshes

### The Pattern

Instead of row-level MERGE, overwrite entire partitions:

```sql
-- Step 1: Determine which partitions have new data
SELECT DISTINCT DATE(event_timestamp) AS affected_date
FROM staging.new_events;

-- Step 2: For each affected partition, full overwrite
-- BigQuery: INSERT OVERWRITE (partition-level)
INSERT OVERWRITE analytics.fct_events
PARTITION (event_date = '2024-01-15')
SELECT *
FROM staging.stg_events
WHERE DATE(event_timestamp) = '2024-01-15';

-- Snowflake: DELETE + INSERT within transaction
BEGIN;
DELETE FROM analytics.fct_events WHERE event_date = '2024-01-15';
INSERT INTO analytics.fct_events
SELECT * FROM staging.stg_events WHERE DATE(event_timestamp) = '2024-01-15';
COMMIT;
```

### Why Partition-Level Refreshes?

| Advantage | Explanation |
|-----------|-------------|
| **Naturally idempotent** | Overwriting a partition with the same data = same result |
| **Handles late data** | Late data for a partition triggers a full partition rebuild |
| **Simpler than MERGE** | No need to define match conditions or update logic |
| **Better performance** | Partition operations are optimized in warehouses |
| **Atomic** | Either the whole partition is replaced or nothing changes |

### dbt: insert_overwrite Strategy

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={
        'field': 'event_date',
        'data_type': 'date',
        'granularity': 'day'
    }
) }}

SELECT
    DATE(event_timestamp) AS event_date,
    event_id,
    user_id,
    event_type,
    amount
FROM {{ ref('stg_events') }}

{% if is_incremental() %}
    -- Only rebuild partitions that have new data
    WHERE DATE(event_timestamp) IN (
        SELECT DISTINCT DATE(event_timestamp)
        FROM {{ ref('stg_events') }}
        WHERE _loaded_at > (SELECT MAX(_loaded_at) FROM {{ this }})
    )
{% endif %}
```

---

## Exactly-Once Semantics

### End-to-End Exactly-Once

True exactly-once requires coordination across the entire pipeline:

```
Source → Extract → Load → Transform → Serve

Each step must guarantee:
1. No data is lost (at-least-once delivery)
2. No data is duplicated (idempotent processing)
3. State is consistent (atomic commits)
```

### Implementation Patterns

```python
# Pattern: Transactional outbox
# Ensure extraction + metadata update are atomic

def process_batch(batch_id):
    """Exactly-once processing with transactional metadata."""
    
    # Check if already processed (idempotency check)
    if is_already_processed(batch_id):
        logger.info(f"Batch {batch_id} already processed, skipping")
        return
    
    # Process the data
    result = transform(get_batch_data(batch_id))
    
    # Atomic: write result + mark as processed
    with database.transaction():
        write_to_target(result)
        mark_as_processed(batch_id)  # Same transaction as the write
    
    # If transaction fails, neither the write nor the mark persists
    # Safe to retry — idempotency check will prevent double-processing
```

### Kafka Exactly-Once (Transactional)

```python
from confluent_kafka import Producer

producer = Producer({
    'bootstrap.servers': 'kafka:9092',
    'enable.idempotence': True,          # Dedup at broker level
    'transactional.id': 'my-producer-1', # Enable transactions
})

producer.init_transactions()

try:
    producer.begin_transaction()
    
    # All messages in this transaction are atomic
    producer.produce('output-topic', key=key, value=value)
    producer.produce('output-topic', key=key2, value=value2)
    
    # Commit consumer offsets + produced messages atomically
    producer.send_offsets_to_transaction(consumer_offsets, consumer_group)
    producer.commit_transaction()
except Exception:
    producer.abort_transaction()
```

> **Critical Insight:** "Exactly-once" in practice usually means "at-least-once delivery with idempotent processing." True distributed exactly-once (across systems from different vendors) is nearly impossible. Instead, design each component to handle duplicates gracefully: UPSERT on unique keys, partition overwrites, or dedup tables.

---

## Interview Talking Points

> **"How do you design an incremental pipeline?"**
>
> "I start by identifying the change indicator — a reliable timestamp (`updated_at`), a sequence number, or CDC from the transaction log. Then I choose the merge strategy: UPSERT for mutable data, partition overwrite for time-series facts, or append with dedup for immutable events. I always include a lookback window to handle late-arriving data — typically 2-3 days based on observed lateness distribution. The entire pipeline is idempotent, meaning I can safely re-run any date without creating duplicates."

> **"How do you handle late-arriving data?"**
>
> "I analyze the historical distribution of data lateness first — how late does data typically arrive? Then I set a lookback window that covers 99% of late arrivals. For a daily pipeline, if 99% of late data arrives within 48 hours, I re-process the last 3 days on each run. Since my pipeline uses MERGE or partition overwrite, re-processing is idempotent. For the rare 1% that arrives very late, I run periodic full refreshes (weekly or monthly) to catch everything."

> **"What does idempotency mean and why is it important for data pipelines?"**
>
> "An idempotent pipeline produces the same output regardless of how many times it runs for the same input. This is crucial because pipelines fail and get retried, schedules overlap, and backfills re-process historical data. Without idempotency, these events create duplicates or corrupt data. I achieve idempotency through patterns like DELETE+INSERT (partition-level), MERGE on unique keys, or deterministic file paths that overwrite on re-run."

---

## Common Mistakes

| ❌ Mistake | ✅ Best Practice |
|-----------|-----------------|
| INSERT without dedup (creates duplicates on retry) | Use MERGE, DELETE+INSERT, or INSERT with NOT EXISTS |
| Watermark based on processing time, not event time | Use event time watermarks; processing time is unreliable |
| No lookback window for late data | Add a lookback (e.g., 3 days) based on observed lateness |
| Incremental on data without reliable timestamps | Use CDC or full refresh if timestamps are unreliable |
| Starting with incremental before proving full refresh is too slow | Start with full refresh; optimize to incremental when needed |
| Dedup at the mart layer instead of staging | Dedup at the lowest layer so all downstream models benefit |
| MERGE without handling deletes | If source deletes records, use CDC or periodic full refresh |
| No monitoring for incremental completeness | Compare row counts: incremental output vs full refresh periodically |
| Relying on auto-increment IDs as the sole watermark | IDs don't capture updates; combine with timestamps |
| Partition overwrite without transaction isolation | Use BEGIN/COMMIT to ensure atomic partition replacement |

---

## Rapid-Fire Q&A

**Q1: What is the difference between full refresh and incremental processing?**
> Full refresh reprocesses the entire dataset every run. Incremental only processes new or changed records, using watermarks or change tracking to identify what's new.

**Q2: What is a high-water mark?**
> The maximum timestamp (or sequence number) successfully processed in the last run. The next run processes only records above this mark.

**Q3: How does MERGE INTO work?**
> It compares source and target on a key. If the key exists in target, it updates the row. If the key doesn't exist, it inserts a new row. It's idempotent.

**Q4: What is a lookback window and why use one?**
> Re-processing a buffer of recent data (e.g., last 3 days) on each run to catch late-arriving records that arrived after the initial processing.

**Q5: How do you handle deletes in incremental pipelines?**
> Timestamps don't capture deletes. Use CDC (log-based) to detect deletes, or periodically run a full refresh to reconcile, or use soft deletes with an `is_deleted` flag.

**Q6: What makes a write operation idempotent?**
> It produces the same result regardless of execution count. DELETE+INSERT, MERGE, and deterministic file overwrites are idempotent. Plain INSERT is not.

**Q7: When should you use partition-level refresh vs row-level MERGE?**
> Partition-level for time-series data naturally aligned to date partitions. Row-level MERGE for dimension tables or data without clean partition alignment.

**Q8: What is Change Data Capture (CDC)?**
> A technique to detect changes (inserts, updates, deletes) in source systems, typically by reading the database transaction log. Tools: Debezium, AWS DMS.

**Q9: How do you detect if your incremental pipeline is missing data?**
> Periodically compare incremental output counts against a full refresh. Monitor for unexpected drops in row counts. Alert on freshness violations.

**Q10: What's the risk of a too-small lookback window?**
> Late-arriving data outside the window is permanently missed. The incremental table will have gaps that are invisible without active monitoring.

---

## ASCII Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════╗
║        INCREMENTAL PROCESSING & IDEMPOTENCY CHEAT SHEET          ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  DECISION: FULL REFRESH vs INCREMENTAL                           ║
║  ┌─────────────────────────────────────────────┐                 ║
║  │ Table < 10M rows?          → Full refresh   │                 ║
║  │ No reliable timestamp?     → Full refresh   │                 ║
║  │ Table > 100M rows?         → Incremental    │                 ║
║  │ Runs every hour?           → Incremental    │                 ║
║  │ Source data is mutable?    → MERGE          │                 ║
║  │ Source data is append-only?→ INSERT + dedup  │                 ║
║  └─────────────────────────────────────────────┘                 ║
║                                                                  ║
║  IDEMPOTENT WRITE PATTERNS:                                      ║
║  ┌────────────────────────────────────────────┐                  ║
║  │ 1. DELETE partition + INSERT (safest)       │                  ║
║  │ 2. MERGE / UPSERT on unique key            │                  ║
║  │ 3. INSERT OVERWRITE (BigQuery partitions)  │                  ║
║  │ 4. Deterministic file path (S3 overwrite)  │                  ║
║  │ 5. INSERT WHERE NOT EXISTS (append + dedup)│                  ║
║  └────────────────────────────────────────────┘                  ║
║                                                                  ║
║  WATERMARK + LOOKBACK:                                           ║
║                                                                  ║
║  ──────────────────────────────────────────▶ time                ║
║  │←─lookback─→│←──new data──→│                                   ║
║  │ re-process │ first-time   │                                   ║
║  │ (catch late│  processing  │                                   ║
║  │  arrivals) │              │                                   ║
║       ▲                  ▲                                       ║
║  HWM - 3 days        Current HWM                                 ║
║                                                                  ║
║  DEDUP PATTERN:                                                  ║
║  ROW_NUMBER() OVER (                                             ║
║      PARTITION BY business_key                                   ║
║      ORDER BY loaded_at DESC                                     ║
║  ) = 1                                                           ║
║                                                                  ║
║  LATE DATA HANDLING:                                             ║
║  1. Measure lateness distribution                                ║
║  2. Set lookback to cover 99% of late data                       ║
║  3. Use MERGE so re-processing is safe                           ║
║  4. Periodic full refresh catches the remaining 1%               ║
║                                                                  ║
║  EXACTLY-ONCE = At-Least-Once + Idempotent Processing            ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 19 of 45*
