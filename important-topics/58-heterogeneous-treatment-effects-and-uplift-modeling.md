# 🎯 Topic 58: Heterogeneous Treatment Effects & Uplift Modeling

> *"Not everyone benefits equally from treatment — the real question isn't 'does it work on average?' but 'for whom does it work best?'"*

---

## Table of Contents

1. [What Are Heterogeneous Treatment Effects?](#what-are-htes)
2. [CATE: Conditional Average Treatment Effect](#cate)
3. [Uplift Modeling Approaches](#uplift-modeling-approaches)
4. [T-Learner (Two-Model Approach)](#t-learner)
5. [S-Learner (Single-Model Approach)](#s-learner)
6. [X-Learner (Cross-Estimation)](#x-learner)
7. [Causal Forests](#causal-forests)
8. [Targeting Policies](#targeting-policies)
9. [Evaluation: Qini Curves and AUUC](#evaluation)
10. [Connection to Personalization](#connection-to-personalization)
11. [Practical Applications](#practical-applications)
12. [Python Implementation](#python-implementation)
13. [Interview Talking Points](#interview-talking-points)
14. [Common Mistakes](#common-mistakes)
15. [Rapid-Fire Q&A](#rapid-fire-qa)
16. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## What Are Heterogeneous Treatment Effects? {#what-are-htes}

The **Average Treatment Effect (ATE)** tells you the mean impact across the whole population. But this average hides critical variation:

```
Population ATE = +5% conversion lift

Decomposed by segment:
  Power users (20%):     +15% lift  (strong responders)
  Casual users (50%):    +3% lift   (mild responders)
  Price-sensitive (20%): +0% lift   (indifferent)
  Annoyed users (10%):   -8% lift   (negative responders!)
```

> **Critical Insight:** Treating everyone uniformly wastes budget on indifferent users AND actively harms negative responders. HTE estimation lets you build targeting policies that treat only those who benefit, typically delivering 2-3x ROI improvement over blanket rollout.

### The Four Customer Segments (Uplift Quadrant)

| Segment | Without Treatment | With Treatment | Uplift | Action |
|---------|-------------------|----------------|--------|--------|
| **Sure Things** | Convert | Convert | 0 | Don't treat (save cost) |
| **Persuadables** | Don't convert | Convert | Positive | TARGET THESE |
| **Lost Causes** | Don't convert | Don't convert | 0 | Don't treat (save cost) |
| **Sleeping Dogs** | Convert | Don't convert | Negative | NEVER treat |

The goal of uplift modeling is to identify persuadables (maximize positive targeting) while avoiding sleeping dogs (prevent harm).

---

## CATE: Conditional Average Treatment Effect {#cate}

```
CATE(x) = E[Y(1) | X=x] - E[Y(0) | X=x]

Y(1) = potential outcome under treatment
Y(0) = potential outcome under control
X    = feature vector (age, usage history, etc.)

Fundamental problem: we observe Y_obs = T*Y(1) + (1-T)*Y(0)
=> Never see both potential outcomes for same individual
=> Must ESTIMATE the counterfactual
```

> **Critical Insight:** CATE estimation requires **randomized data** (from an A/B test) or valid causal identification. You cannot estimate uplift from purely observational data without strong assumptions. Always start with experimental data from a properly randomized test.

### CATE vs ATE vs ATT

| Estimand | Definition | Use Case |
|----------|-----------|----------|
| **ATE** | E[Y(1) - Y(0)] | Overall population effect |
| **ATT** | E[Y(1) - Y(0) \| T=1] | Effect on the treated |
| **CATE** | E[Y(1) - Y(0) \| X=x] | Effect for subgroup with features x |

---

## Uplift Modeling Approaches {#uplift-modeling-approaches}

### Method Comparison Table

| Method | How It Works | Pros | Cons | Best For |
|--------|-------------|------|------|----------|
| **T-Learner** | Two separate models | Simple, flexible | High variance | Large datasets |
| **S-Learner** | Single model, T as feature | Shares info, lower variance | May ignore treatment | Small effects |
| **X-Learner** | Cross-estimation | Works with imbalanced groups | Complex | Unequal splits |
| **Causal Forest** | Adapted random forest | CIs, non-parametric | Expensive | Complex heterogeneity |
| **DR-Learner** | Doubly robust | Robust to misspec | Needs propensity | Observational data |

---

## T-Learner (Two-Model Approach) {#t-learner}

The simplest uplift method: train two separate models, one on treatment data, one on control.

```python
from sklearn.ensemble import GradientBoostingClassifier
import numpy as np

class TLearner:
    """Two-model approach: separate models for treatment and control."""
    
    def __init__(self):
        self.model_treatment = GradientBoostingClassifier(n_estimators=100, max_depth=5)
        self.model_control = GradientBoostingClassifier(n_estimators=100, max_depth=5)
    
    def fit(self, X, treatment, y):
        """Fit separate models on treatment and control subsets."""
        self.model_treatment.fit(X[treatment == 1], y[treatment == 1])
        self.model_control.fit(X[treatment == 0], y[treatment == 0])
        return self
    
    def predict_uplift(self, X):
        """CATE = E[Y|X, T=1] - E[Y|X, T=0]"""
        p_treat = self.model_treatment.predict_proba(X)[:, 1]
        p_control = self.model_control.predict_proba(X)[:, 1]
        return p_treat - p_control
```

**Limitation:** Each model only sees half the data. The subtraction of two noisy estimates amplifies variance. Works best with large datasets where each half still has sufficient signal.

---

## S-Learner (Single-Model Approach) {#s-learner}

Use ONE model with treatment as just another input feature:

```python
class SLearner:
    """Single model with treatment indicator as a feature."""
    
    def __init__(self):
        self.model = GradientBoostingClassifier(n_estimators=200, max_depth=6)
    
    def fit(self, X, treatment, y):
        """Fit single model with treatment appended as feature."""
        X_augmented = np.column_stack([X, treatment])
        self.model.fit(X_augmented, y)
        return self
    
    def predict_uplift(self, X):
        """Compare predictions at T=1 vs T=0 for same features."""
        X_treat = np.column_stack([X, np.ones(len(X))])
        X_control = np.column_stack([X, np.zeros(len(X))])
        p_treat = self.model.predict_proba(X_treat)[:, 1]
        p_control = self.model.predict_proba(X_control)[:, 1]
        return p_treat - p_control
```

> **Critical Insight:** The S-Learner often **underestimates heterogeneity** because regularization shrinks treatment interactions toward zero. Tree-based models may not even split on the treatment variable if other features are more predictive of the outcome. Best when effects are subtle and you need to share information across groups.

---

## X-Learner (Cross-Estimation) {#x-learner}

The X-Learner uses cross-predictions to impute counterfactual outcomes, especially powerful when group sizes are imbalanced (e.g., 90% treated, 10% control).

```python
class XLearner:
    """Cross-estimation: impute individual effects, then model them."""
    
    def fit(self, X, treatment, y):
        X_t, y_t = X[treatment == 1], y[treatment == 1]
        X_c, y_c = X[treatment == 0], y[treatment == 0]
        
        # Step 1: Fit outcome models on each group
        self.mu0 = GradientBoostingRegressor(n_estimators=100).fit(X_c, y_c)
        self.mu1 = GradientBoostingRegressor(n_estimators=100).fit(X_t, y_t)
        
        # Step 2: Impute individual treatment effects via cross-prediction
        D1 = y_t - self.mu0.predict(X_t)  # Treated: actual - predicted control
        D0 = self.mu1.predict(X_c) - y_c  # Control: predicted treat - actual
        
        # Step 3: Model the imputed effects
        self.tau1 = GradientBoostingRegressor(n_estimators=100).fit(X_t, D1)
        self.tau0 = GradientBoostingRegressor(n_estimators=100).fit(X_c, D0)
        
        # Step 4: Propensity for weighted combination
        self.propensity = GradientBoostingClassifier().fit(X, treatment)
        return self
    
    def predict_uplift(self, X):
        """Weighted average of both CATE estimates."""
        e = self.propensity.predict_proba(X)[:, 1]  # P(T=1|X)
        return e * self.tau0.predict(X) + (1 - e) * self.tau1.predict(X)
```

**Key advantage:** When you have 95% treatment and 5% control, the X-Learner uses the large treated group's model to help estimate effects for the small control group.

---

## Causal Forests {#causal-forests}

Causal forests (Wager & Athey, 2018) adapt random forests to directly estimate CATE with valid confidence intervals via "honest" estimation.

### Key Design: Honesty

```
Standard RF: Same data for splitting AND estimation -> overfitting
Causal Forest: Split data into:
  - "Splitting sample" (determines tree structure)
  - "Estimation sample" (estimates leaf-level effects)
This enables valid statistical inference on CATE.
```

```python
from econml.dml import CausalForestDML

def fit_causal_forest(X, treatment, y, W=None):
    """
    X: effect modifiers (what drives heterogeneity)
    W: additional controls (not effect modifiers)
    """
    cf = CausalForestDML(
        n_estimators=1000,
        min_samples_leaf=10,
        criterion='het',       # Split on heterogeneity, not prediction
        honest=True,           # Separate split/estimation samples
        inference=True         # Enable confidence intervals
    )
    cf.fit(y, T=treatment, X=X, W=W)
    
    cate = cf.effect(X)
    ci_lower, ci_upper = cf.effect_interval(X, alpha=0.05)
    importance = cf.feature_importances_
    
    print(f"Mean CATE: {cate.mean():.4f}")
    print(f"Std CATE: {cate.std():.4f}")
    print(f"% with significant positive effect: "
          f"{(ci_lower > 0).mean()*100:.1f}%")
    return cf, cate, (ci_lower, ci_upper)
```

> **Critical Insight:** Causal forests provide confidence intervals. If CATE = +2% but 95% CI is [-1%, +5%], you may not want to confidently target this user. Use the CI lower bound for conservative targeting policies where avoiding harm is paramount.

---

## Targeting Policies {#targeting-policies}

Once you have CATE estimates, build a targeting policy to decide who receives treatment:

| Policy | Rule | Use Case |
|--------|------|----------|
| **Treat All** | Everyone | ATE > 0, treatment is cheap |
| **Top-K** | Top K% by CATE | Fixed budget constraint |
| **Threshold** | CATE > c | Known cost per treatment |
| **Cost-Sensitive** | CATE > cost(x) | Variable treatment cost |
| **Conservative** | CI_lower > 0 | Risk-averse, avoid harm |

```python
def optimal_targeting(cate_estimates, treatment_costs, budget):
    """Maximize total uplift subject to budget constraint."""
    # ROI per user = uplift / cost
    roi = cate_estimates / treatment_costs
    ranked = np.argsort(-roi)
    
    treated = np.zeros(len(cate_estimates), dtype=bool)
    remaining_budget = budget
    
    for idx in ranked:
        if treatment_costs[idx] <= remaining_budget and cate_estimates[idx] > 0:
            treated[idx] = True
            remaining_budget -= treatment_costs[idx]
    
    total_uplift = cate_estimates[treated].sum()
    print(f"Treat {treated.sum()} / {len(treated)} users ({treated.mean()*100:.1f}%)")
    print(f"Total uplift: {total_uplift:.2f}, Cost: ${budget - remaining_budget:.2f}")
    return treated
```

---

## Evaluation: Qini Curves and AUUC {#evaluation}

The **Qini curve** plots incremental uplift as you treat increasing fractions of the population (sorted by predicted CATE descending). A good model shows steep initial rise.

```python
def qini_curve(y_true, treatment, uplift_scores):
    """Compute Qini: cumulative uplift vs fraction treated."""
    order = np.argsort(-uplift_scores)
    y_sorted = y_true[order]
    t_sorted = treatment[order]
    n = len(y_true)
    n_t = treatment.sum()
    n_c = n - n_t
    
    fractions, qini_values = [], []
    for k in range(1, n + 1):
        top_treat_success = (y_sorted[:k] * t_sorted[:k]).sum()
        top_ctrl_success = (y_sorted[:k] * (1 - t_sorted[:k])).sum()
        top_n_t = t_sorted[:k].sum()
        top_n_c = k - top_n_t
        
        if top_n_t > 0 and top_n_c > 0:
            qini = (top_treat_success / n_t - top_ctrl_success / n_c) * n
            fractions.append(k / n)
            qini_values.append(qini)
    
    return np.array(fractions), np.array(qini_values)

# AUUC = Area Under Uplift Curve (higher = better targeting)
from sklearn.metrics import auc
auuc = auc(fractions, qini_values)
```

**Always compare against TWO baselines:**
1. **Random targeting** (diagonal line) — is the model better than chance?
2. **Outcome model** (rank by P(Y=1)) — is it truly uplift, not just propensity?

> **Critical Insight:** Many teams accidentally build outcome models thinking they're uplift models. The Qini curve exposes this — an outcome model ranks sure things highest, which is useless for incremental targeting. True uplift models rank persuadables highest.

---

## Connection to Personalization {#connection-to-personalization}

Uplift modeling IS the causal foundation of personalization:

| Personalization Type | Uplift Connection |
|---------------------|-------------------|
| Feature exposure | CATE of showing feature X to user type Y |
| Notification frequency | CATE varies by engagement level |
| Discount amount | CATE varies by price sensitivity |
| Content format | CATE of video vs article by user preference |

**Pipeline:** Run RCT -> Estimate CATE -> Identify positive/negative subgroups -> Build targeting policy -> Deploy with 10% random holdout for ongoing validation.

---

## Practical Applications {#practical-applications}

### Application 1: Coupon Targeting

```python
from causalml.inference.meta import BaseTClassifier

# After RCT with random coupon assignment
learner = BaseTClassifier(
    learner=GradientBoostingClassifier(n_estimators=200, max_depth=4),
    control_name='control'
)
cate = learner.fit_predict(X, treatment_labels, y)

# Target where uplift > coupon cost as fraction of order value
coupon_cost = 5.0
avg_order_value = 35.0
threshold = coupon_cost / avg_order_value  # ~14% lift needed to break even
send_coupon = cate.flatten() > threshold
print(f"Send to {send_coupon.mean()*100:.1f}% of users (was 100%)")
```

### Application 2: Notification Optimization

- **Persuadables:** Dormant users who re-engage when nudged
- **Sleeping Dogs:** Users who uninstall when over-notified (critical to identify!)
- **Sure Things:** Daily active users who engage regardless of notifications

### Application 3: Multi-Treatment Content Recommendation

For multiple treatment options (video / article / infographic), estimate CATE for each vs control, then assign each user their highest-uplift format:

```python
# T=0: article (control), T=1: video, T=2: infographic
# Estimate CATE for each treatment vs control separately
# Assign: argmax(CATE_video, CATE_infographic, 0)
```

---

## Python Implementation {#python-implementation}

### Full Pipeline with causalml and econml

```python
from causalml.inference.meta import BaseTClassifier, BaseSClassifier, BaseXClassifier
from causalml.metrics import auuc_score
from econml.dml import CausalForestDML
from sklearn.model_selection import train_test_split
from sklearn.ensemble import GradientBoostingClassifier

def full_uplift_pipeline(X, treatment, y):
    """End-to-end uplift modeling with model comparison."""
    X_train, X_test, t_train, t_test, y_train, y_test = train_test_split(
        X, treatment, y, test_size=0.3, stratify=treatment, random_state=42)
    
    t_str_train = np.where(t_train == 1, 'treatment', 'control')
    t_str_test = np.where(t_test == 1, 'treatment', 'control')
    
    # Fit multiple uplift models
    models = {
        'T-Learner': BaseTClassifier(learner=GradientBoostingClassifier(n_estimators=200)),
        'S-Learner': BaseSClassifier(learner=GradientBoostingClassifier(n_estimators=200)),
        'X-Learner': BaseXClassifier(
            outcome_learner=GradientBoostingClassifier(n_estimators=200),
            effect_learner=GradientBoostingClassifier(n_estimators=200)),
    }
    
    results = {}
    for name, model in models.items():
        model.fit_predict(X_train, t_str_train, y_train)
        cate_test = model.predict(X_test, t_str_test).flatten()
        score = auuc_score(y_test, cate_test, t_test)
        results[name] = {'auuc': score, 'cate': cate_test}
        print(f"{name} AUUC: {score:.4f}")
    
    # Causal Forest with confidence intervals
    cf = CausalForestDML(n_estimators=500, honest=True, inference=True)
    cf.fit(y_train, T=t_train, X=X_train)
    cate_cf = cf.effect(X_test).flatten()
    ci_lower, ci_upper = cf.effect_interval(X_test, alpha=0.05)
    cf_auuc = auuc_score(y_test, cate_cf, t_test)
    print(f"Causal Forest AUUC: {cf_auuc:.4f}")
    print(f"  Significant positive: {(ci_lower > 0).mean()*100:.1f}%")
    print(f"  Significant negative: {(ci_upper < 0).mean()*100:.1f}%")
    
    return results, cf
```

---

## Interview Talking Points {#interview-talking-points}

> **"We ran a coupon A/B test with +3% conversion. Should we roll out to everyone?"**
>
> "The 3% ATE hides heterogeneity. Sure things convert regardless (wasted coupon), persuadables convert only because of it (our target), sleeping dogs get annoyed (we're hurting them). I'd build an uplift model, target only where CATE exceeds coupon cost / order value. Typically delivers 2-3x ROI vs blanket rollout."

> **"How would you evaluate your uplift model?"**
>
> "Qini curve and AUUC. Compare against random targeting AND an outcome-propensity model baseline. For production validation, keep a 10% holdout where we randomly treat/control within the high-CATE group to verify predicted effects materialize in reality."

> **"T-Learner vs S-Learner vs X-Learner — when each?"**
>
> "T-Learner: large data, strong heterogeneity. S-Learner: subtle effects, limited data. X-Learner: imbalanced groups (90/10). For production, fit all three plus causal forest, select on held-out Qini."

---

## Common Mistakes {#common-mistakes}

| # | Mistake (❌) | Correct Approach (✅) |
|---|-------------|----------------------|
| 1 | ❌ Build outcome model and call it "uplift" | ✅ True uplift = P(Y\|T=1,X) - P(Y\|T=0,X), not P(Y\|X) |
| 2 | ❌ Train on observational data without causal adjustment | ✅ Use randomized experiment data for unbiased CATE |
| 3 | ❌ Evaluate with standard AUC or F1 | ✅ Use Qini curve and AUUC (uplift-specific metrics) |
| 4 | ❌ Target users with highest P(conversion) | ✅ Target by highest CATE — uplift, not propensity |
| 5 | ❌ Ignore sleeping dogs | ✅ Model and exclude users with negative CATE |
| 6 | ❌ Use CATE point estimate for all targeting | ✅ Use CI lower bound for risk-averse decisions |
| 7 | ❌ Ignore treatment cost in targeting | ✅ Only treat when CATE > treatment cost / value |
| 8 | ❌ Deploy without validation holdout | ✅ Keep 10% randomized holdout for ongoing verification |
| 9 | ❌ Assume CATE is static over time | ✅ Re-estimate periodically (effects decay as users change) |
| 10 | ❌ Use same features for outcome and heterogeneity | ✅ Think carefully about what DRIVES variation in effect |

---

## Rapid-Fire Q&A {#rapid-fire-qa}

**Q1: What's the difference between ATE and CATE?**
> ATE = average treatment effect across whole population. CATE = effect conditional on features X — tells you how the effect VARIES across subgroups.

**Q2: What's a "sleeping dog" in uplift modeling?**
> A user who converts WITHOUT treatment but STOPS converting when treated. Treating them has negative uplift. Example: loyal customer annoyed by promotional emails who unsubscribes.

**Q3: Why can't you use standard AUC to evaluate uplift?**
> AUC evaluates outcome prediction (who converts), not uplift (who converts BECAUSE of treatment). Perfect AUC might rank sure things highest — useless for targeting.

**Q4: What data do you need for uplift modeling?**
> Randomized experimental data with features X, random treatment assignment T, and outcomes Y. Randomization ensures causal, not confounded, estimates.

**Q5: How does a causal forest differ from regular random forest?**
> Splits nodes to maximize heterogeneity of treatment effects (not prediction). Uses honest estimation (separate split vs estimate data) for valid confidence intervals.

**Q6: When does X-Learner outperform T-Learner?**
> When groups are very imbalanced (e.g., 95% treated, 5% control). Cross-predictions leverage the larger group's model for estimating effects in the smaller group.

**Q7: How validate uplift in production?**
> Withhold treatment from 10% of "should-treat" users (high CATE). Compare outcomes of treated vs withheld — this directly measures real-world uplift.

**Q8: Connection between uplift modeling and personalization?**
> Personalization IS uplift applied broadly: "what content/feature/experience has highest CATE for THIS specific user?" It's causal recommendation.

**Q9: Can you do uplift with multiple treatments?**
> Yes. Estimate CATE for each treatment vs control, assign each user the treatment with highest positive CATE. Libraries like causalml support multi-treatment.

**Q10: What's doubly-robust CATE estimation?**
> DR-Learner combines outcome model AND propensity model. If EITHER is correctly specified, the CATE estimate is consistent — more robust to misspecification.

---

## ASCII Cheat Sheet {#ascii-cheat-sheet}

```
╔════════════════════════════════════════════════════════════╗
║       UPLIFT MODELING & HTE CHEAT SHEET                    ║
╠════════════════════════════════════════════════════════════╣
║                                                            ║
║  UPLIFT QUADRANT:                                          ║
║  ┌────────────────┬────────────────┐                       ║
║  │  SURE THINGS   │  PERSUADABLES  │                       ║
║  │  Convert anyway │  Convert ONLY  │                       ║
║  │  Don't waste $  │  with treatment│                       ║
║  │                 │  TARGET THESE  │                       ║
║  ├────────────────┼────────────────┤                       ║
║  │  LOST CAUSES   │  SLEEPING DOGS │                       ║
║  │  Won't convert │  STOP converting│                      ║
║  │  regardless    │  when treated   │                      ║
║  │  Don't waste $ │  NEVER treat    │                      ║
║  └────────────────┴────────────────┘                       ║
║                                                            ║
║  CATE FORMULA:                                             ║
║  CATE(x) = E[Y(1)|X=x] - E[Y(0)|X=x]                    ║
║                                                            ║
║  METHOD SELECTION:                                         ║
║  ┌─────────────────┬──────────────────────────────┐        ║
║  │ Large data      │ T-Learner or Causal Forest    │        ║
║  │ Small data      │ S-Learner (shares info)       │        ║
║  │ Imbalanced      │ X-Learner (cross-estimation)  │        ║
║  │ Need CIs        │ Causal Forest (honest trees)  │        ║
║  │ Observational   │ DR-Learner (doubly robust)    │        ║
║  └─────────────────┴──────────────────────────────┘        ║
║                                                            ║
║  TARGETING RULE:                                           ║
║  Treat if: CATE(x) > treatment_cost / value_per_conv      ║
║  Conservative: Treat if CI_lower(CATE) > 0                 ║
║  Budget: Rank by CATE/cost, treat until budget = 0         ║
║                                                            ║
║  EVALUATION:                                               ║
║  - Qini curve: incremental uplift vs % treated             ║
║  - AUUC: Area Under Uplift Curve (higher = better)         ║
║  - Compare vs random AND outcome-model baselines           ║
║  - Production: 10% holdout, always randomized              ║
║                                                            ║
║  PYTHON LIBRARIES:                                         ║
║  - causalml (Uber): BaseTClassifier, BaseXClassifier       ║
║  - econml (Microsoft): CausalForestDML, LinearDML          ║
║  - scikit-uplift: SoloModel, TwoModels                     ║
║                                                            ║
╚════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 58 of 60*
