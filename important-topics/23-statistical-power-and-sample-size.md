# 🎯 Topic 23: Statistical Power and Sample Size

> *"How long should we run this test?" — the question every data scientist must answer with precision, not hand-waving.*

---

## Table of Contents

1. [Core Concepts: Power, Alpha, Beta](#core-concepts)
2. [The Four Pillars of Sample Size](#four-pillars)
3. [Effect Size Measures](#effect-size-measures)
4. [Sample Size Formulas](#sample-size-formulas)
5. [Minimum Detectable Effect (MDE)](#minimum-detectable-effect)
6. [Factors That Increase/Decrease Required N](#factors-affecting-n)
7. [Power Curves](#power-curves)
8. [Practical Calculator Walkthrough](#practical-calculator)
9. [How Long to Run This Test?](#how-long-to-run)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#cheat-sheet)

---

## Core Concepts: Power, Alpha, Beta {#core-concepts}

### The Hypothesis Testing Framework

| Term | Symbol | Definition | Typical Value |
|------|--------|-----------|---------------|
| Significance Level | α | P(reject H₀ \| H₀ true) — Type I error | 0.05 |
| Type II Error Rate | β | P(fail to reject H₀ \| H₁ true) | 0.20 |
| Statistical Power | 1 - β | P(reject H₀ \| H₁ true) — detecting a real effect | 0.80 |
| Confidence Level | 1 - α | P(fail to reject H₀ \| H₀ true) | 0.95 |

### The Decision Matrix

```
                    Reality
                 H₀ True    H₁ True
Decision  ┌──────────────┬──────────────┐
Reject H₀ │ Type I Error │  Correct!    │
           │    (α)       │  (Power)     │
           ├──────────────┼──────────────┤
Fail to    │  Correct!    │ Type II Error│
Reject H₀  │  (1 - α)     │    (β)       │
           └──────────────┴──────────────┘
```

> **Critical Insight** 💡
> Power = 1 - β is NOT a property of the test alone. It depends on the true effect size, sample size, variance, and alpha. You cannot say "my test has 80% power" without specifying what effect size you're trying to detect.

### Why 80% Power?

The 80% convention balances:
- Cost of Type I error (false positive → ship a bad feature)
- Cost of Type II error (false negative → miss a good feature)
- Practical constraints on sample size and test duration

For high-stakes decisions (e.g., removing a revenue feature), teams often use 90% or 95% power.

---

## The Four Pillars of Sample Size {#four-pillars}

Sample size is determined by **four interconnected quantities** — fix any three, and the fourth is determined:

```
┌─────────────────────────────────────────────┐
│         The Four Pillars                     │
│                                             │
│  1. Significance Level (α)                  │
│  2. Power (1 - β)                           │
│  3. Effect Size (Δ / σ)                     │
│  4. Sample Size (N)                         │
│                                             │
│  Fix any 3 → 4th is determined              │
└─────────────────────────────────────────────┘
```

> **Critical Insight** 💡
> In practice, α and power are typically fixed by convention (0.05 and 0.80). The real trade-off is between **effect size** (what's the smallest change worth detecting?) and **sample size** (how many users/how long?).

---

## Effect Size Measures {#effect-size-measures}

### Cohen's d (Continuous Outcomes)

```
         μ_treatment - μ_control
d = ─────────────────────────────────
            σ_pooled

where σ_pooled = √[(σ₁² + σ₂²) / 2]
```

| Cohen's d | Interpretation | Example |
|-----------|---------------|---------|
| 0.2 | Small | Moving avg session time from 5.0 to 5.1 min |
| 0.5 | Medium | Moving avg session time from 5.0 to 5.5 min |
| 0.8 | Large | Moving avg session time from 5.0 to 5.8 min |

### Cohen's h (Proportions)

```
h = 2 * arcsin(√p₁) - 2 * arcsin(√p₂)
```

Used when comparing two proportions (e.g., conversion rates).

### Relative vs. Absolute Effect Size

| Type | Formula | Example |
|------|---------|---------|
| Absolute | p₂ - p₁ | 5.2% - 5.0% = 0.2 pp |
| Relative | (p₂ - p₁) / p₁ | 0.2% / 5.0% = 4% lift |

> **Critical Insight** 💡
> A "2% lift" means very different things for different baselines. A 2% relative lift on a 50% conversion rate (1 pp absolute) requires far fewer samples than a 2% relative lift on a 1% conversion rate (0.02 pp absolute).

---

## Sample Size Formulas {#sample-size-formulas}

### For Two-Sample Proportion Test (Most Common in A/B Testing)

```
         (Z_{α/2} + Z_β)² × [p₁(1-p₁) + p₂(1-p₂)]
n = ──────────────────────────────────────────────────────
                    (p₂ - p₁)²

Where:
  n    = sample size PER GROUP
  Z_{α/2} = 1.96 for α = 0.05 (two-sided)
  Z_β  = 0.84 for power = 0.80
  p₁   = baseline conversion rate
  p₂   = expected conversion rate under treatment
```

**Simplified approximation** (when p₁ ≈ p₂ ≈ p):

```
         2 × (Z_{α/2} + Z_β)² × p(1-p)
n ≈ ─────────────────────────────────────────
              (p₂ - p₁)²
```

### For Two-Sample Means Test

```
         2 × (Z_{α/2} + Z_β)² × σ²
n = ────────────────────────────────────
              (μ₂ - μ₁)²

         2 × (Z_{α/2} + Z_β)²
n = ────────────────────────────────
              d²

Where d = Cohen's d = (μ₂ - μ₁) / σ
```

### Quick Reference Table

| α | Power | Z_{α/2} | Z_β | (Z_{α/2} + Z_β)² |
|---|-------|---------|-----|-------------------|
| 0.05 | 0.80 | 1.96 | 0.84 | 7.84 |
| 0.05 | 0.90 | 1.96 | 1.28 | 10.50 |
| 0.01 | 0.80 | 2.58 | 0.84 | 11.68 |
| 0.10 | 0.80 | 1.645 | 0.84 | 6.18 |

### Python Implementation

```python
import numpy as np
from scipy import stats
from statsmodels.stats.power import NormalIndPower, TTestIndPower
from statsmodels.stats.proportion import proportion_effectsize

# Method 1: Using statsmodels for proportions
p1 = 0.05  # baseline conversion rate (5%)
p2 = 0.055  # expected treatment rate (5.5%) — 10% relative lift

effect_size = proportion_effectsize(p1, p2)  # Cohen's h
analysis = NormalIndPower()
n_per_group = analysis.solve_power(
    effect_size=effect_size,
    alpha=0.05,
    power=0.80,
    alternative='two-sided'
)
print(f"Sample size per group: {int(np.ceil(n_per_group)):,}")

# Method 2: Manual calculation for proportions
z_alpha = stats.norm.ppf(1 - 0.05/2)  # 1.96
z_beta = stats.norm.ppf(0.80)          # 0.84
numerator = (z_alpha + z_beta)**2 * (p1*(1-p1) + p2*(1-p2))
denominator = (p2 - p1)**2
n_manual = int(np.ceil(numerator / denominator))
print(f"Manual calculation: {n_manual:,} per group")

# Method 3: For means (t-test)
cohen_d = 0.1  # small effect
analysis_t = TTestIndPower()
n_means = analysis_t.solve_power(
    effect_size=cohen_d,
    alpha=0.05,
    power=0.80,
    alternative='two-sided'
)
print(f"For Cohen's d = {cohen_d}: {int(np.ceil(n_means)):,} per group")
```

---

## Minimum Detectable Effect (MDE) {#minimum-detectable-effect}

MDE is the **smallest effect size** your experiment can reliably detect given your sample size, alpha, and power.

```
                   ┌─────────────────────────────────────────────┐
MDE (absolute) =   │  (Z_{α/2} + Z_β) × √(2 × p(1-p) / n)      │
                   └─────────────────────────────────────────────┘

MDE (relative) = MDE_absolute / baseline_rate
```

### Why MDE Matters More Than Sample Size

> **Critical Insight** 💡
> The real question isn't "how many samples do I need?" but "what's the smallest effect worth detecting?" If your MDE is a 0.1% lift on a metric, but business doesn't care about anything less than 2%, you're over-powering your test (wasting traffic/time).

### MDE Decision Framework

```
Step 1: What is the baseline metric value?
Step 2: What is the minimum lift that would justify shipping?
Step 3: Calculate required N for that MDE
Step 4: Can we get that N in a reasonable time frame?
  → YES: Run the test
  → NO: Can we increase MDE (accept only larger effects)?
     → YES: Adjust MDE, rerun calculation
     → NO: Consider alternative approaches (CUPED, stratification)
```

---

## Factors That Increase/Decrease Required N {#factors-affecting-n}

| Factor | Change | Effect on N | Why |
|--------|--------|-------------|-----|
| Smaller α (0.05 → 0.01) | ↓ α | ↑ N | Harder to cross higher threshold |
| Higher Power (80% → 90%) | ↑ power | ↑ N | Need more evidence to avoid Type II |
| Smaller effect size | ↓ Δ | ↑↑ N | Quadratic relationship! |
| Higher variance | ↑ σ² | ↑ N | More noise to overcome |
| One-sided → Two-sided | — | ↑ N | Splitting alpha across both tails |
| Variance reduction (CUPED) | ↓ σ² | ↓ N | Less noise = smaller N needed |
| Stratified randomization | ↓ σ² | ↓ N | Removes between-strata variance |
| Unequal allocation (70/30) | — | ↑ N | Less efficient than 50/50 |
| Metric with lower variance | — | ↓ N | Choose metrics wisely |

### The Quadratic Penalty

```
N ∝ 1 / (effect size)²

If effect size halves → N quadruples!

  MDE = 10% → N = 1,000
  MDE =  5% → N = 4,000
  MDE =  2% → N = 25,000
  MDE =  1% → N = 100,000
```

> **Critical Insight** 💡
> This quadratic relationship is why experienced data scientists push back on "Can we detect a 0.1% change?" — detecting tiny effects requires enormous sample sizes. Always translate sample size to test duration and ask whether the business can wait that long.

---

## Power Curves {#power-curves}

A power curve plots **power vs. effect size** for a given sample size (or power vs. sample size for a given effect size).

```python
import matplotlib.pyplot as plt
import numpy as np
from statsmodels.stats.power import NormalIndPower

analysis = NormalIndPower()
effect_sizes = np.linspace(0.01, 0.5, 100)

# Power curve for different sample sizes
for n in [100, 500, 1000, 5000]:
    powers = [analysis.solve_power(es, nobs1=n, alpha=0.05, power=None)
              for es in effect_sizes]
    plt.plot(effect_sizes, powers, label=f'n = {n:,}')

plt.axhline(y=0.8, color='r', linestyle='--', label='80% Power')
plt.xlabel('Effect Size (Cohen\'s h)')
plt.ylabel('Power')
plt.title('Power Curves by Sample Size')
plt.legend()
plt.grid(True)
plt.show()
```

### Reading a Power Curve

```
Power
1.0 │                          ___________
    │                     ____/
0.8 │- - - - - - - - ─ ─/─ ─ ─ ─ ─ ─ ─ ─    ← 80% power line
    │                 /
0.6 │               /
    │             /
0.4 │           /
    │         /
0.2 │       /
    │     /
0.0 │___/
    └──────────────────────────────────────→ Effect Size
         ↑                ↑
    Under-powered     Adequately powered
    (can't detect)     (can detect)
```

---

## Practical Calculator Walkthrough {#practical-calculator}

### Scenario: E-commerce Checkout Experiment

**Business context:** We want to test a new checkout flow. Current conversion rate is 3.2%. We want to detect a 5% relative improvement (3.2% → 3.36%).

```python
# Step 1: Define parameters
baseline_rate = 0.032      # 3.2% conversion
relative_lift = 0.05       # 5% relative improvement
expected_rate = baseline_rate * (1 + relative_lift)  # 3.36%
alpha = 0.05
power = 0.80

# Step 2: Calculate effect size
absolute_lift = expected_rate - baseline_rate  # 0.0016
print(f"Absolute lift: {absolute_lift*100:.2f} percentage points")
print(f"Relative lift: {relative_lift*100:.1f}%")

# Step 3: Calculate sample size
from statsmodels.stats.proportion import proportion_effectsize
from statsmodels.stats.power import NormalIndPower

effect_size = proportion_effectsize(baseline_rate, expected_rate)
analysis = NormalIndPower()
n_per_group = analysis.solve_power(
    effect_size=effect_size,
    alpha=alpha,
    power=power,
    alternative='two-sided'
)
n_per_group = int(np.ceil(n_per_group))
total_n = n_per_group * 2
print(f"\nRequired per group: {n_per_group:,}")
print(f"Total required: {total_n:,}")

# Step 4: Translate to duration
daily_traffic = 50000  # users entering checkout per day
days_needed = total_n / daily_traffic
print(f"\nWith {daily_traffic:,} daily users:")
print(f"Minimum duration: {int(np.ceil(days_needed))} days")
print(f"Recommended (full weeks): {int(np.ceil(days_needed/7))*7} days")
```

**Output:**
```
Absolute lift: 0.16 percentage points
Relative lift: 5.0%

Required per group: 296,652
Total required: 593,304

With 50,000 daily users:
Minimum duration: 12 days
Recommended (full weeks): 14 days
```

### Step 5: Sensitivity Check — What if We Can Only Run 7 Days?

```python
# With 7 days, what MDE can we achieve?
n_available = 50000 * 7 / 2  # per group for 7 days
mde = analysis.solve_power(
    effect_size=None,
    nobs1=n_available,
    alpha=0.05,
    power=0.80,
    alternative='two-sided'
)
# Convert effect size back to absolute lift
# (approximation for proportions)
mde_absolute = mde * np.sqrt(baseline_rate * (1-baseline_rate))
mde_relative = mde_absolute / baseline_rate
print(f"7-day MDE: {mde_relative*100:.1f}% relative lift")
```

---

## How Long to Run This Test? {#how-long-to-run}

### The Complete Framework

```
Duration = max(
    ceil(Total_N / Daily_Traffic),
    Minimum_for_day_of_week_effects (7 days),
    Minimum_for_novelty_effects (14-21 days),
    Business_cycle_coverage
)
```

### Duration Considerations

| Factor | Minimum Duration | Reason |
|--------|-----------------|--------|
| Day-of-week effects | 7 days (1 full week) | Traffic patterns vary by day |
| Pay cycle effects | 14-30 days | Spending varies by pay period |
| Novelty/primacy effects | 14-21 days | Initial behavior ≠ steady state |
| Monthly seasonality | 28-30 days | Some metrics have monthly cycles |
| Statistical power | Calculated above | Must reach required N |

> **Critical Insight** 💡
> Never run a test for less than one full week, even if you hit your sample size in 2 days. Day-of-week effects can create Simpson's paradox — what appears to be a treatment effect may just be differential exposure across days.

### Variance Reduction Techniques to Shorten Tests

| Technique | Typical Variance Reduction | Effect on Duration |
|-----------|---------------------------|-------------------|
| CUPED (pre-experiment data) | 20-50% | 20-50% shorter |
| Stratified randomization | 5-15% | 5-15% shorter |
| Winsorization (capping outliers) | 10-30% | 10-30% shorter |
| Triggered analysis (only exposed users) | Variable | Often dramatic |
| More sensitive metric | Variable | Depends on metric |

```python
# CUPED example: if variance reduction is 40%
variance_reduction = 0.40
n_with_cuped = n_per_group * (1 - variance_reduction)
days_with_cuped = (n_with_cuped * 2) / daily_traffic
print(f"With CUPED (40% reduction): {int(np.ceil(days_with_cuped))} days")
# vs 12 days without CUPED → ~7 days with CUPED
```

---

## Interview Talking Points {#interview-talking-points}

> **"Walk me through how you'd determine sample size for an A/B test."**
>
> "I start by anchoring on the business question: what's the minimum effect that would change our decision? For example, if we're testing a new recommendation algorithm and anything less than a 1% lift in revenue-per-session isn't worth the engineering cost to maintain, then 1% becomes my MDE. From there, I calculate the required sample size using the baseline metric value and its variance, standard alpha of 0.05, and 80% power. I always translate the sample size into calendar days — accounting for day-of-week effects by rounding up to full weeks — and then discuss with stakeholders whether that timeline is acceptable. If not, I explore variance reduction techniques like CUPED or consider whether we can accept detecting only larger effects."

> **"Your PM says 'let's just run it for a week and see.' How do you respond?"**
>
> "I'd first calculate what MDE we can actually detect in one week given our traffic. If our traffic supports detecting a meaningful effect in that timeframe, great. But often a week isn't enough — I'd show them: 'With our traffic, in one week we can only detect a 15% relative lift. Are you okay declaring 'no effect' if the true lift is 5%?' This usually reframes the conversation from 'how long' to 'what can we learn' within the given constraints."

> **"What's the relationship between power and sample size?"**
>
> "Power increases with sample size, but with diminishing returns — it follows a sigmoid curve. Going from 50% to 80% power might require doubling your sample, but going from 80% to 95% might require tripling it again. The key insight is that sample size scales with the *square* of the inverse effect size — halving your MDE quadruples the required N. This is why I always push teams to think carefully about the smallest practically meaningful effect rather than chasing arbitrary precision."

---

## Common Mistakes {#common-mistakes}

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Running test until significant (peeking) | Pre-calculate N and commit to it |
| Using total N instead of per-group N | Formulas give per-group N; multiply by # groups |
| Ignoring day-of-week effects | Run minimum 1 full week |
| Using one-sided test to reduce N | Two-sided unless strong prior justification |
| Assuming effect size without data | Use historical data or pilot experiments |
| Powering for relative lift without specifying baseline | Always specify baseline + absolute change |
| Forgetting that N scales quadratically with 1/MDE | Communicate this to stakeholders early |
| Ignoring clustering (if randomization unit ≠ analysis unit) | Inflate N by design effect factor |
| Post-hoc power analysis | Meaningless — use confidence intervals instead |
| Equal allocation when traffic is constrained | Consider 80/20 split with efficiency correction |

---

## Rapid-Fire Q&A {#rapid-fire-qa}

**Q1: What is statistical power?**
> Power = P(reject H₀ | H₁ is true) = 1 - β. It's the probability of detecting a real effect when one exists.

**Q2: If you double your sample size, does power double?**
> No. Power follows a sigmoid curve. Doubling N does NOT double power. The relationship is nonlinear — power approaches 1.0 asymptotically.

**Q3: What happens to required N if you want to detect half the effect size?**
> N quadruples. Sample size is proportional to 1/(effect_size)².

**Q4: What is MDE and why does it matter?**
> Minimum Detectable Effect — the smallest true effect your test can reliably detect. It defines what you CAN'T learn from the experiment, not just what you can.

**Q5: When would you use 90% power instead of 80%?**
> High-stakes decisions where missing a real effect is very costly — e.g., testing whether to remove a safety feature, or in pharmaceutical trials where patient health is at stake.

**Q6: What is post-hoc power analysis and why is it problematic?**
> Calculating power AFTER seeing results. It's circular — observed effect size determines post-hoc power, which adds no information beyond the p-value itself.

**Q7: How does CUPED reduce required sample size?**
> CUPED uses pre-experiment data to explain away variance, effectively reducing σ². Since N ∝ σ², a 40% variance reduction means 40% fewer samples needed.

**Q8: Why not just run every test with millions of users?**
> Over-powered tests detect effects too small to matter (statistical significance ≠ practical significance). Also, opportunity cost — that traffic could be used for other experiments.

**Q9: How do unequal allocation ratios affect power?**
> Unequal splits (e.g., 90/10) reduce effective sample size. Effective N = 4 * n₁ * n₂ / (n₁ + n₂). 50/50 maximizes power for a given total N.

**Q10: What's the minimum test duration regardless of sample size?**
> One full business cycle — typically one full week minimum to capture day-of-week variation. For metrics affected by pay cycles or monthly patterns, consider 2-4 weeks.

---

## ASCII Cheat Sheet {#cheat-sheet}

```
╔══════════════════════════════════════════════════════════════════════╗
║              STATISTICAL POWER & SAMPLE SIZE CHEAT SHEET            ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  POWER = 1 - β = P(detect effect | effect exists)                   ║
║                                                                      ║
║  ┌─────────────────────────────────────────────────────────┐        ║
║  │  SAMPLE SIZE FOR PROPORTIONS (per group):               │        ║
║  │                                                         │        ║
║  │       (Z_α/2 + Z_β)² × [p₁(1-p₁) + p₂(1-p₂)]        │        ║
║  │  n = ───────────────────────────────────────────────     │        ║
║  │                    (p₂ - p₁)²                           │        ║
║  └─────────────────────────────────────────────────────────┘        ║
║                                                                      ║
║  ┌─────────────────────────────────────────────────────────┐        ║
║  │  SAMPLE SIZE FOR MEANS (per group):                     │        ║
║  │                                                         │        ║
║  │       2 × (Z_α/2 + Z_β)² × σ²                         │        ║
║  │  n = ────────────────────────────────                    │        ║
║  │              (μ₂ - μ₁)²                                 │        ║
║  └─────────────────────────────────────────────────────────┘        ║
║                                                                      ║
║  KEY Z-VALUES:                                                       ║
║  ┌──────────────────────────────────────┐                           ║
║  │  α = 0.05 → Z_α/2 = 1.96           │                           ║
║  │  α = 0.01 → Z_α/2 = 2.576          │                           ║
║  │  Power 80% → Z_β = 0.842           │                           ║
║  │  Power 90% → Z_β = 1.282           │                           ║
║  └──────────────────────────────────────┘                           ║
║                                                                      ║
║  N SCALING RULES:                                                    ║
║  • N ∝ 1/(effect_size)²  — halve MDE → 4x the N                   ║
║  • N ∝ σ²               — double variance → 2x the N              ║
║  • N ∝ (Z_α/2 + Z_β)²  — 80%→90% power ≈ 1.34x the N            ║
║                                                                      ║
║  DURATION = max(N/daily_traffic, 7 days, novelty_buffer)            ║
║                                                                      ║
║  VARIANCE REDUCTION:                                                 ║
║  • CUPED: 20-50% reduction → 20-50% less time                      ║
║  • Stratification: 5-15% reduction                                  ║
║  • Winsorization: 10-30% reduction                                  ║
║                                                                      ║
║  EFFECT SIZE BENCHMARKS (Cohen's d):                                 ║
║  • Small:  d = 0.2  (N ≈ 394 per group)                            ║
║  • Medium: d = 0.5  (N ≈ 64 per group)                             ║
║  • Large:  d = 0.8  (N ≈ 26 per group)                             ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 23 of 45*
