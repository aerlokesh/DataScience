# 🎯 Topic 61: Statistics & Probability Theory

> *"The foundation of every data science interview — from Bayes' theorem to hypothesis testing, explained for confident interview delivery."*

---

## 📋 Table of Contents

1. [Probability Foundations](#1-probability-foundations)
2. [Conditional Probability & Bayes' Theorem](#2-conditional-probability--bayes-theorem)
3. [Common Distributions](#3-common-distributions)
4. [Central Limit Theorem & Law of Large Numbers](#4-central-limit-theorem--law-of-large-numbers)
5. [Hypothesis Testing Framework](#5-hypothesis-testing-framework)
6. [Z-Test vs T-Test](#6-z-test-vs-t-test)
7. [Chi-Square Test & ANOVA](#7-chi-square-test--anova)
8. [Non-Parametric Tests](#8-non-parametric-tests)
9. [Confidence Intervals](#9-confidence-intervals)
10. [Correlation vs Causation](#10-correlation-vs-causation)
11. [Interview Talking Points](#11-interview-talking-points)
12. [Common Mistakes](#12-common-mistakes)
13. [Rapid-Fire Q&A](#13-rapid-fire-qa)
14. [ASCII Cheat Sheet](#14-ascii-cheat-sheet)

---

## 1. Probability Foundations

### The Three Axioms of Probability (Kolmogorov)

1. **Non-negativity**: P(A) >= 0 for any event A
2. **Normalization**: P(sample space) = 1
3. **Additivity**: For mutually exclusive events, P(A or B) = P(A) + P(B)

### Key Rules

| Rule | Formula | When to Use |
|------|---------|-------------|
| Addition Rule | P(A or B) = P(A) + P(B) - P(A and B) | Events can overlap |
| Multiplication Rule | P(A and B) = P(A) * P(B\|A) | Sequential events |
| Complement Rule | P(not A) = 1 - P(A) | Easier to compute "not happening" |
| Independence | P(A and B) = P(A) * P(B) | Events don't affect each other |

> **Critical Insight**: Independence and mutual exclusivity are NOT the same thing. If events are mutually exclusive (can't happen together), they are actually DEPENDENT — knowing one happened tells you the other didn't!

---

## 2. Conditional Probability & Bayes' Theorem

### Conditional Probability

P(A|B) = P(A and B) / P(B)

"Given that B has occurred, what's the probability of A?"

### Bayes' Theorem

```
P(A|B) = P(B|A) * P(A) / P(B)
```

**Components:**
- P(A|B) = Posterior (what we want to know)
- P(B|A) = Likelihood (how likely is the evidence given our hypothesis)
- P(A) = Prior (our initial belief)
- P(B) = Evidence (normalizing constant)

### The Classic Disease Testing Example

**Setup**: A disease affects 1% of the population. A test has:
- Sensitivity (True Positive Rate): 99%
- Specificity (True Negative Rate): 95%

**Question**: You test positive. What's the probability you actually have the disease?

```
P(Disease | Positive) = P(Positive | Disease) * P(Disease) / P(Positive)

P(Positive) = P(Pos|Disease)*P(Disease) + P(Pos|No Disease)*P(No Disease)
            = 0.99 * 0.01 + 0.05 * 0.99
            = 0.0099 + 0.0495
            = 0.0594

P(Disease | Positive) = 0.0099 / 0.0594 = 16.7%
```

> **Critical Insight**: Despite a "99% accurate" test, a positive result only means ~17% chance of disease! This is the BASE RATE FALLACY — when the condition is rare, even highly accurate tests produce many false positives relative to true positives.

### Interview Explanation Script

> "Bayes' theorem lets us update our beliefs based on new evidence. The key insight is that the prior probability matters enormously. In the disease example, even with a 99% accurate test, the low base rate (1%) means most positives are false positives. This is why we need confirmatory tests."

---

## 3. Common Distributions

### Distribution Comparison Table

| Distribution | Type | Parameters | Use When | Example |
|---|---|---|---|---|
| **Normal** | Continuous | mu, sigma | Natural phenomena, measurement errors | Heights, test scores |
| **Binomial** | Discrete | n, p | Fixed trials, binary outcomes | Coin flips, conversion rates |
| **Poisson** | Discrete | lambda | Counting events in fixed interval | Customer arrivals, bug reports/week |
| **Exponential** | Continuous | lambda | Time between events | Time between purchases |
| **Uniform** | Both | a, b | Equal probability across range | Random number generation |
| **Bernoulli** | Discrete | p | Single binary trial | Single coin flip |

### When to Use Each — Decision Framework

```
Is it counting successes in fixed trials?  --> Binomial
Is it counting events in a time/space interval?  --> Poisson
Is it measuring time between events?  --> Exponential
Is it a sum of many independent factors?  --> Normal
Is every value equally likely?  --> Uniform
```

### The Normal Distribution — Why It's Everywhere

The 68-95-99.7 Rule:
- 68% of data falls within 1 standard deviation
- 95% within 2 standard deviations
- 99.7% within 3 standard deviations

> **Critical Insight**: The Normal distribution appears everywhere NOT because natural things are inherently normal, but because of the Central Limit Theorem — sums and averages of many independent things tend toward normality regardless of the underlying distribution.

---

## 4. Central Limit Theorem & Law of Large Numbers

### Central Limit Theorem (CLT)

**Statement**: The sampling distribution of the sample mean approaches a normal distribution as sample size increases, regardless of the population's distribution.

**Requirements:**
- Independent observations
- Sample size sufficiently large (rule of thumb: n >= 30)
- Finite variance in the population

**Why It Matters in Data Science:**
1. Justifies using normal-based confidence intervals
2. Enables hypothesis testing even for non-normal data
3. Explains why ensemble methods work (averaging many weak learners)

### Law of Large Numbers (LLN)

**Statement**: As sample size increases, the sample mean converges to the population mean.

**Two Versions:**
- **Weak LLN**: Convergence in probability
- **Strong LLN**: Almost sure convergence (more rigorous)

### CLT vs LLN — The Distinction

| Aspect | CLT | LLN |
|--------|-----|-----|
| What it says | Shape of sampling distribution is normal | Sample mean converges to true mean |
| Focus | Distribution shape | Point convergence |
| Practical use | Building confidence intervals | Justifying larger samples |
| Key requirement | n >= 30 (approx) | n -> infinity |

> **Critical Insight**: CLT tells you the SHAPE of uncertainty (normal bell curve). LLN tells you the uncertainty SHRINKS. Together they say: "With enough data, your estimate is approximately normal and centered on the truth."

---

## 5. Hypothesis Testing Framework

### The 5-Step Framework

1. **State hypotheses**: H0 (null = no effect) vs H1 (alternative = there IS an effect)
2. **Choose significance level**: alpha (typically 0.05)
3. **Calculate test statistic**: From your data
4. **Find p-value**: Probability of seeing this result (or more extreme) IF H0 is true
5. **Decision**: Reject H0 if p-value < alpha

### Critical Definitions

| Term | Definition | Common Misinterpretation |
|------|------------|--------------------------|
| p-value | P(data this extreme \| H0 true) | NOT P(H0 true \| data) |
| Alpha | Threshold for rejecting H0 | NOT the probability of being wrong |
| Type I Error | Rejecting H0 when it's true (false positive) | "Crying wolf" |
| Type II Error | Failing to reject H0 when it's false (false negative) | "Missing the signal" |
| Power | 1 - P(Type II Error) | Ability to detect real effects |

> **Critical Insight**: A p-value of 0.03 does NOT mean "3% chance the null hypothesis is true." It means "IF the null were true, there's a 3% chance of seeing data this extreme." This is the most common misinterpretation in interviews!

### One-Tailed vs Two-Tailed Tests

| Aspect | One-Tailed | Two-Tailed |
|--------|-----------|------------|
| H1 | Directional (> or <) | Non-directional (!=) |
| When to use | You KNOW the direction a priori | Default; effect could go either way |
| Power | More powerful for that direction | Less powerful but more conservative |
| p-value | Half of two-tailed | Standard |

---

## 6. Z-Test vs T-Test

### Comparison Table

| Aspect | Z-Test | T-Test |
|--------|--------|--------|
| Population variance | Known | Unknown (estimated from sample) |
| Sample size | Large (n > 30) | Any size (especially n < 30) |
| Distribution used | Standard Normal | Student's t-distribution |
| Degrees of freedom | N/A | n - 1 |
| When they converge | Always applicable for large n | Approaches Z for large n |
| Practical usage | Rare (who knows pop variance?) | Very common |

### Types of T-Tests

1. **One-sample t-test**: Is this sample mean different from a known value?
2. **Independent two-sample t-test**: Are two group means different?
3. **Paired t-test**: Before/after measurements on the same subjects

### Assumptions for T-Test

- Data is continuous
- Sample is randomly selected
- Data is approximately normally distributed (less critical for n > 30 due to CLT)
- Equal variances (for independent two-sample; use Welch's t-test if violated)

> **Critical Insight**: In practice, you almost ALWAYS use a t-test because you almost never know the true population variance. The z-test is more of a theoretical concept for teaching, though it's used in some large-sample proportion tests.

---

## 7. Chi-Square Test & ANOVA

### Chi-Square Test

**Purpose**: Test relationships between categorical variables

**Two Types:**
1. **Goodness of Fit**: Does observed data match an expected distribution?
2. **Test of Independence**: Are two categorical variables independent?

**Example**: Does customer churn differ by subscription tier?

| | Churned | Retained | Total |
|--|---------|----------|-------|
| Basic | 45 | 155 | 200 |
| Premium | 20 | 180 | 200 |

**Assumptions:**
- Expected frequency >= 5 in each cell
- Independent observations
- Categorical data

### ANOVA (Analysis of Variance)

**Purpose**: Compare means across 3+ groups simultaneously

**Why not multiple t-tests?** Each t-test has alpha=0.05 error rate. With 3 groups, you'd need 3 comparisons, inflating overall error (family-wise error rate).

**One-Way ANOVA:**
- One independent variable (factor) with 3+ levels
- Tests: "Are ANY of the group means different?"
- Does NOT tell you WHICH groups differ (need post-hoc tests like Tukey's HSD)

**Assumptions:**
- Independence of observations
- Normality within groups
- Homogeneity of variances (Levene's test)

> **Critical Insight**: ANOVA tells you "something is different" but not "what is different." Always follow a significant ANOVA with post-hoc pairwise comparisons. And remember: ANOVA is actually a special case of linear regression!

---

## 8. Non-Parametric Tests

### When to Use Non-Parametric Tests

- Data violates normality assumption
- Ordinal data (rankings)
- Small samples where distribution is unknown
- Outliers that would distort parametric tests

### Non-Parametric Test Comparison Table

| Parametric Test | Non-Parametric Alternative | Data Type | Use Case |
|----------------|---------------------------|-----------|----------|
| One-sample t-test | Wilcoxon Signed-Rank | Ordinal/continuous | One group vs. known value |
| Independent t-test | Mann-Whitney U | Ordinal/continuous | Compare 2 independent groups |
| Paired t-test | Wilcoxon Signed-Rank | Ordinal/continuous | Before/after, same subjects |
| One-way ANOVA | Kruskal-Wallis | Ordinal/continuous | Compare 3+ independent groups |
| Pearson correlation | Spearman rank correlation | Ordinal | Monotonic relationship |

### Key Non-Parametric Tests Explained

**Mann-Whitney U Test:**
- Compares two independent groups
- Tests if one group tends to have larger values
- Works with ordinal data and non-normal distributions
- Based on ranking all observations

**Wilcoxon Signed-Rank Test:**
- Compares paired observations
- Tests if median difference is zero
- Accounts for both direction and magnitude of differences

**Kruskal-Wallis Test:**
- Non-parametric version of one-way ANOVA
- Compares 3+ independent groups
- Tests if groups come from same distribution
- If significant, follow with Dunn's test for pairwise comparisons

### Trade-offs

| Aspect | Parametric | Non-Parametric |
|--------|-----------|----------------|
| Power (when assumptions met) | Higher | Lower |
| Robustness to violations | Lower | Higher |
| Sample size needed | Moderate | Can work with smaller |
| Interpretability | Mean differences | Rank/distribution differences |

> **Critical Insight**: Non-parametric tests are NOT assumption-free! They still require independence. They're also less powerful when parametric assumptions ARE met — use them as alternatives, not defaults.

---

## 9. Confidence Intervals

### Correct Interpretation

**What a 95% CI means:**
"If we repeated this experiment many times, 95% of the calculated intervals would contain the true parameter."

**What it does NOT mean:**
- NOT "95% probability the true value is in this interval"
- NOT "95% of the data falls in this interval"
- The true value is FIXED; it's either in the interval or it's not

### CI Formula (for means)

```
CI = sample_mean +/- (critical_value * standard_error)

Where:
  standard_error = sample_std / sqrt(n)
  critical_value = 1.96 for 95% CI (z-based)
                   t-value for small samples
```

### Factors Affecting CI Width

| Factor | Effect on CI Width | Why |
|--------|-------------------|-----|
| Larger sample size | Narrower | SE decreases with sqrt(n) |
| Higher confidence level | Wider | Need bigger net to be more sure |
| More variability | Wider | Less certain about the mean |

### CI vs Hypothesis Testing

- They contain the SAME information
- If 95% CI for difference doesn't include 0 → reject H0 at alpha=0.05
- CI is often MORE informative (gives effect size + uncertainty)

> **Critical Insight**: In interviews, the #1 mistake candidates make is saying "there's a 95% probability the true value is in this interval." This is a FREQUENTIST confidence interval — the true value is fixed, not random. The randomness is in the INTERVAL, not the parameter.

---

## 10. Correlation vs Causation

### Why Correlation != Causation

**Three requirements for causal claims:**
1. Correlation (statistical association)
2. Temporal precedence (cause before effect)
3. No confounders (rule out alternative explanations)

### Common Confounding Patterns

| Pattern | Example |
|---------|---------|
| Common cause (confounder) | Ice cream sales & drowning (both caused by hot weather) |
| Reverse causation | Hospital visits & death (being sick causes both) |
| Selection bias | Successful companies & feature X (survivorship bias) |
| Coincidence | Nicolas Cage movies & pool drownings |

### Establishing Causation

| Method | Strength | Practical |
|--------|----------|-----------|
| Randomized Controlled Trial (RCT) | Gold standard | Expensive, not always ethical |
| A/B Testing | Strong (randomized) | Common in tech |
| Instrumental Variables | Moderate | Requires valid instrument |
| Difference-in-Differences | Moderate | Natural experiments |
| Propensity Score Matching | Moderate | Observational data |
| Granger Causality | Weak (temporal only) | Time series only |

> **Critical Insight**: In a data science interview, when someone shows you a correlation and asks "does X cause Y?", the correct answer is almost always: "Correlation alone can't establish causation. We'd need to run an A/B test or control for confounders. Let me describe what confounders might exist..."

---

## 11. Interview Talking Points

### "Explain p-value to a non-technical person"

> "Imagine you're flipping a coin and you get 8 heads in a row. The p-value answers: 'If this coin were fair, how surprising would this result be?' A small p-value means the result is very surprising under our assumption, suggesting our assumption might be wrong."

### "When would you use Bayesian vs Frequentist?"

> "Frequentist methods are great when you have lots of data and want objective, reproducible results — like A/B testing at scale. Bayesian methods shine when you have prior knowledge to incorporate, small samples, or need to update beliefs incrementally — like personalization systems that learn from each user interaction."

### "How do you choose the right statistical test?"

> "I follow a decision tree: First, what type of data? Continuous vs categorical. Second, how many groups? One, two, or more. Third, are assumptions met? Normality, independence, equal variances. If assumptions are violated, I switch to non-parametric alternatives. Finally, what's the question — difference in means, relationship, or distribution fit?"

### "Explain confidence intervals to a product manager"

> "A confidence interval gives us a range of plausible values for what we're measuring, along with our certainty level. For example, 'We're 95% confident the true conversion rate is between 4.2% and 5.8%.' Wider intervals mean more uncertainty — we might need more data. It's more useful than a single number because it shows both our best estimate AND how much we should trust it."

---

## 12. Common Mistakes

| ❌ Wrong | ✅ Right |
|----------|----------|
| "p=0.03 means 3% chance H0 is true" | "If H0 were true, 3% chance of this extreme result" |
| "95% probability the true mean is in the CI" | "95% of such intervals contain the true mean" |
| "Correlation implies causation" | "Correlation suggests association; causation needs experiments" |
| "Fail to reject = H0 is true" | "Fail to reject = insufficient evidence against H0" |
| "Larger p-value = larger effect" | "p-value depends on sample size AND effect size" |
| "Non-significant = no effect" | "Non-significant = effect not detected (could be underpowered)" |
| "Alpha = probability of being wrong" | "Alpha = probability of Type I error specifically" |
| "Use multiple t-tests for 3+ groups" | "Use ANOVA to control family-wise error rate" |
| "Normal distribution required for all tests" | "CLT allows relaxing normality for large samples" |
| "One-tailed test because result happened to be positive" | "Must specify direction BEFORE seeing data" |

---

## 13. Rapid-Fire Q&A

**Q1: What's the difference between population and sample?**
A: Population is the entire group; sample is a subset. We use samples to make inferences about populations because measuring everyone is impractical.

**Q2: What's a Type I vs Type II error in real terms?**
A: Type I = convicting an innocent person (false alarm). Type II = letting a guilty person go free (missed detection). There's always a trade-off.

**Q3: Why is sample size important?**
A: Larger samples reduce standard error, narrow confidence intervals, increase power to detect real effects, and make CLT kick in for non-normal data.

**Q4: What's the difference between standard deviation and standard error?**
A: SD measures spread of individual data points. SE measures precision of the sample mean estimate. SE = SD / sqrt(n), so it shrinks with more data.

**Q5: When should you use a non-parametric test?**
A: When data violates normality (especially with small n), is ordinal, has extreme outliers, or when you care about medians rather than means.

**Q6: What is statistical power and why does it matter?**
A: Power = P(rejecting H0 when it IS false). Low power means you might miss real effects. Increase power with larger n, larger effect sizes, or higher alpha.

**Q7: Can you have a statistically significant but practically meaningless result?**
A: Yes! With huge samples, tiny effects become "significant." Always report effect sizes alongside p-values. A 0.01% improvement might be significant with n=10M but meaningless.

**Q8: What does it mean for events to be independent?**
A: P(A|B) = P(A). Knowing B occurred doesn't change the probability of A. Example: coin flips are independent; card draws without replacement are NOT.

**Q9: How does the Poisson distribution relate to the Binomial?**
A: Poisson is the limit of Binomial when n is large, p is small, and np = lambda. It's for rare events in large populations.

**Q10: What's the difference between Bayesian and Frequentist statistics?**
A: Frequentist treats parameters as fixed and data as random (probability = long-run frequency). Bayesian treats parameters as random with distributions (probability = degree of belief), updates beliefs with data via Bayes' theorem.

---

## 14. ASCII Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║               STATISTICS & PROBABILITY CHEAT SHEET                  ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  PROBABILITY RULES:                                                  ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │ P(A or B) = P(A) + P(B) - P(A and B)                       │    ║
║  │ P(A and B) = P(A) * P(B|A)                                  │    ║
║  │ P(A|B) = P(B|A) * P(A) / P(B)  ← Bayes                    │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  HYPOTHESIS TESTING DECISION:                                        ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │ p-value < alpha  →  REJECT H0 (significant)                 │    ║
║  │ p-value >= alpha →  FAIL TO REJECT H0 (not significant)     │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  TEST SELECTION FLOWCHART:                                           ║
║                                                                      ║
║  Data type?                                                          ║
║  ├── Categorical → Chi-Square                                        ║
║  └── Continuous                                                      ║
║       ├── 1 group → One-sample t-test                                ║
║       ├── 2 groups                                                   ║
║       │    ├── Independent → Independent t-test / Mann-Whitney       ║
║       │    └── Paired → Paired t-test / Wilcoxon                     ║
║       └── 3+ groups → ANOVA / Kruskal-Wallis                        ║
║                                                                      ║
║  NORMAL DISTRIBUTION (68-95-99.7):                                   ║
║                                                                      ║
║           99.7%                                                      ║
║        ┌────┤├────┐                                                  ║
║        │  95%     │                                                  ║
║        │ ┌──┤├──┐ │                                                  ║
║        │ │ 68% │ │                                                   ║
║       _│_│__/\__│_│_                                                 ║
║      /  │ /    \ │  \                                                ║
║     /   │/      \│   \                                               ║
║    /    /|      |\    \                                               ║
║   ─────────────────────                                              ║
║   -3σ -2σ -1σ  μ  1σ  2σ  3σ                                       ║
║                                                                      ║
║  ERROR TYPES:                                                        ║
║  ┌─────────────────┬──────────────┬──────────────────┐              ║
║  │                 │ H0 True      │ H0 False         │              ║
║  ├─────────────────┼──────────────┼──────────────────┤              ║
║  │ Reject H0       │ Type I (α)   │ Correct (Power)  │              ║
║  │ Fail to Reject  │ Correct      │ Type II (β)      │              ║
║  └─────────────────┴──────────────┴──────────────────┘              ║
║                                                                      ║
║  DISTRIBUTION SELECTION:                                             ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │ Binary outcome, n trials        → Binomial(n, p)            │    ║
║  │ Events per interval             → Poisson(λ)                │    ║
║  │ Time between events             → Exponential(λ)            │    ║
║  │ Sum of many factors             → Normal(μ, σ²)             │    ║
║  │ Rare events, large n, small p   → Poisson ≈ Binomial       │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  BAYES THEOREM (intuition):                                          ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │ Updated Belief = (How well evidence fits hypothesis)         │    ║
║  │                  × (Prior belief)                            │    ║
║  │                  ÷ (How common is this evidence overall)     │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 61 of 65*
