# 🎯 Topic 55: Bootstrap Methods & Resampling

> *"The bootstrap is the Swiss army knife of statistics — when you can't derive the formula, simulate it. But like any tool, knowing when NOT to use it separates the expert from the novice."*

---

## 📚 Table of Contents

1. [What Is the Bootstrap?](#what-is-the-bootstrap)
2. [Why It Works: The Plug-In Principle](#why-it-works-the-plug-in-principle)
3. [Bootstrap Confidence Intervals](#bootstrap-confidence-intervals)
4. [Bootstrap Hypothesis Testing](#bootstrap-hypothesis-testing)
5. [Parametric vs Non-Parametric Bootstrap](#parametric-vs-non-parametric-bootstrap)
6. [When Bootstrap Fails](#when-bootstrap-fails)
7. [Bootstrap for A/B Testing](#bootstrap-for-ab-testing)
8. [Bootstrap for Complex Statistics](#bootstrap-for-complex-statistics)
9. [Comparison with Analytical Formulas](#comparison-with-analytical-formulas)
10. [Permutation Tests vs Bootstrap](#permutation-tests-vs-bootstrap)
11. [Python Implementation](#python-implementation)
12. [Interview Talking Points](#interview-talking-points)
13. [Common Mistakes](#common-mistakes)
14. [Rapid-Fire Q&A](#rapid-fire-qa)
15. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## What Is the Bootstrap?

The bootstrap is a **resampling method** that estimates the sampling distribution of a statistic by repeatedly sampling **with replacement** from the observed data.

### The Core Algorithm

```
Given: Original sample X = {x₁, x₂, ..., xₙ}
Goal: Estimate the distribution of statistic θ̂ = f(X)

For b = 1 to B (e.g., B = 10,000):
    1. Draw n observations WITH REPLACEMENT from X → X*_b
    2. Compute θ̂*_b = f(X*_b)

Result: {θ̂*₁, θ̂*₂, ..., θ̂*_B} approximates the sampling distribution of θ̂
```

### Key Properties of Bootstrap Samples

- Each bootstrap sample has the **same size** as the original (n observations)
- On average, each bootstrap sample contains **~63.2%** of the unique original observations
- The remaining **~36.8%** are duplicates (because P(not drawn in n draws) = (1-1/n)^n ≈ e^{-1})
- Different bootstrap samples will have different compositions

> **Critical Insight** 💡
> The bootstrap doesn't create new information — it extracts information about **sampling variability** that's already embedded in the data. It's estimating how much your statistic would change if you could re-run the entire data collection process.

---

## Why It Works: The Plug-In Principle

### The Fundamental Idea

The **empirical distribution function** (EDF) F̂ₙ — which puts probability 1/n on each observed data point — converges to the true population distribution F as n → ∞.

```
Population F  →  generates  →  Sample X  →  computes  →  θ = T(F)
     ↕ (approximate)
Empirical F̂ₙ →  generates  →  Bootstrap X* → computes → θ̂* = T(F̂ₙ)
```

**The logic**:
1. We can't sample from F (the population) — we only have one sample
2. F̂ₙ is our best estimate of F
3. The variability of θ̂* around θ̂ (from bootstrap) approximates the variability of θ̂ around θ (from repeated sampling)

### Mathematical Justification

Under regularity conditions, the bootstrap is **consistent**: as n → ∞, the bootstrap distribution converges to the true sampling distribution.

For smooth functionals T(F):
```
sup_x |P*(T(F̂ₙ*) ≤ x) - P(T(F̂ₙ) ≤ x)| → 0  almost surely
```

> **Critical Insight** 💡
> The bootstrap works because of the **Glivenko-Cantelli theorem**: the empirical CDF uniformly converges to the true CDF. This means resampling from your data asymptotically mimics sampling from the population. But this requires your sample to be a *reasonable representation* of the population — which is why bootstrap fails for extreme values and heavy tails.

---

## Bootstrap Confidence Intervals

### Method 1: Percentile Interval (Simplest)

```python
CI = [θ̂*_(α/2), θ̂*_(1-α/2)]

# 95% CI: take the 2.5th and 97.5th percentiles of bootstrap distribution
```

**Pros**: Simple, intuitive, respects natural parameter boundaries
**Cons**: Biased if θ̂ is biased; doesn't account for skewness properly

### Method 2: Basic (Pivotal) Interval

```python
CI = [2θ̂ - θ̂*_(1-α/2), 2θ̂ - θ̂*_(α/2)]

# "Reflects" the bootstrap distribution around the original estimate
```

**Pros**: First-order correct
**Cons**: Can fall outside natural parameter boundaries

### Method 3: BCa (Bias-Corrected and Accelerated) — Gold Standard

```python
# Adjusts for bias and skewness
# Uses two correction factors:
# z₀ = bias correction (how far bootstrap median is from θ̂)
# a  = acceleration (related to skewness, estimated via jackknife)

z₀ = Φ⁻¹(proportion of θ̂*_b < θ̂)
a  = (1/6) · Σ(θ̂₍₋ᵢ₎ - θ̂₍.₎)³ / [Σ(θ̂₍₋ᵢ₎ - θ̂₍.₎)²]^(3/2)

# Adjusted percentiles:
α₁ = Φ(z₀ + (z₀ + z_α/2)/(1 - a(z₀ + z_α/2)))
α₂ = Φ(z₀ + (z₀ + z_{1-α/2})/(1 - a(z₀ + z_{1-α/2})))

CI = [θ̂*_(α₁), θ̂*_(α₂)]
```

**Pros**: Second-order accurate, handles bias and skewness
**Cons**: More complex, requires jackknife for acceleration constant

### Comparison Table

| Method | Accuracy | Complexity | Best For |
|--------|----------|-----------|----------|
| Percentile | First-order | Simple | Symmetric, unbiased statistics |
| Basic | First-order | Simple | When you want boundary-free intervals |
| BCa | Second-order | Moderate | General use, skewed statistics |
| Bootstrap-t | Second-order | Complex | When variance estimation is feasible |
| Double bootstrap | Third-order | Very complex | Research (rarely practical) |

> **Critical Insight** 💡
> **Interview trap**: "Which bootstrap CI method should you use?" The default answer should be **BCa** — it's the most reliable general-purpose method. But if pressed: percentile is fine when the statistic is roughly symmetric and unbiased (like a sample mean). For ratio metrics, quantiles, or anything with potential skewness, BCa is essential.

---

## Bootstrap Hypothesis Testing

### Approach 1: Confidence Interval Inversion

```python
# If 0 is NOT in the bootstrap CI for (θ_treatment - θ_control),
# reject H₀: θ_treatment = θ_control at corresponding α level
```

### Approach 2: Direct Bootstrap Test

```python
# Under H₀: no difference between groups
# Resample under the null (centering or pooling)

# For two-sample test:
# 1. Pool both samples
# 2. Resample n₁ and n₂ from pool (with replacement)
# 3. Compute test statistic T*_b
# 4. p-value = proportion of T*_b ≥ T_observed
```

### Approach 3: Shift Method

```python
# Shift data to make H₀ true, then bootstrap
x_shifted = x - x.mean() + mu_0  # center at null value
# Bootstrap from x_shifted to get null distribution
```

---

## Parametric vs Non-Parametric Bootstrap

| Feature | Non-Parametric | Parametric |
|---------|---------------|------------|
| **Resampling from** | Original data points | Fitted parametric model |
| **Assumption** | Data is representative of population | Parametric model is correct |
| **When to use** | No strong distributional belief | Good reason to believe specific distribution |
| **Example** | Resample rows from data | Fit Normal(μ̂, σ̂²), generate from it |
| **Advantage** | Distribution-free | More efficient if model correct |
| **Risk** | Needs sufficient n | Wrong model → wrong everything |

### Parametric Bootstrap Example

```python
from scipy.stats import norm
mu_hat, sigma_hat = norm.fit(data)  # Fit parametric model
bootstrap_stats = [np.median(norm.rvs(loc=mu_hat, scale=sigma_hat, size=len(data)))
                   for _ in range(10000)]
```

> **Critical Insight** 💡
> **Parametric bootstrap** is particularly useful for **model diagnostics** and is the foundation of **parametric bootstrap LRT** used in mixed model testing (where standard LRT has boundary issues).

---

## When Bootstrap Fails

The bootstrap is **not universally valid**. Key failure modes:

1. **Extreme order statistics (min, max)**: Bootstrap can never exceed the observed max, so it cannot estimate the true sampling distribution of extremes.

2. **Heavy-tailed distributions (infinite variance)**: If Var(X) = infinity (Cauchy, Pareto with alpha <= 2), bootstrap variance estimates are inconsistent.

3. **Dependent data (time series, spatial)**: Standard i.i.d. bootstrap destroys temporal structure. Fix: block bootstrap (moving, circular, or stationary).

4. **Small sample sizes (n < 15-20)**: Empirical distribution too coarse, bootstrap CIs have severe undercoverage. Use exact or Bayesian methods.

5. **Boundary parameters**: Testing variance = 0 in mixed models — true parameter at boundary breaks bootstrap analog.

### Failure Mode Summary Table

| Scenario | Why It Fails | Fix |
|----------|-------------|-----|
| min/max statistics | Bootstrap bounded by observed data | Extreme value theory |
| Infinite variance | Inconsistent variance estimates | Subsampling or m-out-of-n bootstrap |
| Time series | Destroys dependence structure | Block bootstrap |
| Clustered data | Observations not independent | Cluster bootstrap (resample clusters) |
| Very small n (< 10) | EDF poor approximation | Exact methods, Bayesian |
| Boundary parameters | Null on boundary of space | Parametric bootstrap |

> **Critical Insight** 💡
> **Google's trick question**: "Can bootstrap create information that isn't in the data?" **No.** Bootstrap can only estimate variability of statistics that are **smooth functions** of the empirical distribution. It cannot extrapolate beyond observed data, which is why it fails for extreme quantiles, heavy tails, and situations where the phenomenon of interest is rare in the sample.

---

## Bootstrap for A/B Testing

### When to Use Bootstrap in A/B Tests

1. **Non-normal metrics**: Revenue, session duration (right-skewed)
2. **Ratio metrics**: Click-through rate = clicks/impressions (not a simple average)
3. **Complex statistics**: Percentiles (e.g., p95 latency), trimmed means
4. **Small samples**: Early readouts where CLT hasn't kicked in
5. **Variance reduction validation**: Checking CUPED/regression adjustment assumptions

### Bootstrap A/B Test Implementation

```python
import numpy as np

def bootstrap_ab_test(control, treatment, stat_fn=np.mean, B=10000, alpha=0.05):
    """
    Bootstrap test for difference in stat_fn between treatment and control.
    """
    observed_diff = stat_fn(treatment) - stat_fn(control)
    
    bootstrap_diffs = []
    for _ in range(B):
        ctrl_boot = np.random.choice(control, size=len(control), replace=True)
        treat_boot = np.random.choice(treatment, size=len(treatment), replace=True)
        bootstrap_diffs.append(stat_fn(treat_boot) - stat_fn(ctrl_boot))
    
    bootstrap_diffs = np.array(bootstrap_diffs)
    
    # BCa or percentile CI
    ci_lower = np.percentile(bootstrap_diffs, 100 * alpha/2)
    ci_upper = np.percentile(bootstrap_diffs, 100 * (1 - alpha/2))
    
    # p-value (two-sided): proportion of bootstrap diffs crossing zero
    p_value = 2 * min(
        np.mean(bootstrap_diffs <= 0),
        np.mean(bootstrap_diffs >= 0)
    )
    
    return {
        'observed_diff': observed_diff,
        'ci': (ci_lower, ci_upper),
        'p_value': p_value,
        'se': np.std(bootstrap_diffs)
    }
```

### Bootstrap for Ratio Metrics (Delta Method Alternative)

```python
# Revenue per user = total_revenue / total_users
# Can't just bootstrap individual user revenues if some users have 0 sessions

def bootstrap_ratio(numerator, denominator, B=10000):
    """Bootstrap confidence interval for a ratio metric."""
    n = len(numerator)
    observed_ratio = numerator.sum() / denominator.sum()
    
    ratios = []
    for _ in range(B):
        idx = np.random.randint(0, n, size=n)
        ratio_boot = numerator[idx].sum() / denominator[idx].sum()
        ratios.append(ratio_boot)
    
    return np.percentile(ratios, [2.5, 97.5])
```

> **Critical Insight** 💡
> At Google, the bootstrap is used to **validate** analytical formulas, not replace them. For daily A/B test decisions at scale (thousands of experiments), bootstrap is too slow. Instead, they derive analytical formulas (delta method for ratios, etc.) and use bootstrap to verify those formulas have correct coverage on historical data. Bootstrap is the **referee**, not the **player**.

---

## Bootstrap for Complex Statistics

```python
# Median SE (no simple analytical formula for small n)
medians = [np.median(np.random.choice(data, len(data), replace=True)) 
           for _ in range(10000)]
se_median = np.std(medians)

# Quantiles (e.g., p95 latency — notoriously hard analytically)
p95s = [np.percentile(np.random.choice(data, len(data), replace=True), 95)
        for _ in range(10000)]
ci_p95 = np.percentile(p95s, [2.5, 97.5])

# Correlation (works for any joint distribution, no bivariate normality needed)
def bootstrap_correlation(x, y, B=10000):
    n = len(x)
    corrs = []
    for _ in range(B):
        idx = np.random.randint(0, n, size=n)
        corrs.append(np.corrcoef(x[idx], y[idx])[0, 1])
    return np.percentile(corrs, [2.5, 97.5])
```

The bootstrap handles **any** statistic (difference of medians, ratio of variances, Gini coefficient) — no need to derive custom formulas. This is its superpower.

---

## Comparison with Analytical Formulas

| Criterion | Analytical (CLT-based) | Bootstrap |
|-----------|----------------------|-----------|
| **Speed** | Instantaneous | O(B × n) computation |
| **Assumptions** | Often assumes normality or large n | Assumes i.i.d. and sufficient n |
| **Accuracy for mean** | Excellent (CLT works fast) | Same or slightly worse |
| **Accuracy for median** | Poor for small n | Good |
| **Accuracy for quantiles** | Very limited theory | Good (but needs large n) |
| **Ratio metrics** | Delta method (approximate) | Exact (to simulation error) |
| **Interpretability** | Familiar to stakeholders | "We simulated it" needs explanation |
| **Production systems** | Easy to implement at scale | Expensive at scale |

### When to Use Which

| Use Analytical When... | Use Bootstrap When... |
|----------------------|---------------------|
| Statistic is a mean or proportion | Statistic is median, quantile, ratio |
| n > 30 and distribution isn't extreme | Distribution is skewed or heavy-tailed |
| Need real-time computation (production) | Offline analysis (can afford computation) |
| Formula exists and is validated | No formula exists or formula assumptions violated |
| Communicating to non-technical stakeholders | Technical audience comfortable with simulation |

> **Critical Insight** 💡
> A common interview mistake: using bootstrap for a **sample mean** when n = 10,000. The CLT already gives an excellent approximation for the mean at that sample size. Bootstrap adds computational cost with no practical benefit. Reserve bootstrap for statistics where analytical formulas don't exist or don't work well (medians, quantiles, ratios of small counts).

---

## Permutation Tests vs Bootstrap

| Feature | Permutation Test | Bootstrap |
|---------|-----------------|-----------|
| **Purpose** | Hypothesis testing (is there a difference?) | Estimation (CI, SE, distribution) |
| **Null hypothesis** | Exchangeability (groups are identical) | None built-in (can construct tests) |
| **Resampling scheme** | Shuffle labels WITHOUT replacement | Resample WITH replacement |
| **What it estimates** | Exact p-value under H₀ | Sampling distribution of statistic |
| **Preserves** | Marginal distributions exactly | Approximate marginal distributions |
| **Assumptions** | Exchangeability under H₀ | i.i.d. observations |
| **Can estimate CIs?** | Not directly | Yes (primary use) |
| **Can estimate SE?** | Not directly | Yes |

### When to Use Each

```
"Is there a significant difference?" → Permutation test
"How big is the difference, and how certain?" → Bootstrap CI
"Both?" → Bootstrap CI that excludes 0 gives both
```

### Permutation Test Implementation

```python
def permutation_test(group_a, group_b, stat_fn=lambda a, b: np.mean(a) - np.mean(b), 
                     n_perms=10000):
    """Two-sample permutation test."""
    observed = stat_fn(group_a, group_b)
    combined = np.concatenate([group_a, group_b])
    n_a = len(group_a)
    
    count_extreme = 0
    for _ in range(n_perms):
        np.random.shuffle(combined)
        perm_stat = stat_fn(combined[:n_a], combined[n_a:])
        if abs(perm_stat) >= abs(observed):
            count_extreme += 1
    
    return count_extreme / n_perms  # two-sided p-value
```

---

## Python Implementation

### Complete Bootstrap with BCa

```python
import numpy as np
from scipy.stats import norm

def bootstrap_ci(data, stat_fn=np.median, B=10000, alpha=0.05, method='bca'):
    """Bootstrap CI with percentile or BCa method."""
    n = len(data)
    theta_hat = stat_fn(data)
    
    # Generate bootstrap distribution
    boot_stats = np.array([
        stat_fn(np.random.choice(data, size=n, replace=True))
        for _ in range(B)
    ])
    
    if method == 'percentile':
        return np.percentile(boot_stats, [100*alpha/2, 100*(1-alpha/2)])
    
    # BCa: bias correction
    z0 = norm.ppf(np.mean(boot_stats < theta_hat))
    
    # BCa: acceleration (jackknife)
    jack_stats = np.array([stat_fn(np.delete(data, i)) for i in range(n)])
    jack_mean = jack_stats.mean()
    num = np.sum((jack_mean - jack_stats) ** 3)
    den = 6 * (np.sum((jack_mean - jack_stats) ** 2) ** 1.5)
    a = num / den if den != 0 else 0
    
    # Adjusted percentiles
    z_lo, z_hi = norm.ppf(alpha/2), norm.ppf(1-alpha/2)
    a1 = norm.cdf(z0 + (z0 + z_lo) / (1 - a*(z0 + z_lo)))
    a2 = norm.cdf(z0 + (z0 + z_hi) / (1 - a*(z0 + z_hi)))
    
    return np.percentile(boot_stats, [100*a1, 100*a2])

# Usage
data = np.random.exponential(5, size=200)
print(f"95% BCa CI for median: {bootstrap_ci(data, np.median)}")
print(f"Bootstrap SE: {np.std([np.median(np.random.choice(data, len(data), True)) for _ in range(10000)]):.3f}")
```

### Vectorized Bootstrap (Fast)

```python
def fast_bootstrap_mean(data, B=10000, alpha=0.05):
    """Vectorized bootstrap for the mean (orders of magnitude faster)."""
    n = len(data)
    indices = np.random.randint(0, n, size=(B, n))
    boot_means = data[indices].mean(axis=1)
    return np.percentile(boot_means, [100*alpha/2, 100*(1-alpha/2)])
```

---

## Interview Talking Points

> **"Explain bootstrap to me like I'm a product manager."**
>
> "Imagine you want to know how reliable your metric is — say, average revenue per user is $4.50. How confident should you be in that number? Ideally, you'd run the experiment 10,000 times. You can't do that. But you can *simulate* it: randomly resample from your existing data (with repeats allowed) 10,000 times, compute the metric each time, and see how much it varies. The spread of those 10,000 estimates tells you your uncertainty. If they range from $4.20 to $4.80, that's your confidence interval."

> **"Can bootstrap reduce variance?" (Google's trick question)**
>
> "No. Bootstrap doesn't reduce the variance of your estimator — it *estimates* the variance. Your point estimate θ̂ stays the same regardless of whether you bootstrap or not. Bootstrap tells you how much θ̂ would wiggle if you collected new data. This is a common confusion: people think more bootstrap iterations (larger B) reduces variance of the *original estimate*. It doesn't. It only reduces *Monte Carlo error* in approximating the bootstrap distribution."

> **"When would you NOT use bootstrap?"**
>
> "Five cases: First, when the statistic depends on extreme values — like the maximum — because bootstrap can never exceed the observed max. Second, heavy-tailed distributions with possibly infinite variance, where bootstrap SE doesn't converge. Third, time series data where observations aren't independent — need block bootstrap instead. Fourth, very small samples (n < 15) where the empirical distribution is too coarse. Fifth, when a proven analytical formula exists and is computationally cheaper — bootstrap for a mean with n=100,000 is wasteful."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Sampling WITHOUT replacement (that's a permutation test!) | Always sample WITH replacement for bootstrap |
| Using too few bootstrap iterations (B=100) | Use B >= 10,000 for CIs; B >= 1,000 for SE only |
| Thinking more bootstrap samples improves the original estimate | More B only reduces Monte Carlo error in variance estimation |
| Bootstrap on dependent data (time series) without modification | Use block bootstrap or residual bootstrap for time series |
| Reporting percentile CI when statistic is clearly biased | Use BCa interval which corrects for bias |
| Bootstrap on n=8 and trusting the CI coverage | With very small n, bootstrap CIs have poor coverage; consider exact methods |
| Bootstrapping individual observations in clustered data | Bootstrap at the cluster level (resample entire clusters) |
| "Bootstrap can estimate any population parameter" | Bootstrap fails for extremes, boundary parameters, non-smooth functionals |
| Not setting a random seed for reproducibility | Always set seed; reviewers need to reproduce your results |
| Treating bootstrap p-value as exact | It's a Monte Carlo estimate; report it with the simulation SE |

---

## Rapid-Fire Q&A

**Q1: Can bootstrap reduce variance? (TRICK QUESTION)**
> No. Bootstrap estimates variance — it doesn't reduce it. The original estimator θ̂ is unchanged. Increasing B only reduces Monte Carlo approximation error.

**Q2: Why sample WITH replacement?**
> Without replacement, every resample would be identical to the original data (just reordered). With replacement creates variability in sample composition, which is what drives the variance estimate.

**Q3: How many bootstrap iterations (B) do you need?**
> For standard errors: B = 1,000 is usually sufficient. For confidence intervals: B = 10,000+. For tail probabilities (p-values): B = 50,000+ because you need resolution in the tails.

**Q4: Is the bootstrap CI always symmetric around the point estimate?**
> No! That's the whole point. Percentile and BCa intervals can be asymmetric, correctly reflecting skewness in the sampling distribution. This is an advantage over normal-approximation CIs.

**Q5: What's the "double bootstrap"?**
> Bootstrap the bootstrap — for each bootstrap sample, run another bootstrap to estimate its variance. Used to calibrate CI coverage. Rarely practical (B² computations) but theoretically elegant.

**Q6: Can you bootstrap for prediction intervals (not confidence intervals)?**
> Yes, but you must also add the residual noise. A bootstrap CI for the mean captures estimation uncertainty; a prediction interval must also include observation-level variance.

**Q7: Bootstrap vs jackknife — when to prefer jackknife?**
> Jackknife (leave-one-out) is faster for estimating bias and computing the acceleration constant in BCa. But it gives poor SE estimates for non-smooth statistics (median, quantiles). Bootstrap is generally preferred.

**Q8: How does the bootstrap relate to Bayesian methods?**
> The non-parametric bootstrap is approximately equivalent to a Bayesian posterior under a non-informative Dirichlet prior on the data distribution. The "Bayesian bootstrap" makes this connection explicit by assigning random Dirichlet weights.

**Q9: For an A/B test with 1M users per group, should you bootstrap?**
> Probably not for the primary metric (mean). CLT gives a great approximation at that scale, and bootstrap would be computationally expensive. But bootstrap is still valuable for complex metrics (p95 latency) or validating analytical approximations.

**Q10: What's the m-out-of-n bootstrap?**
> Instead of resampling n observations, resample m < n. This fixes bootstrap failures in cases like heavy tails or non-standard rates of convergence. The tradeoff: wider CIs (less efficiency) but correct coverage.

---

## ASCII Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║              BOOTSTRAP METHODS — QUICK REFERENCE                     ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  THE ALGORITHM:                                                      ║
║  ┌─────────┐    ┌──────────────────────┐    ┌─────────────┐        ║
║  │Original │───▶│Resample WITH replace │───▶│ Compute θ̂*  │        ║
║  │ Sample  │    │(same size n)         │    │ (B times)   │        ║
║  │ n obs   │    └──────────────────────┘    └──────┬──────┘        ║
║  └─────────┘                                       │               ║
║                                      ┌─────────────▼─────────────┐ ║
║                                      │ Bootstrap Distribution:    │ ║
║                                      │ SE = sd(θ̂*₁,...,θ̂*_B)     │ ║
║                                      │ CI = percentiles of θ̂*    │ ║
║                                      └───────────────────────────┘ ║
║                                                                      ║
║  CI METHODS (ranked by reliability):                                 ║
║  ┌────────────────────────────────────────────────────┐             ║
║  │ 1. BCa (bias-corrected accelerated) ← DEFAULT     │             ║
║  │ 2. Percentile (simple, OK for symmetric stats)     │             ║
║  │ 3. Basic/Pivotal (can violate boundaries)          │             ║
║  │ 4. Normal approx (θ̂ ± z·SE_boot) ← worst         │             ║
║  └────────────────────────────────────────────────────┘             ║
║                                                                      ║
║  BOOTSTRAP vs PERMUTATION:                                           ║
║  ┌─────────────────────┬──────────────────────────┐                 ║
║  │    BOOTSTRAP        │     PERMUTATION          │                 ║
║  │ WITH replacement    │ WITHOUT replacement      │                 ║
║  │ Estimates SE, CI    │ Tests H₀ (exact p-value) │                 ║
║  │ No null assumed     │ Assumes exchangeability   │                 ║
║  └─────────────────────┴──────────────────────────┘                 ║
║                                                                      ║
║  WHEN IT FAILS:                                                      ║
║  ✗ Extreme values (min, max)     → Use extreme value theory         ║
║  ✗ Heavy tails (infinite var)    → Use m-out-of-n bootstrap         ║
║  ✗ Time series / spatial         → Use block bootstrap              ║
║  ✗ Clustered data                → Bootstrap entire clusters        ║
║  ✗ n < 15                        → Exact or Bayesian methods        ║
║                                                                      ║
║  KEY NUMBERS:                                                        ║
║  • ~63.2% unique obs per bootstrap sample                            ║
║  • B ≥ 10,000 for CIs                                               ║
║  • B ≥ 1,000 for SE                                                  ║
║  • Bootstrap does NOT reduce variance (only estimates it!)           ║
║                                                                      ║
║  PYTHON ONE-LINER:                                                   ║
║  from scipy.stats import bootstrap                                   ║
║  res = bootstrap((data,), np.median, n_resamples=9999)              ║
║  print(res.confidence_interval)                                      ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 55 of 56*
