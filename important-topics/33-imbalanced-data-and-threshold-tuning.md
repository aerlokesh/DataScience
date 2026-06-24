# 🎯 Topic 33: Imbalanced Data and Threshold Tuning

> *"When 99% of your data belongs to one class, accuracy becomes a vanity metric — the real challenge is finding the needle in the haystack."*

---

## Table of Contents

1. [Why Accuracy Fails on Imbalanced Data](#why-accuracy-fails-on-imbalanced-data)
2. [The Imbalance Spectrum](#the-imbalance-spectrum)
3. [Resampling Strategies](#resampling-strategies)
4. [SMOTE and Variants](#smote-and-variants)
5. [Class Weights](#class-weights)
6. [Cost-Sensitive Learning](#cost-sensitive-learning)
7. [Threshold Optimization](#threshold-optimization)
8. [Business Cost Matrix](#business-cost-matrix)
9. [Evaluation with Imbalance](#evaluation-with-imbalance)
10. [End-to-End Pipeline for Imbalanced Data](#end-to-end-pipeline-for-imbalanced-data)
11. [Interview Talking Points](#interview-talking-points)
12. [Common Mistakes](#common-mistakes)
13. [Rapid-Fire Q&A](#rapid-fire-qa)
14. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Why Accuracy Fails on Imbalanced Data

```python
import numpy as np
from sklearn.metrics import accuracy_score, classification_report

# Fraud detection: 0.5% fraud rate
n_samples = 10000
y_true = np.array([0]*9950 + [1]*50)  # 50 frauds out of 10,000

# A model that ALWAYS predicts "not fraud"
y_pred_naive = np.zeros(n_samples)

print(f"Accuracy: {accuracy_score(y_true, y_pred_naive):.4f}")  # 0.9950!
print(f"\nBut the model catches ZERO fraud cases!")
print(classification_report(y_true, y_pred_naive, target_names=['legit', 'fraud']))
# Recall for fraud = 0.00 -- completely useless!
```

### The Problem Visualized

```
Dataset: [====================NEGATIVE====================][+]  (99.5% vs 0.5%)

Naive model output: "Everything is negative"
Accuracy: 99.5%  (Looks amazing!)
Recall:   0.0%   (Catches nothing!)
Business impact: All fraud goes undetected, millions lost.
```

> **Critical Insight:** Class imbalance breaks the fundamental assumption of accuracy: that all errors are equally costly. In fraud detection, a false negative (missed fraud) might cost $10,000, while a false positive (flagged legitimate transaction) costs $5 in investigation time. Accuracy treats these as equal — it shouldn't.

---

## The Imbalance Spectrum

| Imbalance Ratio | Example | Difficulty | Approach |
|----------------|---------|------------|----------|
| 2:1 to 5:1 (mild) | Churn prediction | Easy | Class weights, stratified CV |
| 10:1 to 50:1 (moderate) | Medical diagnosis | Medium | SMOTE + class weights + threshold tuning |
| 100:1 to 1000:1 (severe) | Fraud detection | Hard | Undersampling + ensemble + cost-sensitive |
| 10000:1+ (extreme) | Anomaly detection | Very Hard | Anomaly detection algorithms, one-class SVM |

---

## Resampling Strategies

### Undersampling (Reduce Majority)

```python
from imblearn.under_sampling import (
    RandomUnderSampler, TomekLinks, 
    EditedNearestNeighbours, NearMiss
)

# Random undersampling
rus = RandomUnderSampler(sampling_strategy=0.5, random_state=42)
# sampling_strategy=0.5 means minority will be 50% of majority after resampling
X_resampled, y_resampled = rus.fit_resample(X_train, y_train)

# Tomek Links (remove borderline majority samples)
tl = TomekLinks()
X_clean, y_clean = tl.fit_resample(X_train, y_train)

# Edited Nearest Neighbours (remove majority samples misclassified by KNN)
enn = EditedNearestNeighbours(n_neighbors=3)
X_clean, y_clean = enn.fit_resample(X_train, y_train)
```

### Oversampling (Increase Minority)

```python
from imblearn.over_sampling import (
    RandomOverSampler, SMOTE, ADASYN, BorderlineSMOTE
)

# Random oversampling (duplicate minority samples)
ros = RandomOverSampler(sampling_strategy=0.5, random_state=42)
X_resampled, y_resampled = ros.fit_resample(X_train, y_train)

# SMOTE (Synthetic Minority Over-sampling Technique)
smote = SMOTE(sampling_strategy=0.5, k_neighbors=5, random_state=42)
X_resampled, y_resampled = smote.fit_resample(X_train, y_train)
```

### Comparison of Resampling Methods

| Method | Strategy | Pros | Cons |
|--------|----------|------|------|
| Random Undersampling | Remove majority randomly | Simple, fast, reduces train time | Loses information |
| Tomek Links | Remove borderline majority | Cleans decision boundary | Minimal rebalancing |
| Random Oversampling | Duplicate minority | Simple, no information loss | Overfitting to duplicates |
| SMOTE | Synthesize minority | Creates new samples, reduces overfitting | Can create noisy samples |
| ADASYN | Adaptive synthesis | Focuses on hard examples | Can amplify noise |
| Combination | Under + Over | Best of both worlds | More hyperparameters |

> **Critical Insight:** Resampling should ONLY be applied to training data, NEVER to validation or test data. If you resample before splitting, your validation metrics will be meaningless. Always split first, then resample only the training fold.

---

## SMOTE and Variants

### How SMOTE Works

```
1. For each minority sample, find its k nearest minority neighbors
2. Randomly select one neighbor
3. Create synthetic sample along the line between sample and neighbor
4. Synthetic = sample + random(0,1) * (neighbor - sample)
```

```python
from imblearn.over_sampling import SMOTE, BorderlineSMOTE, ADASYN
from imblearn.combine import SMOTETomek, SMOTEENN

# Variants: BorderlineSMOTE (near boundary only), ADASYN (harder examples)
bsmote = BorderlineSMOTE(sampling_strategy='auto', k_neighbors=5, random_state=42)

# Combined approaches (best practice): SMOTE + cleaning
smote_tomek = SMOTETomek(smote=SMOTE(sampling_strategy=0.5, random_state=42))
X_resampled, y_resampled = smote_tomek.fit_resample(X_train, y_train)
```

### SMOTE Limitations

| Limitation | Impact | Mitigation |
|-----------|--------|-----------|
| Assumes local linearity | Synthetic samples may be unrealistic | Use domain-aware generation |
| Noise amplification | Can create noisy borderline samples | Use BorderlineSMOTE or SMOTE+ENN |
| High-dimensional issues | Distance metrics break in high-D | Reduce dimensionality first |
| Categorical features | Can't interpolate between categories | Use SMOTENC for mixed types |
| Not for all models | Some models handle imbalance natively | Trees with class_weight often suffice |

---

## Class Weights

```python
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.linear_model import LogisticRegression
import lightgbm as lgb

# Automatic class weight (inversely proportional to frequency)
# weight_i = n_samples / (n_classes * n_samples_i)
lr = LogisticRegression(class_weight='balanced', max_iter=1000)
lr.fit(X_train, y_train)

# Manual class weight specification
# Fraud is 200x more costly to miss
lr_custom = LogisticRegression(
    class_weight={0: 1, 1: 200},  # 200x penalty for misclassifying fraud
    max_iter=1000
)

# Random Forest with balanced class weights
rf = RandomForestClassifier(
    n_estimators=200, class_weight='balanced_subsample', random_state=42
)

# LightGBM with scale_pos_weight
n_negative = (y_train == 0).sum()
n_positive = (y_train == 1).sum()
lgbm = lgb.LGBMClassifier(
    n_estimators=300,
    scale_pos_weight=n_negative / n_positive,  # Ratio of neg to pos
    random_state=42
)

# XGBoost equivalent
import xgboost as xgb
xgb_model = xgb.XGBClassifier(
    n_estimators=300,
    scale_pos_weight=n_negative / n_positive,
    random_state=42
)
```

> **Critical Insight:** Class weights are often simpler and more effective than resampling, especially for tree-based models. `class_weight='balanced'` in sklearn automatically sets weights inversely proportional to class frequencies. For gradient boosting, `scale_pos_weight` is the go-to parameter. Start here before trying SMOTE.

---

## Cost-Sensitive Learning

### Defining the Cost Matrix

```python
import numpy as np
from sklearn.metrics import confusion_matrix

# Cost matrix: cost_matrix[actual][predicted]
# Fraud example: FP costs $5 (investigation), FN costs $10K (missed fraud)
cost_matrix = np.array([[0, -5], [-10000, 0]])  # [TN, FP], [FN, TP]

def total_cost(y_true, y_pred, cost_matrix):
    """Calculate total business cost from confusion matrix."""
    return np.sum(confusion_matrix(y_true, y_pred) * cost_matrix)

# Sweep thresholds to find cost-optimal point
y_proba = model.predict_proba(X_test)[:, 1]
thresholds = np.linspace(0.01, 0.99, 200)
costs = [total_cost(y_true, (y_proba >= t).astype(int), cost_matrix) for t in thresholds]
optimal_threshold = thresholds[np.argmin(costs)]
```

### Cost-Sensitive vs Standard: Standard ML minimizes error count (all mistakes equal). Cost-sensitive ML minimizes total business cost (weighs mistakes by their consequence). This changes the objective, the threshold, and the evaluation metric.

---

## Threshold Optimization

### Why Default 0.5 Is Usually Wrong

```python
from sklearn.metrics import precision_recall_curve, roc_curve
import matplotlib.pyplot as plt

y_proba = model.predict_proba(X_test)[:, 1]

# Method 1: Precision-Recall based threshold
precisions, recalls, pr_thresholds = precision_recall_curve(y_true, y_proba)

# Find threshold for desired precision
target_precision = 0.80
idx = np.argmin(np.abs(precisions[:-1] - target_precision))
threshold_for_precision = pr_thresholds[idx]
print(f"Threshold for {target_precision:.0%} precision: {threshold_for_precision:.3f}")
print(f"Corresponding recall: {recalls[idx]:.3f}")

# Method 2: F1-optimal threshold
f1_scores = 2 * (precisions[:-1] * recalls[:-1]) / (precisions[:-1] + recalls[:-1] + 1e-8)
optimal_idx = np.argmax(f1_scores)
f1_optimal_threshold = pr_thresholds[optimal_idx]
print(f"F1-optimal threshold: {f1_optimal_threshold:.3f}")
print(f"F1 at optimal: {f1_scores[optimal_idx]:.3f}")

# Method 3: Youden's J statistic (ROC-based)
fpr, tpr, roc_thresholds = roc_curve(y_true, y_proba)
j_scores = tpr - fpr  # Youden's J = Sensitivity + Specificity - 1
optimal_j_idx = np.argmax(j_scores)
youden_threshold = roc_thresholds[optimal_j_idx]
print(f"Youden's J threshold: {youden_threshold:.3f}")

# Method 4: Business cost minimization (see previous section)
```

### Threshold Selection Strategies

| Strategy | Method | Best When |
|----------|--------|-----------|
| F1-optimal | Maximize F1 on PR curve | Balanced precision-recall needed |
| Precision-constrained | Set min precision, maximize recall | FP is costly (spam filtering) |
| Recall-constrained | Set min recall, maximize precision | FN is costly (cancer screening) |
| Youden's J | Maximize TPR - FPR | General discrimination |
| Cost-optimal | Minimize business cost function | Known cost matrix |
| Percentile-based | Top k% predictions as positive | Fixed intervention budget |

```python
# Precision@k: "We can only investigate 100 cases per day"
k = 100
top_k_idx = np.argsort(y_proba)[::-1][:k]
top_k_preds = np.zeros(len(y_true))
top_k_preds[top_k_idx] = 1
print(f"Precision@{k}: {precision_score(y_true, top_k_preds):.3f}")
```

> **Critical Insight:** The optimal threshold depends entirely on your business context, NOT on the model. A fraud model might use threshold=0.15 (catch more fraud, accept more false alarms) while a spam filter might use threshold=0.85 (only flag very obvious spam). ALWAYS tune the threshold on a validation set that reflects your production distribution.

---

## Business Cost Matrix

### Real-World Cost Matrix Examples

```python
# Example 1: Fraud Detection
# FP cost: $5 (analyst reviews transaction, finds it legitimate)
# FN cost: $500 (average fraud amount not caught)
# Cost ratio: FN/FP = 100
fraud_cost_ratio = 500 / 5  # = 100

# Example 2: Medical Screening
# FP cost: $200 (unnecessary follow-up test)
# FN cost: $50,000 (missed early-stage cancer, late treatment)
# Cost ratio: FN/FP = 250
medical_cost_ratio = 50000 / 200  # = 250

# Example 3: Email Spam Filter
# FP cost: $100 (important email lost, missed deal)
# FN cost: $0.10 (user sees one spam email)
# Cost ratio: FP/FN = 1000 (note: reversed! FP is costlier here)
spam_cost_ratio = 100 / 0.10  # = 1000, but precision matters more

# Translating cost ratio to class weights
# As a rule of thumb: class_weight ≈ cost ratio
# For fraud: class_weight={0: 1, 1: 100}
```

### Expected Value Framework

```python
def expected_value_analysis(y_true, y_proba, threshold,
                            tp_value, fp_cost, fn_cost, tn_value=0):
    """Calculate expected value per prediction at a given threshold."""
    y_pred = (y_proba >= threshold).astype(int)
    cm = confusion_matrix(y_true, y_pred)
    tn, fp, fn, tp = cm.ravel()
    total_value = (tp * tp_value) + (tn * tn_value) - (fp * fp_cost) - (fn * fn_cost)
    return {'threshold': threshold, 'total_value': total_value,
            'precision': tp/(tp+fp) if (tp+fp) > 0 else 0,
            'recall': tp/(tp+fn) if (tp+fn) > 0 else 0}

# Sweep thresholds to find cost-optimal point
thresholds = np.linspace(0.05, 0.95, 50)
results = [expected_value_analysis(y_true, y_proba, t, 500, 5, 500) for t in thresholds]
best = max(results, key=lambda x: x['total_value'])
print(f"Optimal threshold: {best['threshold']:.3f}, Value: ${best['total_value']:,.0f}")
```

---

## Evaluation with Imbalance

### Metrics That Work with Imbalanced Data

| Metric | Why It Works | When to Use |
|--------|-------------|-------------|
| AUC-PR | Focuses on positive class, ignores TN | Primary metric for imbalanced |
| F1 (minority class) | Balances precision and recall for minority | When both P and R matter |
| Precision@k | Fixed budget: "top k flagged cases" | Resource-constrained intervention |
| Recall@FPR | "Recall when FPR is at x%" | When you can tolerate fixed FP rate |
| Cohen's Kappa | Accounts for chance agreement | Comparing to random baseline |
| Matthews Correlation Coefficient (MCC) | Balanced measure using all CM cells | Single summary metric |

```python
from sklearn.metrics import (
    average_precision_score, f1_score, matthews_corrcoef,
    cohen_kappa_score, precision_recall_curve
)

# Average Precision (AUC-PR)
ap = average_precision_score(y_true, y_proba)
baseline_ap = y_true.mean()  # Random model AP
print(f"AP: {ap:.4f} (baseline: {baseline_ap:.4f}, lift: {ap/baseline_ap:.1f}x)")

# Matthews Correlation Coefficient (ranges from -1 to +1)
y_pred = (y_proba >= optimal_threshold).astype(int)
mcc = matthews_corrcoef(y_true, y_pred)
print(f"MCC: {mcc:.4f}")  # 0 = random, 1 = perfect

# Cohen's Kappa
kappa = cohen_kappa_score(y_true, y_pred)
print(f"Kappa: {kappa:.4f}")

# Precision-Recall at various recall levels
for target_recall in [0.5, 0.7, 0.8, 0.9, 0.95]:
    precisions, recalls, thresholds = precision_recall_curve(y_true, y_proba)
    idx = np.argmin(np.abs(recalls[:-1] - target_recall))
    print(f"Recall={target_recall:.0%}: Precision={precisions[idx]:.3f}")
```

> **Critical Insight:** For imbalanced data, ALWAYS report AUC-PR (Average Precision) as your primary metric. The random baseline for AUC-PR equals the positive class prevalence (e.g., 0.5% for fraud), so even an AP of 0.3 might represent significant lift. Also report precision-recall at your operating threshold.

---

## End-to-End Pipeline for Imbalanced Data

```python
from imblearn.pipeline import Pipeline as ImbPipeline
from imblearn.over_sampling import SMOTE
from sklearn.preprocessing import StandardScaler
from sklearn.model_selection import StratifiedKFold, cross_validate
import lightgbm as lgb

# Option 1: SMOTE in pipeline (imblearn Pipeline handles resampling correctly)
pipeline_smote = ImbPipeline([
    ('scaler', StandardScaler()),
    ('smote', SMOTE(sampling_strategy=0.3, random_state=42)),
    ('classifier', lgb.LGBMClassifier(n_estimators=200, random_state=42))
])

# Option 2: Class weights (simpler, often equally effective)
pipeline_weighted = ImbPipeline([
    ('scaler', StandardScaler()),
    ('classifier', lgb.LGBMClassifier(n_estimators=200, scale_pos_weight=99))
])

# Evaluate with stratified CV + AP as primary metric
cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
results = cross_validate(pipeline_weighted, X, y, cv=cv,
                         scoring=['average_precision', 'f1', 'roc_auc'])
print(f"AP: {results['test_average_precision'].mean():.4f}")

# Final threshold tuning on validation set (F1-optimal)
pipeline_weighted.fit(X_train, y_train)
y_val_proba = pipeline_weighted.predict_proba(X_val)[:, 1]
precisions, recalls, thresholds = precision_recall_curve(y_val, y_val_proba)
f1s = 2 * precisions * recalls / (precisions + recalls + 1e-8)
best_threshold = thresholds[np.argmax(f1s[:-1])]
```

---

## Interview Talking Points

> "In our fraud detection system with 0.2% fraud rate, I moved from accuracy (99.8% but useless) to Average Precision as the primary metric. Using class weights in LightGBM (scale_pos_weight=500) plus threshold optimization on a validation set, we achieved Precision=0.45 at Recall=0.85 — meaning 85% of fraud caught with only 55% of flagged transactions being false alarms."

> "I prefer class weights over SMOTE for most problems because: (1) they don't alter the data distribution, (2) no risk of generating unrealistic synthetic samples, (3) they work in any sklearn pipeline, and (4) they're simpler to tune and explain."

> "Threshold tuning is where the real business value lives. Our model's AUC was fixed at 0.92, but by optimizing the threshold based on our cost matrix ($500 per missed fraud vs $5 per false alarm), we found the optimal point at threshold=0.12, saving the company $2M annually compared to the default 0.5 threshold."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Reporting accuracy on imbalanced data | Use AUC-PR, F1, or precision@recall |
| Applying SMOTE to full dataset before splitting | Apply SMOTE only inside training folds |
| Using SMOTE on validation/test data | Never resample evaluation data |
| Using default 0.5 threshold | Tune threshold based on business cost matrix |
| Over-resampling to 50:50 ratio | Partial resampling (e.g., 70:30) often works better |
| Ignoring the cost asymmetry | Define FP and FN costs explicitly with stakeholders |
| Using SMOTE with tree models that have class_weight | Class weights alone often suffice for trees |
| Evaluating at a single threshold | Report precision-recall curve and AP |
| Not using stratified splits | Always use StratifiedKFold for imbalanced data |
| Applying techniques without measuring lift over baseline | Compare every approach to a simple class-weighted model |

---

## Rapid-Fire Q&A

**Q1: Why is accuracy misleading for imbalanced data?**
A: Because a model predicting the majority class always achieves accuracy equal to the majority proportion (99% accuracy by predicting everything as non-fraud). It tells you nothing about minority class performance.

**Q2: What is SMOTE and how does it work?**
A: SMOTE creates synthetic minority samples by interpolating between existing minority samples and their k-nearest minority neighbors. It generates new points along the line connecting pairs of minority samples.

**Q3: When would you use undersampling vs oversampling?**
A: Undersampling when you have abundant data (losing majority samples is acceptable). Oversampling (SMOTE) when your dataset is small and you can't afford to lose any samples. Often a combination works best.

**Q4: What are class weights and how do they differ from resampling?**
A: Class weights increase the loss penalty for minority misclassifications during training, without altering the data. Unlike SMOTE, they don't create synthetic samples or change the decision boundary geometry — they just re-weight the optimization objective.

**Q5: How do you choose the optimal classification threshold?**
A: Define the business cost matrix (FP cost, FN cost), then sweep thresholds on a validation set to minimize total cost. Alternatively, use the F1-optimal point or constrain precision/recall to a minimum acceptable level.

**Q6: Why should SMOTE only be applied to training data?**
A: SMOTE changes the data distribution. If applied to validation/test data, your metrics won't reflect real-world performance where the natural imbalance exists. Always evaluate on the original (imbalanced) distribution.

**Q7: What is AUC-PR and why is it better than AUC-ROC for imbalanced data?**
A: AUC-PR measures the area under the Precision-Recall curve. Unlike AUC-ROC, it doesn't use True Negatives (which dominate in imbalanced data). A high AUC-ROC can be misleading; AUC-PR directly reflects minority class performance.

**Q8: What is cost-sensitive learning?**
A: Training the model with explicit costs for different types of errors. Instead of minimizing error count, minimize total business cost. Implemented via class_weight, scale_pos_weight, or custom loss functions.

**Q9: When would you NOT use SMOTE?**
A: When class weights suffice (tree models), when imbalance is extreme (>1000:1, consider anomaly detection), when you have enough minority samples (>1000), or when features are mostly categorical (SMOTE assumes continuous interpolation).

**Q10: What is precision@k and when is it useful?**
A: Precision among the top-k scored instances. Useful when you have a fixed budget for intervention (e.g., "we can manually review 50 transactions per day"). It answers: "of the top 50 flagged cases, how many are actually fraud?"

---

## ASCII Cheat Sheet

```
+------------------------------------------------------------------+
|               IMBALANCED DATA STRATEGY GUIDE                      |
+------------------------------------------------------------------+
|                                                                    |
|  Imbalance Ratio?                                                  |
|  |                                                                 |
|  +-- Mild (2:1 to 10:1)                                           |
|  |   --> class_weight='balanced' + StratifiedKFold                 |
|  |                                                                 |
|  +-- Moderate (10:1 to 100:1)                                     |
|  |   --> class_weight + threshold tuning                           |
|  |   --> SMOTE if <5K minority samples                             |
|  |                                                                 |
|  +-- Severe (100:1 to 1000:1)                                     |
|  |   --> Ensemble undersampling (BalancedRandomForest)             |
|  |   --> Cost-sensitive learning                                   |
|  |   --> Focus on AUC-PR, Precision@k                              |
|  |                                                                 |
|  +-- Extreme (>1000:1)                                             |
|      --> Anomaly detection (Isolation Forest, One-Class SVM)       |
|      --> Treat as outlier detection problem                        |
|                                                                    |
+------------------------------------------------------------------+
|               THRESHOLD TUNING DECISION                           |
+------------------------------------------------------------------+
|                                                                    |
|  Business constraint?                                              |
|  |                                                                 |
|  +-- "Catch at least 90% of positives" --> Set min recall=0.90    |
|  |   Find threshold that gives recall>=0.90, report precision      |
|  |                                                                 |
|  +-- "At least 80% of flags should be real" --> Set precision=0.80|
|  |   Find threshold that gives precision>=0.80, report recall      |
|  |                                                                 |
|  +-- "Minimize total cost" --> Define cost matrix                  |
|  |   Sweep thresholds, pick min-cost point                         |
|  |                                                                 |
|  +-- "Fixed budget: review 100/day" --> Use top-100 predictions   |
|      Report Precision@100 and Recall@100                           |
|                                                                    |
+------------------------------------------------------------------+
|               METRICS FOR IMBALANCED DATA                         |
+------------------------------------------------------------------+
|                                                                    |
|  PRIMARY:   Average Precision (AUC-PR)                             |
|  SECONDARY: F1 (minority class), MCC                               |
|  OPERATING: Precision@recall_target, Cost at threshold             |
|                                                                    |
|  AVOID:     Accuracy (misleading)                                  |
|  CAUTION:   AUC-ROC (can be overly optimistic)                     |
|                                                                    |
|  Always report: Baseline AP = positive class prevalence            |
|  Your model AP should be >> baseline to be useful                  |
|                                                                    |
+------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 33 of 45*
