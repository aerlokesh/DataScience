# 🎯 Topic 16: Data Pipeline Design and DAGs

> *Mastering the art of orchestrating data workflows — from DAG theory to production-grade Airflow patterns for reliable, idempotent, and observable pipelines*

---

## 📋 Table of Contents

1. [Core Concepts](#core-concepts)
2. [DAG Theory](#dag-theory)
3. [Apache Airflow Fundamentals](#apache-airflow-fundamentals)
4. [Dependency Patterns](#dependency-patterns)
5. [Idempotency](#idempotency)
6. [Retry and Backfill Strategies](#retry-and-backfill-strategies)
7. [Monitoring and Alerting](#monitoring-and-alerting)
8. [Common Pipeline Patterns](#common-pipeline-patterns)
9. [Failure Handling](#failure-handling)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Core Concepts

### What is a Data Pipeline?

A data pipeline is a series of processing steps that move and transform data from sources to destinations. It automates the flow of data through extraction, validation, transformation, and loading.

> **Critical Insight:** A production pipeline is not just "code that transforms data." It's code + scheduling + dependency management + monitoring + alerting + retry logic + backfill capability + documentation. The transformation logic is often the easy part.

### Pipeline vs Workflow vs DAG

| Term | Definition | Example |
|------|-----------|---------|
| **Pipeline** | End-to-end data flow from source to destination | Stripe payments → warehouse → dashboard |
| **Workflow** | Sequence of tasks with dependencies | Extract → validate → transform → load → test |
| **DAG** | Directed Acyclic Graph — the formal structure | Tasks as nodes, dependencies as directed edges |

---

## DAG Theory

### Directed Acyclic Graph — Definition

A **DAG** is a graph where:
- **Directed**: Edges have a direction (A → B means A runs before B)
- **Acyclic**: No cycles — you cannot follow edges and return to the same node

```
     A
    / \
   B   C
    \ / \
     D   E
      \ /
       F
```

This represents: A runs first, then B and C can run in parallel, D depends on both B and C, E depends on C, and F depends on both D and E.

### Why DAGs?

1. **Deterministic execution order** — Dependencies define exactly what must complete before each task
2. **Parallelism** — Independent tasks run concurrently
3. **No infinite loops** — Acyclic property guarantees termination
4. **Clear lineage** — Trace any output back to its inputs

> **Critical Insight:** The acyclic constraint is not arbitrary. If Task A depends on Task B which depends on Task A, you have a deadlock. DAGs make deadlocks structurally impossible. If you truly need a cycle (e.g., iterative ML training until convergence), you model it as a loop within a single task or use a sensor pattern.

### Topological Sort

The order in which DAG nodes can execute is determined by **topological sorting** — any valid execution order respects all dependency edges.

```python
# Multiple valid topological orders for the DAG above:
# Order 1: A → B → C → D → E → F
# Order 2: A → C → B → E → D → F
# Order 3: A → C → E → B → D → F

# The scheduler picks an order that maximizes parallelism:
# Step 1: A
# Step 2: B, C (parallel)
# Step 3: D, E (parallel, once their parents complete)
# Step 4: F
```

---

## Apache Airflow Fundamentals

### Core Components

| Component | Role | Analogy |
|-----------|------|---------|
| **DAG** | The workflow definition | The blueprint |
| **Task** | A single unit of work | A step in the blueprint |
| **Operator** | Defines what a task does | The tool used for the step |
| **Sensor** | Waits for a condition | A gatekeeper |
| **Executor** | Runs tasks | The construction crew |
| **Scheduler** | Triggers DAGs on schedule | The project manager |
| **Metadata DB** | Stores state | The logbook |

### DAG Definition

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.google.cloud.operators.bigquery import BigQueryInsertJobOperator
from airflow.sensors.external_task import ExternalTaskSensor
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'email_on_failure': True,
    'email': ['data-alerts@company.com'],
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'retry_exponential_backoff': True,
    'execution_timeout': timedelta(hours=2),
}

with DAG(
    dag_id='daily_revenue_pipeline',
    default_args=default_args,
    description='Daily revenue aggregation pipeline',
    schedule_interval='@daily',  # or cron: '0 6 * * *'
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=['revenue', 'production'],
    max_active_runs=1,
) as dag:

    wait_for_ingestion = ExternalTaskSensor(
        task_id='wait_for_stripe_ingestion',
        external_dag_id='stripe_ingestion',
        external_task_id='load_complete',
        timeout=3600,
        poke_interval=300,
    )

    transform_payments = BigQueryInsertJobOperator(
        task_id='transform_payments',
        configuration={
            'query': {
                'query': "{% include 'sql/transform_payments.sql' %}",
                'useLegacySql': False,
                'destinationTable': {
                    'projectId': 'analytics-prod',
                    'datasetId': 'marts',
                    'tableId': 'fct_payments${{ ds_nodash }}',
                },
                'writeDisposition': 'WRITE_TRUNCATE',
            }
        },
    )

    validate_output = PythonOperator(
        task_id='validate_output',
        python_callable=run_data_quality_checks,
        op_kwargs={'table': 'marts.fct_payments', 'date': '{{ ds }}'},
    )

    wait_for_ingestion >> transform_payments >> validate_output
```

### Operators — The Building Blocks

| Operator Type | Use Case | Example |
|---------------|----------|---------|
| **PythonOperator** | Run Python functions | Data validation, API calls |
| **BashOperator** | Run shell commands | dbt run, spark-submit |
| **BigQueryOperator** | Execute BigQuery SQL | Transformations, table creation |
| **S3ToRedshiftOperator** | Load S3 → Redshift | Data loading |
| **ExternalTaskSensor** | Wait for another DAG | Cross-DAG dependencies |
| **FileSensor** | Wait for a file to appear | Landing zone monitoring |
| **EmailOperator** | Send notifications | Alert on completion |
| **BranchPythonOperator** | Conditional branching | Skip weekends, route by data |

### Sensors — Waiting for Conditions

```python
from airflow.sensors.s3_key_sensor import S3KeySensor

wait_for_file = S3KeySensor(
    task_id='wait_for_daily_export',
    bucket_name='data-lake-raw',
    bucket_key='exports/{{ ds }}/customers.parquet',
    aws_conn_id='aws_default',
    timeout=7200,        # 2 hours max wait
    poke_interval=300,   # Check every 5 minutes
    mode='reschedule',   # Release worker slot between checks
)
```

> **Critical Insight:** Always use `mode='reschedule'` for sensors with long wait times. The default `mode='poke'` holds a worker slot for the entire duration — with many sensors, you can starve your executor of available slots for actual work.

---

## Dependency Patterns

### Linear Chain

```python
extract >> transform >> load >> validate
```

### Fan-Out / Fan-In

```python
extract >> [transform_users, transform_orders, transform_products] >> build_mart
```

### Conditional Branching

```python
from airflow.operators.python import BranchPythonOperator

def choose_branch(**context):
    if context['execution_date'].weekday() < 5:
        return 'process_weekday'
    return 'process_weekend'

branch = BranchPythonOperator(
    task_id='branch_by_day',
    python_callable=choose_branch,
)

branch >> [process_weekday, process_weekend] >> merge_results
```

### Cross-DAG Dependencies

```python
# DAG B depends on DAG A completing
sensor = ExternalTaskSensor(
    task_id='wait_for_dag_a',
    external_dag_id='dag_a',
    external_task_id='final_task',
    execution_date_fn=lambda dt: dt,  # Same execution date
)
```

### Dynamic Task Generation

```python
# Generate tasks dynamically based on configuration
tables = ['users', 'orders', 'products', 'payments']

for table in tables:
    extract = PythonOperator(
        task_id=f'extract_{table}',
        python_callable=extract_table,
        op_kwargs={'table_name': table},
    )
    load = PythonOperator(
        task_id=f'load_{table}',
        python_callable=load_table,
        op_kwargs={'table_name': table},
    )
    extract >> load
```

---

## Idempotency

### Definition

A pipeline is **idempotent** if running it multiple times with the same input produces the same output — no duplicates, no corruption.

> **Critical Insight:** Idempotency is the single most important property of a production pipeline. Without it, retries create duplicates, backfills corrupt data, and every failure requires manual intervention. Design for idempotency from day one.

### Strategies for Idempotency

| Strategy | How It Works | When to Use |
|----------|-------------|-------------|
| **WRITE_TRUNCATE** | Delete partition, then write | Partition-level refreshes |
| **MERGE/UPSERT** | Insert or update based on key | Slowly changing dimensions |
| **Deterministic naming** | Output path includes run date | File-based pipelines |
| **Delete-then-insert** | Remove existing records for the window, then insert | Row-level control |

### Idempotent SQL Pattern

```sql
-- IDEMPOTENT: Re-running for the same date always produces the same result
-- Step 1: Delete existing data for this partition
DELETE FROM marts.fct_daily_revenue
WHERE revenue_date = '{{ ds }}';

-- Step 2: Insert fresh data
INSERT INTO marts.fct_daily_revenue
SELECT
    '{{ ds }}' AS revenue_date,
    product_category,
    SUM(amount) AS total_revenue,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM staging.stg_payments
WHERE payment_date = '{{ ds }}'
  AND status = 'succeeded'
GROUP BY product_category;
```

### Non-Idempotent Anti-Pattern

```sql
-- NOT IDEMPOTENT: Running twice creates duplicates!
INSERT INTO marts.fct_daily_revenue
SELECT ...
FROM staging.stg_payments
WHERE payment_date = '{{ ds }}';
```

---

## Retry and Backfill Strategies

### Retry Configuration

```python
default_args = {
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'retry_exponential_backoff': True,  # 5min, 10min, 20min
    'max_retry_delay': timedelta(hours=1),
}
```

### Backfill

Backfilling re-processes historical data. With idempotent pipelines, this is safe.

```bash
# Backfill a specific date range
airflow dags backfill daily_revenue_pipeline \
    --start-date 2024-01-01 \
    --end-date 2024-01-31 \
    --reset-dagruns
```

### Backfill Considerations

1. **Parallelism** — Limit concurrent backfill runs to avoid overwhelming sources
2. **Idempotency** — Ensure pipeline handles re-runs safely
3. **Dependencies** — Upstream data must exist for the backfill period
4. **Resource impact** — Backfills can be expensive; schedule during off-peak

---

## Monitoring and Alerting

### Key Metrics to Monitor

| Metric | Why It Matters | Alert Threshold |
|--------|---------------|-----------------|
| **Task duration** | Detects performance regression | > 2x historical p95 |
| **Task failure rate** | Detects systemic issues | > 0 for critical paths |
| **DAG completion time** | SLA monitoring | Exceeds SLA window |
| **Sensor timeout** | Upstream issues | Any timeout |
| **Queue depth** | Resource saturation | > N tasks queued |
| **Data freshness** | Consumer impact | > X hours stale |

### Alerting Strategy

```python
from airflow.operators.python import PythonOperator

def sla_miss_callback(dag, task_list, blocking_task_list, slas, blocking_tis):
    """Called when SLA is missed."""
    send_slack_alert(
        channel='#data-alerts',
        message=f"SLA MISS: {dag.dag_id} missed its SLA. "
                f"Blocking tasks: {blocking_task_list}"
    )

dag = DAG(
    'daily_revenue_pipeline',
    sla_miss_callback=sla_miss_callback,
    ...
)

# Task-level SLA
task = PythonOperator(
    task_id='critical_transform',
    sla=timedelta(hours=2),  # Must complete within 2h of scheduled time
    ...
)
```

### Observability Stack

```
Pipeline Logs    →  Centralized Logging (ELK/CloudWatch)
Task Metrics     →  Time Series DB (Prometheus/Datadog)
Data Quality     →  dbt tests / Great Expectations / Monte Carlo
Lineage          →  OpenLineage / Marquez / DataHub
Alerting         →  PagerDuty / Slack / Email
```

---

## Common Pipeline Patterns

### 1. Ingestion Pipeline

```
Source API → Extract → Validate Schema → Load Raw → Update Metadata
```

### 2. Transformation Pipeline

```
Sensor (raw ready) → Stage → Intermediate Join → Mart → Test → Publish
```

### 3. ML Training Pipeline

```
Feature Extract → Train/Val Split → Train Model → Evaluate → Register → Deploy
```

### 4. Reverse ETL Pipeline

```
Warehouse Query → Format → Push to SaaS (Salesforce, HubSpot, Braze)
```

### 5. Data Quality Pipeline

```
Schedule → Run Tests → Aggregate Results → Alert on Failures → Dashboard
```

### Pattern: The "Smart Sensor" Approach

```python
# Instead of: extract → transform (coupled)
# Use: ingest_dag (extract → load) + transform_dag (sensor → transform)

# This decouples ingestion from transformation:
# - Ingestion runs on source system's schedule
# - Transformation runs when data is available
# - Different ownership, different SLAs
```

---

## Failure Handling

### Failure Categories

| Category | Example | Response |
|----------|---------|----------|
| **Transient** | Network timeout, rate limit | Retry with backoff |
| **Data quality** | Null primary keys, schema change | Alert + quarantine |
| **Resource** | OOM, disk full | Scale up + retry |
| **Logic** | Bug in transformation SQL | Fix code + backfill |
| **Upstream** | Source system down | Sensor timeout + alert |

### Graceful Failure Patterns

```python
from airflow.exceptions import AirflowSkipException

def extract_with_fallback(**context):
    """Try primary source, fall back to secondary."""
    try:
        data = extract_from_primary()
    except PrimarySourceError:
        logger.warning("Primary source failed, using secondary")
        data = extract_from_secondary()

    if data.empty:
        raise AirflowSkipException("No data for this execution date")

    return data

def on_failure_callback(context):
    """Called when a task fails after all retries."""
    task_instance = context['task_instance']
    send_slack_alert(
        channel='#data-alerts',
        message=f"PIPELINE FAILURE\n"
                f"DAG: {task_instance.dag_id}\n"
                f"Task: {task_instance.task_id}\n"
                f"Execution Date: {context['ds']}\n"
                f"Log URL: {task_instance.log_url}"
    )
```

### Circuit Breaker Pattern

```python
def check_source_health(**context):
    """Stop pipeline if source is unhealthy."""
    health = check_api_health()
    if health['error_rate'] > 0.5:
        raise AirflowFailException(
            "Source API error rate > 50%. Stopping pipeline to avoid loading bad data."
        )
```

> **Critical Insight:** Not all failures should trigger retries. A data quality failure (e.g., source sent corrupt data) should NOT be retried — it will fail the same way every time. Distinguish between transient failures (retry) and persistent failures (alert and stop).

---

## Interview Talking Points

> **"Describe how you would design a data pipeline for a new data source."**
>
> "I start with requirements: latency needs, data volume, schema stability, and SLA. Then I design in layers: an ingestion DAG that extracts and loads raw data (owned by data engineering), a transformation DAG that builds staging and mart tables (owned by analytics engineering), with a sensor bridging them. Each task is idempotent — using partition-level overwrites or MERGE patterns. I add monitoring at every stage: freshness checks on raw data, dbt tests on transformed data, and SLA alerts on the overall pipeline. I always design for backfill from day one."

> **"How do you handle pipeline failures at 3 AM?"**
>
> "Prevention first: I set retries with exponential backoff for transient failures, implement circuit breakers to stop on persistent failures, and use sensors with timeouts to handle upstream delays. For detection: automated alerts via Slack/PagerDuty with context (which task, which date, log links). For recovery: idempotent pipelines mean I can safely re-trigger failed tasks without manual data cleanup. For root cause: I categorize failures (transient vs persistent vs data quality) and address each differently."

> **"What makes a pipeline production-ready?"**
>
> "Five properties: (1) Idempotent — safe to re-run. (2) Observable — metrics, logs, and alerts. (3) Testable — data quality checks at each layer. (4) Documented — what it does, who owns it, what the SLA is. (5) Recoverable — handles failures gracefully with retries, backfill support, and clear alerting."

---

## Common Mistakes

| ❌ Mistake | ✅ Best Practice |
|-----------|-----------------|
| Non-idempotent tasks (INSERT without dedup) | Use DELETE+INSERT, MERGE, or WRITE_TRUNCATE |
| Using `depends_on_past=True` without understanding implications | Use explicit sensors or data-driven dependencies |
| Sensors in `poke` mode holding worker slots | Use `mode='reschedule'` for long-waiting sensors |
| Massive monolithic DAGs (100+ tasks) | Split into smaller DAGs with sensor-based coupling |
| No alerting on silent failures | Add data freshness monitoring and SLA callbacks |
| Hardcoded dates instead of templating | Use Airflow templates (`{{ ds }}`, `{{ data_interval_start }}`) |
| Running backfills without `max_active_runs=1` | Limit concurrency to avoid resource contention |
| Business logic in the DAG file | Keep DAG files thin; logic in SQL/Python modules |
| No documentation on DAG purpose and ownership | Add description, tags, and ownership metadata |
| Ignoring task execution time trends | Monitor and alert on duration anomalies |

---

## Rapid-Fire Q&A

**Q1: What is a DAG and why is the "acyclic" property important?**
> A Directed Acyclic Graph has directed edges (dependencies) and no cycles. The acyclic property guarantees termination — no task can depend on itself directly or indirectly, preventing deadlocks.

**Q2: What is idempotency and why does it matter for pipelines?**
> A pipeline is idempotent if running it multiple times with the same input produces the same output. It enables safe retries, backfills, and failure recovery without duplicates.

**Q3: Explain the difference between `poke` and `reschedule` mode for sensors.**
> `poke` holds a worker slot continuously while waiting. `reschedule` releases the slot between checks and gets rescheduled — better for long waits.

**Q4: How would you backfill 30 days of data?**
> Use `airflow dags backfill` with the date range. Ensure the pipeline is idempotent, limit `max_active_runs`, and verify upstream data exists for the period.

**Q5: What is the difference between an Operator and a Sensor?**
> An Operator performs an action (run SQL, call API). A Sensor waits for a condition to be true (file exists, task completed, time reached) before proceeding.

**Q6: How do you handle cross-DAG dependencies?**
> ExternalTaskSensor waits for a specific task in another DAG to complete. Alternatively, use dataset-aware scheduling (Airflow 2.4+) or event-driven triggers.

**Q7: What does `catchup=False` do?**
> Prevents Airflow from running all historical DAG runs between the start_date and now when the DAG is first deployed. Only future scheduled runs execute.

**Q8: How do you make a task retry on failure?**
> Set `retries` and `retry_delay` in default_args or per-task. Use `retry_exponential_backoff=True` for increasing delays between retries.

**Q9: What's the difference between a Task and a TaskGroup?**
> A Task is a single unit of work. A TaskGroup is a visual/logical grouping of tasks in the UI — it doesn't change execution behavior but improves readability.

**Q10: How do you prevent a pipeline from processing duplicate data?**
> Idempotent writes (MERGE, DELETE+INSERT), deduplication in staging (ROW_NUMBER by key), and ensuring extraction uses watermarks (only fetch new/changed records).

---

## ASCII Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════╗
║              DATA PIPELINE DESIGN CHEAT SHEET                    ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  DAG STRUCTURE:                                                  ║
║  ┌─────┐     ┌─────┐ ┌─────┐     ┌─────┐     ┌──────┐         ║
║  │Sense│────▶│Ext_A│ │Ext_B│────▶│Join │────▶│Test  │         ║
║  └─────┘     └──┬──┘ └──┬──┘     └─────┘     └──────┘         ║
║                  └───┬───┘                                       ║
║                      ▼                                           ║
║                 (fan-in)                                         ║
║                                                                  ║
║  AIRFLOW COMPONENTS:                                             ║
║  DAG ─────── Workflow definition (schedule, dependencies)        ║
║  Task ────── Single unit of work                                 ║
║  Operator ── What the task does (SQL, Python, Bash)              ║
║  Sensor ──── Waits for condition (file, task, time)              ║
║  XCom ────── Share small data between tasks                      ║
║                                                                  ║
║  IDEMPOTENCY PATTERNS:                                           ║
║  ┌────────────────────────────────────────────┐                  ║
║  │ 1. DELETE partition + INSERT (safest)       │                  ║
║  │ 2. MERGE / UPSERT on primary key           │                  ║
║  │ 3. WRITE_TRUNCATE (BigQuery partitions)    │                  ║
║  │ 4. Deterministic output paths (S3 + date)  │                  ║
║  └────────────────────────────────────────────┘                  ║
║                                                                  ║
║  FAILURE HANDLING:                                               ║
║  Transient (timeout, rate limit) → Retry with backoff            ║
║  Data quality (nulls, schema)    → Alert + quarantine            ║
║  Resource (OOM, disk)            → Scale up + retry              ║
║  Logic (bug in SQL)              → Fix code + backfill           ║
║  Upstream (source down)          → Sensor timeout + alert        ║
║                                                                  ║
║  PRODUCTION CHECKLIST:                                           ║
║  □ Idempotent        □ Observable (metrics + alerts)             ║
║  □ Tested (dbt)      □ Documented (owner, SLA)                   ║
║  □ Backfillable      □ Handles failures gracefully               ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 16 of 45*
