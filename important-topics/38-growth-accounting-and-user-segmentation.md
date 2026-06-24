# 🎯 Topic 38: Growth Accounting and User Segmentation

> *"Growth without understanding its components is like driving with a blindfold. Growth accounting tells you not just whether you're growing, but the health and sustainability of that growth."*

---

## 📑 Table of Contents

1. [Growth Accounting Fundamentals](#growth-accounting-fundamentals)
2. [The Growth Equation](#the-growth-equation)
3. [Quick Ratio](#quick-ratio)
4. [User Segments and Lifecycle States](#user-segments-and-lifecycle-states)
5. [Behavioral Segmentation](#behavioral-segmentation)
6. [RFM Analysis](#rfm-analysis)
7. [Reactivation Analysis](#reactivation-analysis)
8. [SQL/Python for Growth Accounting](#sqlpython-for-growth-accounting)
9. [Interview Talking Points](#interview-talking-points)
10. [Common Mistakes](#common-mistakes)
11. [Rapid-Fire Q&A](#rapid-fire-qa)
12. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Growth Accounting Fundamentals

Growth accounting decomposes net user growth into its component flows. Instead of just saying "we grew by 10K MAU," it explains **where** that growth came from and how sustainable it is.

### Why It Matters

Two companies both growing at 10% monthly:
- Company A: 15% new, 5% resurrected, 80% retained, 10% churned (Quick Ratio: 2.0)
- Company B: 25% new, 3% resurrected, 72% retained, 20% churned (Quick Ratio: 1.4)

Company A has healthier growth — it retains more and churns less. Company B is running faster on a leaky treadmill.

> **Critical Insight:** Headline growth numbers are nearly meaningless without decomposition. A company growing 20% month-over-month by pouring money into acquisition while churning 25% is in serious trouble — the moment acquisition spending slows, growth goes negative.

---

## The Growth Equation

```
MAU(t) = MAU(t-1) + New + Resurrected - Churned

Equivalently:
Net Growth = New + Resurrected + Retained - Churned

Where:
- New:         First-time active users this period
- Resurrected: Were inactive (churned), returned this period
- Retained:    Were active last period AND this period
- Churned:     Were active last period, NOT active this period
```

### Visual Flow

```
                    +--------+
         New ------>|        |
                    |  MAU   |
  Resurrected ----->| (this  |-------> Churned (leave)
                    | month) |
    Retained ------>|        |-------> Retained (stay for next month)
                    +--------+
```

### Monthly Growth Accounting Example

```
Month: March 2024
=================
MAU (February):        100,000
  + New Users:          15,000
  + Resurrected Users:   3,000
  - Churned Users:      12,000
  = Retained Users:     88,000  (100,000 - 12,000)
=================
MAU (March):           106,000  (88,000 + 15,000 + 3,000)

Net Growth:             +6,000 (+6%)
```

### Period-Over-Period Decomposition

```
         Feb MAU     Mar MAU
         100,000     106,000
            |           ^
            |           |
            +--Retained (88K)--+
            |                  |
            +--Churned (12K)   +--New (15K)
                               +--Resurrected (3K)
```

---

## Quick Ratio

The Quick Ratio measures growth efficiency — how many users you add for every user you lose.

```
Quick Ratio = (New + Resurrected) / Churned
```

| Quick Ratio | Interpretation |
|-------------|---------------|
| < 1.0 | Shrinking — losing users faster than adding |
| 1.0 - 2.0 | Growing slowly, but inefficiently |
| 2.0 - 4.0 | Healthy growth |
| > 4.0 | Excellent growth efficiency |

### Nuances

```python
# Quick Ratio over time
def quick_ratio(new, resurrected, churned):
    return (new + resurrected) / churned if churned > 0 else float('inf')

# Example monthly tracking
months = {
    'Jan': quick_ratio(15000, 2000, 8000),   # 2.13
    'Feb': quick_ratio(14000, 2500, 9000),   # 1.83
    'Mar': quick_ratio(15000, 3000, 12000),  # 1.50  <-- Degrading!
}
```

> **Critical Insight:** A declining Quick Ratio is a leading indicator of growth stall. Even if absolute MAU is still increasing, a Quick Ratio trending toward 1.0 means you're approaching the ceiling where acquisition can no longer offset churn.

### Quick Ratio Variants

**Revenue Quick Ratio:**
```
Revenue QR = (New MRR + Expansion MRR + Reactivation MRR) / (Churned MRR + Contraction MRR)
```

**Engagement Quick Ratio:**
```
Engagement QR = (Users who increased activity) / (Users who decreased activity)
```

---

## User Segments and Lifecycle States

### Standard User Lifecycle Segments

| Segment | Definition | Typical Action |
|---------|-----------|---------------|
| **New** | First activity ever this period | Onboarding optimization |
| **Activated** | Completed key activation event | Guide to habit formation |
| **Power** | Active > X days/week (top 10-20%) | Referral, premium upsell |
| **Core** | Regular cadence matching product frequency | Maintain, deepen engagement |
| **Casual** | Active but below expected frequency | Nudge toward core behavior |
| **At-Risk** | Declining activity (active but trending down) | Re-engagement campaign |
| **Dormant** | Inactive 1-2 churn periods | Win-back campaign |
| **Churned** | Inactive > churn threshold | Resurrection if possible |
| **Resurrected** | Was churned, returned this period | Re-onboard carefully |

### State Transition Diagram

```
  [New] --activate--> [Activated] --habit--> [Core]
                                                |
                                         +------+------+
                                         |             |
                                    [Power]        [Casual]
                                         |             |
                                         +------+------+
                                                |
                                           [At-Risk]
                                                |
                                           [Dormant]
                                                |
                                           [Churned]
                                                |
                                         (may return)
                                                |
                                         [Resurrected]
```

### Defining Segments Quantitatively

```python
def classify_user(days_active_last_30, days_since_last_active, total_lifetime_days):
    if total_lifetime_days <= 7:
        return 'new'
    elif days_since_last_active > 60:
        return 'churned'
    elif days_since_last_active > 30:
        return 'dormant'
    elif days_active_last_30 >= 20:
        return 'power'
    elif days_active_last_30 >= 10:
        return 'core'
    elif days_active_last_30 >= 3:
        return 'casual'
    else:
        return 'at_risk'
```

---

## Behavioral Segmentation

### Beyond Frequency: Feature-Based Segments

Instead of just "how often," segment by "what they do."

```python
# Feature usage matrix
features_used = df.groupby('user_id').agg({
    'used_search': 'sum',
    'used_recommendations': 'sum',
    'used_social': 'sum',
    'used_creation_tool': 'sum',
    'used_messaging': 'sum'
})

# K-Means clustering on feature usage
from sklearn.cluster import KMeans
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()
features_scaled = scaler.fit_transform(features_used)

kmeans = KMeans(n_clusters=5, random_state=42)
features_used['segment'] = kmeans.fit_predict(features_scaled)

# Interpret clusters by examining centroids
centroids = pd.DataFrame(
    scaler.inverse_transform(kmeans.cluster_centers_),
    columns=features_used.columns[:-1]
)
```

### Segment Profiles Example

| Segment | Primary Behavior | Size | Retention | Revenue/User |
|---------|-----------------|------|-----------|--------------|
| Creators | Heavy content creation | 8% | 85% | $45/mo |
| Browsers | Consume but don't create | 35% | 55% | $12/mo |
| Social | Primarily messaging/sharing | 22% | 72% | $18/mo |
| Searchers | Goal-oriented, find-and-leave | 25% | 40% | $8/mo |
| Power Users | Use everything heavily | 10% | 92% | $65/mo |

> **Critical Insight:** Behavioral segments are more actionable than demographic segments for product decisions. Knowing that "Browsers who start creating retain at 2x the rate" directly suggests a product intervention (nudge browsers to create).

---

## RFM Analysis

RFM (Recency, Frequency, Monetary) is a classic customer segmentation framework.

### Computing RFM Scores

```python
import pandas as pd
from datetime import datetime

# Compute RFM metrics
rfm = df.groupby('customer_id').agg({
    'purchase_date': lambda x: (datetime.now() - x.max()).days,  # Recency
    'order_id': 'count',                                          # Frequency
    'revenue': 'sum'                                              # Monetary
}).rename(columns={
    'purchase_date': 'recency',
    'order_id': 'frequency', 
    'revenue': 'monetary'
})

# Score each dimension 1-5 (5 = best)
rfm['r_score'] = pd.qcut(rfm['recency'], 5, labels=[5,4,3,2,1])
rfm['f_score'] = pd.qcut(rfm['frequency'].rank(method='first'), 5, labels=[1,2,3,4,5])
rfm['m_score'] = pd.qcut(rfm['monetary'].rank(method='first'), 5, labels=[1,2,3,4,5])

# Combined RFM score
rfm['rfm_score'] = rfm['r_score'].astype(str) + rfm['f_score'].astype(str) + rfm['m_score'].astype(str)
```

### RFM Segment Mapping

| RFM Score Pattern | Segment Name | Strategy |
|------------------|--------------|----------|
| 555, 554, 544 | Champions | Reward, ask for referrals |
| 543, 444, 435 | Loyal | Upsell, cross-sell |
| 512, 511, 412 | Recent Customers | Onboard, nurture |
| 525, 524, 434 | Potential Loyalists | Engage more deeply |
| 155, 154, 144 | Can't Lose Them | Win-back immediately |
| 111, 112, 121 | Lost | May not be worth pursuing |
| 311, 411, 331 | About to Sleep | Re-engage now |

### Limitations of RFM

- Arbitrary bin boundaries (why quintiles?)
- Treats dimensions as independent (they're correlated)
- Doesn't capture behavioral richness (what they buy, how they browse)
- Static snapshot — doesn't capture trajectory

---

## Reactivation Analysis

### Understanding Resurrection

```
Resurrection Rate = Users who returned after being churned / Total churned users
```

### Factors Affecting Reactivation

```python
# Analyze what predicts successful reactivation
reactivated_users = df[df['was_churned'] & df['returned']]
stayed_churned = df[df['was_churned'] & ~df['returned']]

# Compare characteristics
comparison = pd.DataFrame({
    'Reactivated': reactivated_users[features].mean(),
    'Stayed Churned': stayed_churned[features].mean(),
    'Lift': reactivated_users[features].mean() / stayed_churned[features].mean()
})
```

### Time-to-Resurrection Distribution

```
Count of Reactivations by Months Since Churn:
  Month 1: ||||||||||||||||||||  (45%)
  Month 2: ||||||||||           (22%)
  Month 3: ||||||              (14%)
  Month 4: ||||               (9%)
  Month 5: |||                (6%)
  Month 6+: ||               (4%)

Insight: Most resurrections happen within 2 months.
         After 6 months, probability drops to near zero.
```

> **Critical Insight:** Resurrection attempts have sharply diminishing returns with time. Focus win-back campaigns in the first 30-60 days after churn. After 90 days, the probability of return drops below 5% for most products.

---

## SQL/Python for Growth Accounting

### Complete Growth Accounting Query (SQL)

```sql
WITH monthly_users AS (
    SELECT 
        user_id,
        DATE_TRUNC('month', event_timestamp) AS active_month
    FROM events
    GROUP BY user_id, DATE_TRUNC('month', event_timestamp)
),
user_first_month AS (
    SELECT user_id, MIN(active_month) AS first_month
    FROM monthly_users
    GROUP BY user_id
),
user_status AS (
    SELECT 
        curr.active_month,
        curr.user_id,
        CASE
            WHEN f.first_month = curr.active_month THEN 'new'
            WHEN prev.user_id IS NOT NULL THEN 'retained'
            ELSE 'resurrected'
        END AS user_type
    FROM monthly_users curr
    JOIN user_first_month f ON curr.user_id = f.user_id
    LEFT JOIN monthly_users prev 
        ON curr.user_id = prev.user_id 
        AND prev.active_month = curr.active_month - INTERVAL '1 month'
),
churned_users AS (
    SELECT 
        prev.active_month + INTERVAL '1 month' AS churn_month,
        prev.user_id
    FROM monthly_users prev
    LEFT JOIN monthly_users curr 
        ON prev.user_id = curr.user_id 
        AND curr.active_month = prev.active_month + INTERVAL '1 month'
    WHERE curr.user_id IS NULL
)
SELECT 
    s.active_month,
    COUNT(CASE WHEN s.user_type = 'new' THEN 1 END) AS new_users,
    COUNT(CASE WHEN s.user_type = 'retained' THEN 1 END) AS retained_users,
    COUNT(CASE WHEN s.user_type = 'resurrected' THEN 1 END) AS resurrected_users,
    COALESCE(c.churned_count, 0) AS churned_users,
    COUNT(*) AS total_mau,
    ROUND(
        (COUNT(CASE WHEN s.user_type = 'new' THEN 1 END) + 
         COUNT(CASE WHEN s.user_type = 'resurrected' THEN 1 END))::NUMERIC / 
        NULLIF(COALESCE(c.churned_count, 0), 0), 2
    ) AS quick_ratio
FROM user_status s
LEFT JOIN (
    SELECT churn_month, COUNT(*) AS churned_count
    FROM churned_users
    GROUP BY churn_month
) c ON s.active_month = c.churn_month
GROUP BY s.active_month, c.churned_count
ORDER BY s.active_month;
```

### Python Growth Accounting Visualization

```python
import pandas as pd
import matplotlib.pyplot as plt

def compute_growth_accounting(events_df):
    """Compute monthly growth accounting from events dataframe."""
    # Get monthly active users
    events_df['month'] = events_df['event_date'].dt.to_period('M')
    monthly = events_df.groupby(['user_id', 'month']).size().reset_index()
    monthly.columns = ['user_id', 'month', 'events']
    
    # First month per user
    first_month = monthly.groupby('user_id')['month'].min().reset_index()
    first_month.columns = ['user_id', 'first_month']
    
    results = []
    months = sorted(monthly['month'].unique())
    
    for i, month in enumerate(months):
        current_users = set(monthly[monthly['month'] == month]['user_id'])
        
        if i == 0:
            results.append({
                'month': month,
                'new': len(current_users),
                'retained': 0,
                'resurrected': 0,
                'churned': 0
            })
            continue
        
        prev_users = set(monthly[monthly['month'] == months[i-1]]['user_id'])
        new_this_month = set(first_month[first_month['first_month'] == month]['user_id'])
        
        retained = current_users & prev_users
        resurrected = (current_users - prev_users) - new_this_month
        churned = prev_users - current_users
        
        results.append({
            'month': month,
            'new': len(new_this_month & current_users),
            'retained': len(retained),
            'resurrected': len(resurrected),
            'churned': len(churned)
        })
    
    return pd.DataFrame(results)

# Visualize as stacked bar chart
def plot_growth_accounting(ga_df):
    fig, ax = plt.subplots(figsize=(12, 6))
    
    months = range(len(ga_df))
    ax.bar(months, ga_df['new'], label='New', color='green')
    ax.bar(months, ga_df['resurrected'], bottom=ga_df['new'], 
           label='Resurrected', color='blue')
    ax.bar(months, -ga_df['churned'], label='Churned', color='red')
    
    ax.set_xlabel('Month')
    ax.set_ylabel('Users')
    ax.set_title('Growth Accounting')
    ax.legend()
    plt.tight_layout()
```

### User Segmentation Query

```sql
WITH user_activity AS (
    SELECT 
        user_id,
        COUNT(DISTINCT DATE(event_timestamp)) AS days_active_30d,
        DATEDIFF(CURRENT_DATE, MAX(DATE(event_timestamp))) AS days_since_last,
        DATEDIFF(CURRENT_DATE, MIN(DATE(event_timestamp))) AS account_age_days
    FROM events
    WHERE event_timestamp >= CURRENT_DATE - INTERVAL '90 days'
    GROUP BY user_id
)
SELECT 
    user_id,
    CASE
        WHEN account_age_days <= 7 THEN 'new'
        WHEN days_since_last > 60 THEN 'churned'
        WHEN days_since_last > 30 THEN 'dormant'
        WHEN days_active_30d >= 20 THEN 'power'
        WHEN days_active_30d >= 10 THEN 'core'
        WHEN days_active_30d >= 3 THEN 'casual'
        ELSE 'at_risk'
    END AS user_segment
FROM user_activity;
```

---

## Interview Talking Points

> "I implemented growth accounting for our product and discovered that while we were growing 8% month-over-month, our Quick Ratio had declined from 3.2 to 1.8 over six months. The growth was increasingly driven by paid acquisition rather than organic retention. I presented this to leadership and we shifted $500K of acquisition budget toward retention features, which improved our Quick Ratio back to 2.5 within a quarter."

> "For user segmentation, I went beyond simple frequency-based buckets and used K-Means clustering on feature usage vectors. This revealed a 'dormant creator' segment — users who used to create content heavily but had shifted to passive consumption. A targeted campaign encouraging them to use our new simplified creation tool reactivated 23% of this segment."

> "When building the RFM model, I found that traditional quintile-based scoring created artificial boundaries. I instead used the raw RFM values as features in a random forest model to predict next-30-day revenue, which gave us a continuous scoring that was more actionable for personalization."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Counting only MAU growth without decomposition | Always decompose into new, retained, resurrected, churned |
| Confusing "retained" with "not churned" (they include new users) | Retained specifically means active in BOTH previous AND current period |
| Setting arbitrary segment thresholds without data | Use data distribution (percentiles, natural breaks) to define boundaries |
| Treating user segments as static | Recalculate segments regularly; track migration between segments |
| Using RFM scores as the final model | Use RFM as features in a predictive model, not as the model itself |
| Ignoring resurrected users in growth accounting | Resurrected users are critical — they validate your re-engagement efforts |
| Computing Quick Ratio only monthly | Compute weekly Quick Ratio for faster signal; monthly for trends |
| Applying same re-engagement strategy to all churned users | Segment churned users by reason, tenure, and recency for targeting |

---

## Rapid-Fire Q&A

**Q1: What is the growth accounting equation?**
MAU(t) = Retained + New + Resurrected; with Churned being those in MAU(t-1) but not in MAU(t). Net growth = New + Resurrected - Churned.

**Q2: What's a good Quick Ratio?**
Above 2.0 is healthy, above 4.0 is excellent. Below 1.0 means you're shrinking.

**Q3: How do you distinguish "resurrected" from "new"?**
Resurrected users were active in some prior period but not last period, then returned. New users have their first-ever activity this period.

**Q4: What's wrong with only looking at DAU/MAU ratio?**
It tells you engagement depth of current users but nothing about growth dynamics — you could have 80% DAU/MAU ratio while your total base shrinks rapidly.

**Q5: How would you use growth accounting to diagnose a growth slowdown?**
Check whether it's acquisition slowing (fewer new), retention worsening (more churned/fewer retained), or resurrection declining. Each points to different interventions.

**Q6: What's the difference between RFM and behavioral segmentation?**
RFM uses only purchase behavior (recency, frequency, monetary). Behavioral segmentation uses product interaction patterns (features used, content types, journeys).

**Q7: When would you prefer clustering over rule-based segmentation?**
When you don't have strong prior hypotheses about segment boundaries, when segments should be data-driven, or when the feature space is high-dimensional.

**Q8: How do you measure the success of a re-engagement campaign?**
Resurrection rate among targeted dormant/churned users vs. control group, plus post-resurrection retention (do they stay?).

**Q9: What's the relationship between Quick Ratio and steady-state MAU?**
At Quick Ratio = 1.0, MAU stabilizes. The steady-state MAU is determined by the equilibrium where inflow = outflow.

**Q10: How do you handle users who toggle between active and inactive?**
Define a "consistently active" threshold. Track streak length. Consider these "intermittent" users as a separate segment with its own expected cadence.

---

## ASCII Cheat Sheet

```
+============================================================+
|       GROWTH ACCOUNTING & SEGMENTATION CHEAT SHEET          |
+============================================================+

GROWTH EQUATION:
  MAU(t) = Retained + New + Resurrected
  Churned = MAU(t-1) - Retained
  Net Growth = New + Resurrected - Churned

QUICK RATIO:
  QR = (New + Resurrected) / Churned
  < 1.0  --> Shrinking
  2.0+   --> Healthy
  4.0+   --> Excellent

USER LIFECYCLE:
  New -> Activated -> Core -> Power
                        |
                     Casual -> At-Risk -> Dormant -> Churned
                                                       |
                                                  Resurrected

SEGMENT DEFINITIONS (typical):
  Power:      >= 20 days active / 30 days
  Core:       10-19 days active / 30 days
  Casual:     3-9 days active / 30 days
  At-Risk:    < 3 days active, still recent
  Dormant:    30-60 days since last active
  Churned:    60+ days since last active

RFM SCORING:
  R = Days since last purchase (lower = better)
  F = Total purchase count (higher = better)
  M = Total revenue (higher = better)
  Score each 1-5, combine for segment label

GROWTH HEALTH SIGNALS:
  Healthy:   Quick Ratio stable > 2.0
             Retention improving over time
             New user share < 30% of MAU
  
  Unhealthy: Quick Ratio declining toward 1.0
             Churned > New + Resurrected
             Growth entirely from paid acquisition

KEY METRICS TO TRACK:
  [ ] Monthly Growth Accounting (New/Retained/Resurrected/Churned)
  [ ] Quick Ratio (weekly + monthly)
  [ ] Segment distribution over time
  [ ] Segment migration matrix
  [ ] Resurrection rate by time-since-churn
  [ ] New user activation rate

+============================================================+
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 38 of 45*
