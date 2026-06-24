# 🎯 Topic 43: Data Freshness, Lineage, and Observability

> *"You can't trust data you can't trace. Data observability answers three questions: Is the data fresh? Where did it come from? Is it behaving normally? Without answers to all three, every analysis is an act of faith."*

---

## 📑 Table of Contents

1. [Data Freshness and SLAs](#data-freshness-and-slas)
2. [Staleness Detection](#staleness-detection)
3. [Data Lineage Fundamentals](#data-lineage-fundamentals)
4. [Lineage Tools and Implementations](#lineage-tools-and-implementations)
5. [Data Observability](#data-observability)
6. [Incident Response for Data Issues](#incident-response-for-data-issues)
7. [Root Cause Analysis for Pipeline Failures](#root-cause-analysis-for-pipeline-failures)
8. [Building an Observability Stack](#building-an-observability-stack)
9. [Interview Talking Points](#interview-talking-points)
10. [Common Mistakes](#common-mistakes)
11. [Rapid-Fire Q&A](#rapid-fire-qa)
12. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Data Freshness and SLAs

### What Is Data Freshness?

Data freshness measures the gap between when an event occurred in the real world and when it's available for querying in your data system.

```
Real-world event       Ingestion          Transformation       Available for query
    (t=0)         →    (t + lag1)    →    (t + lag1 + lag2) →   (t + total_latency)
    
    ┌─────────────────────────────────────────────────────────┐
    │                    Total Freshness Gap                    │
    └─────────────────────────────────────────────────────────┘
          |               |                    |
     Ingestion Lag    Transform Lag      Serving Lag
```

### Freshness Tiers

| Tier | Latency | Use Case | Infrastructure |
|------|---------|----------|----------------|
| Real-time | < 1 minute | Fraud detection, live ops | Kafka + streaming (Flink/Spark) |
| Near-real-time | 1-15 minutes | Operational dashboards | Micro-batch or streaming |
| Hourly | 15-60 minutes | Intra-day business metrics | Scheduled micro-batch |
| Daily | 1-6 hours | Business dashboards, reports | Daily batch ETL (Airflow) |
| Weekly/Monthly | Hours to days | Strategic analytics, ML training | Scheduled batch |

### Defining Freshness SLAs

```yaml
# freshness_slas.yml
tables:
  - name: events_raw
    owner: platform-engineering
    freshness_sla: 15 minutes
    measurement: MAX(event_timestamp) vs NOW()
    alert_threshold: 30 minutes (2x SLA)
    escalation: page data-eng-oncall
    
  - name: daily_revenue_summary
    owner: analytics-engineering
    freshness_sla: 6:00 AM UTC (available by)
    measurement: MAX(partition_date) = YESTERDAY
    alert_threshold: 7:00 AM UTC
    escalation: slack #data-alerts
    
  - name: ml_feature_store
    owner: ml-platform
    freshness_sla: 4 hours
    measurement: feature_timestamp lag
    alert_threshold: 6 hours
    escalation: page ml-oncall
```

> **Critical Insight:** Freshness SLAs should be set based on consumer needs, not producer convenience. Ask: "What's the latest this data can arrive before it impacts a business decision?" Then set the SLA with buffer. A daily dashboard checked at 9 AM doesn't need real-time data — but it absolutely needs yesterday's data by 8 AM.

---

## Staleness Detection

### Methods for Detecting Stale Data

**Method 1: Timestamp-Based (Most Common)**
```sql
-- How old is the newest record?
SELECT 
    table_name,
    MAX(event_timestamp) AS latest_record,
    EXTRACT(EPOCH FROM (NOW() - MAX(event_timestamp))) / 3600 AS hours_stale,
    CASE 
        WHEN EXTRACT(EPOCH FROM (NOW() - MAX(event_timestamp))) / 3600 > 4 
        THEN 'STALE' 
        ELSE 'FRESH' 
    END AS status
FROM events
GROUP BY table_name;
```

**Method 2: Partition-Based**
```sql
-- Is today's partition populated?
SELECT 
    partition_date,
    COUNT(*) AS row_count
FROM daily_metrics
WHERE partition_date >= CURRENT_DATE - INTERVAL '3 days'
GROUP BY partition_date
ORDER BY partition_date DESC;
-- If today's partition missing or row_count = 0 → STALE
```

**Method 3: Watermark-Based (Streaming)**
```python
# Track the event-time watermark
# Watermark = "I've seen all events up to this timestamp"
current_watermark = get_stream_watermark('user_events')
expected_watermark = datetime.now() - timedelta(minutes=5)

if current_watermark < expected_watermark:
    alert(f"Stream lagging: watermark at {current_watermark}, expected {expected_watermark}")
```

**Method 4: Row Count Anomaly**
```sql
-- Compare today's volume to historical baseline
WITH daily_counts AS (
    SELECT 
        DATE(created_at) AS dt,
        COUNT(*) AS row_count
    FROM events
    WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
    GROUP BY DATE(created_at)
),
stats AS (
    SELECT 
        AVG(row_count) AS avg_count,
        STDDEV(row_count) AS std_count
    FROM daily_counts
    WHERE dt < CURRENT_DATE
)
SELECT 
    d.dt,
    d.row_count,
    s.avg_count,
    (d.row_count - s.avg_count) / s.std_count AS z_score
FROM daily_counts d, stats s
WHERE d.dt = CURRENT_DATE;
-- z_score < -2 suggests data is partially loaded (stale/incomplete)
```

### Freshness Monitoring Dashboard

```
┌─────────────────────────────────────────────┐
│ DATA FRESHNESS MONITOR                       │
├─────────────┬──────────┬────────┬───────────┤
│ Table       │ SLA      │ Actual │ Status    │
├─────────────┼──────────┼────────┼───────────┤
│ events_raw  │ 15 min   │ 8 min  │ ✓ FRESH   │
│ user_dim    │ 4 hours  │ 2.5 hr │ ✓ FRESH   │
│ revenue     │ 6:00 AM  │ MISSING│ ✗ STALE   │
│ ml_features │ 4 hours  │ 3.8 hr │ ⚠ AT RISK │
└─────────────┴──────────┴────────┴───────────┘
```

---

## Data Lineage Fundamentals

### What Is Data Lineage?

Data lineage traces the path of data from source to destination — answering "where did this number come from?" and "what does this table depend on?"

### Types of Lineage

| Type | Granularity | Example |
|------|-------------|---------|
| **Table-level** | Which tables feed into which tables | `orders` + `users` → `order_summary` |
| **Column-level** | Which specific columns feed which | `orders.revenue` → `summary.total_revenue` |
| **Row-level** | Which specific records contributed | Order #123 → row in daily_totals for March 15 |
| **Value-level** | Full provenance of a specific value | "$1.2M revenue" = SUM of 45,000 specific orders |

### Lineage Directions

```
UPSTREAM (Backward Lineage):
  "Where does this table's data come from?"
  daily_revenue ← orders ← raw_events ← kafka_topic ← app_server
  
  Use case: "This number looks wrong — which source could be the culprit?"

DOWNSTREAM (Forward Lineage / Impact Analysis):
  "What breaks if I change this table?"
  raw_events → user_events → session_table → DAU_metric → exec_dashboard
  
  Use case: "If I change the events schema, what's the blast radius?"
```

### Lineage Graph Example

```
                ┌──────────┐
                │ Kafka    │
                │ Events   │
                └────┬─────┘
                     │
              ┌──────┴──────┐
              │             │
        ┌─────┴────┐  ┌────┴─────┐
        │raw_events│  │raw_clicks│
        └─────┬────┘  └────┬─────┘
              │             │
        ┌─────┴────┐       │
        │user_events│      │
        └─────┬────┘       │
              │             │
              └──────┬──────┘
                     │
              ┌──────┴──────┐
              │sessions_table│
              └──────┬──────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
    ┌────┴───┐ ┌─────┴────┐ ┌───┴────┐
    │  DAU   │ │conversion│ │retention│
    │ metric │ │  funnel  │ │ cohorts │
    └────┬───┘ └─────┬────┘ └───┬────┘
         │           │           │
         └───────────┼───────────┘
                     │
              ┌──────┴──────┐
              │exec_dashboard│
              └─────────────┘
```

> **Critical Insight:** Lineage is most valuable during incidents. When a dashboard shows wrong numbers, lineage lets you trace upstream to find the broken step. Without it, debugging a multi-step pipeline is like debugging code without a stack trace.

---

## Lineage Tools and Implementations

### dbt Lineage (Built-in)

```sql
-- dbt models automatically generate lineage via ref()
-- models/sessions.sql
SELECT * FROM {{ ref('raw_events') }}  -- Creates lineage edge

-- dbt docs generate → Visual lineage graph
-- dbt ls --select +daily_revenue+ → Show upstream/downstream
```

### OpenLineage (Open Standard)

```json
{
  "eventType": "COMPLETE",
  "eventTime": "2024-03-15T10:30:00Z",
  "run": {"runId": "abc-123"},
  "job": {"name": "transform_orders", "namespace": "analytics"},
  "inputs": [
    {"name": "raw_orders", "namespace": "warehouse"},
    {"name": "users", "namespace": "warehouse"}
  ],
  "outputs": [
    {"name": "order_summary", "namespace": "warehouse"}
  ]
}
```

### Tools Comparison

| Tool | Type | Lineage Level | Best For |
|------|------|---------------|----------|
| **dbt** | Transform-layer | Table + column | SQL-based pipelines |
| **OpenLineage** | Open standard | Job-level | Cross-platform lineage |
| **Marquez** | Metadata store | Table + column | OpenLineage backend |
| **DataHub** | Metadata platform | Full stack | Enterprise discovery + lineage |
| **Atlan** | Data catalog | Full stack | Business-friendly governance |
| **Monte Carlo** | Observability | Table + column | Anomaly detection + lineage |
| **Apache Atlas** | Governance | Table-level | Hadoop ecosystem |
| **Amundsen** | Discovery | Table-level | Search-first catalog |

### Column-Level Lineage

```
Column-level lineage tells you EXACTLY which source column
feeds each destination column through transformations.

Example:
  daily_revenue.total_rev 
    ← SUM(orders.order_value)
    ← orders.order_value comes from raw_events.purchase_amount
    ← raw_events.purchase_amount comes from Kafka field "amount"

Why it matters:
  If "amount" in Kafka changes from cents to dollars,
  column-level lineage tells you exactly which downstream
  columns are affected (total_rev, avg_order_value, ltv, etc.)
```

---

## Data Observability

### The Five Pillars of Data Observability

| Pillar | What It Monitors | Alert Trigger |
|--------|-----------------|---------------|
| **Freshness** | Is data arriving on time? | Latency exceeds SLA |
| **Volume** | Is the expected amount of data arriving? | Row count anomaly |
| **Distribution** | Are values within normal ranges? | Statistical drift |
| **Schema** | Has the structure changed? | Unexpected column add/remove |
| **Lineage** | What upstream/downstream dependencies exist? | Dependency failure |

### Distribution Monitoring

```python
# Detect distribution anomalies
import numpy as np
from scipy import stats

def detect_distribution_shift(current_data, baseline_data, threshold=0.05):
    """Use Kolmogorov-Smirnov test to detect distribution shift."""
    ks_stat, p_value = stats.ks_2samp(current_data, baseline_data)
    
    return {
        'shifted': p_value < threshold,
        'ks_statistic': ks_stat,
        'p_value': p_value,
        'current_mean': np.mean(current_data),
        'baseline_mean': np.mean(baseline_data),
        'current_std': np.std(current_data),
        'baseline_std': np.std(baseline_data)
    }

# Monitor key columns daily
columns_to_monitor = ['order_value', 'session_duration', 'items_per_order']
for col in columns_to_monitor:
    result = detect_distribution_shift(
        today_df[col].dropna(),
        baseline_30d_df[col].dropna()
    )
    if result['shifted']:
        alert(f"Distribution shift detected in {col}: p={result['p_value']:.4f}")
```

### Volume Monitoring

```sql
-- Daily volume anomaly detection
WITH daily_volumes AS (
    SELECT 
        DATE(created_at) AS dt,
        COUNT(*) AS volume
    FROM orders
    WHERE created_at >= CURRENT_DATE - INTERVAL '60 days'
    GROUP BY DATE(created_at)
),
stats AS (
    SELECT 
        AVG(volume) AS mean_vol,
        STDDEV(volume) AS std_vol,
        -- Day-of-week adjusted baseline
        EXTRACT(DOW FROM dt) AS dow
    FROM daily_volumes
    WHERE dt < CURRENT_DATE - INTERVAL '7 days'
    GROUP BY EXTRACT(DOW FROM dt)
)
SELECT 
    d.dt,
    d.volume,
    s.mean_vol AS expected,
    (d.volume - s.mean_vol) / NULLIF(s.std_vol, 0) AS z_score,
    CASE 
        WHEN ABS((d.volume - s.mean_vol) / NULLIF(s.std_vol, 0)) > 3 THEN 'ANOMALY'
        WHEN ABS((d.volume - s.mean_vol) / NULLIF(s.std_vol, 0)) > 2 THEN 'WARNING'
        ELSE 'NORMAL'
    END AS status
FROM daily_volumes d
JOIN stats s ON EXTRACT(DOW FROM d.dt) = s.dow
WHERE d.dt >= CURRENT_DATE - INTERVAL '7 days'
ORDER BY d.dt;
```

### Schema Change Detection

```python
# Track schema changes over time
def detect_schema_changes(current_schema, previous_schema):
    """Compare two schema snapshots and identify changes."""
    current_cols = set(current_schema.keys())
    previous_cols = set(previous_schema.keys())
    
    changes = {
        'added_columns': current_cols - previous_cols,
        'removed_columns': previous_cols - current_cols,
        'type_changes': [],
        'nullable_changes': []
    }
    
    for col in current_cols & previous_cols:
        if current_schema[col]['type'] != previous_schema[col]['type']:
            changes['type_changes'].append({
                'column': col,
                'old_type': previous_schema[col]['type'],
                'new_type': current_schema[col]['type']
            })
        if current_schema[col]['nullable'] != previous_schema[col]['nullable']:
            changes['nullable_changes'].append({
                'column': col,
                'old_nullable': previous_schema[col]['nullable'],
                'new_nullable': current_schema[col]['nullable']
            })
    
    return changes
```

> **Critical Insight:** Data observability is to data pipelines what APM (Application Performance Monitoring) is to microservices. You wouldn't run a production service without health checks, metrics, and alerting. Your data pipelines deserve the same rigor.

---

## Incident Response for Data Issues

### Data Incident Severity Levels

| Severity | Definition | Response Time | Example |
|----------|-----------|---------------|---------|
| **SEV-1** | Critical data serving wrong values to production | 15 minutes | Revenue dashboard showing 10x values |
| **SEV-2** | Data missing or stale beyond SLA | 1 hour | Daily pipeline failed, no fresh data |
| **SEV-3** | Quality degradation in non-critical data | 4 hours | Null rate spike in optional column |
| **SEV-4** | Minor issue, no business impact | Next business day | Schema documentation outdated |

### Incident Response Playbook

```
1. DETECT
   └── Alert fires (automated) or stakeholder reports issue
   
2. TRIAGE (5 minutes)
   ├── Severity assessment (SEV-1 to SEV-4)
   ├── Blast radius (which consumers affected?)
   └── Assign owner (who's on-call?)
   
3. MITIGATE (Minutes to hours)
   ├── Can we serve stale data with warning? (mark freshness)
   ├── Can we rollback to last known good? (point-in-time recovery)
   └── Can we disable the broken transform? (circuit breaker)
   
4. DIAGNOSE (Parallel with mitigation)
   ├── Check pipeline execution logs
   ├── Trace lineage upstream (where did it break?)
   ├── Check source system health
   └── Identify the specific failure point
   
5. RESOLVE
   ├── Fix the root cause
   ├── Backfill affected data
   ├── Validate fix (run quality checks)
   └── Confirm with consumers
   
6. POST-MORTEM
   ├── Timeline of events
   ├── Root cause analysis
   ├── Impact assessment
   ├── Prevention measures
   └── Update runbooks/alerts
```

### Communication During Incidents

```
INITIAL NOTIFICATION (within 15 min of detection):
"[DATA INCIDENT - SEV-2] daily_revenue table is stale. 
Expected by 6 AM, currently showing March 14 data (24hr stale).
Impact: Executive dashboard showing yesterday's numbers.
Owner: @data-eng-oncall investigating. ETA for update: 30 min."

UPDATE (every 30-60 min):
"[UPDATE] Root cause identified: upstream Kafka consumer 
crashed at 3 AM. Restarting consumer and will backfill.
ETA for fresh data: 2 hours."

RESOLUTION:
"[RESOLVED] daily_revenue table is now current. 
Pipeline restarted at 8:30 AM, backfill completed at 9:15 AM.
All downstream dashboards verified correct. 
Post-mortem scheduled for Thursday."
```

---

## Root Cause Analysis for Pipeline Failures

### Common Failure Categories

```
1. INFRASTRUCTURE FAILURES
   - Compute resource exhaustion (OOM, disk full)
   - Network timeout to source/destination
   - Service dependency unavailable
   - Scheduler failure (Airflow worker crashed)

2. DATA FAILURES  
   - Schema change in source (new/removed column)
   - Data volume spike (10x normal → timeout)
   - Corrupt data (malformed JSON, encoding issues)
   - Late-arriving data (partition not yet complete)

3. CODE FAILURES
   - Bug in transformation logic (bad JOIN, wrong filter)
   - Dependency version conflict
   - Permission/credential expiration
   - Configuration drift (hardcoded values)

4. EXTERNAL FAILURES
   - Source system outage
   - Third-party API rate limiting
   - Cloud provider incident
   - Network partition
```

### Diagnostic Checklist

```sql
-- Step 1: When did it last succeed?
SELECT execution_date, state, duration
FROM airflow_dag_runs
WHERE dag_id = 'daily_revenue_pipeline'
ORDER BY execution_date DESC
LIMIT 10;

-- Step 2: What error occurred?
SELECT task_id, state, error_message, start_date, end_date
FROM airflow_task_instances
WHERE dag_id = 'daily_revenue_pipeline'
  AND execution_date = '2024-03-15'
  AND state = 'failed';

-- Step 3: Is the source data available?
SELECT MAX(event_timestamp), COUNT(*)
FROM source_events
WHERE event_date = '2024-03-15';

-- Step 4: Upstream dependency check
SELECT upstream_table, MAX(updated_at), freshness_status
FROM data_freshness_monitor
WHERE downstream_table = 'daily_revenue';
```

### The 5 Whys for Data Failures

```
Problem: Daily revenue dashboard showed $0 this morning.

Why 1: The daily_revenue table wasn't updated.
Why 2: The Airflow DAG failed at the transform step.
Why 3: The SQL query timed out after 2 hours.
Why 4: The orders table had 50M rows instead of usual 5M.
Why 5: A bulk data migration ran overnight, inserting historical 
       records into the live orders table without a filter.

Root Cause: No safeguard against bulk inserts affecting production pipeline.
Fix: Add row-count circuit breaker + source partition filtering.
```

> **Critical Insight:** Most pipeline failures fall into patterns. After enough incidents, you'll find that 80% of failures come from 3-4 root cause categories. Build automated detection for those specific patterns rather than trying to anticipate every possible failure mode.

---

## Building an Observability Stack

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY STACK                        │
├──────────────┬──────────────┬──────────────┬────────────────┤
│  COLLECTION  │   STORAGE    │  DETECTION   │  ACTION        │
├──────────────┼──────────────┼──────────────┼────────────────┤
│ dbt metadata │ Metadata DB  │ Rule-based   │ Slack alerts   │
│ Query logs   │ (Postgres/   │ alerts       │ PagerDuty      │
│ Pipeline logs│  Snowflake)  │              │ Auto-remediate │
│ Schema snaps │              │ Statistical  │ Circuit break  │
│ OpenLineage  │ Time-series  │ anomaly      │ Runbook link   │
│ events       │ (for metrics)│ detection    │                │
└──────────────┴──────────────┴──────────────┴────────────────┘
```

### Minimum Viable Observability

```
Week 1-2: Freshness monitoring
  - For top 10 tables, track MAX(timestamp) vs NOW()
  - Alert via Slack if stale beyond threshold

Week 3-4: Volume monitoring  
  - Daily row counts with 7-day rolling average
  - Alert on > 2 standard deviation drop

Week 5-6: Quality checks
  - Null rate monitoring on critical columns
  - Uniqueness checks on primary keys

Week 7-8: Lineage
  - Enable dbt docs (if using dbt)
  - Document manual dependencies

Week 9-10: Distribution monitoring
  - Track mean/median/stdev of key numeric columns
  - Alert on significant drift

Week 11-12: Incident process
  - Severity definitions
  - On-call rotation
  - Runbook templates
  - Post-mortem process
```

---

## Interview Talking Points

> "I built our data observability system starting with freshness as the highest-value signal. We tracked MAX(event_timestamp) for our top 20 tables and set SLAs based on consumer needs. Within the first week, we caught a silent pipeline failure that had been serving 3-day-stale data to a financial report — no one had noticed because there was no freshness indicator."

> "For lineage, I leveraged dbt's built-in ref() graph plus custom OpenLineage integration for our Python-based pipelines. When our exec dashboard showed incorrect revenue, I traced through lineage to identify that a new join condition in an intermediate table was dropping 8% of orders. Without lineage, that diagnosis would have taken days instead of 30 minutes."

> "I implemented a data incident response process modeled after SRE practices. We defined severity levels, established a 15-minute acknowledgment SLA for SEV-1 issues, and created runbooks for our top 5 failure modes. Our mean time to resolution dropped from 6 hours to 45 minutes."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| No freshness indicator on dashboards | Always show "Last Updated" and alert if stale |
| Manual lineage documentation (quickly outdated) | Use automated lineage tools (dbt, OpenLineage) |
| Monitoring only end-state tables | Monitor at every transformation step |
| Same alert threshold regardless of day/time | Adjust for day-of-week seasonality and known patterns |
| No incident process for data issues | Define severity levels, on-call, communication templates |
| Only detecting failures, not quality degradation | Monitor distributions, not just availability |
| Lineage only at table level | Column-level lineage enables precise impact analysis |
| No post-mortem process for data incidents | Every SEV-1/2 gets a post-mortem with prevention actions |

---

## Rapid-Fire Q&A

**Q1: What's the difference between data freshness and data latency?**
Latency is the time from event occurrence to data availability (a property of the system). Freshness is the age of the most recent data point available right now (observable state).

**Q2: What's column-level lineage and why is it more valuable than table-level?**
Column-level lineage tracks exactly which source columns feed each destination column through transformations. It enables precise impact analysis — knowing that changing `orders.amount` affects `dashboard.total_revenue` is more actionable than knowing `orders` table feeds `dashboard` table.

**Q3: How do you detect silent pipeline failures?**
Monitor freshness (MAX timestamp), volume (row count vs baseline), and distribution (statistical profile). A pipeline can "succeed" technically but produce wrong/incomplete results — these monitors catch that.

**Q4: What are the five pillars of data observability?**
Freshness, volume, distribution, schema, and lineage.

**Q5: How is data observability different from data quality testing?**
Data quality tests are point-in-time assertions (pass/fail on each run). Data observability is continuous monitoring with anomaly detection, trend analysis, and incident response — it watches between test runs too.

**Q6: What's OpenLineage?**
An open standard for collecting lineage metadata from data pipelines. It defines a common format for lineage events that tools (Spark, Airflow, dbt) can emit, stored in backends like Marquez.

**Q7: How do you handle late-arriving data?**
Design pipelines to be re-runnable (idempotent). Use watermarks to track completeness. Re-process affected partitions when late data arrives. Set a "close window" after which late data goes to a separate reconciliation process.

**Q8: What's a circuit breaker in data pipelines?**
A mechanism that automatically stops pipeline execution when anomalies are detected (e.g., 10x normal volume, zero rows from source). Prevents bad data from propagating downstream.

**Q9: How do you prioritize which tables to monitor?**
By business impact (revenue-impacting first), query frequency (most-used tables), and criticality chain (tables with many downstream dependents per lineage graph).

**Q10: What should a data incident post-mortem include?**
Timeline of events, root cause (using 5 Whys), blast radius (what was affected), detection delay (how long until noticed), resolution steps, and prevention measures (what changes will prevent recurrence).

---

## ASCII Cheat Sheet

```
+============================================================+
|    DATA FRESHNESS, LINEAGE & OBSERVABILITY — CHEAT SHEET    |
+============================================================+

FRESHNESS TIERS:
  Real-time:     < 1 minute    (fraud, live ops)
  Near-real:     1-15 minutes  (operational dashboards)
  Hourly:        15-60 min     (intra-day metrics)
  Daily:         1-6 hours     (business dashboards)
  Batch:         6-24 hours    (reports, ML training)

FIVE PILLARS OF DATA OBSERVABILITY:
  1. Freshness:     Is data arriving on time?
  2. Volume:        Is the expected amount arriving?
  3. Distribution:  Are values within normal ranges?
  4. Schema:        Has the structure changed?
  5. Lineage:       What depends on what?

LINEAGE TYPES:
  Table-level:   orders → order_summary → dashboard
  Column-level:  orders.revenue → summary.total_rev
  Row-level:     Specific records → aggregated values
  
  Upstream:  "Where does this data come from?"
  Downstream: "What breaks if this changes?"

STALENESS DETECTION:
  Timestamp:   MAX(event_ts) vs NOW() > threshold
  Partition:   Expected partition missing
  Volume:      Row count < baseline - 2*stdev
  Watermark:   Stream lag exceeds threshold

INCIDENT SEVERITY:
  SEV-1: Wrong data in production    → 15 min response
  SEV-2: Missing/stale beyond SLA    → 1 hour response
  SEV-3: Quality degradation         → 4 hours
  SEV-4: Minor, no business impact   → Next day

INCIDENT RESPONSE FLOW:
  DETECT → TRIAGE → MITIGATE → DIAGNOSE → RESOLVE → POST-MORTEM

PIPELINE FAILURE CATEGORIES:
  1. Infrastructure (OOM, disk, network)
  2. Data (schema change, volume spike, corrupt)
  3. Code (bug, permissions, config drift)
  4. External (source outage, API limit)

TOOLS:
  Lineage:       dbt, OpenLineage, DataHub, Marquez
  Observability: Monte Carlo, Bigeye, Soda, Great Expectations
  Catalog:       DataHub, Atlan, Amundsen, Alation

MINIMUM VIABLE OBSERVABILITY:
  [ ] Freshness on top 10 tables (MAX timestamp)
  [ ] Volume baseline + alerting (row counts)
  [ ] Null rate on critical columns
  [ ] Lineage via dbt or documentation
  [ ] Incident severity definitions
  [ ] On-call rotation + runbooks

+============================================================+
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 43 of 45*
