# 🎯 Topic 28: Novelty Effects, Network Effects, and Interference

> *"The hardest experiments are those where individual outcomes depend on what happens to everyone else."*

---

## Table of Contents

1. [Novelty Effects](#novelty-effects)
2. [Primacy Effects](#primacy-effects)
3. [Detecting Temporal Effect Changes](#detecting-temporal-changes)
4. [Network Effects and SUTVA Violation](#network-effects-sutva)
5. [Cluster Randomization](#cluster-randomization)
6. [Switchback Experiments](#switchback-experiments)
7. [Ego-Network Randomization](#ego-network-randomization)
8. [Marketplace Interference](#marketplace-interference)
9. [Design Strategies for Interference](#design-strategies)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#cheat-sheet)

---

## Novelty Effects {#novelty-effects}

### What Are Novelty Effects?

Users engage MORE with a new feature simply because it is new, not because it provides lasting value. The initial lift fades as the novelty wears off.

```
Engagement
    │     ●
    │    ● ●  ← Novelty spike
    │   ●   ●
    │  ●     ●
    │ ●       ● ● ← True steady-state effect
    │●           ● ● ● ● ● ● ●
    │
    │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ← Control baseline
    └──────────────────────────────→ Time
      ↑                      ↑
   Launch              Novelty
                       worn off
```

### Why Novelty Effects Are Dangerous

| Issue | Consequence |
|-------|-------------|
| Over-estimated treatment effect | Ship a feature that doesn't actually help long-term |
| Premature test termination | Stop test during novelty peak → inflated results |
| Misleading post-launch metrics | Initial success followed by decay → confusion |
| Wrong resource allocation | Invest in "successful" feature that plateaus |

### Common Scenarios

| Product Change | Novelty Mechanism |
|---------------|-------------------|
| New UI element (button, badge) | Users explore unfamiliar element out of curiosity |
| New notification type | Users open novel notifications; fatigue sets in later |
| Redesigned feed | Users scroll more to understand new layout |
| New content format (Stories, Reels) | Early adopter enthusiasm; may not sustain |
| Gamification (badges, streaks) | Initial excitement; eventual habituation |

> **Critical Insight** 💡
> The most insidious novelty effects are those that look like genuine engagement improvements for 2-3 weeks. A test running for "just two weeks" can easily be dominated by novelty. Always ask: "Would this user still engage at this level in month 3?" The only way to know is to run the test longer or analyze cohort-level decay.

---

## Primacy Effects {#primacy-effects}

### What Are Primacy Effects?

Users engage LESS with a change because they resist departing from their established habits. The negative reaction fades as users adapt.

```
Engagement
    │
    │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ← Control baseline
    │                   ● ● ● ● ● ← True steady-state (better!)
    │               ● ●
    │            ●●
    │         ●●  ← Adaptation period
    │       ●
    │    ● ●   ← Primacy dip (user resistance)
    │   ●
    └──────────────────────────────→ Time
      ↑                      ↑
   Launch              Users
                       adapted
```

### Novelty vs. Primacy: Opposite Problems

| Dimension | Novelty Effect | Primacy Effect |
|-----------|---------------|----------------|
| Direction | Inflates positive result | Suppresses positive result |
| Risk | Ship bad feature | Kill good feature |
| Mechanism | Curiosity about new | Resistance to change |
| Who's affected | All users equally | Power users most (strong habits) |
| Recovery | Effect decays to true level | Effect rises to true level |
| Time to stabilize | 1-4 weeks typically | 2-6 weeks typically |

### When to Suspect Primacy Effects

- Major UI redesigns (menu reorganization, navigation changes)
- Removing or relocating established features
- Changing default settings that users rely on
- Any change that disrupts muscle memory

> **Critical Insight** 💡
> Primacy effects are UNDERRATED in industry. Many potentially good features are killed because a 2-week test showed a negative result. The team never ran it long enough to see users adapt. If your change requires learning (new location, new workflow), expect a primacy dip and design your test to capture the full adaptation curve.

---

## Detecting Temporal Effect Changes {#detecting-temporal-changes}

### Time-Series Analysis of Treatment Effect

```python
import pandas as pd
import numpy as np
import statsmodels.api as sm

def analyze_temporal_effect(df, metric_col, group_col, time_col):
    """
    Analyze how treatment effect changes over time since exposure.
    """
    # Calculate daily treatment effect
    daily_effects = []
    for day in sorted(df[time_col].unique()):
        day_data = df[df[time_col] == day]
        treatment = day_data[day_data[group_col] == 'treatment'][metric_col].mean()
        control = day_data[day_data[group_col] == 'control'][metric_col].mean()
        effect = treatment - control
        daily_effects.append({'day': day, 'effect': effect})
    
    effects_df = pd.DataFrame(daily_effects)
    
    # Test for trend in effects (is effect changing over time?)
    X = sm.add_constant(effects_df['day'])
    model = sm.OLS(effects_df['effect'], X).fit()
    
    # Negative slope → novelty wearing off (or primacy recovering)
    # Positive slope → primacy recovering (or delayed effect)
    trend_pvalue = model.pvalues[1]
    trend_direction = "declining" if model.params[1] < 0 else "increasing"
    
    return effects_df, trend_direction, trend_pvalue
```

### Cohort-Based Detection

```
Method: Compare users by TENURE in experiment

Cohort 1: Users in experiment for 1 week  → effect = 15%
Cohort 2: Users in experiment for 2 weeks → effect = 12%
Cohort 3: Users in experiment for 3 weeks → effect = 8%
Cohort 4: Users in experiment for 4 weeks → effect = 7%

Declining across cohorts → NOVELTY EFFECT present
(Each cohort has longer exposure; effect diminishes with exposure time)
```

### Time-Series of Treatment Effect Plot

```python
import matplotlib.pyplot as plt

def plot_treatment_effect_over_time(effects_df):
    """Plot treatment effect with confidence intervals over time."""
    fig, ax = plt.subplots(figsize=(12, 6))
    
    ax.plot(effects_df['day'], effects_df['effect'], marker='o', linewidth=2)
    ax.axhline(y=0, color='red', linestyle='--', alpha=0.5)
    
    # Add trend line
    z = np.polyfit(effects_df['day'], effects_df['effect'], 1)
    p = np.poly1d(z)
    ax.plot(effects_df['day'], p(effects_df['day']), 
            color='orange', linestyle='--', label=f'Trend: {z[0]:.4f}/day')
    
    ax.set_xlabel('Days Since Experiment Start')
    ax.set_ylabel('Treatment Effect (Treatment - Control)')
    ax.set_title('Treatment Effect Over Time (Check for Novelty/Primacy)')
    ax.legend()
    plt.show()
```

### Decision Framework

```
Is the treatment effect CHANGING over time?
│
├── Declining (novelty suspected):
│   ├── Has it stabilized? → Use stabilized level as true effect
│   ├── Still declining at test end? → Extend test duration
│   └── Converged to ≈ 0? → Feature has no lasting value; don't ship
│
├── Increasing (primacy suspected):
│   ├── Has it stabilized at positive level? → Ship with confidence
│   ├── Still increasing at test end? → Extend test
│   └── Unclear trajectory? → Run longer or use holdout group for post-ship validation
│
└── Stable (no temporal effects):
    └── Use standard analysis → straightforward interpretation
```

---

## Network Effects and SUTVA Violation {#network-effects-sutva}

### SUTVA Recap

**S**table **U**nit **T**reatment **V**alue **A**ssumption:
1. A unit's outcome depends ONLY on its own treatment assignment
2. There is no interference between units

### When SUTVA Is Violated

```
Standard assumption:
  Y_i = f(T_i)  → My outcome depends only on MY treatment

Reality with network effects:
  Y_i = f(T_i, T_{friends(i)})  → My outcome depends on MY treatment
                                     AND my friends' treatments

This means:
  - Control users affected by treated friends → control is "contaminated"
  - Treatment effect estimate is BIASED
  - Direction of bias depends on the mechanism
```

### Types of Interference

| Type | Mechanism | Example | Bias Direction |
|------|-----------|---------|----------------|
| Direct spillover | Treated user shares with control user | Social features, referrals | Underestimates effect |
| Displacement | Treatment takes from control | Marketplace competition | Overestimates effect |
| Social learning | Control learns from treatment | Product recommendations | Underestimates effect |
| Resource competition | Shared finite resource | Server capacity, driver pool | Overestimates effect |
| Network externalities | Value depends on adoption | Communication features | Underestimates effect |

### Quantifying Interference

```
True treatment effect = Direct effect + Indirect (spillover) effect

What standard A/B test estimates:
  τ_naive = E[Y_i | T_i=1] - E[Y_i | T_i=0]
  
  This conflates:
  • Direct effect of being treated
  • Indirect effect of having treated neighbors (which differs 
    between treatment and control groups)

Example: Viral feature
  Treatment user: direct benefit + spillover from treated friends
  Control user: no direct benefit + SOME spillover from treated friends
  
  Naive estimate < True total effect (because control is "helped" too)
```

> **Critical Insight** 💡
> The direction of bias from interference is NOT always the same. In social/viral settings, interference typically UNDERESTIMATES the true effect (control gets spillover benefits). In competitive/marketplace settings, interference typically OVERESTIMATES (control is hurt by treatment). Understanding the mechanism is crucial for determining whether your naive estimate is a lower or upper bound.

---

## Cluster Randomization {#cluster-randomization}

### The Core Idea

Instead of randomizing individuals, randomize GROUPS (clusters) where interference happens within groups but not between them.

```
Standard: Randomize individuals → interference contaminates estimate
Cluster:  Randomize whole groups → interference contained within clusters

Examples of clusters:
  • Geographic regions / cities
  • Schools / classrooms
  • Friend groups / connected components
  • Company teams / organizations
  • Time periods (switchback)
```

### Design and Analysis

```python
import numpy as np
from scipy import stats

def cluster_randomized_analysis(clusters_treatment, clusters_control):
    """
    Analyze cluster-randomized experiment.
    Unit of analysis = cluster, not individual.
    """
    # Aggregate to cluster level (e.g., cluster means)
    treatment_means = [np.mean(cluster) for cluster in clusters_treatment]
    control_means = [np.mean(cluster) for cluster in clusters_control]
    
    # Test at cluster level
    t_stat, p_value = stats.ttest_ind(treatment_means, control_means)
    
    effect = np.mean(treatment_means) - np.mean(control_means)
    
    # Effective sample size is NUMBER OF CLUSTERS, not individuals
    n_clusters = len(treatment_means) + len(control_means)
    
    return {
        'effect': effect,
        'p_value': p_value,
        'n_clusters': n_clusters,
        'n_individuals': sum(len(c) for c in clusters_treatment + clusters_control)
    }
```

### The Design Effect (Power Penalty)

```
Design Effect (DEFF) = 1 + (m - 1) × ICC

Where:
  m = average cluster size
  ICC = Intra-Cluster Correlation 
      = Var(between clusters) / Var(total)

Effective sample size = N_total / DEFF

Example:
  m = 100 people per cluster
  ICC = 0.05
  DEFF = 1 + (100-1) × 0.05 = 5.95
  
  1000 users in clusters ≈ 168 independent users in power!
```

### Power Implications

| ICC | Cluster Size | DEFF | Power Penalty |
|-----|-------------|------|---------------|
| 0.01 | 50 | 1.49 | Moderate |
| 0.01 | 500 | 5.99 | Severe |
| 0.05 | 50 | 3.45 | Severe |
| 0.05 | 500 | 25.95 | Extreme |
| 0.10 | 50 | 5.90 | Extreme |
| 0.10 | 100 | 10.9 | Nearly unusable |

> **Critical Insight** 💡
> Cluster randomization dramatically reduces statistical power. With ICC = 0.05 and 100 users per cluster, you need roughly 6x more clusters (= 6x more users) than an individual-level experiment. This is why teams invest heavily in variance reduction or choose designs that minimize cluster sizes (like ego-networks over cities).

---

## Switchback Experiments {#switchback-experiments}

### The Concept

Alternate between treatment and control at the SAME units over different time periods.

```
Time →  T1    T2    T3    T4    T5    T6    T7    T8
Market  Treat Ctrl  Treat Ctrl  Treat Ctrl  Treat Ctrl

OR (randomized order):
Market  Ctrl  Treat Ctrl  Ctrl  Treat Treat Ctrl  Treat
```

### Why Switchback?

| Advantage | Explanation |
|-----------|-------------|
| Handles interference | All units in same condition at same time |
| More power than geo | Same unit serves as own control |
| Works for marketplaces | Buyers and sellers in same condition simultaneously |
| Handles time-varying confounders | Randomized across time periods |

### When to Use Switchback

- **Marketplaces** (Uber, Lyft, DoorDash): Riders and drivers must be in same condition
- **Shared resources**: When treatment affects a shared pool
- **Platform-level features**: Features that affect ALL users simultaneously
- **Short-lived effects**: Where carryover between periods is minimal

### Design Considerations

```
Key decisions:
  1. Period length: Long enough for effect to manifest
                    Short enough to get many periods (power)
                    
  2. Washout period: Time between switches where data is excluded
                     Prevents carryover contamination
                     
  3. Randomization: Random assignment of time periods to treatment
                    Can block by hour-of-day, day-of-week

Common period lengths:
  • Uber/Lyft: 30 min - 2 hours
  • Food delivery: 1-3 hours
  • Search ranking: hours to days
  • Pricing: days to weeks
```

### Analysis

```python
def switchback_analysis(df, outcome_col, period_col, treatment_col):
    """
    Analyze switchback experiment with period fixed effects.
    """
    import statsmodels.formula.api as smf
    
    # Model: outcome ~ treatment + time_of_day + day_of_week
    model = smf.ols(
        f'{outcome_col} ~ {treatment_col} + C(hour_of_day) + C(day_of_week)',
        data=df
    ).fit(cov_type='HAC', cov_kwds={'maxlags': 2})  # Autocorrelation-robust SE
    
    return model

# Key: cluster standard errors at the PERIOD level, not individual level
# Account for autocorrelation between adjacent periods
```

### Carryover Effects

```
Problem: Effect from period t leaks into period t+1

If treatment in period 1 has lingering effects in period 2 (now control):
  → Control estimate is contaminated
  → Treatment effect is UNDERESTIMATED

Solutions:
  1. Washout periods: Exclude first X minutes of each period
  2. Longer periods: Carryover effect is small relative to period length  
  3. Carryover modeling: Include lagged treatment in regression
  4. Use only non-adjacent periods: Compare T periods to C periods
     that don't immediately follow a T period
```

---

## Ego-Network Randomization {#ego-network-randomization}

### The Concept

Randomize at the level of an "ego" (focal user) + their immediate network (alters).

```
Standard: Randomize user independently
  → User's friends may be in opposite condition → interference

Ego-network: Randomize user AND their immediate connections
  → Ego in treatment → all of ego's friends also in treatment
  → This "saturates" the local network → can observe full network effect
```

### Graph Cluster Randomization

```
┌───────────────┐     ┌───────────────┐
│   CLUSTER A   │     │   CLUSTER B   │
│  (Treatment)  │     │  (Control)    │
│               │     │               │
│    ●──●       │     │    ○──○       │
│   /│   \      │     │   /│   \      │
│  ● │    ●     │     │  ○ │    ○     │
│   \│   /      │     │   \│   /      │
│    ●──●       │     │    ○──○       │
│               │     │               │
└───────────────┘     └───────────────┘
    ↕ Few edges between clusters (interference minimized)
```

### Implementation Approaches

| Approach | Description | Trade-off |
|----------|-------------|-----------|
| Connected components | Randomize each disconnected subgraph | Often 1 giant component exists |
| Graph partitioning | K-way cut to minimize cross-edges | Some interference leaks across |
| Ego-cluster | User + 1-hop neighbors as unit | Overlapping egos create complexity |
| Weighted randomization | Weight by network position | Better balance but complex |
| Two-stage randomization | Randomize clusters, then proportion within | Measures direct + spillover separately |

### Measuring Direct vs. Spillover Effects

```
Two-stage randomization design:

Stage 1: Randomize clusters to "high saturation" (75% treated) 
          vs "low saturation" (25% treated)

Stage 2: Within each cluster, randomly assign individuals

This gives you:
  - Direct effect: Treated in low-sat vs. Control in low-sat
  - Total effect: Treated in high-sat vs. Control in low-sat
  - Spillover effect: Control in high-sat vs. Control in low-sat
  
  Total = Direct + Spillover (approximately)
```

> **Critical Insight** 💡
> Ego-network randomization helps measure network effects but has severe power limitations. Networks are large and interconnected; clean clusters are hard to form. In practice, many companies (LinkedIn, Facebook) combine this with computational approaches — simulate what would happen at full deployment using structural models fit on partial experimental data.

---

## Marketplace Interference {#marketplace-interference}

### The Two-Sided Problem

In marketplaces, treatment on one side affects outcomes on the other side.

```
BUYER EXPERIMENT:
  - Treatment buyers get better recommendations
  - They buy different items → affects which items SELLERS sell
  - Control buyers now face different inventory/competition
  - → Interference across buyers through the shared supply

SELLER EXPERIMENT:  
  - Treatment sellers get better pricing suggestions
  - They attract more buyers → control sellers lose buyers
  - → Displacement: treatment gain ≈ control loss
  - → Overestimated treatment effect
```

### Marketplace-Specific Challenges

| Challenge | Explanation | Example |
|-----------|-------------|---------|
| Budget constraints | Buyer spends fixed budget; allocation shifts but total constant | E-commerce: treatment redirects spend, not create new spend |
| Inventory constraints | Fixed supply means treatment claims from control | Rideshare: treatment riders get drivers faster → control waits longer |
| Price effects | Treatment changes prices through supply/demand | Surge pricing experiments affect all riders in area |
| Two-sided matching | Both sides interact; can't isolate one side | Dating apps: treatment user matches affect control matches |

### Solutions for Marketplace Experiments

```
1. GEO-RANDOMIZATION
   Randomize by geography (cities, regions)
   → Supply and demand within a geo are in same condition
   → Limited interference across geos (if markets are local)
   → Challenge: few geos → low power

2. TIME-BASED SWITCHBACK
   All supply and demand in same condition per time period
   → No cross-condition interference within a period
   → Challenge: carryover effects between periods

3. BUDGET-CONSTRAINED DESIGN
   Account for budget constraints in the analysis
   → Model total market-level outcome (not per-user)
   → Challenge: requires market-level randomization

4. SYNTHETIC CONTROL ON GEOS
   Launch in subset of markets, use synthetic control for others
   → Handles few treated geos naturally
   → Challenge: few observations for inference
```

### Marketplace Effect Decomposition

```python
# Decomposing marketplace experiment effects
# 
# Total Market Effect = Direct Effect + Displacement + Market Expansion
#
# Example: New search algorithm for buyers
#   Direct: Users shown better results buy more from those items
#   Displacement: Those items sold MORE → other items sold LESS (zero-sum)
#   Expansion: Better experience → users spend more total (positive-sum)
#
# A standard user-level A/B test captures:
#   Direct + partial Displacement (on control group)
#   = OVERESTIMATES if displacement dominated
#   = UNDERESTIMATES if expansion is the main channel
#
# Need: market-level analysis to capture all three

def marketplace_effect_analysis(df, geo_col, treatment_col, outcome_col):
    """
    Analyze at market level to capture displacement.
    """
    # Aggregate to market level
    market_df = df.groupby([geo_col, treatment_col]).agg(
        total_outcome=(outcome_col, 'sum'),
        n_users=(outcome_col, 'count')
    ).reset_index()
    
    # Compare total market-level outcomes between treated and control markets
    treated_markets = market_df[market_df[treatment_col] == 1]['total_outcome']
    control_markets = market_df[market_df[treatment_col] == 0]['total_outcome']
    
    market_level_effect = treated_markets.mean() - control_markets.mean()
    return market_level_effect  # captures displacement
```

---

## Design Strategies for Interference {#design-strategies}

### Strategy Comparison

| Strategy | Interference Handled | Power | Complexity | Best For |
|----------|---------------------|-------|-----------|----------|
| Cluster randomization | Within-cluster | Low | Medium | Social features |
| Geo randomization | Within-geo | Very low | Low | Marketplace, location-based |
| Switchback | All (within time) | Medium | High | Rideshare, delivery |
| Ego-network | 1-hop neighbors | Low | High | Social network features |
| Two-stage (saturation) | Measures spillover | Very low | Very high | Research, understanding mechanisms |
| Ghost ads / intent-to-treat | Content delivery | High | Low | Ads, recommendations |

### Practical Decision Framework

```
Is there interference in your experiment?
│
├── Social/viral (sharing, referrals)?
│   ├── Can you identify clusters? → Cluster randomization
│   └── Highly connected graph? → Consider saturation design or model-based
│
├── Marketplace (shared inventory/supply)?
│   ├── Local market (rides, food)? → Geo-level or switchback
│   └── Global market (e-commerce)? → Very difficult; budget-split or model-based
│
├── Shared resources (servers, content pool)?
│   └── Time-based switchback (all users same condition per period)
│
└── Unsure if interference exists?
    └── Run both individual AND cluster-level analysis
        If estimates differ significantly → interference likely exists
```

### Detecting Interference When Not Designed For

```python
def detect_interference(df, user_col, treatment_col, outcome_col, network_edges):
    """
    Post-hoc check: does having treated friends affect control users?
    """
    # Calculate fraction of treated friends for each control user
    control_users = df[df[treatment_col] == 0]
    
    for idx, row in control_users.iterrows():
        user = row[user_col]
        friends = network_edges[network_edges['source'] == user]['target']
        friend_data = df[df[user_col].isin(friends)]
        fraction_treated_friends = friend_data[treatment_col].mean()
        control_users.loc[idx, 'frac_treated_friends'] = fraction_treated_friends
    
    # If control users with MORE treated friends have different outcomes
    # → interference exists
    correlation = control_users[['frac_treated_friends', outcome_col]].corr()
    
    # Alternatively: regression of control outcomes on fraction of treated neighbors
    # Significant coefficient → interference
    
    return correlation
```

---

## Interview Talking Points {#interview-talking-points}

> **"How do you detect novelty effects in an A/B test?"**
>
> "I look at the treatment effect over time — specifically, I plot the daily (or weekly) difference between treatment and control. If the effect is large early and decays toward zero, that's novelty. I also use cohort analysis: group users by how long they've been in the experiment, and compare the effect for 'new-to-treatment' users versus those with 2+ weeks of exposure. If long-exposed users show a smaller effect, novelty is present. The key implication: I can't trust the overall test result if I only ran it for two weeks — I need to project the steady-state effect."

> **"Your team wants to test a social sharing feature. How would you design the experiment?"**
>
> "A standard user-level A/B test won't work here because of interference — if a treated user shares content with a control user, the control user's engagement changes due to treatment. I'd recommend cluster randomization: identify relatively independent social clusters (communities, friend groups with few cross-cluster connections) and randomize at the cluster level. The trade-off is power — I need many clusters, and with high intra-cluster correlation, my effective sample size drops significantly. I'd calculate the design effect (1 + (cluster_size - 1) x ICC), determine if I have enough clusters for adequate power, and if not, explore ego-network designs or accept that I'll need a longer test duration."

> **"How does marketplace interference bias your estimates?"**
>
> "The direction depends on the mechanism. In a rideshare pricing experiment: if treatment riders get lower prices and request more rides, they absorb more of the driver supply — control riders now wait longer. This OVERESTIMATES the treatment effect because control is hurt. Conversely, in a social feature experiment: treatment users share benefits with control users, UNDERESTIMATING the true effect because control benefits from spillover. For marketplaces, I'd use geo-level or switchback randomization where all participants in a local market are in the same condition, eliminating within-market interference."

> **"How long should you run a test to account for novelty effects?"**
>
> "There's no universal answer, but my heuristic is: run until the treatment effect stabilizes for at least one full week. Typically 3-4 weeks is needed for novelty to wear off. I monitor the time-series of the treatment effect — once it's flat (no statistically significant downward trend) for 7+ days, I'm more confident in the steady-state estimate. For major UI changes that may trigger primacy effects, I'd want 4-6 weeks. And I always compare new users (no novelty because they never saw the old version) versus existing users to separate novelty from the true long-term effect."

---

## Common Mistakes {#common-mistakes}

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Shipping after 2-week test without checking for novelty decay | Plot treatment effect over time; check for declining trend |
| Killing a feature after short test without considering primacy | Run longer tests for UX changes; compare new vs existing users |
| Running standard A/B test with social features | Use cluster randomization to avoid spillover bias |
| Geo-experiment with 4 cities (2 treatment, 2 control) | Very low power; need 10+ units per group or use synthetic control |
| Ignoring carryover in switchback experiments | Include washout periods between switches |
| Measuring per-user effect in marketplace (ignoring displacement) | Measure market-level aggregate outcome to capture displacement |
| Assuming interference only goes one direction | Model both direct effects AND spillover/displacement separately |
| Using individual-level standard errors in cluster-randomized design | Cluster standard errors at randomization unit level |
| Not testing for interference before assuming SUTVA holds | Check if control outcomes correlate with neighbor treatment rates |
| Same switchback period for all metrics | Match period length to metric's time-to-manifest |

---

## Rapid-Fire Q&A {#rapid-fire-qa}

**Q1: What is a novelty effect and how does it bias experiments?**
> Users engage more with a new feature simply because it's new. It upward-biases the treatment effect estimate, especially in short-duration tests. The measured effect exceeds the long-term steady-state value.

**Q2: How do novelty and primacy effects differ?**
> Novelty: users engage MORE initially because something is new (effect decays to true level). Primacy: users engage LESS initially because they resist change (effect rises to true level). Novelty risks shipping bad features; primacy risks killing good ones.

**Q3: How do you detect novelty effects?**
> (1) Plot treatment effect over time — declining trend indicates novelty. (2) Cohort analysis by exposure duration. (3) Compare new users (never saw old version) vs. existing users.

**Q4: What is SUTVA and why does it matter for experiments?**
> Stable Unit Treatment Value Assumption: each unit's outcome depends only on its own treatment. When violated (networks, marketplaces), standard A/B test estimates are biased because control users are affected by treatment.

**Q5: What is cluster randomization and when do you use it?**
> Randomize groups (clusters) instead of individuals. Use when interference occurs within clusters (social features, local markets). Key cost: massive power reduction due to intra-cluster correlation.

**Q6: What is a switchback experiment?**
> Alternate between treatment and control over time for the same units. All users are in the same condition during each time period, eliminating within-period interference. Used heavily in rideshare and delivery platforms.

**Q7: What is the design effect in cluster experiments?**
> DEFF = 1 + (m-1) x ICC, where m is cluster size and ICC is intra-cluster correlation. Effective sample size = N/DEFF. With ICC=0.05 and m=100, you need ~6x more users than individual-level randomization.

**Q8: How does marketplace interference differ from social interference?**
> Marketplace interference often creates displacement (zero-sum: treatment gains at control's expense → overestimates). Social interference often creates spillover (positive-sum: control benefits from treatment → underestimates).

**Q9: What is ego-network randomization?**
> Randomize users along with their immediate network neighbors. Ensures the focal user's local network is saturated with the same treatment, allowing measurement of network-mediated effects.

**Q10: How do you choose between geo-randomization and switchback?**
> Geo if: markets are independent, you have many markets (15+), effects are stable over time. Switchback if: markets interact, few markets available, effects are immediate and short-lived, carryover is minimal.

---

## ASCII Cheat Sheet {#cheat-sheet}

```
╔══════════════════════════════════════════════════════════════════════╗
║     NOVELTY, NETWORK EFFECTS & INTERFERENCE CHEAT SHEET            ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  TEMPORAL EFFECTS:                                                   ║
║  ┌─────────────────────────────────────────────────────────┐        ║
║  │  NOVELTY: Effect DECLINES over time                     │        ║
║  │    • Users try new thing out of curiosity               │        ║
║  │    • Risk: Ship feature with no lasting value           │        ║
║  │    • Fix: Run test 3-4+ weeks; check for decay          │        ║
║  │                                                         │        ║
║  │  PRIMACY: Effect INCREASES over time                    │        ║
║  │    • Users resist change to habits                      │        ║
║  │    • Risk: Kill good feature prematurely                │        ║
║  │    • Fix: Run test 4-6 weeks; compare new vs old users  │        ║
║  └─────────────────────────────────────────────────────────┘        ║
║                                                                      ║
║  DETECTION: Plot daily treatment effect                              ║
║    Declining → Novelty    Increasing → Primacy    Flat → Stable     ║
║                                                                      ║
║  INTERFERENCE (SUTVA VIOLATION):                                     ║
║  ┌─────────────────────────────────────────────────────────┐        ║
║  │  Social spillover: Control helped by treatment          │        ║
║  │    → UNDERESTIMATES true effect                         │        ║
║  │                                                         │        ║
║  │  Marketplace displacement: Control hurt by treatment    │        ║
║  │    → OVERESTIMATES true effect                          │        ║
║  └─────────────────────────────────────────────────────────┘        ║
║                                                                      ║
║  SOLUTIONS FOR INTERFERENCE:                                         ║
║  ┌──────────────────┬─────────────┬──────────────────┐             ║
║  │ Method           │ Power       │ Best For          │             ║
║  ├──────────────────┼─────────────┼──────────────────┤             ║
║  │ Cluster random.  │ Low         │ Social features   │             ║
║  │ Geo experiments  │ Very low    │ Marketplace       │             ║
║  │ Switchback       │ Medium      │ Rideshare/delivery│             ║
║  │ Ego-network      │ Low         │ Network effects   │             ║
║  │ Saturation design│ Very low    │ Measuring spillover│            ║
║  └──────────────────┴─────────────┴──────────────────┘             ║
║                                                                      ║
║  DESIGN EFFECT (Power Penalty):                                      ║
║  DEFF = 1 + (cluster_size - 1) × ICC                               ║
║  Effective N = Total N / DEFF                                        ║
║                                                                      ║
║  ICC = 0.01, m = 100 → DEFF = 1.99 (need ~2x users)               ║
║  ICC = 0.05, m = 100 → DEFF = 5.95 (need ~6x users)               ║
║  ICC = 0.10, m = 100 → DEFF = 10.9 (need ~11x users)              ║
║                                                                      ║
║  SWITCHBACK DESIGN:                                                  ║
║  • Period length: long enough for effect, short enough for power    ║
║  • Washout: exclude transition minutes between periods              ║
║  • Carryover: model or eliminate through longer periods             ║
║  • Analysis: period-level with HAC standard errors                  ║
║                                                                      ║
║  MARKETPLACE EFFECT:                                                 ║
║  Total = Direct + Displacement + Market Expansion                   ║
║  Measure at MARKET level (not user level) to capture all components ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 28 of 45*
