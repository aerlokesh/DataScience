# 🎯 Topic 48: Survival Analysis and Censored Data

> *"Every time-to-event question in data science — time to churn, time to convert, time to failure — is a survival analysis problem in disguise. The moment you ignore censoring and treat it as standard classification, you systematically bias your estimates and make worse decisions."*

---

## 📑 Table of Contents

1. [What Is Survival Analysis and Why It Matters](#what-is-survival-analysis-and-why-it-matters)
2. [Censoring: The Core Challenge](#censoring-the-core-challenge)
3. [Key Functions: Survival, Hazard, and Cumulative Hazard](#key-functions-survival-hazard-and-cumulative-hazard)
4. [Kaplan-Meier Estimator](#kaplan-meier-estimator)
5. [Log-Rank Test: Comparing Survival Curves](#log-rank-test-comparing-survival-curves)
6. [Cox Proportional Hazards Model](#cox-proportional-hazards-model)
7. [Parametric Survival Models](#parametric-survival-models)
8. [Applications in Data Science](#applications-in-data-science)
9. [Comparison with Classification Approaches](#comparison-with-classification-approaches)
10. [Python Implementation (lifelines)](#python-implementation-lifelines)
11. [Interview Talking Points](#interview-talking-points)
12. [Common Mistakes](#common-mistakes)
13. [Rapid-Fire Q&A](#rapid-fire-qa)
14. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## What Is Survival Analysis and Why It Matters

### The Core Question

Survival analysis answers: **"How long until an event occurs?"** — and critically, it handles the fact that for some observations, the event hasn't happened yet.

```
STANDARD REGRESSION/CLASSIFICATION:         SURVIVAL ANALYSIS:
  Input: features                            Input: features + time + event indicator
  Output: prediction (number or class)       Output: probability of event over TIME

  "Will this customer churn?" (yes/no)       "WHEN will this customer churn?"
  "Will this user convert?" (yes/no)         "What's the probability of conversion
                                              by day 7? Day 14? Day 30?"

  Problem: ignores TIME dimension            Advantage: models the FULL time curve
  Problem: can't handle "don't know yet"     Advantage: handles censored observations
```

### Why Standard Approaches Fail

```
SCENARIO: Predict customer churn for a subscription product

You have 10,000 customers. As of today:
  - 3,000 have churned (you know exactly when)
  - 7,000 are still active (haven't churned YET)

WRONG approach (binary classification):
  - Label active users as "not churned" (0)
  - Train model: features → {0, 1}
  - Problem: A user who signed up YESTERDAY is labeled same as
    a user active for 3 YEARS. They're fundamentally different!
  - Result: Underestimates churn risk (treats "not yet" as "never")

RIGHT approach (survival analysis):
  - For churned users: event=1, time=days until churn
  - For active users: event=0, time=days since signup (CENSORED)
  - Model learns the SHAPE of churn over time
  - Can predict: "30% probability of churning within 90 days"
```

> **Critical Insight:** Any time you're building a model where some subjects haven't experienced the outcome yet, you're dealing with censored data. Ignoring censoring biases your estimates. Excluding censored observations wastes data. Survival analysis is the only framework that correctly handles both.

---

## Censoring: The Core Challenge

### Types of Censoring

```
RIGHT CENSORING (most common — ~95% of real-world cases):
  We know the subject has survived AT LEAST until time T,
  but don't know the actual event time.
  
  Example: Customer signed up Jan 1, still active on June 30
           → Censored at 180 days (we know they survived 180+ days)
  
  Causes:
    - Study ends before event occurs
    - Subject drops out / lost to follow-up
    - Subject experiences a competing event

LEFT CENSORING (rare):
  The event already occurred BEFORE observation began.
  
  Example: Testing when users first notice a bug.
           Some users report it on day 1 — it may have existed earlier.
  
INTERVAL CENSORING:
  Event occurred between two observation points.
  
  Example: Weekly surveys — user was satisfied in week 3, 
           dissatisfied in week 4. Exact churn time is between week 3-4.
```

### Visual Representation

```
Subject A: ──────────────X              (Event at day 45)
Subject B: ──────────────────────→      (Censored at day 60, still active)
Subject C: ────X                        (Event at day 12)
Subject D: ─────────────────→          (Censored at day 50, lost to follow-up)
Subject E: ────────────X                (Event at day 30)
Subject F: ──────────────────────→      (Censored at day 60, study ended)

            ├────────────────────────┤
            0                        60 days
            
Key insight: B, D, F all have different censoring REASONS 
but are treated the same analytically (non-informative censoring assumption)
```

### The Non-Informative Censoring Assumption

```
CRITICAL ASSUMPTION in survival analysis:
  Censoring is INDEPENDENT of the event.
  
  Meaning: A censored subject has the same future risk as 
  an uncensored subject who has survived the same duration.

VALID (non-informative):
  - Study ends on a fixed date (administrative censoring)
  - Customer moves to a country where service isn't available
  - Random loss to follow-up

INVALID (informative censoring):
  - Sickest patients drop out because they're too ill to continue
  - Users who are about to churn stop logging in (we lose tracking)
  - Healthy people leave a clinical trial because they feel fine
  
  If censoring IS informative, standard survival analysis is BIASED.
  Solutions: sensitivity analysis, joint models, competing risks
```

> **Critical Insight:** In an interview, mentioning the non-informative censoring assumption and when it might be violated shows depth. Example: "In our churn model, users who delete the app are censored, but they might be MORE likely to churn than those who keep it installed — this informative censoring could bias our survival estimates downward."

---

## Key Functions: Survival, Hazard, and Cumulative Hazard

### Mathematical Relationships

```
SURVIVAL FUNCTION S(t):
  S(t) = P(T > t) = Probability of surviving past time t
  
  Properties:
    - S(0) = 1 (everyone alive at start)
    - S(∞) = 0 (everyone eventually experiences event)
    - Monotonically non-increasing
    - "What fraction of customers are still active at day t?"

HAZARD FUNCTION h(t):
  h(t) = lim[Δt→0] P(t ≤ T < t+Δt | T ≥ t) / Δt
  
  Intuition: "Given you've survived until time t, what's your
  INSTANTANEOUS risk of the event happening RIGHT NOW?"
  
  Properties:
    - Always non-negative
    - NOT a probability (can exceed 1)
    - Can increase, decrease, or be constant over time
    - "If a customer has been active for 90 days, what's their
       risk of churning TODAY?"

CUMULATIVE HAZARD H(t):
  H(t) = ∫₀ᵗ h(u) du = -ln(S(t))
  
  Intuition: "Total accumulated risk up to time t"
  
RELATIONSHIPS:
  S(t) = exp(-H(t))
  h(t) = -d/dt [ln(S(t))]
  H(t) = -ln(S(t))
```

### Hazard Shapes and Their Meaning

```
CONSTANT HAZARD (Exponential distribution):
  h(t) = λ (flat line)
  "Risk of event is the same regardless of how long you've survived"
  Example: Radioactive decay, random equipment failures
  
  h(t)
  │     ─────────────────────────
  │
  └──────────────────────────────── t

INCREASING HAZARD (Weibull with shape > 1):
  "The longer you survive, the HIGHER your risk"
  Example: Aging/wear-out failures, subscription fatigue
  
  h(t)
  │                          ╱
  │                     ╱
  │                ╱
  │           ╱
  │      ╱
  └──────────────────────────────── t

DECREASING HAZARD (Weibull with shape < 1):
  "The longer you survive, the LOWER your risk"
  Example: Early infant mortality, new user churn (if you survive week 1, you're likely safe)
  
  h(t)
  │  ╲
  │    ╲
  │      ╲───────────────────────
  │
  └──────────────────────────────── t

BATHTUB HAZARD (combination):
  "High risk early, stable middle, increasing risk late"
  Example: Human mortality, hardware lifecycle
  
  h(t)
  │  ╲                          ╱
  │    ╲                     ╱
  │      ╲──────────────╱
  │
  └──────────────────────────────── t
```

> **Critical Insight:** The shape of the hazard function tells you WHEN interventions matter most. If hazard is highest in week 1 (decreasing hazard), invest in onboarding. If hazard increases over time, invest in long-term engagement. Plot the hazard before building complex models — the shape alone drives strategy.

---

## Kaplan-Meier Estimator

### The Non-Parametric Survival Curve

```
KAPLAN-MEIER FORMULA:
  S_hat(t) = ∏(tᵢ ≤ t) [(nᵢ - dᵢ) / nᵢ]

  Where:
    tᵢ = distinct event times
    nᵢ = number at risk just before tᵢ
    dᵢ = number of events at tᵢ
    
  Step-by-step:
    At each event time, multiply survival by (1 - events/at_risk)
    Censored observations reduce "at risk" pool but don't create steps
```

### Worked Example

```
DATA: 5 customers tracked for churn
  Customer A: churned at day 10
  Customer B: censored at day 15 (still active)
  Customer C: churned at day 20
  Customer D: churned at day 20
  Customer E: censored at day 25 (still active)

CALCULATION:
  t=0:   S(0) = 1.000  (all 5 at risk)
  t=10:  n=5, d=1  →  S(10) = 1.0 × (5-1)/5 = 0.800
  t=15:  CENSORING (B leaves risk set, no event, no step)
  t=20:  n=3, d=2  →  S(20) = 0.8 × (3-2)/3 = 0.267
  t=25:  CENSORING (E leaves, no event)

SURVIVAL CURVE:
  S(t)
  1.0 │─────────┐
  0.8 │         └─────────┐
  0.6 │                   │
  0.4 │                   │
  0.267│                  └──────── →
  0.0 │
      └───────────────────────────── t
      0    10    15    20    25

  Reading: 80% survive past day 10, 26.7% survive past day 20
  Median survival: between day 10-20 (where curve crosses 0.5)
```

### Python Implementation

```python
from lifelines import KaplanMeierFitter
import pandas as pd
import matplotlib.pyplot as plt

# Data preparation
df = pd.DataFrame({
    'duration': [10, 15, 20, 20, 25, 30, 35, 40, 45, 50],
    'event': [1, 0, 1, 1, 0, 1, 0, 1, 1, 0]  # 1=churned, 0=censored
})

# Fit Kaplan-Meier
kmf = KaplanMeierFitter()
kmf.fit(durations=df['duration'], event_observed=df['event'])

# Key outputs
print(f"Median survival time: {kmf.median_survival_time_}")
print(f"Survival at day 30: {kmf.predict(30):.3f}")

# Survival table
print(kmf.survival_function_)

# Plot with confidence intervals
kmf.plot_survival_function(ci_show=True)
plt.xlabel('Days Since Signup')
plt.ylabel('Probability of Remaining Active')
plt.title('Customer Survival Curve')
plt.show()
```

> **Critical Insight:** Kaplan-Meier is your FIRST step in any survival analysis — before any modeling. It's the equivalent of plotting a histogram before regression. It shows you the shape of survival, where the biggest drops occur, and whether groups differ. Present KM curves to stakeholders first; they're intuitive even to non-technical audiences.

---

## Log-Rank Test: Comparing Survival Curves

### When to Use

The log-rank test answers: **"Do these two (or more) groups have statistically different survival experiences?"**

```
USE CASES:
  - Do premium users have different churn curves than free users?
  - Does treatment A lead to longer survival than treatment B?
  - Do users acquired from channel X retain differently than channel Y?
  
  H0: Survival curves are identical across groups
  H1: At least one group has a different survival experience
```

### Python Implementation

```python
from lifelines.statistics import logrank_test

# Compare survival between two segments
premium = df[df['plan'] == 'premium']
free = df[df['plan'] == 'free']

results = logrank_test(
    durations_A=premium['duration'],
    durations_B=free['duration'],
    event_observed_A=premium['event'],
    event_observed_B=free['event']
)

print(f"Test statistic: {results.test_statistic:.3f}")
print(f"P-value: {results.p_value:.4f}")

if results.p_value < 0.05:
    print("Significant difference in survival between premium and free users")

# Visual comparison
fig, ax = plt.subplots()
kmf_premium = KaplanMeierFitter()
kmf_premium.fit(premium['duration'], premium['event'], label='Premium')
kmf_premium.plot_survival_function(ax=ax)

kmf_free = KaplanMeierFitter()
kmf_free.fit(free['duration'], free['event'], label='Free')
kmf_free.plot_survival_function(ax=ax)

plt.xlabel('Days')
plt.ylabel('Survival Probability')
plt.title('Retention by Plan Type')
```

### Limitations of Log-Rank

```
LOG-RANK WORKS WHEN:
  ✓ Comparing groups (categorical variable)
  ✓ Proportional hazards assumption holds (curves don't cross)
  ✓ You want an overall test (not time-specific)

LOG-RANK FAILS WHEN:
  ✗ Curves cross (one group survives better early, worse later)
    → Use Wilcoxon (weighted log-rank) or piecewise analysis
  ✗ You need to control for confounders
    → Use Cox regression (multivariable)
  ✗ You want to quantify the effect size
    → Log-rank gives p-value only, not hazard ratio
    → Use Cox model for effect size
```

---

## Cox Proportional Hazards Model

### The Workhorse of Survival Analysis

```
COX MODEL:
  h(t|X) = h₀(t) × exp(β₁X₁ + β₂X₂ + ... + βₚXₚ)

  Where:
    h(t|X)  = hazard at time t given covariates X
    h₀(t)   = BASELINE hazard (unspecified — the "semi-parametric" part)
    exp(βX) = relative effect of covariates (CONSTANT over time)

  KEY PROPERTY:
    The hazard RATIO between any two subjects is CONSTANT over time:
    h(t|X₁) / h(t|X₂) = exp(β(X₁ - X₂))  ← no time dependence!
```

### Interpretation of Coefficients

```
COEFFICIENT INTERPRETATION:
  β > 0  →  exp(β) > 1  →  HIGHER hazard (event happens SOONER)
  β < 0  →  exp(β) < 1  →  LOWER hazard (event happens LATER)
  β = 0  →  exp(β) = 1  →  NO effect

EXAMPLE OUTPUT:
  Variable           coef     exp(coef)    p-value    Interpretation
  ─────────────────────────────────────────────────────────────────
  is_premium        -0.45     0.64         0.002      Premium users have 36%
                                                       LOWER hazard (churn slower)
  support_tickets    0.12     1.13         0.015      Each additional ticket 
                                                       increases churn hazard by 13%
  days_since_login   0.08     1.08         <0.001     Each day without login 
                                                       increases churn hazard by 8%
  
  READING exp(coef):
    exp(coef) = 0.64 means "64% of baseline hazard" = 36% risk REDUCTION
    exp(coef) = 1.13 means "113% of baseline hazard" = 13% risk INCREASE
    exp(coef) = 1.00 means no effect
```

### Python Implementation

```python
from lifelines import CoxPHFitter

# Prepare data: each row is a subject with duration, event, and features
df = pd.DataFrame({
    'duration': [30, 45, 60, 15, 90, 120, 25, 55, 80, 35],
    'churned': [1, 0, 1, 1, 0, 0, 1, 1, 0, 1],
    'is_premium': [0, 1, 0, 0, 1, 1, 0, 0, 1, 0],
    'num_logins_first_week': [2, 8, 3, 1, 12, 15, 2, 4, 9, 3],
    'support_tickets': [3, 0, 2, 5, 1, 0, 4, 2, 1, 3]
})

# Fit Cox model
cph = CoxPHFitter()
cph.fit(df, duration_col='duration', event_col='churned')

# Results
cph.print_summary()

# Hazard ratios
print("\nHazard Ratios:")
print(cph.hazard_ratios_)

# Predict survival for a specific customer
new_customer = pd.DataFrame({
    'is_premium': [1],
    'num_logins_first_week': [5],
    'support_tickets': [2]
})
survival_curve = cph.predict_survival_function(new_customer)
print(f"\nProbability active at day 60: {survival_curve.loc[60].values[0]:.3f}")
print(f"Probability active at day 90: {survival_curve.loc[90].values[0]:.3f}")
```

### Checking the Proportional Hazards Assumption

```python
# The PH assumption: hazard ratio between groups is CONSTANT over time
# If violated: coefficients don't have a single interpretation

# Test PH assumption
cph.check_assumptions(df, p_value_threshold=0.05)

# Schoenfeld residual test
# H0: coefficient does not vary with time
# If p < 0.05 → PH violated for that variable

# WHAT TO DO if PH is violated:
# 1. Stratify by the violating variable
#    cph.fit(df, duration_col='duration', event_col='event', strata=['segment'])
# 2. Add time interaction (time-varying coefficient)
# 3. Use a different model (AFT model, random survival forest)
# 4. Split time into intervals (piecewise Cox)
```

> **Critical Insight:** The hazard ratio from Cox regression is the single most important output to communicate. "Premium users have a 36% lower hazard of churning" is actionable. It directly quantifies the value of the premium plan in retention terms. In interviews, always translate coefficients to business impact.

---

## Parametric Survival Models

### When to Use Parametric vs Semi-Parametric

```
COX MODEL (semi-parametric):
  - Makes no assumption about h₀(t) shape
  - Only estimates relative effects (hazard ratios)
  - Cannot predict absolute survival probabilities without Breslow estimator
  - Best when: you care about covariate effects, not the baseline hazard shape

PARAMETRIC MODELS (Weibull, Exponential, Log-Normal, Log-Logistic):
  - Assume a specific distribution for survival times
  - Can predict exact survival probabilities
  - Can extrapolate beyond observed data
  - Best when: you need time predictions, forecasting, or simulation
```

### Common Parametric Distributions

| Distribution | Hazard Shape | Use When |
|-------------|-------------|----------|
| **Exponential** | Constant | Risk doesn't change with time (memoryless) |
| **Weibull** | Monotone increasing or decreasing | Risk steadily grows or shrinks |
| **Log-Normal** | Increases then decreases | Risk peaks at a certain time then drops |
| **Log-Logistic** | Increases then decreases | Similar to Log-Normal, heavier tails |
| **Gompertz** | Exponentially increasing | Aging processes |

### Weibull Model Example

```python
from lifelines import WeibullFitter, WeibullAFTFitter

# Non-parametric vs parametric comparison
wf = WeibullFitter()
wf.fit(df['duration'], df['churned'])

print(f"Weibull shape (rho): {wf.rho_:.3f}")
print(f"Weibull scale (lambda): {wf.lambda_:.3f}")

# Interpretation of shape parameter:
# rho < 1: decreasing hazard (early risk, survivors stabilize)
# rho = 1: constant hazard (exponential — no time dependence)
# rho > 1: increasing hazard (risk grows with time)

# Accelerated Failure Time (AFT) model — parametric regression
aft = WeibullAFTFitter()
aft.fit(df, duration_col='duration', event_col='churned')
aft.print_summary()

# AFT interpretation: coefficients ACCELERATE or DECELERATE event time
# exp(coef) > 1: extends survival (protective)
# exp(coef) < 1: shortens survival (risk factor)
```

---

## Applications in Data Science

### 1. Churn Prediction (Time-to-Churn)

```python
# Instead of: "Will they churn in 30 days?" (binary)
# Answer: "What's their survival curve?" (continuous)

cph = CoxPHFitter()
cph.fit(churn_df, duration_col='tenure_days', event_col='churned')

# Predict individual survival curves
customer_survival = cph.predict_survival_function(customer_features)

# Business actions based on survival predictions:
# If P(churn within 30 days) > 0.4 → trigger retention campaign
# If P(churn within 7 days) > 0.7 → escalate to human outreach
# If median remaining lifetime < 60 days → offer discount
```

### 2. Customer Lifetime Value (LTV)

```python
# LTV = ARPU × Expected Lifetime
# Expected Lifetime = ∫₀^∞ S(t) dt  (area under survival curve)

from lifelines.utils import restricted_mean_survival_time

# Expected lifetime (restricted to 365 days)
rmst = restricted_mean_survival_time(kmf, t=365)
print(f"Expected active days in first year: {rmst:.1f}")

# Per-customer LTV
monthly_arpu = 49.99
expected_months = rmst / 30
ltv = monthly_arpu * expected_months
print(f"Expected LTV: ${ltv:.2f}")

# Segment LTV by cohort
for cohort in cohorts:
    kmf.fit(cohort['duration'], cohort['event'])
    rmst = restricted_mean_survival_time(kmf, t=365)
    print(f"Cohort {cohort.name}: Expected lifetime = {rmst:.0f} days")
```

### 3. Time-to-Conversion (Marketing)

```python
# How long from first visit to purchase?
# Censored: users who haven't purchased yet

conversion_df = pd.DataFrame({
    'days_to_convert': days_since_first_visit,
    'converted': purchased_flag,  # 1 if purchased, 0 if still browsing
    'channel': acquisition_channel,
    'first_action': first_action_type,
    'page_views_day1': pv_count
})

cph = CoxPHFitter()
cph.fit(conversion_df, duration_col='days_to_convert', event_col='converted')

# Insight: "Users from organic search convert 40% faster (HR=1.4)
#           than paid social, controlling for engagement"
```

### 4. Subscription Renewal / Contract Expiry

```python
# Model time until subscription cancellation
# Account for different contract lengths

from lifelines import CoxPHFitter

# Time-varying covariates (e.g., usage declining month over month)
# Requires long-format data (one row per time interval per subject)

# Episode splitting for time-varying features:
from lifelines.utils import to_long_format, add_covariate_to_timeline

long_df = to_long_format(base_df, duration_col='months_active')
long_df = add_covariate_to_timeline(
    long_df, 
    monthly_usage_df, 
    duration_col='months_active',
    id_col='customer_id',
    event_col='churned'
)
```

### 5. A/B Test with Time-to-Event Outcome

```python
# Standard A/B test: "Is conversion rate higher in treatment?"
# Survival A/B test: "Does treatment ACCELERATE conversion?"

# Advantage: you can end the test earlier (detect differences sooner)
# because survival analysis uses timing information, not just counts

from lifelines.statistics import logrank_test

result = logrank_test(
    durations_A=control['time_to_event'],
    durations_B=treatment['time_to_event'],
    event_observed_A=control['converted'],
    event_observed_B=treatment['converted']
)
print(f"p-value: {result.p_value:.4f}")
# "Treatment group converts significantly faster (HR=1.3, p=0.02)"
```

---

## Comparison with Classification Approaches

### The Fundamental Tradeoff

| Aspect | Binary Classification | Survival Analysis |
|--------|---------------------|-------------------|
| **Question answered** | "Will event happen by day X?" | "When will event happen?" |
| **Handles censoring** | No (must label or exclude) | Yes (core feature) |
| **Time information** | Lost (fixed window) | Preserved |
| **Output** | Probability at ONE time point | Probability at ALL time points |
| **Window dependency** | Sensitive to chosen window | Window-free |
| **Data efficiency** | Wastes censored observations | Uses all observations |
| **Interpretability** | Feature importances | Hazard ratios (time-aware) |
| **Flexibility** | Random forests, XGBoost, etc. | Cox, AFT, survival forests |
| **Common tools** | sklearn, XGBoost | lifelines, scikit-survival |

### When Classification IS Appropriate

```
USE CLASSIFICATION WHEN:
  - You have a FIXED, well-defined prediction window
    "Will they churn in exactly 30 days?"
  - Very few censored observations (< 5%)
  - You need state-of-the-art prediction accuracy (XGBoost often wins)
  - Stakeholders only care about one time point
  - You need feature importance (SHAP on GBT is well-understood)

USE SURVIVAL ANALYSIS WHEN:
  - Significant censoring (> 20% of observations)
  - You need predictions at MULTIPLE time horizons
  - "When" matters as much as "whether"
  - You want to compare groups (KM curves, log-rank)
  - You need hazard ratios for communication
  - LTV estimation (need the full survival curve)
```

### Hybrid Approach

```python
# Random Survival Forests — best of both worlds
from sksurv.ensemble import RandomSurvivalForest

rsf = RandomSurvivalForest(
    n_estimators=200,
    min_samples_split=10,
    min_samples_leaf=5,
    random_state=42
)

# Input: structured array with event indicator and time
from sksurv.util import Surv
y = Surv.from_dataframe('churned', 'duration', df)
X = df[['is_premium', 'logins_week1', 'support_tickets']]

rsf.fit(X, y)

# Predict survival function for each individual
surv_funcs = rsf.predict_survival_function(X_test)

# Concordance index (C-index) — survival equivalent of AUC
from sksurv.metrics import concordance_index_censored
c_index = concordance_index_censored(
    y_test['churned'], y_test['duration'], rsf.predict(X_test)
)
print(f"C-index: {c_index[0]:.3f}")  # >0.5 = better than random
```

---

## Python Implementation (lifelines)

### Complete Workflow

```python
import pandas as pd
import numpy as np
from lifelines import (
    KaplanMeierFitter, 
    CoxPHFitter, 
    WeibullAFTFitter
)
from lifelines.statistics import logrank_test, multivariate_logrank_test
from lifelines.utils import median_survival_times
import matplotlib.pyplot as plt

# ============================================================
# STEP 1: Data Preparation
# ============================================================
# Each row: one subject with duration, event indicator, features
df = pd.read_csv('customer_survival.csv')
# duration: days from signup to churn (or censoring)
# event:    1 = churned, 0 = still active (censored)

print(f"Total customers: {len(df)}")
print(f"Churned: {df['event'].sum()} ({df['event'].mean()*100:.1f}%)")
print(f"Censored: {(1-df['event']).sum()} ({(1-df['event']).mean()*100:.1f}%)")

# ============================================================
# STEP 2: Kaplan-Meier (Non-parametric exploration)
# ============================================================
kmf = KaplanMeierFitter()
kmf.fit(df['duration'], df['event'], label='All Customers')

# Key statistics
print(f"Median survival: {kmf.median_survival_time_:.0f} days")
print(f"Survival at 30 days: {kmf.predict(30):.3f}")
print(f"Survival at 90 days: {kmf.predict(90):.3f}")
print(f"Survival at 365 days: {kmf.predict(365):.3f}")

# ============================================================
# STEP 3: Compare Segments (Log-Rank Test)
# ============================================================
fig, ax = plt.subplots(figsize=(10, 6))

for segment in df['plan_type'].unique():
    mask = df['plan_type'] == segment
    kmf.fit(df[mask]['duration'], df[mask]['event'], label=segment)
    kmf.plot_survival_function(ax=ax)

# Formal test
results = multivariate_logrank_test(
    df['duration'], df['plan_type'], df['event']
)
print(f"Log-rank p-value: {results.p_value:.4f}")

# ============================================================
# STEP 4: Cox Regression (Multivariable)
# ============================================================
cph = CoxPHFitter(penalizer=0.01)  # L2 regularization
cph.fit(
    df[['duration', 'event', 'is_premium', 'logins_week1', 
        'support_tickets', 'age', 'referral_source']],
    duration_col='duration',
    event_col='event'
)

cph.print_summary(decimals=3)

# Check proportional hazards assumption
cph.check_assumptions(df, show_plots=True)

# ============================================================
# STEP 5: Individual Predictions
# ============================================================
new_customers = pd.DataFrame({
    'is_premium': [1, 0],
    'logins_week1': [7, 2],
    'support_tickets': [0, 3],
    'age': [35, 22],
    'referral_source': [1, 0]
})

# Survival curves per customer
surv = cph.predict_survival_function(new_customers)
# Median expected lifetime
medians = cph.predict_median(new_customers)
print(f"Customer 1 median lifetime: {medians.iloc[0]:.0f} days")
print(f"Customer 2 median lifetime: {medians.iloc[1]:.0f} days")
```

### Evaluation Metrics for Survival Models

```python
from lifelines.utils import concordance_index
from sksurv.metrics import (
    concordance_index_censored,
    integrated_brier_score
)

# C-INDEX (concordance): P(model ranks higher-risk subjects correctly)
# Range: 0.5 (random) to 1.0 (perfect)
c_index = concordance_index(
    df['duration'], -cph.predict_partial_hazard(df), df['event']
)
print(f"C-index: {c_index:.3f}")

# BRIER SCORE: Calibration metric (predicted probability vs actual outcome)
# Lower is better, 0.25 = random for binary

# TIME-DEPENDENT AUC: AUC at specific time points
# "At day 30, how well does the model discriminate who will churn?"
```

---

## Interview Talking Points

> "I used survival analysis to model time-to-first-purchase for our e-commerce platform. The key insight was that 40% of users who eventually convert do so within 48 hours, but treating unconverted users as 'will never convert' (binary classification) drastically underestimated conversion potential for recent visitors. The Cox model revealed that users who added items to cart had a 3.2x higher conversion hazard, which informed our retargeting timing — we now retarget cart-abandoners within 24 hours instead of 72."

> "For our subscription LTV model, I chose survival analysis over a simple average because 60% of our customer base hadn't churned yet — they were all censored. Using Kaplan-Meier, I estimated that median customer lifetime was 14 months, but the naive approach of averaging only churned customers gave 6 months — a massive underestimate because it ignored the long-tenured active customers. This corrected LTV fed directly into our CAC targets."

> "When presenting to our product team, I use Kaplan-Meier curves because they're visually intuitive — 'here's the percentage of users still active over time, and here's how it differs between cohorts.' When I need to quantify individual feature effects, I use Cox regression and communicate hazard ratios: 'Users who complete onboarding have 45% lower churn hazard — that's the single highest-leverage intervention.'"

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Excluding censored observations from analysis | Include them — they contain information about survival |
| Using binary classification when >20% data is censored | Use survival analysis to properly handle censored observations |
| Ignoring the proportional hazards assumption | Test with Schoenfeld residuals; stratify or use AFT if violated |
| Reporting mean survival when distribution is skewed | Report median survival time (more robust to outliers/censoring) |
| Interpreting Cox coefficient directly (not exp(coef)) | Always report hazard ratios: exp(coef) = % change in hazard |
| Using Kaplan-Meier with many confounders | KM is univariate; use Cox regression for multivariable adjustment |
| Assuming censoring is non-informative without checking | Evaluate whether censoring mechanism is related to outcome |
| Predicting beyond observed time range with non-parametric models | Use parametric models (Weibull) for extrapolation, with caveats |
| Confusing hazard with probability | Hazard is an instantaneous rate (can exceed 1), not a probability |
| Computing LTV as ARPU x average observed tenure | Use area under KM curve (accounts for censoring and full distribution) |

---

## Rapid-Fire Q&A

**Q1: What is censoring and why does it matter?**
Censoring occurs when we don't observe the actual event time for some subjects (e.g., they're still active, or lost to follow-up). It matters because ignoring it biases estimates — treating censored as "no event" underestimates risk; excluding them wastes data and introduces selection bias. Survival analysis is specifically designed to handle this.

**Q2: What's the difference between survival function and hazard function?**
Survival S(t) = probability of surviving PAST time t (cumulative, decreasing from 1 to 0). Hazard h(t) = instantaneous risk of event AT time t, given survival up to t (a rate, not bounded by 1). They're mathematically linked: S(t) = exp(-integral of h(t)).

**Q3: How would you model time-to-first-purchase?**
Use Cox regression with: features captured at first visit (channel, device, pages viewed, items browsed, cart additions), duration = days until purchase, event = 1 if purchased, 0 if still hasn't. Report hazard ratios for each feature. Use predicted survival curves to optimize retargeting timing. Check PH assumption — cart abandoners may have non-proportional hazards (high initial risk that decays).

**Q4: What is the proportional hazards assumption?**
In Cox regression, it assumes the hazard ratio between any two subjects is CONSTANT over time. If subject A has 2x the hazard of B at day 10, they still have 2x at day 100. Test with Schoenfeld residuals. If violated: stratify the model, add time interactions, or switch to an AFT model.

**Q5: What's the concordance index (C-index)?**
The C-index measures how well the model ranks subjects by risk. It's the probability that, given two subjects where one had the event earlier, the model assigned higher risk to that subject. Range: 0.5 (random) to 1.0 (perfect discrimination). It's the survival analysis equivalent of AUC-ROC.

**Q6: When would you use Kaplan-Meier vs Cox regression?**
Kaplan-Meier: descriptive, non-parametric, one variable at a time (compare groups), no covariates, great for visualization and communication. Cox regression: inferential, semi-parametric, multiple covariates simultaneously, controls for confounders, quantifies individual feature effects as hazard ratios.

**Q7: How does survival analysis help with LTV estimation?**
LTV = ARPU x Expected Lifetime. Expected lifetime = area under the survival curve (integral of S(t)). Survival analysis handles censored customers correctly — you don't underestimate lifetime by averaging only churned customers. KM or parametric models give the full survival curve; integrate it for expected duration.

**Q8: What's the difference between Cox (PH) and AFT models?**
Cox models the HAZARD: "how does X change the risk of event?" (multiplicative on hazard). AFT models the TIME directly: "how does X accelerate or decelerate the event?" (multiplicative on survival time). AFT is more intuitive ("premium extends lifetime by 40%") but requires distributional assumption. Cox is more flexible (no baseline hazard assumption).

**Q9: How do you handle time-varying covariates in survival analysis?**
Split each subject's timeline into intervals. In each interval, update the covariate value. Use "episode splitting" (long format: one row per interval per subject). Cox regression supports this via start/stop notation. Example: monthly login count changes — create one row per month per customer with that month's login count.

**Q10: What are competing risks and when do they matter?**
Competing risks occur when multiple event types exist and one precludes the other. Example: a customer can churn voluntarily OR be involuntarily deactivated — if deactivated, they can no longer voluntarily churn. Standard survival analysis (treating competing events as censored) overestimates the probability of each event. Use cause-specific hazards or Fine-Gray subdistribution model.

---

## ASCII Cheat Sheet

```
+============================================================+
|    SURVIVAL ANALYSIS & CENSORED DATA — CHEAT SHEET         |
+============================================================+

CORE CONCEPT:
  Q: "How long until an event occurs?"
  Challenge: Some subjects haven't experienced the event yet (CENSORED)
  Solution: Survival analysis handles this correctly

CENSORING TYPES:
  Right (most common): event hasn't occurred yet (still active)
  Left: event occurred before observation started
  Interval: event occurred between two observation points
  ASSUMPTION: censoring is NON-INFORMATIVE (independent of outcome)

KEY FUNCTIONS:
  S(t) = P(survive past time t)     [decreasing, 1 → 0]
  h(t) = instantaneous risk at t    [rate, not bounded]
  H(t) = cumulative hazard          [increasing, 0 → ∞]
  S(t) = exp(-H(t))

HAZARD SHAPES:
  Constant:   same risk at all times     (Exponential)
  Increasing: risk grows with time       (Weibull, shape>1)
  Decreasing: risk drops with time       (Weibull, shape<1)
  Bathtub:    high-low-high              (mixture)

METHODS HIERARCHY:
  1. Kaplan-Meier:   non-parametric, descriptive, one group
  2. Log-Rank Test:  compare KM curves between groups
  3. Cox PH Model:   semi-parametric, multivariable, hazard ratios
  4. Parametric:     Weibull/AFT, extrapolation, time predictions
  5. ML models:      Random Survival Forest, DeepSurv

COX MODEL ESSENTIALS:
  h(t|X) = h0(t) × exp(β₁X₁ + β₂X₂ + ...)
  exp(coef) = Hazard Ratio
    HR > 1: increases risk (bad for survival)
    HR < 1: decreases risk (protective)
    HR = 1: no effect
  ASSUMPTION: proportional hazards (constant HR over time)
  TEST: Schoenfeld residuals (p < 0.05 → PH violated)

EVALUATION METRICS:
  C-index:     discrimination (like AUC), 0.5 = random, 1.0 = perfect
  Brier Score: calibration, lower = better
  Log-rank:    group comparison significance test

DS APPLICATIONS:
  Churn:       time until subscription cancellation
  LTV:         ARPU × area under survival curve
  Conversion:  time until first purchase
  Engagement:  time until feature adoption
  A/B Testing: "does treatment accelerate the outcome?"

SURVIVAL vs CLASSIFICATION:
  Use survival when: >20% censored, need "when" not just "if",
  multiple time horizons, LTV estimation
  Use classification when: fixed window, few censored, need SHAP

PYTHON TOOLS:
  lifelines:       KaplanMeierFitter, CoxPHFitter, WeibullAFTFitter
  scikit-survival: RandomSurvivalForest, concordance_index_censored
  statsmodels:     PHReg (less user-friendly)

LTV FORMULA:
  Expected Lifetime = ∫₀^T S(t) dt = area under KM curve
  LTV = ARPU × Expected Lifetime
  Don't just average churned customer tenures! (ignores censored)

COMMON INTERVIEW PATTERNS:
  "Model time to first purchase"    → Cox with visit-level features
  "Compare retention across plans"  → KM curves + log-rank test
  "Predict when a user will churn"  → Cox or Weibull AFT, individual curves
  "Calculate LTV correctly"         → KM + restricted mean survival time

+============================================================+
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 48 of 51*
