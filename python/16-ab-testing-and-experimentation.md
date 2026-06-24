# 🎯 Topic 16: A/B Testing & Experimentation

> A comprehensive guide to designing, running, and analyzing controlled experiments — covering hypothesis testing, power analysis, variance reduction, sequential testing, Bayesian methods, and the practical gotchas that separate senior practitioners from beginners. This is the single most important topic for Meta, Google, Netflix, and Uber DS interviews.

---

## Table of Contents

1. [Hypothesis Formulation](#1-hypothesis-formulation)
2. [Type I and Type II Errors](#2-type-i-and-type-ii-errors)
3. [Sample Size Calculation & Power Analysis](#3-sample-size-calculation--power-analysis)
4. [Z-Test for Proportions](#4-z-test-for-proportions)
5. [T-Test for Means](#5-t-test-for-means)
6. [Multiple Testing Correction](#6-multiple-testing-correction)
7. [Experiment Design Principles](#7-experiment-design-principles)
8. [Novelty Effects](#8-novelty-effects)
9. [Network Effects & SUTVA Violations](#9-network-effects--sutva-violations)
10. [Bayesian A/B Testing](#10-bayesian-ab-testing)
11. [Sequential Testing](#11-sequential-testing)
12. [CUPED Variance Reduction](#12-cuped-variance-reduction)
13. [Metric Selection](#13-metric-selection)
14. [Sample Ratio Mismatch (SRM)](#14-sample-ratio-mismatch-srm)
15. [Practical Experiment Gotchas at Scale](#15-practical-experiment-gotchas-at-scale)
16. [Interview Talking Points & Scripts](#16-interview-talking-points--scripts)
17. [Common Interview Mistakes](#17-common-interview-mistakes)
18. [Rapid-Fire Q&A](#18-rapid-fire-qa)
19. [Summary Cheat Sheet](#19-summary-cheat-sheet)

---

## 1. Hypothesis Formulation

Every experiment begins with a clear hypothesis. The hypothesis must be **specific**, **measurable**, and **falsifiable**.

**Null Hypothesis (H₀):** There is no difference between treatment and control.  
**Alternative Hypothesis (H₁):** There is a difference (two-tailed) or a directional change (one-tailed).

| Aspect | One-Tailed | Two-Tailed |
|--------|-----------|------------|
| H₁ direction | Specified (e.g., μ_T > μ_C) | Any direction (μ_T ≠ μ_C) |
| Alpha allocation | All α in one tail | α/2 in each tail |
| Power | Higher for correct direction | Lower, but catches both directions |
| When to use | Strong prior belief in direction | Default choice in industry |
| Industry preference | Rare (Netflix uses for degradation) | Standard at Meta, Google, Uber |

> 💡 **Critical Insight:** In interviews, always default to two-tailed tests unless you explicitly state why one-tailed is justified. Saying "I'd use a one-tailed test because I expect the metric to go up" is a red flag — the whole point of testing is that you might be wrong. One-tailed tests are appropriate when you only care about detecting degradation (guardrail metrics) or when there's a strong theoretical basis for directionality.

---

## 2. Type I and Type II Errors

| | H₀ is True (No Effect) | H₀ is False (Real Effect) |
|---|---|---|
| **Reject H₀** | Type I Error (α) — False Positive | Correct Decision (Power = 1-β) |
| **Fail to Reject H₀** | Correct Decision | Type II Error (β) — False Negative |

**Key relationships:**
- **α (significance level):** Probability of a false positive. Industry standard: 0.05.
- **β:** Probability of a false negative. Typically 0.20.
- **Power (1-β):** Probability of detecting a real effect. Target: 0.80.
- **Trade-off:** Decreasing α increases β (harder to detect real effects).

> 💡 **Critical Insight:** At companies running thousands of experiments (Meta runs ~10,000/year), even a 5% false positive rate means ~500 false launches. This is why companies invest heavily in multiple testing corrections and meta-analysis of past experiments.

---

## 3. Sample Size Calculation & Power Analysis

Sample size depends on four factors: significance level (α), power (1-β), minimum detectable effect (MDE), and baseline variance.

**Formula for proportions:**

n = (Z_{α/2} + Z_β)² × [p₁(1-p₁) + p₂(1-p₂)] / (p₁ - p₂)²

```python
import numpy as np
from statsmodels.stats.power import TTestIndPower, NormalIndPower
from statsmodels.stats.proportion import proportion_effectsize

def sample_size_proportions(baseline_rate, mde_relative, alpha=0.05, power=0.80):
    """Calculate sample size per group for a two-proportion z-test."""
    new_rate = baseline_rate * (1 + mde_relative)
    effect_size = proportion_effectsize(baseline_rate, new_rate)
    analysis = NormalIndPower()
    n = analysis.solve_power(effect_size=effect_size, alpha=alpha,
                             power=power, alternative='two-sided')
    return int(np.ceil(n))

# Example: 10% baseline CTR, detect 5% relative lift → ~31,000 per group
n = sample_size_proportions(0.10, 0.05)
print(f"Sample size per group: {n:,}")

def sample_size_means(baseline_mean, baseline_std, mde_relative, alpha=0.05, power=0.80):
    """Calculate sample size per group for a two-sample t-test."""
    effect_size = (baseline_mean * mde_relative) / baseline_std  # Cohen's d
    analysis = TTestIndPower()
    n = analysis.solve_power(effect_size=effect_size, alpha=alpha,
                             power=power, alternative='two-sided')
    return int(np.ceil(n))

# Example: Revenue/user mean=$50, std=$80, detect 2% lift
n = sample_size_means(50, 80, 0.02)
print(f"Sample size per group: {n:,}")
```

> 💡 **Critical Insight:** The #1 mistake in experiment design is choosing an MDE that's too small. If your product change is expected to move a metric by 1%, don't power for 0.1% — you'll need an impossibly large sample. Always ask: "What's the smallest effect that would be worth shipping?" That's your MDE.

---

## 4. Z-Test for Proportions

Used when comparing conversion rates, CTRs, or any binary outcome between groups.

```python
import numpy as np
from scipy import stats

def two_proportion_z_test(successes_c, n_c, successes_t, n_t):
    """Two-proportion z-test with confidence interval."""
    p_c, p_t = successes_c / n_c, successes_t / n_t
    # Pooled proportion under H0
    p_pool = (successes_c + successes_t) / (n_c + n_t)
    se = np.sqrt(p_pool * (1 - p_pool) * (1/n_c + 1/n_t))
    z = (p_t - p_c) / se
    p_value = 2 * (1 - stats.norm.cdf(abs(z)))
    # CI for the difference (unpooled SE)
    se_diff = np.sqrt(p_c*(1-p_c)/n_c + p_t*(1-p_t)/n_t)
    ci = ((p_t - p_c) - 1.96*se_diff, (p_t - p_c) + 1.96*se_diff)
    return {'lift': (p_t-p_c)/p_c, 'z': z, 'p_value': p_value, 'ci_95': ci,
            'significant': p_value < 0.05}

# Example: New checkout flow (9.8% vs 10.3% conversion)
results = two_proportion_z_test(4900, 50000, 5150, 50000)
print(results)
```

---

## 5. T-Test for Means

| Scenario | Test | When to Use |
|----------|------|-------------|
| Large n (>30), known σ | Z-test | Rarely in practice |
| Comparing two group means | Two-sample t-test | Revenue, session duration, pages/visit |
| Paired observations | Paired t-test | Before/after on same users |
| Unequal variances | Welch's t-test | Default — always use Welch's |

```python
from scipy import stats
import numpy as np

def welch_t_test(control_data, treatment_data):
    """Welch's t-test — does NOT assume equal variances. Default choice."""
    t_stat, p_value = stats.ttest_ind(treatment_data, control_data, equal_var=False)
    pooled_std = np.sqrt((np.std(control_data, ddof=1)**2 + 
                          np.std(treatment_data, ddof=1)**2) / 2)
    cohens_d = (np.mean(treatment_data) - np.mean(control_data)) / pooled_std
    return {'lift': (np.mean(treatment_data)-np.mean(control_data))/np.mean(control_data),
            't_stat': t_stat, 'p_value': p_value, 'cohens_d': cohens_d}
```

> 💡 **Critical Insight:** Always use Welch's t-test (equal_var=False). The classic Student's t-test assumes equal variances — an assumption that's almost never exactly true and can lead to inflated false positives when violated. Welch's is robust and has no downside.

---

## 6. Multiple Testing Correction

When testing multiple metrics or variants, the family-wise error rate (FWER) inflates.

| Method | Controls | Formula | When to Use |
|--------|----------|---------|-------------|
| Bonferroni | FWER | α_adj = α / m | Few comparisons, conservative |
| Holm (step-down) | FWER | Ordered p-values, adaptive α | Default for FWER control |
| Benjamini-Hochberg | FDR | Ranked p-values × m/rank | Many metrics, exploratory |
| Sidak | FWER | 1-(1-α)^(1/m) | Independent tests |

```python
from statsmodels.stats.multitest import multipletests

p_values = [0.03, 0.04, 0.12, 0.001, 0.08]

# Bonferroni, Holm (uniformly more powerful), BH-FDR
_, pvals_bonf, _, _ = multipletests(p_values, method='bonferroni', alpha=0.05)
_, pvals_holm, _, _ = multipletests(p_values, method='holm', alpha=0.05)
_, pvals_bh, _, _   = multipletests(p_values, method='fdr_bh', alpha=0.05)

for raw, bonf, holm, bh in zip(p_values, pvals_bonf, pvals_holm, pvals_bh):
    print(f"Raw: {raw:.3f} | Bonf: {bonf:.3f} | Holm: {holm:.3f} | BH: {bh:.3f}")
```

> 💡 **Critical Insight:** In interviews, mention that you'd use Holm (not Bonferroni) for FWER control because Holm is uniformly more powerful and there's no reason to use Bonferroni. For exploratory analysis with many secondary metrics, BH-FDR is appropriate. Google and Meta both use hierarchical approaches — pre-specify your primary metric to avoid correction on it.

---

## 7. Experiment Design Principles

**Randomization Unit:** The entity being randomly assigned to treatment/control.

| Unit | Examples | Tradeoffs |
|------|----------|-----------|
| User-level | Most web experiments | Standard; avoids inconsistent experience |
| Session-level | Rare; testing session features | Can give inconsistent experience across sessions |
| Page-level | Ad auctions | Fast iteration; user sees both variants |
| Geo/Cluster | Uber surge pricing, marketplace changes | Fewer units → less power; handles network effects |
| Device-level | Cross-platform features | User may have multiple devices |

**Key Design Decisions:**
1. **Randomization unit** — Must match the level at which the treatment operates
2. **Analysis unit** — Can differ from randomization unit (e.g., randomize by user, analyze by session)
3. **Duration** — Must capture full business cycles (weekday/weekend, pay cycles)
4. **Traffic allocation** — Start small (1-5%), ramp up after SRM check passes
5. **Holdout groups** — Long-running control for measuring cumulative effect

---

## 8. Novelty Effects

Novelty effects occur when users react to a change simply because it's new, not because it's better.

**Detection methods:**
- **Cohort analysis:** Compare users who entered the experiment early vs. late
- **Time-series decomposition:** Plot the treatment effect over time — novelty effects decay
- **New-user vs. returning-user split:** New users have no baseline expectation

**Mitigation:**
- Run experiments for at least 2-4 weeks
- Exclude the first 3-7 days from analysis (burn-in period)
- Segment by user tenure: if the effect only exists for existing users and decays, it's novelty
- Use cookie-day analysis: track individual user behavior over time within the experiment

> 💡 **Critical Insight:** The opposite of novelty effect is **primacy effect** (change aversion) — users dislike change initially but adapt. Both decay over time. The solution is the same: run experiments long enough and examine time trends. Netflix famously found that many UI changes show strong primacy effects that fade within 2 weeks.

---

## 9. Network Effects & SUTVA Violations

**SUTVA (Stable Unit Treatment Value Assumption):** One user's treatment assignment does not affect another user's outcome. This is violated in:

- **Social networks** (Meta): Treating user A changes their posts, which affects user B's feed
- **Marketplaces** (Uber, Airbnb): Showing a driver more rides reduces availability for other drivers
- **Two-sided platforms** (DoorDash): Changing merchant ranking affects both merchants and consumers

**Solutions:**

| Approach | How It Works | Company Example |
|----------|-------------|-----------------|
| Cluster randomization | Randomize at geo/network-cluster level | Uber (city-level) |
| Ego-network randomization | Treat user + their connections as a unit | Meta (friend clusters) |
| Switchback experiments | Alternate treatment/control over time | Uber, Lyft (time-based) |
| Synthetic control | Use non-treated markets as counterfactual | DoorDash |
| Interference modeling | Model spillover explicitly | LinkedIn |

```python
from scipy import stats
import numpy as np

def cluster_randomized_test(cluster_means_control, cluster_means_treatment):
    """Effective sample size = number of CLUSTERS, not individual users."""
    t_stat, p_value = stats.ttest_ind(
        cluster_means_treatment, cluster_means_control, equal_var=False)
    return {'effect': np.mean(cluster_means_treatment) - np.mean(cluster_means_control),
            'p_value': p_value, 'n_clusters': len(cluster_means_control)}
```

> 💡 **Critical Insight:** In a Uber/Lyft interview, if asked to design an experiment for surge pricing, you MUST mention SUTVA violations. The correct answer involves switchback experiments (alternating treatment/control across time windows within a city) or geo-based randomization. User-level randomization is wrong because one user's surge price affects driver supply for all users.

---

## 10. Bayesian A/B Testing

Bayesian testing provides probability statements about which variant is better, rather than just "reject/fail to reject."

| Aspect | Frequentist | Bayesian |
|--------|------------|---------|
| Output | p-value, CI | Posterior distribution, P(B>A) |
| Interpretation | "Probability of data given H₀" | "Probability that B is better" |
| Sample size | Fixed, pre-determined | Can be flexible |
| Peeking | Inflates error rate | Built-in validity |
| Decision rule | p < α | P(B>A) > threshold, or expected loss |
| Prior knowledge | Not incorporated | Explicitly incorporated |

```python
import numpy as np

def bayesian_ab_test(successes_a, trials_a, successes_b, trials_b,
                     prior_alpha=1, prior_beta=1, n_sim=100000):
    """Bayesian A/B test using Beta-Binomial model with Monte Carlo."""
    # Posterior: Beta(prior + successes, prior + failures)
    samples_a = np.random.beta(prior_alpha + successes_a,
                               prior_beta + (trials_a - successes_a), n_sim)
    samples_b = np.random.beta(prior_alpha + successes_b,
                               prior_beta + (trials_b - successes_b), n_sim)
    
    prob_b_better = np.mean(samples_b > samples_a)
    # Expected loss: risk of choosing wrong variant
    loss_b = np.mean(np.maximum(samples_a - samples_b, 0))
    loss_a = np.mean(np.maximum(samples_b - samples_a, 0))
    
    return {'prob_b_better': prob_b_better, 'loss_choosing_b': loss_b,
            'loss_choosing_a': loss_a,
            'uplift_estimate': np.mean((samples_b - samples_a) / samples_a)}

# Example: 9.8% vs 10.4% conversion
results = bayesian_ab_test(490, 5000, 520, 5000)
print(f"P(B > A) = {results['prob_b_better']:.3f}")
```

> 💡 **Critical Insight:** Bayesian testing is preferred when: (1) you want to peek at results without inflating error, (2) you need intuitive probability statements for stakeholders, (3) you have strong priors from historical data, or (4) you want to make cost-aware decisions via expected loss. Netflix and Spotify use Bayesian frameworks extensively.

---

## 11. Sequential Testing

Sequential testing allows you to monitor results continuously and stop early (for either efficacy or futility) without inflating Type I error.

**Methods:**
- **Group Sequential Testing (GST):** Pre-planned interim analyses with adjusted boundaries (O'Brien-Fleming, Pocock)
- **Always-Valid p-values:** Confidence sequences that remain valid at any stopping time
- **mSPRT (mixture Sequential Probability Ratio Test):** Used by Optimizely, supports continuous monitoring

```python
import numpy as np
from scipy import stats

def obrien_fleming_boundary(alpha, n_looks):
    """O'Brien-Fleming: conservative early, liberal at end. Used at Google/Meta."""
    boundaries = []
    for k in range(1, n_looks + 1):
        t = k / n_looks  # information fraction
        z_boundary = stats.norm.ppf(1 - alpha/2) / np.sqrt(t)
        boundaries.append((k, t, z_boundary, 2*(1-stats.norm.cdf(z_boundary))))
    return boundaries

# 5 planned looks, alpha = 0.05
for look, frac, z_b, p_equiv in obrien_fleming_boundary(0.05, 5):
    print(f"Look {look}: info={frac:.0%}, z_boundary={z_b:.3f}, equiv_p={p_equiv:.5f}")
```

> 💡 **Critical Insight:** "Peeking" at classical fixed-horizon tests inflates your false positive rate dramatically — checking a test 5 times at p<0.05 gives you ~14% actual Type I error, not 5%. Sequential testing solves this formally. In interviews, always mention that you'd use sequential methods if there's any plan to monitor results before the pre-determined end date.

---

## 12. CUPED Variance Reduction

**CUPED (Controlled-experiment Using Pre-Experiment Data)** reduces metric variance by leveraging pre-experiment data as a covariate, typically reducing required sample size by 20-50%.

**How it works:**
- For each user, compute a "pre-experiment covariate" X (e.g., their metric value in the 2 weeks before the experiment)
- Adjust the outcome: Y_adjusted = Y - θ * X, where θ = Cov(Y, X) / Var(X)
- The adjusted metric has lower variance since we've removed predictable variation

```python
import numpy as np
from scipy import stats

def cuped_analysis(y_control, y_treatment, x_control, x_treatment):
    """CUPED: adjust outcomes using pre-experiment covariate to reduce variance."""
    y_all = np.concatenate([y_control, y_treatment])
    x_all = np.concatenate([x_control, x_treatment])
    # Optimal theta = Cov(Y,X) / Var(X)
    theta = np.cov(y_all, x_all)[0, 1] / np.var(x_all)
    # Adjust outcomes
    y_c_adj = y_control - theta * (x_control - np.mean(x_all))
    y_t_adj = y_treatment - theta * (x_treatment - np.mean(x_all))
    # Variance reduction
    var_reduction = 1 - np.var(y_c_adj) / np.var(y_control)
    _, p_cuped = stats.ttest_ind(y_t_adj, y_c_adj, equal_var=False)
    _, p_orig = stats.ttest_ind(y_treatment, y_control, equal_var=False)
    return {'variance_reduction_pct': var_reduction*100, 'p_original': p_orig,
            'p_cuped': p_cuped, 'theta': theta}

# Simulation: pre-period covariate strongly predicts post-period outcome
np.random.seed(42)
n = 10000
pre = np.random.normal(50, 20, n*2)
y_c = pre[:n] + np.random.normal(0, 15, n)
y_t = pre[n:] + np.random.normal(1.5, 15, n)  # true lift = 1.5

r = cuped_analysis(y_c, y_t, pre[:n], pre[n:])
print(f"Variance reduction: {r['variance_reduction_pct']:.1f}%")
print(f"P-value original: {r['p_original']:.4f} | CUPED: {r['p_cuped']:.4f}")
```

> 💡 **Critical Insight:** CUPED is used at virtually every major tech company (Microsoft invented it, Meta/Google/Uber all use variants). In interviews, mentioning CUPED demonstrates you understand production experimentation beyond textbook statistics. The key insight is: if pre-experiment behavior is correlated with post-experiment behavior (which it almost always is — ρ > 0.5 for most engagement metrics), you can "subtract out" that predictable variation and get a much cleaner signal.

---

## 13. Metric Selection

| Metric Type | Purpose | Examples | Properties |
|-------------|---------|----------|------------|
| **Primary (OEC)** | Single decision metric | Revenue/user, DAU, bookings | Sensitive, directional, aligned with long-term value |
| **Secondary** | Supporting evidence | Session duration, pages/visit | Explain mechanism of primary change |
| **Guardrail** | Prevent harm | Latency, crash rate, churn | One-sided test (only care about degradation) |
| **Debug/Diagnostic** | Understand segments | Metric by device, new vs. old users | Not used for ship/no-ship decision |

**Good metric properties:**
1. **Sensitive** — Moves when the product changes (not too noisy)
2. **Attributable** — Clearly connected to the treatment
3. **Timely** — Observable within experiment duration (not "5-year retention")
4. **Trustworthy** — Not gameable, accurately measured

> 💡 **Critical Insight:** The Overall Evaluation Criterion (OEC) should be a single metric that captures long-term business value in a short-term measurable way. Bing famously uses "sessions per user" as their OEC because it correlates with long-term revenue better than click-through rate. When asked "what metric would you use?", always ask yourself: "Does optimizing this metric align with long-term user and business value?"

---

## 14. Sample Ratio Mismatch (SRM)

SRM occurs when the observed split between control and treatment differs significantly from the expected split. It indicates a bug in the experiment infrastructure.

```python
from scipy import stats

def check_srm(n_control, n_treatment, expected_ratio=0.5):
    """Chi-squared SRM test. Run BEFORE analyzing any results."""
    total = n_control + n_treatment
    exp_c, exp_t = total * expected_ratio, total * (1 - expected_ratio)
    chi2 = (n_control-exp_c)**2/exp_c + (n_treatment-exp_t)**2/exp_t
    p_value = 1 - stats.chi2.cdf(chi2, df=1)
    return {'actual_ratio': n_control/total, 'p_value': p_value,
            'srm_detected': p_value < 0.001,
            'action': 'STOP - investigate bug' if p_value < 0.001 else 'OK'}

result = check_srm(502341, 497812)
print(f"SRM: p={result['p_value']:.4f} → {result['action']}")
```

**Common SRM causes:**
- Bot filtering applied differently to control/treatment
- Redirects or slow page loads causing differential attrition
- Experiment assignment happening after a filter (survivorship bias)
- Hash function collision or implementation bugs

> 💡 **Critical Insight:** SRM is the first thing you should check in ANY experiment. It's a red flag question in interviews — if you present experiment results without mentioning SRM, interviewers may ask "how do you know the randomization worked?" Always state: "Before looking at metrics, I'd verify there's no sample ratio mismatch using a chi-squared test at a strict threshold (p < 0.001)."

---

## 15. Practical Experiment Gotchas at Scale

| Gotcha | Problem | Solution |
|--------|---------|----------|
| Peeking | Inflates false positives | Sequential testing or fixed-horizon discipline |
| Day-of-week effects | Mon behavior ≠ Sun behavior | Run for full weeks |
| Carryover effects | Prior treatment persists | Sufficient washout period between experiments |
| Survivorship bias | Analyzing only users who "survived" | Intent-to-treat analysis |
| Simpson's paradox | Segment-level trends reverse at aggregate | Pre-stratification (CUPED) |
| Interaction effects | Two experiments interact | Experiment isolation or factorial design |
| Triggered analysis | Treatment only affects subset | Analyze triggered users, report on full population |
| Data delays | Metric data arrives late | Wait for data maturity before analysis |

---

## 16. Interview Talking Points & Scripts

### "Your test shows p=0.06. What do you do?"

> *"A p-value of 0.06 means the result is not statistically significant at the conventional 0.05 threshold. I would NOT call this a 'trend' or ship based on it. Here's my framework: First, I'd check if the experiment was properly powered — if we powered for 80% and the observed effect is close to our MDE, this might be a power issue. Second, I'd look at the confidence interval — if it's [−0.1%, +2.5%], there might be a real effect we're underpowered to detect. Third, I'd check secondary metrics and segments for consistency. Fourth, I'd consider extending the experiment to accumulate more data if the potential upside justifies the cost. What I would NOT do is: run the test longer until it hits p<0.05 (that's peeking), change to a one-tailed test after the fact, or cherry-pick a subgroup where it's significant."*

### "How would you design an experiment for [feature X]?"

> *"I'd follow a structured framework: (1) Define the hypothesis and primary metric — what specifically are we trying to improve and how will we measure it? (2) Choose the randomization unit — typically user-level unless there are network effects. (3) Calculate sample size based on the minimum detectable effect that would justify shipping. (4) Define guardrail metrics — what must NOT degrade? (5) Plan the experiment duration accounting for weekly cycles and novelty effects. (6) Decide on traffic allocation — start at 5%, verify no SRM or crashes, then ramp to 50/50. (7) Pre-register the analysis plan: primary metric, correction for multiple comparisons, how we handle edge cases. Then after launch: check SRM within 24 hours, monitor guardrails daily, and analyze at the pre-determined end date."*

### "How would you handle a case where your experiment shows a positive result but stakeholders don't want to ship?"

> *"I'd first verify the result is robust: check for novelty effects by looking at the time trend, verify no SRM, confirm guardrail metrics are clean, and check if the effect is consistent across key segments. If the result is solid, I'd try to understand the stakeholder concern — maybe there's a qualitative dimension the metric doesn't capture, or there are engineering costs that outweigh the measured benefit. Data informs decisions but doesn't make them alone. I'd present the evidence clearly with confidence intervals and practical significance, and propose a longer holdout if long-term effects are the concern."*

---

## 17. Common Interview Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Saying "p=0.06 shows a trend" | "Not significant at α=0.05; may extend experiment if powered correctly" |
| Using one-tailed test without justification | Default to two-tailed; one-tailed only for guardrail degradation checks |
| Forgetting to check SRM first | "Before any analysis, I'd verify randomization with a chi-squared SRM check" |
| Ignoring multiple testing | "With 5 metrics, I'd apply Holm correction to control FWER" |
| Not mentioning practical significance | "Statistical significance alone isn't enough — is the effect size meaningful?" |
| Randomizing at wrong unit | Match randomization to how treatment operates; mention SUTVA for marketplace |
| Peeking and declaring significance | Use sequential testing or commit to fixed horizon |
| Ignoring novelty/primacy effects | Run 2+ weeks; analyze time trend; segment by user tenure |
| Power analysis after the experiment | Power analysis is ALWAYS done before; post-hoc power is meaningless |
| Saying "the experiment failed" when non-significant | "We failed to detect an effect ≥ our MDE; this is informative, not a failure" |

---

## 18. Rapid-Fire Q&A

**Q1: What's the difference between statistical significance and practical significance?**  
A: Statistical significance (p<0.05) means the effect is unlikely due to chance. Practical significance means the effect size is large enough to matter for the business. You can have a statistically significant but practically meaningless 0.001% lift with a huge sample.

**Q2: When would you use a paired t-test in an A/B test?**  
A: When you have before/after measurements on the same users (within-subject design), or when using CUPED-style adjustments where you compare each user's pre vs. post behavior.

**Q3: What is the minimum experiment duration you'd recommend?**  
A: At least one full week (to capture day-of-week effects), typically 2-4 weeks. For subscription products or low-frequency behaviors, potentially longer.

**Q4: How do you handle experiments where the metric has high variance (e.g., revenue)?**  
A: (1) Use CUPED with pre-period data, (2) Winsorize outliers at the 99th percentile, (3) Use log-transformation if right-skewed, (4) Consider trimmed means, (5) Increase sample size.

**Q5: What's an A/A test and why run one?**  
A: An experiment where both groups get the same experience. Used to validate the experimentation platform: confirm no SRM, verify that 5% of A/A tests produce p<0.05 (calibration check), and detect systematic biases.

**Q6: How does Instagram/Meta handle experiments on the feed?**  
A: User-level randomization for feed algorithm changes. They account for network effects by measuring "ecosystem metrics" that capture spillover, and use ego-network clustering for social features.

**Q7: What's the difference between intent-to-treat (ITT) and per-protocol analysis?**  
A: ITT analyzes all users as randomized, regardless of whether they actually saw the treatment. Per-protocol only analyzes users who complied. ITT is the standard because it preserves randomization and avoids selection bias.

**Q8: How would you handle an experiment where only 10% of users trigger the feature?**  
A: Use triggered analysis: analyze only triggered users for effect estimation, but report significance on the full population (diluted effect). This gives you both the "did it work for those who saw it" and "what's the overall impact" answers.

**Q9: What is an interleaving experiment?**  
A: Used for ranking systems (search, recommendations). Show users a mixed list from both algorithms and measure preference. Much more sensitive than A/B tests — Netflix reports 100x more sensitivity for ranking experiments.

**Q10: When should you NOT run an A/B test?**  
A: (1) Ethical violations (withholding critical features), (2) No meaningful metric exists, (3) Already know the answer, (4) Effect is so large it's obvious, (5) Cannot randomize properly (regulatory constraints), (6) Sample too small for required MDE.

---

## 19. Summary Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════════╗
║                    A/B TESTING CHEAT SHEET                              ║
╠══════════════════════════════════════════════════════════════════════════╣
║                                                                        ║
║  BEFORE EXPERIMENT:                                                    ║
║  ┌─────────────────────────────────────────────────────────────────┐   ║
║  │ 1. Define hypothesis (H₀ vs H₁)                                │   ║
║  │ 2. Choose primary metric (OEC) + guardrails                     │   ║
║  │ 3. Calculate sample size (power analysis)                       │   ║
║  │ 4. Pick randomization unit (user > session > page)              │   ║
║  │ 5. Plan duration (≥1 week, usually 2-4 weeks)                  │   ║
║  │ 6. Pre-register analysis plan                                   │   ║
║  └─────────────────────────────────────────────────────────────────┘   ║
║                                                                        ║
║  DURING EXPERIMENT:                                                    ║
║  ┌─────────────────────────────────────────────────────────────────┐   ║
║  │ • Check SRM within 24 hours (chi-squared, p < 0.001)           │   ║
║  │ • Monitor guardrails (crash rate, latency)                      │   ║
║  │ • Do NOT peek at primary metric (or use sequential testing)     │   ║
║  │ • Watch for novelty effects (time trend analysis)               │   ║
║  └─────────────────────────────────────────────────────────────────┘   ║
║                                                                        ║
║  ANALYSIS:                                                             ║
║  ┌─────────────────────────────────────────────────────────────────┐   ║
║  │ • Proportions → Z-test | Means → Welch's t-test                │   ║
║  │ • Multiple metrics → Holm correction (FWER) or BH (FDR)        │   ║
║  │ • Variance reduction → CUPED (20-50% reduction)                │   ║
║  │ • Report: effect size + CI + practical significance             │   ║
║  │ • Segment analysis: new/old users, platforms, geos             │   ║
║  └─────────────────────────────────────────────────────────────────┘   ║
║                                                                        ║
║  DECISION FRAMEWORK:                                                   ║
║  ┌─────────────────────────────────────────────────────────────────┐   ║
║  │ p < 0.05 + meaningful effect + guardrails clean → SHIP          │   ║
║  │ p < 0.05 + tiny effect → Cost-benefit analysis                  │   ║
║  │ p > 0.05 + CI excludes MDE → No effect, MOVE ON                │   ║
║  │ p > 0.05 + CI includes MDE → UNDERPOWERED, extend or redesign  │   ║
║  │ Guardrail violation → DO NOT SHIP regardless of primary         │   ║
║  └─────────────────────────────────────────────────────────────────┘   ║
║                                                                        ║
║  KEY NUMBERS:                                                          ║
║  α = 0.05 | Power = 0.80 | SRM threshold: p < 0.001                  ║
║  CUPED: ~30-50% variance reduction | Sequential: O'Brien-Fleming      ║
║  Minimum duration: 1 full week | Typical: 2-4 weeks                   ║
║                                                                        ║
╚══════════════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 16 of 25*
