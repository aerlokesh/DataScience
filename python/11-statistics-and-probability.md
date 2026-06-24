# 🎯 Topic 11: Statistics & Probability

> **Scope:** Descriptive statistics, probability distributions, hypothesis testing, confidence intervals, Bayesian vs Frequentist inference, Central Limit Theorem, p-values, and effect sizes. This is a **TOP-3 most important topic** for Data Science interviews at big tech companies. Expect 2-4 questions per loop covering both conceptual depth and applied scenarios.

---

## Table of Contents

1. [Descriptive Statistics](#1-descriptive-statistics)
2. [Probability Distributions](#2-probability-distributions)
3. [Central Limit Theorem (CLT)](#3-central-limit-theorem-clt)
4. [Hypothesis Testing Framework](#4-hypothesis-testing-framework)
5. [Specific Statistical Tests](#5-specific-statistical-tests)
6. [Confidence Intervals](#6-confidence-intervals)
7. [P-Values — What They Really Mean](#7-p-values--what-they-really-mean)
8. [Effect Size (Cohen's d)](#8-effect-size-cohens-d)
9. [Bayesian vs Frequentist Inference](#9-bayesian-vs-frequentist-inference)
10. [Parametric vs Non-Parametric Tests](#10-parametric-vs-non-parametric-tests)
11. [When to Use Which Test](#11-when-to-use-which-test)
12. [Python Code for Every Test](#12-python-code-for-every-test)
13. [Interview Talking Points & Scripts](#13-interview-talking-points--scripts)
14. [Common Interview Mistakes](#14-common-interview-mistakes)
15. [Rapid-Fire Q&A](#15-rapid-fire-qa)
16. [Summary Cheat Sheet](#16-summary-cheat-sheet)

---

## 1. Descriptive Statistics

### Measures of Central Tendency
- **Mean:** Sensitive to outliers; use for symmetric distributions
- **Median:** Robust to outliers; preferred for skewed data (e.g., income, latency)
- **Mode:** Useful for categorical data or multimodal distributions

### Measures of Spread
- **Variance / Standard Deviation:** Quantifies dispersion around the mean
- **IQR (Interquartile Range):** Robust spread measure; basis for boxplot whiskers
- **Coefficient of Variation (CV):** SD / Mean; enables comparison across scales

> **Critical Insight:** In interviews, always state which measure you would use AND why. Saying "I'd report the median with IQR because revenue data is right-skewed" demonstrates statistical maturity.

---

## 2. Probability Distributions

### Normal (Gaussian)
- Continuous; defined by mu and sigma
- 68-95-99.7 rule for 1/2/3 standard deviations
- Foundation for parametric tests

### Binomial
- Discrete; n trials, probability p of success
- Models: click-through rates, conversion events
- Approximates Normal when np >= 10 and n(1-p) >= 10

### Poisson
- Discrete; models count of events in fixed interval
- Parameter lambda = mean = variance
- Models: page visits per hour, defects per batch

### Exponential
- Continuous; models time between Poisson events
- Memoryless property: P(X > s+t | X > s) = P(X > t)
- Models: time between customer arrivals, server failures

> **Critical Insight:** Interviewers test whether you can match a real business problem to the right distribution. "Modeling the number of support tickets per day" = Poisson. "Modeling whether a user converts" = Bernoulli/Binomial.

---

## 3. Central Limit Theorem (CLT)

**Statement:** Regardless of the population distribution, the sampling distribution of the sample mean approaches a Normal distribution as n increases (typically n >= 30).

**Why it matters:**
- Justifies using z-tests and t-tests even when data is not perfectly normal
- Underpins confidence interval construction
- Explains why averages of metrics behave predictably in A/B tests

> **Critical Insight:** CLT applies to the *sampling distribution of the mean*, NOT to the raw data. A common interview trap is asking whether CLT makes your data normal — it does not.

---

## 4. Hypothesis Testing Framework

### The 5-Step Process
1. **State hypotheses:** H0 (null) and H1 (alternative)
2. **Choose significance level:** alpha (typically 0.05)
3. **Select test statistic:** Based on data type and assumptions
4. **Compute p-value:** Probability of observing data at least as extreme under H0
5. **Make decision:** Reject H0 if p < alpha

### Type I and Type II Errors

| Error Type | Definition | Consequence | Control |
|------------|-----------|-------------|---------|
| Type I (alpha) | Reject H0 when H0 is true | False positive; ship a bad feature | Set alpha (0.05, 0.01) |
| Type II (beta) | Fail to reject H0 when H1 is true | False negative; miss a real effect | Increase sample size, power |

### Statistical Power
- Power = 1 - beta = P(reject H0 | H1 is true)
- Aim for power >= 0.80
- Increases with: larger n, larger effect size, larger alpha

---

## 5. Specific Statistical Tests

### Z-Test
- Known population variance OR large sample (n > 30)
- Compares sample mean to population mean

### T-Test (Student's)
- Unknown population variance, small sample
- **One-sample:** Compare sample mean to known value
- **Two-sample (independent):** Compare means of two groups
- **Paired:** Compare means from same subjects (before/after)

### Chi-Square Test
- **Goodness of fit:** Does observed frequency match expected?
- **Test of independence:** Are two categorical variables related?
- Requires expected cell counts >= 5

### ANOVA (F-Test)
- Compare means across 3+ groups simultaneously
- Avoids multiple comparison inflation from repeated t-tests
- Assumes: normality, homogeneity of variance, independence
- Follow up with post-hoc tests (Tukey HSD) if significant

> **Critical Insight:** Never run multiple t-tests instead of ANOVA. With k=5 groups, 10 pairwise t-tests at alpha=0.05 gives a family-wise error rate of 1-(0.95)^10 = 0.40 — a 40% chance of at least one false positive.

---

## 6. Confidence Intervals

**Correct interpretation:** "If we repeated this experiment many times, 95% of the constructed intervals would contain the true parameter."

**Incorrect interpretation:** "There is a 95% probability that the true mean lies in this interval." (The parameter is fixed; the interval is random.)

### Formula (for mean, large sample):
CI = x_bar +/- z_(alpha/2) * (sigma / sqrt(n))

> **Critical Insight:** A confidence interval that does NOT contain zero (for a difference) is equivalent to rejecting H0 at the corresponding alpha level. This duality between CI and hypothesis testing is frequently tested.

---

## 7. P-Values — What They Really Mean

**Definition:** The probability of observing a test statistic at least as extreme as the one calculated, *assuming H0 is true*.

**What p=0.03 means:**
- If the null hypothesis were true (no real effect), there is a 3% chance of seeing data this extreme or more extreme
- It does NOT mean there is a 3% chance H0 is true
- It does NOT mean there is a 97% chance the effect is real

**What p-values do NOT tell you:**
- The magnitude of the effect
- Whether the result is practically significant
- The probability that H0 or H1 is true

> **Critical Insight:** A small p-value with a tiny effect size is common in big data. With n=10 million users, even a 0.01% lift can be "statistically significant." Always pair p-values with effect sizes and confidence intervals.

---

## 8. Effect Size (Cohen's d)

**Formula:** d = (mean1 - mean2) / pooled_SD

### Interpretation Guidelines

| Cohen's d | Interpretation | Example |
|-----------|---------------|---------|
| 0.2 | Small effect | Subtle UI change on engagement |
| 0.5 | Medium effect | New recommendation algorithm |
| 0.8 | Large effect | Major product redesign |

**Why it matters in interviews:**
- Statistical significance != practical significance
- Effect size is sample-size independent
- Essential for power analysis and experiment planning

> **Critical Insight:** Always report effect size alongside p-values. Saying "the result was significant (p=0.02) with a Cohen's d of 0.15" shows the interviewer you understand that statistical significance alone is insufficient for business decisions.

---

## 9. Bayesian vs Frequentist Inference

| Dimension | Frequentist | Bayesian |
|-----------|-------------|----------|
| **Probability definition** | Long-run frequency | Degree of belief |
| **Parameters** | Fixed but unknown | Random variables with distributions |
| **Prior information** | Not formally incorporated | Explicitly encoded as prior |
| **Result** | P-value, CI | Posterior distribution, credible interval |
| **Interpretation of CI/CrI** | "95% of intervals contain truth" | "95% probability parameter is here" |
| **Multiple comparisons** | Requires correction (Bonferroni) | Naturally handled via hierarchical models |
| **Sample size** | Requires pre-fixed n | Can update continuously |
| **Computational cost** | Low (closed-form) | Higher (MCMC sampling) |
| **Common in** | Clinical trials, regulatory | Tech A/B testing, personalization |

> **Critical Insight:** Big tech increasingly uses Bayesian methods for A/B testing because they allow "peeking" at results without inflating false positive rates — a major advantage for fast-moving product teams.

---

## 10. Parametric vs Non-Parametric Tests

| Aspect | Parametric | Non-Parametric |
|--------|-----------|----------------|
| **Assumption** | Data follows known distribution (usually Normal) | No distributional assumption |
| **Data type** | Continuous, interval/ratio scale | Ordinal, ranked, or non-normal continuous |
| **Power** | Higher (when assumptions met) | Lower (trades power for flexibility) |
| **Sample size** | Works with smaller n if assumptions hold | Often needs larger n for same power |
| **Examples** | t-test, ANOVA, Pearson r | Mann-Whitney U, Kruskal-Wallis, Spearman rho |
| **When to use** | Symmetric data, no extreme outliers | Skewed data, ordinal scales, small n with non-normality |

### When Would You Use a Non-Parametric Test?

1. Data is ordinal (e.g., survey ratings 1-5)
2. Distribution is heavily skewed and n is small
3. Significant outliers that cannot be removed
4. Sample size too small to verify normality
5. Comparing medians rather than means is more appropriate

> **Critical Insight:** Do not default to non-parametric tests "to be safe." If your data reasonably meets parametric assumptions, parametric tests are more powerful. Use Shapiro-Wilk or Q-Q plots to check.

---

## 11. When to Use Which Test

| Scenario | Test | Why |
|----------|------|-----|
| Compare one sample mean to known value | One-sample t-test | Continuous, unknown sigma |
| Compare two independent group means | Independent t-test / Welch's | Two groups, continuous outcome |
| Compare paired measurements | Paired t-test | Same subjects, before/after |
| Compare 3+ group means | One-way ANOVA | Avoids multiple comparison inflation |
| Association between two categorical variables | Chi-square test of independence | Both variables categorical |
| Compare two groups, non-normal data | Mann-Whitney U | Non-parametric alternative to t-test |
| Compare 3+ groups, non-normal data | Kruskal-Wallis | Non-parametric alternative to ANOVA |
| Correlation (continuous, normal) | Pearson r | Linear relationship, both normal |
| Correlation (ordinal or non-normal) | Spearman rho | Monotonic relationship |
| Proportion comparison (two groups) | Z-test for proportions | Large n, binary outcome |

---

## 12. Python Code for Every Test

### Descriptive Statistics

```python
import numpy as np
from scipy import stats

data = np.array([23, 25, 28, 30, 22, 27, 35, 29, 31, 26])

# Central tendency
mean = np.mean(data)
median = np.median(data)
mode = stats.mode(data, keepdims=True).mode[0]

# Spread
std = np.std(data, ddof=1)  # sample std (Bessel's correction)
iqr = stats.iqr(data)
cv = std / mean

print(f"Mean: {mean:.2f}, Median: {median:.2f}, Mode: {mode}")
print(f"Std: {std:.2f}, IQR: {iqr:.2f}, CV: {cv:.2f}")
```

### Z-Test (One Sample)

```python
from statsmodels.stats.weightstats import ztest

sample = np.random.normal(loc=52, scale=10, size=100)
z_stat, p_value = ztest(sample, value=50)
print(f"Z-statistic: {z_stat:.4f}, P-value: {p_value:.4f}")
```

### T-Tests

```python
from scipy import stats

# One-sample t-test
sample = np.random.normal(loc=105, scale=15, size=30)
t_stat, p_val = stats.ttest_1samp(sample, popmean=100)
print(f"One-sample t-test: t={t_stat:.4f}, p={p_val:.4f}")

# Independent two-sample t-test (Welch's)
group_a = np.random.normal(loc=50, scale=10, size=50)
group_b = np.random.normal(loc=55, scale=12, size=50)
t_stat, p_val = stats.ttest_ind(group_a, group_b, equal_var=False)
print(f"Welch's t-test: t={t_stat:.4f}, p={p_val:.4f}")

# Paired t-test
before = np.random.normal(loc=70, scale=8, size=30)
after = before + np.random.normal(loc=5, scale=3, size=30)
t_stat, p_val = stats.ttest_rel(before, after)
print(f"Paired t-test: t={t_stat:.4f}, p={p_val:.4f}")
```

### Chi-Square Test

```python
from scipy.stats import chi2_contingency

# Contingency table: rows=device, cols=converted/not
observed = np.array([[120, 880],   # mobile
                     [200, 800]])  # desktop
chi2, p_val, dof, expected = chi2_contingency(observed)
print(f"Chi-square: {chi2:.4f}, p={p_val:.4f}, dof={dof}")
print(f"Expected frequencies:\n{expected}")
```

### ANOVA

```python
from scipy import stats

group1 = np.random.normal(loc=20, scale=5, size=30)
group2 = np.random.normal(loc=22, scale=5, size=30)
group3 = np.random.normal(loc=25, scale=5, size=30)

f_stat, p_val = stats.f_oneway(group1, group2, group3)
print(f"ANOVA: F={f_stat:.4f}, p={p_val:.4f}")

# Post-hoc Tukey HSD
from statsmodels.stats.multicomp import pairwise_tukeyhsd

all_data = np.concatenate([group1, group2, group3])
labels = ['G1']*30 + ['G2']*30 + ['G3']*30
tukey = pairwise_tukeyhsd(all_data, labels, alpha=0.05)
print(tukey)
```

### Mann-Whitney U (Non-Parametric)

```python
from scipy.stats import mannwhitneyu

group_a = np.random.exponential(scale=5, size=40)
group_b = np.random.exponential(scale=7, size=40)

u_stat, p_val = mannwhitneyu(group_a, group_b, alternative='two-sided')
print(f"Mann-Whitney U: U={u_stat:.4f}, p={p_val:.4f}")
```

### Cohen's d (Effect Size)

```python
def cohens_d(group1, group2):
    n1, n2 = len(group1), len(group2)
    var1, var2 = np.var(group1, ddof=1), np.var(group2, ddof=1)
    pooled_std = np.sqrt(((n1 - 1) * var1 + (n2 - 1) * var2) / (n1 + n2 - 2))
    return (np.mean(group1) - np.mean(group2)) / pooled_std

control = np.random.normal(loc=50, scale=10, size=200)
treatment = np.random.normal(loc=52, scale=10, size=200)
d = cohens_d(treatment, control)
print(f"Cohen's d: {d:.4f}")
print(f"Interpretation: {'Small' if abs(d)<0.5 else 'Medium' if abs(d)<0.8 else 'Large'}")
```

### Confidence Interval

```python
from scipy import stats
import numpy as np

data = np.random.normal(loc=100, scale=15, size=50)
confidence = 0.95

mean = np.mean(data)
se = stats.sem(data)
ci = stats.t.interval(confidence, df=len(data)-1, loc=mean, scale=se)
print(f"Mean: {mean:.2f}")
print(f"95% CI: ({ci[0]:.2f}, {ci[1]:.2f})")
```

### Power Analysis

```python
from statsmodels.stats.power import TTestIndPower

analysis = TTestIndPower()

# Calculate required sample size
effect_size = 0.3  # small-medium effect
alpha = 0.05
power = 0.80
n = analysis.solve_power(effect_size=effect_size, alpha=alpha, power=power)
print(f"Required sample size per group: {n:.0f}")

# Calculate power for given n
achieved_power = analysis.solve_power(effect_size=0.3, nobs1=200, alpha=0.05)
print(f"Power with n=200 per group: {achieved_power:.4f}")
```

---

## 13. Interview Talking Points & Scripts

### On "What does p=0.03 mean?"

> *"A p-value of 0.03 means that if the null hypothesis were true — meaning there is genuinely no difference between our groups — we would observe a test statistic this extreme or more extreme only 3% of the time by random chance alone. It does NOT mean there is a 3% probability that the null hypothesis is true, nor does it tell us the magnitude of the effect. That is why I always pair the p-value with a confidence interval and an effect size measure like Cohen's d to give a complete picture of both statistical and practical significance."*

### On "When would you use a non-parametric test?"

> *"I would choose a non-parametric test in four scenarios. First, when the data is ordinal rather than continuous — for example, user satisfaction ratings on a 1-to-5 scale. Second, when the data is heavily skewed and the sample size is too small for the Central Limit Theorem to provide adequate normality of the sampling distribution. Third, when there are significant outliers that are genuine data points and should not be removed. And fourth, when I am specifically interested in comparing medians or distributions rather than means. However, I would not default to non-parametric tests unnecessarily because they have lower statistical power when parametric assumptions are actually met."*

### On "How do you handle multiple comparisons?"

> *"When testing multiple hypotheses simultaneously, the probability of at least one false positive inflates rapidly — this is the family-wise error rate problem. My approach depends on context. For a small number of pre-planned comparisons, I use Bonferroni correction, dividing alpha by the number of tests. For post-hoc pairwise comparisons after ANOVA, I use Tukey's HSD which controls the family-wise error rate while being less conservative than Bonferroni. In high-dimensional settings like genomics or feature selection with hundreds of tests, I control the False Discovery Rate using the Benjamini-Hochberg procedure, which is more powerful at the cost of allowing a controlled proportion of false positives."*

### On "Bayesian vs Frequentist for A/B testing?"

> *"In my experience at scale, Bayesian A/B testing offers practical advantages for product teams. The key benefit is that Bayesian methods allow continuous monitoring — you can check results daily without inflating your false positive rate, whereas frequentist sequential testing requires pre-specified stopping rules. Bayesian results are also more intuitive for stakeholders: I can say 'there is a 92% probability that variant B is better' rather than explaining what 'failing to reject the null' means. The tradeoff is that you need to specify a prior, but with sufficient data the prior's influence diminishes quickly. I would still use frequentist methods when regulatory requirements demand them or when I need the simplicity of a pre-registered, fixed-horizon test."*

---

## 14. Common Interview Mistakes

| Mistake | Why It Is Wrong | Correct Approach |
|---------|----------------|------------------|
| "There is a 95% probability the true mean is in this CI" | CI is frequentist — the parameter is fixed, the interval is random | "95% of similarly constructed intervals would contain the true mean" |
| Running multiple t-tests instead of ANOVA | Inflates Type I error rate exponentially | Use ANOVA, then post-hoc tests if significant |
| Ignoring effect size when p is small | Large n can make trivial differences significant | Always report Cohen's d or practical lift alongside p |
| P-hacking: testing many variables until one is significant | Inflates false discovery rate; results won't replicate | Pre-register hypotheses; apply multiple comparison corrections |
| Confusing "fail to reject H0" with "H0 is true" | Absence of evidence is not evidence of absence | State: "insufficient evidence to conclude a difference exists" |
| Using parametric tests on ordinal data | Violates interval-scale assumption | Use Mann-Whitney U or Kruskal-Wallis for ordinal data |
| Not checking test assumptions before applying | Violated assumptions produce unreliable p-values | Check normality (Shapiro-Wilk), variance homogeneity (Levene's) |
| Reporting one-tailed p-value without justification | Often done to artificially halve the p-value | Use two-tailed unless direction was hypothesized a priori |
| Stopping an experiment early when p < 0.05 | "Peeking" inflates false positive rate in frequentist framework | Use sequential testing methods or Bayesian approaches |

---

## 15. Rapid-Fire Q&A

**Q1: What is the difference between standard deviation and standard error?**
SD measures variability in the data; SE measures variability of the sample mean (SE = SD / sqrt(n)). SE decreases with more data; SD does not.

**Q2: When is the median preferred over the mean?**
When data is skewed (income, latency, revenue) or contains outliers. The median is robust to extreme values.

**Q3: Can you have a significant p-value but no practical significance?**
Yes. With very large n, even trivially small differences become statistically significant. Always check effect size.

**Q4: What does a 95% confidence interval of [1.2, 3.8] for a difference mean?**
We are 95% confident the true difference lies between 1.2 and 3.8. Since zero is not included, we reject H0 at alpha=0.05.

**Q5: How does sample size affect hypothesis testing?**
Larger n increases power (reduces Type II error), narrows confidence intervals, and can detect smaller effects — but also makes trivial effects "significant."

**Q6: What is the relationship between confidence level and interval width?**
Higher confidence (e.g., 99% vs 95%) produces wider intervals. There is a precision-confidence tradeoff.

**Q7: Why can't you just increase alpha to get more power?**
Increasing alpha (e.g., from 0.05 to 0.10) increases power but also increases Type I error rate. The proper way to increase power is to increase sample size.

**Q8: What is the Law of Large Numbers vs CLT?**
LLN: sample mean converges to population mean as n grows. CLT: the distribution of sample means becomes Normal regardless of population shape.

**Q9: How do you determine sample size for an A/B test?**
Specify: minimum detectable effect (MDE), desired power (0.80), significance level (0.05), and baseline metric variance. Use power analysis formula or simulation.

**Q10: What is the difference between a credible interval and a confidence interval?**
Credible interval (Bayesian): 95% probability the parameter is in this range. Confidence interval (Frequentist): 95% of such intervals would contain the parameter.

**Q11: When would you use a one-tailed vs two-tailed test?**
One-tailed only when you have strong a priori reason to test in one direction AND you would not act on an effect in the opposite direction. Default to two-tailed.

**Q12: What is the assumption of homoscedasticity?**
Equal variances across groups. Required for standard t-test and ANOVA. Use Levene's test to check; use Welch's t-test if violated.

---

## 16. Summary Cheat Sheet

```
+============================================================================+
|                    STATISTICS & PROBABILITY CHEAT SHEET                      |
+============================================================================+
|                                                                              |
|  DISTRIBUTIONS:                                                              |
|    Normal    -> Continuous, symmetric, mu/sigma      (heights, errors)       |
|    Binomial  -> Discrete, n trials, prob p           (conversions, CTR)      |
|    Poisson   -> Discrete, count per interval         (events/hour)           |
|    Exponential -> Continuous, time between events    (inter-arrival time)    |
|                                                                              |
|  HYPOTHESIS TESTING DECISION TREE:                                           |
|    1 group mean vs known       -> One-sample t-test                          |
|    2 independent group means   -> Independent t-test (Welch's)               |
|    2 paired measurements       -> Paired t-test                              |
|    3+ group means              -> ANOVA -> Tukey HSD                         |
|    2 categorical variables     -> Chi-square independence                    |
|    Non-normal / ordinal        -> Mann-Whitney U / Kruskal-Wallis            |
|                                                                              |
|  KEY FORMULAS:                                                               |
|    SE = SD / sqrt(n)                                                         |
|    CI = x_bar +/- z * SE                                                     |
|    Cohen's d = (M1 - M2) / pooled_SD                                        |
|    Power = 1 - beta (target >= 0.80)                                         |
|                                                                              |
|  GOLDEN RULES:                                                               |
|    - Always pair p-value with effect size                                    |
|    - CI excluding 0 <=> reject H0 at corresponding alpha                    |
|    - CLT applies to sampling distribution, NOT raw data                      |
|    - "Fail to reject H0" != "H0 is true"                                    |
|    - Pre-register hypotheses to avoid p-hacking                              |
|    - Bayesian for continuous monitoring; Frequentist for fixed-horizon        |
|                                                                              |
|  EFFECT SIZE BENCHMARKS (Cohen's d):                                         |
|    Small = 0.2  |  Medium = 0.5  |  Large = 0.8                             |
|                                                                              |
+============================================================================+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 11 of 25*
