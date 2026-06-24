# 🎯 Topic 57: Marketplace & Two-Sided Experiment Design

> *"In a marketplace, treating one side always spills over to the other — standard A/B testing is fundamentally broken here."*

---

## Table of Contents

1. [Why Standard A/B Fails in Marketplaces](#why-standard-ab-fails)
2. [Interference and Spillover Effects](#interference-and-spillover)
3. [Switchback Experiments](#switchback-experiments)
4. [Cluster and Geo Randomization](#cluster-and-geo-randomization)
5. [Supply-Side vs Demand-Side Experiments](#supply-vs-demand-experiments)
6. [Cannibalization and Displacement Effects](#cannibalization-and-displacement)
7. [Marketplace-Specific Metrics](#marketplace-specific-metrics)
8. [Designing Experiments for Marketplace Features](#designing-experiments-for-features)
9. [Industry Case Studies](#industry-case-studies)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Why Standard A/B Fails in Marketplaces {#why-standard-ab-fails}

Standard A/B assumes **SUTVA** (Stable Unit Treatment Value Assumption) — one user's treatment doesn't affect another. In marketplaces, SUTVA is violated because supply is shared:

| Scenario | Why SUTVA Breaks |
|----------|-----------------|
| Rider gets new matching algo | Drivers pulled away from control riders |
| Restaurant promoted | Other restaurants lose orders (zero-sum) |
| Dasher gets batching | Nearby dashers get fewer orders |
| Host gets dynamic pricing | Neighboring hosts lose bookings |

> **Critical Insight:** When treatment users get better matching, you simultaneously *worsen* control outcomes by consuming finite supply. The observed treatment effect is **doubled** — half real uplift, half stolen from control.

### The Math of Interference

```
Observed TE = True TE + Spillover bias

If treatment "steals" from control:
  Observed TE = True TE + |Negative spillover on control|
  => OVERESTIMATES the true global effect

If treatment creates positive externalities:
  Observed TE = True TE - |Positive spillover to control|
  => UNDERESTIMATES the true global effect
```

### When Standard A/B Still Works

- Pure UI changes with no supply interaction (button color on consumer app)
- Features that don't affect matching/allocation (notification copy)
- Demand-side changes where supply is effectively unlimited
- Information-only features (estimated arrival display, restaurant hours)

---

## Interference and Spillover Effects {#interference-and-spillover}

### Types of Interference

| Type | Definition | Example |
|------|-----------|---------|
| **Direct** | Treatment affects control directly | Better ETA for treated riders pulls drivers from control |
| **Indirect** | Propagates through shared network | Promoted restaurant's orders slow nearby deliveries |
| **Temporal** | Persists across time periods | Dasher who earned more yesterday logs on more today |
| **Geographic** | Bleeds across boundaries | Surge in Zone A pushes drivers to Zone B |

### Quantifying Interference with Ring Analysis

```python
import numpy as np

def estimate_interference_decay(outcomes, treatment, distance_matrix):
    """
    Compare control outcomes at different distances from treated units.
    If control near treatment is worse than distant control, interference exists.
    """
    rings = [0, 1, 2, 5, 10, 25]  # km boundaries
    pure_control_mask = (treatment == 0) & \
        (distance_matrix[:, treatment == 1].min(axis=1) > 25)
    baseline = outcomes[pure_control_mask].mean()
    
    for i in range(len(rings) - 1):
        inner, outer = rings[i], rings[i+1]
        ring_mask = (treatment == 0) & \
            (distance_matrix[:, treatment == 1].min(axis=1) >= inner) & \
            (distance_matrix[:, treatment == 1].min(axis=1) < outer)
        
        if ring_mask.sum() > 0:
            spillover = outcomes[ring_mask].mean() - baseline
            print(f"Ring {inner}-{outer}km: spillover = {spillover:.4f} "
                  f"(n={ring_mask.sum()})")
```

> **Critical Insight:** DoorDash found that interference effects extend 5-10km in dense urban areas. A "clean" control group must be geographically separated by at least this distance, which is why geo-randomization uses entire cities or regions.

---

## Switchback Experiments {#switchback-experiments}

Switchback (time-based randomization) alternates the entire market between treatment and control over time intervals. Everyone in the market gets the same condition at any given moment.

```
Time: |--T--|--W--|--C--|--W--|--T--|--W--|--C--|
       30min 15m  30min 15m  30min 15m  30min

T = Treatment period, C = Control period, W = Washout (discard)
```

### Design Parameters

| Parameter | Guidance | Rationale |
|-----------|----------|-----------|
| Interval length | 15-60 minutes | Long enough for stabilization, short for many switches |
| Washout period | 10-15 minutes | Clears carryover from previous period |
| Minimum switches | 50+ per condition | Statistical power requirement |
| Stratification | Hour-of-day, day-of-week | Controls temporal confounders |
| Duration | 2+ weeks | Captures full weekly cycle |

### When to Use Switchback

- **Best for:** Dispatch/matching algorithms, pricing changes, batching logic, supply allocation
- **Bad for:** Features requiring user learning, long-horizon effects, UI changes

### Statistical Analysis

```python
from statsmodels.regression.mixed_linear_model import MixedLM

def analyze_switchback(df):
    """
    Mixed effects model: treatment + time controls (fixed),
    market-level variation (random).
    df needs: period_id, treatment, outcome, hour_of_day, day_of_week, market_id
    """
    formula = "outcome ~ treatment + C(hour_of_day) + C(day_of_week)"
    model = MixedLM.from_formula(
        formula, data=df,
        groups=df["market_id"],
        re_formula="~treatment"
    )
    result = model.fit()
    te = result.params['treatment']
    ci = result.conf_int().loc['treatment']
    print(f"Treatment Effect: {te:.4f}, 95% CI: [{ci[0]:.4f}, {ci[1]:.4f}]")
    return result
```

> **Critical Insight:** The biggest pitfall is **carryover effects**. Drivers don't instantly reposition when you switch back to control. DoorDash uses 15-minute washout periods and discards all data from those windows. Test for carryover by comparing outcomes at period-start vs period-end.

---

## Cluster and Geo Randomization {#cluster-and-geo-randomization}

Geographic randomization creates physically separated treatment and control groups, eliminating within-market interference entirely.

### Approaches Compared

| Method | Unit | Pros | Cons |
|--------|------|------|------|
| **City-level** | Entire cities | Zero interference | Very few units, low power |
| **Region-level** | Sub-city zones | More units | Cross-boundary leakage |
| **Synthetic control** | Market + matching | Handles heterogeneity | Needs pre-period data |
| **Paired markets** | Matched pairs | Controls for size | Still limited N |

### Power Boosting Strategies

With few clusters, standard approaches lack power. Mitigations:

1. **Paired randomization** — match cities by size/density, randomize within pairs
2. **Difference-in-differences** — leverage pre-period trends for variance reduction
3. **Synthetic control** — construct counterfactual from weighted donor pool
4. **CUPED** — use pre-treatment covariates to reduce residual variance
5. **Extended duration** — more time periods increase effective sample

```python
def geo_experiment_power(n_markets, effect_size, icc, alpha=0.05):
    """Power for geo-randomized experiment."""
    from scipy import stats
    avg_cluster_size = 10000  # orders per market per period
    design_effect = 1 + (avg_cluster_size - 1) * icc
    n_effective = (n_markets * avg_cluster_size) / design_effect
    se = 1.0 / np.sqrt(n_effective / 2)
    z_alpha = stats.norm.ppf(1 - alpha/2)
    power = stats.norm.cdf((effect_size / se) - z_alpha)
    print(f"Design effect: {design_effect:.1f}, Power: {power:.3f}")
    return power
```

---

## Supply-Side vs Demand-Side Experiments {#supply-vs-demand-experiments}

### Decision Framework

```
Does the feature change WHO gets matched or HOW supply is allocated?
  YES -> Supply-side experiment (need geo/switchback)
  NO  -> Does it affect behavior that impacts supply utilization?
           YES -> Hybrid (user-level A/B + supply guardrails)
           NO  -> Standard user-level A/B is fine
```

### Supply-Side Experiments

| Feature | Randomization | Why |
|---------|--------------|-----|
| Driver matching algo | Switchback or geo | Changes allocation for ALL riders |
| Dasher pay structure | Dasher-level (careful) | Affects behavior, spillover risk |
| Surge pricing | Geo or switchback | Affects all supply/demand in area |
| Batching algorithm | Switchback | Changes which orders grouped |
| Delivery radius | Geo | Alters supply pool for an area |

### Demand-Side Experiments

| Feature | Randomization | Key Consideration |
|---------|--------------|-------------------|
| App UI/UX | User-level | Safe if no supply interaction |
| Search ranking | User-level | Watch for supply constraints |
| Promo/coupon | User-level | Monitor demand spikes |
| Notification copy | User-level | Generally safe |
| Subscription pricing | User-level | Watch for word-of-mouth spillover |

> **Critical Insight:** Even demand-side experiments can have supply spillovers. If your coupon experiment increases demand by 20%, you may exhaust local supply, degrading experience for everyone. Always monitor fill rate and supply utilization as guardrail metrics.

---

## Cannibalization and Displacement Effects {#cannibalization-and-displacement}

### Cannibalization

When a feature shifts existing transactions rather than creating new ones:

```
Naive analysis: "Promoted restaurants got +30% orders!"
Reality: Total marketplace orders unchanged
Truth: Feature just moved orders between restaurants (zero-sum)
```

### Displacement

When improving treatment outcomes comes at direct expense of control:

```
100 drivers, 100 riders (fixed supply)
Treatment (50 riders): Better matching -> 3 min wait
Control (50 riders):   Worse matching  -> 7 min wait (drivers stolen)

Observed difference: 4 min (INFLATED)
True global effect:  avg wait 5 min -> 4 min (only 1 min improvement)
```

### Detection Methods

```python
def detect_cannibalization(treatment_markets, control_markets, metric):
    """
    Compare total volume in treatment vs control MARKETS (not users).
    If treatment-market total is same as control-market total,
    then per-unit gains are pure cannibalization.
    """
    treat_total = treatment_markets[metric].sum()
    ctrl_total = control_markets[metric].sum()
    market_lift = (treat_total - ctrl_total) / ctrl_total
    
    # Compare to per-unit lift
    treat_promoted = treatment_markets[treatment_markets['promoted']][metric].mean()
    ctrl_baseline = control_markets[metric].mean()
    unit_lift = (treat_promoted - ctrl_baseline) / ctrl_baseline
    
    cannibalization_pct = max(0, 1 - market_lift / unit_lift) * 100
    print(f"Unit-level lift: {unit_lift*100:.1f}%")
    print(f"Market-level lift: {market_lift*100:.1f}%")
    print(f"Cannibalization: {cannibalization_pct:.0f}% of unit lift is redistributed")
    return cannibalization_pct
```

---

## Marketplace-Specific Metrics {#marketplace-specific-metrics}

### Core Health Metrics

| Metric | Formula | Measures |
|--------|---------|----------|
| **Fill Rate** | Orders filled / Orders requested | Supply adequacy |
| **ETA Accuracy** | abs(Actual - Predicted) / Predicted | Prediction quality |
| **Utilization** | Active time / Online time | Supply efficiency |
| **Match Rate** | Successful matches / Attempts | Matching effectiveness |
| **Cancel Rate** | Cancellations / Orders placed | Experience quality |
| **Defect Rate** | (Late + Wrong + Missing) / Delivered | Fulfillment quality |

### The Marketplace Tension Triangle

```
        Fill Rate
       /         \
      /  TENSION   \
     /              \
ETA Accuracy ----- Utilization

Improving one often degrades another:
- Higher utilization -> Longer ETAs (drivers already busy)
- Better ETAs -> Lower utilization (need idle supply buffer)
- Higher fill rate -> More supply needed -> Lower utilization
```

> **Critical Insight:** Always define a **primary metric** AND **guardrail metrics** for marketplace experiments. A dispatch change might improve fill rate by 5% but degrade ETA by 20%. DoorDash uses a composite "consumer experience score" combining multiple signals.

---

## Designing Experiments for Marketplace Features {#designing-experiments-for-features}

### Case 1: Order Batching Algorithm

- **Randomization:** Switchback (30-min intervals, 15-min washout)
- **Why not user-level?** Batching decisions involve multiple users simultaneously
- **Primary metric:** Delivery time per order
- **Guardrails:** Consumer satisfaction, dasher pay/hour, food quality
- **Duration:** 2+ weeks, stratify peak vs off-peak

### Case 2: Thermal Bag Requirement

- **Randomization:** Geo-level (some markets require, some don't)
- **Why?** Reduces supply pool, affecting all consumers in market
- **Primary metric:** Food quality rating, reorder rate
- **Guardrails:** Fill rate, ETA, dasher churn
- **Duration:** 4-6 weeks (observe supply adjustment)

### Case 3: Bicycle Dashers in Urban Areas

- **Randomization:** Geo-level (specific zones within city)
- **Why?** Bicycles compete with cars for same orders
- **Primary metric:** Short-distance delivery time
- **Guardrails:** Overall fill rate, earnings equity, weather robustness

### Case 4: Store Recommendations

- **Randomization:** User-level (safe: recommendation doesn't change allocation)
- **Primary metric:** Conversion rate, order frequency
- **Guardrails:** AOV (not shifting to cheaper options), cannibalization of preferred stores
- **Watch for:** If recommendations become too aggressive, may cannibalize

---

## Industry Case Studies {#industry-case-studies}

### DoorDash: Dispatch Algorithm

- Switchback experiments for all dispatch changes
- 20-minute treatment/control intervals, 10-minute washout
- Cluster by metro area, run 2+ weeks
- Key learning: carryover effects require careful washout design

### Uber: Surge Pricing

- Cannot A/B test prices (trust violation if riders see different prices)
- City-level randomization with synthetic control
- Match treatment cities to donor cities on pre-period trends
- Need 10+ pre-treatment periods for good synthetic control fit

### Airbnb: Search Ranking

- Fractional exposure: listing's treatment status = fraction of treated searchers viewing it
- Tracks listing-level exposure intensity
- Adjusts treatment effects for partial exposure
- Handles two-sided interference without full geo-randomization

### Instacart: Shopper Experiments

- Paired market design (match on size, density, demand patterns)
- Randomize within pairs for variance reduction
- Account for shopper mobility (shoppers working multiple zones)
- Longer experiments needed due to shopper behavior adaptation

---

## Interview Talking Points {#interview-talking-points}

> **"Why can't we just A/B test our new dispatch algorithm?"**
>
> "Standard A/B violates SUTVA in marketplaces because supply is shared. If I give treatment riders better matching, those riders consume the same finite driver pool — actively degrading outcomes for control riders. The observed effect is inflated because it includes both genuine improvement AND harm to control. Instead, use switchback experiments where the entire market alternates between treatment and control over time intervals, giving a clean estimate of the global effect."

> **"How would you design an experiment for a new batching feature?"**
>
> "Batching inherently involves multiple users — you can't batch one user's order without affecting another's delivery time. I'd use switchback with 30-minute intervals, 15-minute washout periods. Primary metric: average delivery latency. Guardrails: consumer satisfaction, dasher hourly earnings. Run for 2+ weeks to capture weekly patterns, stratify by peak vs off-peak, analyze with mixed-effects model."

> **"Your geo experiment has only 20 cities. How do you get power?"**
>
> "With few clusters: (1) paired randomization — match cities by size, randomize within pairs; (2) DiD with pre-period data; (3) synthetic control for better counterfactuals; (4) extend duration for more time periods; (5) CUPED variance reduction. Run power calculations upfront assuming ICC 0.05-0.10 to set realistic detectable effect expectations."

---

## Common Mistakes {#common-mistakes}

| # | Mistake (❌) | Correct Approach (✅) |
|---|-------------|----------------------|
| 1 | ❌ A/B test dispatch changes at user level | ✅ Use switchback or geo-randomization |
| 2 | ❌ Ignore washout periods in switchback | ✅ Include 10-15 min washout, discard that data |
| 3 | ❌ Measure only treatment-group metrics | ✅ Measure total marketplace health (both sides) |
| 4 | ❌ Conclude "feature adds X orders" without checking cannibalization | ✅ Check if total market volume increased or just shifted |
| 5 | ❌ Use individual-level standard errors in geo experiments | ✅ Cluster standard errors at the randomization unit level |
| 6 | ❌ Run geo experiment for 3 days | ✅ Run 2-4 weeks minimum for weekly cycle coverage |
| 7 | ❌ Optimize one metric in isolation | ✅ Define primary metric + guardrail metrics together |
| 8 | ❌ Assume supply is unlimited | ✅ Monitor fill rate and supply utilization as guardrails |
| 9 | ❌ Treat all markets as identical | ✅ Stratify by market maturity, density, and size |
| 10 | ❌ Ignore carryover effects in switchback | ✅ Model temporal dependencies and test for carryover |

---

## Rapid-Fire Q&A {#rapid-fire-qa}

**Q1: What is SUTVA and why does it fail in marketplaces?**
> SUTVA = one unit's treatment doesn't affect others. Fails because supply is shared — treating one user's experience necessarily affects others competing for the same pool.

**Q2: When switchback vs geo-randomization?**
> Switchback: short-lived effects, single-market changes (dispatch, pricing). Geo: long-lived effects, features needing sustained exposure, when temporal switching has too much carryover.

**Q3: Minimum number of switchbacks?**
> At least 50 periods per condition for adequate power, ideally 100+. Each period 15-60 min.

**Q4: How detect interference in a running experiment?**
> Compare control outcomes to pre-experiment baselines. Ring analysis: control units near treatment should show worse outcomes than distant controls if interference exists.

**Q5: What's a displacement effect?**
> Treatment improves one group by taking resources from another, inflating the observed difference beyond the true global improvement.

**Q6: How does Airbnb handle search ranking experiments?**
> Fractional exposure: each listing's treatment status determined by fraction of treated vs control searchers who view it. Weight outcomes by exposure intensity.

**Q7: Why paired market design?**
> Reduces variance by matching similar markets and randomizing within pairs. Controls for market-level confounders with limited cluster count.

**Q8: Cannibalization test for a promotional feature?**
> Compare total marketplace GMV (not just promoted items). If promoted gains but total flat = redistribution, not growth.

**Q9: How handle carryover in switchback?**
> Washout periods (discard data), test for carryover (compare start vs end of period outcomes), use longer intervals if carryover detected.

**Q10: Three guardrail metrics for DoorDash?**
> Dasher hourly earnings (supply side), consumer delivery satisfaction (demand side), fill rate (marketplace health).

---

## ASCII Cheat Sheet {#ascii-cheat-sheet}

```
╔════════════════════════════════════════════════════════════╗
║     MARKETPLACE EXPERIMENT DESIGN DECISION TREE           ║
╠════════════════════════════════════════════════════════════╣
║                                                            ║
║  Does feature change supply allocation?                    ║
║    ├─ YES → Effect short-lived (<1hr)?                     ║
║    │         ├─ YES → SWITCHBACK EXPERIMENT                ║
║    │         └─ NO  → GEO RANDOMIZATION                    ║
║    └─ NO  → Demand in supply-constrained market?           ║
║              ├─ YES → USER A/B + SUPPLY GUARDRAILS         ║
║              └─ NO  → STANDARD USER A/B                    ║
║                                                            ║
║  SWITCHBACK CHECKLIST:                                     ║
║  [ ] Interval: 15-60 min                                   ║
║  [ ] Washout: 10-15 min (discard this data)                ║
║  [ ] Periods: 50+ per condition                            ║
║  [ ] Duration: 2+ weeks                                    ║
║  [ ] Analysis: mixed effects with time FE                  ║
║  [ ] Carryover test: start vs end of period                ║
║                                                            ║
║  GEO RANDOMIZATION CHECKLIST:                              ║
║  [ ] Markets: 10+ per arm (ideally 20+)                    ║
║  [ ] Pair by size, density, trends                         ║
║  [ ] Pre-period: 4+ weeks for synthetic control            ║
║  [ ] Cluster SEs at market level                           ║
║  [ ] DiD or synthetic control estimation                   ║
║                                                            ║
║  KEY FORMULAS:                                             ║
║  Observed TE = True TE + Spillover bias                    ║
║  Design Effect = 1 + (m-1) * ICC                           ║
║  Effective N = Total N / Design Effect                     ║
║  Cannibalization = (Unit lift - Market lift) / Unit lift    ║
║                                                            ║
║  INTERFERENCE DETECTION:                                   ║
║  [ ] Control vs pre-experiment baseline                    ║
║  [ ] Ring analysis by distance from treatment              ║
║  [ ] Total marketplace GMV (not just treatment group)      ║
║                                                            ║
╚════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 57 of 60*
