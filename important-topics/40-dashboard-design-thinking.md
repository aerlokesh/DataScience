# 🎯 Topic 40: Dashboard Design Thinking

> *"A great dashboard answers your top questions at a glance, without requiring you to ask them. A bad dashboard is a cemetery of charts that nobody visits. Design for decision-making, not decoration."*

---

## 📑 Table of Contents

1. [Dashboard vs Report vs Analysis](#dashboard-vs-report-vs-analysis)
2. [Information Hierarchy](#information-hierarchy)
3. [Above-the-Fold Metrics](#above-the-fold-metrics)
4. [Drill-Down Patterns](#drill-down-patterns)
5. [Alert Thresholds and Monitoring](#alert-thresholds-and-monitoring)
6. [Refresh Frequency](#refresh-frequency)
7. [Audience-Specific Dashboards](#audience-specific-dashboards)
8. [Common Anti-Patterns](#common-anti-patterns)
9. [Tools Landscape](#tools-landscape)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Dashboard vs Report vs Analysis

### Key Distinctions

| Dimension | Dashboard | Report | Ad-Hoc Analysis |
|-----------|-----------|--------|-----------------|
| **Purpose** | Monitor ongoing health | Summarize a period | Answer a specific question |
| **Audience** | Daily operational users | Periodic stakeholders | Decision-makers (one-time) |
| **Update Frequency** | Real-time to daily | Weekly/Monthly/Quarterly | One-time (snapshot) |
| **Interactivity** | High (filters, drill-down) | Low (static tables/charts) | Medium (notebooks, iterative) |
| **Shelf Life** | Months to years | One reporting cycle | Until question is answered |
| **Metric Count** | 5-15 key metrics | 20-50 metrics with context | Varies (1-100+) |
| **Design Goal** | Reduce time to insight | Provide complete picture | Provide rigorous answer |

### When to Build Each

```
"How is the business doing RIGHT NOW?"      --> Dashboard
"How did we perform last quarter?"          --> Report
"Why did conversion drop last Tuesday?"     --> Ad-Hoc Analysis
"Should we launch feature X?"              --> Analysis (with recommendation)
```

> **Critical Insight:** The #1 mistake teams make is building dashboards when they need analyses, and vice versa. If the question has a definitive answer that won't change, write an analysis. If you need ongoing monitoring of a metric, build a dashboard. If you conflate these, you end up with 50-chart dashboards nobody uses.

---

## Information Hierarchy

### The Inverted Pyramid for Dashboards

```
┌─────────────────────────────────────────────┐
│         HEADLINE METRICS (KPIs)              │  <- Answer "how are we doing?"
│         2-4 big numbers with trends          │     in < 3 seconds
├─────────────────────────────────────────────┤
│         SUPPORTING CONTEXT                   │  <- Why are KPIs where they are?
│         Trend charts, comparisons            │     in < 30 seconds
├─────────────────────────────────────────────┤
│         DIAGNOSTIC DETAIL                    │  <- Where to dig deeper?
│         Segment breakdowns, tables           │     in < 2 minutes
├─────────────────────────────────────────────┤
│         EXPLORATION LAYER                    │  <- Self-serve investigation
│         Filters, drill-down, raw data        │     as needed
└─────────────────────────────────────────────┘
```

### Visual Weight Distribution

```
Top 20% of screen:    60% of importance (headline KPIs)
Middle 40% of screen: 30% of importance (context & trends)
Bottom 40% of screen: 10% of importance (detail & exploration)
```

### The 5-Second Rule

A well-designed dashboard should answer "Are we on track?" within 5 seconds of looking at it. This means:

1. **Color coding:** Green/Red/Yellow for status at a glance
2. **Comparison context:** vs. target, vs. last period, vs. benchmark
3. **Trend direction:** Up/Down arrows or sparklines
4. **Anomaly highlighting:** Call out what needs attention

---

## Above-the-Fold Metrics

### What Goes Above the Fold

"Above the fold" = visible without scrolling. This is prime real estate.

```
ABOVE THE FOLD (visible immediately):
┌─────────────────────────────────────────────┐
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐   │
│  │ DAU  │  │  Rev │  │  CVR │  │ CSAT │   │
│  │ 124K │  │ $2.1M│  │ 3.2% │  │  4.2 │   │
│  │ +5%  │  │ -2%  │  │ +0.1 │  │ flat │   │
│  │  ↑   │  │  ↓   │  │  ↑   │  │  →   │   │
│  └──────┘  └──────┘  └──────┘  └──────┘   │
│                                             │
│  [Primary Trend Chart - 30 day view]        │
│  ████████████████████████████████           │
│                                             │
└─────────────────────────────────────────────┘

BELOW THE FOLD (scroll to see):
  - Segment breakdowns
  - Secondary metrics  
  - Detailed tables
  - Exploration filters
```

### Selecting Above-the-Fold Metrics

Choose metrics that:
1. Directly tie to business objectives (North Star or key results)
2. Are actionable (someone can do something about them)
3. Change frequently enough to warrant monitoring
4. Are understood by the dashboard's primary audience

**Rule of thumb:** 3-6 headline metrics. More than 6 = nothing stands out.

> **Critical Insight:** If you can't decide which 4-5 metrics go above the fold, you haven't clarified the dashboard's purpose. A dashboard without a clear owner and decision it supports will inevitably become a metric graveyard.

---

## Drill-Down Patterns

### Progressive Disclosure

```
Level 0: Overview      "Revenue is $2.1M, down 2%"
    |
Level 1: Dimension     "Desktop revenue flat, Mobile down 8%"
    |
Level 2: Sub-Segment   "Mobile iOS down 12%, Android stable"
    |
Level 3: Detail        "iOS Safari conversion dropped after v15.4 update"
```

### Common Drill-Down Dimensions

```
Time:       Year → Quarter → Month → Week → Day → Hour
Geography:  Global → Region → Country → City
Platform:   All → Web/Mobile → iOS/Android → Device Model
Product:    All → Category → Sub-Category → SKU
Customer:   All → Segment → Cohort → Individual
Channel:    All → Organic/Paid → Specific Channel → Campaign
```

### Implementation Patterns

**1. Click-to-Filter:** Click a bar in a chart to filter everything else
**2. Linked Pages:** Top-level links to detail pages per dimension
**3. Parameter Passing:** URL parameters carry context between views
**4. Tooltip Detail:** Hover for one level of additional detail

### Drill-Down Design Rules

- Never more than 3-4 levels deep
- Each level should answer a more specific question
- Breadcrumbs so users know where they are
- "Back to top" always available
- Default view should be the most useful aggregation

---

## Alert Thresholds and Monitoring

### Types of Alerts

| Alert Type | Trigger | Example |
|-----------|---------|---------|
| Absolute Threshold | Value > X or < Y | Revenue < $100K/day |
| Relative Change | WoW or DoD > Z% | DAU drops > 15% WoW |
| Anomaly Detection | Beyond expected range | Revenue outside 2-sigma band |
| SLA Breach | Exceeds agreement | Data freshness > 4 hours |
| Trend Alert | Sustained directional change | 5 consecutive days of decline |

### Setting Good Thresholds

```python
# Data-driven threshold setting
import numpy as np

# Use historical data to calibrate
historical_values = get_last_90_days('revenue')

# Statistical approach: alert if outside normal range
mean = np.mean(historical_values)
std = np.std(historical_values)
upper_threshold = mean + 2 * std  # ~97.5th percentile
lower_threshold = mean - 2 * std  # ~2.5th percentile

# Or percentile-based (more robust to outliers)
lower_threshold = np.percentile(historical_values, 5)
upper_threshold = np.percentile(historical_values, 95)
```

### Alert Fatigue Prevention

- **Tiered severity:** Critical (page someone) vs. Warning (email) vs. Info (dashboard only)
- **Cooldown periods:** Don't re-alert for same issue within X hours
- **Composite alerts:** Require multiple signals before alerting
- **Business hours only:** Non-critical alerts only during work hours
- **Regular threshold review:** Adjust quarterly as business scales

> **Critical Insight:** An alert that fires more than once a week without requiring action should be removed or recalibrated. Alert fatigue is the single biggest killer of monitoring systems — when everything alerts, nothing alerts.

---

## Refresh Frequency

### Choosing Refresh Rate

| Use Case | Refresh Rate | Example |
|----------|-------------|---------|
| Real-time operations | Seconds to minutes | Fraud detection, live traffic |
| Operational monitoring | 15 min to 1 hour | Customer support queue, server health |
| Business dashboards | Daily (morning refresh) | Revenue, DAU, conversion |
| Executive dashboards | Daily to weekly | Strategic KPIs, OKR tracking |
| Monthly reviews | Monthly | Cohort analysis, financial reporting |

### Cost-Benefit of Faster Refresh

```
Real-time:   Expensive (streaming infra), justified only if action 
             must happen within minutes (fraud, outages)

Hourly:      Moderate cost, good for operations teams making 
             intra-day decisions

Daily:       Low cost, sufficient for 90% of business dashboards
             (refreshed by 9 AM for morning review)

Weekly:      Minimal cost, appropriate for strategic metrics that 
             don't change meaningfully day-to-day
```

### Implementation Considerations

- **Incremental vs. full refresh:** Append new data vs. recompute everything
- **Cache layers:** Pre-aggregate common views, refresh cache on schedule
- **Freshness indicator:** Always show "Last updated: X" on the dashboard
- **Fallback:** If refresh fails, show stale data with warning, not blank dashboard

---

## Audience-Specific Dashboards

### Executive Dashboard

```
Audience: C-suite, VP+
Metrics:  North Star, Revenue, Growth Rate, Key OKR progress
Design:   
  - MAX 5-6 metrics
  - Heavy use of comparisons (vs target, vs last year)
  - Traffic light status (on-track, at-risk, off-track)
  - Minimal interactivity (they won't click)
  - Monthly/quarterly trends only
  - Commentary/narrative alongside numbers
```

### Analyst Dashboard (Self-Serve)

```
Audience: Data analysts, product managers
Metrics:  Wide coverage, multiple dimensions
Design:   
  - Extensive filters and parameters
  - Drill-down to raw data
  - Segment comparison tools
  - Date range selectors
  - Export to CSV/notebook capability
  - Custom calculated fields
```

### Operational Dashboard

```
Audience: Operations teams, on-call engineers
Metrics:  Real-time health indicators, queue lengths, SLAs
Design:   
  - Real-time or near-real-time refresh
  - Clear red/yellow/green status
  - Alerts and anomaly flags front and center
  - Runbook links next to alerting metrics
  - Historical context (is this normal for this time?)
  - Mobile-friendly (check from phone)
```

### Comparison Table

| Feature | Executive | Analyst | Operations |
|---------|-----------|---------|------------|
| Metrics count | 5-6 | 20-50 | 10-15 |
| Interactivity | Low | High | Medium |
| Refresh | Daily/Weekly | Hourly/Daily | Real-time |
| Complexity | Simple | Complex | Moderate |
| Context | vs. targets | vs. segments | vs. thresholds |
| Action | Strategic decisions | Deep investigation | Immediate response |
| Chart types | KPI cards, sparklines | Scatter, cohort, funnel | Gauges, timeseries |

---

## Common Anti-Patterns

### The Metric Graveyard

```
Problem:  50+ charts, no hierarchy, no one knows what to look at
Cause:    "Add this metric too" accumulation over months
Fix:      Ruthlessly prioritize. Archive unused charts. 
          Rule: if no one looked at it in 30 days, remove it.
```

### The Data Dump

```
Problem:  Tables of raw numbers with no visualization or context
Cause:    SQL query results pasted into a dashboard tool
Fix:      Every number needs context — is it good or bad? 
          Compared to what? Trending which way?
```

### The Vanity Dashboard

```
Problem:  All metrics are "up and to the right" — no diagnostic value
Cause:    Designed to impress, not inform. Only shows flattering metrics.
Fix:      Include leading indicators and segments where problems hide.
          A dashboard that never shows bad news isn't useful.
```

### The Swiss Army Knife

```
Problem:  One dashboard tries to serve executives, analysts, and ops
Cause:    "Let's just have one dashboard for everything"
Fix:      Different audiences need different dashboards. 
          Build audience-specific views.
```

### The Misleading Chart

```
Problem:  Truncated Y-axis, wrong chart type, misleading comparisons
Cause:    Tool defaults or careless design
Fix:      Y-axis starts at 0 for bar charts, clear labels, 
          appropriate chart type for data type
```

### The Stale Dashboard

```
Problem:  Data is days/weeks old, no freshness indicator
Cause:    Pipeline broke, no one noticed, no monitoring
Fix:      Always show "last updated" timestamp.
          Alert if data is staler than SLA.
```

> **Critical Insight:** The biggest anti-pattern is building dashboards without a clear owner and use case. Before building, answer: Who will use this? What decision will it inform? How often will they check it? If you can't answer these, don't build it.

---

## Tools Landscape

### Major BI Tools Comparison

| Tool | Strengths | Weaknesses | Best For |
|------|-----------|------------|----------|
| **Looker** | Strong modeling layer (LookML), governed metrics, git-versioned | Steep learning curve, expensive, slow for exploration | Enterprises wanting single source of truth |
| **Tableau** | Beautiful visuals, powerful exploration, large community | Expensive at scale, governance challenges, extract-based | Visual exploration and storytelling |
| **Mode** | SQL-native, notebooks + dashboards, analyst-friendly | Less polished for non-technical users | Data teams doing analysis and sharing |
| **Superset** | Open-source, SQL-native, lightweight | Less polished UX, fewer enterprise features | Budget-conscious teams, engineering orgs |
| **Metabase** | Easy setup, self-serve friendly, open-source option | Less powerful for complex analyses | Small teams needing quick self-serve BI |
| **Power BI** | Microsoft integration, affordable, DAX is powerful | Microsoft ecosystem lock-in, less flexible | Microsoft-heavy organizations |
| **Hex** | Notebooks + dashboards, Python/SQL, reactive | Newer product, smaller community | Modern data teams wanting code + viz |
| **Sigma** | Spreadsheet-like interface, live connection | Newer, less adoption | Business users who think in spreadsheets |

### Choosing a Tool

```
Decision factors:
1. Who is the primary user? (Technical → Mode/Hex, Business → Tableau/Sigma)
2. Data governance requirements? (High → Looker, Low → Any)
3. Budget? (Low → Superset/Metabase, High → Looker/Tableau)
4. Existing ecosystem? (AWS → QuickSight, Google → Looker, Microsoft → Power BI)
5. Customization needs? (High → Hex/Mode/custom, Low → Metabase/Sigma)
```

---

## Interview Talking Points

> "When I design dashboards, I start with the user and the decision, not the data. For our exec team, I built a one-page 'Business Pulse' with 4 KPIs above the fold: revenue vs. target, active users vs. plan, conversion rate with 7-day trend, and NPS. Each had a traffic light status so the CEO could assess health in under 5 seconds. Details were available on click but rarely used — the headline told the story."

> "I inherited a 40-chart Looker dashboard that had 2 views per week. I interviewed stakeholders, found that only 6 charts informed actual decisions, and rebuilt it as a focused 6-chart dashboard with clear hierarchy. Usage went to 15 views per day. The lesson: less is more when each chart earns its place."

> "For alert thresholds, I used a data-driven approach — computed the historical 5th and 95th percentile of daily revenue, and set alerts at those boundaries. This initially fired 3 times per month, which was our team's capacity. As the business grew, I automatically recalibrated thresholds quarterly to maintain that alert frequency."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Building without knowing the audience | Define the primary user and their top 3 questions first |
| Too many metrics on one page | 5-6 above the fold, progressive disclosure for the rest |
| No comparison context for numbers | Every metric needs vs. target, vs. last period, or vs. benchmark |
| Same dashboard for executives and analysts | Build audience-specific views with appropriate detail level |
| Never archiving unused charts | Review quarterly; remove charts with < 5 views/month |
| Real-time refresh when daily suffices | Match refresh to decision cadence; faster = more expensive |
| No freshness indicator | Always show "Last Updated" timestamp prominently |
| Designing for data exploration (that's analysis) | Dashboards are for monitoring; separate exploration tool for deep dives |
| Inconsistent metric definitions across dashboards | Use a semantic layer (LookML, dbt metrics) for single source of truth |
| No mobile consideration for exec dashboards | Executives check on phones; design key metrics for mobile viewability |

---

## Rapid-Fire Q&A

**Q1: What's the difference between a dashboard and a report?**
A dashboard is a living, regularly-refreshed monitoring tool. A report is a point-in-time summary of a specific period, typically with narrative context.

**Q2: How many metrics should be above the fold?**
3-6. Fewer than 3 is too sparse; more than 6 and nothing stands out.

**Q3: How do you decide refresh frequency?**
Match it to the decision cadence. If decisions are made daily, refresh daily. If hourly operational decisions happen, refresh hourly. Never refresh faster than needed.

**Q4: What makes a good KPI card?**
The metric value, comparison context (vs target or prior period), trend direction (arrow or sparkline), and status color (green/yellow/red).

**Q5: How do you avoid alert fatigue?**
Tiered severity, data-driven thresholds (not arbitrary), cooldown periods, and regular recalibration. Target: critical alerts < 2/week.

**Q6: When should you use a bar chart vs line chart?**
Line charts for time series (continuous data showing trends). Bar charts for categorical comparisons (discrete groups). Never use a line for non-ordered categories.

**Q7: What's the biggest dashboard anti-pattern?**
The metric graveyard — 50+ charts accumulated over time with no hierarchy, no owner, and no one who actually uses them for decisions.

**Q8: How do you handle stakeholders who want "just one more chart"?**
Ask: "What decision will this inform? How often? Is it already answerable from existing charts?" If it passes, add it. If not, suggest an ad-hoc analysis instead.

**Q9: Should dashboards have commentary/narrative?**
Executive dashboards benefit from brief narrative context. Operational dashboards should be self-explanatory through design (labels, colors, thresholds).

**Q10: How do you measure if a dashboard is successful?**
Usage metrics (views/day, unique viewers), time-to-decision (are meetings shorter?), and stakeholder feedback. A great dashboard reduces the number of ad-hoc data requests.

---

## ASCII Cheat Sheet

```
+============================================================+
|          DASHBOARD DESIGN THINKING — CHEAT SHEET            |
+============================================================+

DASHBOARD vs REPORT vs ANALYSIS:
  Dashboard: Ongoing monitoring (living, refreshed)
  Report:    Period summary (static, narrative)
  Analysis:  Answer specific question (one-time, deep)

INFORMATION HIERARCHY:
  Top:     KPI cards (3-6 metrics, 5-second read)
  Middle:  Trend charts (30-second context)
  Bottom:  Segment detail (2-min investigation)
  Hidden:  Exploration filters (on-demand)

ABOVE-THE-FOLD CHECKLIST:
  [ ] Metric value (big, readable number)
  [ ] Comparison (vs target, vs last period)
  [ ] Direction (up/down arrow or sparkline)
  [ ] Status (green/yellow/red)
  [ ] Label (clear metric name)
  MAX 6 CARDS.

AUDIENCE-SPECIFIC:
  Executive: 5 metrics, weekly, traffic lights, no filters
  Analyst:   Many metrics, daily, heavy filters, drill-down
  Operations: Real-time, alerts, thresholds, runbook links

DRILL-DOWN DEPTH: Max 3-4 levels
  L0: Overview  → L1: Dimension → L2: Segment → L3: Detail

REFRESH FREQUENCY:
  Real-time:  Operations, fraud, live events
  Hourly:     Support queue, intraday ops
  Daily:      Business metrics (90% of dashboards)
  Weekly:     Strategic/exec metrics

ANTI-PATTERNS TO AVOID:
  - Metric graveyard (50+ unused charts)
  - No context (numbers without comparison)
  - Swiss army knife (one dashboard for all)
  - Vanity dashboard (only good news)
  - Stale data (no freshness indicator)

TOOL SELECTION:
  Governed metrics + enterprise → Looker
  Beautiful exploration         → Tableau
  SQL-native analysts           → Mode / Hex
  Budget-conscious              → Superset / Metabase
  Microsoft shop                → Power BI

DESIGN PROCESS:
  1. WHO is the user? (persona)
  2. WHAT decision does this support?
  3. HOW OFTEN will they check?
  4. WHAT metrics answer their questions?
  5. Build, test with user, iterate

+============================================================+
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 40 of 45*
