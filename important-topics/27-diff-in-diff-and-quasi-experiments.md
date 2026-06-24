# 🎯 Topic 27: Difference-in-Differences and Quasi-Experiments

> *"When you can't randomize, let time and natural variation be your experimental design."*

---

## Table of Contents

1. [Difference-in-Differences (DiD) Method](#did-method)
2. [The Parallel Trends Assumption](#parallel-trends)
3. [DiD Implementation and Extensions](#did-implementation)
4. [Synthetic Control Method](#synthetic-control)
5. [Interrupted Time Series (ITS)](#interrupted-time-series)
6. [Regression Discontinuity Design (RDD)](#regression-discontinuity)
7. [When to Use Each Method](#when-to-use)
8. [Real-World Examples](#real-examples)
9. [Interview Talking Points](#interview-talking-points)
10. [Common Mistakes](#common-mistakes)
11. [Rapid-Fire Q&A](#rapid-fire-qa)
12. [ASCII Cheat Sheet](#cheat-sheet)

---

## Difference-in-Differences (DiD) Method {#did-method}

### The Core Idea

Compare the CHANGE in outcomes over time between a treated group and a control group.

```
                  Before    After     Difference
Treatment group:   Y_T0      Y_T1     ΔY_T = Y_T1 - Y_T0
Control group:     Y_C0      Y_C1     ΔY_C = Y_C1 - Y_C0

DiD Estimate = ΔY_T - ΔY_C = (Y_T1 - Y_T0) - (Y_C1 - Y_C0)
             = (Treatment change) - (What would have happened anyway)
```

### Visual Intuition

```
Outcome
    │            
    │        ● Treatment (actual)
    │       /
    │      / ← DiD Effect (causal impact)
    │     /
    │    ○─ ─ ─ ─ ○ Treatment (counterfactual, parallel to control)
    │   /         /
    │  /         /
    │ ●─────────● Control (actual)
    │
    └────────┼──────────→ Time
          Treatment
          Introduced
```

### Why "Difference-in-Differences"?

```
First difference:  Remove time-invariant group differences
  → Y_T1 - Y_T0 controls for permanent treatment group traits
  
Second difference: Remove common time trends
  → (Y_T1 - Y_T0) - (Y_C1 - Y_C0) removes shared temporal changes

What's left: the treatment effect
  → Only thing that changed for treatment but NOT control
```

> **Critical Insight** 💡
> DiD eliminates two types of confounding: (1) permanent differences between groups (treatment group may always be higher/lower), and (2) common time trends (both groups may be growing/declining). What it CANNOT eliminate: differential time trends — if the treatment group was already growing FASTER than control before the intervention.

### The DiD Regression

```python
import statsmodels.api as sm
import pandas as pd

# Standard DiD regression:
# Y = β₀ + β₁ × Treatment + β₂ × Post + β₃ × (Treatment × Post) + ε
#
# β₀: baseline (control, pre-period)
# β₁: permanent group difference
# β₂: common time trend
# β₃: THE DiD ESTIMATE (causal effect)

model = sm.OLS.from_formula(
    'outcome ~ treatment + post + treatment:post',
    data=df
)
results = model.fit(cov_type='cluster', cov_kwds={'groups': df['unit_id']})
print(results.summary())
# β₃ (treatment:post coefficient) = DiD estimate
```

### DiD as a 2x2 Table

| | Pre-Period | Post-Period | Difference |
|---|-----------|-------------|-----------|
| **Treatment** | A | B | B - A |
| **Control** | C | D | D - C |
| **Difference** | A - C | B - D | **(B-A) - (D-C)** = DiD |

---

## The Parallel Trends Assumption {#parallel-trends}

### What It Requires

In the ABSENCE of treatment, the treatment group would have followed the same trend as the control group.

```
Key: We can NEVER directly test this (it's about a counterfactual)
But we CAN check if trends were parallel BEFORE treatment.
```

### Testing Pre-Trends

```python
import matplotlib.pyplot as plt
import numpy as np

# Method 1: Visual inspection of pre-treatment trends
fig, ax = plt.subplots(figsize=(10, 6))
for group in ['treatment', 'control']:
    group_data = df[df['group'] == group].groupby('time')['outcome'].mean()
    ax.plot(group_data.index, group_data.values, marker='o', label=group)
ax.axvline(x=treatment_time, color='red', linestyle='--', label='Treatment')
ax.legend()
ax.set_title('Pre-Treatment Parallel Trends Check')
plt.show()

# Method 2: Event study / leads and lags regression
# Y = α + Σ_t β_t × Treatment × 1(time=t) + γ × Treatment + δ_t + ε
# Pre-treatment β_t should be ≈ 0 (parallel trends)
# Post-treatment β_t = dynamic treatment effect

# Event study specification
formula = 'outcome ~ C(time) + treatment'
for t in range(-4, 5):  # 4 pre-periods, 4 post-periods
    if t != -1:  # reference period
        formula += f' + treatment:I(time=={t})'
        
event_study = sm.OLS.from_formula(formula, data=df).fit(
    cov_type='cluster', cov_kwds={'groups': df['unit_id']}
)
```

### Event Study Plot

```
Effect
    │
 +3 │                              ●
    │                         ●
 +2 │                    ●
    │
 +1 │               ●
    │          │
  0 ├──●──●──●──●──┤─────────────────  ← Pre-treatment should be flat at 0
    │               │
 -1 │               │
    │               │
    └───────────────┼──────────────→ Time relative to treatment
                    0
    ←── Pre-treatment ──→←── Post-treatment ──→
    (should be ≈ 0)       (= treatment effect)
```

> **Critical Insight** 💡
> Parallel pre-trends are NECESSARY but NOT SUFFICIENT for the parallel trends assumption. Even if trends were parallel for 5 years before treatment, they could diverge exactly when treatment occurs for reasons unrelated to treatment. Always consider: "What else changed at the same time that might affect the treatment group differently?"

### When Parallel Trends Fails

| Situation | Problem | Possible Fix |
|-----------|---------|-------------|
| Treatment group was growing faster | Pre-existing differential trend | Add group-specific time trends |
| Anticipation effects | Groups react before treatment | Exclude anticipation period |
| Composition changes | Who's in the group changes | Use consistent panel |
| Concurrent shocks | Other policy hit treatment group | Find additional controls or use synthetic control |

---

## DiD Implementation and Extensions {#did-implementation}

### Staggered DiD (Multiple Timing)

When treatment occurs at different times for different units:

```
Unit A: treated from t=3
Unit B: treated from t=5
Unit C: treated from t=7
Unit D: never treated (pure control)

Problem with two-way fixed effects (TWFE):
  Standard TWFE can use already-treated units as "controls"
  for newly-treated units → BIASED if effects change over time
```

### Modern Solutions for Staggered DiD

| Method | Key Idea | When to Use |
|--------|----------|-------------|
| Callaway & Sant'Anna (2021) | Group-time ATTs, never/not-yet-treated as controls | Heterogeneous effects across cohorts |
| Sun & Abraham (2021) | Interaction-weighted estimator | Want to decompose TWFE bias |
| de Chaisemartin & D'Haultfoeuille | Remove "negative weights" problem | Dynamic treatment effects |
| Borusyak et al. (imputation) | Impute counterfactual from clean controls | Flexible, handles dynamics |

```python
# Simplified Callaway-Sant'Anna approach (conceptual)
# For each cohort g (group treated at time g):
#   1. Identify clean control group (never-treated or not-yet-treated)
#   2. Estimate DiD for that cohort relative to its treatment time
#   3. Aggregate cohort-level estimates

# Using the 'did' package concept:
# ATT(g, t) = E[Y_t(g) - Y_t(0) | G=g]  for each group g at time t
# Overall ATT = weighted average of ATT(g,t)
```

### Triple Differences (DDD)

Add a THIRD difference to address remaining concerns:

```
DiD:  (Treatment_Post - Treatment_Pre) - (Control_Post - Control_Pre)

DDD adds a third comparison group:
  Compare the DiD across an affected subgroup vs. unaffected subgroup

Example: New policy affects women's wages (treatment=women, control=men)
  But wages changed for other reasons too
  DDD: Compare (Women_Treated_State vs Men_Treated_State) DiD
       vs (Women_Untreated_State vs Men_Untreated_State) DiD
```

---

## Synthetic Control Method {#synthetic-control}

### When to Use

- Few treated units (often just ONE: a single state, country, or market)
- Many potential control units
- Standard DiD can't find a good parallel control

### The Idea

Create a "synthetic" control by finding the weighted combination of untreated units that best matches the treated unit's pre-treatment trajectory.

```
Treated unit: California
Potential donors: Other 49 states

Synthetic California = 0.25×Utah + 0.35×Colorado + 0.15×Nevada + 0.25×Oregon
(weights chosen to match California's pre-treatment outcomes and covariates)
```

### Visual

```
Outcome
    │        ● California (actual)
    │       / 
    │      /  ← Treatment effect (gap)
    │     /
    │    / ○── Synthetic California (counterfactual)
    │   ○ /
    │  ○○
    │ ●●
    │●○  (pre-treatment: treated ≈ synthetic)
    └────────┼──────────→ Time
          Treatment
```

### Implementation

```python
import numpy as np
from scipy.optimize import minimize

def synthetic_control(treated_pre, donor_pre, treated_post, donor_post):
    """
    Find weights for donor units that minimize pre-treatment distance.
    """
    n_donors = donor_pre.shape[1]
    
    # Objective: minimize squared difference in pre-treatment period
    def objective(weights):
        synthetic = donor_pre @ weights
        return np.sum((treated_pre - synthetic) ** 2)
    
    # Constraints: weights sum to 1, all non-negative
    constraints = {'type': 'eq', 'fun': lambda w: np.sum(w) - 1}
    bounds = [(0, 1)] * n_donors
    
    # Initial weights (uniform)
    w0 = np.ones(n_donors) / n_donors
    
    result = minimize(objective, w0, bounds=bounds, constraints=constraints)
    weights = result.x
    
    # Estimate treatment effect
    synthetic_post = donor_post @ weights
    treatment_effect = treated_post - synthetic_post
    
    return weights, treatment_effect

# Inference: Placebo tests (apply same method to each donor unit)
# If treated unit's gap is unusually large compared to placebo gaps,
# the effect is "significant"
```

### Placebo Tests for Inference

```
1. Apply synthetic control to EACH donor unit (pretend it was treated)
2. Compute the "gap" for each donor unit
3. Compare real treated unit's gap to distribution of placebo gaps
4. p-value ≈ rank of treated unit's gap / (N_donors + 1)

If treated unit has the largest gap out of 20 donors:
  p-value ≈ 1/21 ≈ 0.048
```

> **Critical Insight** 💡
> Synthetic control is most powerful when you have ONE treated unit and a good donor pool. Its strength is transparency — you can SEE the counterfactual and assess pre-treatment fit visually. Its weakness: inference relies on placebo tests and can be underpowered with few donor units.

---

## Interrupted Time Series (ITS) {#interrupted-time-series}

### When to Use

- No control group available at all
- Treatment occurs at a known point in time for ALL units
- Long pre-treatment time series available

### The Model

```
Y_t = β₀ + β₁×time + β₂×intervention + β₃×time_after_intervention + ε_t

β₀: baseline level
β₁: pre-intervention trend (slope)
β₂: immediate level change at intervention (LEVEL EFFECT)
β₃: change in trend after intervention (SLOPE EFFECT)
```

### Visual

```
Outcome
    │                    ● ●
    │                  ● 
    │               ●← β₃ (slope change)
    │            ●
    │         │●← β₂ (level shift)
    │       ● │
    │     ●   │
    │   ●     │
    │ ●       │
    │●        │
    └─────────┼──────────→ Time
         Intervention
```

### Implementation

```python
import statsmodels.api as sm
import pandas as pd
import numpy as np

# Create ITS variables
df['time'] = range(len(df))
df['intervention'] = (df['date'] >= intervention_date).astype(int)
df['time_after'] = df['time'] * df['intervention'] - \
                   df[df['intervention']==1]['time'].min() * df['intervention']
df['time_after'] = df['time_after'].clip(lower=0)

# ITS regression
model = sm.OLS.from_formula(
    'outcome ~ time + intervention + time_after',
    data=df
)
results = model.fit(cov_type='HAC', cov_kwds={'maxlags': 4})  # Newey-West for autocorrelation
print(results.summary())

# Interpretation:
# intervention coefficient: immediate effect at intervention time
# time_after coefficient: change in trend due to intervention
```

### Threats to ITS Validity

| Threat | Description | Mitigation |
|--------|-------------|-----------|
| Co-occurring events | Something else changed at same time | Add control series |
| Seasonality | Confounded with seasonal pattern | Include seasonal dummies |
| Autocorrelation | Residuals correlated over time | Newey-West SE or ARIMA errors |
| Non-linear pre-trend | Linear model misspecifies trend | Use flexible pre-trend specification |
| Anticipation | Behavior changes before intervention | Exclude lead-up period |
| Regression to mean | Intervention triggered by extreme values | Use longer pre-period |

---

## Regression Discontinuity Design (RDD) {#regression-discontinuity}

### The Setup

Treatment assigned by whether a continuous "running variable" exceeds a cutoff.

```
Treatment rule: T = 1 if X ≥ c

Key insight: At the cutoff, assignment is as-if random
  - Units at X = c - 0.01 and X = c + 0.01 are nearly identical
  - But one gets treatment and the other doesn't
```

### Sharp RDD

```python
import numpy as np
from sklearn.linear_model import LinearRegression
# Using rdrobust is preferred in practice

def sharp_rdd(running_var, outcome, cutoff, bandwidth):
    """Estimate local linear RDD."""
    # Select observations within bandwidth of cutoff
    mask = np.abs(running_var - cutoff) <= bandwidth
    X_local = running_var[mask]
    Y_local = outcome[mask]
    
    # Create design matrix
    above = (X_local >= cutoff).astype(float)
    X_centered = X_local - cutoff
    interaction = above * X_centered
    
    design = np.column_stack([
        np.ones(len(X_local)),
        above,
        X_centered,
        interaction
    ])
    
    # Fit local linear regression
    model = LinearRegression(fit_intercept=False)
    model.fit(design, Y_local)
    
    # The coefficient on 'above' is the RDD estimate
    rdd_effect = model.coef_[1]
    return rdd_effect
```

### Fuzzy RDD

When crossing the cutoff doesn't guarantee treatment but increases its probability:

```
Sharp: P(T=1 | X≥c) = 1,  P(T=1 | X<c) = 0
Fuzzy: P(T=1 | X≥c) > P(T=1 | X<c) but not 0 or 1

Fuzzy RDD = IV approach:
  Instrument: Z = 1(X ≥ c) (being above cutoff)
  Treatment: T (actual treatment received)
  
  Effect = (jump in outcome at c) / (jump in treatment probability at c)
         = Intent-to-Treat / First Stage
         = LATE for compliers at the cutoff
```

### Bandwidth Selection

| Approach | Description |
|----------|-------------|
| MSE-optimal (IK/CCT) | Minimizes mean squared error of RDD estimator |
| CER-optimal | Minimizes coverage error rate of confidence intervals |
| Sensitivity check | Report results at 0.5×, 1×, 1.5×, 2× bandwidth |

### Validation Checks

```
1. McCrary density test: Is there bunching at the cutoff?
   → If yes, people are manipulating their score

2. Covariate balance: Are pre-treatment covariates smooth at cutoff?
   → Plot each covariate against running variable

3. Placebo cutoffs: Run RDD at fake cutoffs (where no treatment)
   → Should find no effect at placebo cutoffs

4. Sensitivity to bandwidth: Does estimate change dramatically?
   → Robust results should be stable across reasonable bandwidths
```

---

## When to Use Each Method {#when-to-use}

| Scenario | Best Method | Why |
|----------|-------------|-----|
| Treatment at one time for some units, good controls exist | DiD | Compare changes across groups |
| Treatment at one time, ONE treated unit | Synthetic Control | Construct weighted counterfactual |
| Treatment at one time, NO control group | Interrupted Time Series | Use pre-trend as control |
| Treatment determined by a score/threshold | RDD | Local randomization at cutoff |
| Treatment at different times for different units | Staggered DiD | Callaway-Sant'Anna estimator |
| Feature rolled out to all users simultaneously | ITS or pre/post with CUPED | No untreated group exists |
| Geographic rollout (few regions treated) | Synthetic Control or Geo-DiD | Region-level comparison |

### Decision Tree

```
Do you have a CONTROL GROUP?
├── YES
│   ├── Treatment at same time → Standard DiD
│   ├── Treatment at different times → Staggered DiD
│   └── Only one treated unit → Synthetic Control
│
└── NO
    ├── Is there a threshold/cutoff? → RDD
    └── No cutoff → Interrupted Time Series
```

---

## Real-World Examples {#real-examples}

### Example 1: Minimum Wage Study (Card & Krueger, 1994)

```
Setting: New Jersey raised minimum wage; Pennsylvania did not
Design: DiD

Treatment: NJ fast-food restaurants (post wage increase)
Control: PA fast-food restaurants (same time period)

DiD = (NJ employment_after - NJ employment_before) 
    - (PA employment_after - PA employment_before)

Finding: No significant negative effect on employment
Key assumption: PA trend = NJ counterfactual trend (parallel trends)
```

### Example 2: Feature Rollout to Specific Markets

```
Setting: New recommendation algorithm launched in 5 markets, 15 remain unchanged
Design: Staggered DiD with Synthetic Control

Pre-period: 8 weeks before rollout
Post-period: 4 weeks after each market's rollout

Steps:
1. Build synthetic control for each treated market from donor pool
2. Estimate market-level effects
3. Aggregate for overall estimate
4. Placebo tests: apply to donor markets for inference
```

### Example 3: Policy Change with Eligibility Threshold

```
Setting: Users with credit score ≥ 700 get a new credit limit
Design: RDD

Running variable: Credit score
Cutoff: 700
Outcome: Spending in next 6 months

RDD estimate: Compare spending of users at score 699 vs 701
  → Local causal effect of higher credit limit on spending
  → Cannot generalize to users at score 500 or 800
```

### Example 4: COVID Lockdown Impact on Rideshare

```
Setting: Sudden lockdown, no untreated comparison
Design: Interrupted Time Series

Pre-period: 12 months of daily ride data
Intervention: Lockdown date
Post-period: 6 months after

Model: rides_t = β₀ + β₁×t + β₂×lockdown + β₃×t_after + seasonal_dummies + ε
  β₂: immediate drop in rides at lockdown
  β₃: gradual recovery rate (change in trend)
```

> **Critical Insight** 💡
> In industry, the most common quasi-experimental scenario is a feature launched to a subset of markets or user segments without randomization. DiD with synthetic control is the go-to approach. The key challenge is always defending the parallel trends assumption — invest time in pre-trend analysis and consider placebo tests.

---

## Interview Talking Points {#interview-talking-points}

> **"Explain difference-in-differences in simple terms."**
>
> "DiD compares how much the treatment group changed versus how much the control group changed over the same period. The control group's change represents what would have happened to the treatment group WITHOUT the intervention — it captures common time trends. The difference between the two changes isolates the treatment effect. It's like asking: 'How much of the treatment group's change can we attribute to the treatment versus what would have happened anyway?'"

> **"What's the parallel trends assumption and how do you test it?"**
>
> "Parallel trends means that absent the treatment, both groups would have followed the same trajectory. I test this by examining pre-treatment data: if trends were parallel for multiple periods before treatment, it's more plausible they would have continued parallel. I use event study plots — regressing on leads and lags of treatment — where pre-treatment coefficients should be close to zero. But I always caveat: parallel pre-trends are necessary but not sufficient. Something could have changed at exactly the treatment time that differentially affected the treatment group."

> **"When would you use synthetic control instead of standard DiD?"**
>
> "Synthetic control excels when I have very few treated units — especially just one (e.g., one country adopted a policy). Standard DiD requires the control group to be a good counterfactual, but with few treated units it's hard to find a single comparison unit. Synthetic control creates a weighted combination of multiple donors that matches the treated unit's pre-treatment trajectory, giving a better counterfactual. It also makes the weights transparent — I can show exactly how the counterfactual was constructed."

> **"A feature was launched to all users without a holdout. Can you still estimate the causal effect?"**
>
> "Yes, but with weaker assumptions. I'd use interrupted time series: model the pre-launch trend and extrapolate what WOULD have happened, then compare to what actually happened. I'd check for seasonality, autocorrelation, and concurrent events. If there are similar products or markets that didn't get the feature, I'd combine ITS with a control series (comparative ITS) for a stronger design. But I'd be transparent that without a proper control group, the estimate relies heavily on the assumption that the pre-trend would have continued unchanged."

---

## Common Mistakes {#common-mistakes}

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Claiming parallel trends are "proven" by pre-trend data | Pre-trends support plausibility but can't prove the assumption |
| Using two-way fixed effects with staggered treatment naively | Use modern staggered DiD estimators (Callaway-Sant'Anna) |
| Ignoring autocorrelation in DiD standard errors | Cluster standard errors at the unit level |
| Synthetic control with poor pre-treatment fit | Report fit quality; if RMSPE is large, results are unreliable |
| ITS without addressing seasonality | Include seasonal dummies or use seasonal differencing |
| RDD with too-wide bandwidth | Use MSE-optimal bandwidth; show sensitivity |
| Extrapolating RDD beyond the cutoff neighborhood | RDD is LOCAL; state this limitation explicitly |
| Not checking for manipulation in RDD | Always run McCrary density test |
| Ignoring concurrent events in ITS | Discuss threats; use control series if possible |
| Choosing control group based on post-treatment outcomes | Select controls based on pre-treatment characteristics only |

---

## Rapid-Fire Q&A {#rapid-fire-qa}

**Q1: What does DiD control for that a simple pre-post comparison doesn't?**
> DiD controls for common time trends (things that changed for everyone). Pre-post comparison confounds the treatment effect with other temporal changes (seasonality, economic shifts).

**Q2: What happens if parallel trends is violated?**
> The DiD estimate is biased. If the treatment group was already growing faster, DiD will overestimate the effect. If growing slower, it will underestimate.

**Q3: How do you handle staggered treatment timing in DiD?**
> Use modern estimators (Callaway & Sant'Anna, Sun & Abraham) that avoid the "negative weights" problem of two-way fixed effects. These estimate group-time effects separately and aggregate properly.

**Q4: What is a synthetic control and when is it better than DiD?**
> A weighted combination of untreated units that matches the treated unit pre-treatment. Better when you have few treated units (especially one) and need a data-driven counterfactual rather than assuming any single unit is a good control.

**Q5: How do you do inference with synthetic control?**
> Placebo tests: apply the synthetic control method to each untreated unit, compute their "gaps." If the treated unit's gap is extreme relative to the distribution of placebo gaps, the effect is statistically notable.

**Q6: What are the key assumptions of Interrupted Time Series?**
> (1) The pre-intervention trend would have continued absent intervention. (2) No other event occurred at the intervention time. (3) The intervention was sudden (not gradual). (4) Composition of the observed group doesn't change.

**Q7: What's the difference between sharp and fuzzy RDD?**
> Sharp: crossing the cutoff determines treatment perfectly (T=1 if X>=c). Fuzzy: crossing increases treatment probability but some don't comply. Fuzzy RDD uses the cutoff as an instrument and gives LATE for compliers.

**Q8: How do you choose the bandwidth in RDD?**
> Use MSE-optimal or CER-optimal bandwidth (e.g., Imbens-Kalyanaraman or Calonico-Cattaneo-Titiunik methods). Always show robustness to different bandwidths.

**Q9: Can you combine DiD with synthetic control?**
> Yes. Augmented Synthetic Control Method (ASCM) or Synthetic DiD combines the strengths of both — uses synthetic weights AND time-period re-weighting for doubly-robust estimates.

**Q10: What's the biggest practical challenge with quasi-experiments in industry?**
> Finding or defending the identifying assumption. For DiD it's parallel trends; for ITS it's "nothing else changed"; for RDD it's "no manipulation." These require deep domain knowledge and are ultimately unprovable — the best you can do is present evidence and be transparent about limitations.

---

## ASCII Cheat Sheet {#cheat-sheet}

```
╔══════════════════════════════════════════════════════════════════════╗
║         DIFF-IN-DIFF & QUASI-EXPERIMENTS CHEAT SHEET                ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  DIFFERENCE-IN-DIFFERENCES (DiD):                                    ║
║  DiD = (Y_T,post - Y_T,pre) - (Y_C,post - Y_C,pre)                ║
║                                                                      ║
║  Regression: Y = β₀ + β₁×Treat + β₂×Post + β₃×(Treat×Post) + ε   ║
║              β₃ = DiD estimate (causal effect)                      ║
║                                                                      ║
║  Key Assumption: PARALLEL TRENDS                                     ║
║  • Cannot prove — can only support with pre-trend evidence          ║
║  • Event study: pre-treatment coefficients ≈ 0                      ║
║  • Violation → biased estimate                                      ║
║                                                                      ║
║  SYNTHETIC CONTROL:                                                  ║
║  • Best for: 1 treated unit, many potential donors                  ║
║  • Creates weighted counterfactual from donor pool                  ║
║  • Inference: placebo tests across donors                           ║
║  • Requires: good pre-treatment fit                                 ║
║                                                                      ║
║  INTERRUPTED TIME SERIES (ITS):                                      ║
║  • Best for: no control group, long pre-treatment series            ║
║  • Y_t = β₀ + β₁×t + β₂×intervention + β₃×t_after + ε            ║
║  • β₂: level shift; β₃: slope change                               ║
║  • Threats: concurrent events, seasonality                          ║
║                                                                      ║
║  REGRESSION DISCONTINUITY (RDD):                                     ║
║  • Best for: treatment assigned by threshold on running variable    ║
║  • Sharp: T = 1(X≥c) perfectly                                     ║
║  • Fuzzy: P(T) jumps at c (use as IV)                              ║
║  • Checks: McCrary density, covariate balance, bandwidth robustness║
║  • Limitation: LOCAL effect at cutoff only                          ║
║                                                                      ║
║  METHOD SELECTION:                                                   ║
║  ┌────────────────────────────────────────────────────────┐         ║
║  │ Have control group?                                     │         ║
║  │ ├─YES─→ DiD (or Synthetic Control if few treated)      │         ║
║  │ └─NO──→ Threshold exists? → RDD                        │         ║
║  │         No threshold? → ITS                             │         ║
║  └────────────────────────────────────────────────────────┘         ║
║                                                                      ║
║  STAGGERED DiD WARNING:                                              ║
║  • Standard TWFE is BIASED with heterogeneous effects               ║
║  • Use: Callaway-Sant'Anna, Sun-Abraham, de Chaisemartin            ║
║  • Always check for negative weights in TWFE                        ║
║                                                                      ║
║  VALIDITY CHECKLIST:                                                 ║
║  □ DiD: Pre-trends parallel? Event study coefficients ≈ 0?         ║
║  □ Synthetic: Pre-treatment fit good? Placebo gaps smaller?         ║
║  □ ITS: Concurrent events ruled out? Seasonality modeled?           ║
║  □ RDD: No bunching at cutoff? Covariates smooth? Bandwidth robust?║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 27 of 45*
