# 🎯 Topic 41: Data Quality Frameworks and Validation

> *"Bad data is worse than no data — it leads to confidently wrong decisions. A mature data team treats data quality with the same rigor that software engineers treat code quality: automated testing, CI/CD, and clear ownership."*

---

## 📑 Table of Contents

1. [Data Quality Dimensions](#data-quality-dimensions)
2. [Testing Strategies](#testing-strategies)
3. [dbt Tests and Data Contracts](#dbt-tests-and-data-contracts)
4. [Great Expectations Concepts](#great-expectations-concepts)
5. [Data Contracts and SLAs](#data-contracts-and-slas)
6. [Alerting Strategies](#alerting-strategies)
7. [Building a Data Quality Program](#building-a-data-quality-program)
8. [SQL Patterns for Validation](#sql-patterns-for-validation)
9. [Interview Talking Points](#interview-talking-points)
10. [Common Mistakes](#common-mistakes)
11. [Rapid-Fire Q&A](#rapid-fire-qa)
12. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Data Quality Dimensions

### The Six Dimensions of Data Quality

| Dimension | Definition | Example Violation |
|-----------|-----------|-------------------|
| **Completeness** | All expected data is present | NULL email for 30% of users |
| **Accuracy** | Data reflects reality | Revenue stored in wrong currency |
| **Consistency** | Same data doesn't contradict itself | User age differs between tables |
| **Timeliness** | Data is available when needed | Dashboard shows yesterday's data at 2 PM |
| **Uniqueness** | No unwanted duplicates | Same transaction recorded twice |
| **Validity** | Data conforms to expected format/range | Age = -5, or email without @ |

### Extended Dimensions

| Dimension | Definition | Example |
|-----------|-----------|---------|
| **Referential Integrity** | Foreign keys point to valid records | order.user_id not in users table |
| **Conformity** | Data follows defined formats | Mixed date formats: 2024-01-15 vs 01/15/2024 |
| **Relevance** | Data is useful for its intended purpose | Storing deprecated fields no one queries |
| **Precision** | Appropriate level of granularity | Lat/long truncated to 2 decimal places |

### Impact Assessment Matrix

```
                 High Business Impact
                        │
    ┌───────────────────┼───────────────────┐
    │                   │                   │
    │   CRITICAL        │   HIGH PRIORITY   │
    │   (Fix NOW)       │   (Fix this week) │
    │   Revenue data    │   User PII        │
    │   wrong           │   incomplete      │
    │                   │                   │
────┼───────────────────┼───────────────────┼────
    │                   │                   │    Affects
    │   MONITOR         │   LOW PRIORITY    │    Many/Few
    │   (Track, batch)  │   (Backlog)       │    Records
    │   Minor format    │   Legacy field    │
    │   issues          │   inconsistency   │
    │                   │                   │
    └───────────────────┼───────────────────┘
                        │
                 Low Business Impact
```

> **Critical Insight:** Not all data quality issues are equally important. A completeness issue in a revenue column is catastrophic; the same issue in an optional description field might be acceptable. Always prioritize by business impact, not technical severity.

---

## Testing Strategies

### Test Categories

**1. Schema Tests (Structure)**
```sql
-- Column exists, correct type, not null constraints
-- Table has expected columns
-- Data types match specification
```

**2. Volume Tests (Quantity)**
```sql
-- Row count within expected range
-- Row count matches source system
-- No unexpected empty tables
SELECT COUNT(*) FROM orders WHERE order_date = CURRENT_DATE;
-- Expected: between 50,000 and 200,000 (based on historical range)
```

**3. Freshness Tests (Timeliness)**
```sql
-- Most recent record within expected window
SELECT MAX(updated_at) FROM events;
-- Should be within last 2 hours for this table
```

**4. Null Checks (Completeness)**
```sql
-- Critical columns have acceptable null rates
SELECT 
    COUNT(*) AS total_rows,
    SUM(CASE WHEN email IS NULL THEN 1 ELSE 0 END) AS null_emails,
    ROUND(100.0 * SUM(CASE WHEN email IS NULL THEN 1 ELSE 0 END) / COUNT(*), 2) AS null_pct
FROM users;
-- null_pct should be < 1%
```

**5. Distribution Tests (Accuracy)**
```sql
-- Values fall within expected ranges
-- Distribution hasn't shifted dramatically
SELECT 
    AVG(order_value) AS avg_aov,
    STDDEV(order_value) AS std_aov,
    MIN(order_value) AS min_aov,
    MAX(order_value) AS max_aov,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY order_value) AS p99
FROM orders
WHERE order_date = CURRENT_DATE;
-- avg_aov should be between $40-$80 (historical range)
```

**6. Referential Integrity Tests (Consistency)**
```sql
-- Foreign keys resolve correctly
SELECT COUNT(*) AS orphan_orders
FROM orders o
LEFT JOIN users u ON o.user_id = u.user_id
WHERE u.user_id IS NULL;
-- Should be 0
```

**7. Uniqueness Tests**
```sql
-- Primary keys are actually unique
SELECT order_id, COUNT(*) AS cnt
FROM orders
GROUP BY order_id
HAVING COUNT(*) > 1;
-- Should return 0 rows
```

**8. Cross-System Consistency**
```sql
-- Revenue in orders table matches revenue in financial system
SELECT 
    SUM(o.revenue) AS orders_revenue,
    SUM(f.revenue) AS finance_revenue,
    ABS(SUM(o.revenue) - SUM(f.revenue)) AS discrepancy
FROM orders o
FULL OUTER JOIN finance f ON o.order_id = f.order_id
WHERE o.order_date = '2024-03-15';
-- discrepancy should be < $100 (rounding tolerance)
```

### Testing Pyramid for Data

```
          /\
         /  \  End-to-End (Cross-system reconciliation)
        /    \   FEW: Expensive, slow, high-value
       /------\
      /        \  Integration (Referential integrity, joins)
     /          \   SOME: Cross-table consistency
    /------------\
   /              \  Unit (Column-level checks)
  /                \   MANY: Fast, cheap, granular
 /==================\
  Null, type, range, uniqueness, format
```

> **Critical Insight:** Most data quality issues are caught by the simplest tests — null checks, row counts, and uniqueness constraints. Invest heavily in these "unit tests" before building complex cross-system validations.

---

## dbt Tests and Data Contracts

### Built-in dbt Tests

```yaml
# schema.yml
version: 2

models:
  - name: orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: user_id
        tests:
          - not_null
          - relationships:
              to: ref('users')
              field: user_id
      - name: status
        tests:
          - accepted_values:
              values: ['pending', 'shipped', 'delivered', 'cancelled']
      - name: order_value
        tests:
          - not_null
```

### Custom dbt Tests

```sql
-- tests/assert_positive_revenue.sql
-- This test fails if any row has negative revenue (returns failing rows)
SELECT order_id, revenue
FROM {{ ref('orders') }}
WHERE revenue < 0

-- tests/assert_recent_data.sql
-- Fails if no data from today
SELECT 1
WHERE NOT EXISTS (
    SELECT 1 FROM {{ ref('orders') }}
    WHERE order_date = CURRENT_DATE
)
```

### dbt Test Severity Levels

```yaml
models:
  - name: orders
    columns:
      - name: order_id
        tests:
          - unique:
              severity: error    # Block pipeline
          - not_null:
              severity: error    # Block pipeline
      - name: promo_code
        tests:
          - not_null:
              severity: warn     # Log but don't block
```

### Data Contracts

A **data contract** is a formal agreement between a data producer and consumer about the shape, quality, and SLA of data.

```yaml
# data_contract.yml
contract:
  name: orders_daily
  version: 2.1.0
  owner: payments-team
  
  schema:
    - name: order_id
      type: STRING
      required: true
      unique: true
    - name: user_id
      type: STRING
      required: true
    - name: order_value
      type: DECIMAL(10,2)
      required: true
      constraints:
        min: 0.01
        max: 100000.00
    - name: created_at
      type: TIMESTAMP
      required: true
      freshness:
        max_delay: 2 hours
  
  sla:
    freshness: 4 hours
    completeness: 99.5%
    availability: 99.9%
  
  consumers:
    - analytics-team
    - ml-team
    - finance-team
```

> **Critical Insight:** Data contracts shift data quality from "hope" to "guarantee." They make the producer accountable for quality and give consumers confidence. Without contracts, every downstream team is independently validating (or worse, discovering issues in production dashboards).

---

## Great Expectations Concepts

### Core Concepts

```python
import great_expectations as gx

# Context: manages connections, data sources, and expectations
context = gx.get_context()

# Data Source: connection to your data
datasource = context.sources.add_pandas("my_source")

# Expectations: individual quality checks
batch = datasource.read_dataframe(df)

# Define expectations
batch.expect_column_values_to_not_be_null("user_id")
batch.expect_column_values_to_be_between("age", min_value=0, max_value=150)
batch.expect_column_values_to_be_unique("order_id")
batch.expect_column_values_to_match_regex("email", r"^[\w\.-]+@[\w\.-]+\.\w+$")
batch.expect_table_row_count_to_be_between(min_value=10000, max_value=500000)
```

### Expectation Categories

| Category | Examples |
|----------|----------|
| Table-level | Row count between X and Y, column count matches |
| Column existence | Column exists, column type matches |
| Null handling | Column not null, null percentage < X% |
| Value range | Values between min/max, no negative values |
| Set membership | Values in allowed set, no unexpected categories |
| String patterns | Matches regex, length between X and Y |
| Statistical | Mean between X and Y, stdev < threshold |
| Distribution | KL divergence < threshold vs. reference |
| Cross-column | Column A > Column B, sum equals total |

### Validation Workflow

```
1. Profile:    Automatically generate expectations from data
2. Review:     Human reviews and adjusts generated expectations
3. Validate:   Run expectations against new data batches
4. Document:   Auto-generate data docs (HTML reports)
5. Alert:      Notify on failures, block pipeline if critical
```

---

## Data Contracts and SLAs

### Freshness SLAs

```
Tier 1 (Real-time):    < 5 minutes  (streaming data, fraud signals)
Tier 2 (Near-real):    < 1 hour     (operational dashboards)
Tier 3 (Daily):        < 4 hours    (business dashboards, by 9 AM)
Tier 4 (Batch):        < 24 hours   (weekly reports, ML training)
```

### Completeness SLAs

```
Critical columns:      99.9% non-null (order_id, user_id, timestamp)
Important columns:     99.0% non-null (email, country, device)
Optional columns:      95.0% non-null (promo_code, referrer, utm_source)
Best-effort columns:   No SLA         (free-text fields, metadata)
```

### Defining an SLA Document

```
DATA SLA: user_events table
─────────────────────────────
Producer:     Tracking Engineering Team
Consumer:     Product Analytics, ML Platform
Refresh:      Continuous (streaming), batch reconciliation daily at 3 AM UTC
Freshness:    Max 15 minutes delay (streaming), full consistency by 6 AM UTC
Volume:       Expect 50M-200M rows/day. Alert if < 40M or > 250M.
Schema:       Changes require 2-week notice and consumer sign-off.
Completeness: user_id 99.99%, event_type 99.99%, timestamp 99.99%
              device_id 98%, session_id 97%, geo_country 95%
Incident SLA: Producer acknowledges quality issues within 1 hour.
              Resolution within 4 hours for Tier 1, 24 hours for Tier 2.
```

> **Critical Insight:** SLAs without enforcement are suggestions. Implement automated monitoring that alerts when SLAs are breached, and establish clear escalation paths. A data SLA should be as binding as a service uptime SLA.

---

## Alerting Strategies

### Alert Hierarchy

```
Level 1 — CRITICAL (Page on-call):
  - Table missing (pipeline completely broken)
  - Revenue data more than 6 hours stale
  - Primary key duplicates detected
  - Row count drops > 50% from expected

Level 2 — WARNING (Slack notification):
  - Null rate exceeds threshold (but not catastrophic)
  - Freshness approaching SLA limit
  - Distribution shift detected (p-value < 0.01)
  - Row count outside 2-sigma range

Level 3 — INFO (Dashboard only):
  - Minor completeness issues in optional fields
  - Slight distribution drift
  - Schema change detected (but backward compatible)
```

### Alert Design Principles

```python
# Good alert: specific, actionable, not noisy
alert = {
    "name": "orders_table_freshness",
    "condition": "MAX(created_at) < NOW() - INTERVAL '4 hours'",
    "severity": "critical",
    "owner": "data-engineering-oncall",
    "runbook": "https://wiki/runbooks/orders-freshness",
    "cooldown": "2 hours",  # Don't re-alert within 2 hours
    "context": "Check Airflow DAG 'orders_etl' and upstream Kafka lag"
}
```

### Reducing False Positives

1. **Baseline with day-of-week seasonality:** Don't alert on normal Monday dips
2. **Use rolling averages:** Alert on 3-day average deviation, not single-day
3. **Composite conditions:** Require BOTH freshness AND volume anomaly
4. **Grace periods:** Allow 30-minute window before escalating
5. **Scheduled maintenance windows:** Suppress during known downtime

---

## Building a Data Quality Program

### Maturity Model

```
Level 1 — Reactive:
  - Issues discovered by end users ("The dashboard looks wrong")
  - No systematic testing
  - Fix issues one-by-one when reported

Level 2 — Basic Testing:
  - Null checks and row counts on critical tables
  - Manual review before major decisions
  - Basic alerts (email on failure)

Level 3 — Proactive:
  - Automated test suites (dbt tests, Great Expectations)
  - Data contracts with producers
  - Freshness and completeness SLAs
  - Tiered alerting with on-call rotation

Level 4 — Predictive:
  - Anomaly detection on distributions
  - Schema change impact analysis
  - Data lineage-based impact assessment
  - Self-healing pipelines (auto-retry, fallback)

Level 5 — Cultural:
  - Data quality is everyone's responsibility
  - Quality metrics are OKRs for data teams
  - Consumers trust data because trust is earned continuously
```

### Quick Win Implementation Plan

```
Week 1-2: Identify top 10 most-queried tables
Week 3-4: Add null, uniqueness, and freshness tests
Week 5-6: Set up alerting (Slack) for failures
Week 7-8: Establish SLAs with top 3 producers
Week 9-10: Build data quality dashboard (meta-monitoring)
Week 11-12: Document and communicate data contracts
```

---

## SQL Patterns for Validation

### Comprehensive Table Health Check

```sql
WITH table_stats AS (
    SELECT
        COUNT(*) AS total_rows,
        COUNT(DISTINCT order_id) AS unique_orders,
        MAX(created_at) AS latest_record,
        MIN(created_at) AS earliest_record,
        -- Completeness
        ROUND(100.0 * SUM(CASE WHEN user_id IS NULL THEN 1 ELSE 0 END) / COUNT(*), 2) AS null_user_pct,
        ROUND(100.0 * SUM(CASE WHEN revenue IS NULL THEN 1 ELSE 0 END) / COUNT(*), 2) AS null_revenue_pct,
        -- Validity
        SUM(CASE WHEN revenue < 0 THEN 1 ELSE 0 END) AS negative_revenue_count,
        SUM(CASE WHEN revenue > 100000 THEN 1 ELSE 0 END) AS suspicious_high_revenue,
        -- Freshness
        EXTRACT(EPOCH FROM (NOW() - MAX(created_at))) / 3600 AS hours_since_latest,
        -- Uniqueness
        COUNT(*) - COUNT(DISTINCT order_id) AS duplicate_count
    FROM orders
    WHERE created_at >= CURRENT_DATE - INTERVAL '1 day'
)
SELECT 
    *,
    CASE 
        WHEN hours_since_latest > 4 THEN 'STALE'
        WHEN null_user_pct > 1 THEN 'COMPLETENESS_ISSUE'
        WHEN duplicate_count > 0 THEN 'DUPLICATES'
        WHEN negative_revenue_count > 0 THEN 'VALIDITY_ISSUE'
        ELSE 'HEALTHY'
    END AS overall_status
FROM table_stats;
```

### Distribution Drift Detection

```sql
-- Compare this week's distribution to last week's
WITH current_dist AS (
    SELECT 
        NTILE(10) OVER (ORDER BY order_value) AS decile,
        AVG(order_value) AS avg_value,
        COUNT(*) AS cnt
    FROM orders
    WHERE order_date >= CURRENT_DATE - INTERVAL '7 days'
    GROUP BY 1
),
previous_dist AS (
    SELECT 
        NTILE(10) OVER (ORDER BY order_value) AS decile,
        AVG(order_value) AS avg_value,
        COUNT(*) AS cnt
    FROM orders
    WHERE order_date BETWEEN CURRENT_DATE - INTERVAL '14 days' 
                         AND CURRENT_DATE - INTERVAL '7 days'
    GROUP BY 1
)
SELECT 
    c.decile,
    c.avg_value AS current_avg,
    p.avg_value AS previous_avg,
    ROUND(100.0 * (c.avg_value - p.avg_value) / p.avg_value, 1) AS pct_shift
FROM current_dist c
JOIN previous_dist p ON c.decile = p.decile
ORDER BY c.decile;
```

### Cross-Table Reconciliation

```sql
-- Ensure source and target tables agree
SELECT 
    'source' AS system,
    COUNT(*) AS row_count,
    SUM(amount) AS total_amount,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM source_payments
WHERE payment_date = '2024-03-15'

UNION ALL

SELECT 
    'target' AS system,
    COUNT(*) AS row_count,
    SUM(amount) AS total_amount,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM warehouse_payments
WHERE payment_date = '2024-03-15';
```

---

## Interview Talking Points

> "I built our data quality framework from scratch using dbt tests and custom SQL assertions. We started by profiling our top 20 tables — measuring null rates, distribution stats, and freshness — then set thresholds at p5/p95 of historical values. Within 3 months we went from discovering issues through broken dashboards to catching 95% of issues before they impacted consumers."

> "I implemented data contracts between our tracking team (producer) and analytics team (consumer). We formalized SLAs: event data must be available within 4 hours, user_id must be 99.99% non-null, and schema changes require 2-week notice. The contract reduced our 'data fire' incidents from 4 per month to fewer than 1."

> "For distribution drift detection, I built a daily job that computes KL divergence between today's value distributions and a 30-day rolling baseline. When divergence exceeds a threshold, it triggers a Slack alert with the specific column and a comparison histogram. This caught a currency conversion bug within 2 hours that previously would have gone unnoticed for days."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Only testing after issues are reported | Proactively test on every pipeline run (CI for data) |
| Setting arbitrary thresholds (null rate < 5%) | Use data-driven thresholds based on historical distribution |
| Testing only critical tables, ignoring intermediate ones | Test at every transformation step; bugs in staging propagate |
| Not having a freshness indicator on dashboards | Always surface "last updated" and alert on staleness |
| Same alert severity for all issues | Tier alerts: critical (page), warning (Slack), info (dashboard) |
| No runbook for alert resolution | Every alert needs: what it means, who owns it, how to fix it |
| Treating data quality as purely an engineering problem | Quality is shared responsibility: engineering, analytics, and business |
| Checking quality only at ingestion | Test at ingestion AND after transformation AND at serving layer |

---

## Rapid-Fire Q&A

**Q1: What are the six dimensions of data quality?**
Completeness, accuracy, consistency, timeliness, uniqueness, and validity.

**Q2: What's the difference between accuracy and validity?**
Validity means data conforms to expected format/range (age > 0). Accuracy means data reflects reality (the age stored is actually the person's real age). Data can be valid but inaccurate.

**Q3: What's a data contract?**
A formal agreement between producer and consumer specifying schema, quality thresholds, SLAs, and change management processes for a dataset.

**Q4: How do you detect distribution drift?**
Compare current period's distribution to a baseline using KL divergence, Kolmogorov-Smirnov test, or simpler methods like percentile comparison or mean/stdev shift detection.

**Q5: What's the difference between dbt tests and Great Expectations?**
dbt tests are SQL-based, tightly integrated with dbt's DAG, and run during transformation. Great Expectations is framework-agnostic, supports Python, offers richer statistical checks, and auto-generates documentation.

**Q6: How do you handle false positive alerts?**
Increase threshold sensitivity, add day-of-week seasonality adjustment, use composite conditions (require 2+ signals), and implement cooldown periods.

**Q7: What's referential integrity and why does it matter?**
Ensuring foreign keys point to valid primary keys. Broken referential integrity means JOINs silently drop rows, leading to under-counting.

**Q8: What freshness SLA would you set for a daily business dashboard?**
Data should be available by 9 AM local time for the business user. If the pipeline typically completes by 6 AM, set the SLA at 4 hours latency with alerting at 3 hours.

**Q9: How do you prioritize which tables to test first?**
By business impact and query frequency. Test tables that feed executive dashboards, financial reports, and ML models first. Use query logs to identify most-queried tables.

**Q10: What's the testing pyramid for data?**
Many cheap unit tests (null, type, range checks) at the base, some integration tests (referential integrity, cross-table) in the middle, and few expensive end-to-end tests (cross-system reconciliation) at the top.

---

## ASCII Cheat Sheet

```
+============================================================+
|       DATA QUALITY FRAMEWORKS — CHEAT SHEET                 |
+============================================================+

SIX DIMENSIONS:
  C - Completeness  (all data present?)
  A - Accuracy      (reflects reality?)
  C - Consistency   (no contradictions?)
  T - Timeliness    (available when needed?)
  U - Uniqueness    (no duplicates?)
  V - Validity      (correct format/range?)

TEST TYPES (from cheapest to most expensive):
  1. Schema:       Column exists, correct type
  2. Null:         Critical columns not null
  3. Uniqueness:   Primary keys are unique
  4. Range:        Values within expected bounds
  5. Freshness:    Latest record within SLA
  6. Volume:       Row count within range
  7. Referential:  Foreign keys resolve
  8. Distribution: Statistical profile stable
  9. Cross-system: Source matches target

ALERT TIERS:
  CRITICAL → Page on-call (table missing, pipeline dead)
  WARNING  → Slack channel (threshold breach, drift)
  INFO     → Dashboard only (minor, non-urgent)

DATA CONTRACT COMPONENTS:
  [ ] Schema definition (columns, types, constraints)
  [ ] Quality thresholds (null rates, valid ranges)
  [ ] Freshness SLA (max acceptable delay)
  [ ] Change management (notice period, review process)
  [ ] Ownership (producer team, consumer teams)
  [ ] Incident response (acknowledgment, resolution SLA)

dbt TEST TYPES:
  Built-in:  unique, not_null, accepted_values, relationships
  Custom:    Any SQL that returns failing rows
  Severity:  error (blocks), warn (logs)

FRESHNESS TIERS:
  Real-time:  < 5 min  (streaming, fraud)
  Near-real:  < 1 hour (operational)
  Daily:      < 4 hours (business dashboards)
  Batch:      < 24 hours (weekly reports)

MATURITY MODEL:
  L1: Reactive    (users find bugs)
  L2: Basic       (some null checks)
  L3: Proactive   (automated suites, SLAs)
  L4: Predictive  (anomaly detection, lineage)
  L5: Cultural    (quality is an OKR)

+============================================================+
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 41 of 45*
