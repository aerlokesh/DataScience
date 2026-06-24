# 🎯 Topic 13: Feature Engineering & Preprocessing

> A comprehensive guide to transforming raw data into model-ready features — covering encoding strategies, scaling methods, imputation techniques, feature creation, selection algorithms, imbalanced data handling, and production-ready sklearn Pipelines. Tailored for DS/DA candidates with 5 YOE targeting Google, Meta, Amazon, Netflix, Spotify, and Uber.

---

## Table of Contents

1. [Encoding Categorical Variables](#1-encoding-categorical-variables)
2. [Feature Scaling & Normalization](#2-feature-scaling--normalization)
3. [Missing Value Imputation](#3-missing-value-imputation)
4. [Feature Creation](#4-feature-creation)
5. [Feature Selection](#5-feature-selection)
6. [Handling Imbalanced Data](#6-handling-imbalanced-data)
7. [sklearn Pipelines for Production](#7-sklearn-pipelines-for-production)
8. [Interview Talking Points & Scripts](#8-interview-talking-points--scripts)
9. [Common Interview Mistakes](#9-common-interview-mistakes)
10. [Rapid-Fire Q&A](#10-rapid-fire-qa)
11. [Summary Cheat Sheet](#11-summary-cheat-sheet)

---

## 1. Encoding Categorical Variables

### Comparison Table

| Method | When to Use | Pros | Cons | Example Use Case |
|--------|-------------|------|------|-----------------|
| **One-Hot** | Nominal, low cardinality (<15 categories) | No ordinal assumption; works with all models | Dimensionality explosion; sparse matrix | Country (5-10 values) |
| **Label Encoding** | Ordinal features; tree-based models | Compact; preserves order | Implies false ordering for nominal vars | Education level (HS < BS < MS < PhD) |
| **Target Encoding** | High cardinality nominal; Kaggle competitions | Captures target relationship; single column | Prone to overfitting; requires regularization | ZIP code, product_id |
| **Ordinal Encoding** | Natural order exists | Preserves meaningful order; single column | Assumes equal spacing between levels | Satisfaction (low/medium/high) |

> 💡 **Critical Insight:** At companies like Uber and Spotify, high-cardinality features (city_id, artist_id) are ubiquitous. Target encoding with smoothing or leave-one-out is preferred over one-hot encoding which would create thousands of sparse columns.

```python
import pandas as pd
from sklearn.preprocessing import OneHotEncoder, OrdinalEncoder, LabelEncoder
from category_encoders import TargetEncoder

# One-Hot Encoding
ohe = OneHotEncoder(drop='first', sparse_output=False, handle_unknown='ignore')
X_encoded = ohe.fit_transform(X[['city']])

# Ordinal Encoding (with explicit order)
ordinal = OrdinalEncoder(categories=[['low', 'medium', 'high', 'very_high']])
X['satisfaction_encoded'] = ordinal.fit_transform(X[['satisfaction']])

# Target Encoding with smoothing (prevents overfitting)
target_enc = TargetEncoder(smoothing=10, min_samples_leaf=20)
X['zip_encoded'] = target_enc.fit_transform(X['zip_code'], y)

# Label Encoding (for tree models only)
le = LabelEncoder()
X['category_label'] = le.fit_transform(X['category'])
```

---

## 2. Feature Scaling & Normalization

### When-to-Use Guide

| Scaler | Formula | Best For | Sensitive To | Use When |
|--------|---------|----------|--------------|----------|
| **StandardScaler** | (x - mean) / std | Linear models, SVMs, Neural Nets | Outliers | Data is approximately Gaussian |
| **MinMaxScaler** | (x - min) / (max - min) | Neural networks, image data | Outliers | You need bounded [0,1] range |
| **RobustScaler** | (x - median) / IQR | Any model with outlier-heavy data | Nothing (robust by design) | Outliers present, cannot remove them |

> 💡 **Critical Insight:** Tree-based models (XGBoost, Random Forest, LightGBM) do NOT need scaling — they split on rank order. Scaling is critical for distance-based methods (KNN, SVM, K-Means) and gradient-based optimization (logistic regression, neural nets).

```python
from sklearn.preprocessing import StandardScaler, MinMaxScaler, RobustScaler
import numpy as np

# StandardScaler — zero mean, unit variance
scaler_std = StandardScaler()
X_scaled = scaler_std.fit_transform(X[['income', 'age']])

# MinMaxScaler — bounded to [0, 1]
scaler_mm = MinMaxScaler(feature_range=(0, 1))
X_minmax = scaler_mm.fit_transform(X[['pixel_intensity']])

# RobustScaler — uses median and IQR (handles outliers gracefully)
scaler_robust = RobustScaler()
X_robust = scaler_robust.fit_transform(X[['transaction_amount']])

# When to apply log transform BEFORE scaling
X['log_income'] = np.log1p(X['income'])  # log1p handles zeros
```

---

## 3. Missing Value Imputation

### Strategy Comparison

| Method | Assumption | Pros | Cons | Best For |
|--------|-----------|------|------|----------|
| **Mean/Median** | MCAR (Missing Completely at Random) | Fast, simple | Distorts variance; ignores relationships | Quick baseline; numeric features |
| **Mode** | MCAR | Works for categorical | Can bias toward majority class | Categorical features |
| **KNN Imputer** | MAR (Missing at Random); local structure | Captures local patterns | Slow on large datasets; sensitive to k | When features are correlated |
| **Iterative (MICE)** | MAR | Models feature interactions | Computationally expensive; complex | When missingness has structure |
| **Indicator + Impute** | Missingness is informative | Preserves missingness signal | Adds extra columns | When "missing" itself is a feature |

> 💡 **Critical Insight:** At Netflix and Spotify, missing data often IS the signal. A user with no rating history or no playlist activity is fundamentally different. Always consider adding a binary `is_missing` indicator column before imputing.

```python
from sklearn.impute import SimpleImputer, KNNImputer
from sklearn.experimental import enable_iterative_imputer
from sklearn.impute import IterativeImputer
import numpy as np

# Simple imputation
median_imputer = SimpleImputer(strategy='median')
X['age'] = median_imputer.fit_transform(X[['age']])

# KNN Imputation (leverages feature correlations)
knn_imputer = KNNImputer(n_neighbors=5, weights='distance')
X_imputed = knn_imputer.fit_transform(X[['age', 'income', 'credit_score']])

# Iterative Imputer (MICE — Multiple Imputation by Chained Equations)
mice_imputer = IterativeImputer(max_iter=10, random_state=42)
X_mice = mice_imputer.fit_transform(X)

# Missingness indicator pattern
X['income_missing'] = X['income'].isna().astype(int)
X['income'] = median_imputer.fit_transform(X[['income']])
```

---

## 4. Feature Creation

### Interaction & Polynomial Features

```python
from sklearn.preprocessing import PolynomialFeatures
import pandas as pd

# Manual interaction features (domain-driven)
X['price_per_sqft'] = X['price'] / X['sqft']
X['room_density'] = X['rooms'] / X['sqft']
X['income_to_debt_ratio'] = X['income'] / (X['debt'] + 1)

# Polynomial features (automated, use sparingly)
poly = PolynomialFeatures(degree=2, interaction_only=True, include_bias=False)
X_poly = poly.fit_transform(X[['age', 'income', 'tenure']])
```

### Date/Time Feature Extraction

```python
# Datetime feature engineering (critical at Uber, Spotify, Netflix)
X['hour'] = X['timestamp'].dt.hour
X['day_of_week'] = X['timestamp'].dt.dayofweek
X['is_weekend'] = X['day_of_week'].isin([5, 6]).astype(int)
X['month'] = X['timestamp'].dt.month
X['quarter'] = X['timestamp'].dt.quarter
X['days_since_signup'] = (X['event_date'] - X['signup_date']).dt.days

# Cyclical encoding for periodic features (avoids discontinuity at midnight)
import numpy as np
X['hour_sin'] = np.sin(2 * np.pi * X['hour'] / 24)
X['hour_cos'] = np.cos(2 * np.pi * X['hour'] / 24)
X['dow_sin'] = np.sin(2 * np.pi * X['day_of_week'] / 7)
X['dow_cos'] = np.cos(2 * np.pi * X['day_of_week'] / 7)
```

> 💡 **Critical Insight:** At Uber, time features are everything — demand prediction depends on hour, day of week, holidays, and local events. Cyclical encoding (sin/cos) prevents the model from thinking 23:00 and 00:00 are far apart.

### Aggregation & Window Features

```python
# User-level aggregation features (common at Meta, Spotify)
user_agg = df.groupby('user_id').agg(
    total_sessions=('session_id', 'count'),
    avg_session_duration=('duration', 'mean'),
    max_purchase=('purchase_amount', 'max'),
    days_active=('date', 'nunique'),
    recency=('date', lambda x: (pd.Timestamp.now() - x.max()).days)
).reset_index()

# Rolling window features (time series)
df['rolling_7d_avg'] = df.groupby('user_id')['metric'].transform(
    lambda x: x.rolling(7, min_periods=1).mean()
)
```

---

## 5. Feature Selection

### Methods Comparison

| Method | Type | Pros | Cons | Complexity |
|--------|------|------|------|-----------|
| **Correlation Filter** | Filter | Fast; interpretable | Only captures linear relationships | O(n^2) |
| **Mutual Information** | Filter | Captures nonlinear; model-agnostic | Slow for continuous; binning required | O(n*m) |
| **RFE** | Wrapper | Model-aware; considers interactions | Expensive; depends on model choice | O(n*k*model) |
| **L1 Regularization** | Embedded | Built into training; automatic | Only for linear/logistic models | O(model) |
| **Tree Importance** | Embedded | Fast; captures nonlinear | Biased toward high-cardinality features | O(model) |

```python
from sklearn.feature_selection import (
    mutual_info_classif, SelectKBest, RFE, SelectFromModel
)
from sklearn.linear_model import LassoCV, LogisticRegression
from sklearn.ensemble import RandomForestClassifier
import numpy as np

# 1. Correlation-based filtering
corr_matrix = X.corr().abs()
upper_tri = corr_matrix.where(np.triu(np.ones(corr_matrix.shape), k=1).astype(bool))
to_drop = [col for col in upper_tri.columns if any(upper_tri[col] > 0.90)]
X_filtered = X.drop(columns=to_drop)

# 2. Mutual Information (captures nonlinear relationships)
mi_scores = mutual_info_classif(X, y, random_state=42)
mi_ranking = pd.Series(mi_scores, index=X.columns).sort_values(ascending=False)
top_features = mi_ranking.head(20).index.tolist()

# 3. Recursive Feature Elimination (RFE)
estimator = RandomForestClassifier(n_estimators=100, random_state=42)
rfe = RFE(estimator, n_features_to_select=15, step=5)
rfe.fit(X, y)
selected_rfe = X.columns[rfe.support_].tolist()

# 4. L1 Regularization (Lasso) — automatic feature selection
lasso = LassoCV(cv=5, random_state=42)
lasso.fit(X, y)
important_features = X.columns[lasso.coef_ != 0].tolist()

# 5. Tree-based importance with threshold
rf = RandomForestClassifier(n_estimators=200, random_state=42)
rf.fit(X, y)
selector = SelectFromModel(rf, threshold='median')
X_selected = selector.transform(X)
```

> 💡 **Critical Insight:** In interviews at Google and Meta, interviewers want you to articulate WHY you chose a feature selection method. Correlation is a sanity check; mutual information is for exploring nonlinear signals; RFE is for final model tuning; L1 is elegant when you need interpretability.

---

## 6. Handling Imbalanced Data

### Strategy Comparison

| Technique | Level | When to Use | Risk |
|-----------|-------|-------------|------|
| **Class Weights** | Algorithm | First try; no data generation | May underfit majority class |
| **SMOTE** | Data | Moderate imbalance (1:10) | Can create noisy synthetic examples |
| **SMOTE + Tomek** | Data | Need cleaner decision boundary | Slower; removes borderline samples |
| **Random Undersampling** | Data | Very large dataset; speed needed | Loses information |
| **Threshold Tuning** | Post-hoc | When you control decision threshold | Requires calibrated probabilities |

```python
from sklearn.utils.class_weight import compute_class_weight
from imblearn.over_sampling import SMOTE, ADASYN
from imblearn.combine import SMOTETomek
from imblearn.pipeline import Pipeline as ImbPipeline
import numpy as np

# Method 1: Class weights (built into most sklearn estimators)
weights = compute_class_weight('balanced', classes=np.unique(y), y=y)
model = LogisticRegression(class_weight='balanced')  # or pass dict

# Method 2: SMOTE (only on training data!)
smote = SMOTE(sampling_strategy=0.5, k_neighbors=5, random_state=42)
X_resampled, y_resampled = smote.fit_resample(X_train, y_train)

# Method 3: SMOTE + Tomek Links (cleaner boundaries)
smt = SMOTETomek(random_state=42)
X_clean, y_clean = smt.fit_resample(X_train, y_train)

# Method 4: Integrate into pipeline (prevents leakage)
imb_pipeline = ImbPipeline([
    ('scaler', StandardScaler()),
    ('smote', SMOTE(random_state=42)),
    ('classifier', LogisticRegression(max_iter=1000))
])
imb_pipeline.fit(X_train, y_train)

# Method 5: Threshold tuning post-training
from sklearn.metrics import precision_recall_curve
y_probs = model.predict_proba(X_test)[:, 1]
precisions, recalls, thresholds = precision_recall_curve(y_test, y_probs)
# Choose threshold that optimizes your business metric (e.g., F2 for fraud)
```

> 💡 **Critical Insight:** SMOTE must ONLY be applied to training data, NEVER to validation/test. At Amazon (fraud detection) and Meta (content moderation), class weights are preferred in production because they avoid creating synthetic data that may not represent real patterns.

---

## 7. sklearn Pipelines for Production

### Why Pipelines Matter

Pipelines prevent **data leakage** by ensuring transformations are fit only on training data, and they make deployment trivial by serializing the entire preprocessing + model chain.

```python
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.impute import SimpleImputer
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.model_selection import cross_val_score
import joblib

# Define column groups
numeric_features = ['age', 'income', 'tenure', 'transaction_count']
categorical_features = ['city', 'device_type', 'subscription_tier']

# Numeric preprocessing pipeline
numeric_pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='median')),
    ('scaler', StandardScaler())
])

# Categorical preprocessing pipeline
categorical_pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='most_frequent')),
    ('encoder', OneHotEncoder(drop='first', handle_unknown='ignore'))
])

# Combine with ColumnTransformer
preprocessor = ColumnTransformer([
    ('num', numeric_pipeline, numeric_features),
    ('cat', categorical_pipeline, categorical_features)
], remainder='drop')

# Full pipeline: preprocessing + model
full_pipeline = Pipeline([
    ('preprocessor', preprocessor),
    ('classifier', GradientBoostingClassifier(
        n_estimators=200, learning_rate=0.1, max_depth=5, random_state=42
    ))
])

# Train and evaluate (no leakage — CV handles fit/transform correctly)
scores = cross_val_score(full_pipeline, X, y, cv=5, scoring='roc_auc')
print(f"AUC: {scores.mean():.4f} +/- {scores.std():.4f}")

# Fit final model and serialize for production
full_pipeline.fit(X_train, y_train)
joblib.dump(full_pipeline, 'model_pipeline.pkl')

# In production: load and predict
pipeline_loaded = joblib.load('model_pipeline.pkl')
predictions = pipeline_loaded.predict(new_data)
```

> 💡 **Critical Insight:** In big tech interviews, saying "I'd use a Pipeline" demonstrates production maturity. It shows you understand that preprocessing parameters (mean, std, category mappings) are learned artifacts that must be versioned alongside the model.

---

## 8. Interview Talking Points & Scripts

### "How would you handle missing data?"

> *"My approach to missing data is systematic. First, I quantify the missingness — what percentage, and is it MCAR, MAR, or MNAR? For MCAR at low rates (under 5%), median imputation is fine. For MAR, I use KNN or iterative imputation that leverages correlated features. For MNAR — like users who didn't report income — the missingness itself is informative, so I add a binary indicator column before imputing. In production, I always implement this inside a Pipeline to prevent train-test leakage. At scale, I've found that simply adding a missing indicator often gives 80% of the benefit of complex imputation, and it's far more robust in production."*

### "How do you avoid data leakage?"

> *"Data leakage is the number one silent killer of ML projects. I guard against it at three levels. First, temporal: I never use future information to predict the past — my train-test splits respect time ordering. Second, preprocessing: all transformations — scaling, imputation, encoding — are fit exclusively on training data using sklearn Pipelines. Third, feature engineering: I audit every feature to ensure it wouldn't be available at prediction time. For example, in a churn model, using 'cancellation_reason' leaks the outcome. I've caught subtle leakage by checking that cross-validation scores aren't suspiciously high — if AUC is 0.99 on a real-world problem, something is leaking."*

### "Walk me through your feature engineering process"

> *"I follow a structured approach. I start with domain understanding — what signals would a human expert use? Then I build baseline features: raw numerics scaled appropriately, categoricals encoded based on cardinality. Next, I create interaction features grounded in domain logic — for an e-commerce model, price-per-unit matters more than price alone. I extract temporal features with cyclical encoding for periodic patterns. Finally, I use mutual information and feature importance to prune — I aim for the smallest feature set that retains 95% of model performance. This discipline keeps models interpretable and fast in production."*

---

## 9. Common Interview Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| One-hot encoding high-cardinality features (1000+ categories) | Use target encoding with smoothing or embeddings |
| Scaling features before train-test split | Scale INSIDE a Pipeline or after splitting |
| Imputing test data with test statistics | Fit imputer on train, transform test |
| Applying SMOTE to entire dataset | Apply SMOTE only to training fold |
| Using correlation alone for feature selection | Combine with mutual information for nonlinear |
| Dropping all rows with missing values | Analyze missingness pattern; impute or add indicator |
| Creating features from the target variable | Audit every feature for temporal/causal leakage |
| Ignoring feature distributions before scaling | Visualize distributions; log-transform skewed features first |
| Using label encoding for nominal features in linear models | Use one-hot for nominal; label only for ordinal or trees |
| Building features without domain context | Start with domain knowledge; automate second |

---

## 10. Rapid-Fire Q&A

**Q1: When would you NOT scale features?**
Tree-based models (XGBoost, Random Forest, LightGBM) are invariant to monotonic transformations. Scaling adds no value and can hurt interpretability.

**Q2: What's the difference between StandardScaler and Normalizer?**
StandardScaler operates column-wise (features). Normalizer operates row-wise (samples) — it scales each sample to unit norm. They solve fundamentally different problems.

**Q3: How do you encode a feature with 50,000 unique values?**
Target encoding with regularization, frequency encoding, or learned embeddings (if using a neural network). Never one-hot encode.

**Q4: Can you use target encoding without leakage?**
Yes — use leave-one-out encoding or k-fold target encoding where each fold's encoding is computed from out-of-fold data.

**Q5: When is mean imputation dangerous?**
When data is not MCAR — it biases estimates, reduces variance, and can mask informative missingness patterns. Also problematic when missingness exceeds 20-30%.

**Q6: What's the difference between RFE and L1 regularization for selection?**
RFE is model-agnostic and wrapper-based — it retrains iteratively. L1 is embedded in training and only works with linear models. RFE is more flexible; L1 is more efficient.

**Q7: Why might SMOTE hurt model performance?**
SMOTE generates synthetic points by interpolating between neighbors. In high-dimensional sparse data, this can create unrealistic samples in empty regions of feature space, introducing noise.

**Q8: How do you handle a feature that's missing 60% of values?**
Three options: (1) If missingness is informative, keep as binary indicator only. (2) If correlated features exist, use iterative/KNN imputation. (3) If neither, consider dropping — but test both approaches.

**Q9: What's the leakage risk with target encoding?**
The target mean for a category is computed using the label — if done naively on the full dataset, the encoding contains target information that won't generalize. Always compute on training folds only.

**Q10: Why use ColumnTransformer instead of applying transformations manually?**
It guarantees consistent column ordering, handles train/test alignment automatically, serializes with the model, and prevents accidental train-test contamination.

---

## 11. Summary Cheat Sheet

```
+===========================================================================+
|                  FEATURE ENGINEERING CHEAT SHEET                           |
+===========================================================================+
|                                                                           |
|  ENCODING:                                                                |
|    Low cardinality + nominal  --> One-Hot (drop='first')                  |
|    Ordinal                    --> OrdinalEncoder with explicit order       |
|    High cardinality           --> Target Encoding (with smoothing)        |
|    Tree models only           --> Label Encoding is acceptable            |
|                                                                           |
|  SCALING:                                                                 |
|    Gaussian-ish data          --> StandardScaler                          |
|    Bounded range needed       --> MinMaxScaler                            |
|    Outliers present           --> RobustScaler                            |
|    Trees/boosting             --> No scaling needed                       |
|                                                                           |
|  MISSING VALUES:                                                          |
|    MCAR, <5% missing          --> Median/Mode imputation                 |
|    MAR, correlated features   --> KNN or Iterative (MICE)                |
|    MNAR, informative          --> Binary indicator + simple impute        |
|    High missingness (>50%)    --> Indicator only or drop                  |
|                                                                           |
|  FEATURE CREATION:                                                        |
|    Domain ratios              --> price/sqft, income/debt                 |
|    Time features              --> hour, dow, month (cyclical sin/cos)     |
|    Aggregations               --> user-level count, mean, recency         |
|    Interactions               --> PolynomialFeatures(interaction_only)    |
|                                                                           |
|  FEATURE SELECTION:                                                       |
|    Quick filter               --> Correlation (>0.9 drop one)            |
|    Nonlinear signal           --> Mutual Information                      |
|    Model-aware                --> RFE or tree importance                  |
|    Automatic sparsity         --> L1 (Lasso)                             |
|                                                                           |
|  IMBALANCED DATA:                                                         |
|    First try                  --> class_weight='balanced'                 |
|    Moderate imbalance         --> SMOTE (train only!)                     |
|    Post-hoc calibration       --> Threshold tuning on PR curve           |
|                                                                           |
|  PRODUCTION:                                                              |
|    Always                     --> sklearn Pipeline + ColumnTransformer    |
|    Serialize                  --> joblib.dump(pipeline)                   |
|    Deploy                     --> Single .pkl contains everything         |
|                                                                           |
|  DATA LEAKAGE PREVENTION:                                                 |
|    1. Fit transformers on TRAIN only                                      |
|    2. Respect temporal ordering                                           |
|    3. Audit features for future information                               |
|    4. If CV AUC > 0.99, suspect leakage                                  |
|                                                                           |
+===========================================================================+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 13 of 25*
