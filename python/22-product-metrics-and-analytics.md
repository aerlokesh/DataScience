# 🎯 Topic 22: Product Metrics & Analytics

> **Scope:** The definitive guide to product metrics, analytics frameworks, and metric-driven decision-making for Data Science interviews at top tech companies. Covers engagement, revenue, growth, and experimentation metrics with Python implementations, real interview scenarios, and structured frameworks for metric definition, decomposition, and investigation.

---

## Table of Contents

1. [Core Engagement Metrics](#1-core-engagement-metrics)
2. [Retention & Churn Metrics](#2-retention--churn-metrics)
3. [Revenue & Monetization Metrics](#3-revenue--monetization-metrics)
4. [Funnel Analysis](#4-funnel-analysis)
5. [Cohort Analysis](#5-cohort-analysis)
6. [Growth Accounting Framework](#6-growth-accounting-framework)
7. [Metric Decomposition & Trees](#7-metric-decomposition--trees)
8. [North Star Metrics by Company](#8-north-star-metrics-by-company)
9. [A/B Test Metric Selection](#9-ab-test-metric-selection)
10. [Interview Scenarios & Frameworks](#10-interview-scenarios--frameworks)
11. [Python Implementations](#11-python-implementations)
12. [Interview Talking Points & Scripts](#12-interview-talking-points--scripts)
13. [Common Interview Mistakes](#13-common-interview-mistakes)
14. [Rapid-Fire Q&A](#14-rapid-fire-qa)
15. [Summary Cheat Sheet](#15-summary-cheat-sheet)

---

## 1. Core Engagement Metrics

### DAU / WAU / MAU

| Metric | Definition | Typical Use Case |
|--------|-----------|-----------------|
| **DAU** (Daily Active Users) | Unique users who perform a qualifying action in a 24-hour window | Social media, messaging apps |
| **WAU** (Weekly Active Users) | Unique users active in a 7-day rolling window | Professional tools, marketplaces |
| **MAU** (Monthly Active Users) | Unique users active in a 30-day rolling window | Subscription products, platforms |

> **Critical Insight:** "Active" must be carefully defined per product. For Facebook, it might be any login. For Spotify, it could be streaming at least one song. For Uber, it could be completing a ride. Always clarify the "qualifying action" before calculating DAU/WAU/MAU.

### Stickiness

Stickiness measures how habitually users return to a product.

| Stickiness Ratio | Formula | Benchmark |
|-----------------|---------|-----------|
| Daily Stickiness | DAU / MAU | Social: 50%+, SaaS: 20-30% |
| Weekly Stickiness | WAU / MAU | Social: 70%+, SaaS: 40-60% |

A DAU/MAU ratio of 0.5 means that on any given day, 50% of monthly users are active — indicating strong daily habit formation.

### Engagement Depth Metrics

| Metric | Formula | What It Reveals |
|--------|---------|----------------|
| Sessions per DAU | Total sessions / DAU | Multi-session behavior |
| Time Spent per Session | Total time / Total sessions | Content depth |
| Actions per Session | Total actions / Total sessions | Feature utilization |
| L7 (Days active in 7 days) | Count of active days per user in 7-day window | Engagement frequency |

> **Critical Insight:** L7 distribution is more informative than averages. A bimodal L7 (users at 1 day vs 6-7 days) suggests two distinct user segments requiring different strategies.

---

## 2. Retention & Churn Metrics

### Day-N Retention

Day-N retention measures the percentage of users who return on exactly day N (or within day N) after their first action.

| Retention Type | Definition | Use Case |
|---------------|-----------|----------|
| **Classic (Day-N)** | % of cohort active on exactly day N | Gaming, social apps |
| **Rolling (Day-N)** | % of cohort active on day N or later | Subscription, SaaS |
| **Bounded (Window)** | % of cohort active within a window (e.g., Day 7-13) | Weekly-use products |
| **Unbounded** | % of cohort active on day N or any day after | Long-term value assessment |

### Key Retention Benchmarks

| Time Period | Good (Social) | Good (SaaS) | Good (E-commerce) |
|-------------|--------------|-------------|-------------------|
| Day 1 | 40-60% | 70-80% | 20-30% |
| Day 7 | 25-35% | 50-60% | 10-15% |
| Day 30 | 15-25% | 40-50% | 5-10% |
| Day 90 | 10-15% | 35-45% | 3-7% |

### Churn Rate

| Churn Type | Formula | Context |
|-----------|---------|---------|
| User Churn Rate | Users lost in period / Users at start of period | Subscription services |
| Revenue Churn (Gross) | MRR lost from churned customers / MRR at start | SaaS businesses |
| Revenue Churn (Net) | (MRR lost - MRR expansion) / MRR at start | Growth-stage SaaS |
| Logo Churn | Accounts lost / Accounts at start | B2B products |

> **Critical Insight:** Net negative churn (expansion revenue exceeds churn revenue) is the holy grail of SaaS. It means your revenue grows even without acquiring new customers. Always distinguish between gross and net churn in interviews.

### Survival Analysis Perspective

Retention curves typically follow a pattern:
1. **Steep early drop** (Day 0-3): Casual/accidental users leave
2. **Inflection point** (Day 7-14): "Aha moment" separates retained from churned
3. **Flattening** (Day 30+): Core users stabilize — this is your "retention floor"

---

## 3. Revenue & Monetization Metrics

### Core Revenue Metrics

| Metric | Formula | Interpretation |
|--------|---------|---------------|
| **ARPU** | Total Revenue / Total Users | Average revenue per user (all users) |
| **ARPPU** | Total Revenue / Paying Users | Average revenue per paying user |
| **LTV** (Lifetime Value) | ARPU x Average Lifetime | Total value of a customer relationship |
| **CAC** (Customer Acquisition Cost) | Total Acquisition Spend / New Customers Acquired | Cost to acquire one customer |
| **LTV:CAC Ratio** | LTV / CAC | Unit economics health (target: 3:1+) |
| **Payback Period** | CAC / Monthly ARPU | Months to recover acquisition cost |

### LTV Calculation Methods

| Method | Formula | When to Use |
|--------|---------|-------------|
| **Simple** | ARPU x (1 / Churn Rate) | Stable, mature products |
| **Cohort-based** | Sum of discounted future revenue per cohort | Products with changing behavior |
| **Predictive (BG/NBD)** | Probabilistic model of purchase frequency | E-commerce, variable purchase patterns |
| **Margin-adjusted** | LTV x Gross Margin % | When comparing across business lines |

> **Critical Insight:** LTV:CAC > 3 is considered healthy for most businesses. LTV:CAC < 1 means you're losing money on every customer acquired. In interviews, always mention the payback period alongside LTV:CAC — a 3:1 ratio with a 36-month payback is very different from a 3:1 ratio with a 6-month payback.

### Monetization Funnel Metrics

| Stage | Key Metrics |
|-------|------------|
| Awareness | Impressions, Reach, CPM |
| Conversion | CTR, Conversion Rate, CPC, CPA |
| Transaction | AOV, Items per Order, Purchase Frequency |
| Expansion | Upsell Rate, Cross-sell Rate, Basket Size Growth |

---

## 4. Funnel Analysis

### Standard Conversion Funnel

| Stage | Example (E-commerce) | Example (SaaS) | Example (Social) |
|-------|---------------------|----------------|-----------------|
| Awareness | Site Visit | Landing Page View | App Download |
| Interest | Product View | Feature Page View | Account Creation |
| Consideration | Add to Cart | Free Trial Start | Profile Setup |
| Intent | Checkout Start | Plan Selection | First Post |
| Purchase | Order Complete | Subscription Start | Daily Active |
| Loyalty | Repeat Purchase | Annual Renewal | Power User |

### Funnel Metrics

| Metric | Formula | Significance |
|--------|---------|-------------|
| Step Conversion Rate | Users at step N+1 / Users at step N | Identifies biggest drop-offs |
| Overall Conversion Rate | Users at final step / Users at first step | End-to-end efficiency |
| Drop-off Rate | 1 - Step Conversion Rate | Friction identification |
| Time to Convert | Median time between first and last step | Speed of decision |
| Funnel Velocity | Conversions per unit time | Throughput of the funnel |

> **Critical Insight:** When investigating funnel drops, always segment by platform (iOS/Android/Web), user cohort (new/returning), and geography before concluding there is a universal problem. A 30% drop at checkout might be 5% for returning users and 60% for new users — requiring very different solutions.

---

## 5. Cohort Analysis

### Types of Cohort Analysis

| Cohort Type | Grouping Basis | Example Question |
|-------------|---------------|-----------------|
| **Acquisition Cohort** | Sign-up date | "Do Jan users retain better than Feb users?" |
| **Behavioral Cohort** | Action taken | "Do users who complete onboarding retain better?" |
| **Spend Cohort** | Revenue tier | "Do high-spenders in month 1 maintain spending?" |
| **Feature Cohort** | Feature adoption | "Do users who adopt Stories engage more?" |
| **Channel Cohort** | Acquisition source | "Do organic users have better LTV than paid?" |

### Reading a Cohort Retention Table

```
         Month 0  Month 1  Month 2  Month 3  Month 4  Month 5
Jan '25   100%     45%      35%      30%      28%      27%
Feb '25   100%     48%      38%      33%      30%       -
Mar '25   100%     50%      40%      35%       -        -
Apr '25   100%     52%      42%       -        -        -
May '25   100%     55%       -        -        -        -
```

> **Critical Insight:** Look for two patterns in cohort tables: (1) **Vertical improvement** — are newer cohorts retaining better? This suggests product improvements are working. (2) **Horizontal flattening** — at what month does retention stabilize? This is your natural "retention floor" and indicates product-market fit strength.

---

## 6. Growth Accounting Framework

### The Growth Equation

```
MAU(t) = MAU(t-1) + New + Resurrected - Churned

Where:
- New: First-time users in period t
- Retained: Users active in both t-1 and t (not in the equation, but measured)
- Resurrected: Users active in t but NOT in t-1, who were active before t-1
- Churned: Users active in t-1 but NOT in t
```

### Growth Accounting Metrics

| Metric | Formula | Healthy Range |
|--------|---------|--------------|
| Quick Ratio | (New + Resurrected) / Churned | > 4 (high growth), > 2 (healthy), < 1 (declining) |
| Growth Rate | (New + Resurrected - Churned) / MAU(t-1) | Depends on stage |
| Retention Rate | Retained / MAU(t-1) | > 80% for mature products |
| Resurrection Rate | Resurrected / Previously Churned | Higher = strong re-engagement |
| Net Growth | New + Resurrected - Churned | Positive = growing |

> **Critical Insight:** The Quick Ratio is one of the most revealing growth health metrics. A Quick Ratio of 4 means for every user lost, 4 are gained (through new acquisition or resurrection). Companies with QR < 1 are shrinking regardless of how many new users they acquire.

### Growth Accounting Segments

| Segment | Definition | Strategic Implication |
|---------|-----------|---------------------|
| New | First activity ever | Acquisition effectiveness |
| Current (Retained) | Active this period AND last period | Core product strength |
| Resurrected | Active now, inactive last period, active before | Re-engagement effectiveness |
| Churned (from Retained) | Was retained last period, inactive now | Product/value issues |
| Churned (from New) | Was new last period, inactive now | Onboarding/activation issues |

---

## 7. Metric Decomposition & Trees

### The Art of Metric Decomposition

Metric trees break a high-level metric into component parts to:
1. Identify root causes of changes
2. Prioritize improvement opportunities
3. Communicate clearly in investigations

### Example: Revenue Decomposition

```
Revenue
+-- Users x ARPU
|   +-- Users = New Users + Returning Users
|   |   +-- New Users = Traffic x Signup Rate
|   |   +-- Returning Users = Previous Users x Retention Rate
|   +-- ARPU = Transactions per User x AOV
|       +-- Transactions per User = Sessions x Conversion Rate
|       +-- AOV = Items per Order x Avg Item Price
```

### Example: Engagement Decomposition (Social Media)

```
Total Time Spent (platform)
+-- DAU x Time per DAU
|   +-- DAU = MAU x (DAU/MAU stickiness)
|   |   +-- MAU = New + Retained + Resurrected
|   |   +-- Stickiness = f(notifications, content quality, habits)
|   +-- Time per DAU = Sessions per DAU x Time per Session
|       +-- Sessions per DAU = f(push notifications, habits)
|       +-- Time per Session = f(content depth, feed algorithm)
```

### Example: Marketplace Decomposition (Uber/Airbnb)

```
Gross Bookings
+-- Trips x Average Fare
|   +-- Trips = Riders x Trips per Rider
|   |   +-- Riders = Active Riders x Request Rate
|   |   +-- Trips per Rider = f(price, availability, wait time)
|   +-- Average Fare = f(distance, surge, trip type)
|       +-- Distance = f(geography, use case mix)
|       +-- Surge = f(supply-demand balance)
```

> **Critical Insight:** In interviews, always decompose MULTIPLICATIVELY (A = B x C) rather than additively where possible. Multiplicative decomposition helps you identify whether a revenue drop is driven by fewer users OR lower spend per user, which leads to very different diagnoses and solutions.

---

## 8. North Star Metrics by Company

| Company | North Star Metric | Why This Metric |
|---------|------------------|-----------------|
| **Meta (Facebook)** | DAU and Time Spent | Measures daily engagement habit; ad revenue follows |
| **Google (Search)** | Searches per User per Day | Reflects search quality and habit strength |
| **Netflix** | Hours Streamed per Subscriber | Correlates with retention and content satisfaction |
| **Spotify** | Time Spent Listening | Indicates content discovery and engagement depth |
| **Uber** | Completed Trips per Week | Represents core marketplace transaction |
| **Airbnb** | Nights Booked | Directly tied to revenue and host/guest match quality |
| **Amazon (Retail)** | Purchase Frequency | Reflects convenience, selection, and trust |
| **Slack** | Messages Sent per Org per Day | Indicates team adoption and stickiness |
| **Pinterest** | Weekly Active Pinners (saves) | Represents intent-driven engagement |
| **LinkedIn** | Monthly Active Sessions | Professional network value realization |
| **Twitter/X** | Daily Tweet Impressions | Content consumption and creation ecosystem health |
| **YouTube** | Watch Time | Content satisfaction driving ad revenue |

### Supporting Metrics Framework

For each North Star, companies track supporting metrics across dimensions:

| Dimension | Example Supporting Metrics |
|-----------|--------------------------|
| **Breadth** | MAU, market penetration, new user activation |
| **Depth** | Sessions per user, features used, time spent |
| **Frequency** | DAU/MAU, return frequency, purchase cadence |
| **Efficiency** | Time to value, task completion rate, error rate |
| **Monetization** | ARPU, conversion rate, LTV |

> **Critical Insight:** A good North Star metric satisfies three criteria: (1) it correlates with long-term business value, (2) it is actionable by the product team, and (3) it is measurable and interpretable. "Revenue" fails criterion 2 for most product teams. "DAU" might fail criterion 1 if growth is driven by low-quality users.

---

## 9. A/B Test Metric Selection

### Metric Hierarchy for Experiments

| Metric Type | Purpose | Example (Feed Algorithm Change) |
|-------------|---------|-------------------------------|
| **Primary (OEC)** | Single metric to make ship/no-ship decision | Time Spent in Feed |
| **Secondary** | Metrics expected to move in same direction | Likes, Comments, Shares |
| **Guardrail** | Metrics that must NOT degrade | App crashes, Load time, Revenue |
| **Counter** | Metrics expected to move opposite to primary | Time in other surfaces |
| **Diagnostic** | Help explain WHY primary moved | Scroll depth, Content type mix |

### Choosing the Primary Metric (OEC)

| Criterion | Good OEC | Bad OEC |
|-----------|----------|---------|
| Sensitivity | Moves detectably with real changes | Requires millions of users to detect |
| Directionality | Higher is unambiguously better | Can be "gamed" (e.g., clickbait boosts CTR) |
| Timeliness | Observable within experiment duration | Takes months to manifest |
| Alignment | Correlated with long-term business value | Short-term proxy with known divergence |

### Guardrail Metric Categories

| Category | Example Guardrails |
|----------|-------------------|
| **Performance** | Page load time, App crash rate, Latency p99 |
| **Revenue** | ARPU, Ad revenue per session, Subscription cancellation rate |
| **Quality** | Report rate, Spam rate, Content quality score |
| **Ecosystem** | Creator posting rate, Supply growth, Partner satisfaction |
| **Trust & Safety** | Misinformation flags, Harassment reports |

> **Critical Insight:** The most common interview mistake is choosing a primary metric that can be gamed. For example, "clicks" can be boosted by clickbait. "Time spent" can increase due to confusion. Always pair your primary metric with a quality guardrail. In Meta interviews especially, mention the tradeoff between engagement and well-being metrics.

---

## 10. Interview Scenarios & Frameworks

### Framework: METRIC (for "Define metrics for X")

| Step | Action | Example |
|------|--------|---------|
| **M** - Mission | State the product's core mission | "Instagram Stories helps users share ephemeral moments" |
| **E** - Ecosystem | Identify all stakeholders | Creators, viewers, advertisers |
| **T** - Top-line | Define the North Star metric | Stories views per DAU |
| **R** - Related | Add secondary/supporting metrics | Creation rate, completion rate, reply rate |
| **I** - Investigate | Define diagnostic metrics | Drop-off by story position, time between stories |
| **C** - Counter/Guardrail | Add safety metrics | Feed time spent (guardrail), creator burnout signals |

### Scenario 1: "How would you measure success of Instagram Stories?"

**Structured Answer:**

1. **Mission:** Enable ephemeral sharing and casual communication
2. **Stakeholders:** Creators, Viewers, Advertisers
3. **Primary Metrics:**
   - Stories DAU / Instagram DAU (adoption)
   - Stories created per creator per day (supply)
   - Stories viewed per viewer per day (demand)
4. **Engagement Metrics:**
   - Completion rate (% who watch full story)
   - Reply rate (% stories that receive a reply)
   - Tap-forward vs. tap-back ratio (interest signal)
5. **Growth Metrics:**
   - New Stories creators per week
   - D7 retention of new Stories users
6. **Guardrails:**
   - Feed engagement (should not cannibalize)
   - Creator posting frequency in main feed (should not decrease)
   - Time to first story creation for new users

### Scenario 2: "Revenue dropped 10% — investigate."

**Structured Investigation Framework (DECOMPOSE):**

1. **Define scope:** When did the drop start? Is it sudden or gradual?
2. **External factors:** Seasonality? Holiday? Competitor launch? Outage?
3. **Decompose the metric:**
   ```
   Revenue = Users x Transactions/User x Revenue/Transaction
   ```
   Which component dropped?
4. **Segment:** Cut by geography, platform, user cohort, product line
5. **Correlate:** What other metrics moved at the same time?
6. **Root cause:** Form hypotheses and validate with data

**Investigation Decision Tree:**

| Finding | Next Step |
|---------|-----------|
| Users dropped | Check acquisition channels, app store changes, outages |
| Transactions/User dropped | Check product changes, pricing, inventory |
| Revenue/Transaction dropped | Check discount campaigns, product mix, currency |
| Isolated to one segment | Deep-dive that segment specifically |
| Gradual decline | Check retention trends, competitive analysis |
| Sudden drop | Check deployments, bugs, external events |

### Scenario 3: "How would you define metrics for a new feature?"

**Framework (PRE-LAUNCH):**

| Phase | Question | Metrics |
|-------|----------|---------|
| **Pre-launch** | What does success look like? | Target adoption %, target impact on North Star |
| **Launch** | Are users finding/using it? | Discovery rate, activation rate, error rate |
| **Early adoption** | Are users getting value? | Task completion, time-to-value, D1/D7 retention |
| **Growth** | Is it growing organically? | Viral coefficient, organic vs prompted usage |
| **Maturity** | Is it sustainable? | Long-term retention, impact on overall engagement |

---

## 11. Python Implementations

### DAU / WAU / MAU Calculation

```python
import pandas as pd
import numpy as np
from datetime import datetime, timedelta

def calculate_active_users(events_df, date_col='event_date', user_col='user_id'):
    """
    Calculate DAU, WAU, MAU for each date.
    """
    events_df[date_col] = pd.to_datetime(events_df[date_col])
    dates = sorted(events_df[date_col].unique())
    
    results = []
    for date in dates:
        # DAU: unique users on this date
        dau = events_df[events_df[date_col] == date][user_col].nunique()
        
        # WAU: unique users in the 7-day window ending on this date
        week_start = date - timedelta(days=6)
        wau_mask = (events_df[date_col] >= week_start) & (events_df[date_col] <= date)
        wau = events_df[wau_mask][user_col].nunique()
        
        # MAU: unique users in the 30-day window ending on this date
        month_start = date - timedelta(days=29)
        mau_mask = (events_df[date_col] >= month_start) & (events_df[date_col] <= date)
        mau = events_df[mau_mask][user_col].nunique()
        
        results.append({
            'date': date, 'dau': dau, 'wau': wau, 'mau': mau,
            'stickiness_dau_mau': round(dau / mau, 4) if mau > 0 else 0
        })
    
    return pd.DataFrame(results)
```

### Retention Calculation (Day-N)

```python
def calculate_day_n_retention(events_df, user_col='user_id', date_col='event_date',
                               retention_days=[1, 3, 7, 14, 30]):
    """
    Calculate Day-N retention by acquisition cohort (weekly cohorts).
    """
    events_df[date_col] = pd.to_datetime(events_df[date_col])
    
    # Determine each user's first activity date
    first_activity = events_df.groupby(user_col)[date_col].min().reset_index()
    first_activity.columns = [user_col, 'cohort_date']
    
    merged = events_df.merge(first_activity, on=user_col)
    merged['days_since_signup'] = (merged[date_col] - merged['cohort_date']).dt.days
    merged['cohort_week'] = merged['cohort_date'].dt.to_period('W')
    
    results = []
    for cohort, group in merged.groupby('cohort_week'):
        cohort_users = group[group['days_since_signup'] == 0][user_col].nunique()
        row = {'cohort': str(cohort), 'cohort_size': cohort_users}
        for day_n in retention_days:
            retained = group[group['days_since_signup'] == day_n][user_col].nunique()
            row[f'day_{day_n}_retention'] = round(retained / cohort_users, 4) if cohort_users > 0 else 0
        results.append(row)
    
    return pd.DataFrame(results)
```

### Churn Rate Calculation

```python
def calculate_churn_rate(events_df, user_col='user_id', date_col='event_date', period='M'):
    """
    Calculate monthly churn rate: users active last period but not this period.
    """
    events_df[date_col] = pd.to_datetime(events_df[date_col])
    events_df['period'] = events_df[date_col].dt.to_period(period)
    periods = sorted(events_df['period'].unique())
    
    results = []
    for i in range(1, len(periods)):
        prev_users = set(events_df[events_df['period'] == periods[i-1]][user_col])
        curr_users = set(events_df[events_df['period'] == periods[i]][user_col])
        
        churned = prev_users - curr_users
        retained = prev_users & curr_users
        churn_rate = len(churned) / len(prev_users) if prev_users else 0
        
        results.append({
            'period': str(periods[i]),
            'previous_users': len(prev_users),
            'retained': len(retained),
            'churned': len(churned),
            'churn_rate': round(churn_rate, 4),
            'retention_rate': round(1 - churn_rate, 4)
        })
    
    return pd.DataFrame(results)
```

### Revenue Metrics (ARPU, LTV, CAC)

```python
def calculate_revenue_metrics(transactions_df, users_df,
                                user_col='user_id', revenue_col='amount',
                                date_col='transaction_date',
                                acquisition_cost_col='acquisition_cost'):
    """
    Calculate ARPU, ARPPU, estimated LTV, and LTV:CAC ratio.
    """
    total_users = users_df[user_col].nunique()
    paying_users = transactions_df[user_col].nunique()
    total_revenue = transactions_df[revenue_col].sum()
    
    arpu = total_revenue / total_users if total_users > 0 else 0
    arppu = total_revenue / paying_users if paying_users > 0 else 0
    
    # Monthly ARPU for LTV
    transactions_df[date_col] = pd.to_datetime(transactions_df[date_col])
    transactions_df['month'] = transactions_df[date_col].dt.to_period('M')
    monthly_revenue = transactions_df.groupby('month')[revenue_col].sum()
    monthly_arpu = monthly_revenue.mean() / total_users
    
    # Estimate churn for LTV
    monthly_users = transactions_df.groupby('month')[user_col].nunique()
    if len(monthly_users) > 1:
        churn_rates = []
        for i in range(1, len(monthly_users)):
            prev, curr = monthly_users.iloc[i-1], monthly_users.iloc[i]
            churn_rates.append(max(0, (prev - curr) / prev))
        avg_churn = np.mean(churn_rates) if churn_rates else 0.05
    else:
        avg_churn = 0.05

    ltv = monthly_arpu / avg_churn if avg_churn > 0 else monthly_arpu * 60
    cac = users_df[acquisition_cost_col].sum() / total_users if total_users > 0 else 0
    ltv_cac_ratio = ltv / cac if cac > 0 else float('inf')
    
    return {
        'arpu': round(arpu, 2), 'arppu': round(arppu, 2),
        'estimated_ltv': round(ltv, 2), 'cac': round(cac, 2),
        'ltv_cac_ratio': round(ltv_cac_ratio, 2),
        'payback_months': round(cac / monthly_arpu, 1) if monthly_arpu > 0 else None
    }
```

### Funnel Analysis

```python
def funnel_analysis(events_df, funnel_steps, user_col='user_id', event_col='event_name'):
    """
    Calculate conversion rates through a defined funnel.
    """
    results = []
    total_top = events_df[events_df[event_col] == funnel_steps[0]][user_col].nunique()
    prev_users = None
    
    for i, step in enumerate(funnel_steps):
        current_users = events_df[events_df[event_col] == step][user_col].nunique()
        step_conv = current_users / prev_users if prev_users and prev_users > 0 else 1.0
        overall_conv = current_users / total_top if total_top > 0 else 0
        
        results.append({
            'step': i + 1, 'event': step, 'users': current_users,
            'step_conversion': round(step_conv, 4),
            'overall_conversion': round(overall_conv, 4),
            'dropoff_rate': round(1 - step_conv, 4) if i > 0 else 0
        })
        prev_users = current_users
    
    return pd.DataFrame(results)
```

### Growth Accounting

```python
def growth_accounting(events_df, user_col='user_id', date_col='event_date', period='M'):
    """
    Classify users as New, Retained, Resurrected, or Churned each period.
    """
    events_df[date_col] = pd.to_datetime(events_df[date_col])
    events_df['period'] = events_df[date_col].dt.to_period(period)
    
    first_seen = events_df.groupby(user_col)[date_col].min().reset_index()
    first_seen.columns = [user_col, 'first_date']
    first_seen['first_period'] = first_seen['first_date'].dt.to_period(period)
    
    periods = sorted(events_df['period'].unique())
    results = []
    prev_period_users = set()
    
    for p in periods:
        current_users = set(events_df[events_df['period'] == p][user_col])
        new_this_period = set(first_seen[first_seen['first_period'] == p][user_col])
        
        new = current_users & new_this_period
        retained = current_users & prev_period_users
        resurrected = current_users - new - retained
        churned = prev_period_users - current_users
        
        quick_ratio = ((len(new) + len(resurrected)) / len(churned)
                       if len(churned) > 0 else float('inf'))
        
        results.append({
            'period': str(p), 'total_active': len(current_users),
            'new': len(new), 'retained': len(retained),
            'resurrected': len(resurrected), 'churned': len(churned),
            'quick_ratio': round(quick_ratio, 2)
        })
        prev_period_users = current_users
    
    return pd.DataFrame(results)
```

### Metric Decomposition (Revenue Tree)

```python
def decompose_revenue(transactions_df, user_col='user_id',
                       revenue_col='amount', date_col='transaction_date',
                       compare_periods=('2025-01', '2025-02')):
    """
    Decompose revenue change: Revenue = Users x Txn/User x Rev/Txn
    Uses log-based attribution for multiplicative decomposition.
    """
    transactions_df[date_col] = pd.to_datetime(transactions_df[date_col])
    transactions_df['period'] = transactions_df[date_col].dt.to_period('M').astype(str)
    
    metrics = {}
    for period in compare_periods:
        data = transactions_df[transactions_df['period'] == period]
        users = data[user_col].nunique()
        txns = len(data)
        revenue = data[revenue_col].sum()
        metrics[period] = {
            'revenue': revenue, 'users': users,
            'txn_per_user': txns / users if users else 0,
            'rev_per_txn': revenue / txns if txns else 0
        }
    
    p1, p2 = compare_periods
    # Log-based multiplicative attribution
    if all(metrics[p1][k] > 0 for k in ['users', 'txn_per_user', 'rev_per_txn']):
        user_contrib = np.log(metrics[p2]['users'] / metrics[p1]['users'])
        freq_contrib = np.log(metrics[p2]['txn_per_user'] / metrics[p1]['txn_per_user'])
        val_contrib = np.log(metrics[p2]['rev_per_txn'] / metrics[p1]['rev_per_txn'])
        total = abs(user_contrib) + abs(freq_contrib) + abs(val_contrib)
        attribution = {
            'user_volume_pct': round(user_contrib / total * 100, 1) if total else 0,
            'frequency_pct': round(freq_contrib / total * 100, 1) if total else 0,
            'value_pct': round(val_contrib / total * 100, 1) if total else 0
        }
    else:
        attribution = {'user_volume_pct': 0, 'frequency_pct': 0, 'value_pct': 0}
    
    return {'metrics': metrics, 'attribution': attribution}
```

### A/B Test Metric Evaluation

```python
from scipy import stats

def evaluate_ab_metrics(control_df, treatment_df,
                         primary_metric, secondary_metrics, guardrail_metrics,
                         user_col='user_id', alpha=0.05):
    """
    Evaluate A/B test results across primary, secondary, and guardrail metrics.
    Returns ship/no-ship recommendation.
    """
    def test_metric(ctrl_vals, treat_vals, name, mtype):
        t_stat, p_val = stats.ttest_ind(ctrl_vals, treat_vals)
        ctrl_mean, treat_mean = ctrl_vals.mean(), treat_vals.mean()
        lift = (treat_mean - ctrl_mean) / ctrl_mean * 100 if ctrl_mean else 0
        pooled_std = np.sqrt((ctrl_vals.std()**2 + treat_vals.std()**2) / 2)
        cohens_d = (treat_mean - ctrl_mean) / pooled_std if pooled_std else 0
        
        is_degraded = None
        if mtype == 'guardrail':
            _, p_one = stats.ttest_ind(ctrl_vals, treat_vals, alternative='greater')
            is_degraded = p_one < alpha
        
        return {
            'metric': name, 'type': mtype,
            'control_mean': round(ctrl_mean, 4),
            'treatment_mean': round(treat_mean, 4),
            'lift_pct': round(lift, 2),
            'p_value': round(p_val, 4),
            'significant': p_val < alpha,
            'cohens_d': round(cohens_d, 3),
            'is_degraded': is_degraded
        }
    
    results = {'primary': test_metric(
        control_df[primary_metric], treatment_df[primary_metric],
        primary_metric, 'primary'
    )}
    results['secondary'] = [test_metric(control_df[m], treatment_df[m], m, 'secondary')
                            for m in secondary_metrics]
    results['guardrails'] = [test_metric(control_df[m], treatment_df[m], m, 'guardrail')
                             for m in guardrail_metrics]
    
    primary_wins = results['primary']['significant'] and results['primary']['lift_pct'] > 0
    guardrails_safe = all(not g['is_degraded'] for g in results['guardrails'])
    results['ship'] = primary_wins and guardrails_safe
    
    return results
```

---

## 12. Interview Talking Points & Scripts

### On Defining Metrics for a Product

> *"When defining metrics for a product, I follow a structured approach. First, I clarify the product's core mission and who the key stakeholders are. Then I identify the North Star metric — the single metric that best captures whether we are delivering value. From there, I build a metrics hierarchy: primary metrics for decision-making, secondary metrics to understand the 'why', and guardrail metrics to ensure we are not causing unintended harm. For example, for a messaging product, the North Star might be messages sent per DAU, with read receipts as a secondary metric and app crash rate as a guardrail."*

### On Investigating a Metric Drop

> *"My approach to investigating a metric drop is systematic. First, I establish the timeline — when exactly did the drop start, and is it sudden or gradual? Then I check for external factors: seasonality, outages, competitor events. Next, I decompose the metric into its multiplicative components. For example, if revenue dropped, I check whether it is driven by fewer users, fewer transactions per user, or lower transaction value. I then segment across key dimensions — platform, geography, user cohort, product line — to isolate the affected population. Finally, I look for correlated metric movements to form and validate hypotheses about root cause."*

### On A/B Test Metric Selection

> *"Choosing the right metrics for an A/B test is crucial because it determines what we optimize for and what guardrails we set. I start with the primary metric — this should be sensitive enough to detect real changes within our experiment timeline, directionally clear, and aligned with long-term business value. I then identify guardrail metrics — things we absolutely cannot degrade. Counter metrics help us understand tradeoffs. Finally, I include diagnostic metrics that help explain why we saw the result we did."*

### On Growth vs. Retention Tradeoffs

> *"One of the most important frameworks I use is growth accounting. The Quick Ratio tells you whether growth is sustainable. A company can grow DAU by 10% while having terrible retention if they are spending heavily on acquisition. I always look at both acquisition and retention. Improving Day-1 to Day-7 retention by even 2-3 percentage points often has a larger long-term impact than doubling acquisition spend, because retained users compound over time and drive organic growth through referrals."*

### On Metric Decomposition Under Pressure

> *"When presented with a vague 'revenue dropped' question, I resist the urge to jump to hypotheses. Instead, I build a metric tree. Revenue equals users times frequency times value. I ask: which branch moved? This immediately cuts the problem space. If users are constant but frequency dropped, I am looking at a completely different set of causes than if users dropped. This multiplicative decomposition is the single most powerful tool for metric investigation."*

---

## 13. Common Interview Mistakes

| Mistake | Better Approach |
|---------|----------------|
| Jumping to a single metric without considering tradeoffs | Present a metrics hierarchy: primary, secondary, guardrail, and counter metrics |
| Defining "active" without specifying the qualifying action | Always state what counts as "active" — e.g., "opened the app" vs "completed a core action" |
| Confusing correlation with causation when investigating drops | Use temporal sequencing, segment analysis, and A/B test evidence to establish causality |
| Forgetting to mention guardrail metrics in A/B test design | Always include performance, revenue, and safety guardrails alongside the primary metric |
| Using total counts instead of per-user metrics in experiments | Randomize at user level and analyze per-user averages to avoid Simpson's Paradox |
| Not segmenting when investigating a metric change | Always cut by platform, geography, user type, and time before concluding root cause |
| Proposing vanity metrics (page views, downloads) as success metrics | Focus on metrics that reflect actual value delivery — retention, task completion, repeat usage |
| Ignoring seasonality and external factors in trend analysis | Establish baselines, compare year-over-year, and note external events before attributing changes |
| Suggesting LTV without mentioning time horizon or discount rate | Specify the prediction window, discount rate, and whether margin-adjusted |
| Defining North Star without explaining why it drives long-term value | Articulate the causal chain from your metric to business outcomes |
| Treating all users as homogeneous | Segment by lifecycle stage (new/active/power/at-risk) and acknowledge different behaviors |
| Proposing "time spent" as always positive | Acknowledge that increased time might indicate confusion — pair with satisfaction signals |

---

## 14. Rapid-Fire Q&A

**Q1: What is the difference between ARPU and ARPPU?**
ARPU divides revenue by ALL users (including non-paying); ARPPU divides only by paying users. ARPU = ARPPU x Paying User Percentage.

**Q2: DAU/MAU is 0.6 — is that good?**
A DAU/MAU of 0.6 (60%) is excellent for social/messaging apps (Facebook is ~66%). Compare against category benchmarks, not absolute standards.

**Q3: How do you choose between Day-N retention and rolling retention?**
Use Day-N for daily-use products (social, gaming). Use rolling for products with natural gaps (e-commerce, travel). Rolling gives higher numbers and is better for weekly/monthly products.

**Q4: When should you use cohort analysis vs. aggregate metrics?**
Use cohorts when you suspect changes over time are affecting different user groups differently. Aggregate metrics can mask Simpson's Paradox. Always use cohorts for retention analysis.

**Q5: What is the Quick Ratio and what is a good benchmark?**
Quick Ratio = (New + Resurrected) / Churned. QR > 4: high growth. QR 2-4: healthy. QR < 1: shrinking regardless of acquisition.

**Q6: How do you handle metrics that conflict in an A/B test?**
Verify conflicts are real (not noise). If primary is positive and guardrails are safe, a secondary decline may be an acceptable tradeoff. Document and escalate for product judgment.

**Q7: What is the difference between leading and lagging indicators?**
Leading indicators predict future outcomes (onboarding completion predicts D30 retention). Lagging indicators confirm past performance (monthly revenue). Use leading indicators as primary metrics in experiments for faster signal.

**Q8: How would you detect if a metric improvement is just a novelty effect?**
Run the experiment 3-4 weeks minimum. Plot treatment effect over time — decay toward zero means novelty. Compare early vs late enrollees. Use holdback groups post-launch.

**Q9: What is metric decomposition and when do you use it?**
Breaking a high-level metric into multiplicative components (Revenue = Users x Frequency x Value). Use when a metric changed and you need the driver, when prioritizing levers, or when setting team OKRs.

**Q10: How do you handle the cold-start problem in metrics for new features?**
Define target metrics before launch. Use proxy metrics from similar features. Set minimum detectable effect based on business impact threshold. Allow longer experiment durations. Supplement with qualitative signals.

---

## 15. Summary Cheat Sheet

```
+------------------------------------------------------------------+
|           PRODUCT METRICS & ANALYTICS CHEAT SHEET                 |
+------------------------------------------------------------------+
|                                                                    |
|  ENGAGEMENT:                                                       |
|    DAU/WAU/MAU = Unique active users per time window               |
|    Stickiness  = DAU/MAU (benchmark: social 50%+, SaaS 20-30%)    |
|    L7/L28      = Days active in 7/28-day window (distribution!)    |
|                                                                    |
|  RETENTION:                                                        |
|    Day-N       = % cohort active on exactly day N                  |
|    Churn Rate  = Users lost / Users at start of period             |
|    Look for: steep drop -> inflection -> floor pattern             |
|                                                                    |
|  REVENUE:                                                          |
|    ARPU = Revenue / All Users                                      |
|    LTV  = ARPU / Churn (simple) or cohort-based sum                |
|    CAC  = Acquisition Spend / New Customers                        |
|    Target: LTV:CAC > 3:1, Payback < 12 months                     |
|                                                                    |
|  GROWTH ACCOUNTING:                                                |
|    MAU(t) = MAU(t-1) + New + Resurrected - Churned                 |
|    Quick Ratio = (New + Resurrected) / Churned                     |
|    QR > 4 = high growth | QR < 1 = shrinking                      |
|                                                                    |
|  METRIC TREES (decompose multiplicatively):                        |
|    Revenue = Users x Txn/User x Rev/Txn                            |
|    Engagement = DAU x Sessions/DAU x Time/Session                  |
|                                                                    |
|  A/B TEST METRICS:                                                 |
|    Primary   -> Ship/no-ship decision (sensitive + aligned)        |
|    Secondary -> Expected to co-move (explains mechanism)           |
|    Guardrail -> Must NOT degrade (revenue, perf, safety)           |
|    Counter   -> Expected opposite movement (tradeoff tracking)     |
|                                                                    |
|  INVESTIGATION FRAMEWORK:                                          |
|    1. Timeline (when/how sudden?)                                  |
|    2. External factors (season/outage/competitor?)                  |
|    3. Decompose (which branch of metric tree?)                     |
|    4. Segment (platform/geo/cohort/product?)                       |
|    5. Correlate (what else moved?)                                  |
|    6. Root cause + validate                                        |
|                                                                    |
|  NORTH STARS:                                                      |
|    Meta -> DAU + Time Spent | Google -> Searches/User/Day          |
|    Netflix -> Hours Streamed | Spotify -> Time Listening           |
|    Uber -> Completed Trips  | Amazon -> Purchase Frequency         |
|                                                                    |
|  INTERVIEW FRAMEWORK (METRIC):                                     |
|    M-ission -> E-cosystem -> T-op-line -> R-elated ->              |
|    I-nvestigate -> C-ounter/Guardrail                              |
|                                                                    |
+------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 22 of 25*
