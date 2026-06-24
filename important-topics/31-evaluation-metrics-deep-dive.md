# 🎯 Topic 31: Evaluation Metrics Deep Dive

> *"The metric you optimize is the behavior you incentivize — choose poorly and your model will optimize for the wrong thing."*

---

## Table of Contents

1. [Classification Metrics](#classification-metrics)
2. [The Confusion Matrix](#the-confusion-matrix)
3. [Threshold-Dependent Metrics](#threshold-dependent-metrics)
4. [Threshold-Independent Metrics](#threshold-independent-metrics)
5. [Regression Metrics](#regression-metrics)
6. [Ranking Metrics](#ranking-metrics)
7. [When to Use Which Metric](#when-to-use-which-metric)
8. [Business-Aligned Metrics](#business-aligned-metrics)
9. [Calibration](#calibration)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Classification Metrics

### The Confusion Matrix

```
                    Predicted
                 Positive  Negative
Actual Positive [   TP    |   FN   ]
Actual Negative [   FP    |   TN   ]
```

```python
from sklearn.metrics import confusion_matrix, classification_report
import seaborn as sns
import matplotlib.pyplot as plt

y_true = [1, 0, 1, 1, 0, 1, 0, 0, 1, 0]
y_pred = [1, 0, 1, 0, 0, 1, 1, 0, 1, 0]

cm = confusion_matrix(y_true, y_pred)
# [[3, 1],   TN=3, FP=1
#  [1, 5]]   FN=1, TP=5 (note: sklearn uses different order)

# Visualize
sns.heatmap(cm, annot=True, fmt='d', cmap='Blues',
            xticklabels=['Neg', 'Pos'], yticklabels=['Neg', 'Pos'])
plt.xlabel('Predicted')
plt.ylabel('Actual')
plt.title('Confusion Matrix')
```

> **Critical Insight:** The confusion matrix is the foundation of ALL classification metrics. Every metric is just a different ratio of TP, FP, TN, FN. Understanding which errors matter more for your business determines which metric to optimize.

---

## Threshold-Dependent Metrics

### Core Metrics

| Metric | Formula | Optimizes For |
|--------|---------|---------------|
| Accuracy | (TP+TN) / (TP+TN+FP+FN) | Overall correctness |
| Precision | TP / (TP+FP) | Minimizing false positives |
| Recall (Sensitivity) | TP / (TP+FN) | Minimizing false negatives |
| Specificity | TN / (TN+FP) | Minimizing false positives (negative class) |
| F1 Score | 2 * (Precision * Recall) / (Precision + Recall) | Balance of precision and recall |
| F-beta | (1+beta^2) * (P*R) / (beta^2*P + R) | Weighted precision-recall balance |

```python
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, 
    f1_score, fbeta_score, classification_report
)

# All threshold-dependent metrics
print(classification_report(y_true, y_pred, target_names=['Negative', 'Positive']))

# F-beta (beta=2 weights recall 2x more than precision)
f2 = fbeta_score(y_true, y_pred, beta=2)  # Recall-heavy
f05 = fbeta_score(y_true, y_pred, beta=0.5)  # Precision-heavy

# Custom threshold selection
from sklearn.metrics import precision_recall_curve

y_proba = model.predict_proba(X_test)[:, 1]
precisions, recalls, thresholds = precision_recall_curve(y_true, y_proba)

# Find threshold for desired recall
target_recall = 0.90
idx = np.argmin(np.abs(recalls - target_recall))
optimal_threshold = thresholds[idx]
print(f"Threshold for {target_recall:.0%} recall: {optimal_threshold:.3f}")
```

### When Accuracy Fails

```python
# Imbalanced dataset example: 99% negative, 1% positive (fraud)
y_true_imbalanced = [0]*990 + [1]*10
y_pred_always_negative = [0]*1000

accuracy = accuracy_score(y_true_imbalanced, y_pred_always_negative)
# accuracy = 0.99 -- looks great but catches ZERO frauds!

recall = recall_score(y_true_imbalanced, y_pred_always_negative)
# recall = 0.0 -- reveals the failure
```

> **Critical Insight:** Accuracy is misleading whenever classes are imbalanced. A model predicting the majority class achieves high accuracy while being useless. For imbalanced problems, use precision, recall, F1, or AUC-PR instead.

---

## Threshold-Independent Metrics

### AUC-ROC (Area Under the ROC Curve)

```python
from sklearn.metrics import roc_auc_score, roc_curve

# ROC curve
fpr, tpr, thresholds = roc_curve(y_true, y_proba)
auc_roc = roc_auc_score(y_true, y_proba)

# Interpretation:
# AUC = 0.5: Random classifier (no discrimination)
# AUC = 0.7-0.8: Acceptable
# AUC = 0.8-0.9: Good
# AUC = 0.9+: Excellent
# AUC = 1.0: Perfect (suspicious -- check for leakage!)

plt.plot(fpr, tpr, label=f'Model (AUC={auc_roc:.3f})')
plt.plot([0, 1], [0, 1], 'k--', label='Random')
plt.xlabel('False Positive Rate')
plt.ylabel('True Positive Rate (Recall)')
plt.title('ROC Curve')
plt.legend()
```

### AUC-PR (Area Under the Precision-Recall Curve)

```python
from sklearn.metrics import average_precision_score, precision_recall_curve

# Precision-Recall curve
precisions, recalls, thresholds = precision_recall_curve(y_true, y_proba)
auc_pr = average_precision_score(y_true, y_proba)

# AUC-PR baseline is the positive class prevalence (not 0.5!)
baseline = y_true.mean()  # e.g., 0.01 for 1% fraud

plt.plot(recalls, precisions, label=f'Model (AP={auc_pr:.3f})')
plt.axhline(y=baseline, color='r', linestyle='--', label=f'Baseline ({baseline:.3f})')
plt.xlabel('Recall')
plt.ylabel('Precision')
plt.title('Precision-Recall Curve')
plt.legend()
```

### AUC-ROC vs AUC-PR

| Aspect | AUC-ROC | AUC-PR |
|--------|---------|--------|
| Baseline | 0.5 (random) | Prevalence of positive class |
| Imbalanced data | Overly optimistic | More informative |
| What it measures | Discrimination across all thresholds | Precision at various recall levels |
| Best for | Balanced classes, general comparison | Imbalanced classes, focus on positives |
| Affected by TN | Yes (via FPR) | No (ignores TN) |

> **Critical Insight:** For imbalanced problems (fraud, rare disease, anomaly detection), AUC-PR is MUCH more informative than AUC-ROC. AUC-ROC can look great (0.95+) even when the model performs poorly on the minority class, because TNs dominate the FPR calculation.

### Log Loss (Cross-Entropy)

```python
from sklearn.metrics import log_loss

# Log loss penalizes confident wrong predictions heavily
ll = log_loss(y_true, y_proba)

# Why it matters: a model that predicts P(fraud)=0.99 for a non-fraud
# gets punished much more than one predicting P(fraud)=0.6
# This incentivizes well-calibrated probabilities
```

---

## Regression Metrics

| Metric | Formula | Properties |
|--------|---------|------------|
| MSE | mean((y - y_hat)^2) | Penalizes large errors, in squared units |
| RMSE | sqrt(MSE) | Same units as target, interpretable |
| MAE | mean(abs(y - y_hat)) | Robust to outliers, same units |
| MAPE | mean(abs((y - y_hat)/y)) * 100 | Percentage, undefined when y=0 |
| R-squared | 1 - SS_res/SS_tot | Proportion of variance explained |
| Adjusted R-squared | Penalizes extra features | Better for model comparison |

```python
from sklearn.metrics import (
    mean_squared_error, mean_absolute_error, r2_score,
    mean_absolute_percentage_error
)
import numpy as np

y_true_reg = np.array([100, 200, 300, 400, 500])
y_pred_reg = np.array([110, 190, 310, 380, 520])

mse = mean_squared_error(y_true_reg, y_pred_reg)
rmse = np.sqrt(mse)  # or mean_squared_error(..., squared=False)
mae = mean_absolute_error(y_true_reg, y_pred_reg)
mape = mean_absolute_percentage_error(y_true_reg, y_pred_reg) * 100
r2 = r2_score(y_true_reg, y_pred_reg)

print(f"MSE:  {mse:.2f}")
print(f"RMSE: {rmse:.2f}")
print(f"MAE:  {mae:.2f}")
print(f"MAPE: {mape:.2f}%")
print(f"R^2:  {r2:.4f}")

# Custom: Symmetric MAPE (handles y=0 cases)
def smape(y_true, y_pred):
    denominator = (np.abs(y_true) + np.abs(y_pred)) / 2
    diff = np.abs(y_true - y_pred) / denominator
    diff[denominator == 0] = 0
    return np.mean(diff) * 100
```

### Choosing Regression Metrics

| Scenario | Recommended Metric | Why |
|----------|-------------------|-----|
| General purpose | RMSE | Penalizes large errors, interpretable units |
| Outlier-robust | MAE | Less sensitive to extreme values |
| Percentage interpretation | MAPE | Business-friendly "we're off by X%" |
| Target has zeros | SMAPE or MAE | MAPE is undefined at y=0 |
| Model comparison | R-squared | Normalized, 0-1 scale |
| Cost proportional to error | MSE | Quadratic penalty matches cost |
| Cost proportional to absolute error | MAE | Linear penalty |

> **Critical Insight:** MSE/RMSE penalizes outliers quadratically — one prediction off by 100 contributes 10,000 to MSE, while ten predictions off by 10 each contribute only 1,000. Choose MSE when large errors are disproportionately costly. Choose MAE when all errors are equally bad per unit.

---

## Ranking Metrics

### NDCG (Normalized Discounted Cumulative Gain)

```python
from sklearn.metrics import ndcg_score
import numpy as np

# Relevance scores (true) and predicted scores
y_true_ranking = np.array([[3, 2, 3, 0, 1, 2]])  # True relevance
y_pred_ranking = np.array([[3.1, 2.8, 1.5, 0.2, 2.1, 1.9]])  # Model scores

# NDCG@k: measures ranking quality at position k
ndcg_5 = ndcg_score(y_true_ranking, y_pred_ranking, k=5)
ndcg_10 = ndcg_score(y_true_ranking, y_pred_ranking, k=10)

# DCG formula: sum(rel_i / log2(i+1)) for i in positions
# NDCG = DCG / ideal_DCG (normalized to [0, 1])
```

### MAP (Mean Average Precision)

```python
def average_precision_at_k(actual, predicted, k):
    """Compute AP@k for a single query."""
    if not actual:
        return 0.0
    
    predicted = predicted[:k]
    score = 0.0
    hits = 0
    
    for i, pred in enumerate(predicted):
        if pred in actual:
            hits += 1
            score += hits / (i + 1)
    
    return score / min(len(actual), k)

def mean_average_precision(actuals, predicteds, k):
    """MAP@k across all queries."""
    return np.mean([
        average_precision_at_k(a, p, k) 
        for a, p in zip(actuals, predicteds)
    ])
```

### MRR (Mean Reciprocal Rank)

```python
def reciprocal_rank(actual, predicted):
    """RR for a single query: 1/rank of first correct answer."""
    for i, pred in enumerate(predicted):
        if pred in actual:
            return 1.0 / (i + 1)
    return 0.0

def mean_reciprocal_rank(actuals, predicteds):
    """MRR across all queries."""
    return np.mean([
        reciprocal_rank(a, p) 
        for a, p in zip(actuals, predicteds)
    ])
```

| Metric | Measures | Use Case |
|--------|----------|----------|
| NDCG@k | Quality of ranked list with graded relevance | Search results, recommendations |
| MAP@k | Average precision across recall levels | Information retrieval |
| MRR | Position of first relevant result | Navigational queries, QA |
| Precision@k | Fraction of top-k that are relevant | "Show me top 10" scenarios |
| Hit Rate@k | Whether any relevant item in top-k | Recommendation systems |

---

## When to Use Which Metric

| Business Problem | Primary Metric | Secondary Metrics | Why |
|-----------------|---------------|-------------------|-----|
| Fraud detection | AUC-PR, Recall@FPR | Precision, F2 | Must catch fraud, FP cost < FN cost |
| Spam filtering | Precision | Recall, F1 | FP (losing good email) is costly |
| Medical screening | Recall (Sensitivity) | Specificity, NPV | Missing disease is dangerous |
| Ad click prediction | Log Loss (calibration) | AUC-ROC | Need calibrated probabilities for bidding |
| Recommendation | NDCG@k, MAP@k | Hit Rate, Coverage | Ranking quality matters |
| Churn prediction | F1, AUC-ROC | Precision@k | Balance: reach all churners vs. wasted outreach |
| Demand forecasting | MAPE, MAE | RMSE, Bias | Business interprets percentage errors |
| House price prediction | RMSE, MAPE | R-squared, MAE | Large errors are costly (pricing decisions) |

> **Critical Insight:** ALWAYS translate ML metrics to business metrics. "AUC improved from 0.82 to 0.87" means nothing to a VP. "We now catch 15% more fraud while keeping the false alarm rate constant, saving $2M/year" gets budget approved.

---

## Business-Aligned Metrics

```python
# Cost-sensitive evaluation
def business_cost(y_true, y_pred, fp_cost=10, fn_cost=100):
    """
    Calculate total business cost of predictions.
    fp_cost: cost of false positive (e.g., wasted investigation)
    fn_cost: cost of false negative (e.g., missed fraud)
    """
    cm = confusion_matrix(y_true, y_pred)
    tn, fp, fn, tp = cm.ravel()
    
    total_cost = fp * fp_cost + fn * fn_cost
    # Could also include: tp * reward, tn * 0
    return total_cost

# Find threshold that minimizes business cost
thresholds = np.linspace(0.01, 0.99, 100)
costs = []
for t in thresholds:
    y_pred_t = (y_proba >= t).astype(int)
    costs.append(business_cost(y_true, y_pred_t, fp_cost=10, fn_cost=500))

optimal_threshold = thresholds[np.argmin(costs)]
print(f"Cost-optimal threshold: {optimal_threshold:.3f}")

# Expected profit calculation
def expected_profit(y_true, y_proba, threshold, 
                    tp_profit=100, fp_cost=-20, fn_cost=-500, tn_profit=0):
    y_pred = (y_proba >= threshold).astype(int)
    cm = confusion_matrix(y_true, y_pred)
    tn, fp, fn, tp = cm.ravel()
    return tp*tp_profit + fp*fp_cost + fn*fn_cost + tn*tn_profit
```

---

## Calibration

### What Is Calibration?

A model is well-calibrated if predicted probability P(Y=1|score=0.7) is actually about 70%.

```python
from sklearn.calibration import calibration_curve, CalibratedClassifierCV

# Plot calibration curve
prob_true, prob_pred = calibration_curve(y_true, y_proba, n_bins=10)

plt.plot(prob_pred, prob_true, marker='o', label='Model')
plt.plot([0, 1], [0, 1], 'k--', label='Perfectly calibrated')
plt.xlabel('Mean Predicted Probability')
plt.ylabel('Fraction of Positives')
plt.title('Calibration Curve (Reliability Diagram)')
plt.legend()

# Brier Score (lower is better, combines calibration + discrimination)
from sklearn.metrics import brier_score_loss
brier = brier_score_loss(y_true, y_proba)

# Calibrate a model (post-hoc)
calibrated_model = CalibratedClassifierCV(model, method='isotonic', cv=5)
calibrated_model.fit(X_train, y_train)
y_proba_calibrated = calibrated_model.predict_proba(X_test)[:, 1]
```

### When Calibration Matters

| Scenario | Why Calibration Needed |
|----------|----------------------|
| Risk scoring | Probabilities feed into decision rules |
| Ensemble combination | Must be on same scale to average |
| Cost-sensitive decisions | Threshold optimization needs true probabilities |
| Customer-facing | "80% chance of delivery tomorrow" must be honest |
| Ad bidding | Bid = P(click) * value; miscalibration = money wasted |

> **Critical Insight:** Many models (especially gradient boosting and SVMs) produce well-ranked but poorly calibrated scores. If you only care about ranking (AUC), calibration doesn't matter. If you need actual probabilities (for thresholding, cost analysis, or communication), calibrate with Platt scaling or isotonic regression.

---

## Interview Talking Points

> "For our fraud model, I chose AUC-PR over AUC-ROC because with 0.1% fraud rate, ROC was misleadingly high (0.98). AUC-PR revealed the true picture — we were at 0.45 AP, with room to improve to 0.62 after better features."

> "I always define the metric BEFORE modeling. In one project, the team optimized for accuracy (96%) on a churn model, but recall was only 40% — we were missing 60% of churners. Switching to F2-score as the objective completely changed our model development direction."

> "When presenting to business stakeholders, I translate everything to dollars. Instead of 'AUC increased by 0.05', I say 'at our operating threshold, we now catch 200 more fraudulent transactions per month, saving approximately $150K annually.'"

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Using accuracy on imbalanced data | Use F1, AUC-PR, or recall for imbalanced problems |
| Reporting AUC-ROC for rare events | Use AUC-PR which is more sensitive to minority class |
| Comparing models at default 0.5 threshold | Compare at the operating threshold or use threshold-free metrics |
| Optimizing log loss when ranking is the goal | Use NDCG or MAP for ranking problems |
| Ignoring calibration when probabilities matter | Calibrate with Platt/isotonic and check reliability diagram |
| Using MAPE when target contains zeros | Use SMAPE or MAE instead |
| Reporting single-number metric without confidence interval | Use bootstrap or cross-val to get error bars |
| Using R-squared alone for regression | Supplement with residual plots and MAE/RMSE |
| Maximizing recall without considering precision | The precision-recall tradeoff is fundamental; optimize F-beta |
| Not connecting metric to business value | Always translate to dollars, time saved, or user impact |

---

## Rapid-Fire Q&A

**Q1: What is the difference between precision and recall?**
A: Precision = "Of those I predicted positive, how many actually are?" (TP/(TP+FP)). Recall = "Of those actually positive, how many did I catch?" (TP/(TP+FN)). They trade off: higher threshold = higher precision, lower recall.

**Q2: When would you optimize F2 over F1?**
A: F2 weights recall twice as much as precision. Use when missing positives is more costly than false alarms — medical screening, fraud detection, safety systems.

**Q3: Why can a model have AUC-ROC of 0.99 but still be useless?**
A: With extreme imbalance (0.01% positive), the ROC is dominated by the huge number of true negatives. The model may have terrible precision on positives while still having high AUC-ROC.

**Q4: What does a well-calibrated model mean?**
A: When the model says "P(positive)=0.8", approximately 80% of those cases should actually be positive. Important when you use the probability directly (not just the ranking).

**Q5: How is NDCG different from MAP?**
A: NDCG handles graded relevance (0-5 scale) and uses logarithmic discount. MAP assumes binary relevance (relevant/not) and gives equal weight to precision at each recall level.

**Q6: Why is log loss preferred for probability estimation?**
A: Log loss directly penalizes the predicted probability — a confident wrong prediction (P=0.99, actual=0) gets massive penalty. It incentivizes calibration, not just ranking.

**Q7: When would MAE be preferred over RMSE?**
A: When outliers exist and shouldn't dominate the metric, or when the cost of errors is linear (not quadratic). MAE is also more interpretable: "on average, we're off by X units."

**Q8: What is the relationship between F1 and the harmonic mean?**
A: F1 IS the harmonic mean of precision and recall. It penalizes extreme imbalance between them — F1 is high only when BOTH precision and recall are reasonably high.

**Q9: How do you choose the optimal classification threshold?**
A: (1) Define cost matrix (FP cost vs FN cost), (2) Plot cost vs threshold, (3) Choose threshold minimizing total cost. Alternatively, use Youden's J = TPR - FPR for the geometric optimal point on ROC.

**Q10: What is R-squared and when can it be negative?**
A: R-squared = 1 - (residual variance / total variance). It can be negative when the model is WORSE than predicting the mean — typically happens on test data when the model overfits training data.

---

## ASCII Cheat Sheet

```
+------------------------------------------------------------------+
|                    METRICS SELECTION GUIDE                         |
+------------------------------------------------------------------+
|                                                                    |
|  CLASSIFICATION:                                                   |
|  +-- Balanced? --> Accuracy, F1, AUC-ROC                           |
|  +-- Imbalanced? --> AUC-PR, F1, Recall/Precision                  |
|  +-- Need probabilities? --> Log Loss, Brier Score                 |
|  +-- FP costly? --> Precision, Specificity                         |
|  +-- FN costly? --> Recall (Sensitivity), F2                       |
|                                                                    |
|  REGRESSION:                                                       |
|  +-- General? --> RMSE                                             |
|  +-- Outliers? --> MAE                                             |
|  +-- Percentage? --> MAPE (if no zeros)                            |
|  +-- Comparison? --> R-squared                                     |
|                                                                    |
|  RANKING:                                                          |
|  +-- Graded relevance? --> NDCG@k                                  |
|  +-- Binary relevance? --> MAP@k                                   |
|  +-- First correct? --> MRR                                        |
|                                                                    |
+------------------------------------------------------------------+
|                    PRECISION-RECALL TRADEOFF                       |
+------------------------------------------------------------------+
|                                                                    |
|  Precision                                                         |
|   1.0 |\.                                                          |
|       |  \.                                                        |
|       |    \.                                                      |
|       |      \.___                                                 |
|       |           \.__                                             |
|       |               \___                                         |
|   0.0 |___________________|                                        |
|       0.0              1.0  Recall                                 |
|                                                                    |
|  High threshold = high precision, low recall (top-right to left)   |
|  Low threshold = high recall, low precision (left to bottom-right) |
|                                                                    |
+------------------------------------------------------------------+
|                    CALIBRATION VISUAL                              |
+------------------------------------------------------------------+
|                                                                    |
|  Fraction                                                          |
|  Positive                                                          |
|   1.0 |              /  <-- Perfectly calibrated                   |
|       |            /                                               |
|       |       ___/   <-- Under-confident (S-shaped below)          |
|       |    __/                                                     |
|       |  /                                                         |
|   0.0 |/_______________                                            |
|       0.0          1.0  Mean Predicted Probability                 |
|                                                                    |
+------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 31 of 45*
