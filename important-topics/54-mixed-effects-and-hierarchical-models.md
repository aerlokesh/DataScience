# 🎯 Topic 54: Mixed Effects & Hierarchical Models

> *"When your data has structure — users within regions, students within schools, repeated measures — ignoring that structure doesn't make it go away. It makes your estimates wrong."*

---

## 📚 Table of Contents

1. [Why Hierarchical Structure Matters](#why-hierarchical-structure-matters)
2. [Fixed Effects vs Random Effects](#fixed-effects-vs-random-effects)
3. [Random Intercepts vs Random Slopes](#random-intercepts-vs-random-slopes)
4. [Partial Pooling and Shrinkage](#partial-pooling-and-shrinkage)
5. [Connection to A/B Testing](#connection-to-ab-testing)
6. [Comparison with OLS + Cluster-Robust SE](#comparison-with-ols--cluster-robust-se)
7. [Implementation (Python & R)](#implementation-python--r)
8. [Model Selection](#model-selection)
9. [Real-World Examples](#real-world-examples)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Why Hierarchical Structure Matters

Most real-world data is **nested**. Ignoring this structure leads to:

- **Underestimated standard errors** (you think you have more independent info than you do)
- **Biased coefficients** (Simpson's Paradox-type reversals)
- **Incorrect p-values** (false positives inflate dramatically)

### Examples of Nested Data

| Level 1 (Observation) | Level 2 (Group) | Level 3 (Higher Group) |
|------------------------|-----------------|------------------------|
| Student test scores | Classrooms | Schools |
| User clicks | Users | Countries |
| Daily ad revenue | Ad campaigns | Advertisers |
| Patient outcomes | Hospitals | Hospital networks |
| Repeated measures | Individuals | Treatment groups |

> **Critical Insight** 💡
> If you run a simple regression on 10,000 observations from 50 users (200 obs each), your effective sample size is closer to 50, not 10,000. Ignoring this means your confidence intervals are far too narrow — you'll declare effects "significant" when they're noise.

---

## Fixed Effects vs Random Effects

This is the **#1 interview question** on this topic. The terminology is confusing because it means different things in different fields.

### Comparison Table: Fixed vs Random Effects

| Dimension | Fixed Effects | Random Effects |
|-----------|--------------|----------------|
| **What it represents** | Specific levels of interest | Levels drawn from a population |
| **Goal** | Estimate effect for *these* specific groups | Estimate population-level variance |
| **Number of groups** | Small, exhaustive (e.g., 4 seasons) | Large, sample from population (e.g., 500 users) |
| **Inference extends to** | Only these groups | New unseen groups from same population |
| **Parameters estimated** | One coefficient per group | Variance component (1-2 params) |
| **Degrees of freedom cost** | High (N_groups - 1) | Low (1-2 parameters) |
| **Example** | Treatment vs Control | User-specific baselines |

### The Econometrics vs Biostatistics Confusion

| Field | "Fixed Effect" means... | "Random Effect" means... |
|-------|------------------------|-------------------------|
| **Econometrics** | Group-specific intercept (eliminates between-group variation) | Group intercepts correlated with covariates? Then FE. Uncorrelated? Then RE. |
| **Biostatistics/ML** | Coefficients that are the same for all groups | Coefficients that vary by group (drawn from distribution) |

> **Critical Insight** 💡
> **Google's favorite framing**: "You have user engagement data across 30 countries. Should country be a fixed or random effect?" 
> - **Fixed**: If you only care about these 30 countries specifically and won't generalize
> - **Random**: If you view these 30 as a *sample* of countries and want to generalize to new markets
> - **Practical rule**: >5-6 groups where you want to generalize → random effect

---

## Random Intercepts vs Random Slopes

### Random Intercepts Only

Each group gets its own baseline, but the effect of predictors is the same across groups.

```
y_ij = (β₀ + u₀ⱼ) + β₁·x_ij + ε_ij

where u₀ⱼ ~ N(0, σ²_u)  ← group-specific deviation from grand mean
      ε_ij ~ N(0, σ²_e)  ← observation-level noise
```

**Interpretation**: Users start at different baselines but respond identically to the treatment.

### Random Intercepts + Random Slopes

Each group gets its own baseline AND its own sensitivity to the predictor.

```
y_ij = (β₀ + u₀ⱼ) + (β₁ + u₁ⱼ)·x_ij + ε_ij

where [u₀ⱼ]   ~  N([0], [σ²_u0    σ_u01 ])
      [u₁ⱼ]      ([0], [σ_u01    σ²_u1 ])
```

**Interpretation**: Users start at different baselines AND have different treatment effect sizes.

### When to Use Which

| Scenario | Model Choice |
|----------|-------------|
| Users have different base engagement but same response to feature | Random intercepts |
| Users differ in base engagement AND treatment sensitivity | Random intercepts + slopes |
| Only interested in average effect, want correct SE | Random intercepts (minimum) |
| Few observations per group (<5) | Random intercepts only (slopes won't converge) |

> **Critical Insight** 💡
> **Barr et al. (2013)** recommend: "Keep it maximal." Include random slopes for all within-group predictors unless convergence fails. Under-specifying random effects inflates Type I error — you're essentially ignoring that treatment effects vary across groups.

---

## Partial Pooling and Shrinkage

The magic of mixed models lies in **partial pooling** — a principled compromise between:

1. **Complete pooling** (ignore groups, one regression) — biased if groups differ
2. **No pooling** (separate regression per group) — high variance, especially for small groups
3. **Partial pooling** (mixed model) — borrows strength across groups

### How Shrinkage Works

```
Estimated group effect = (weight) × group-specific estimate + (1 - weight) × grand mean

weight = n_j · σ²_e / (n_j · σ²_e + σ²_u)
```

- **Large groups** (high n_j): weight → 1, trust the group's own data
- **Small groups** (low n_j): weight → 0, shrink toward the grand mean
- **High between-group variance** (σ²_u): less shrinkage (groups really are different)

### Visual Intuition

```
No Pooling          Partial Pooling       Complete Pooling
(group-specific)    (mixed model)         (ignore groups)

  *                    *                      ___________
   *                    *                    |           |
    *                    *                   |   MEAN    |
     *                    *                  |___________|
      *                    *
-------*--------    --------*--------    
        *                    *
         *                    *
          *                   *          ← small groups shrink MORE
```

> **Critical Insight** 💡
> Shrinkage is NOT a bug — it's **regularization**. It reduces mean squared error (MSE) even though individual estimates become biased. This is the bias-variance tradeoff in action. James-Stein estimator proved that shrinkage dominates no-pooling for 3+ groups.

---

## Connection to A/B Testing

### Why Mixed Models Matter for Experiments

**Problem**: In A/B tests, users are the unit of randomization, but you observe multiple events per user. User-level random effects capture this.

```python
# Model: metric_ij = β₀ + β₁·treatment_i + u_i + ε_ij
# u_i = user-level random effect (captures persistent user differences)
# β₁ = the treatment effect (what we care about)
```

### Benefits for A/B Testing

| Benefit | Explanation |
|---------|-------------|
| Correct standard errors | Accounts for within-user correlation |
| Variance reduction | User random effects explain baseline variation |
| Handles unequal exposure | Users with more observations don't dominate |
| Heterogeneous effects | Random slopes reveal who benefits most |

### Practical A/B Testing Setup

```python
# Instead of:
# t-test on user-level averages (loses info) or
# t-test on observation-level (inflated N, wrong SE)

# Use:
import statsmodels.formula.api as smf

model = smf.mixedlm(
    "clicks ~ treatment",         # fixed: treatment effect
    data=df,
    groups=df["user_id"],         # random: user intercepts
)
result = model.fit()
# β₁ gives treatment effect with correct SE
```

> **Critical Insight** 💡
> At Google, mixed models are used for metrics with **high within-user variance** (e.g., session duration, revenue per visit). The user random effect absorbs individual baseline variation, giving tighter confidence intervals than a simple two-sample t-test on user averages — sometimes 20-40% power improvement.

---

## Comparison with OLS + Cluster-Robust SE

| Feature | Mixed Model | OLS + Cluster-Robust SE |
|---------|-------------|------------------------|
| **SE estimation** | Model-based (assumes normality of random effects) | Design-based (no distributional assumptions) |
| **Efficiency** | More efficient if model is correct | Less efficient but robust to misspecification |
| **Group-level predictions** | Yes (BLUPs) | No |
| **Handles few clusters** | Better (still valid with 5-10 groups) | Needs 30-50+ clusters for reliability |
| **Unbalanced data** | Handles naturally via shrinkage | Handles but doesn't exploit structure |
| **When to prefer** | Want predictions, few clusters, model assumptions plausible | Many clusters, model assumptions questionable, want robustness |
| **Complexity** | Higher (convergence issues, specification choices) | Simpler (just add one option to OLS) |

### When to Use Each

- **Mixed model**: You have 10-30 clusters, want group-level estimates, or have very unbalanced data
- **Cluster-robust SE**: You have 50+ clusters, only care about population-average effect, or distrust distributional assumptions

---

## Implementation (Python & R)

### Python (statsmodels)

```python
import statsmodels.formula.api as smf
import pandas as pd

# Random intercept model
model_ri = smf.mixedlm(
    "score ~ study_hours + ses",   # fixed effects formula
    data=df,
    groups=df["school_id"]         # grouping variable
)
result_ri = model_ri.fit()
print(result_ri.summary())

# Random intercept + random slope
model_rs = smf.mixedlm(
    "score ~ study_hours + ses",
    data=df,
    groups=df["school_id"],
    re_formula="~study_hours"      # study_hours effect varies by school
)
result_rs = model_rs.fit()

# Extract random effects (BLUPs)
random_effects = result_rs.random_effects  # dict: group -> [intercept, slope]
```

### R (lme4) — You'll See This in Interviews

```r
library(lme4)

# Random intercept
model_ri <- lmer(score ~ study_hours + ses + (1 | school_id), data = df)

# Random intercept + slope
model_rs <- lmer(score ~ study_hours + ses + (1 + study_hours | school_id), data = df)

# Random intercept + slope (uncorrelated)
model_uncorr <- lmer(score ~ study_hours + ses + (1 | school_id) + 
                     (0 + study_hours | school_id), data = df)

summary(model_rs)
ranef(model_rs)    # BLUPs
fixef(model_rs)    # Fixed effect estimates
```

### lme4 Formula Syntax Cheat Sheet

| Formula | Meaning |
|---------|---------|
| `(1 | group)` | Random intercept for group |
| `(1 + x | group)` | Random intercept + slope for x, correlated |
| `(0 + x | group)` | Random slope only (no random intercept) |
| `(1 | group1/group2)` | Nested: group2 within group1 |
| `(1 | group1) + (1 | group2)` | Crossed random effects |

---

## Model Selection

### Likelihood Ratio Test (LRT)

Compare nested models — does adding random slopes improve fit enough to justify complexity?

```python
from scipy.stats import chi2

# Compare: model with random slopes vs model with only random intercepts
LR_stat = 2 * (loglik_complex - loglik_simple)
df_diff = n_params_complex - n_params_simple
p_value = chi2.sf(LR_stat, df_diff)

# CAVEAT: For testing variance = 0, the null is on the boundary
# Use 50:50 mixture of chi²(0) and chi²(1) → halve the p-value
p_value_corrected = 0.5 * chi2.sf(LR_stat, 1)
```

### AIC/BIC for Model Comparison

| Criterion | Formula | Prefers |
|-----------|---------|---------|
| AIC | -2·loglik + 2k | Better prediction (less penalty) |
| BIC | -2·loglik + k·ln(n) | Simpler models (more penalty) |

```python
# In statsmodels
print(f"AIC: {result.aic}, BIC: {result.bic}")

# Lower is better — compare across models
```

### ICC (Intraclass Correlation Coefficient)

Before fitting a mixed model, check if grouping matters:

```
ICC = σ²_between / (σ²_between + σ²_within)
```

| ICC Value | Interpretation | Action |
|-----------|---------------|--------|
| < 0.05 | Negligible group effect | OLS might be fine |
| 0.05 - 0.15 | Moderate clustering | Mixed model recommended |
| > 0.15 | Strong clustering | Mixed model essential |

> **Critical Insight** 💡
> Even ICC = 0.01 can lead to massively inflated Type I errors if cluster sizes are large. With 100 observations per cluster, ICC = 0.01 means the effective sample size is only ~50% of the nominal sample size. **Always use a mixed model when data is clustered**, regardless of ICC.

---

## Real-World Examples

### Example 1: YouTube Ad Quality Scores (Google)

**Problem**: Predict ad quality score. Ads are nested within advertisers, and advertisers have consistent quality patterns.

```python
# Model: quality_ij = β₀ + β₁·relevance + β₂·landing_page_score + u_advertiser + ε_ij
model = smf.mixedlm(
    "quality_score ~ relevance + landing_page_score + ctr_expected",
    data=ads_df,
    groups=ads_df["advertiser_id"]
)
```

**Why mixed model?** New ads from a known advertiser get "shrunk" toward that advertiser's average — better predictions than treating each ad independently.

### Example 2: User Engagement Nested in Countries

**Problem**: Measure feature impact on engagement. Users in same country share cultural/infrastructure factors.

```python
# Crossed random effects: user AND country
# (In practice, users are nested within countries)
model = smf.mixedlm(
    "daily_sessions ~ new_feature + app_version",
    data=engagement_df,
    groups=engagement_df["country"],
    re_formula="~new_feature"  # treatment effect varies by country
)
```

**Insight**: The random slope on `new_feature` reveals whether the feature works differently across markets — crucial for launch decisions.

### Example 3: Repeated Measures in Clinical Trials

**Problem**: Patients measured at multiple time points. Want to estimate treatment trajectory.

```r
model <- lmer(
  blood_pressure ~ time * treatment + age + (1 + time | patient_id),
  data = clinical_df
)
```

**Why?** Each patient has their own trajectory (random slope on time) and baseline (random intercept). The treatment × time interaction gives the average treatment effect on the slope.

---

## Interview Talking Points

> **"Explain mixed effects models to me."**
>
> "Mixed effects models handle data with natural grouping — like users within countries, or students within schools. The 'mixed' means we have both fixed effects, which are population-level parameters we want to estimate, and random effects, which capture group-specific deviations. The key insight is partial pooling: small groups borrow strength from the overall population rather than relying solely on their sparse data. This gives us better predictions for small groups and correct standard errors that account for the non-independence within groups."

> **"When would you use a mixed model over a simple regression with dummy variables?"**
>
> "Three situations: First, when you have many groups — 500 users means 499 dummy variables, which wastes degrees of freedom and can't generalize to new users. A random effect captures this with just one variance parameter. Second, when groups have very unequal sample sizes — mixed models naturally handle this through shrinkage, pulling small-sample estimates toward the mean. Third, when you want to generalize to new groups you haven't seen — random effects let you predict for new users based on the population distribution."

> **"How does this connect to A/B testing at Google?"**
>
> "In A/B testing, users generate multiple observations but were randomized at the user level. A mixed model with user random intercepts correctly captures this within-user correlation. This gives two benefits: First, correct standard errors — without accounting for clustering, you'll overstate significance. Second, variance reduction — the random intercept absorbs baseline user differences, giving you a tighter estimate of the treatment effect. It's similar in spirit to CUPED but naturally handles unbalanced observation counts across users."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Treating 10,000 user-sessions as independent when they come from 100 users | Use random intercepts for users to account for within-user correlation |
| Using fixed effects for 200+ groups and trying to interpret each | Use random effects — you want the variance, not 200 individual estimates |
| Including random slopes when you have <5 observations per group | Start with random intercepts only; slopes need sufficient within-group variation |
| Ignoring convergence warnings ("model failed to converge") | Simplify random effects structure, center predictors, or increase iterations |
| Testing whether variance = 0 using standard LRT without boundary correction | Halve the p-value or use parametric bootstrap for proper null distribution |
| Using REML estimates for model comparison via likelihood ratio test | Use ML (not REML) when comparing models with different fixed effects |
| Not centering Level-1 predictors (group-mean centering) | Center to separate within-group and between-group effects and improve convergence |
| Reporting random effects as "the effect for group j" without noting shrinkage | Always clarify these are BLUPs (shrunken estimates), not raw group means |

---

## Rapid-Fire Q&A

**Q1: What's the difference between a mixed model and a multilevel model?**
> They're the same thing. "Mixed effects model," "multilevel model," "hierarchical linear model" (HLM), and "random coefficient model" are all synonyms.

**Q2: Can you have crossed random effects (not nested)?**
> Yes. Example: students take multiple tests across multiple schools (summer programs). Both student and school are random effects but not nested. Formula: `(1|student) + (1|school)`.

**Q3: What does the ICC tell you?**
> The proportion of total variance attributable to between-group differences. ICC = 0.3 means 30% of variation in the outcome is between groups, 70% is within groups.

**Q4: REML vs ML estimation — when to use which?**
> REML (Restricted Maximum Likelihood) gives unbiased variance estimates — use by default. ML is needed when comparing models with different fixed effects via LRT or AIC.

**Q5: What are BLUPs?**
> Best Linear Unbiased Predictors — the estimated random effects for each group. They're "shrunken" estimates that borrow strength from the population. Not truly unbiased (they're biased toward zero) but have minimum MSE.

**Q6: How many groups do you need for random effects?**
> Rule of thumb: at least 5-6 groups, ideally 20+. With fewer than 5, variance estimates are extremely imprecise. Consider fixed effects for 2-4 groups.

**Q7: Can mixed models handle non-normal outcomes?**
> Yes — Generalized Linear Mixed Models (GLMMs). Example: logistic regression with random effects for binary outcomes. Use `glmer()` in R or specify `family` in statsmodels.

**Q8: What's the relationship between mixed models and Bayesian hierarchical models?**
> Frequentist mixed models estimate point values for variance components. Bayesian hierarchical models put priors on these variances and get full posterior distributions. Bayesian approach is more natural (shrinkage emerges from the prior) and handles small samples better.

**Q9: How do mixed models relate to Ridge regression?**
> Random effects are mathematically equivalent to a Ridge penalty on group-specific coefficients. The variance ratio (σ²_e / σ²_u) acts like the regularization parameter λ.

**Q10: "My mixed model says the treatment effect is significant, but a simple t-test on user averages says it's not. Which is right?"**
> The mixed model is likely more powerful because it uses all observations (not just means) and models the correlation structure. However, verify model assumptions. If the t-test on user averages is properly computed, it's a valid (but less efficient) approach. The discrepancy usually means the mixed model gained power through modeling, not that one is "wrong."

---

## ASCII Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║              MIXED EFFECTS MODELS — QUICK REFERENCE                  ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  DATA STRUCTURE:                                                     ║
║  ┌─────────────────────────────────────┐                            ║
║  │ Level 2: Groups (j = 1,...,J)       │                            ║
║  │  ┌────────────────────────────────┐ │                            ║
║  │  │ Level 1: Obs (i = 1,...,n_j)   │ │                            ║
║  │  │  y_ij = (β₀+u₀ⱼ) + β₁x + ε   │ │                            ║
║  │  └────────────────────────────────┘ │                            ║
║  └─────────────────────────────────────┘                            ║
║                                                                      ║
║  THREE POOLING STRATEGIES:                                           ║
║  ┌──────────┐  ┌──────────────┐  ┌──────────────┐                  ║
║  │ COMPLETE │  │   PARTIAL    │  │     NO       │                   ║
║  │ POOLING  │  │   POOLING    │  │   POOLING    │                   ║
║  │(1 line)  │  │(mixed model) │  │(J lines)     │                   ║
║  │ Biased   │  │  BEST MSE    │  │ High var     │                   ║
║  └──────────┘  └──────────────┘  └──────────────┘                  ║
║                                                                      ║
║  DECISION TREE:                                                      ║
║  Is data grouped/nested?                                             ║
║    NO → Simple regression                                            ║
║    YES → How many groups?                                            ║
║          2-4  → Fixed effects (dummies)                              ║
║          5+   → Random effects                                       ║
║                  → Is effect of X same across groups?                ║
║                     YES → Random intercept only                      ║
║                     NO  → Random intercept + slope                   ║
║                                                                      ║
║  KEY FORMULAS:                                                       ║
║  ICC = σ²_u / (σ²_u + σ²_e)                                        ║
║  Shrinkage = n_j / (n_j + σ²_e/σ²_u)                               ║
║  LRT = 2(LL_complex - LL_simple) ~ χ²(df)                          ║
║                                                                      ║
║  PYTHON:                           R:                                ║
║  smf.mixedlm("y ~ x",             lmer(y ~ x + (1+x|group),        ║
║    data=df,                          data = df)                      ║
║    groups=df["grp"],                                                 ║
║    re_formula="~x")                                                  ║
║                                                                      ║
║  RED FLAGS IN INTERVIEWS:                                            ║
║  • "I'd use fixed effects for 1000 users" → Too many params         ║
║  • "Random effects don't matter if ICC is low" → Wrong if n large   ║
║  • "I'll just cluster-robust SE" → Only works with 30+ clusters     ║
║  • Ignoring convergence warnings → Results meaningless               ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 54 of 56*
