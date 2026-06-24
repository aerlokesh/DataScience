# 🎯 Topic 56: Linear Regression Deep Dive

> *"Every data scientist thinks they understand linear regression until a quant fund interviewer asks, 'What happens to your OLS estimates when you duplicate every observation?' Then the silence begins."*

---

## 📚 Table of Contents

1. [OLS Derivation](#ols-derivation)
2. [Gauss-Markov Assumptions](#gauss-markov-assumptions)
3. [What Happens When Assumptions Are Violated](#what-happens-when-assumptions-are-violated)
4. [Interpreting Coefficients](#interpreting-coefficients)
5. [R-Squared and Adjusted R-Squared](#r-squared-and-adjusted-r-squared)
6. [Hypothesis Testing](#hypothesis-testing)
7. [Multicollinearity and VIF](#multicollinearity-and-vif)
8. [Heteroscedasticity](#heteroscedasticity)
9. [Omitted Variable Bias and Endogeneity](#omitted-variable-bias-and-endogeneity)
10. [OLS vs GLS vs WLS](#ols-vs-gls-vs-wls)
11. [Regularization Connection](#regularization-connection)
12. [Duplicated Observations (Google's Favorite)](#duplicated-observations-googles-favorite)
13. [Explaining to Non-Technical Stakeholders](#explaining-to-non-technical-stakeholders)
14. [Interview Talking Points](#interview-talking-points)
15. [Common Mistakes](#common-mistakes)
16. [Rapid-Fire Q&A](#rapid-fire-qa)
17. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## OLS Derivation

### Setup and Derivation

```
Model: Y = Xβ + ε  (Y: n×1, X: n×(p+1), β: (p+1)×1, ε: n×1)

Minimize: S(β) = (Y - Xβ)ᵀ(Y - Xβ) = Σᵢ(yᵢ - xᵢᵀβ)²

∂S/∂β = -2Xᵀ(Y - Xβ) = 0
→ XᵀXβ̂ = XᵀY
→ β̂ = (XᵀX)⁻¹XᵀY    ← THE NORMAL EQUATIONS

Second-order: ∂²S/∂β∂βᵀ = 2XᵀX (positive definite if full rank) → minimum
```

### Geometric Interpretation

```
Ŷ = X(XᵀX)⁻¹Xᵀ Y = H·Y    (H = "hat matrix", projection matrix)

OLS projects Y onto the column space of X.
Residuals e = Y - Ŷ are ORTHOGONAL to column space of X: Xᵀe = 0
```

> **Critical Insight** 💡
> The normal equations reveal everything: β̂ = (XᵀX)⁻¹XᵀY. If XᵀX is not invertible (perfect multicollinearity), OLS has no unique solution. If XᵀX is nearly singular (high multicollinearity), the inverse amplifies small perturbations, making estimates unstable — this is why VIF matters.

---

## Gauss-Markov Assumptions

For OLS to be **BLUE** (Best Linear Unbiased Estimator):

| # | Assumption | Mathematical Statement | Plain English |
|---|-----------|----------------------|---------------|
| 1 | **Linearity** | E[Y|X] = Xβ | True relationship is linear in parameters |
| 2 | **Strict Exogeneity** | E[ε|X] = 0 | Errors are uncorrelated with all predictors |
| 3 | **Homoscedasticity** | Var(ε|X) = σ²Iₙ | Error variance is constant across observations |
| 4 | **No Autocorrelation** | Cov(εᵢ, εⱼ) = 0 for i ≠ j | Errors are independent of each other |
| 5 | **No Perfect Multicollinearity** | rank(X) = p+1 | No predictor is a perfect linear combination of others |

### Additional Assumption for Inference (not required for BLUE)

| 6 | **Normality** | ε ~ N(0, σ²I) | Errors are normally distributed |

### What BLUE Means

- **B**est: Lowest variance among all...
- **L**inear: ...estimators that are linear functions of Y
- **U**nbiased: ...and are unbiased (E[β̂] = β)
- **E**stimator

> **Critical Insight** 💡
> BLUE doesn't mean OLS is the best estimator *period*. It's only the best among *linear unbiased* estimators. A biased estimator (like Ridge) can have lower MSE by trading bias for reduced variance. And non-linear estimators (like LASSO) can outperform OLS. Gauss-Markov gives you a baseline, not an optimality guarantee.

---

## What Happens When Assumptions Are Violated

### Violation Consequence Table

| Assumption Violated | Effect on β̂ | Effect on SE(β̂) | Effect on Inference | Fix |
|--------------------|-------------|------------------|--------------------|----|
| **Linearity** | Biased | Meaningless | Invalid | Add polynomials, transforms, splines |
| **Exogeneity** (omitted var) | Biased | Biased | Invalid | IV, add confounder, diff-in-diff |
| **Homoscedasticity** | Unbiased | Biased (usually too small) | Invalid (wrong p-values) | WLS, robust SE (HC0-HC3) |
| **No Autocorrelation** | Unbiased | Biased (usually too small) | Invalid | GLS, Newey-West SE, cluster SE |
| **No Multicollinearity** | Unbiased but high variance | Inflated | Low power | Drop variables, PCA, Ridge |
| **Normality** | Unbiased | Correct (asymptotically) | Approximate (CLT kicks in for large n) | Bootstrap, robust methods |

### Key Insight for Each

**Linearity violation**: Everything breaks — no technical fix for wrong functional form except specifying the right one.

**Exogeneity violation**: The killer. If a predictor correlates with the error (because of reverse causality, measurement error, or omitted confounders), your coefficient is biased and inconsistent — more data doesn't help.

**Heteroscedasticity**: Your estimates are still unbiased! You just can't trust your t-statistics and p-values. Easy fix: use heteroscedasticity-robust standard errors.

**Autocorrelation**: Same as heteroscedasticity — estimates unbiased but SEs wrong. Common in time series and clustered data.

**Multicollinearity**: Estimates are unbiased but bouncy — tiny data changes cause big coefficient swings. Individual coefficients are unreliable but predictions can still be fine.

> **Critical Insight** 💡
> **Interview killer**: "If homoscedasticity is violated, is OLS biased?" Answer: **No!** OLS coefficients remain unbiased and consistent under heteroscedasticity. Only the standard errors (and thus p-values) are affected. This distinction between coefficient bias and SE bias is tested repeatedly at quant firms.

---

## Interpreting Coefficients

### Continuous Predictor

```
y = β₀ + β₁·x + ε
β₁: "A one-unit increase in x is associated with a β₁-unit change in y, 
     holding all else constant."
```

### Binary (Dummy) Predictor

```
y = β₀ + β₁·D + ε    (D ∈ {0, 1})
β₁: "The average difference in y between group D=1 and group D=0."
β₀: "The average y for the reference group (D=0)."
```

### Log-Transformed Variables

| Model | Interpretation of β₁ |
|-------|----------------------|
| y = β₀ + β₁·x | 1-unit ↑ in x → β₁ unit ↑ in y |
| ln(y) = β₀ + β₁·x | 1-unit ↑ in x → ~(β₁×100)% ↑ in y |
| y = β₀ + β₁·ln(x) | 1% ↑ in x → β₁/100 unit ↑ in y |
| ln(y) = β₀ + β₁·ln(x) | 1% ↑ in x → β₁% ↑ in y (elasticity) |

### Interaction Terms

```
y = β₀ + β₁·x₁ + β₂·x₂ + β₃·(x₁·x₂) + ε

Effect of x₁ on y = β₁ + β₃·x₂  (depends on x₂!)
β₁ alone: effect of x₁ when x₂ = 0
β₃: how much the effect of x₁ changes per unit of x₂
```

> **Critical Insight** 💡
> "Holding all else constant" is the most dangerous phrase in regression. It implies a ceteris paribus interpretation that only works if: (1) all confounders are controlled, (2) other variables can actually be held constant (no perfect multicollinearity with the varying predictor), and (3) the variation is meaningful (a 1-unit change in x while holding correlated z constant may be implausible).

---

## R-Squared and Adjusted R-Squared

### R-Squared (Coefficient of Determination)

```
R² = 1 - SS_res/SS_tot = 1 - Σ(yᵢ - ŷᵢ)² / Σ(yᵢ - ȳ)²

Equivalently: R² = SS_reg/SS_tot = Var(ŷ)/Var(y)
```

**Interpretation**: Proportion of variance in Y explained by the model.

### Adjusted R²

```
R²_adj = 1 - (1 - R²) · (n - 1)/(n - p - 1)
```

Penalizes for number of predictors. Unlike R², it CAN decrease when useless variables are added.

### When R² Is Misleading

| Situation | R² Behavior | Reality |
|-----------|-------------|---------|
| Added noise variable | Increases (slightly) | Model got worse |
| Duplicated observations | Unchanged | You didn't gain information |
| Comparing y vs log(y) models | Can't compare directly | Different scales |
| Time trend in both X and Y | Very high R² | Could be spurious |
| Small n, many p | Inflated | Overfitting |

> **Critical Insight** 💡
> **Interview question** (Two Sigma): "What's a good R²?" There is NO universal answer. For stock returns prediction, R² = 0.01 can be extremely profitable. For physics experiments, R² < 0.99 means something is wrong. R² depends on inherent noise in the domain. Never judge a model solely by R².

---

## Hypothesis Testing

```
t-test (individual):  t = β̂ⱼ / SE(β̂ⱼ) ~ t(n-p-1)
                      SE(β̂ⱼ) = √(σ̂² · [(XᵀX)⁻¹]ⱼⱼ),  σ̂² = RSS/(n-p-1)

F-test (overall):     F = (R²/p) / ((1-R²)/(n-p-1)) ~ F(p, n-p-1)
                      Tests H₀: all slopes = 0

F-test (nested):      F = ((RSS_restr - RSS_full)/(p₁-p₂)) / (RSS_full/(n-p₁))
                      Compares full model vs restricted model
```

**Key relationship**: For a single coefficient, F = t². They give identical p-values.

---

## Multicollinearity and VIF

### What Is Multicollinearity?

Predictors are highly correlated with each other. Not a bias problem — a **precision** problem.

### Variance Inflation Factor (VIF)

```
VIF_j = 1 / (1 - R²_j)

where R²_j is R-squared from regressing x_j on all other predictors.
```

| VIF Value | Interpretation | Action |
|-----------|---------------|--------|
| 1 | No collinearity | Fine |
| 1-5 | Moderate | Usually OK |
| 5-10 | High | Investigate |
| > 10 | Severe | Must address |

### Effects of Multicollinearity

```python
# With high multicollinearity:
# Var(β̂_j) = σ² · (XᵀX)⁻¹_jj = σ² · VIF_j / Var(x_j)

# VIF = 10 means: SE is √10 ≈ 3.2x larger than without collinearity
# VIF = 100 means: SE is 10x larger — coefficient could flip sign across samples
```

### Fixes

1. **Drop one variable** (if redundant)
2. **PCA** or factor analysis on correlated predictors
3. **Ridge regression** (L2 penalty stabilizes (XᵀX + λI)⁻¹)
4. **Combine variables** (create index/composite)
5. **Do nothing** — if you only care about prediction (not individual coefficients)

> **Critical Insight** 💡
> Multicollinearity doesn't affect prediction quality! Ŷ = Xβ̂ remains the best linear prediction regardless. It only matters when you want to interpret *individual* coefficients. If your goal is prediction, high VIF is irrelevant. If your goal is causal inference on a specific β, it's critical.

---

## Heteroscedasticity

### Detection

**Visual**: Plot residuals vs fitted values — look for fan/funnel shape.

**Formal Tests**:

| Test | Null Hypothesis | Procedure |
|------|----------------|-----------|
| White test | Homoscedasticity | Regress ε² on X, X², cross-products; test R² |
| Breusch-Pagan | Homoscedasticity | Regress ε² on X; LM = nR² ~ χ²(p) |
| Goldfeld-Quandt | Homoscedasticity | Split data, compare RSS of subsamples |

### Fixes

```python
import statsmodels.api as sm

# Heteroscedasticity-robust standard errors (HC3 recommended for small samples)
model = sm.OLS(y, X).fit(cov_type='HC3')

# Available types:
# HC0: White's original (asymptotic)
# HC1: Small-sample correction (multiply by n/(n-p))
# HC2: Divide by (1-h_ii)
# HC3: Divide by (1-h_ii)² ← most conservative, recommended
```

### Important Properties

- OLS with robust SE: coefficients same, only SEs change
- WLS (Weighted Least Squares): coefficients change, more efficient if weights correct
- Under heteroscedasticity: OLS is unbiased but **not** BLUE (GLS/WLS is more efficient)

---

## Omitted Variable Bias and Endogeneity

### Omitted Variable Bias Formula

```
True model:  Y = β₀ + β₁X + β₂Z + ε
You estimate: Y = α₀ + α₁X + u   (omitting Z)

Bias: E[α̂₁] - β₁ = β₂ · δ

where δ = coefficient from regressing Z on X

Bias = (effect of Z on Y) × (correlation of Z with X)
```

### Direction of Bias

| Sign of β₂ (Z→Y) | Sign of δ (X→Z corr) | Direction of Bias |
|-------------------|-----------------------|-------------------|
| Positive | Positive | Upward (overestimate) |
| Positive | Negative | Downward (underestimate) |
| Negative | Positive | Downward |
| Negative | Negative | Upward |

### Endogeneity Sources

1. **Omitted variables**: Confounders correlated with both X and Y
2. **Simultaneity**: X causes Y AND Y causes X (reverse causality)
3. **Measurement error in X**: Attenuates coefficient toward zero
4. **Sample selection**: Non-random inclusion in sample

### Fixes

| Problem | Solution |
|---------|----------|
| Omitted variable | Include it, or use proxy |
| Unobserved confounder | Instrumental variables (2SLS) |
| Simultaneity | IV, structural equations |
| Measurement error | IV, errors-in-variables models |
| Selection bias | Heckman correction, matching |

> **Critical Insight** 💡
> **This is the most important concept for causal inference interviews.** Adding more controls can help (if they're confounders) or hurt (if they're colliders or mediators). The question isn't "did you control for enough?" — it's "did you control for the *right* things?" Drawing a causal DAG before running regression should be standard practice.

---

## OLS vs GLS vs WLS

| Method | Assumption about errors | Estimator | When to Use |
|--------|------------------------|-----------|-------------|
| **OLS** | Var(ε) = σ²I (homoscedastic, uncorrelated) | (XᵀX)⁻¹XᵀY | Default, assumptions hold |
| **WLS** | Var(εᵢ) = σ²/wᵢ (heteroscedastic, uncorrelated) | (XᵀWX)⁻¹XᵀWY | Known variance pattern |
| **GLS** | Var(ε) = σ²Ω (any covariance structure) | (XᵀΩ⁻¹X)⁻¹XᵀΩ⁻¹Y | Correlated/heterosc. errors |
| **FGLS** | Ω estimated from data | Same as GLS with Ω̂ | Ω unknown (practical GLS) |

### WLS Example

```python
# If variance is proportional to x: Var(ε_i) = σ² · x_i
# Weight = 1/Var(ε_i) = 1/x_i

import statsmodels.api as sm

weights = 1.0 / df['x']  # inverse of known variance pattern
model = sm.WLS(y, X, weights=weights).fit()
```

### Relationship

```
WLS is a special case of GLS where Ω is diagonal.
OLS is a special case of WLS where all weights = 1.
OLS ⊂ WLS ⊂ GLS
```

---

## Regularization Connection

### Ridge Regression = OLS + L2 Penalty

```
OLS:   β̂ = argmin ||Y - Xβ||²
Ridge: β̂_λ = argmin ||Y - Xβ||² + λ||β||²

Solution:
OLS:   β̂ = (XᵀX)⁻¹XᵀY
Ridge: β̂_λ = (XᵀX + λI)⁻¹XᵀY
```

### Key Differences

| Property | OLS | Ridge | LASSO |
|----------|-----|-------|-------|
| Penalty | None | λΣβⱼ² (L2) | λΣ|βⱼ| (L1) |
| Bias | Unbiased | Biased (toward 0) | Biased (toward 0) |
| Variance | Higher | Lower | Lower |
| Feature selection | No | No (shrinks, never zeros) | Yes (sets coefficients to 0) |
| Closed form | Yes | Yes | No (requires optimization) |
| Handles multicollinearity | Poorly | Well | Somewhat |

### Why Ridge Helps with Multicollinearity

```
(XᵀX) may be nearly singular → (XᵀX)⁻¹ has huge entries → high variance

(XᵀX + λI) is ALWAYS invertible (eigenvalues shifted by λ)
→ Stable estimates even with collinear predictors
```

### Bias-Variance Tradeoff Visualization

```
MSE = Bias² + Variance

λ = 0 (OLS):     Bias = 0,    Variance = HIGH    MSE = moderate
λ small:          Bias = low,  Variance = LOWER   MSE = LOWER ✓
λ optimal:        Bias = mod,  Variance = LOW     MSE = MINIMUM ✓✓
λ → ∞:           Bias = HIGH, Variance → 0       MSE = HIGH
```

> **Critical Insight** 💡
> Ridge regression has a Bayesian interpretation: it's equivalent to OLS with a Gaussian prior β ~ N(0, σ²/λ · I) on the coefficients. The penalty λ controls how strongly you believe coefficients should be near zero before seeing data. This connects frequentist regularization to Bayesian shrinkage.

---

## Duplicated Observations (Google's Favorite)

### The Question

> "You accidentally duplicated every row in your dataset. What happens to your OLS regression?"

### The Answer

| Quantity | Effect of Duplication | Why |
|----------|---------------------|-----|
| β̂ (coefficients) | **UNCHANGED** | (X₂ᵀX₂)⁻¹X₂ᵀY₂ = (2XᵀX)⁻¹(2XᵀY) = (XᵀX)⁻¹XᵀY |
| R² | **UNCHANGED** | SS_reg and SS_tot both double, ratio same |
| Residuals | **UNCHANGED** | Same predictions, same errors |
| σ̂² | **UNCHANGED** | RSS doubles but df = 2n - (p+1) ≈ doubles too |
| SE(β̂) | **DECREASES by √2** | SE = σ̂/√(nVar(x)); n doubles, σ̂ same |
| t-statistics | **INCREASE by √2** | Same β̂, smaller SE |
| p-values | **DECREASE** | Larger t → smaller p |
| Confidence intervals | **NARROWER by √2** | Falsely precise |
| F-statistic | **INCREASES** | More "data" seems to confirm model |

### The Mathematical Proof

```
Original: n observations → β̂ = (XᵀX)⁻¹XᵀY, σ̂² = RSS/(n-p-1)
Doubled:  2n observations

X₂ = [X; X] (stack X twice)
Y₂ = [Y; Y] (stack Y twice)

X₂ᵀX₂ = 2·XᵀX
X₂ᵀY₂ = 2·XᵀY
β̂₂ = (2XᵀX)⁻¹(2XᵀY) = (XᵀX)⁻¹XᵀY = β̂  ✓ SAME

Var(β̂₂) = σ²(X₂ᵀX₂)⁻¹ = σ²(2XᵀX)⁻¹ = (1/2)·σ²(XᵀX)⁻¹ = (1/2)·Var(β̂)
→ SE decreases by factor √2
```

### Why This Matters

- It demonstrates that OLS doesn't know about **independent information**
- The model incorrectly treats n_duplicated as n_effective
- Standard errors assume each observation provides independent information
- **This is identical to the problem of correlated observations** — duplicates are perfectly correlated

### Follow-up: What About Weighted Duplication?

If some observations are duplicated more than others (unequal weights):
- Coefficients **DO change** (equivalent to WLS with different weights)
- This is actually the mechanism behind WLS: duplicating observation i means giving it more influence

> **Critical Insight** 💡
> **The meta-lesson**: OLS has no concept of "data quality" or "independence." It treats every row equally. When you duplicate rows, add correlated data, or have clustered observations, OLS happily reports tighter confidence intervals — but they're wrong. This is why understanding the assumption of independent observations is so critical in practice.

---

## Explaining to Non-Technical Stakeholders

> "Linear regression finds the line that best predicts the outcome based on inputs. Each input gets a coefficient: if that input increases by one unit, the prediction changes by that much — holding everything else constant. The confidence interval tells us how certain we are: narrow = reliable, wide = uncertain."

**Key analogies**: Coefficient = "a dial that moves the output"; p-value = "how surprised we'd be if there's actually no effect"; R² = "what % of variation we can explain"; Multicollinearity = "two linked dials — can't tell which does the work".

---

## Interview Talking Points

> **"Derive OLS for me."**
>
> "We minimize the sum of squared residuals: L = (Y - Xβ)ᵀ(Y - Xβ). Taking the derivative with respect to β and setting to zero: -2Xᵀ(Y - Xβ) = 0, which gives us the normal equations XᵀXβ = XᵀY, so β̂ = (XᵀX)⁻¹XᵀY. Geometrically, this projects Y onto the column space of X — the residuals are orthogonal to all predictors. For the inverse to exist, we need X to have full column rank, meaning no perfect multicollinearity."

> **"What assumptions does OLS require, and what breaks when they're violated?"**
>
> "The key distinction is between assumptions for *unbiasedness* versus assumptions for *correct inference*. For unbiased coefficients, we need linearity and exogeneity — if these fail, the estimates themselves are wrong. For correct standard errors, we additionally need homoscedasticity and no autocorrelation — if these fail, coefficients are still unbiased, but p-values and confidence intervals are wrong. The fix for the latter is easy: robust standard errors. The fix for the former requires rethinking the model — instrumental variables, adding confounders, or a different functional form."

> **"What happens when you duplicate all observations?"**
>
> "Coefficients stay exactly the same because β̂ = (XᵀX)⁻¹XᵀY and the factor of 2 cancels. But standard errors shrink by √2 because the model thinks it has twice as much independent information. This makes t-statistics √2 times larger and p-values smaller. The practical lesson is that OLS can't distinguish between genuine replication and copy-paste — it will always report falsely narrow confidence intervals when observations aren't truly independent. This is the same fundamental issue as clustered data or autocorrelation."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| "Heteroscedasticity causes biased coefficients" | OLS coefficients remain unbiased; only SEs are affected |
| Adding every possible variable to "control for everything" | Only control for confounders; don't control for mediators or colliders |
| Interpreting β₁ as causal without establishing exogeneity | State "association" unless you have a causal identification strategy |
| Using R² to compare models with different Y transformations | Can't compare R² between y vs ln(y) models — use out-of-sample metrics |
| "VIF > 5 means the model is wrong" | Multicollinearity is a precision issue, not a validity issue; predictions are fine |
| Dropping observations with high residuals as "outliers" | Investigate why — they could be the most informative points |
| Using p > 0.05 to conclude "no effect" | Absence of evidence ≠ evidence of absence; report CI width |
| Ignoring functional form (assuming linear = linear in variables) | "Linear regression" means linear in *parameters*; x² terms are fine |
| Not checking residual plots before reporting results | Always plot residuals vs fitted and Q-Q plot before trusting SEs |
| Standardizing Y and reporting "standardized β is causal" | Standardization doesn't fix endogeneity or omitted variable bias |

---

## Rapid-Fire Q&A

**Q1: What does "linear" in linear regression actually mean?**
> Linear in *parameters*, not in variables. y = β₀ + β₁x² + β₂log(x) is still a linear regression because β enters linearly. y = β₀ + x^β₁ is NOT linear regression.

**Q2: What happens if you add a totally random noise variable to the model?**
> R² increases (or stays same), Adjusted R² likely decreases, that coefficient's expected value is 0 but will have non-zero estimate due to sampling, and (by definition) it won't be significant on average. But in any given sample, it might appear "significant" by chance (5% of the time at α=0.05).

**Q3: Can R² ever decrease when adding a variable?**
> In OLS: mathematically impossible (R² is non-decreasing with more variables). Adjusted R² can decrease. In some regularized or constrained contexts, fit measures can decrease.

**Q4: If your model has R² = 0.95, is it a good model?**
> Not necessarily. Could be: overfitting (too many predictors relative to n), spurious correlation (both trending with time), or the model could still have badly biased coefficients (high R² doesn't prevent omitted variable bias).

**Q5: What does the intercept mean? Should you always include it?**
> Intercept = predicted Y when all X = 0. Almost always include it — dropping it forces the regression through the origin and can bias all other coefficients. Only drop if theory dictates Y must be 0 when all X are 0.

**Q6: OLS assumes normality of errors — how important is this?**
> For coefficient estimates: not at all (OLS doesn't require normality for unbiasedness or consistency). For finite-sample inference (t-tests, F-tests): moderately important. For large samples: CLT makes inference approximately valid regardless. Normality mainly matters for small-sample exact inference and prediction intervals.

**Q7: How does measurement error in X affect OLS?**
> Classical measurement error in X causes **attenuation bias** — coefficient shrinks toward zero. Intuition: noise in X makes the relationship with Y look weaker. Formula: plim(β̂) = β · σ²_x/(σ²_x + σ²_measurement). Fix: instrumental variables.

**Q8: Can you use linear regression for classification?**
> Technically yes (it's called Linear Probability Model), but predictions can fall outside [0,1] and heteroscedasticity is guaranteed. For binary outcomes, logistic regression is preferred. However, LPM is still used in economics because coefficients are directly interpretable as probability changes.

**Q9: What's the difference between correlation and the regression coefficient?**
> β̂₁ = r · (s_y/s_x) in simple regression. Correlation is unitless (standardized); regression coefficient has units of y per unit of x. Correlation is symmetric (corr(X,Y) = corr(Y,X)); regression is not (regressing Y on X ≠ regressing X on Y).

**Q10: Why would you prefer robust standard errors over WLS?**
> Robust SE (HC0-HC3) is safer because it doesn't require knowing the form of heteroscedasticity — it's consistent regardless. WLS is more efficient but requires correctly specifying weights. If you get the weights wrong, WLS gives wrong SEs AND wrong coefficients. When in doubt, use robust SE.

---

## ASCII Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║             LINEAR REGRESSION — DEEP DIVE REFERENCE                  ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  OLS SOLUTION:                                                       ║
║  β̂ = (XᵀX)⁻¹XᵀY        Var(β̂) = σ²(XᵀX)⁻¹                      ║
║  σ̂² = RSS/(n-p-1)        SE(β̂ⱼ) = √(σ̂²·[(XᵀX)⁻¹]ⱼⱼ)            ║
║                                                                      ║
║  GAUSS-MARKOV (for BLUE):                                           ║
║  1. Linearity: E[Y|X] = Xβ                                         ║
║  2. Exogeneity: E[ε|X] = 0          ← breaks → BIASED              ║
║  3. Homoscedasticity: Var(ε|X) = σ²I ← breaks → wrong SE           ║
║  4. No autocorrelation: Cov(εᵢ,εⱼ)=0 ← breaks → wrong SE          ║
║  5. Full rank X                       ← breaks → no solution        ║
║                                                                      ║
║  WHAT BREAKS WHAT:                                                   ║
║  ┌─────────────────────────────────────────────────────────┐        ║
║  │ Assumption Violated │ β̂ biased? │ SE biased? │ Fix      │        ║
║  │ Linearity           │ YES       │ YES       │ Transform │        ║
║  │ Exogeneity          │ YES       │ YES       │ IV/DAG    │        ║
║  │ Homoscedasticity    │ NO        │ YES       │ Robust SE │        ║
║  │ Autocorrelation     │ NO        │ YES       │ Cluster SE│        ║
║  │ Multicollinearity   │ NO        │ INFLATED  │ Ridge/PCA │        ║
║  └─────────────────────────────────────────────────────────┘        ║
║                                                                      ║
║  DUPLICATED DATA (GOOGLE QUESTION):                                  ║
║  β̂ → SAME    R² → SAME    SE → ÷√2    t → ×√2    p → SMALLER      ║
║  Lesson: OLS can't detect non-independence!                          ║
║                                                                      ║
║  OMITTED VARIABLE BIAS:                                              ║
║  Bias = β₂(omitted effect) × δ(corr with included)                  ║
║  Sign: (+)(+)=↑  (+)(-)=↓  (-)(+)=↓  (-)(-)=↑                     ║
║                                                                      ║
║  REGULARIZATION:                                                     ║
║  OLS:   (XᵀX)⁻¹XᵀY         ← unbiased, high variance             ║
║  Ridge: (XᵀX + λI)⁻¹XᵀY    ← biased, lower variance              ║
║  LASSO: No closed form       ← sparse (feature selection)           ║
║                                                                      ║
║  COEFFICIENT INTERPRETATION:                                         ║
║  y ~ x:       "1 unit ↑x → β₁ units ↑y"                           ║
║  ln(y) ~ x:   "1 unit ↑x → β₁×100 % ↑y"                          ║
║  y ~ ln(x):   "1% ↑x → β₁/100 units ↑y"                          ║
║  ln(y)~ln(x): "1% ↑x → β₁% ↑y" (elasticity)                      ║
║                                                                      ║
║  VIF: VIF_j = 1/(1-R²_j)   Rule: >10 = problem for inference       ║
║  R² = 1 - RSS/TSS           Adj R² penalizes for p                  ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 56 of 56*
