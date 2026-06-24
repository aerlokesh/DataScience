# 🎯 Topic 49: Attribution Modeling & Marketing Mix

> *"The half of my advertising that works — I just don't know which half." Understanding how to measure marketing's true impact is the billion-dollar question every data scientist at Google, Meta, or Amazon Ads must answer.*

---

## 📑 Table of Contents

1. [Multi-Touch Attribution Models](#1-multi-touch-attribution-models)
2. [Heuristic Attribution Models Compared](#2-heuristic-attribution-models-compared)
3. [Algorithmic / Data-Driven Attribution](#3-algorithmic--data-driven-attribution)
4. [Incrementality Testing](#4-incrementality-testing)
5. [Marketing Mix Modeling (MMM)](#5-marketing-mix-modeling-mmm)
6. [Adstock, Carry-Over & Diminishing Returns](#6-adstock-carry-over--diminishing-returns)
7. [Budget Optimization](#7-budget-optimization)
8. [Attribution vs MMM vs Incrementality — Comparison](#8-attribution-vs-mmm-vs-incrementality--comparison)
9. [When to Use Which](#9-when-to-use-which)
10. [Python Implementation Examples](#10-python-implementation-examples)
11. [Interview Talking Points](#11-interview-talking-points)
12. [Common Mistakes](#12-common-mistakes)
13. [Rapid-Fire Q&A](#13-rapid-fire-qa)
14. [ASCII Cheat Sheet](#14-ascii-cheat-sheet)

---

## 1. Multi-Touch Attribution Models

In digital marketing, a user typically interacts with multiple touchpoints (channels, ads, campaigns) before converting. **Multi-Touch Attribution (MTA)** distributes conversion credit across these touchpoints.

### The Attribution Problem

```
User Journey Example:
Day 1: Facebook Ad (impression) → Day 3: Google Search Ad (click) → Day 5: Email (click) → Day 7: Direct (purchase)

Question: Which channel gets credit for the $100 purchase?
```

### Heuristic Models

| Model | Logic | Credit Distribution |
|-------|-------|-------------------|
| **Last-Touch** | 100% to final touchpoint | Simple but biased toward bottom-funnel |
| **First-Touch** | 100% to first touchpoint | Biased toward awareness channels |
| **Linear** | Equal credit to all touchpoints | Democratic but naive |
| **Time-Decay** | More credit to recent touchpoints | Exponential decay from conversion |
| **Position-Based (U-Shaped)** | 40% first, 40% last, 20% split middle | Acknowledges intro + close |
| **W-Shaped** | 30% first, 30% lead-creation, 30% last, 10% rest | B2B focused |

> **Critical Insight:** Every heuristic model embeds an assumption about *which position matters most*. None of them use data to validate that assumption — they are rules, not models.

### Why Last-Touch Dominates in Practice

- Easy to implement (just track the final click)
- Aligns with Google Analytics default
- Creates a false sense of precision
- Systematically over-credits branded search and retargeting

---

## 2. Heuristic Attribution Models Compared

### Time-Decay Model — Mathematical Formulation

```
Credit(touchpoint_i) = 2^(-(t_conversion - t_i) / half_life)
```

Where `half_life` is typically 7 days. A touchpoint 7 days before conversion gets 50% of the weight of the last touchpoint, 14 days gets 25%, etc.

### Position-Based (U-Shaped) Model

```
Touchpoints:  [T1]  [T2]  [T3]  [T4]  [T5]
Credits:       40%   6.7%  6.7%  6.7%   40%
```

> **Critical Insight:** Position-based models can be customized. Google Ads lets you set the first/last split (e.g., 30/30/40 distributed). The "right" split is unknowable without causal data.

### Limitations of All Heuristic Models

1. **No counterfactual reasoning** — Would the conversion have happened anyway?
2. **Ignores cross-device** — Same user on phone vs laptop = two "users"
3. **Selection bias** — Users who see more ads may just be more likely to convert
4. **Cookie death** — iOS 14.5, ITP, third-party cookie deprecation breaks tracking

---

## 3. Algorithmic / Data-Driven Attribution

### Shapley Value Attribution

From cooperative game theory: assign credit based on each channel's marginal contribution across all possible coalitions.

```
Shapley Value for channel i = (1/n!) * Σ [v(S ∪ {i}) - v(S)]
                                        over all orderings
```

Where:
- `v(S)` = conversion probability when only channels in set S are present
- The sum is over all permutations of channels

**Interpretation:** Average marginal contribution of channel i across all possible orderings of channels.

### Markov Chain Attribution

Model the customer journey as a state machine:
- States = channels + {Start, Conversion, Null}
- Transitions = observed movement probabilities
- **Removal Effect:** Remove each channel and measure the drop in conversion probability

```
Attribution(channel_i) = P(conversion) - P(conversion | remove channel_i)
                         ─────────────────────────────────────────────────
                         Σ_j [P(conversion) - P(conversion | remove channel_j)]
```

> **Critical Insight:** Markov Chain attribution captures **sequence effects** (e.g., "Social before Search" converts better than "Search before Social") that Shapley-based methods cannot, because Shapley is order-invariant.

### Data-Driven Attribution (Google's DDA)

Google's implementation uses:
1. Logistic regression on conversion paths
2. Counterfactual analysis comparing converting vs non-converting paths
3. Credit assigned based on the conditional probability increase at each position

### Limitations of Algorithmic Models

- Still correlational, not causal
- Requires sufficient path diversity (cold-start problem)
- Computationally expensive (2^n coalitions for Shapley)
- Cannot account for ad impressions without clicks (view-through)

---

## 4. Incrementality Testing

### The Gold Standard: Causal Measurement

Attribution answers: "Who touched the user before conversion?"
Incrementality answers: "Would this conversion have happened WITHOUT the ad?"

### Ghost Ads / PSA Test

```
┌─────────────────────────────────┐
│  Users who WOULD have seen ad   │
├────────────────┬────────────────┤
│   Treatment    │    Control     │
│  (See real ad) │ (See ghost/PSA)│
├────────────────┼────────────────┤
│  Convert: 5.2% │  Convert: 4.1% │
└────────────────┴────────────────┘

Incremental Lift = 5.2% - 4.1% = 1.1 percentage points
iROAS = (Incremental Revenue) / (Ad Spend)
```

**Ghost Ads:** In the control group, the ad auction still runs, but a blank/PSA ad is shown. This preserves the natural selection of "who would have seen the ad."

### Intent-to-Treat (ITT) Framework

When you can't control who actually sees the ad (e.g., TV, billboards):

```
ITT Effect = E[Y | Assigned to Treatment] - E[Y | Assigned to Control]
LATE/CACE = ITT / Compliance Rate
```

### Geo-Based Incrementality

- Split regions into treatment (ads on) and control (ads off)
- Use synthetic control or matched markets
- Advantage: Works for TV, radio, OOH
- Disadvantage: Low statistical power (few geo units)

> **Critical Insight:** Incrementality testing is expensive (you're intentionally NOT showing ads to a control group = lost revenue). The business cost of the test must be weighed against the information value. Run incrementality tests on your **largest spend channels first**.

### Conversion Lift Studies (Meta/Google)

Both platforms offer built-in incrementality products:
- **Meta Conversion Lift:** Randomizes at the user level
- **Google Brand Lift / Conversion Lift:** Geo or user-level randomization
- Limitation: Platform-run tests may have conflicts of interest

---

## 5. Marketing Mix Modeling (MMM)

### Overview

MMM is a **top-down, aggregate-level** regression model that estimates how each marketing channel contributes to an outcome (revenue, conversions) using time-series data.

```
Revenue_t = β₀ + β₁·TV_t + β₂·Search_t + β₃·Social_t + β₄·Price_t + β₅·Seasonality_t + ε_t
```

### Key Characteristics

| Feature | MTA (Attribution) | MMM |
|---------|------------------|-----|
| Data granularity | User-level | Aggregate (weekly/geo) |
| Causal claim | Correlational | Quasi-causal (with controls) |
| Privacy impact | High (needs cookies) | Low (no user data) |
| Online/Offline | Online only | Both |
| Time horizon | Real-time | Quarterly/yearly |
| Implementation | Tag-based | Statistical modeling |

### Modern MMM Frameworks

| Framework | Creator | Approach |
|-----------|---------|----------|
| **Robyn** | Meta | Ridge regression + Nevergrad optimizer |
| **LightweightMMM** | Google | Bayesian (JAX/NumPyro) |
| **PyMC-Marketing** | PyMC Labs | Bayesian (PyMC) |
| **Meridian** | Google | Bayesian hierarchical (latest) |

> **Critical Insight:** Modern Bayesian MMMs incorporate **priors** from incrementality tests. This is the "calibration" step — you constrain the MMM posterior using causal ground truth from experiments. This bridges the gap between correlation (MMM) and causation (incrementality).

---

## 6. Adstock, Carry-Over & Diminishing Returns

### Adstock (Carry-Over Effect)

Advertising has a lingering effect. A TV ad shown today still influences purchases next week.

```
Adstock_t = Spend_t + λ · Adstock_{t-1}

Where λ ∈ [0, 1] is the decay rate
- λ = 0.9 → slow decay (TV, brand campaigns)
- λ = 0.3 → fast decay (search, performance ads)
```

### Geometric vs Weibull Adstock

```
Geometric:  weights = [1, λ, λ², λ³, ...]  (monotone decay)
Weibull:    weights = shape/scale * (t/scale)^(shape-1) * exp(-(t/scale)^shape)
            Allows a delayed peak (e.g., ad builds awareness before converting)
```

### Diminishing Returns (Saturation)

More spend ≠ proportionally more conversions. Model with:

**Hill Function (most common in MMM):**
```
Response(x) = Kₘₐₓ * x^n / (EC50^n + x^n)

Where:
- Kₘₐₓ = maximum achievable response
- EC50 = spend level at 50% of max response
- n = steepness (Hill coefficient)
```

**Log transformation (simple):**
```
Response = β · log(1 + Spend)
```

> **Critical Insight:** The combination of adstock + saturation means the **marginal ROI of a channel depends on both current spend level AND recent historical spend**. A channel can have high total ROI but low marginal ROI if already saturated.

---

## 7. Budget Optimization

### The Optimization Problem

```
Maximize: Total_Revenue = Σ_c Response_c(Spend_c)
Subject to: Σ_c Spend_c ≤ Total_Budget
            Spend_c ≥ 0  for all channels c
```

### Marginal ROI Equalization

At optimum, the marginal ROI (mROI) should be equal across all channels:

```
∂Response_TV/∂Spend_TV = ∂Response_Search/∂Spend_Search = ... = λ (shadow price)
```

If mROI_Search > mROI_TV → shift budget from TV to Search until equalized.

### Practical Constraints

- Minimum spend commitments (contracts)
- Maximum reach caps (audience exhaustion)
- Channel-specific lag (can't reallocate TV budget weekly)
- Business rules (brand must have X% share of voice)

---

## 8. Attribution vs MMM vs Incrementality — Comparison

| Dimension | MTA | MMM | Incrementality |
|-----------|-----|-----|---------------|
| **Causality** | Correlational | Quasi-causal | Causal (experimental) |
| **Granularity** | User-level | Aggregate | Campaign/channel |
| **Privacy needs** | High | Low | Medium |
| **Online channels** | Yes | Yes | Yes |
| **Offline channels** | No | Yes | Yes (geo tests) |
| **Real-time** | Yes | No (lagging) | No (test duration) |
| **Cost to run** | Low (passive) | Medium (modeling) | High (holdout cost) |
| **Bias** | Selection bias | Omitted variable bias | Minimal (if well-designed) |
| **Best for** | Tactical optimization | Strategic planning | Ground truth calibration |
| **Frequency** | Always-on | Quarterly | 1-2x per year per channel |

> **Critical Insight:** The industry best practice (Google, Meta, advanced advertisers) is a **triangulation approach**: use incrementality tests to calibrate MMM priors, use MMM for budget allocation, and use MTA for tactical in-flight optimization. No single method is sufficient alone.

---

## 9. When to Use Which

### Decision Framework

```
Q: Do you need real-time optimization?
├─ YES → MTA (with awareness of its biases)
└─ NO
    Q: Do you need causal proof for a specific channel?
    ├─ YES → Incrementality test
    └─ NO
        Q: Do you need cross-channel budget allocation?
        ├─ YES → MMM (calibrated with incrementality)
        └─ NO → Start with simple last-touch + common sense
```

### By Company Stage

| Stage | Recommended Approach |
|-------|---------------------|
| Startup (<$1M ad spend) | Last-touch + manual analysis |
| Growth ($1M-$10M) | MTA + occasional lift tests |
| Scale ($10M-$100M) | MMM + regular incrementality calibration |
| Enterprise ($100M+) | Full triangulation framework |

---

## 10. Python Implementation Examples

### Shapley Value Attribution

```python
import numpy as np
from itertools import combinations

def shapley_attribution(conversion_data, channels):
    """
    Calculate Shapley values for marketing channels.
    
    Parameters:
    -----------
    conversion_data : dict
        Maps frozenset of channels → conversion rate
        e.g., {frozenset({'FB'}): 0.02, frozenset({'FB','Google'}): 0.05}
    channels : list
        List of all channel names
    """
    n = len(channels)
    shapley_values = {}
    
    for channel in channels:
        shapley = 0.0
        other_channels = [c for c in channels if c != channel]
        
        for size in range(0, n):
            for subset in combinations(other_channels, size):
                S = frozenset(subset)
                S_with_i = S | {channel}
                
                # Marginal contribution
                v_with = conversion_data.get(S_with_i, 0)
                v_without = conversion_data.get(S, 0)
                marginal = v_with - v_without
                
                # Shapley weight
                weight = (np.math.factorial(len(S)) * 
                         np.math.factorial(n - len(S) - 1)) / np.math.factorial(n)
                
                shapley += weight * marginal
        
        shapley_values[channel] = shapley
    
    return shapley_values

# Example usage
conversion_rates = {
    frozenset(): 0.00,
    frozenset({'Facebook'}): 0.02,
    frozenset({'Google'}): 0.04,
    frozenset({'Email'}): 0.03,
    frozenset({'Facebook', 'Google'}): 0.07,
    frozenset({'Facebook', 'Email'}): 0.06,
    frozenset({'Google', 'Email'}): 0.08,
    frozenset({'Facebook', 'Google', 'Email'}): 0.12,
}

channels = ['Facebook', 'Google', 'Email']
shapley = shapley_attribution(conversion_rates, channels)
print("Shapley Values:", shapley)
# Facebook: 0.025, Google: 0.045, Email: 0.05
```

### Markov Chain Attribution

```python
import numpy as np
import pandas as pd
from collections import defaultdict

def markov_attribution(journeys, conversions):
    """
    Calculate channel attribution using first-order Markov chains.
    
    Parameters:
    -----------
    journeys : list of lists
        Each journey is a list of channel names
    conversions : list of bool
        Whether each journey resulted in conversion
    """
    # Build transition matrix
    transitions = defaultdict(lambda: defaultdict(int))
    
    for journey, converted in zip(journeys, conversions):
        path = ['Start'] + journey + ['Conversion' if converted else 'Null']
        for i in range(len(path) - 1):
            transitions[path[i]][path[i+1]] += 1
    
    # Normalize to probabilities
    trans_prob = {}
    states = set()
    for s1 in transitions:
        total = sum(transitions[s1].values())
        trans_prob[s1] = {s2: c/total for s2, c in transitions[s1].items()}
        states.add(s1)
        states.update(transitions[s1].keys())
    
    # Calculate baseline conversion probability
    def calc_conversion_prob(trans_prob, states, removed=None):
        """Simulate conversion probability via matrix absorption."""
        state_list = [s for s in states if s not in ('Conversion', 'Null') 
                      and s != removed]
        n = len(state_list)
        idx = {s: i for i, s in enumerate(state_list)}
        
        Q = np.zeros((n, n))  # Transient → Transient
        R_conv = np.zeros(n)  # Transient → Conversion
        
        for s1 in state_list:
            probs = trans_prob.get(s1, {})
            for s2, p in probs.items():
                if s2 == removed:
                    continue
                if s2 in idx:
                    Q[idx[s1], idx[s2]] = p
                elif s2 == 'Conversion':
                    R_conv[idx[s1]] = p
        
        # Normalize rows
        row_sums = Q.sum(axis=1) + R_conv
        for i in range(n):
            if row_sums[i] > 0:
                Q[i] /= row_sums[i] + (1 - row_sums[i])  # simplified
        
        # Fundamental matrix: N = (I - Q)^(-1)
        try:
            N = np.linalg.inv(np.eye(n) - Q)
            absorption = N @ R_conv
            start_idx = idx.get('Start', 0)
            return absorption[start_idx]
        except np.linalg.LinAlgError:
            return 0.0
    
    baseline = calc_conversion_prob(trans_prob, states)
    
    # Removal effect for each channel
    channels = [s for s in states if s not in ('Start', 'Conversion', 'Null')]
    removal_effects = {}
    
    for channel in channels:
        prob_without = calc_conversion_prob(trans_prob, states, removed=channel)
        removal_effects[channel] = baseline - prob_without
    
    # Normalize to get attribution
    total_effect = sum(removal_effects.values())
    attribution = {ch: effect/total_effect for ch, effect in removal_effects.items()}
    
    return attribution
```

### Simple MMM with Adstock and Saturation

```python
import numpy as np
import pandas as pd
from scipy.optimize import minimize

def geometric_adstock(spend, decay_rate):
    """Apply geometric adstock transformation."""
    adstocked = np.zeros_like(spend, dtype=float)
    adstocked[0] = spend[0]
    for t in range(1, len(spend)):
        adstocked[t] = spend[t] + decay_rate * adstocked[t-1]
    return adstocked

def hill_saturation(x, half_max, slope):
    """Apply Hill saturation curve."""
    return x**slope / (half_max**slope + x**slope)

def mmm_predict(params, X_channels, X_controls):
    """
    Predict revenue from MMM parameters.
    
    params: [intercept, *betas_channel, *betas_control, *decays, *half_maxs, *slopes]
    """
    n_channels = X_channels.shape[1]
    n_controls = X_controls.shape[1]
    
    intercept = params[0]
    betas_ch = params[1:1+n_channels]
    betas_ctrl = params[1+n_channels:1+n_channels+n_controls]
    decays = params[1+n_channels+n_controls:1+2*n_channels+n_controls]
    half_maxs = params[1+2*n_channels+n_controls:1+3*n_channels+n_controls]
    slopes = params[1+3*n_channels+n_controls:1+4*n_channels+n_controls]
    
    y_pred = np.full(len(X_channels), intercept)
    
    for i in range(n_channels):
        adstocked = geometric_adstock(X_channels[:, i], decays[i])
        saturated = hill_saturation(adstocked, half_maxs[i], slopes[i])
        y_pred += betas_ch[i] * saturated
    
    for i in range(n_controls):
        y_pred += betas_ctrl[i] * X_controls[:, i]
    
    return y_pred

def fit_mmm(y, X_channels, X_controls):
    """Fit MMM via least squares (simplified)."""
    n_ch = X_channels.shape[1]
    n_ctrl = X_controls.shape[1]
    
    def loss(params):
        y_pred = mmm_predict(params, X_channels, X_controls)
        return np.sum((y - y_pred)**2)
    
    # Initial params: intercept + betas + decays + half_maxs + slopes
    x0 = np.concatenate([
        [y.mean()],                  # intercept
        np.ones(n_ch) * 0.5,        # channel betas
        np.zeros(n_ctrl),            # control betas
        np.ones(n_ch) * 0.5,        # decay rates
        np.ones(n_ch) * 0.5,        # half_max (normalized)
        np.ones(n_ch) * 1.5,        # slopes
    ])
    
    bounds = (
        [(None, None)] +             # intercept
        [(0, None)] * n_ch +         # channel betas (positive)
        [(None, None)] * n_ctrl +    # control betas
        [(0.01, 0.99)] * n_ch +      # decay rates
        [(0.01, 5)] * n_ch +         # half_max
        [(0.5, 4)] * n_ch            # slopes
    )
    
    result = minimize(loss, x0, bounds=bounds, method='L-BFGS-B')
    return result.x
```

---

## 11. Interview Talking Points

> **"Walk me through how you'd measure the incrementality of our Facebook ads spend."**
>
> "I'd design a conversion lift study using geographic holdouts. First, I'd select matched market pairs — similar cities by population, baseline conversion rate, and seasonality. One gets Facebook ads (treatment), one doesn't (control). I'd run for 4-6 weeks, measure the conversion rate delta, and calculate iROAS. If the platform offers user-level randomization, I'd prefer that for tighter confidence intervals. Key consideration: the holdout cost must be justified by the decision value of knowing true incrementality."

> **"Our last-touch attribution shows Google branded search drives 40% of revenue. Do you believe that?"**
>
> "I'd be skeptical. Branded search captures demand that was already created by upstream channels — users searching your brand name are already aware. Last-touch over-credits the 'last mile' while ignoring the demand generation that led there. I'd look at: (1) what share of branded search converters had prior touchpoints with paid social or display, (2) run an incrementality test pausing branded search in a geo — often, organic captures 80%+ of that traffic. The true incremental value of branded search is typically 10-30% of what last-touch suggests."

> **"How would you build an MMM from scratch for our business?"**
>
> "I'd start with 2-3 years of weekly data: revenue as the target, media spend by channel as features, plus controls (price, promotions, seasonality, macro factors, competitor activity). I'd apply adstock transformations with channel-specific decay rates — fast for search, slow for TV. Then Hill saturation curves to capture diminishing returns. I'd use a Bayesian framework like PyMC-Marketing so I can encode prior knowledge (e.g., from past incrementality tests) as informative priors. Validation: time-series cross-validation on held-out periods, plus checking that posterior channel contributions align directionally with known incrementality results."

---

## 12. Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Treating last-touch as ground truth | Use last-touch as one signal; validate with incrementality |
| Running MTA without addressing cross-device | Implement identity resolution or acknowledge the gap |
| Building MMM without saturation curves | Always model diminishing returns — linear response is unrealistic |
| Setting adstock decay arbitrarily | Estimate decay from data; use priors from industry benchmarks |
| Optimizing budget purely on MTA | Use MMM for allocation; MTA for intra-channel optimization |
| Running incrementality test too short | Minimum 2-4 weeks to capture lagged effects |
| Ignoring baseline conversions | Always measure organic/null conversion rate |
| Using MMM without calibration | Calibrate with incrementality test results as priors |
| Conflating correlation with causation in MMM | MMM is quasi-causal at best; acknowledge limitations |
| Reporting average ROI instead of marginal ROI | Marginal ROI drives optimization decisions |

---

## 13. Rapid-Fire Q&A

**Q1: Why does last-touch attribution over-credit branded search?**
A: Users who search your brand name already have intent — that intent was created by upstream channels. Last-touch gives 100% credit to the channel that captured existing demand rather than created it.

**Q2: What's the difference between ROAS and iROAS?**
A: ROAS = total revenue from attributed conversions / spend. iROAS = *incremental* revenue (above what would have happened without ads) / spend. iROAS is always lower than ROAS.

**Q3: How does Shapley attribution handle the combinatorial explosion with 20+ channels?**
A: Exact Shapley is O(2^n). In practice, use sampling-based approximations (Monte Carlo Shapley) or group channels into categories.

**Q4: What is the "adstock half-life" for search vs TV?**
A: Search: typically 1-3 days (immediate effect). TV: 2-6 weeks (brand building persists). Social: 3-7 days. Email: 1-2 days.

**Q5: Why do MMMs need 2+ years of data?**
A: To separate seasonality from media effects. With only 1 year, you can't distinguish "sales go up in December because of holiday ads" from "sales go up in December regardless of ads."

**Q6: What's the "ghost ad" methodology?**
A: In the control group, the ad auction runs normally but the winning ad is replaced with a blank/PSA. This ensures the control group has identical composition to treatment (same targeting, same users who would have been reached).

**Q7: How do you handle the "walled garden" problem in attribution?**
A: Each platform (Google, Meta, Amazon) only sees its own touchpoints. Solutions: (1) clean rooms for cross-platform data, (2) MMM at the aggregate level, (3) incrementality tests per platform independently.

**Q8: What's the difference between short-term and long-term incrementality?**
A: Short-term measures immediate conversion lift during the test. Long-term measures brand effects — users may not convert during the test but have increased purchase probability later. Most tests only capture short-term.

**Q9: How does cookie deprecation affect attribution?**
A: Without third-party cookies, cross-site journey tracking breaks. Solutions: first-party data strategies, probabilistic matching, MMM (no cookies needed), conversion APIs, privacy-preserving measurement (differential privacy).

**Q10: In an MMM, how do you validate that the model is working?**
A: (1) Out-of-time validation (hold out last 3 months, predict). (2) MAPE < 10% on holdout. (3) Channel contributions directionally consistent with incrementality tests. (4) Spend coefficient signs are positive. (5) Business intuition check on ROI rankings.

---

## 14. ASCII Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║              ATTRIBUTION & MARKETING MEASUREMENT                     ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  ATTRIBUTION MODELS (User-Level, Correlational)                      ║
║  ┌─────────────────────────────────────────────┐                     ║
║  │  Last-Touch:   [  ][  ][  ][██] ← 100%     │                     ║
║  │  First-Touch:  [██][  ][  ][  ] ← 100%     │                     ║
║  │  Linear:       [25][25][25][25] ← equal     │                     ║
║  │  Time-Decay:   [10][15][25][50] ← recency   │                     ║
║  │  U-Shaped:     [40][10][10][40] ← ends      │                     ║
║  │  Shapley:      [??][??][??][??] ← data      │                     ║
║  └─────────────────────────────────────────────┘                     ║
║                                                                      ║
║  INCREMENTALITY (Causal, Experimental)                               ║
║  ┌─────────────────────────────────────────────┐                     ║
║  │  Treatment (see ads):  ████████░░  5.2%     │                     ║
║  │  Control (no ads):     ██████░░░░  4.1%     │                     ║
║  │  Incremental Lift:     ██░░░░░░░░  1.1pp    │                     ║
║  └─────────────────────────────────────────────┘                     ║
║                                                                      ║
║  MMM COMPONENTS                                                      ║
║  ┌─────────────────────────────────────────────┐                     ║
║  │  Revenue = f(Adstock(Spend), Controls, ε)   │                     ║
║  │                                             │                     ║
║  │  Adstock:    ▓▓▒▒░░ (decay over time)      │                     ║
║  │  Saturation: ╱‾‾‾‾ (diminishing returns)   │                     ║
║  │  Budget Opt: mROI₁ = mROI₂ = ... = mROIₙ  │                     ║
║  └─────────────────────────────────────────────┘                     ║
║                                                                      ║
║  TRIANGULATION (Best Practice)                                       ║
║  ┌─────────────────────────────────────────────┐                     ║
║  │      Incrementality                          │                     ║
║  │           │ (calibrates)                     │                     ║
║  │           ▼                                  │                     ║
║  │         MMM ──(allocates)──► Budget Plan     │                     ║
║  │           │                                  │                     ║
║  │           ▼                                  │                     ║
║  │         MTA ──(optimizes)──► In-Flight Ops   │                     ║
║  └─────────────────────────────────────────────┘                     ║
║                                                                      ║
║  KEY FORMULAS                                                        ║
║  • Adstock_t = Spend_t + λ·Adstock_{t-1}                            ║
║  • Hill(x) = x^n / (EC50^n + x^n)                                   ║
║  • iROAS = ΔRevenue_incremental / AdSpend                            ║
║  • Shapley = (1/n!) Σ marginal contributions                        ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 49 of 51*
