# 🎯 Topic 24: Metric Selection for Experiments

> *"A well-chosen metric makes the experiment; a poorly chosen one wastes months and misleads the business."*

---

## Table of Contents

1. [Metric Taxonomy](#metric-taxonomy)
2. [Primary Metric (OEC)](#primary-metric-oec)
3. [Secondary Metrics](#secondary-metrics)
4. [Guardrail Metrics](#guardrail-metrics)
5. [Counter Metrics](#counter-metrics)
6. [Sensitivity vs. Specificity of Metrics](#sensitivity-vs-specificity)
7. [Composite Metrics](#composite-metrics)
8. [Ratio Metrics and Their Pitfalls](#ratio-metrics)
9. [Noisy vs. Slow Metrics](#noisy-vs-slow)
10. [Practical Examples by Company](#practical-examples)
11. [Interview Talking Points](#interview-talking-points)
12. [Common Mistakes](#common-mistakes)
13. [Rapid-Fire Q&A](#rapid-fire-qa)
14. [ASCII Cheat Sheet](#cheat-sheet)

---

## Metric Taxonomy {#metric-taxonomy}

```
┌─────────────────────────────────────────────────────────────┐
│                    METRIC HIERARCHY                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  PRIMARY (OEC)         → The ONE metric you optimize for    │
│    │                                                        │
│    ├── SECONDARY       → Supporting evidence of success     │
│    │                                                        │
│    ├── GUARDRAIL       → "Do no harm" constraints           │
│    │                                                        │
│    └── COUNTER         → Ensures no gaming/cannibalization  │
│                                                             │
│  DIAGNOSTIC            → Explains WHY primary moved         │
│                                                             │
│  DATA QUALITY          → Validates experiment integrity     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Metric Types at a Glance

| Type | Purpose | Example | Action if Moved |
|------|---------|---------|-----------------|
| Primary (OEC) | Decision criterion | Revenue per user | Ship if positive |
| Secondary | Supporting signal | Click-through rate | Supports narrative |
| Guardrail | Safety boundary | Page load time | Block ship if degraded |
| Counter | Detect gaming | Customer support tickets | Investigate if worsened |
| Diagnostic | Debug mechanism | Funnel stage completion | Understand causality |
| Data quality | Experiment validity | Sample Ratio Mismatch | Invalidate if failed |

---

## Primary Metric (OEC) {#primary-metric-oec}

### Overall Evaluation Criterion

The OEC is the **single metric** that determines whether the experiment is a success. It should:

1. **Align with long-term business value** (not just short-term engagement)
2. **Be measurable within the experiment timeframe**
3. **Be sensitive enough** to detect the expected change
4. **Be attributable** to the treatment (causally connected)

> **Critical Insight** 💡
> The hardest part of metric selection is the tension between what you CAN measure quickly (clicks, page views) and what you CARE about long-term (revenue, retention). The OEC must bridge this gap — it's a measurable proxy that correlates with long-term value.

### Properties of a Good OEC

| Property | Description | Bad Example | Good Example |
|----------|-------------|-------------|--------------|
| Sensitive | Detects change quickly | 30-day retention | Session success rate |
| Directional | Clear what "better" means | Total page views | Revenue per session |
| Not gameable | Can't be inflated trivially | Raw click count | Task completion rate |
| Stable | Low variance relative to signal | Individual purchase amount | Average order value |
| Timely | Observable in test window | LTV | Predicted LTV proxy |

### The OEC Selection Process

```
1. What is the LONG-TERM goal?
   → User retention, revenue growth, user satisfaction

2. What SHORT-TERM metric correlates with it?
   → Run correlational analysis on historical data

3. Is that metric SENSITIVE to this specific change?
   → Check if feature logically impacts the metric

4. Can we MEASURE it in the test window?
   → Ensure data pipeline captures it

5. Is the VARIANCE manageable?
   → Calculate required N — is it feasible?
```

---

## Secondary Metrics {#secondary-metrics}

Secondary metrics provide **additional context** but do NOT determine the ship decision alone.

### Role of Secondary Metrics

- Confirm the primary metric movement makes sense mechanistically
- Provide early signal (leading indicators)
- Help diagnose unexpected primary metric movements
- Build the narrative for stakeholders

### Example: E-commerce Search Improvement

| Metric Type | Metric | Role |
|-------------|--------|------|
| Primary | Revenue per search session | Decision criterion |
| Secondary | Add-to-cart rate | Confirms search relevance improved |
| Secondary | Search result click-through | Early engagement signal |
| Secondary | Items viewed per search | Exploration behavior |
| Guardrail | Zero-result rate | Ensure coverage didn't drop |
| Counter | Return rate | Ensure quality didn't suffer |

> **Critical Insight** 💡
> If your primary metric moves positively but secondary metrics tell a contradictory story, investigate before shipping. A revenue increase driven by fewer but higher-value purchases (when you expected more purchases) signals a different mechanism than intended.

---

## Guardrail Metrics {#guardrail-metrics}

Guardrail metrics define **boundaries that must not be crossed**, regardless of primary metric improvement.

### Common Guardrail Categories

| Category | Metrics | Typical Threshold |
|----------|---------|-------------------|
| Performance | Page load time, latency | No degradation > 100ms |
| Reliability | Error rate, crash rate | No increase > 0.1% |
| User safety | Spam rate, harmful content exposure | No increase |
| Revenue protection | Revenue per user (for engagement tests) | No decrease > 1% |
| Engagement floor | DAU, session length | No decrease > 2% |
| Customer service | Support ticket rate | No increase > 5% |

### One-Sided vs. Two-Sided Testing for Guardrails

```
Primary Metric:   Two-sided test (detect improvement OR degradation)
Guardrail Metric: ONE-SIDED test (only care about degradation)

Why? One-sided tests for guardrails give you:
  • More power to detect harm (lower threshold for concern)
  • Clear decision: "did this get WORSE?"
  • α = 0.05 one-sided ≈ α = 0.10 two-sided in conservatism
```

### Guardrail Decision Framework

```
Primary ↑ AND Guardrails ✓  → SHIP
Primary ↑ AND Guardrails ✗  → INVESTIGATE (trade-off discussion)
Primary → AND Guardrails ✓  → DON'T SHIP (no benefit)
Primary ↓                    → DON'T SHIP (regardless of guardrails)
```

---

## Counter Metrics {#counter-metrics}

Counter metrics detect **unintended negative consequences** or gaming of the primary metric.

### Why Counter Metrics Matter

```
Scenario: "Increase user engagement"
  Primary: Daily Active Users (DAU)
  
  Without counter metrics:
    → Team adds annoying notifications
    → DAU increases (users open app to dismiss)
    → But satisfaction drops, long-term retention suffers
    
  With counter metrics:
    Counter: Notification opt-out rate
    Counter: App uninstall rate
    Counter: NPS/satisfaction score
    → Catches the harmful engagement pattern
```

### Counter Metric Examples by Scenario

| Optimization Goal | Counter Metric | What It Catches |
|------------------|----------------|-----------------|
| Increase clicks | Bounce rate, time-on-page | Clickbait content |
| Increase purchases | Return rate, support contacts | Misleading product info |
| Reduce load time | Content quality score | Over-aggressive compression |
| Increase sign-ups | Day-7 retention | Low-quality sign-ups |
| Increase ad revenue | User session length, DAU | Ad-heavy = users leave |

---

## Sensitivity vs. Specificity of Metrics {#sensitivity-vs-specificity}

### Metric Sensitivity

**Sensitivity** = ability to detect a real change (analogous to statistical power)

- High sensitivity: metric responds to many types of changes
- Low sensitivity: metric only moves with very large changes

### Metric Specificity

**Specificity** = ability to attribute movement to a specific cause

- High specificity: when metric moves, you know why
- Low specificity: metric moves for many reasons (noisy)

### The Sensitivity-Specificity Trade-off

```
                 High Specificity
                      │
    "Perfect metric"  │  "Narrow but clear"
    (rare in practice)│  (e.g., feature-specific
                      │   click rate)
    ──────────────────┼──────────────────── High Sensitivity
                      │
    "Broad but noisy" │  "Useless"
    (e.g., total      │  (insensitive AND
     revenue)         │   non-specific)
                      │
                 Low Specificity
```

| Metric | Sensitivity | Specificity | Best Use |
|--------|-------------|-------------|----------|
| Total revenue | Low | Low | North Star, not for experiments |
| Revenue per session | Medium | Medium | Good OEC for most experiments |
| Feature-specific CTR | High | High | Great for targeted experiments |
| NPS score | Low | Low | Long-term tracking only |
| Page load time | High | High | Performance guardrail |
| Bounce rate | Medium | Low | Diagnostic, not primary |

> **Critical Insight** 💡
> The ideal OEC has HIGH sensitivity (detects your change quickly with small N) and MEDIUM-HIGH specificity (you can trust that movement came from your treatment). Pure specificity isn't always needed if you have a well-randomized experiment.

---

## Composite Metrics {#composite-metrics}

### When to Combine Metrics

Composite metrics combine multiple signals into a single number when:
- No single metric captures the full user experience
- You want to avoid multiple testing correction penalties
- Metrics are correlated and redundant individually

### Types of Composite Metrics

```
1. WEIGHTED SUM
   OEC = w₁ × metric₁ + w₂ × metric₂ + ... + wₙ × metricₙ
   Example: Quality Score = 0.4 × relevance + 0.3 × freshness + 0.3 × diversity

2. SUCCESS RATE (binary composite)
   OEC = P(user achieves goal within session)
   Example: Search success = P(click AND time-on-page > 30s)

3. INDEX (normalized combination)
   OEC = (metric₁/baseline₁ + metric₂/baseline₂) / 2
   Example: Engagement Index = normalized(DAU) + normalized(session_length)
```

### Pitfalls of Composite Metrics

| Pitfall | Description | Mitigation |
|---------|-------------|------------|
| Weight sensitivity | Results change with different weights | Sensitivity analysis on weights |
| Interpretability | Hard to explain to stakeholders | Always decompose for reporting |
| Masking | One component improves, another degrades | Monitor components individually |
| Calibration | Components on different scales | Normalize before combining |

> **Critical Insight** 💡
> Microsoft's Bing team found that a well-designed composite OEC ("query success rate") was more sensitive than any individual metric. But the weights must be validated through offline analysis correlating with long-term outcomes — don't set them by intuition.

---

## Ratio Metrics and Their Pitfalls {#ratio-metrics}

### Common Ratio Metrics

- Conversion rate = purchases / visitors
- Click-through rate = clicks / impressions
- Revenue per user = total revenue / active users
- Average order value = total revenue / orders

### The Ratio Metric Problem

```
Metric: Revenue Per User = Total Revenue / Number of Users

Treatment causes:
  • Revenue increases by 5%
  • But also increases user count by 3% (more users sign up)
  
  Revenue Per User change = (1.05 / 1.03) - 1 ≈ 1.9%
  
  The denominator moved! This dilutes the numerator effect.
```

### Delta Method for Ratio Metric Variance

```python
# Variance of ratio metric Y/X using delta method:
# Var(Y/X) ≈ (1/μ_X²) × [Var(Y) - 2(μ_Y/μ_X)×Cov(X,Y) + (μ_Y/μ_X)²×Var(X)]

import numpy as np

def ratio_metric_variance(y_values, x_values):
    """Calculate variance of ratio Y/X using delta method."""
    mu_y = np.mean(y_values)
    mu_x = np.mean(x_values)
    var_y = np.var(y_values, ddof=1)
    var_x = np.var(x_values, ddof=1)
    cov_xy = np.cov(y_values, x_values, ddof=1)[0,1]
    
    ratio = mu_y / mu_x
    var_ratio = (1/mu_x**2) * (var_y - 2*ratio*cov_xy + ratio**2*var_x)
    return var_ratio / len(y_values)  # SE² of the mean ratio
```

### Ratio Metric Decision Guide

| Approach | When to Use | Advantage |
|----------|-------------|-----------|
| User-level average | Denominator is fixed (users in experiment) | Simple, no dilution |
| Delta method | Ratio of aggregates (revenue/sessions) | Accounts for covariance |
| Linearization | User-level approximation of ratio | Enables CUPED on ratios |
| Separate numerator/denominator | Debugging unexpected ratio movements | Identifies source of change |

---

## Noisy vs. Slow Metrics {#noisy-vs-slow}

### The Noise-Speed Spectrum

```
FAST but NOISY                              SLOW but CLEAN
←─────────────────────────────────────────────────────────→
Clicks     Sessions    Revenue    Retention    LTV    NPS
per page   per day     per week   (day 7)     (6mo)  (quarterly)

Easy to move │                │                │ Hard to move
High variance│                │                │ Low variance
Quick signal │                │                │ Best signal
```

### Metrics That Are Too Noisy

**Problem:** High variance means huge sample sizes needed.

| Noisy Metric | Why Noisy | Better Alternative |
|-------------|-----------|-------------------|
| Individual purchase amount | Power law distribution | Revenue per user (averaged) |
| Time between purchases | Long tail | Purchase frequency (bounded) |
| Raw page views | Heavy users dominate | Capped page views or binary engagement |
| Revenue (uncapped) | Whales skew everything | Winsorized revenue at 99th percentile |

**Variance reduction strategies:**
- Winsorization (cap at percentile)
- Log-transformation
- Binary conversion (did user do X? yes/no)
- CUPED adjustment

### Metrics That Are Too Slow

**Problem:** Effect only visible after weeks/months — can't run experiments efficiently.

| Slow Metric | Why Slow | Surrogate Proxy |
|-------------|----------|-----------------|
| 30-day retention | Takes 30 days to observe | Day-1 retention, session frequency in first week |
| Lifetime Value | Takes months/years | 7-day revenue, predicted LTV |
| Churn rate | Observed after absence period | Engagement decay slope |
| Annual revenue | 12-month window | Monthly run-rate |
| NPS | Infrequent surveys | In-product satisfaction signals |

> **Critical Insight** 💡
> The art of metric selection is finding the "Goldilocks metric" — fast enough to measure in your experiment window, but predictive enough of long-term outcomes. Teams at Netflix, Airbnb, and Microsoft invest heavily in validating that short-term proxies correlate with long-term North Star metrics.

---

## Practical Examples by Company {#practical-examples}

### Netflix: Content Recommendation Experiment

| Type | Metric | Rationale |
|------|--------|-----------|
| Primary | Streaming hours per member | Direct measure of engagement value |
| Secondary | Title-level take rate | Did recommendations lead to plays? |
| Secondary | Completion rate | Are users finishing recommended content? |
| Guardrail | Cancellation rate | Don't drive engagement that annoys |
| Counter | Content diversity consumed | Don't create filter bubbles |

### Uber/Lyft: Pricing Algorithm Change

| Type | Metric | Rationale |
|------|--------|-----------|
| Primary | Gross bookings per market-hour | Revenue accounting for ride volume |
| Secondary | Rider conversion rate | Do riders accept the price? |
| Secondary | Driver utilization rate | Are drivers staying busy? |
| Guardrail | Average wait time | Don't degrade rider experience |
| Guardrail | Driver earnings per hour | Don't hurt driver economics |
| Counter | Rider churn (30-day) | Short-term revenue vs. long-term retention |

### Search Engine (Google/Bing): Ranking Change

| Type | Metric | Rationale |
|------|--------|-----------|
| Primary | Session success rate | User found what they needed |
| Secondary | Clicks on top-3 results | Quality of top results |
| Secondary | Reformulation rate (lower = better) | Users don't need to re-search |
| Guardrail | Queries per session (if it rises, bad) | Don't frustrate users into re-querying |
| Counter | Ad revenue per search | Don't sacrifice monetization |
| Diagnostic | Position of first click | Did ranking improve? |

### E-commerce (Amazon): Product Page Redesign

| Type | Metric | Rationale |
|------|--------|-----------|
| Primary | Revenue per visitor | Direct business value |
| Secondary | Add-to-cart rate | Conversion funnel step |
| Secondary | Units per order | Cross-sell effectiveness |
| Guardrail | Page load time | Performance doesn't degrade |
| Guardrail | Return rate | Don't mislead buyers |
| Counter | Customer service contacts | Don't confuse users |

---

## Interview Talking Points {#interview-talking-points}

> **"How would you choose the primary metric for an experiment?"**
>
> "I follow a structured process. First, I identify what long-term outcome we ultimately care about — usually some form of retention or revenue. Then I look for a shorter-term proxy that (a) is causally connected to the feature change, (b) has been validated to correlate with the long-term outcome, and (c) has low enough variance to detect our expected effect size within a reasonable test duration. I always validate metric sensitivity by checking whether past experiments that shipped positive features actually moved this metric."

> **"What's the difference between a guardrail and a counter metric?"**
>
> "Guardrails are hard constraints — things like latency, error rates, or core revenue that must not degrade regardless of how much the primary improves. They represent non-negotiable user experience or business health. Counter metrics are softer — they detect unintended consequences or gaming. For example, if we optimize for sign-ups, a counter metric like day-7 retention catches the case where we're acquiring low-quality users. Guardrails block a launch; counter metrics trigger deeper investigation."

> **"A test shows your primary metric improved 3% but a secondary metric dropped 2%. What do you do?"**
>
> "First, I check if the secondary metric drop is statistically significant — with multiple metrics, some will move by chance. If significant, I investigate the mechanism. Does the secondary drop EXPLAIN the primary improvement (e.g., fewer searches because users found results faster — fewer searches is good here)? Or does it signal a hidden problem (e.g., engagement up but satisfaction down — unsustainable)? I'd decompose the metrics by user segment and look at the causal pathway before making a ship decision."

---

## Common Mistakes {#common-mistakes}

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Choosing a metric AFTER seeing results | Pre-register primary metric in experiment doc |
| Using too many primary metrics | ONE primary OEC, rest are secondary |
| Ignoring metric sensitivity | Validate with historical experiments or AA tests |
| Optimizing for clicks without quality signals | Add counter metrics for user satisfaction |
| Using total counts instead of per-user rates | Normalize by randomization unit |
| Testing on metric with different unit than randomization | User-randomized? Use user-level metrics |
| No guardrail metrics | Always include latency, errors, and revenue guardrails |
| Composite metric without component monitoring | Decompose and track all components |
| Ratio metric without controlling denominator | Use delta method or linearization |
| Choosing metric that's too slow to observe | Use validated short-term proxy |

---

## Rapid-Fire Q&A {#rapid-fire-qa}

**Q1: What is an OEC?**
> Overall Evaluation Criterion — the single primary metric that determines experiment success. Coined by Ronny Kohavi (Microsoft).

**Q2: Can you have multiple primary metrics?**
> Technically no. Multiple primaries create ambiguity when they disagree. Designate ONE primary and others as secondary, or create a composite OEC.

**Q3: What makes a metric "sensitive"?**
> A sensitive metric has low variance relative to the expected signal, meaning it can detect small effects with reasonable sample sizes. Validate by checking if past shipped changes moved it.

**Q4: How do you validate that a proxy metric predicts long-term outcomes?**
> Correlational analysis on historical data: do users with higher short-term metric values show better long-term outcomes? Also: do past experiments that moved the proxy also move long-term metrics when measured later?

**Q5: When would you use a composite metric?**
> When no single metric captures the full effect, when metrics are correlated (reducing multiple testing burden), or when the business goal is inherently multi-dimensional (e.g., "quality" combining relevance, freshness, diversity).

**Q6: Why are ratio metrics problematic in A/B tests?**
> The denominator can also be affected by treatment, introducing bias. The variance calculation requires the delta method. And individual-level ratios can be undefined (division by zero for users with zero sessions).

**Q7: What is a "Goldilocks metric"?**
> A metric that balances sensitivity (moves quickly enough to measure in test window) with validity (actually predicts long-term business outcomes). Not too noisy, not too slow.

**Q8: How do guardrail metrics differ in hypothesis testing?**
> Guardrails typically use one-sided tests (you only care about degradation). Some teams also use different alpha levels (e.g., α=0.01 for guardrails to be more conservative about blocking a launch).

**Q9: What is metric cannibalization?**
> When improving one metric inadvertently harms another. E.g., increasing email notifications boosts DAU but increases unsubscribe rate. Counter metrics catch this.

**Q10: How do you handle a metric that's significant in the wrong direction?**
> If primary is significantly negative: don't ship. If secondary: investigate mechanism. If guardrail: block ship and investigate. Always check for bugs before accepting unexpected negative results at face value.

---

## ASCII Cheat Sheet {#cheat-sheet}

```
╔══════════════════════════════════════════════════════════════════════╗
║            METRIC SELECTION FOR EXPERIMENTS CHEAT SHEET             ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  METRIC HIERARCHY:                                                   ║
║  ┌────────────────────────────────────────────────────────┐         ║
║  │  PRIMARY (OEC)    →  Ship/No-Ship decision             │         ║
║  │  SECONDARY        →  Supporting evidence + narrative   │         ║
║  │  GUARDRAIL        →  Hard constraints (must not cross) │         ║
║  │  COUNTER          →  Detect gaming / side effects      │         ║
║  │  DIAGNOSTIC       →  Explain WHY metric moved          │         ║
║  └────────────────────────────────────────────────────────┘         ║
║                                                                      ║
║  OEC SELECTION CRITERIA:                                             ║
║  ✓ Aligns with long-term business value                             ║
║  ✓ Measurable within experiment window                               ║
║  ✓ Sensitive to the specific change being tested                    ║
║  ✓ Low enough variance for feasible sample size                     ║
║  ✓ Not easily gameable                                               ║
║  ✓ Validated correlation with North Star metric                     ║
║                                                                      ║
║  SENSITIVITY vs. SPEED TRADE-OFF:                                    ║
║  Fast/Noisy ←──────────────────────────────→ Slow/Clean             ║
║  Clicks      Revenue/wk    Retention    LTV                         ║
║  (high var)  (moderate)    (30-day lag) (months)                    ║
║                                                                      ║
║  RATIO METRIC CHECKLIST:                                             ║
║  □ Is denominator affected by treatment?                            ║
║  □ Using delta method for variance?                                 ║
║  □ Monitoring numerator and denominator separately?                 ║
║  □ Handling zero-denominator users?                                 ║
║                                                                      ║
║  GUARDRAIL TESTING:                                                  ║
║  • Use ONE-SIDED tests (only detect degradation)                    ║
║  • Consider stricter α (0.01) for critical guardrails              ║
║  • Include: latency, errors, revenue, core engagement              ║
║                                                                      ║
║  VARIANCE REDUCTION FOR BETTER SENSITIVITY:                          ║
║  • CUPED (pre-experiment covariate adjustment)                      ║
║  • Winsorization (cap extreme values)                               ║
║  • Triggered analysis (only analyze exposed users)                  ║
║  • More granular metric (session-level vs. day-level)              ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 24 of 45*
