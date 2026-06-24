# 🎯 Topic 37: Retention, Churn, and Lifetime Value (LTV)

> *"Acquiring a user means nothing if you can't keep them. Retention is the single most important lever for sustainable growth — and LTV is how you put a dollar value on that retention."*

---

## 📑 Table of Contents

1. [Retention Fundamentals](#retention-fundamentals)
2. [Types of Retention Metrics](#types-of-retention-metrics)
3. [Cohort Retention Tables](#cohort-retention-tables)
4. [Churn: Contractual vs Non-Contractual](#churn-contractual-vs-non-contractual)
5. [Churn Prediction Approaches](#churn-prediction-approaches)
6. [LTV Calculation Methods](#ltv-calculation-methods)
7. [LTV:CAC Ratio and Unit Economics](#ltvcac-ratio-and-unit-economics)
8. [SQL Examples for Retention](#sql-examples-for-retention)
9. [Interview Talking Points](#interview-talking-points)
10. [Common Mistakes](#common-mistakes)
11. [Rapid-Fire Q&A](#rapid-fire-qa)
12. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Retention Fundamentals

Retention measures whether users return after their first experience. It is the foundation of product health because:

- It reflects actual product value delivery
- It compounds over time (small retention improvements lead to large user base growth)
- It determines whether your growth engine is filling a leaky bucket

### The Retention Curve

```
100% |*
     | *
     |  *
     |   *
     |    **
     |      ***
     |         ****
     |             *********
     |                      ************
     +-------------------------------------> Days
     D0  D1  D3  D7  D14  D30  D60  D90
```

A healthy retention curve **flattens** over time. If it keeps declining toward zero, the product has no core value proposition that retains users.

> **Critical Insight:** The shape of the retention curve matters more than any single point. A curve that flattens at 20% is healthier than one still declining at Day 30, even if Day 7 retention was higher.

---

## Types of Retention Metrics

### Classic (Day-N) Retention

User must be active on **exactly** Day N after their first action.

```
Day-7 Retention = Users active on exactly Day 7 / Users who joined on Day 0
```

**Pros:** Simple, standardized, easy to compare across products
**Cons:** Volatile (miss by one day and you're "churned"), penalizes irregular usage patterns

### Rolling (Unbounded) Retention

User must be active on Day N **or any day after** Day N.

```
Rolling Day-7 Retention = Users active on Day 7+ / Users who joined on Day 0
```

**Pros:** Less noisy, captures users who return later
**Cons:** Always >= classic retention, can mask deterioration

### Bounded (Bracket/Window) Retention

User must be active within a **window** around Day N (e.g., Day 7 +/- 1 day).

```
Bracket Day-7 Retention = Users active on Days 6-8 / Users who joined on Day 0
```

**Pros:** Balances precision with noise reduction
**Cons:** Requires defining window size (arbitrary)

### Comparison Table

| Metric | Definition | Best For | Weakness |
|--------|-----------|----------|----------|
| Classic Day-N | Active on exactly Day N | Daily-use apps (social media, games) | Very noisy for infrequent products |
| Rolling Day-N | Active on Day N or later | Long-term retention view | Overstates; hard to compare periods |
| Bounded Day-N | Active within window of Day N | Weekly/monthly products (e-commerce) | Window size is arbitrary |
| N-Day Active | Active at least once in N days | General engagement | Doesn't capture frequency depth |

> **Critical Insight:** Choose your retention metric based on your product's natural usage frequency. A grocery app (weekly) should NOT use Day-1 classic retention — it will look terrible even if the product is healthy.

---

## Cohort Retention Tables

A cohort retention table groups users by their join date and tracks activity over time.

```
             Week 0  Week 1  Week 2  Week 3  Week 4  Week 5
Jan 1 Cohort  100%    45%     32%     28%     25%     24%
Jan 8 Cohort  100%    48%     35%     30%     27%       -
Jan 15 Cohort 100%    50%     37%     32%       -       -
Jan 22 Cohort 100%    52%     38%       -       -       -
Jan 29 Cohort 100%    55%       -       -       -       -
```

**Reading the table:**
- **Rows** (horizontal): One cohort's journey over time
- **Columns** (vertical): How different cohorts perform at the same maturity
- **Diagonal** (top-right to bottom-left): What happened at the same calendar time

### What to Look For

1. **Improving columns** = product improvements are working (newer cohorts retain better)
2. **Flattening rows** = users find core value and stick
3. **Diagonal patterns** = external events affecting all users simultaneously (holiday, outage)

---

## Churn: Contractual vs Non-Contractual

### Contractual Churn

The customer explicitly cancels (subscription model). You **know** exactly when they leave.

- Examples: Netflix, gym memberships, SaaS subscriptions
- Churn rate = Customers who canceled in period / Customers at start of period
- Clear signal, easy to measure

### Non-Contractual Churn

The customer simply stops coming. You must **define** when they've churned.

- Examples: E-commerce, mobile apps, free-to-play games
- Requires defining an inactivity threshold (30 days? 90 days?)
- Much harder — is someone who hasn't visited in 45 days churned, or on vacation?

### Defining Churn Thresholds

```python
# Approach: Analyze inter-purchase intervals
import numpy as np

# For each user, compute gaps between consecutive actions
gaps = compute_inter_event_gaps(user_events)

# Use percentile to define threshold
# If 90% of active users return within X days, then X days of inactivity = churn
churn_threshold = np.percentile(gaps, 90)  # e.g., 42 days
```

> **Critical Insight:** In non-contractual settings, churn is a **modeling decision**, not an observable fact. Your threshold choice directly affects churn rate, LTV estimates, and intervention timing. Document and validate this choice rigorously.

---

## Churn Prediction Approaches

### Feature Engineering for Churn Models

| Feature Category | Examples |
|-----------------|----------|
| Recency | Days since last activity, days since last purchase |
| Frequency | Sessions per week, purchases per month (trend) |
| Monetary | Average order value, total spend, spend trend |
| Engagement | Features used, pages viewed, session duration |
| Behavioral Change | Week-over-week activity decline, feature abandonment |
| Customer Support | Tickets opened, complaints, negative feedback |
| Lifecycle | Account age, onboarding completion, plan type |

### Modeling Approaches

**1. Logistic Regression** — Interpretable, good baseline
```python
# Binary: will churn in next 30 days? Yes/No
from sklearn.linear_model import LogisticRegression
model = LogisticRegression()
model.fit(X_train, y_train)  # y = 1 if churned within 30 days
```

**2. Survival Analysis** — Models time-to-event, handles censoring
```python
from lifelines import CoxPHFitter
cph = CoxPHFitter()
cph.fit(df, duration_col='tenure_days', event_col='churned')
```

**3. Gradient Boosted Trees** — Best predictive performance
```python
import xgboost as xgb
model = xgb.XGBClassifier(objective='binary:logistic')
model.fit(X_train, y_train)
```

**4. BG/NBD (Probabilistic)** — For non-contractual settings
```python
from lifetimes import BetaGeoFitter
bgf = BetaGeoFitter()
bgf.fit(df['frequency'], df['recency'], df['T'])
# Predicts probability of being "alive"
```

### Key Considerations

- **Prediction window**: Predict churn within 7/14/30 days? Choose based on intervention lead time
- **Class imbalance**: Monthly churn is often 2-5%, use stratified sampling or SMOTE
- **Temporal validation**: NEVER use random train/test split — always split by time

---

## LTV Calculation Methods

### Method 1: Simple Formula (Contractual, Constant Churn)

```
LTV = ARPU / Churn Rate

Where:
- ARPU = Average Revenue Per User (per period)
- Churn Rate = fraction leaving per period
```

Example: ARPU = $50/month, Monthly Churn = 5%
```
LTV = $50 / 0.05 = $1,000
```

**With discount rate:**
```
LTV = ARPU / (Churn Rate + Discount Rate)
LTV = $50 / (0.05 + 0.01) = $833
```

### Method 2: Cohort-Based (Historical)

Track actual revenue from a cohort over time and extrapolate.

```python
# Sum revenue by cohort age
cohort_revenue = df.groupby(['cohort', 'months_since_join'])['revenue'].sum()
cumulative_ltv = cohort_revenue.groupby('cohort').cumsum()

# Extrapolate using curve fitting for immature cohorts
from scipy.optimize import curve_fit

def log_curve(x, a, b):
    return a * np.log(x + 1) + b

popt, _ = curve_fit(log_curve, months, cumulative_revenue)
projected_ltv = log_curve(36, *popt)  # Project to 36 months
```

### Method 3: Probabilistic (BG/NBD + Gamma-Gamma)

For non-contractual settings. Separates purchase frequency from monetary value.

```python
from lifetimes import BetaGeoFitter, GammaGammaFitter

# Step 1: Model purchase frequency
bgf = BetaGeoFitter()
bgf.fit(df['frequency'], df['recency'], df['T'])

# Step 2: Model monetary value (conditional on purchase)
ggf = GammaGammaFitter()
ggf.fit(df['frequency'], df['monetary_value'])

# Step 3: Combine for LTV
ltv = ggf.customer_lifetime_value(
    bgf, df['frequency'], df['recency'], df['T'],
    df['monetary_value'],
    time=12,  # months
    discount_rate=0.01
)
```

### Method 4: Markov Chain / State-Based

Model users transitioning between states (active, at-risk, dormant, churned).

```
States: Active -> At-Risk -> Dormant -> Churned
Each state has expected revenue and transition probabilities
LTV = sum of discounted expected revenue across all future states
```

### Comparison of LTV Methods

| Method | Complexity | Best For | Key Assumption |
|--------|-----------|----------|----------------|
| Simple ARPU/Churn | Low | Subscription, stable churn | Constant churn rate |
| Cohort-Based | Medium | Any (needs history) | Past cohorts predict future |
| BG/NBD + GG | High | Non-contractual | Statistical model assumptions |
| Markov Chain | High | Complex lifecycles | Memoryless transitions |

> **Critical Insight:** The simple LTV = ARPU/Churn formula assumes constant churn, which is almost never true. Early-tenure churn is always higher. Use cohort-based or probabilistic methods for any serious analysis.

---

## LTV:CAC Ratio and Unit Economics

### The Core Ratio

```
LTV:CAC Ratio = Customer Lifetime Value / Customer Acquisition Cost
```

| Ratio | Interpretation |
|-------|---------------|
| < 1:1 | Losing money on every customer acquired |
| 1:1 to 3:1 | Unprofitable to marginally profitable |
| 3:1 | Industry benchmark for healthy business |
| > 5:1 | Possibly under-investing in growth |

### Payback Period

```
Payback Period = CAC / (ARPU × Gross Margin)
```

- Measures months to recoup acquisition cost
- Target: < 12 months for most businesses
- Critical for capital-constrained startups

### Unit Economics Table

```
Revenue per customer/month:          $80
- COGS (hosting, support):           $20
= Gross Profit/month:                $60
Gross Margin:                         75%

Average Lifetime:                    24 months
LTV (gross):                         $1,920
LTV (net of COGS):                   $1,440

CAC:                                 $400
LTV:CAC:                             3.6:1
Payback:                             6.7 months
```

---

## SQL Examples for Retention

### Classic Day-N Retention

```sql
WITH user_first_day AS (
    SELECT user_id, 
           DATE(MIN(event_timestamp)) AS first_day
    FROM events
    GROUP BY user_id
),
retention AS (
    SELECT 
        f.first_day AS cohort_date,
        DATEDIFF(DATE(e.event_timestamp), f.first_day) AS day_n,
        COUNT(DISTINCT e.user_id) AS retained_users
    FROM user_first_day f
    JOIN events e ON f.user_id = e.user_id
    WHERE DATEDIFF(DATE(e.event_timestamp), f.first_day) IN (1, 3, 7, 14, 30)
    GROUP BY f.first_day, DATEDIFF(DATE(e.event_timestamp), f.first_day)
),
cohort_size AS (
    SELECT first_day AS cohort_date, 
           COUNT(*) AS total_users
    FROM user_first_day
    GROUP BY first_day
)
SELECT 
    r.cohort_date,
    r.day_n,
    r.retained_users,
    c.total_users,
    ROUND(100.0 * r.retained_users / c.total_users, 2) AS retention_rate
FROM retention r
JOIN cohort_size c ON r.cohort_date = c.cohort_date
ORDER BY r.cohort_date, r.day_n;
```

### Rolling Retention

```sql
WITH user_first_day AS (
    SELECT user_id, DATE(MIN(event_timestamp)) AS first_day
    FROM events GROUP BY user_id
),
last_active AS (
    SELECT user_id, DATE(MAX(event_timestamp)) AS last_active_day
    FROM events GROUP BY user_id
)
SELECT 
    f.first_day AS cohort_date,
    COUNT(*) AS cohort_size,
    SUM(CASE WHEN DATEDIFF(l.last_active_day, f.first_day) >= 7 THEN 1 ELSE 0 END) AS rolling_d7,
    ROUND(100.0 * SUM(CASE WHEN DATEDIFF(l.last_active_day, f.first_day) >= 7 
          THEN 1 ELSE 0 END) / COUNT(*), 2) AS rolling_d7_rate
FROM user_first_day f
JOIN last_active l ON f.user_id = l.user_id
GROUP BY f.first_day;
```

### Monthly Churn Rate

```sql
WITH monthly_active AS (
    SELECT user_id, DATE_TRUNC('month', event_timestamp) AS active_month
    FROM events
    GROUP BY user_id, DATE_TRUNC('month', event_timestamp)
),
churned AS (
    SELECT 
        a.user_id,
        a.active_month AS last_active_month
    FROM monthly_active a
    LEFT JOIN monthly_active b 
        ON a.user_id = b.user_id 
        AND b.active_month = a.active_month + INTERVAL '1 month'
    WHERE b.user_id IS NULL
)
SELECT 
    last_active_month,
    COUNT(*) AS churned_users,
    (SELECT COUNT(DISTINCT user_id) FROM monthly_active 
     WHERE active_month = churned.last_active_month) AS total_active,
    ROUND(100.0 * COUNT(*) / 
        (SELECT COUNT(DISTINCT user_id) FROM monthly_active 
         WHERE active_month = churned.last_active_month), 2) AS churn_rate
FROM churned
GROUP BY last_active_month
ORDER BY last_active_month;
```

---

## Interview Talking Points

> "When I approach retention analysis, I first define the metric carefully. For our mobile app, I chose bounded Day-7 retention with a +/- 1 day window because our users engaged roughly weekly, and classic Day-7 was too noisy. I then built a cohort retention table that showed newer cohorts retaining 8 percentage points better after our onboarding redesign — which translated to a projected 15% LTV increase using our cohort-based model."

> "For churn prediction, I built a gradient-boosted model using behavioral decline features — specifically the week-over-week change in session count was our strongest predictor. We achieved 0.82 AUC and deployed it to trigger proactive re-engagement campaigns 14 days before predicted churn, reducing churn by 12% in the treated group."

> "I calculated LTV using the BG/NBD + Gamma-Gamma approach because we were a non-contractual e-commerce business. The simple ARPU/churn formula overestimated LTV by 40% because it assumes constant churn, whereas our data showed heavy early-period attrition. The probabilistic model gave us segment-level LTV estimates that guided our marketing spend allocation across channels."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Using classic Day-1 retention for a weekly-use product | Use bounded retention matching natural usage frequency |
| Computing churn without defining the inactivity threshold rigorously | Analyze inter-event intervals to set data-driven thresholds |
| Using LTV = ARPU/Churn with non-constant churn | Use cohort-based or probabilistic LTV models |
| Including marketing costs in LTV (confusing LTV with net margin) | LTV is revenue or gross profit; compare separately to CAC |
| Random train/test split for churn prediction | Use temporal split (train on past, test on future) |
| Ignoring discount rate in multi-year LTV calculations | Apply discount rate (10-15% annually) for time value of money |
| Treating all churned users the same regardless of tenure | Segment by tenure — early churn is product failure, late churn is competition |
| Not accounting for resurrection in churn calculations | Track net churn (gross churn - resurrections) alongside gross churn |

---

## Rapid-Fire Q&A

**Q1: What's the difference between classic and rolling retention?**
Classic requires activity on exactly Day N; rolling requires activity on Day N or any day after. Rolling is always >= classic.

**Q2: How do you define churn for a non-contractual business?**
Analyze the distribution of inter-purchase intervals. Set threshold at 90th percentile — if 90% of returning users come back within X days, then X days of inactivity = churned.

**Q3: When does LTV = ARPU/Churn break down?**
When churn rate isn't constant (it almost never is — it's higher early), when ARPU varies by tenure, or when there's no natural subscription period.

**Q4: What's a good LTV:CAC ratio?**
3:1 is the benchmark. Below 1:1 is value-destructive. Above 5:1 suggests under-investment in growth.

**Q5: How would you improve retention by 5%?**
Identify where in the curve the biggest drop is (D1? D7? D30?), segment by user behavior at that point, find what retained users did differently, then design interventions to push at-risk users toward retained behavior.

**Q6: What features are most predictive of churn?**
Behavioral decline features (week-over-week session decrease), recency of last activity, engagement breadth (number of features used), and support ticket volume.

**Q7: How do you handle right-censoring in survival analysis for churn?**
Users still active haven't churned YET — they're censored. Survival models (Kaplan-Meier, Cox PH) handle this natively by treating them as "still at risk" without assuming they'll never churn.

**Q8: Why can't you just use retention rate as 1 - churn rate?**
You can in contractual settings. In non-contractual settings, retained users include those who haven't yet had the opportunity to return, so the math is more nuanced.

**Q9: What's the payback period and why does it matter?**
Months to recoup CAC from gross profit. It matters for cash flow — even with LTV:CAC of 5:1, if payback is 3 years, you need significant capital to grow.

**Q10: How do you validate a churn prediction model?**
Temporal validation (train on months 1-6, test on months 7-8), calibration plots (predicted probability matches actual churn rate), and lift charts (top decile captures 3-5x the base churn rate).

---

## ASCII Cheat Sheet

```
+============================================================+
|           RETENTION, CHURN & LTV CHEAT SHEET                |
+============================================================+

RETENTION TYPES:
  Classic Day-N:   Active on EXACTLY Day N
  Rolling Day-N:   Active on Day N OR LATER
  Bounded Day-N:   Active WITHIN WINDOW of Day N

RETENTION CURVE HEALTH:
  Good:  Drops then FLATTENS (found core value)
  Bad:   Keeps declining toward zero (no stickiness)
  Great: Smile curve — drops, flattens, then RISES

CHURN TYPES:
  Contractual:     Customer explicitly cancels
  Non-Contractual: Customer silently disappears
                   (YOU define the threshold)

LTV METHODS:
  Simple:          LTV = ARPU / Churn Rate
  With Discount:   LTV = ARPU / (Churn + Discount Rate)
  Cohort:          Track actual revenue, extrapolate curve
  Probabilistic:   BG/NBD (frequency) + Gamma-Gamma (value)

UNIT ECONOMICS:
  LTV:CAC < 1    --> Losing money (stop spending!)
  LTV:CAC = 3    --> Healthy benchmark
  LTV:CAC > 5    --> Under-investing in growth

  Payback Period = CAC / (ARPU * Gross Margin)
  Target: < 12 months

CHURN PREDICTION CHECKLIST:
  [ ] Define prediction window (7/14/30 days)
  [ ] Engineer behavioral decline features
  [ ] Handle class imbalance (2-5% churn)
  [ ] TEMPORAL train/test split (never random)
  [ ] Evaluate with AUC + calibration + lift
  [ ] Define intervention trigger threshold

KEY SQL PATTERNS:
  Cohort:     DATE(MIN(event_timestamp)) per user
  Day-N:      DATEDIFF from first_day to event_day
  Churn:      LEFT JOIN next month, WHERE NULL
  Rate:       retained / cohort_size * 100

+============================================================+
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 37 of 45*
