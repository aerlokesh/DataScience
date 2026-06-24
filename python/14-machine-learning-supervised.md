# 🎯 Topic 14: Supervised Machine Learning

> A comprehensive interview guide covering the core supervised learning algorithms, evaluation metrics, tuning strategies, and decision frameworks that top tech companies (Google, Meta, Amazon, Netflix, Spotify, Uber) expect from a Data Scientist with 5 years of experience.

---

## Table of Contents

1. [Linear Regression](#1-linear-regression)
2. [Logistic Regression](#2-logistic-regression)
3. [Decision Trees](#3-decision-trees)
4. [Random Forest](#4-random-forest)
5. [Gradient Boosting (XGBoost / LightGBM)](#5-gradient-boosting-xgboost--lightgbm)
6. [Support Vector Machines (SVM)](#6-support-vector-machines-svm)
7. [K-Nearest Neighbors (KNN)](#7-k-nearest-neighbors-knn)
8. [Bias-Variance Tradeoff](#8-bias-variance-tradeoff)
9. [Regularization (L1 / L2 / ElasticNet)](#9-regularization-l1--l2--elasticnet)
10. [Evaluation Metrics](#10-evaluation-metrics)
11. [Cross-Validation](#11-cross-validation)
12. [Hyperparameter Tuning](#12-hyperparameter-tuning)
13. [Model Selection Decision Guide](#13-model-selection-decision-guide)
14. [Interview Talking Points & Scripts](#14-interview-talking-points--scripts)
15. [Common Interview Mistakes](#15-common-interview-mistakes)
16. [Rapid-Fire Q&A](#16-rapid-fire-qa)
17. [Summary Cheat Sheet](#17-summary-cheat-sheet)

---

## 1. Linear Regression

**Core Idea:** Model a continuous target as a linear combination of features: `y = Xβ + ε`

**Key Assumptions (LINE):**
- **L**inearity — relationship between X and y is linear
- **I**ndependence — residuals are independent of each other
- **N**ormality — residuals are normally distributed
- **E**qual variance (homoscedasticity) — constant variance of residuals

**Math Intuition:** OLS minimizes the sum of squared residuals. The closed-form solution is `β = (X'X)^{-1} X'y`. In practice, gradient descent is used for large datasets.

> 💡 **Critical Insight:** Interviewers at Google/Meta often ask "what happens when assumptions are violated?" If homoscedasticity fails, your coefficient estimates are still unbiased but standard errors are wrong — use robust standard errors (heteroscedasticity-consistent estimators). If multicollinearity is severe, coefficients become unstable — use VIF > 5 as a flag, then apply regularization or drop features.

```python
import numpy as np
from sklearn.linear_model import LinearRegression
from sklearn.metrics import mean_squared_error, r2_score

# Fit linear regression
model = LinearRegression()
model.fit(X_train, y_train)

y_pred = model.predict(X_test)
print(f"R-squared: {r2_score(y_test, y_pred):.4f}")
print(f"RMSE: {np.sqrt(mean_squared_error(y_test, y_pred)):.4f}")

# Check coefficients for interpretation
for feat, coef in zip(feature_names, model.coef_):
    print(f"  {feat}: {coef:.4f}")
```

---

## 2. Logistic Regression

**Core Idea:** Model the log-odds of a binary outcome as a linear function of features: `log(p / (1 - p)) = Xβ`

**Key Differences from Linear Regression:**
- Output is a probability (0 to 1) via the sigmoid function
- Optimized using maximum likelihood estimation (MLE), not OLS
- Coefficients represent log-odds ratios

**Assumptions:**
- Linear relationship between features and log-odds (not the outcome itself)
- Independence of observations
- No severe multicollinearity
- Large sample size relative to number of features

> 💡 **Critical Insight:** When an interviewer asks "why not just use linear regression for classification?" — the answer is that LR can predict values outside [0,1], violating probability axioms. Logistic regression naturally constrains predictions through the sigmoid, and MLE gives proper probabilistic interpretation. Additionally, the decision boundary of logistic regression is linear in feature space.

```python
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import classification_report, roc_auc_score

model = LogisticRegression(penalty='l2', C=1.0, max_iter=1000)
model.fit(X_train, y_train)

y_pred = model.predict(X_test)
y_prob = model.predict_proba(X_test)[:, 1]

print(classification_report(y_test, y_pred))
print(f"AUC-ROC: {roc_auc_score(y_test, y_prob):.4f}")
```

---

## 3. Decision Trees

**Splitting Criteria:**

| Criterion | Used For | Formula |
|-----------|----------|---------|
| Gini Impurity | Classification | `1 - Σ(p_i²)` |
| Entropy (Info Gain) | Classification | `-Σ(p_i * log2(p_i))` |
| MSE Reduction | Regression | Variance reduction at split |

**Pruning Strategies:**
- **Pre-pruning:** Set max_depth, min_samples_split, min_samples_leaf before training
- **Post-pruning (cost-complexity):** Grow full tree, then prune branches that don't improve cross-validated performance using the `ccp_alpha` parameter

> 💡 **Critical Insight:** Decision trees are greedy — they pick the locally optimal split at each node. This means the globally optimal tree may never be found. This is why ensemble methods (Random Forest, Gradient Boosting) dramatically outperform single trees. In interviews, always mention that a single decision tree is rarely used in production due to high variance.

```python
from sklearn.tree import DecisionTreeClassifier
from sklearn.tree import export_text

dt = DecisionTreeClassifier(max_depth=5, min_samples_leaf=10, random_state=42)
dt.fit(X_train, y_train)

# Visualize tree rules
print(export_text(dt, feature_names=feature_names, max_depth=3))
```

---

## 4. Random Forest

**Core Mechanism: Bagging + Feature Randomness**
1. Bootstrap sampling — each tree trains on a random sample (with replacement) of the data
2. Feature subsampling — at each split, only `sqrt(p)` features are considered (classification) or `p/3` (regression)
3. Aggregation — majority vote (classification) or mean (regression)

**Why It Works:** Each tree is a high-variance, low-bias estimator. Averaging uncorrelated trees reduces variance without increasing bias.

**Feature Importance Methods:**
- **Mean Decrease in Impurity (MDI):** Sum of Gini/entropy reductions weighted by samples reaching node — fast but biased toward high-cardinality features
- **Permutation Importance:** Measure accuracy drop when a feature is shuffled — slower but more reliable

> 💡 **Critical Insight:** At Amazon and Netflix, interviewers probe whether you know that MDI-based feature importance is biased. High-cardinality categorical features (like user_id or zip_code) will appear artificially important. Always use permutation importance or SHAP values for reliable interpretation in production.

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.inspection import permutation_importance

rf = RandomForestClassifier(
    n_estimators=500,
    max_depth=15,
    min_samples_leaf=5,
    max_features='sqrt',
    n_jobs=-1,
    random_state=42
)
rf.fit(X_train, y_train)

# Permutation importance (more reliable than .feature_importances_)
perm_imp = permutation_importance(rf, X_test, y_test, n_repeats=10, random_state=42)
for i in perm_imp.importances_mean.argsort()[::-1][:10]:
    print(f"  {feature_names[i]}: {perm_imp.importances_mean[i]:.4f} +/- {perm_imp.importances_std[i]:.4f}")
```

---

## 5. Gradient Boosting (XGBoost / LightGBM)

**Core Idea:** Build trees sequentially, where each new tree corrects the residual errors of the ensemble so far. The model is an additive ensemble: `F(x) = Σ αₜ hₜ(x)` where each `hₜ` is a weak learner.

**XGBoost vs. LightGBM:**

| Feature | XGBoost | LightGBM |
|---------|---------|----------|
| Tree growth | Level-wise | Leaf-wise (faster) |
| Categorical handling | Requires encoding | Native support |
| Speed on large data | Slower | Faster |
| Memory usage | Higher | Lower |
| Overfitting risk | Moderate | Higher (needs tuning) |

> 💡 **Critical Insight:** In Uber/Spotify DS interviews, they expect you to know the key hyperparameters and their effects: `learning_rate` (lower = more trees needed but better generalization), `max_depth` (controls complexity), `subsample` and `colsample_bytree` (add randomness like RF), and `min_child_weight` (minimum hessian sum in leaf, acts as regularization).

```python
import xgboost as xgb
import lightgbm as lgb

# XGBoost
xgb_model = xgb.XGBClassifier(
    n_estimators=1000,
    learning_rate=0.05,
    max_depth=6,
    subsample=0.8,
    colsample_bytree=0.8,
    min_child_weight=5,
    reg_alpha=0.1,       # L1 regularization
    reg_lambda=1.0,      # L2 regularization
    eval_metric='auc',
    early_stopping_rounds=50,
    random_state=42
)
xgb_model.fit(X_train, y_train, eval_set=[(X_val, y_val)], verbose=False)

# LightGBM
lgb_model = lgb.LGBMClassifier(
    n_estimators=1000,
    learning_rate=0.05,
    max_depth=-1,          # No limit (leaf-wise growth)
    num_leaves=63,
    subsample=0.8,
    colsample_bytree=0.8,
    min_child_samples=20,
    reg_alpha=0.1,
    reg_lambda=1.0,
    random_state=42
)
lgb_model.fit(
    X_train, y_train,
    eval_set=[(X_val, y_val)],
    callbacks=[lgb.early_stopping(50), lgb.log_evaluation(0)]
)
```

---

## 6. Support Vector Machines (SVM)

**Core Idea:** Find the hyperplane that maximizes the margin between classes. For non-linearly separable data, use the kernel trick to implicitly map features to a higher-dimensional space.

**Kernel Types:**
- **Linear:** Best for high-dimensional data (text, genomics) where n_features >> n_samples
- **RBF (Gaussian):** Default choice; captures non-linear boundaries; controlled by `gamma`
- **Polynomial:** Explicit degree of non-linearity

**When to Use SVM:**
- Small-to-medium datasets (< 100K samples) — does not scale well
- High-dimensional feature spaces (text classification with TF-IDF)
- When you need a clear margin of separation

> 💡 **Critical Insight:** SVMs are rarely used in production at big tech for tabular data — gradient boosting dominates. However, they still appear in interviews to test your understanding of margins, the kernel trick (computing dot products in high-dimensional space without explicitly transforming), and the dual formulation. Know that `C` controls the tradeoff between margin width and misclassification penalty.

---

## 7. K-Nearest Neighbors (KNN)

**Core Idea:** Classify a point by majority vote of its k nearest neighbors in feature space.

**The Curse of Dimensionality:** As dimensions increase, all points become equidistant — the concept of "nearest" loses meaning. In 100+ dimensions, KNN performs poorly without dimensionality reduction.

**Practical Considerations:**
- Requires feature scaling (distance-based)
- Computationally expensive at prediction time (lazy learner)
- Choice of k: small k = high variance, large k = high bias
- Distance metric matters: Euclidean, Manhattan, Minkowski

> 💡 **Critical Insight:** KNN is almost never the right answer in a system design interview at big tech. It does not scale for real-time inference, requires storing the entire training set, and suffers in high dimensions. Mention it as a baseline or for recommendation systems (approximate nearest neighbors with libraries like FAISS or Annoy).

---

## 8. Bias-Variance Tradeoff

**The Decomposition:** `Expected Error = Bias² + Variance + Irreducible Noise`

| Property | High Bias | High Variance |
|----------|-----------|---------------|
| Symptom | Underfitting | Overfitting |
| Train error | High | Low |
| Test error | High | High |
| Gap (test - train) | Small | Large |
| Fix | More features, complex model | More data, regularization, ensemble |

**Model Complexity Spectrum:**
- Low complexity (high bias): Linear Regression, Naive Bayes
- Medium: Regularized models, shallow trees
- High complexity (high variance): Deep trees, KNN with k=1, unregularized neural nets

> 💡 **Critical Insight:** The most powerful interview answer to "how do you handle overfitting?" is a structured framework: (1) diagnose with learning curves, (2) add regularization, (3) reduce model complexity, (4) get more data, (5) use ensemble methods, (6) apply dropout/early stopping. Never jump to "get more data" without first confirming the model is actually overfitting.

---

## 9. Regularization (L1 / L2 / ElasticNet)

| Regularization | Penalty | Effect | Best For |
|----------------|---------|--------|----------|
| L1 (Lasso) | `λ Σ|βᵢ|` | Drives coefficients to exactly zero | Feature selection, sparse models |
| L2 (Ridge) | `λ Σβᵢ²` | Shrinks coefficients toward zero | Multicollinearity, all features matter |
| ElasticNet | `α * L1 + (1-α) * L2` | Combines both | Groups of correlated features |

**Geometric Intuition:**
- L1 has diamond-shaped constraint region — solutions hit corners (sparse)
- L2 has circular constraint region — solutions shrink uniformly

```python
from sklearn.linear_model import Lasso, Ridge, ElasticNet
from sklearn.model_selection import cross_val_score

# Compare regularization approaches
for name, model in [
    ("Ridge", Ridge(alpha=1.0)),
    ("Lasso", Lasso(alpha=0.01)),
    ("ElasticNet", ElasticNet(alpha=0.01, l1_ratio=0.5))
]:
    scores = cross_val_score(model, X_train, y_train, cv=5, scoring='r2')
    print(f"{name:12s} — R2: {scores.mean():.4f} (+/- {scores.std():.4f})")
```

---

## 10. Evaluation Metrics

### Classification Metrics

| Metric | Formula | When to Prioritize |
|--------|---------|-------------------|
| Precision | TP / (TP + FP) | Cost of false positives is high (spam filter) |
| Recall | TP / (TP + FN) | Cost of false negatives is high (fraud, disease) |
| F1 Score | 2 * (P * R) / (P + R) | Balance between precision and recall |
| AUC-ROC | Area under TPR vs FPR curve | Ranking ability, threshold-independent |
| AUC-PR | Area under Precision-Recall curve | Imbalanced datasets |

### Confusion Matrix Interpretation

```python
from sklearn.metrics import (
    confusion_matrix, classification_report,
    roc_auc_score, average_precision_score, roc_curve
)
import matplotlib.pyplot as plt

y_prob = model.predict_proba(X_test)[:, 1]
y_pred = model.predict(X_test)

# Full classification report
print(classification_report(y_test, y_pred, digits=4))

# AUC metrics
print(f"AUC-ROC: {roc_auc_score(y_test, y_prob):.4f}")
print(f"AUC-PR:  {average_precision_score(y_test, y_prob):.4f}")

# ROC Curve
fpr, tpr, thresholds = roc_curve(y_test, y_prob)
plt.plot(fpr, tpr, label=f'AUC = {roc_auc_score(y_test, y_prob):.3f}')
plt.plot([0, 1], [0, 1], 'k--')
plt.xlabel('False Positive Rate')
plt.ylabel('True Positive Rate')
plt.title('ROC Curve')
plt.legend()
plt.show()
```

> 💡 **Critical Insight:** At Amazon, the most common follow-up is "how do you choose the threshold?" The answer: it depends on the business cost matrix. If a false negative costs 100x a false positive (e.g., fraud detection), you lower the threshold to maximize recall. Plot precision-recall vs. threshold and pick the operating point that minimizes expected business cost.

---

## 11. Cross-Validation

| Method | When to Use | Key Detail |
|--------|-------------|------------|
| K-Fold (k=5 or 10) | General purpose | Each fold is test set once |
| Stratified K-Fold | Imbalanced classification | Preserves class distribution in each fold |
| Time Series Split | Temporal data | Train on past, test on future — no leakage |
| Group K-Fold | Grouped observations | Ensures all samples from one group stay together |
| Leave-One-Out (LOO) | Very small datasets | High variance, computationally expensive |

```python
from sklearn.model_selection import (
    cross_val_score, StratifiedKFold, TimeSeriesSplit
)

# Stratified K-Fold for imbalanced classification
skf = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
scores = cross_val_score(model, X, y, cv=skf, scoring='roc_auc')
print(f"AUC: {scores.mean():.4f} (+/- {scores.std():.4f})")

# Time Series Split (no data leakage from future)
tscv = TimeSeriesSplit(n_splits=5)
scores = cross_val_score(model, X, y, cv=tscv, scoring='neg_mean_squared_error')
print(f"MSE: {-scores.mean():.4f} (+/- {scores.std():.4f})")
```

> 💡 **Critical Insight:** Time series cross-validation is a common trap in interviews. Standard k-fold shuffles data randomly, causing future information to leak into training. Always use `TimeSeriesSplit` or a rolling-window approach for temporal data. If asked "how would you validate a demand forecasting model?" — this is the expected answer.

---

## 12. Hyperparameter Tuning

| Method | Pros | Cons | Best For |
|--------|------|------|----------|
| GridSearchCV | Exhaustive, reproducible | Exponential cost in # params | Few hyperparameters (< 4) |
| RandomizedSearchCV | Faster, covers more space | May miss optimum | Medium search spaces |
| Bayesian (Optuna) | Smart exploration, efficient | More complex setup | Large search spaces, expensive models |

```python
from sklearn.model_selection import RandomizedSearchCV
import optuna

# Randomized Search
param_dist = {
    'n_estimators': [100, 300, 500, 1000],
    'max_depth': [5, 10, 15, 20, None],
    'min_samples_leaf': [1, 2, 5, 10],
    'max_features': ['sqrt', 'log2', 0.3, 0.5]
}
search = RandomizedSearchCV(
    RandomForestClassifier(random_state=42),
    param_distributions=param_dist,
    n_iter=50, cv=5, scoring='roc_auc', n_jobs=-1, random_state=42
)
search.fit(X_train, y_train)
print(f"Best AUC: {search.best_score_:.4f}")
print(f"Best params: {search.best_params_}")

# Bayesian Optimization with Optuna
def objective(trial):
    params = {
        'n_estimators': trial.suggest_int('n_estimators', 100, 1000),
        'max_depth': trial.suggest_int('max_depth', 3, 20),
        'learning_rate': trial.suggest_float('learning_rate', 0.01, 0.3, log=True),
        'subsample': trial.suggest_float('subsample', 0.6, 1.0),
        'colsample_bytree': trial.suggest_float('colsample_bytree', 0.6, 1.0),
    }
    model = xgb.XGBClassifier(**params, random_state=42, eval_metric='auc')
    scores = cross_val_score(model, X_train, y_train, cv=5, scoring='roc_auc')
    return scores.mean()

study = optuna.create_study(direction='maximize')
study.optimize(objective, n_trials=100, show_progress_bar=True)
print(f"Best AUC: {study.best_value:.4f}")
print(f"Best params: {study.best_params}")
```

---

## 13. Model Selection Decision Guide

| Scenario | Recommended Model | Why |
|----------|------------------|-----|
| Tabular data, medium-large dataset | XGBoost / LightGBM | Best off-the-shelf performance |
| Need interpretability + predictions | Logistic Regression / GAM | Coefficients are directly interpretable |
| Very high-dimensional sparse features | Logistic Regression (L1) / Linear SVM | Efficient, handles sparsity well |
| Small dataset (< 1K samples) | Random Forest / SVM (RBF) | Less prone to overfitting on small data |
| Real-time low-latency inference | Logistic Regression / small tree | Fast prediction time |
| Text classification (TF-IDF) | Linear SVM / Logistic Regression | Works well in high dimensions |
| Imbalanced classification | XGBoost (scale_pos_weight) | Built-in handling + strong performance |
| Baseline / sanity check | Logistic Regression | Simple, fast, hard to mess up |

> 💡 **Critical Insight:** In Meta/Google interviews, saying "I'd use XGBoost" is not enough. They want your reasoning framework: (1) What's the data size and type? (2) Is interpretability required? (3) What's the latency constraint? (4) How imbalanced is the target? (5) Is there a cold-start problem? Show you can think in tradeoffs, not just reach for the most popular tool.

---

## 14. Interview Talking Points & Scripts

### "Walk me through how a Random Forest works"

> *"A Random Forest is an ensemble of decision trees that uses two sources of randomness to create diverse, uncorrelated trees. First, each tree is trained on a bootstrap sample — a random draw with replacement from the training data, typically the same size as the original dataset. This means each tree sees about 63% of the data, and the remaining 37% (out-of-bag samples) can be used for internal validation. Second, at each split, only a random subset of features is considered — typically the square root of total features for classification. This prevents any single dominant feature from appearing at the top of every tree. At prediction time, each tree votes independently, and we take the majority vote for classification or the average for regression. The key insight is that while each individual tree may overfit, averaging many uncorrelated trees dramatically reduces variance without increasing bias. In production, I typically start with 500 trees, tune max_depth and min_samples_leaf, and use permutation importance over the default Gini importance because it is less biased toward high-cardinality features."*

### "How do you handle overfitting?"

> *"I approach overfitting systematically. First, I diagnose it by comparing training and validation performance — a large gap signals overfitting. I then plot learning curves to understand whether more data or more regularization would help. For tree-based models, I reduce max_depth, increase min_samples_leaf, or lower the number of estimators. For linear models, I increase the regularization strength (lower C in logistic regression, higher alpha in Ridge/Lasso). For gradient boosting specifically, I lower the learning rate, add subsampling, and use early stopping based on validation loss. Cross-validation is critical here — I never tune on a single train/test split. If all else fails, I consider whether my features are leaking future information or whether the model is simply too complex for the available data. At Uber, I found that switching from a deep XGBoost model to a shallower one with early stopping improved our production churn model's stability significantly."*

### "When would you choose Logistic Regression over XGBoost?"

> *"I would choose logistic regression in three scenarios. First, when interpretability is a hard requirement — for example, in credit decisions or healthcare where regulators need to understand why a prediction was made. Each coefficient is a log-odds ratio with a clear interpretation. Second, when latency is critical — logistic regression is a single matrix multiplication at inference time, whereas XGBoost requires traversing hundreds of trees. Third, when the dataset is high-dimensional and sparse, like text classification with bag-of-words features — logistic regression with L1 regularization is both fast and effective. XGBoost wins on raw predictive performance for dense tabular data, but the real-world decision involves interpretability, latency, maintainability, and how much tuning effort the team can invest."*

---

## 15. Common Interview Mistakes

| ❌ Mistake | ✅ Better Approach |
|-----------|-------------------|
| "I always use XGBoost" | Explain the tradeoff framework: data size, interpretability, latency |
| Using accuracy on imbalanced data | Report precision, recall, F1, and AUC-PR; discuss threshold selection |
| Forgetting to mention feature scaling | State which models need it (SVM, KNN, regularized linear) vs. which don't (trees) |
| Applying standard k-fold to time series | Use TimeSeriesSplit to prevent future data leakage |
| Tuning on test set | Use train/validation/test split or nested cross-validation |
| Saying "more data always helps" | More data helps high-variance models; for high-bias models, you need better features or a more complex model |
| Ignoring class imbalance | Discuss SMOTE, class weights, threshold tuning, or evaluation metric choice |
| Not mentioning business context for metrics | Always tie the metric choice back to the cost of errors in the specific domain |

---

## 16. Rapid-Fire Q&A

**Q1: What is the difference between bagging and boosting?**
Bagging trains models in parallel on bootstrap samples and averages them (reduces variance). Boosting trains models sequentially, each correcting the errors of the previous (reduces bias and variance).

**Q2: Why does Random Forest reduce variance but not bias?**
Each tree is a full-complexity (low-bias) estimator. Averaging independent estimates reduces variance while preserving the low bias of individual trees.

**Q3: What does the C parameter in SVM control?**
C is the inverse regularization strength. High C = small margin (fits training data tightly, risk of overfitting). Low C = large margin (more misclassifications allowed, better generalization).

**Q4: How do you handle multicollinearity in logistic regression?**
Use Ridge (L2) regularization, drop one of the correlated features, or use PCA/VIF analysis to identify and address it.

**Q5: When is AUC-ROC misleading?**
On heavily imbalanced datasets. A model predicting all negatives can still have a high AUC-ROC. Use AUC-PR (precision-recall curve) instead.

**Q6: What is early stopping in gradient boosting?**
Monitor validation loss after each boosting round. Stop adding trees when validation loss hasn't improved for N rounds. Prevents overfitting without expensive cross-validation.

**Q7: Why is feature scaling not needed for tree-based models?**
Trees split on feature thresholds — the relative ordering of values matters, not their magnitude. Scaling doesn't change the optimal split point.

**Q8: What is the out-of-bag (OOB) error in Random Forest?**
Each tree is trained on ~63% of the data (bootstrap). The remaining 37% is used to estimate generalization error without needing a separate validation set.

**Q9: How does L1 regularization achieve feature selection?**
The L1 penalty creates a diamond-shaped constraint region. The optimal solution tends to lie at corners where some coefficients are exactly zero, effectively selecting features.

**Q10: What is the kernel trick in SVM?**
Computing dot products in a high-dimensional space without explicitly transforming the data. For example, RBF kernel maps to infinite dimensions but only requires pairwise distance computation in the original space.

---

## 17. Summary Cheat Sheet

```
+------------------------------------------------------------------------+
|              SUPERVISED ML — INTERVIEW CHEAT SHEET                      |
+------------------------------------------------------------------------+
|                                                                        |
|  MODEL FAMILIES:                                                       |
|  - Linear: Logistic Reg (interpretable), Ridge/Lasso (regularized)     |
|  - Trees:  Decision Tree (unstable) -> RF (bagging) -> XGB (boosting)  |
|  - Distance: KNN (curse of dim), SVM (kernel trick, small data)        |
|                                                                        |
|  BIAS-VARIANCE:                                                        |
|  - Underfitting (high bias): add features, increase complexity         |
|  - Overfitting (high variance): regularize, simplify, more data        |
|  - Diagnosis: learning curves, train vs. validation gap                |
|                                                                        |
|  REGULARIZATION:                                                       |
|  - L1 (Lasso): sparse solutions, feature selection                     |
|  - L2 (Ridge): shrinkage, handles multicollinearity                    |
|  - ElasticNet: combines both, correlated feature groups                |
|                                                                        |
|  EVALUATION:                                                           |
|  - Balanced classes: Accuracy, F1, AUC-ROC                             |
|  - Imbalanced: AUC-PR, Recall@K, threshold tuning                     |
|  - Always tie metric to business cost of FP vs. FN                     |
|                                                                        |
|  CROSS-VALIDATION:                                                     |
|  - Standard: Stratified K-Fold (k=5)                                   |
|  - Temporal: TimeSeriesSplit (no future leakage!)                       |
|  - Grouped: GroupKFold (e.g., by user_id)                              |
|                                                                        |
|  TUNING:                                                               |
|  - Quick: RandomizedSearchCV (50-100 iterations)                       |
|  - Optimal: Optuna/Bayesian (large search spaces)                      |
|  - Key: early stopping > exhaustive grid search                        |
|                                                                        |
|  INTERVIEW FRAMEWORK:                                                  |
|  1. Understand the problem (classification vs. regression)             |
|  2. Consider constraints (latency, interpretability, data size)        |
|  3. Start simple (logistic reg baseline)                               |
|  4. Iterate (RF -> XGBoost, with proper validation)                    |
|  5. Evaluate with business-relevant metrics                            |
|  6. Discuss tradeoffs, not just "best" model                           |
|                                                                        |
+------------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 14 of 25*
