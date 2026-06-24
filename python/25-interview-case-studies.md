# 🎯 Topic 25: Interview Case Studies

> **Data Science Interview — End-to-End Projects**
> Five complete case studies structured like real DS interview take-home assignments. Each covers problem framing, features, modeling, evaluation, and business impact — the complete arc interviewers want to see.

---

## Table of Contents

1. [How to Present Case Studies](#how-to-present-case-studies)
2. [Case Study 1: Customer Churn Prediction](#case-study-1-customer-churn-prediction)
3. [Case Study 2: Recommendation System](#case-study-2-recommendation-system)
4. [Case Study 3: Fraud Detection](#case-study-3-fraud-detection)
5. [Case Study 4: Search Ranking](#case-study-4-search-ranking)
6. [Case Study 5: Pricing Optimization](#case-study-5-pricing-optimization)
7. [Interview Talking Points & Scripts](#interview-talking-points--scripts)
8. [Common Interview Mistakes](#common-interview-mistakes)
9. [Rapid-Fire Q&A](#rapid-fire-qa)
10. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## How to Present Case Studies

### The 5-Minute Framework

Every case study presentation should follow this structure:

| Step | Time | What to Cover |
|------|------|---------------|
| **Problem** | 1 min | Business context, success metric, constraints |
| **EDA** | 1 min | Data shape, distributions, key insights |
| **Approach** | 1.5 min | Features, model choice, why this over alternatives |
| **Results** | 1 min | Metrics, comparison to baseline, error analysis |
| **Impact** | 0.5 min | Business value in dollars/percentage, next steps |

**Golden Rule:** Start with the business problem, end with business impact. Technical details are the bridge, not the destination.

---

## Case Study 1: Customer Churn Prediction

### Problem Framing

- **Business Question:** Which subscribers will cancel in the next 30 days?
- **Success Metric:** Reduce monthly churn rate from 5.2% to 4.0%
- **Constraint:** Retention offers cost $50/user; only intervene on high-confidence predictions

### Key Features

| Feature Category | Examples | Rationale |
|-----------------|----------|-----------|
| Tenure | months_active, contract_type | New users churn more |
| Recency | days_since_last_login, last_purchase_days | Disengagement signal |
| Frequency | sessions_per_week, support_tickets | Usage patterns |
| Monetary | monthly_spend, plan_tier | Value commitment |
| Behavioral | feature_adoption_rate, NPS_score | Satisfaction proxy |

### Implementation

```python
import pandas as pd
import numpy as np
from sklearn.model_selection import TimeSeriesSplit
from sklearn.metrics import roc_auc_score, precision_recall_curve
import lightgbm as lgb

# Feature engineering
def build_churn_features(df):
    """Create features from raw user activity data."""
    features = pd.DataFrame()
    features['tenure_months'] = (pd.Timestamp.now() - df['signup_date']).dt.days / 30
    features['days_since_last_login'] = (pd.Timestamp.now() - df['last_login']).dt.days
    features['avg_sessions_per_week'] = df['total_sessions'] / features['tenure_months'] / 4
    features['support_tickets_last_30d'] = df['recent_tickets']
    features['monthly_spend'] = df['total_revenue'] / features['tenure_months']
    features['feature_adoption'] = df['features_used'] / df['total_features']
    # Trend features — compare last 30 days to prior 30 days
    features['session_trend'] = df['sessions_last_30d'] / (df['sessions_prior_30d'] + 1)
    features['spend_trend'] = df['spend_last_30d'] / (df['spend_prior_30d'] + 1)
    return features

# Model training with temporal validation
def train_churn_model(X, y, dates):
    tscv = TimeSeriesSplit(n_splits=5)
    aucs = []
    for train_idx, val_idx in tscv.split(X):
        X_train, X_val = X.iloc[train_idx], X.iloc[val_idx]
        y_train, y_val = y.iloc[train_idx], y.iloc[val_idx]

        model = lgb.LGBMClassifier(
            n_estimators=500, learning_rate=0.05,
            max_depth=6, num_leaves=31,
            scale_pos_weight=len(y_train[y_train==0]) / len(y_train[y_train==1])
        )
        model.fit(X_train, y_train, eval_set=[(X_val, y_val)],
                  callbacks=[lgb.early_stopping(50)])
        aucs.append(roc_auc_score(y_val, model.predict_proba(X_val)[:, 1]))
    print(f"Mean AUC: {np.mean(aucs):.4f} (+/- {np.std(aucs):.4f})")
    return model

# Threshold tuning for business constraint
def find_optimal_threshold(y_true, y_scores, offer_cost=50, customer_ltv=600):
    precisions, recalls, thresholds = precision_recall_curve(y_true, y_scores)
    profits = []
    for p, r, t in zip(precisions[:-1], recalls[:-1], thresholds):
        true_churners_caught = r * y_true.sum()
        false_positives = (1 - p) * true_churners_caught / p
        profit = true_churners_caught * customer_ltv - (true_churners_caught + false_positives) * offer_cost
        profits.append(profit)
    best_idx = np.argmax(profits)
    return thresholds[best_idx], profits[best_idx]
```

### Metrics & Business Impact

- **AUC:** 0.87 | **Precision@top-10%:** 0.72
- **Business Impact:** Targeting top 10% risk scores saved **$2.1M annually** (4,200 users retained x $500 LTV)

---

## Case Study 2: Recommendation System

### Problem Framing

- **Business Question:** What items should we show on the homepage to maximize engagement?
- **Success Metric:** Increase click-through rate from 3.2% to 5.0%
- **Approach Trade-off:** Collaborative filtering (behavioral) vs. content-based (item attributes)

### Architecture

```
User History --> Collaborative Filtering (ALS) --\
                                                  +--> Hybrid Ranker --> Top-K Items
Item Metadata --> Content-Based (TF-IDF sim)  --/
```

### Implementation

```python
import numpy as np
from scipy.sparse import csr_matrix
from sklearn.metrics.pairwise import cosine_similarity

class HybridRecommender:
    """Combines collaborative filtering with content-based recommendations."""

    def __init__(self, n_factors=50, alpha=0.7):
        self.n_factors = n_factors
        self.alpha = alpha  # weight for collaborative vs content

    def fit_collaborative(self, interactions_matrix):
        """ALS-based matrix factorization."""
        from implicit.als import AlternatingLeastSquares
        model = AlternatingLeastSquares(factors=self.n_factors, iterations=20)
        model.fit(csr_matrix(interactions_matrix))
        self.user_factors = model.user_factors
        self.item_factors = model.item_factors
        return self

    def fit_content(self, item_features):
        """Content-based similarity using item attributes."""
        self.item_similarity = cosine_similarity(item_features)
        return self

    def recommend(self, user_id, user_history, n=10):
        # Collaborative scores
        collab_scores = self.user_factors[user_id] @ self.item_factors.T
        # Content scores — average similarity to items user liked
        content_scores = self.item_similarity[user_history].mean(axis=0)
        # Hybrid blend
        final_scores = self.alpha * collab_scores + (1 - self.alpha) * content_scores
        # Exclude already-seen items
        final_scores[user_history] = -np.inf
        return np.argsort(final_scores)[-n:][::-1]

    def handle_cold_start(self, item_features, n=10):
        """For new users: recommend popular items in diverse categories."""
        # Fallback: popularity-weighted diverse selection
        popular_items = self.popularity_scores.argsort()[-50:][::-1]
        # Ensure category diversity via MMR
        return self._maximal_marginal_relevance(popular_items, item_features, n)
```

### Evaluation

| Metric | Collaborative | Content-Based | Hybrid |
|--------|:------------:|:-------------:|:------:|
| NDCG@10 | 0.34 | 0.28 | **0.41** |
| MAP@10 | 0.22 | 0.18 | **0.27** |
| Coverage | 45% | 72% | **68%** |
| Cold-Start NDCG | 0.05 | 0.25 | **0.22** |

### Cold Start Solutions

1. **New users:** Content-based until 5+ interactions, then blend in collaborative
2. **New items:** Use item metadata similarity to bootstrap scores
3. **Explore-exploit:** Epsilon-greedy to surface new items (epsilon=0.1)

---

## Case Study 3: Fraud Detection

### Problem Framing

- **Business Question:** Flag fraudulent transactions in real-time (< 100ms latency)
- **Class Imbalance:** 0.1% fraud rate (1,000 fraud in 1M transactions)
- **Key Constraint:** Cannot block > 2% of legitimate transactions (false positive rate)

### Feature Engineering

```python
import pandas as pd
import numpy as np

def engineer_fraud_features(transactions):
    """Create velocity, distance, and time-based fraud signals."""
    features = pd.DataFrame(index=transactions.index)

    # Velocity features — how fast is money moving?
    features['txn_count_1h'] = transactions.groupby('user_id')['amount'].transform(
        lambda x: x.rolling('1H').count()
    )
    features['amount_sum_24h'] = transactions.groupby('user_id')['amount'].transform(
        lambda x: x.rolling('24H').sum()
    )
    features['amount_vs_avg'] = transactions['amount'] / transactions.groupby('user_id')['amount'].transform('mean')

    # Distance features — is this geographically unusual?
    features['distance_from_home'] = haversine(
        transactions[['lat', 'lon']].values,
        transactions[['home_lat', 'home_lon']].values
    )
    features['distance_from_last_txn'] = transactions.groupby('user_id').apply(
        lambda g: haversine(g[['lat','lon']].values[1:], g[['lat','lon']].values[:-1])
    )

    # Time features — unusual timing?
    features['hour_of_day'] = transactions['timestamp'].dt.hour
    features['is_weekend'] = transactions['timestamp'].dt.weekday >= 5
    features['minutes_since_last_txn'] = transactions.groupby('user_id')['timestamp'].diff().dt.total_seconds() / 60

    # Device/merchant risk
    features['merchant_fraud_rate'] = transactions['merchant_id'].map(merchant_fraud_rates)
    features['new_device'] = (~transactions['device_id'].isin(known_devices)).astype(int)

    return features
```

### Handling Imbalance & Threshold Tuning

```python
from sklearn.metrics import precision_recall_curve, f1_score
from imblearn.over_sampling import SMOTE
import xgboost as xgb

def train_fraud_model(X_train, y_train, X_val, y_val):
    # Strategy 1: Class weights (preferred for production)
    model = xgb.XGBClassifier(
        scale_pos_weight=y_train.value_counts()[0] / y_train.value_counts()[1],
        max_depth=5, n_estimators=300, learning_rate=0.1,
        eval_metric='aucpr'  # area under precision-recall curve
    )
    model.fit(X_train, y_train, eval_set=[(X_val, y_val)], early_stopping_rounds=30)

    # Threshold tuning: maximize F1 subject to FPR < 2%
    y_scores = model.predict_proba(X_val)[:, 1]
    precisions, recalls, thresholds = precision_recall_curve(y_val, y_scores)

    best_threshold = 0.5
    best_f1 = 0
    for p, r, t in zip(precisions[:-1], recalls[:-1], thresholds):
        fpr = (1 - p) * r * y_val.sum() / (len(y_val) - y_val.sum())
        if fpr <= 0.02:  # constraint: FPR < 2%
            f1 = 2 * p * r / (p + r) if (p + r) > 0 else 0
            if f1 > best_f1:
                best_f1, best_threshold = f1, t

    print(f"Optimal threshold: {best_threshold:.3f}, F1: {best_f1:.3f}")
    return model, best_threshold
```

### Results

- **Precision:** 0.68 | **Recall:** 0.82 | **FPR:** 1.8% (within constraint)
- **Business Impact:** Prevented **$4.3M in annual fraud losses** while blocking only 1.8% of good transactions

---

## Case Study 4: Search Ranking

### Problem Framing

- **Business Question:** Rank search results to maximize user satisfaction (measured by clicks + conversions)
- **Approach:** Learning-to-Rank (LambdaMART)
- **Offline Metric:** NDCG@10 | **Online Metric:** Click-through rate, time-to-click

### Features

| Feature Type | Examples | Signal |
|-------------|----------|--------|
| Text Match | BM25 score, title match, exact match | Relevance |
| Click History | CTR for query-doc pair, dwell time | Engagement |
| Freshness | document_age, last_updated | Recency value |
| Authority | PageRank, domain authority, backlinks | Quality |
| User Context | location, past queries, device | Personalization |

### Implementation

```python
import lightgbm as lgb
import numpy as np
from sklearn.metrics import ndcg_score

def train_ranking_model(train_queries, train_labels, train_features):
    """Train LambdaMART model for search ranking."""
    # Group by query for pairwise learning
    query_groups = train_queries.groupby('query_id').size().values

    train_data = lgb.Dataset(
        train_features, label=train_labels,
        group=query_groups
    )

    params = {
        'objective': 'lambdarank',
        'metric': 'ndcg',
        'ndcg_eval_at': [5, 10],
        'learning_rate': 0.05,
        'num_leaves': 63,
        'min_data_in_leaf': 50,
        'feature_fraction': 0.8
    }

    model = lgb.train(params, train_data, num_boost_round=500)
    return model

def evaluate_ranking(model, test_queries, test_features, test_labels):
    """Compute NDCG@10 per query and report aggregate."""
    scores = model.predict(test_features)
    ndcgs = []
    for qid in test_queries['query_id'].unique():
        mask = test_queries['query_id'] == qid
        true_rel = test_labels[mask].values.reshape(1, -1)
        pred_scores = scores[mask].reshape(1, -1)
        if true_rel.shape[1] > 1:
            ndcgs.append(ndcg_score(true_rel, pred_scores, k=10))
    return np.mean(ndcgs)

# Offline vs Online metric comparison
def offline_online_correlation(offline_ndcg, online_ctr):
    """Check if offline improvements translate to online gains."""
    correlation = np.corrcoef(offline_ndcg, online_ctr)[0, 1]
    print(f"Offline-Online correlation: {correlation:.3f}")
    # Rule of thumb: correlation > 0.6 means offline metric is trustworthy
    return correlation
```

### Offline vs Online Metrics

| Metric Type | Metric | Value | Notes |
|------------|--------|-------|-------|
| Offline | NDCG@10 | 0.72 | +8% over BM25 baseline |
| Offline | MAP@10 | 0.58 | +12% over baseline |
| Online | CTR | 34.2% | +3.1pp vs. control (A/B test) |
| Online | Time-to-click | 4.2s | -1.8s faster (good) |

---

## Case Study 5: Pricing Optimization

### Problem Framing

- **Business Question:** What price maximizes revenue for each product category?
- **Approach:** Estimate price elasticity, find revenue-maximizing price
- **Constraint:** Prices must stay within +/-20% of current to avoid brand damage

### Price Elasticity Model

```python
import numpy as np
import pandas as pd
from scipy.optimize import minimize_scalar
import statsmodels.api as sm

def estimate_elasticity(price_data):
    """Log-log demand model: log(Q) = a + b*log(P) + controls.
    Coefficient b is the price elasticity of demand."""
    price_data['log_price'] = np.log(price_data['price'])
    price_data['log_quantity'] = np.log(price_data['quantity_sold'] + 1)

    # Control for seasonality and competitor pricing
    X = price_data[['log_price', 'month_sin', 'month_cos', 'competitor_price_index']]
    X = sm.add_constant(X)
    y = price_data['log_quantity']

    model = sm.OLS(y, X).fit(cov_type='HC1')  # robust standard errors
    elasticity = model.params['log_price']
    print(f"Price Elasticity: {elasticity:.3f} (p={model.pvalues['log_price']:.4f})")
    print(f"Interpretation: 1% price increase -> {elasticity:.1f}% demand change")
    return model, elasticity

def find_optimal_price(current_price, elasticity, margin_pct=0.4):
    """Revenue = P * Q(P). With log-log model: Q = A * P^elasticity.
    Optimal price for revenue: P* = current_price * elasticity / (elasticity + 1)
    Optimal price for profit: P* = marginal_cost * elasticity / (elasticity + 1)"""

    marginal_cost = current_price * (1 - margin_pct)

    # Revenue-maximizing price
    if elasticity < -1:  # elastic demand
        optimal_revenue_price = current_price * elasticity / (elasticity + 1)
    else:
        optimal_revenue_price = current_price * 1.20  # hit upper constraint

    # Profit-maximizing price (Lerner index)
    optimal_profit_price = marginal_cost * elasticity / (elasticity + 1)

    # Apply constraints
    lower_bound = current_price * 0.80
    upper_bound = current_price * 1.20
    final_price = np.clip(optimal_profit_price, lower_bound, upper_bound)

    print(f"Current: ${current_price:.2f} | Optimal: ${final_price:.2f}")
    print(f"Expected revenue change: {((final_price/current_price)**(1+elasticity) - 1)*100:.1f}%")
    return final_price

# A/B testing prices
def design_price_experiment(products, n_variants=3):
    """Randomized price experiment design."""
    experiments = []
    for product in products:
        base_price = product['current_price']
        variants = np.linspace(base_price * 0.9, base_price * 1.1, n_variants)
        experiments.append({
            'product_id': product['id'],
            'variants': variants,
            'min_sample_per_variant': 1000,  # for 80% power at 5% MDE
            'duration_days': 14
        })
    return experiments
```

### Results

- **Elasticity Found:** -1.8 (elastic — price decreases boost revenue)
- **Optimal Price:** $42.50 (down from $49.99)
- **Revenue Impact:** +12% revenue lift in A/B test, projected **$8.2M annual gain**

---

## Interview Talking Points & Scripts

### How to Explain Model Choice

> "I chose gradient boosting (LightGBM) for three reasons: first, the data has mixed feature types — numerical and categorical — which tree models handle natively. Second, we have moderate data size (500K rows) where GBM outperforms deep learning. Third, feature importance from GBM directly answers stakeholder questions about what drives predictions."

### How to Quantify Business Impact

> "We can frame impact as: (number of correct predictions) x (value per correct action) - (cost of false positives). In the churn case: we correctly identified 4,200 churners, retention offers cost $50 each ($210K total cost), and retained customers have $500 LTV, so net impact is 4,200 x $500 - 4,200 x $50 = $1.89M."

### How to Discuss Trade-offs

> "The key trade-off was precision vs. recall. Higher recall catches more fraud but increases false positives, which damages user experience. We set our threshold where FPR stays below 2% — this was a business constraint from the product team, not a purely statistical decision."

---

## Common Interview Mistakes

| ❌ Mistake | ✅ Better Approach |
|-----------|-------------------|
| Jumping straight to modeling | Frame the business problem first, define success metric |
| Using accuracy on imbalanced data | Use AUC, precision@k, or F1 on the minority class |
| Not having a baseline | Always compare to a simple baseline (mean, logistic regression) |
| Ignoring data leakage | Use temporal splits; never use future data to predict past |
| Overfitting to offline metrics | Discuss online A/B testing to validate real-world impact |
| Not quantifying impact | Always convert to dollars, users affected, or percentage gains |
| Presenting only the final model | Show iteration: baseline → improvements → final model |
| Ignoring deployment constraints | Mention latency, memory, retraining frequency |
| Using jargon without explanation | "We used SMOTE" → "We oversampled fraud cases because..." |
| No error analysis | Always discuss where the model fails and why |

---

## Rapid-Fire Q&A

**Q1: How do you choose between precision and recall?**
A: Depends on the cost ratio. High false-positive cost (spam filter) → optimize precision. High false-negative cost (fraud, medical) → optimize recall.

**Q2: When would you NOT use a complex model?**
A: When interpretability is required (regulatory), data is tiny (< 1K rows), or a simple model already achieves the business threshold.

**Q3: How do you handle missing data in production?**
A: Train with missingness patterns intact; use model-native handling (LightGBM handles NaN). Never impute with test-set statistics.

**Q4: What's your first step on a new dataset?**
A: Check shape, dtypes, missing rates, target distribution, and time range. Then one visualization per feature type.

**Q5: How do you detect data leakage?**
A: If model performance seems "too good" (AUC > 0.99), inspect feature importance — leaked features will dominate.

**Q6: How do you decide sample size for an A/B test?**
A: Power analysis: specify MDE (minimum detectable effect), significance level (0.05), power (0.80), and baseline conversion rate.

**Q7: When do you retrain a model?**
A: When monitoring shows performance decay (distribution shift), or on a fixed schedule (weekly/monthly) validated by holdout performance.

**Q8: How do you explain a model to non-technical stakeholders?**
A: Lead with business impact, use SHAP for individual predictions ("this customer churned because their login frequency dropped 60%").

**Q9: What do you do when your model performs well offline but poorly online?**
A: Check for train-serve skew, feature freshness issues, selection bias in training data, or latency causing stale predictions.

**Q10: How do you prioritize features to build?**
A: Expected lift (from EDA/domain knowledge) x feasibility (data availability, compute cost). Build high-lift easy features first.

---

## Summary Cheat Sheet

```
+------------------------------------------------------------------+
|              INTERVIEW CASE STUDY FRAMEWORK                       |
+------------------------------------------------------------------+
|                                                                  |
|  Step 1: PROBLEM   - Business context, metric, constraints      |
|  Step 2: EDA       - Data shape, distributions, key insights    |
|  Step 3: APPROACH  - Features, model, why this over alternatives|
|  Step 4: RESULTS   - Metrics vs. baseline, error analysis       |
|  Step 5: IMPACT    - Dollars saved, users affected, next steps  |
|                                                                  |
+------------------------------------------------------------------+
|  KEY PRINCIPLES:                                                 |
|  - Always have a baseline (even naive)                          |
|  - Use temporal validation for time-series problems             |
|  - Convert metrics to business value                            |
|  - Discuss trade-offs, not just final numbers                   |
|  - Show iteration: V1 -> V2 -> V3                              |
+------------------------------------------------------------------+
|  MODELS BY PROBLEM:                                              |
|  Churn/Classification -> GBM (LightGBM/XGBoost)                |
|  Recommendations     -> Hybrid (CF + Content)                   |
|  Fraud Detection     -> GBM + threshold tuning                  |
|  Search Ranking      -> LambdaMART (Learning-to-Rank)           |
|  Pricing             -> Log-log regression + optimization       |
+------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 25 of 25*
