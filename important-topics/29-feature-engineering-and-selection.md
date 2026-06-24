# 🎯 Topic 29: Feature Engineering and Selection

> *"The art of transforming raw data into predictive signals — where domain knowledge meets algorithmic rigor."*

---

## Table of Contents

1. [Why Feature Engineering Matters](#why-feature-engineering-matters)
2. [Encoding Strategies](#encoding-strategies)
3. [Numerical Transforms](#numerical-transforms)
4. [Interaction and Polynomial Features](#interaction-and-polynomial-features)
5. [Time-Based Features](#time-based-features)
6. [Text Features](#text-features)
7. [Feature Selection Methods](#feature-selection-methods)
8. [Feature Importance vs Permutation Importance](#feature-importance-vs-permutation-importance)
9. [Data Leakage Prevention](#data-leakage-prevention)
10. [Sklearn Pipelines](#sklearn-pipelines)
11. [Interview Talking Points](#interview-talking-points)
12. [Common Mistakes](#common-mistakes)
13. [Rapid-Fire Q&A](#rapid-fire-qa)
14. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Why Feature Engineering Matters

Feature engineering is often the single most impactful step in a machine learning pipeline. Models are only as good as the features they consume. A well-engineered feature can outperform a model upgrade.

> **Critical Insight:** In Kaggle competitions, top competitors consistently report that 80% of their performance gains come from feature engineering, not model tuning. In production, good features also improve model interpretability and reduce training time.

---

## Encoding Strategies

### Categorical Encoding Comparison

| Method | When to Use | Pros | Cons |
|--------|------------|------|------|
| One-Hot Encoding | Low cardinality (<15 levels) | No ordinal assumption | Curse of dimensionality |
| Label Encoding | Tree-based models | Memory efficient | Implies false ordering |
| Target Encoding | High cardinality | Captures target relationship | Leakage risk |
| Frequency Encoding | When count matters | Simple, no leakage | Collisions possible |
| Binary Encoding | Medium cardinality | Compact representation | Less interpretable |
| Ordinal Encoding | Natural ordering exists | Preserves order | Spacing assumption |
| Leave-One-Out Encoding | High cardinality + regularization | Reduces leakage vs target enc | Computationally expensive |

### Python Implementation

```python
import pandas as pd
import numpy as np
from sklearn.preprocessing import OneHotEncoder, OrdinalEncoder
from category_encoders import TargetEncoder, BinaryEncoder

# One-Hot Encoding (sklearn)
ohe = OneHotEncoder(sparse_output=False, handle_unknown='ignore')
X_encoded = ohe.fit_transform(X[['category_col']])

# Target Encoding with smoothing (prevents leakage)
te = TargetEncoder(smoothing=10, min_samples_leaf=5)
X['category_target_enc'] = te.fit_transform(X['category_col'], y)

# Frequency Encoding (manual)
freq_map = X['category_col'].value_counts(normalize=True).to_dict()
X['category_freq'] = X['category_col'].map(freq_map)

# Binary Encoding
be = BinaryEncoder(cols=['category_col'])
X_binary = be.fit_transform(X)
```

> **Critical Insight:** Target encoding MUST use cross-validation folds during training to prevent data leakage. Never fit target encoding on the full training set — use `fit` on train folds only, then `transform` on validation folds.

---

## Numerical Transforms

### Common Transforms and Their Purpose

| Transform | When to Use | Effect |
|-----------|------------|--------|
| Log transform | Right-skewed data (income, prices) | Reduces skewness, stabilizes variance |
| Square root | Count data | Moderate skew reduction |
| Box-Cox | Any positive continuous | Finds optimal power transform |
| Yeo-Johnson | Continuous (incl. negatives) | Generalized Box-Cox |
| Standard scaling | Linear models, SVM, KNN | Zero mean, unit variance |
| Min-Max scaling | Neural networks, bounded features | Maps to [0, 1] |
| Robust scaling | Outlier-heavy data | Uses median and IQR |
| Quantile transform | Heavy-tailed distributions | Maps to uniform/normal |

```python
from sklearn.preprocessing import (
    StandardScaler, MinMaxScaler, RobustScaler,
    PowerTransformer, QuantileTransformer
)

# Log transform (handle zeros)
X['log_income'] = np.log1p(X['income'])

# Box-Cox (requires positive values)
pt_boxcox = PowerTransformer(method='box-cox')
X['income_boxcox'] = pt_boxcox.fit_transform(X[['income']])

# Yeo-Johnson (handles negatives)
pt_yj = PowerTransformer(method='yeo-johnson')
X['feature_yj'] = pt_yj.fit_transform(X[['feature']])

# Robust scaling for outlier-heavy data
rs = RobustScaler(quantile_range=(25.0, 75.0))
X_robust = rs.fit_transform(X[numerical_cols])

# Binning continuous features
X['age_bin'] = pd.cut(X['age'], bins=[0, 25, 35, 50, 65, 100],
                      labels=['young', 'early_career', 'mid_career', 'senior', 'retired'])
```

> **Critical Insight:** Tree-based models (Random Forest, XGBoost) are invariant to monotonic transformations. Scaling and log transforms help linear models but won't affect tree performance. However, binning CAN help trees by reducing noise.

---

## Interaction and Polynomial Features

```python
from sklearn.preprocessing import PolynomialFeatures

# Polynomial features (degree=2, interaction only)
poly = PolynomialFeatures(degree=2, interaction_only=True, include_bias=False)
X_interactions = poly.fit_transform(X[['feature_a', 'feature_b']])

# Manual interaction features (more controlled)
X['price_per_sqft'] = X['price'] / X['sqft']
X['bmi'] = X['weight'] / (X['height'] ** 2)
X['revenue_per_user'] = X['revenue'] / X['active_users']

# Ratio features (domain-driven)
X['debt_to_income'] = X['total_debt'] / X['annual_income']
X['click_through_rate'] = X['clicks'] / X['impressions']
```

| Approach | Pros | Cons |
|----------|------|------|
| PolynomialFeatures | Automated, exhaustive | Explosion of features |
| Manual interactions | Domain-driven, interpretable | Requires domain knowledge |
| Feature crosses (categorical) | Captures joint effects | High cardinality |

---

## Time-Based Features

```python
# Datetime decomposition
X['hour'] = X['timestamp'].dt.hour
X['day_of_week'] = X['timestamp'].dt.dayofweek
X['month'] = X['timestamp'].dt.month
X['is_weekend'] = X['day_of_week'].isin([5, 6]).astype(int)
X['quarter'] = X['timestamp'].dt.quarter

# Cyclical encoding (preserves circular nature)
X['hour_sin'] = np.sin(2 * np.pi * X['hour'] / 24)
X['hour_cos'] = np.cos(2 * np.pi * X['hour'] / 24)
X['month_sin'] = np.sin(2 * np.pi * X['month'] / 12)
X['month_cos'] = np.cos(2 * np.pi * X['month'] / 12)

# Lag features (time series)
X['sales_lag_1'] = X.groupby('store_id')['sales'].shift(1)
X['sales_lag_7'] = X.groupby('store_id')['sales'].shift(7)

# Rolling statistics
X['sales_rolling_7d_mean'] = (
    X.groupby('store_id')['sales']
    .transform(lambda x: x.rolling(7, min_periods=1).mean())
)
X['sales_rolling_7d_std'] = (
    X.groupby('store_id')['sales']
    .transform(lambda x: x.rolling(7, min_periods=1).std())
)

# Time since event
X['days_since_last_purchase'] = (X['current_date'] - X['last_purchase_date']).dt.days
```

> **Critical Insight:** Lag and rolling features are the most common source of data leakage in time series. ALWAYS ensure your lag window only looks backward from the prediction point. Use `shift()` carefully and validate that no future information leaks.

---

## Text Features

```python
from sklearn.feature_extraction.text import TfidfVectorizer, CountVectorizer

# TF-IDF features
tfidf = TfidfVectorizer(max_features=5000, ngram_range=(1, 2),
                        min_df=5, max_df=0.95)
X_text = tfidf.fit_transform(X['description'])

# Basic text statistics
X['text_length'] = X['description'].str.len()
X['word_count'] = X['description'].str.split().str.len()
X['avg_word_length'] = X['description'].apply(
    lambda x: np.mean([len(w) for w in x.split()]) if x else 0
)
X['uppercase_ratio'] = X['description'].apply(
    lambda x: sum(1 for c in x if c.isupper()) / len(x) if x else 0
)

# Sentiment (using pre-trained)
from textblob import TextBlob
X['sentiment'] = X['description'].apply(lambda x: TextBlob(x).sentiment.polarity)
```

---

## Feature Selection Methods

### Filter Methods (Pre-model)

```python
from sklearn.feature_selection import (
    mutual_info_classif, f_classif, chi2,
    SelectKBest, VarianceThreshold
)

# Variance threshold (remove near-constant features)
vt = VarianceThreshold(threshold=0.01)
X_filtered = vt.fit_transform(X)

# Mutual information (non-linear relationships)
mi_scores = mutual_info_classif(X, y, random_state=42)
mi_ranking = pd.Series(mi_scores, index=X.columns).sort_values(ascending=False)

# Correlation-based removal
corr_matrix = X.corr().abs()
upper_tri = corr_matrix.where(np.triu(np.ones(corr_matrix.shape), k=1).astype(bool))
high_corr_cols = [col for col in upper_tri.columns if any(upper_tri[col] > 0.95)]
X_reduced = X.drop(columns=high_corr_cols)
```

### Wrapper Methods (Model-dependent)

```python
from sklearn.feature_selection import RFE, SequentialFeatureSelector
from sklearn.ensemble import RandomForestClassifier

# Recursive Feature Elimination
rfe = RFE(estimator=RandomForestClassifier(n_estimators=100, random_state=42),
          n_features_to_select=20, step=5)
rfe.fit(X, y)
selected_features = X.columns[rfe.support_]

# Sequential Feature Selection (forward)
sfs = SequentialFeatureSelector(
    RandomForestClassifier(n_estimators=50, random_state=42),
    n_features_to_select=15, direction='forward', cv=5
)
sfs.fit(X, y)
```

### Embedded Methods (During training)

```python
from sklearn.linear_model import LassoCV
from sklearn.ensemble import GradientBoostingClassifier

# L1 regularization (Lasso) for feature selection
lasso = LassoCV(cv=5, random_state=42)
lasso.fit(X, y)
selected = X.columns[lasso.coef_ != 0]

# Tree-based feature importance
gb = GradientBoostingClassifier(n_estimators=200, random_state=42)
gb.fit(X, y)
importance_df = pd.DataFrame({
    'feature': X.columns,
    'importance': gb.feature_importances_
}).sort_values('importance', ascending=False)
```

| Category | Method | Speed | Considers Interactions | Model-Agnostic |
|----------|--------|-------|----------------------|----------------|
| Filter | Variance Threshold | Fast | No | Yes |
| Filter | Mutual Information | Medium | Yes (nonlinear) | Yes |
| Filter | Correlation | Fast | Pairwise only | Yes |
| Wrapper | RFE | Slow | Yes | No |
| Wrapper | Sequential Selection | Very Slow | Yes | No |
| Embedded | L1 (Lasso) | Medium | Limited | No |
| Embedded | Tree Importance | Medium | Yes | No |

---

## Feature Importance vs Permutation Importance

```python
from sklearn.inspection import permutation_importance

# Built-in feature importance (biased toward high-cardinality)
model = RandomForestClassifier(n_estimators=200, random_state=42)
model.fit(X_train, y_train)
builtin_importance = model.feature_importances_

# Permutation importance (unbiased, model-agnostic)
perm_imp = permutation_importance(
    model, X_val, y_val, n_repeats=30, random_state=42, scoring='roc_auc'
)
perm_importance_df = pd.DataFrame({
    'feature': X.columns,
    'importance_mean': perm_imp.importances_mean,
    'importance_std': perm_imp.importances_std
}).sort_values('importance_mean', ascending=False)
```

| Aspect | Built-in (MDI) | Permutation Importance |
|--------|---------------|----------------------|
| Bias | Favors high-cardinality | Unbiased |
| Computed on | Training data | Validation data |
| Correlated features | Splits importance | Can underestimate both |
| Speed | Free (from training) | Requires re-evaluation |
| Interpretation | Information gain | Performance drop |

> **Critical Insight:** Built-in tree feature importance (MDI) is computed on training data and is biased toward high-cardinality and continuous features. ALWAYS validate with permutation importance on held-out data. For correlated features, consider SHAP values which handle multicollinearity better.

---

## Data Leakage Prevention

### Types of Leakage

1. **Target leakage**: Feature derived from the target (e.g., "is_fraud_reported" predicting fraud)
2. **Train-test contamination**: Preprocessing uses test data statistics
3. **Temporal leakage**: Using future information in time-series features
4. **Group leakage**: Same entity appears in train and test (e.g., same patient)

### Prevention Strategies

```python
# WRONG: Fitting scaler on all data
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)  # Leakage!
X_train, X_test = train_test_split(X_scaled, ...)

# RIGHT: Fit only on training data
X_train, X_test = train_test_split(X, ...)
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)  # Only transform!

# WRONG: Target encoding on full dataset
X['city_encoded'] = X.groupby('city')['target'].transform('mean')  # Leakage!

# RIGHT: Target encoding with CV folds
from sklearn.model_selection import KFold
kf = KFold(n_splits=5, shuffle=True, random_state=42)
X['city_encoded'] = np.nan
for train_idx, val_idx in kf.split(X):
    means = X.iloc[train_idx].groupby('city')['target'].mean()
    X.loc[X.index[val_idx], 'city_encoded'] = X.iloc[val_idx]['city'].map(means)
```

---

## Sklearn Pipelines

```python
from sklearn.pipeline import Pipeline, make_pipeline
from sklearn.compose import ColumnTransformer
from sklearn.impute import SimpleImputer

# Define column groups
numerical_cols = ['age', 'income', 'tenure']
categorical_cols = ['city', 'gender', 'plan_type']

# Numerical pipeline
numerical_pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='median')),
    ('scaler', StandardScaler()),
])

# Categorical pipeline
categorical_pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='most_frequent')),
    ('encoder', OneHotEncoder(handle_unknown='ignore', sparse_output=False)),
])

# Combined preprocessor
preprocessor = ColumnTransformer([
    ('num', numerical_pipeline, numerical_cols),
    ('cat', categorical_pipeline, categorical_cols),
], remainder='drop')

# Full pipeline with model
from sklearn.ensemble import GradientBoostingClassifier
full_pipeline = Pipeline([
    ('preprocessor', preprocessor),
    ('feature_selection', SelectKBest(mutual_info_classif, k=20)),
    ('classifier', GradientBoostingClassifier(n_estimators=200))
])

# Fit and predict (no leakage!)
full_pipeline.fit(X_train, y_train)
y_pred = full_pipeline.predict(X_test)

# Cross-validation with pipeline (leakage-free)
from sklearn.model_selection import cross_val_score
scores = cross_val_score(full_pipeline, X_train, y_train, cv=5, scoring='roc_auc')
```

> **Critical Insight:** Pipelines are the ONLY safe way to ensure preprocessing is applied correctly during cross-validation and production deployment. Every transform that learns from data (imputation, scaling, encoding) MUST be inside the pipeline. This guarantees the transform is fit only on training folds.

---

## Interview Talking Points

> "In my experience at [company], feature engineering was the highest-leverage activity. For a churn prediction model, I engineered time-based features like days-since-last-login and rolling 7-day session counts. These behavioral features improved AUC from 0.72 to 0.84 — more than any model architecture change."

> "I always use sklearn Pipelines to prevent data leakage. In one project, I discovered that a colleague had fit the StandardScaler on the full dataset before splitting. After fixing this with a proper pipeline, our test AUC dropped from 0.91 to 0.85 — that 6-point gap was pure leakage."

> "For feature selection, I use a layered approach: first remove zero-variance and highly correlated features (filter), then use permutation importance on a baseline model (embedded), and finally validate with SHAP values to understand feature interactions."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| One-hot encoding high-cardinality features (1000+ levels) | Use target encoding or embedding |
| Scaling features before train/test split | Scale inside pipeline, fit on train only |
| Using target encoding without CV fold separation | Use fold-based target encoding |
| Creating lag features without checking for temporal leakage | Verify all lags point strictly backward |
| Using all polynomial features without selection | Generate then select top-k by importance |
| Treating tree importance as ground truth | Validate with permutation importance on val set |
| Ignoring feature correlations before selection | Remove highly correlated pairs first |
| Feature engineering on full dataset then splitting | Split first, engineer within pipeline |
| Using future data in rolling window features | Ensure window only covers past observations |
| Not handling unknown categories at inference | Use handle_unknown='ignore' in encoders |

---

## Rapid-Fire Q&A

**Q1: When would you use target encoding over one-hot encoding?**
A: When cardinality exceeds ~15 categories. Target encoding maps each category to the smoothed mean of the target variable, avoiding dimensionality explosion while capturing predictive signal.

**Q2: Why do tree-based models not need feature scaling?**
A: Trees split on thresholds — they only care about feature ordering, not magnitude. A split at "income > 50000" works identically whether income is raw or standardized.

**Q3: How do you handle cyclical features like hour-of-day?**
A: Use sin/cos encoding: `sin(2*pi*hour/24)` and `cos(2*pi*hour/24)`. This preserves the circular relationship where hour 23 is close to hour 0.

**Q4: What is the difference between filter, wrapper, and embedded feature selection?**
A: Filter methods rank features independently (mutual information), wrapper methods search subsets using a model (RFE), embedded methods select during training (L1 regularization, tree importance).

**Q5: How does L1 regularization perform feature selection?**
A: L1 penalty adds |coefficient| to the loss. The diamond-shaped constraint region causes some coefficients to become exactly zero, effectively removing those features.

**Q6: What is data leakage and how do you prevent it?**
A: Data leakage occurs when information from outside the training data influences the model. Prevent it by using pipelines, splitting before preprocessing, and validating temporal integrity of features.

**Q7: Why is permutation importance preferred over built-in feature importance?**
A: Built-in MDI importance is biased toward high-cardinality features and computed on training data. Permutation importance is unbiased, computed on validation data, and works for any model.

**Q8: When would you bin a continuous feature?**
A: When the relationship with target is non-monotonic (age groups), when you want interaction with categorical features, or when reducing noise helps tree models avoid overfitting.

**Q9: How do you handle features with many missing values?**
A: Add a binary "is_missing" indicator feature, then impute. For tree models, imputation may not matter (XGBoost handles NaN natively). For linear models, median/mean imputation is common.

**Q10: What makes a good feature in production?**
A: Available at prediction time, not leaking future information, stable over time (no concept drift), computationally cheap to compute, and interpretable to stakeholders.

---

## ASCII Cheat Sheet

```
+------------------------------------------------------------------+
|              FEATURE ENGINEERING DECISION TREE                     |
+------------------------------------------------------------------+
|                                                                    |
|  Raw Data Type?                                                    |
|  |                                                                 |
|  +-- Categorical                                                   |
|  |   +-- Low cardinality (<15) --> One-Hot Encoding                |
|  |   +-- High cardinality (15+) --> Target/Frequency Encoding      |
|  |   +-- Ordinal nature --> Ordinal Encoding                       |
|  |                                                                 |
|  +-- Numerical                                                     |
|  |   +-- Skewed? --> Log/Box-Cox Transform                         |
|  |   +-- Outliers? --> Robust Scaler / Winsorize                   |
|  |   +-- For linear model? --> StandardScaler                      |
|  |   +-- For neural net? --> MinMaxScaler                          |
|  |                                                                 |
|  +-- DateTime                                                      |
|  |   +-- Extract: hour, dow, month, quarter                        |
|  |   +-- Cyclical: sin/cos encoding                                |
|  |   +-- Lags: shift(1), shift(7)                                  |
|  |   +-- Rolling: mean, std over window                            |
|  |                                                                 |
|  +-- Text                                                          |
|      +-- TF-IDF / Count Vectors                                    |
|      +-- Length, word count, sentiment                              |
|      +-- Embeddings (if deep learning)                             |
|                                                                    |
+------------------------------------------------------------------+
|              FEATURE SELECTION STRATEGY                            |
+------------------------------------------------------------------+
|                                                                    |
|  Step 1: Remove zero/low variance          [Filter]                |
|  Step 2: Remove highly correlated (>0.95)  [Filter]                |
|  Step 3: Mutual Information ranking        [Filter]                |
|  Step 4: Permutation importance            [Embedded]              |
|  Step 5: RFE if features still too many    [Wrapper]               |
|  Step 6: Validate with SHAP               [Interpretability]       |
|                                                                    |
+------------------------------------------------------------------+
|              LEAKAGE PREVENTION CHECKLIST                          |
+------------------------------------------------------------------+
|                                                                    |
|  [ ] All preprocessing inside sklearn Pipeline                     |
|  [ ] Train/test split BEFORE any feature engineering               |
|  [ ] Target encoding uses CV folds (not full train)                |
|  [ ] Time-series features use only past data                       |
|  [ ] No features derived from target variable                      |
|  [ ] Group splits for repeated measurements                        |
|  [ ] Validation metric matches production performance              |
|                                                                    |
+------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 29 of 45*
