# 🎯 Topic 25: Pitfalls — Peeking, Multiple Testing, and SRM

> *"The most dangerous experiments aren't the ones that fail — they're the ones that succeed for the wrong reasons."*

---

## Table of Contents

1. [The Peeking Problem](#peeking-problem)
2. [Sequential Testing Solutions](#sequential-testing)
3. [Multiple Testing Problem](#multiple-testing)
4. [Correction Methods: Bonferroni and BH/FDR](#correction-methods)
5. [Sample Ratio Mismatch (SRM)](#sample-ratio-mismatch)
6. [Simpson's Paradox in Experiments](#simpsons-paradox)
7. [Survivorship Bias](#survivorship-bias)
8. [Interference Between Units](#interference)
9. [Interview Talking Points](#interview-talking-points)
10. [Common Mistakes](#common-mistakes)
11. [Rapid-Fire Q&A](#rapid-fire-qa)
12. [ASCII Cheat Sheet](#cheat-sheet)

---

## The Peeking Problem {#peeking-problem}

### What Is Peeking?

Peeking = checking experiment results repeatedly and stopping when you see significance.

```
Day 1: p = 0.23  → Keep running
Day 2: p = 0.08  → Keep running
Day 3: p = 0.04  → "It's significant! Ship it!" ← WRONG
Day 4: p = 0.15  → (would have reverted if you waited)
Day 5: p = 0.31  → (clearly not significant)
```

### Why Peeking Inflates Type I Error

Under continuous monitoring, the probability of EVER seeing p < 0.05 is much higher than 5%:

| Number of Checks | Actual Type I Error Rate |
|-----------------|--------------------------|
| 1 (no peeking) | 5.0% |
| 2 | 8.3% |
| 5 | 14.2% |
| 10 | 19.3% |
| 50 | 32.2% |
| 100 | 37.1% |
| Continuous | ~100% (eventually crosses) |

> **Critical Insight** 💡
> Under continuous monitoring, a random walk WILL cross any fixed threshold given enough time. This means if you peek indefinitely at a test with no real effect, you will eventually see "significance" — it's a mathematical certainty, not a rare event.

### The Random Walk Intuition

```
Test Statistic (Z-score)
    │
 2.0├─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ SIGNIFICANCE BOUNDARY ─ ─
    │           /\        /\
 1.0├─        /  \      /  \     /\
    │       /    \    /    \   /  \
 0.0├──────/──────\──/──────\─/────\──────  (true effect = 0)
    │                               \    /
-1.0├─                               \  /
    │                                 \/
-2.0├─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
    └──────────────────────────────────────────────────────→ Time
         Day 1   Day 2   Day 3   Day 4   Day 5   Day 6
```

The test statistic under H₀ follows a random walk — it will eventually hit any boundary.

### Simulation: Peeking Inflation

```python
import numpy as np
from scipy import stats

def simulate_peeking(n_sims=10000, max_looks=50, sample_per_look=100, alpha=0.05):
    """Simulate Type I error under peeking (H₀ is true)."""
    false_positives = 0
    
    for _ in range(n_sims):
        control = []
        treatment = []
        stopped_early = False
        
        for look in range(max_looks):
            # Add more data (from same distribution — no real effect)
            control.extend(np.random.normal(0, 1, sample_per_look))
            treatment.extend(np.random.normal(0, 1, sample_per_look))
            
            # Check significance
            t_stat, p_value = stats.ttest_ind(treatment, control)
            if p_value < alpha:
                false_positives += 1
                stopped_early = True
                break
    
    actual_error_rate = false_positives / n_sims
    print(f"Peeking {max_looks} times: actual Type I error = {actual_error_rate:.1%}")
    print(f"Expected without peeking: {alpha:.1%}")

simulate_peeking()
# Output: "Peeking 50 times: actual Type I error = 32.4%"
```

---

## Sequential Testing Solutions {#sequential-testing}

### Group Sequential Methods (Spending Functions)

Allocate alpha "budget" across pre-planned interim analyses.

| Method | Approach | Boundaries | When to Use |
|--------|----------|-----------|-------------|
| Pocock | Equal spending at each look | Same boundary every look | Few planned looks |
| O'Brien-Fleming | Conservative early, liberal late | Very strict early, ≈0.05 final | Want to preserve final analysis |
| Alpha spending (Lan-DeMets) | Flexible schedule | Computed from spending function | Unknown look schedule |

**O'Brien-Fleming boundaries example (5 looks, α=0.05):**

| Look | Cumulative Info | Boundary (Z) | Boundary (p) |
|------|----------------|---------------|--------------|
| 1 (20%) | 0.20 | 4.56 | 0.000005 |
| 2 (40%) | 0.40 | 3.23 | 0.001 |
| 3 (60%) | 0.60 | 2.63 | 0.008 |
| 4 (80%) | 0.80 | 2.28 | 0.023 |
| 5 (100%) | 1.00 | 2.04 | 0.041 |

> **Critical Insight** 💡
> O'Brien-Fleming boundaries are extremely conservative early on (virtually impossible to stop early), which means the final analysis boundary is close to the fixed-sample boundary (0.041 vs. 0.05). This is why many teams prefer it — you pay almost no penalty at the final analysis.

### Always-Valid Confidence Sequences

Modern approach: construct confidence intervals that are valid at ANY stopping time.

```python
# Confidence sequence using mixture method
# At any time t, the confidence interval contains the true effect
# with probability >= 1 - alpha, regardless of when you stop

def always_valid_ci(data_control, data_treatment, alpha=0.05, rho=1.0):
    """
    Simplified always-valid confidence interval.
    Based on mixture sequential probability ratio test.
    """
    n = len(data_control)
    mean_diff = np.mean(data_treatment) - np.mean(data_control)
    pooled_var = (np.var(data_control) + np.var(data_treatment)) / 2
    se = np.sqrt(2 * pooled_var / n)
    
    # Width grows as sqrt(log(log(n))) instead of fixed
    width = se * np.sqrt(2 * (np.log(np.log(max(n, 2))) + 0.72 - np.log(alpha)))
    
    return mean_diff - width, mean_diff + width
```

### Bayesian Stopping Rules

```
Instead of fixed alpha boundary:
  → Define a decision threshold on P(treatment > control | data)
  → "Stop when P(treatment better) > 0.95 OR P(treatment better) < 0.05"
  → OR "Stop when expected loss of shipping < threshold"

Advantages:
  ✓ Natural stopping rules
  ✓ No alpha inflation (different framework)
  ✓ Can incorporate prior knowledge

Disadvantages:
  ✗ Results depend on prior choice
  ✗ Not universally accepted
  ✗ Harder to calibrate for frequentist guarantees
```

---

## Multiple Testing Problem {#multiple-testing}

### The Problem

Testing multiple hypotheses simultaneously inflates the family-wise error rate (FWER):

```
P(at least one false positive) = 1 - (1 - α)^k

k=1:   P(FP) = 5.0%
k=5:   P(FP) = 22.6%
k=10:  P(FP) = 40.1%
k=20:  P(FP) = 64.2%
k=50:  P(FP) = 92.3%
k=100: P(FP) = 99.4%
```

### When Does Multiple Testing Apply?

| Scenario | Multiple Testing? | Why |
|----------|------------------|-----|
| One primary metric, multiple secondary | Pre-register primary; adjust secondaries | Primary is protected; secondaries are exploratory |
| Multiple treatment variants (A/B/C/D) | YES | Each comparison is a separate test |
| Multiple segments analyzed | YES if pre-specified; NO if post-hoc exploratory | Post-hoc is hypothesis generating, not confirming |
| Multiple metrics, all primary | YES | Must control FWER or FDR |
| Same metric at different time points | YES (this IS peeking!) | Each look is a test |

---

## Correction Methods: Bonferroni and BH/FDR {#correction-methods}

### Bonferroni Correction (Controls FWER)

```
Adjusted α = α / k

Where k = number of tests

Example: 10 metrics, α = 0.05
  → Test each at α = 0.05 / 10 = 0.005
  → FWER ≤ 0.05 guaranteed
```

| Pros | Cons |
|------|------|
| Simple to implement | Very conservative |
| Controls FWER strictly | Power drops rapidly with k |
| No assumptions about test dependence | Impractical for k > 20 |
| Easy to explain | Many real effects will be missed |

### Holm-Bonferroni (Step-Down, Less Conservative)

```
1. Order p-values: p_(1) ≤ p_(2) ≤ ... ≤ p_(k)
2. For i-th smallest p-value, compare to α / (k - i + 1)
3. Reject until first non-rejection, then stop

Example: k=4, α=0.05, p-values = [0.001, 0.015, 0.030, 0.045]
  p_(1) = 0.001 < 0.05/4 = 0.0125 → Reject ✓
  p_(2) = 0.015 < 0.05/3 = 0.0167 → Reject ✓
  p_(3) = 0.030 < 0.05/2 = 0.025? → 0.030 > 0.025 → STOP
  p_(4) = 0.045 → Not rejected (even though < 0.05)
```

### Benjamini-Hochberg (Controls FDR)

```
FDR = E[False Positives / Total Positives]

Procedure:
1. Order p-values: p_(1) ≤ p_(2) ≤ ... ≤ p_(k)
2. Find largest i where p_(i) ≤ (i/k) × α
3. Reject all hypotheses 1, ..., i

Example: k=5, α=0.05, p-values = [0.005, 0.011, 0.023, 0.040, 0.120]
  i=1: 0.005 ≤ (1/5)×0.05 = 0.01 → ✓
  i=2: 0.011 ≤ (2/5)×0.05 = 0.02 → ✓
  i=3: 0.023 ≤ (3/5)×0.05 = 0.03 → ✓
  i=4: 0.040 ≤ (4/5)×0.05 = 0.04 → ✓ (exactly!)
  i=5: 0.120 ≤ (5/5)×0.05 = 0.05 → ✗
  
  Largest valid i = 4 → Reject first 4 hypotheses
```

### Comparison Table

| Method | Controls | Conservatism | Best For |
|--------|----------|-------------|----------|
| Bonferroni | FWER | Most conservative | Few critical tests |
| Holm-Bonferroni | FWER | Slightly less | Few tests, want more power |
| Benjamini-Hochberg | FDR | Moderate | Many metrics, exploratory |
| No correction | Nothing | Liberal | Pre-specified single primary |

> **Critical Insight** 💡
> In industry A/B testing, the practical approach is often: (1) pre-register ONE primary metric with no correction needed, (2) treat secondary metrics as directional evidence without formal correction, (3) apply BH-FDR if you genuinely have multiple primary metrics. Bonferroni is mainly used in multi-arm experiments.

```python
from statsmodels.stats.multitest import multipletests

p_values = [0.005, 0.011, 0.023, 0.040, 0.120]

# Bonferroni
reject_bonf, pvals_bonf, _, _ = multipletests(p_values, method='bonferroni')
print(f"Bonferroni rejections: {reject_bonf}")

# Holm
reject_holm, pvals_holm, _, _ = multipletests(p_values, method='holm')
print(f"Holm rejections: {reject_holm}")

# Benjamini-Hochberg
reject_bh, pvals_bh, _, _ = multipletests(p_values, method='fdr_bh')
print(f"BH-FDR rejections: {reject_bh}")
```

---

## Sample Ratio Mismatch (SRM) {#sample-ratio-mismatch}

### What Is SRM?

SRM occurs when the observed ratio of users in treatment vs. control deviates significantly from the intended ratio.

```
Intended: 50% / 50% split (1:1 ratio)
Observed: 51.3% / 48.7%

Chi-squared test: χ² = (observed - expected)² / expected
                     = very significant if N is large

Even a 0.5% deviation with N=1,000,000 is highly significant!
```

### Why SRM Is Critical

> **Critical Insight** 💡
> SRM is the #1 data quality check for any experiment. If your sample ratio doesn't match the intended allocation, something is SYSTEMATICALLY different about who ends up in treatment vs. control. Any treatment effect estimate is likely BIASED and untrustworthy.

### Common Causes of SRM

| Cause | Mechanism | How to Detect |
|-------|-----------|---------------|
| Bot filtering | Bots trigger treatment differently | Check bot rates by group |
| Redirect bugs | Treatment redirect fails, users fall back to control | Check redirect error logs |
| Browser caching | Cached pages serve old variant | Check by browser/device |
| Trigger timing | Assignment before/after user action | Check assignment timestamps |
| Variant-specific crashes | Treatment causes app crash | Check crash rates by group |
| Cookie expiration | Treatment cookies expire differently | Check assignment persistence |
| Delayed logging | Treatment log events take longer | Check logging latency |

### SRM Detection

```python
from scipy import stats

def check_srm(n_control, n_treatment, expected_ratio=0.5, alpha=0.001):
    """
    Check for Sample Ratio Mismatch using chi-squared test.
    Uses very strict alpha (0.001) because SRM is serious.
    """
    total = n_control + n_treatment
    expected_control = total * (1 - expected_ratio)
    expected_treatment = total * expected_ratio
    
    chi2 = ((n_control - expected_control)**2 / expected_control +
            (n_treatment - expected_treatment)**2 / expected_treatment)
    p_value = 1 - stats.chi2.cdf(chi2, df=1)
    
    print(f"Control: {n_control:,} | Treatment: {n_treatment:,}")
    print(f"Observed ratio: {n_treatment/total:.4f} (expected {expected_ratio})")
    print(f"Chi-squared: {chi2:.2f}, p-value: {p_value:.2e}")
    
    if p_value < alpha:
        print("⚠️  SRM DETECTED — experiment results may be invalid!")
        return True
    else:
        print("✓ No SRM detected")
        return False

# Example: slight imbalance with large N
check_srm(n_control=502_100, n_treatment=497_900)
```

### What to Do When SRM Is Detected

```
1. DO NOT trust the experiment results
2. Investigate root cause:
   a. Check by platform (iOS/Android/Web)
   b. Check by day (did ratio change over time?)
   c. Check assignment vs. trigger events
   d. Look at pre-treatment covariates (age, country)
3. If root cause found and fixable:
   → Fix and re-run experiment
4. If root cause found and affects one segment:
   → Analyze unaffected segment separately (with caveats)
5. If root cause NOT found:
   → Cannot trust results — escalate to platform team
```

---

## Simpson's Paradox in Experiments {#simpsons-paradox}

### The Paradox

A trend that appears in aggregate data **reverses** when data is split into subgroups.

```
Overall:  Treatment conversion = 8.2% > Control conversion = 7.9%
          → "Treatment wins!"

But segmented:
  Mobile:  Treatment = 6.0% < Control = 6.5%  → Control wins
  Desktop: Treatment = 12.0% < Control = 12.3% → Control wins

How? Treatment had more desktop users (higher baseline conversion).
The "effect" was just differential segment composition.
```

### When Simpson's Paradox Appears in A/B Tests

| Scenario | Root Cause | Prevention |
|----------|-----------|------------|
| Non-uniform traffic over time | Treatment exposed more on weekdays (higher baseline) | Run full weeks, check day balance |
| Platform imbalance | Treatment assigned more mobile (or desktop) users | Check SRM by platform |
| New vs. returning users | Treatment got more returning users | Stratify randomization |
| Geographic mix | Treatment over-represented high-value regions | Check geo balance |

> **Critical Insight** 💡
> Simpson's paradox in a PROPERLY randomized experiment indicates an SRM or randomization bug. In a correctly randomized test, subgroup proportions should be balanced. If you see Simpson's paradox, investigate data quality before trusting any results.

---

## Survivorship Bias {#survivorship-bias}

### In Experiments

```
Scenario: Testing a new onboarding flow

Day 1: 10,000 users enter experiment
Day 7: Only 3,000 users remain active

If you analyze only "users who were active on Day 7":
  → You're conditioning on a POST-TREATMENT variable
  → Users who churned (maybe because of bad treatment) are excluded
  → Estimate is BIASED toward treatment looking good
```

### Common Forms in A/B Testing

| Form | Example | Fix |
|------|---------|-----|
| Analyzing only converters | "Among purchasers, AOV is higher" | Include all assigned users |
| Analyzing only retained users | "Active users engage more" | Intent-to-treat analysis |
| Excluding early churners | "After removing bounces..." | Don't exclude post-treatment |
| Time-to-event without censoring | "Users who completed..." | Use survival analysis |

### Intent-to-Treat (ITT) vs. Per-Protocol

```
ITT (Correct for A/B tests):
  Analyze ALL users as randomized, regardless of actual exposure
  → Unbiased estimate of ASSIGNMENT effect
  → May dilute effect if many don't see treatment

Per-Protocol (Biased but informative):
  Analyze only users who actually experienced the treatment
  → Biased (non-random subset)
  → Use CACE/LATE for unbiased version (instrumental variables)
```

---

## Interference Between Units {#interference}

### SUTVA Violation

SUTVA (Stable Unit Treatment Value Assumption): One user's treatment doesn't affect another user's outcome.

**When SUTVA is violated:**
- Social networks (friends influence each other)
- Marketplaces (buyers and sellers interact)
- Shared resources (if treatment uses more server capacity)
- Geographic spillover (treatment in one region affects adjacent regions)

### Examples of Interference

```
1. SOCIAL NETWORK: User A (treatment) shares content with User B (control)
   → B's engagement changes because of A's treatment
   → Control group is "contaminated"

2. MARKETPLACE: Seller pricing experiment
   → Treatment sellers lower prices
   → Control sellers lose sales (displacement)
   → Treatment effect is overestimated

3. RIDE-SHARING: New matching algorithm for treatment riders
   → Treatment riders get faster matches
   → Control riders get SLOWER matches (same driver pool)
   → Effect is 2x the true improvement
```

### Solutions for Interference

| Solution | Mechanism | Trade-off |
|----------|-----------|-----------|
| Cluster randomization | Randomize groups (cities, friend clusters) | Fewer independent units → less power |
| Switchback experiments | Alternate treatment/control over time | Time effects confound |
| Geo experiments | Randomize by geography | Few regions → very low power |
| Ego-network randomization | Randomize ego + alters together | Complex, partial solution |
| Ghost ads / intent-to-treat | Measure counterfactual without delivery | Only for ads/content |

> **Critical Insight** 💡
> Interference is most dangerous when you DON'T know it exists. The standard A/B test assumes SUTVA. If your feature involves social sharing, marketplace dynamics, or shared infrastructure, you MUST design for interference — otherwise your effect estimates could be 2-3x overstated.

---

## Interview Talking Points {#interview-talking-points}

> **"You notice your test hits significance on Day 3 of a planned 14-day test. What do you do?"**
>
> "I do NOT stop the test. Early significance is expected under continuous monitoring — even when there's no real effect, a random walk will cross the significance boundary about 30% of the time if you check daily. I would either: (a) use a sequential testing framework with spending functions that allocates alpha across looks, or (b) commit to the full 14-day duration and only look at the final result. If I absolutely need to peek, I'd use always-valid confidence sequences that maintain coverage at any stopping time."

> **"How do you handle testing 20 metrics in one experiment?"**
>
> "I'd structure metrics into tiers. One pre-registered primary OEC — no correction needed for that. Then I'd categorize the other 19 as secondary (directional evidence, no formal correction) or as additional primaries requiring correction. If multiple metrics are truly co-primary, I'd use Benjamini-Hochberg FDR control rather than Bonferroni, since FDR is more appropriate when I expect some metrics to truly move and I want to control the fraction of false discoveries rather than any single false positive."

> **"Your experiment shows a positive result but fails the SRM check. What's your conclusion?"**
>
> "I cannot trust the result. SRM means the randomization or measurement is compromised — some systematic factor is causing differential attrition or assignment between groups. Even if the treatment effect looks positive, it could be entirely explained by the compositional imbalance. I'd investigate: check by platform, check over time, look for redirect failures or bot filtering asymmetries. Only after identifying and resolving the root cause would I consider re-running the experiment."

---

## Common Mistakes {#common-mistakes}

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Stopping when first significant | Use sequential testing or commit to fixed duration |
| Applying Bonferroni to 50 metrics | Use BH-FDR, or tier metrics (one primary, rest secondary) |
| Ignoring SRM because "it's close to 50/50" | Always run chi-squared test; even 0.3% imbalance at scale is concerning |
| "We'll just adjust for the imbalance" | SRM signals UNMEASURED bias; adjustment won't fix unknown confounds |
| Post-hoc subgroup analysis without correction | Label as exploratory; don't claim significance |
| Analyzing only users who "completed" treatment | Intent-to-treat: include all randomized users |
| Assuming SUTVA in marketplace experiments | Design for interference: cluster randomization or switchback |
| Restarting failed experiments without fixing root cause | Identify WHY SRM/bugs occurred before re-running |
| Multiple comparisons across experiments | Each experiment has its own alpha; cross-experiment is different |
| Choosing one-sided test after seeing direction | Must pre-specify; switching is p-hacking |

---

## Rapid-Fire Q&A {#rapid-fire-qa}

**Q1: What is the peeking problem?**
> Repeatedly checking experiment results and stopping when significant inflates Type I error far beyond the nominal alpha. Under continuous monitoring with no real effect, you'll eventually see significance by random fluctuation.

**Q2: How do O'Brien-Fleming boundaries differ from Pocock?**
> O'Brien-Fleming boundaries are very strict early (hard to stop early) but almost unchanged at the final analysis. Pocock uses equal boundaries throughout, making early stopping easier but the final analysis more conservative.

**Q3: What is FDR and when do you prefer it over FWER?**
> FDR (False Discovery Rate) = expected proportion of rejected hypotheses that are false positives. Prefer it when testing many hypotheses and some are expected to be truly positive (exploratory analysis with many metrics).

**Q4: What causes Sample Ratio Mismatch?**
> Bot filtering differences, redirect failures, browser caching, variant-specific crashes, trigger timing bugs, cookie expiration issues, or delayed logging.

**Q5: Can you trust experiment results if SRM is detected?**
> No. SRM indicates systematic bias in group composition. Even if you can explain the mechanism, the bias likely extends to unmeasured variables.

**Q6: What is Simpson's paradox and when does it occur in A/B tests?**
> A trend reverses when data is disaggregated. In A/B tests, it signals differential segment composition between groups — usually a randomization or measurement problem.

**Q7: What is SUTVA and why does it matter?**
> Stable Unit Treatment Value Assumption: one unit's treatment assignment doesn't affect another unit's outcome. Violated in networks, marketplaces, and shared-resource settings, causing biased effect estimates.

**Q8: When should you use cluster randomization?**
> When interference between units is expected — social features, marketplace dynamics, or geographic spillovers. Randomize clusters (cities, friend groups) instead of individuals.

**Q9: What is intent-to-treat analysis?**
> Analyzing all users as randomized, regardless of whether they actually saw or engaged with the treatment. Prevents survivorship bias and maintains the validity of randomization.

**Q10: How many times can you peek at an experiment without inflating error?**
> With no correction: zero additional times beyond the pre-planned analysis. With sequential testing: as many as designed for, using spending functions that allocate alpha across looks. With always-valid methods: unlimited peeks with valid inference.

---

## ASCII Cheat Sheet {#cheat-sheet}

```
╔══════════════════════════════════════════════════════════════════════╗
║         EXPERIMENT PITFALLS CHEAT SHEET                             ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  PEEKING:                                                            ║
║  • Checking daily with α=0.05 → actual error ≈ 30%+                ║
║  • Fix: Sequential testing (O'Brien-Fleming, alpha spending)        ║
║  • Fix: Always-valid confidence sequences                           ║
║  • Fix: Pre-commit to duration, analyze only at end                 ║
║                                                                      ║
║  MULTIPLE TESTING:                                                   ║
║  ┌──────────────────┬───────────────┬─────────────────┐            ║
║  │ Method           │ Controls      │ Conservatism    │            ║
║  ├──────────────────┼───────────────┼─────────────────┤            ║
║  │ Bonferroni       │ FWER          │ Very strict     │            ║
║  │ Holm             │ FWER          │ Less strict     │            ║
║  │ Benjamini-Hochb. │ FDR           │ Moderate        │            ║
║  │ No correction    │ Nothing       │ Liberal         │            ║
║  └──────────────────┴───────────────┴─────────────────┘            ║
║                                                                      ║
║  FWER: P(any false positive) ≤ α                                    ║
║  FDR:  E[false positives / total positives] ≤ α                     ║
║                                                                      ║
║  SRM (Sample Ratio Mismatch):                                        ║
║  • Test: χ² = Σ (observed - expected)² / expected                   ║
║  • Use strict α = 0.001 (SRM is serious)                            ║
║  • Common causes: bots, redirects, caching, crashes                 ║
║  • SRM detected → DO NOT TRUST RESULTS                              ║
║                                                                      ║
║  SIMPSON'S PARADOX:                                                  ║
║  • Aggregate trend reverses in subgroups                            ║
║  • In A/B tests: signals randomization failure                      ║
║  • Check segment balance before trusting results                    ║
║                                                                      ║
║  SURVIVORSHIP BIAS:                                                  ║
║  • Never condition on post-treatment variables                      ║
║  • Use intent-to-treat: analyze ALL randomized users                ║
║  • Include churners, non-converters, all assigned users             ║
║                                                                      ║
║  INTERFERENCE (SUTVA VIOLATION):                                     ║
║  • Networks, marketplaces, shared resources                         ║
║  • Solutions: cluster randomization, switchback, geo experiments    ║
║  • Standard A/B test OVERESTIMATES effect under interference        ║
║                                                                      ║
║  DATA QUALITY CHECKLIST (before trusting results):                   ║
║  □ SRM check passes                                                 ║
║  □ No Simpson's paradox by platform/day                             ║
║  □ Pre-treatment covariates balanced                                ║
║  □ No differential attrition                                        ║
║  □ Logging complete in both groups                                  ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 25 of 45*
