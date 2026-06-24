# 🎯 Topic 18: Batch vs Streaming Processing

> *Understanding when milliseconds matter and when they don't — the architecture decisions between batch, streaming, and hybrid processing paradigms*

---

## 📋 Table of Contents

1. [Core Concepts](#core-concepts)
2. [Batch Processing](#batch-processing)
3. [Stream Processing](#stream-processing)
4. [Latency Requirements Table](#latency-requirements-table)
5. [Delivery Semantics](#delivery-semantics)
6. [Windowing Strategies](#windowing-strategies)
7. [Lambda vs Kappa Architecture](#lambda-vs-kappa-architecture)
8. [When Streaming Is Not Worth It](#when-streaming-is-not-worth-it)
9. [Technology Comparison](#technology-comparison)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Core Concepts

### The Fundamental Trade-Off

```
Latency ◄──────────────────────────────► Simplicity
(How fresh is the data?)               (How easy to build/maintain?)

Streaming: Low latency, high complexity
Batch:     High latency, low complexity
```

> **Critical Insight:** The question is never "should we use streaming?" The question is "does the business value of fresher data justify the engineering complexity of streaming?" For most analytics workloads, the answer is no. For fraud detection, the answer is yes. Know where your use case falls.

### Definitions

| Term | Definition |
|------|-----------|
| **Batch processing** | Process a bounded dataset all at once on a schedule |
| **Stream processing** | Process an unbounded dataset continuously as events arrive |
| **Micro-batch** | Process small batches at very short intervals (seconds) |
| **Event time** | When the event actually occurred |
| **Processing time** | When the system processes the event |
| **Watermark** | Tracker for completeness — "we believe all events before time T have arrived" |

---

## Batch Processing

### Characteristics

- **Bounded data**: Input has a defined start and end
- **Scheduled execution**: Runs at fixed intervals (hourly, daily)
- **High throughput**: Optimized for processing large volumes efficiently
- **Higher latency**: Data is stale until the next batch runs
- **Simpler architecture**: Easier to reason about, test, and debug

### Batch Technologies

| Technology | Best For | Scale |
|-----------|----------|-------|
| **Apache Spark** | Large-scale distributed processing | PB-scale |
| **BigQuery** | SQL-based warehouse transformations | PB-scale |
| **dbt** | SQL transformations in warehouse | Warehouse-limited |
| **AWS Glue** | Serverless ETL on AWS | TB-scale |
| **Pandas/Polars** | Single-machine processing | GB-scale |
| **Snowflake Tasks** | Scheduled SQL in Snowflake | Warehouse-limited |

### Batch Processing Example (Spark)

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum, count, date_trunc

spark = SparkSession.builder.appName("daily_revenue").getOrCreate()

# Read yesterday's partition (bounded input)
events = (
    spark.read.parquet(f"s3://data-lake/events/dt={yesterday}")
    .filter(col("event_type") == "purchase")
)

# Transform: aggregate daily revenue
daily_revenue = (
    events
    .groupBy(date_trunc("hour", col("event_timestamp")).alias("hour"))
    .agg(
        sum("amount").alias("total_revenue"),
        count("*").alias("transaction_count")
    )
)

# Write output (overwrite partition for idempotency)
daily_revenue.write.mode("overwrite").parquet(
    f"s3://data-lake/marts/hourly_revenue/dt={yesterday}"
)
```

### Batch Scheduling Patterns

```
Hourly:   Every hour at :05 (5-min buffer for late data)
Daily:    Every day at 06:00 UTC (after overnight ingestion)
Weekly:   Sunday midnight (for weekly rollups)
Monthly:  1st of month at 03:00 UTC (for month-end close)
```

---

## Stream Processing

### Characteristics

- **Unbounded data**: Input has no defined end — events keep flowing
- **Continuous execution**: Always running, processing events as they arrive
- **Low latency**: Results available in milliseconds to seconds
- **Higher complexity**: Must handle ordering, late data, state management
- **Exactly-once is hard**: Requires careful engineering for correctness

### Streaming Technologies

| Technology | Type | Best For |
|-----------|------|----------|
| **Apache Kafka** | Message broker / event store | Event streaming backbone |
| **Apache Flink** | Stream processing engine | Complex event processing, stateful computation |
| **Amazon Kinesis** | Managed streaming | AWS-native real-time pipelines |
| **Apache Kafka Streams** | Library (not a cluster) | Lightweight stream processing in Java |
| **Spark Structured Streaming** | Micro-batch streaming | Batch + streaming unified API |
| **Google Dataflow** | Managed (Apache Beam) | Unified batch/stream on GCP |
| **Apache Pulsar** | Message broker | Multi-tenancy, geo-replication |

### Stream Processing Example (Flink SQL)

```sql
-- Create a Kafka source table
CREATE TABLE purchase_events (
    user_id STRING,
    product_id STRING,
    amount DECIMAL(10, 2),
    event_time TIMESTAMP(3),
    WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'purchases',
    'properties.bootstrap.servers' = 'kafka:9092',
    'format' = 'json',
    'scan.startup.mode' = 'latest-offset'
);

-- Real-time aggregation with tumbling window
CREATE TABLE hourly_revenue (
    window_start TIMESTAMP,
    window_end TIMESTAMP,
    total_revenue DECIMAL(10, 2),
    transaction_count BIGINT
) WITH (
    'connector' = 'jdbc',
    'url' = 'jdbc:postgresql://db:5432/analytics'
);

INSERT INTO hourly_revenue
SELECT
    TUMBLE_START(event_time, INTERVAL '1' HOUR) AS window_start,
    TUMBLE_END(event_time, INTERVAL '1' HOUR) AS window_end,
    SUM(amount) AS total_revenue,
    COUNT(*) AS transaction_count
FROM purchase_events
GROUP BY TUMBLE(event_time, INTERVAL '1' HOUR);
```

### Kafka Fundamentals

```
Producer → Topic (partitioned) → Consumer Group

┌──────────┐     ┌─────────────────────────┐     ┌──────────┐
│ Producer │────▶│  Topic: purchases        │────▶│ Consumer │
│ (App)    │     │  ┌─────┐ ┌─────┐ ┌─────┐│     │ (Flink)  │
└──────────┘     │  │ P0  │ │ P1  │ │ P2  ││     └──────────┘
                 │  └─────┘ └─────┘ └─────┘│
                 └─────────────────────────┘

Key concepts:
- Topics: Named channels for events
- Partitions: Parallelism unit within a topic
- Consumer groups: Parallel consumers, each partition read by one consumer
- Offsets: Position tracking within a partition
- Retention: How long messages are kept (7 days default)
```

---

## Latency Requirements Table

| Use Case | Acceptable Latency | Approach | Why |
|----------|-------------------|----------|-----|
| **Fraud detection** | < 100ms | Streaming | Must block transaction in real-time |
| **Real-time bidding (ads)** | < 50ms | Streaming | Bid must be placed before auction closes |
| **Live dashboards** | 1-30 seconds | Streaming / micro-batch | Users expect live data |
| **Recommendation updates** | 1-5 minutes | Micro-batch | Near-real-time improves relevance |
| **Operational alerts** | 1-5 minutes | Streaming | Faster detection = faster response |
| **Daily business reports** | Hours | Batch | Stakeholders check once per day |
| **Monthly financial close** | Days | Batch | Accuracy matters more than speed |
| **ML model training** | Hours-days | Batch | Large historical datasets needed |
| **A/B test analysis** | Hours-days | Batch | Statistical significance needs volume |
| **Data warehouse refresh** | Hourly-daily | Batch | Cost-effective for analytics |

> **Critical Insight:** Map your latency requirements BEFORE choosing technology. If the business says "we need real-time," ask "what's the cost of being 1 hour late?" If they can't quantify the cost, you probably don't need streaming. "Real-time" often means "faster than it is now," not "sub-second."

---

## Delivery Semantics

### The Three Guarantees

| Semantic | Meaning | Risk | Example |
|----------|---------|------|---------|
| **At-most-once** | Message processed 0 or 1 times | Data loss | Fire-and-forget logging |
| **At-least-once** | Message processed 1 or more times | Duplicates | Most messaging systems default |
| **Exactly-once** | Message processed exactly 1 time | Most complex | Financial transactions |

### How Exactly-Once Works

True exactly-once is achieved through:

1. **Idempotent producers** — Kafka can deduplicate producer retries
2. **Transactional writes** — Atomic read-process-write cycles
3. **Consumer offset management** — Commit offset only after successful processing

```python
# At-least-once: commit AFTER processing (may reprocess on failure)
for message in consumer:
    process(message)        # If this fails after processing but before commit,
    consumer.commit()       # message will be reprocessed (duplicate)

# Exactly-once pattern: idempotent processing
for message in consumer:
    if not already_processed(message.key):  # Dedup check
        process(message)
        mark_processed(message.key)         # Atomic with processing
    consumer.commit()
```

> **Critical Insight:** In practice, most systems achieve exactly-once semantics through "at-least-once delivery + idempotent processing." The stream delivers duplicates, but your processing logic handles them gracefully (e.g., UPSERT on a unique key, dedup table). True exactly-once end-to-end requires transactional integration between the stream and the sink.

---

## Windowing Strategies

### Why Windowing?

Unbounded streams need boundaries to produce finite results. Windows create bounded subsets of the infinite stream.

### Window Types

```
TUMBLING WINDOW (fixed, non-overlapping):
|─── 5min ───|─── 5min ───|─── 5min ───|
|  events A  |  events B  |  events C  |

SLIDING WINDOW (fixed, overlapping):
|─── 5min ───|
    |─── 5min ───|
        |─── 5min ───|
(slides every 1 minute → events can be in multiple windows)

SESSION WINDOW (dynamic, gap-based):
|──user activity──|  (gap > 30min)  |──user activity──|
   Session 1                           Session 2
```

### Window Implementations

```sql
-- Flink SQL: Tumbling window (non-overlapping 1-hour buckets)
SELECT
    TUMBLE_START(event_time, INTERVAL '1' HOUR) AS window_start,
    user_id,
    COUNT(*) AS event_count
FROM events
GROUP BY TUMBLE(event_time, INTERVAL '1' HOUR), user_id;

-- Flink SQL: Sliding window (1-hour window, slides every 15 minutes)
SELECT
    HOP_START(event_time, INTERVAL '15' MINUTE, INTERVAL '1' HOUR) AS window_start,
    SUM(amount) AS rolling_revenue
FROM purchases
GROUP BY HOP(event_time, INTERVAL '15' MINUTE, INTERVAL '1' HOUR);

-- Flink SQL: Session window (30-minute gap)
SELECT
    SESSION_START(event_time, INTERVAL '30' MINUTE) AS session_start,
    user_id,
    COUNT(*) AS events_in_session
FROM user_events
GROUP BY SESSION(event_time, INTERVAL '30' MINUTE), user_id;
```

### Handling Late Data

```
Watermark at T=10:05 means:
"We believe all events with event_time <= 10:05 have arrived"

What if an event arrives with event_time=10:03 after the watermark?
→ It's LATE DATA

Options:
1. Drop it (simple, some data loss)
2. Update the result (complex, more accurate)
3. Side-output to a late data stream (flexible)
```

```sql
-- Allow 5 seconds of lateness
WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND

-- Flink: Allow 1 hour of late data to update window results
SELECT
    TUMBLE_START(event_time, INTERVAL '1' HOUR) AS window_start,
    COUNT(*) AS event_count
FROM events
GROUP BY TUMBLE(event_time, INTERVAL '1' HOUR)
-- Late events up to 1 hour after watermark will update the window
```

---

## Lambda vs Kappa Architecture

### Lambda Architecture

Two parallel processing paths: batch for accuracy, streaming for speed.

```
                    ┌─────────────────┐
                    │  Batch Layer    │
         ┌────────▶│  (Spark, daily) │────────┐
         │         │  = "Truth"      │        │
         │         └─────────────────┘        ▼
┌────────┤                              ┌──────────┐
│ Event  │                              │  Serving │──▶ Consumers
│ Stream │                              │  Layer   │
└────────┤                              └──────────┘
         │         ┌─────────────────┐        ▲
         └────────▶│  Speed Layer    │────────┘
                   │  (Flink, real-  │
                   │   time approx.) │
                   └─────────────────┘
```

**Pros:** Accuracy (batch corrects streaming approximations)
**Cons:** Two codebases to maintain, complex merging logic

### Kappa Architecture

Single streaming path for everything — replay the stream for corrections.

```
┌────────┐    ┌─────────────────┐    ┌──────────┐
│ Event  │───▶│  Stream Layer   │───▶│  Serving │──▶ Consumers
│ Stream │    │  (Flink)        │    │  Layer   │
└────────┘    └─────────────────┘    └──────────┘
     │
     └── Replay from offset 0 for reprocessing
```

**Pros:** Single codebase, simpler architecture
**Cons:** Requires replayable stream (Kafka with long retention), reprocessing can be slow

### Comparison

| Dimension | Lambda | Kappa |
|-----------|--------|-------|
| **Codebases** | Two (batch + stream) | One (stream only) |
| **Complexity** | Higher (dual maintenance) | Lower (single path) |
| **Accuracy** | Batch layer corrects errors | Must get streaming right |
| **Reprocessing** | Re-run batch job | Replay stream from beginning |
| **Late data** | Batch handles it naturally | Must design for it explicitly |
| **Storage** | Kafka + batch store + serving | Kafka (long retention) + serving |
| **Modern preference** | Declining | Growing (with mature stream tech) |

> **Critical Insight:** Lambda architecture was created because early streaming systems couldn't guarantee correctness. Modern systems (Flink with exactly-once, Spark Structured Streaming) largely eliminate this need. Most new architectures prefer Kappa or a simplified hybrid where streaming handles real-time and periodic batch jobs reconcile.

---

## When Streaming Is Not Worth It

### Signs You Don't Need Streaming

1. **Consumers check data daily** — A dashboard viewed at 9 AM doesn't need sub-second data
2. **Source data arrives in batches** — If your source exports daily CSVs, streaming the load adds no value
3. **Aggregations span large windows** — Monthly revenue doesn't benefit from streaming
4. **Team lacks streaming expertise** — Operational complexity will cause more downtime than batch latency
5. **Volume is low** — If you process 10K records/day, scheduled SQL is simpler and cheaper
6. **Correctness > speed** — Financial reconciliation, regulatory reporting

### Cost Comparison

| Factor | Batch | Streaming |
|--------|-------|-----------|
| **Infrastructure** | Pay only when running | Always-on clusters |
| **Engineering time** | Lower (SQL, simple scheduling) | Higher (state management, failure handling) |
| **Debugging** | Reproduce with same input | Non-deterministic, harder to reproduce |
| **Testing** | Standard unit/integration tests | Complex (time, ordering, late data) |
| **Monitoring** | Task success/failure | Lag, throughput, backpressure |
| **On-call burden** | Low (run once, fix, re-run) | High (24/7 process, state recovery) |

### The Pragmatic Middle Ground: Micro-Batch

```
Full Batch (hourly)  →  Micro-Batch (5 min)  →  True Streaming (ms)
     Simple                  Moderate               Complex
     High latency            Low latency            Lowest latency
     Cheapest                Moderate cost           Expensive
```

Spark Structured Streaming with trigger intervals of 1-5 minutes gives near-real-time with batch-like simplicity.

```python
# Spark micro-batch: processes every 2 minutes
(
    spark.readStream
    .format("kafka")
    .option("subscribe", "events")
    .load()
    .writeStream
    .trigger(processingTime="2 minutes")  # Micro-batch every 2 min
    .outputMode("append")
    .format("parquet")
    .start("s3://data-lake/events/")
)
```

---

## Technology Comparison

| Feature | Spark (Batch) | Spark Streaming | Flink | Kafka Streams |
|---------|---------------|-----------------|-------|---------------|
| **Processing model** | Batch | Micro-batch | True streaming | True streaming |
| **Latency** | Minutes-hours | Seconds-minutes | Milliseconds | Milliseconds |
| **State management** | External (checkpoints) | Built-in (limited) | Built-in (RocksDB) | Built-in |
| **Exactly-once** | Via idempotent writes | Yes (with caveats) | Yes (native) | Yes (Kafka transactions) |
| **Deployment** | Cluster (YARN, K8s) | Cluster (YARN, K8s) | Cluster (standalone, K8s) | Library (in your app) |
| **Windowing** | Time-based | Limited | Rich (event time, sessions) | Rich |
| **SQL support** | Spark SQL | Structured Streaming SQL | Flink SQL | KSQL (separate) |
| **Best for** | Large-scale ETL | Near-real-time analytics | Complex event processing | Lightweight app-level |

---

## Interview Talking Points

> **"When would you choose streaming over batch?"**
>
> "I'd choose streaming when the business value of fresher data clearly justifies the complexity. Fraud detection — you must block a transaction in real-time. Live operational dashboards — support teams need current data. User-facing recommendations that degrade with staleness. But for most analytics, reporting, and ML training pipelines, batch is the right choice. I always ask: what's the cost of the data being 1 hour old versus 1 second old?"

> **"Explain the difference between event time and processing time."**
>
> "Event time is when something actually happened — the timestamp embedded in the event by the source. Processing time is when the system processes it. They differ because of network delays, buffering, and out-of-order delivery. For correct aggregations, you almost always want event time. Watermarks help track completeness — telling the system 'we believe all events before time T have arrived, so windows before T can be finalized.'"

> **"What's the difference between Lambda and Kappa architecture?"**
>
> "Lambda runs two parallel systems — a batch layer for accuracy and a speed layer for low latency — merging results in a serving layer. It was created because early streaming couldn't guarantee correctness. Kappa uses a single streaming pipeline for everything, replaying the stream when you need to reprocess. Modern tools like Flink make Kappa viable because they provide exactly-once guarantees. I'd choose Kappa for new systems to avoid dual-codebase maintenance."

---

## Common Mistakes

| ❌ Mistake | ✅ Best Practice |
|-----------|-----------------|
| Choosing streaming because "real-time sounds impressive" | Start with batch; add streaming only when latency requirements demand it |
| Ignoring late-arriving data in streaming | Define watermarks and late data handling strategy |
| Using processing time when event time is needed | Default to event time for business aggregations |
| No monitoring of consumer lag | Alert when Kafka consumer lag exceeds threshold |
| Exactly-once without understanding the guarantee scope | Clarify end-to-end vs within-system guarantees |
| Streaming without a dead letter queue | Route unparseable/failed events to DLQ for investigation |
| State growing unbounded in streaming jobs | Set state TTLs and manage state cleanup |
| Lambda architecture for new projects | Prefer Kappa or simplified hybrid unless you have specific accuracy requirements |
| Not testing with out-of-order events | Simulate late and out-of-order data in integration tests |
| Treating streaming as "faster batch" | Streaming requires fundamentally different thinking about completeness and state |

---

## Rapid-Fire Q&A

**Q1: What is the key difference between batch and stream processing?**
> Batch processes bounded datasets on a schedule (high throughput, higher latency). Streaming processes unbounded data continuously (low latency, higher complexity).

**Q2: Name a use case where streaming is essential.**
> Fraud detection — you must evaluate and potentially block a transaction before it completes, requiring sub-second processing.

**Q3: What is a watermark in stream processing?**
> A watermark is a declaration that all events with timestamps before a certain time have been observed. It signals when a time-based window can be finalized.

**Q4: What are the three delivery semantics?**
> At-most-once (may lose data), at-least-once (may duplicate), exactly-once (correct but complex). Most systems achieve exactly-once through idempotent processing.

**Q5: Explain tumbling vs sliding windows.**
> Tumbling: fixed-size, non-overlapping (each event in exactly one window). Sliding: fixed-size but overlapping (events can be in multiple windows, slides at a smaller interval).

**Q6: What is a session window?**
> A dynamic window defined by inactivity gaps. A session ends when no events arrive for a specified duration (e.g., 30 minutes of user inactivity).

**Q7: What is consumer lag in Kafka?**
> The difference between the latest produced offset and the consumer's committed offset. High lag means the consumer is falling behind — events are processed with increasing delay.

**Q8: Why is Kappa architecture gaining popularity?**
> Modern streaming engines (Flink) provide exactly-once guarantees, eliminating the need for a separate batch layer for correctness. Single codebase = simpler maintenance.

**Q9: What is micro-batching?**
> Processing data in very small batches at short intervals (seconds). Spark Structured Streaming uses this — it provides near-real-time latency with batch-like simplicity.

**Q10: When is batch processing the better choice?**
> When data consumers don't need sub-minute freshness, source data arrives in batches, the team lacks streaming expertise, or when correctness and cost-efficiency matter more than speed.

---

## ASCII Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════╗
║           BATCH vs STREAMING DECISION FRAMEWORK                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  DECISION TREE:                                                  ║
║  Is data loss/delay costly in real-time?                         ║
║  ├── YES (fraud, bidding, alerts) → STREAMING                   ║
║  └── NO → Is sub-minute freshness needed?                        ║
║       ├── YES (live dashboards) → MICRO-BATCH                   ║
║       └── NO (reports, ML) → BATCH                              ║
║                                                                  ║
║  LATENCY SPECTRUM:                                               ║
║  ◄─────────────────────────────────────────────────────────►    ║
║  ms        sec        min        hour        day                 ║
║  │         │          │          │           │                   ║
║  Flink   Kafka     Spark SS   Scheduled   Daily                  ║
║  Fraud   Streams   Micro-     SQL jobs    batch                  ║
║  detect  Live UI   batch      Analytics   reports                ║
║                                                                  ║
║  WINDOWING:                                                      ║
║  Tumbling: |──W1──|──W2──|──W3──|  (non-overlapping)            ║
║  Sliding:  |──W1──|                (overlapping)                 ║
║               |──W2──|                                           ║
║  Session:  |─activity─| gap |─activity─|  (dynamic)             ║
║                                                                  ║
║  DELIVERY SEMANTICS:                                             ║
║  At-most-once  → Fast, may lose data                             ║
║  At-least-once → Safe, may duplicate → add dedup                 ║
║  Exactly-once  → Correct, complex → idempotent writes            ║
║                                                                  ║
║  ARCHITECTURES:                                                  ║
║  Lambda: Batch + Stream → Serving (accuracy + speed)             ║
║  Kappa:  Stream only → Serving (simplicity, replay)              ║
║                                                                  ║
║  RULE OF THUMB:                                                  ║
║  "If you can't quantify the cost of 1-hour latency,             ║
║   you probably don't need streaming."                            ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 18 of 45*
