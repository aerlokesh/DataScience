# 🎯 Topic 35: When NOT to Use Machine Learning

> *"The most underrated skill in data science is knowing when a simple rule, a SQL query, or a conversation with a domain expert is better than a model."*

---

## Table of Contents

1. [The ML Hammer Problem](#the-ml-hammer-problem)
2. [Rules-Based Alternatives](#rules-based-alternatives)
3. [When Data Is Insufficient](#when-data-is-insufficient)
4. [When Interpretability Is Critical](#when-interpretability-is-critical)
5. [When Latency or Cost Prohibits](#when-latency-or-cost-prohibits)
6. [When the Problem Isn't ML-Shaped](#when-the-problem-isnt-ml-shaped)
7. [Heuristics That Beat ML](#heuristics-that-beat-ml)
8. [The Decision Framework](#the-decision-framework)
9. [Examples of ML Overkill](#examples-of-ml-overkill)
10. [Maintenance Cost of ML Systems](#maintenance-cost-of-ml-systems)
11. [Interview Talking Points](#interview-talking-points)
12. [Common Mistakes](#common-mistakes)
13. [Rapid-Fire Q&A](#rapid-fire-qa)
14. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## The ML Hammer Problem

> "When all you have is a hammer, everything looks like a nail."

Machine learning is powerful, but it introduces significant complexity. Before reaching for ML, ask:

1. **Is there a pattern to learn?** (If the relationship is known and deterministic, code it directly)
2. **Can I get enough labeled data?** (ML needs data; rules need expertise)
3. **Is the cost of errors acceptable?** (ML makes mistakes; rules are predictable)
4. **Can I maintain this?** (ML systems degrade; rules are stable)
5. **Does the ROI justify the investment?** (ML is expensive to build and maintain)

> **Critical Insight:** Google's research paper "Rules of Machine Learning" (Martin Zinkevich) states Rule #1: "Don't be afraid to launch a product without machine learning." Many successful products launched with simple heuristics and only added ML later when the basic system was validated and data was abundant.

---

## Rules-Based Alternatives

### When Rules Win

| Scenario | Rule-Based Solution | Why Not ML |
|----------|-------------------|------------|
| Business logic with clear criteria | If age >= 65, apply senior discount | Deterministic, no learning needed |
| Regulatory compliance | If income < threshold, flag for review | Must be explainable and auditable |
| Known domain patterns | If temp > 100°F, alert doctor | Expert knowledge is complete |
| Low volume, high stakes | Manual review of >$1M transactions | Not enough data, too costly to err |
| Exact string matching | If email contains "unsubscribe" | Pattern is known exactly |
| Simple thresholds | If response time > 5s, page on-call | Threshold is well-defined |

### Rule Engine Pattern

```python
# Priority-based rule engine: evaluate rules in order, return first match
# Example: Fraud detection rules (the FIRST line of defense in production)

def evaluate_fraud_rules(transaction):
    """Rules handle 80% of clear-cut cases; ML handles the ambiguous 20%."""
    if transaction['amount'] > 10000 and transaction['country'] != transaction['home_country']:
        return {'decision': 'block', 'reason': 'large_foreign_transaction'}
    if transaction['velocity_1h'] > 5:
        return {'decision': 'review', 'reason': 'high_velocity'}
    if transaction['merchant_category'] in ['gambling', 'crypto'] and transaction['amount'] > 500:
        return {'decision': 'review', 'reason': 'high_risk_merchant'}
    return None  # No rule matched -> pass to ML model
```

### Rules + ML Hybrid (Best Practice)

```python
def fraud_decision_pipeline(transaction):
    """
    Layered approach: Rules first, ML for ambiguous cases.
    """
    # Layer 1: Hard rules (instant, deterministic)
    if transaction['amount'] > 50000:
        return 'BLOCK', 'rule: extreme_amount'
    if transaction['card_country'] != transaction['merchant_country'] and \
       transaction['amount'] > 5000:
        return 'REVIEW', 'rule: foreign_high_value'
    
    # Layer 2: Known-good patterns (allowlist)
    if transaction['merchant_id'] in trusted_merchants:
        return 'APPROVE', 'rule: trusted_merchant'
    
    # Layer 3: ML model (for ambiguous cases)
    ml_score = fraud_model.predict_proba(transaction.features)[0][1]
    if ml_score > 0.85:
        return 'BLOCK', f'model: score={ml_score:.3f}'
    elif ml_score > 0.50:
        return 'REVIEW', f'model: score={ml_score:.3f}'
    else:
        return 'APPROVE', f'model: score={ml_score:.3f}'
```

> **Critical Insight:** Most production ML systems are actually hybrid rule+ML systems. Rules catch obvious cases (fast, cheap, explainable), and ML handles the ambiguous middle ground. This layered approach reduces model load, provides safety nets, and ensures that critical business logic is never at the mercy of a model's vagaries.

---

## When Data Is Insufficient

### Minimum Data Guidelines

| Model Type | Rough Minimum | Why |
|-----------|---------------|-----|
| Linear Regression | 10-20x features | Needs 10 samples per coefficient |
| Logistic Regression | 10x features per class | Similar to linear, per class |
| Decision Tree | 100+ per leaf node | Needs enough for splits |
| Random Forest | 1000+ total | Ensemble averaging needs volume |
| Gradient Boosting | 1000-10000 | Sequential learning needs variety |
| Deep Learning | 10,000-100,000+ | Many parameters to learn |
| NLP (fine-tuning BERT) | 1000-5000 labeled examples | Transfer learning reduces need |

### Red Flags: Not Enough Data

- Samples-per-feature ratio < 10
- Minority class has < 50 examples
- Total dataset < 500 rows
- Signal-to-noise ratio is too low for reliable learning

When any of these hold, use rules/heuristics or collect more data before investing in ML.

### Alternatives When Data Is Scarce

| Alternative | When to Use | Example |
|-------------|------------|---------|
| Expert rules | Domain knowledge is strong | Medical protocols |
| Lookup tables | Finite, known mappings | Tax bracket calculation |
| Transfer learning | Related task has data | Pre-trained NLP models |
| Few-shot learning | Only 5-10 examples per class | LLM-based classification |
| Simulation | Can model the system | Monte Carlo pricing |
| Bayesian methods | Strong priors available | Clinical trials with prior studies |
| Simple heuristics | 80/20 rule applies | "Most recent = most relevant" |

---

## When Interpretability Is Critical

### Domains Requiring Interpretability

| Domain | Regulation/Requirement | Consequence of Black Box |
|--------|----------------------|--------------------------|
| Credit scoring | ECOA, FCRA (US); GDPR Art. 22 (EU) | Legal liability, fines |
| Healthcare | FDA approval requires explanation | Patient harm, lawsuits |
| Criminal justice | Due process rights | Wrongful decisions |
| Insurance pricing | Actuarial standards | Regulatory rejection |
| Hiring decisions | Employment discrimination laws | EEOC complaints |
| Autonomous vehicles | Liability determination | Can't explain accidents |

### The Interpretability Hierarchy

```
Level 1: Fully Transparent (rules, linear models)
  - Anyone can trace the logic
  - "Your loan was denied because debt-to-income > 0.43"

Level 2: Post-hoc Explainable (SHAP, LIME on complex models)
  - Model is black-box but explanations are generated
  - "Top factors: income (-0.3), credit history (+0.5)"

Level 3: Audit-level (model documentation, monitoring)
  - Can demonstrate fairness and stability over time
  - Required for model risk management in banking

Level 4: Black Box (deep learning without explanation)
  - No ability to explain individual decisions
  - UNACCEPTABLE for regulated decisions
```

```python
# When you MUST be interpretable: use inherently interpretable models

# Option 1: Logistic Regression (most interpretable)
from sklearn.linear_model import LogisticRegression
model = LogisticRegression(penalty='l1', C=0.1)
model.fit(X_train, y_train)

# Coefficients ARE the explanation
for feature, coef in sorted(zip(X.columns, model.coef_[0]), key=lambda x: -abs(x[1])):
    odds_ratio = np.exp(coef)
    print(f"{feature}: coef={coef:.3f}, odds_ratio={odds_ratio:.3f}")

# Option 2: Explainable Boosting Machine (EBM) - best of both worlds
from interpret.glassbox import ExplainableBoostingClassifier
ebm = ExplainableBoostingClassifier(max_bins=256, interactions=10)
ebm.fit(X_train, y_train)
# EBM achieves near-XGBoost performance with full interpretability

# Option 3: Decision rules (RuleFit)
# If income > 50K AND credit_score > 700 AND debt_ratio < 0.3 THEN approve
```

> **Critical Insight:** "Just use SHAP on XGBoost" is NOT the same as interpretability. Post-hoc explanations can be unfaithful (they approximate, not replicate, the model's reasoning). For truly regulated decisions, use inherently interpretable models. The performance gap between interpretable models (EBM, penalized linear) and black-box models is often < 2% — rarely worth the compliance risk.

---

## When Latency or Cost Prohibits

### Latency Constraints

| Use Case | Latency Budget | What You Can Run |
|----------|---------------|------------------|
| Ad bidding (RTB) | <10ms | Linear model, lookup table |
| Search autocomplete | <50ms | Precomputed, prefix trie |
| Fraud at point-of-sale | <100ms | Simple rules + fast model |
| Recommendation page | <200ms | Precomputed candidates + light ranker |
| Email personalization | <1s | More complex models OK |
| Batch risk scoring | Hours | Anything goes |

### Cost Comparison

| Dimension | Rules-Based | ML System |
|-----------|------------|-----------|
| Development | 2 weeks | 8+ weeks |
| Infrastructure | ~$100/month | ~$2000/month |
| Maintenance | 2 hrs/month | 20 hrs/month |
| Expertise needed | Domain expert | ML + Data + Domain engineers |
| Retraining | Never | Weekly/monthly |

### When Cost Doesn't Justify ML

- Marginal accuracy improvement (<2%) over rules
- Problem affects only a small number of decisions
- Business impact of errors is low
- Team doesn't have ML expertise to maintain it
- Data pipeline for features is more expensive than the problem's value

---

## When the Problem Isn't ML-Shaped

### Problems That Aren't ML Problems

| Looks Like ML But Isn't | Why Not | Real Solution |
|------------------------|---------|---------------|
| "Predict the tax owed" | Deterministic formula | Implement tax code directly |
| "Detect if email is from blocked sender" | Exact match problem | Blocklist lookup |
| "Route customer to right team" | Decision tree with <10 nodes | If-else logic |
| "Calculate shipping cost" | Known formula (weight + distance) | Direct computation |
| "Determine price tier" | Business rule (5 tiers) | Switch statement |
| "Identify if user is logged in" | Session state check | Auth middleware |
| "Sort results by date" | Known ordering | ORDER BY clause |

### Signs the Problem Isn't ML-Shaped

1. **Deterministic** — same inputs always give same outputs
2. **Experts can articulate ALL rules** — no "gray area"
3. **Decision space is small** — fewer than 10-20 distinct outcomes
4. **Input-output mapping is known** — just complex to code
5. **No prediction needed** — it's computation, not inference
6. **The "pattern" is actually a policy** — decided by humans, not learned

```python
# NOT an ML problem: Pricing tiers (deterministic business logic)
def calculate_price_tier(user):
    if user.is_enterprise and user.seats > 100: return 'enterprise_custom'
    elif user.is_enterprise: return 'enterprise_standard'
    elif user.seats > 10: return 'team'
    else: return 'individual'

# NOT an ML problem: Notification throttling (simple rule)
def should_send_notification(user, last_notification):
    if (datetime.now() - last_notification).days < 3: return False
    if user.preferences.notifications_disabled: return False
    return True
```

> **Critical Insight:** A common anti-pattern is using ML to learn rules that were already known. If a product manager can describe the logic in a flowchart with fewer than 20 decision points, implement it as code. ML adds value only when the mapping between inputs and outputs is too complex for humans to articulate or when the pattern changes over time.

---

## Heuristics That Beat ML

### Simple Baselines That Are Surprisingly Good

| Domain | Heuristic | Performance |
|--------|-----------|-------------|
| Recommendation | "Most popular items" | Beats many CF models for new users |
| Time series | "Same as last period" (naive forecast) | Hard to beat for volatile series |
| Churn prediction | "Days since last activity" threshold | Often 80% of ML performance |
| Search ranking | TF-IDF + BM25 | Competitive with neural retrieval |
| Anomaly detection | Z-score > 3 standard deviations | Catches most obvious anomalies |
| Spam detection | Keyword blocklist + regex | Catches 90% of spam |
| Lead scoring | Recency + Frequency + Monetary (RFM) | Classic marketing, still strong |
| Image classification | Nearest centroid in feature space | Works for simple cases |

```python
# Heuristic: Recency-based churn prediction
# Often achieves 75-85% of a trained model's performance
def heuristic_churn_score(user):
    """Simple recency-based churn scoring."""
    days_inactive = (datetime.now() - user.last_active).days
    
    if days_inactive > 30:
        return 0.95  # Almost certainly churned
    elif days_inactive > 14:
        return 0.70
    elif days_inactive > 7:
        return 0.40
    elif days_inactive > 3:
        return 0.15
    else:
        return 0.05

# Heuristic: Naive forecast (surprisingly hard to beat)
def naive_forecast(time_series, horizon=7):
    """Last observed value repeated forward."""
    return [time_series[-1]] * horizon

def seasonal_naive_forecast(time_series, season_length=7, horizon=7):
    """Same day last week."""
    return [time_series[-(season_length - i % season_length)] for i in range(horizon)]

# Heuristic: Moving average anomaly detection
def simple_anomaly_detection(values, window=30, threshold=3.0):
    """Flag values more than threshold std devs from rolling mean."""
    rolling_mean = pd.Series(values).rolling(window).mean()
    rolling_std = pd.Series(values).rolling(window).std()
    z_scores = (values - rolling_mean) / rolling_std
    return np.abs(z_scores) > threshold
```

> **Critical Insight:** Always benchmark your ML model against the simplest possible heuristic. If your gradient boosting model achieves 0.87 AUC and a simple "days_since_last_login > 14" threshold achieves 0.82 AUC, that 5-point improvement needs to justify months of development, infrastructure costs, and ongoing maintenance. Often it doesn't.

---

## The Decision Framework

### The 6 Gates

1. **Is the pattern deterministic?** If yes, implement as code.
2. **Do you have 500+ labeled examples?** If no, use rules or collect data.
3. **Does a simple heuristic meet the bar?** If yes, ship the heuristic.
4. **Is ML lift * decisions * value > cost?** If no, ROI doesn't justify it.
5. **Can your team maintain it long-term?** If no, build expertise first.
6. **Are there regulatory constraints?** If yes, use interpretable models only.

### Decision Matrix

| Factor | Favor Rules | Favor ML |
|--------|-------------|----------|
| Data volume | < 500 labeled examples | > 5000 examples |
| Pattern complexity | < 20 decision rules | Cannot articulate rules |
| Pattern stability | Rules never change | Pattern evolves over time |
| Error tolerance | Zero tolerance (safety) | Some errors acceptable |
| Interpretability need | Must explain every decision | Aggregate accuracy sufficient |
| Maintenance budget | Minimal ongoing investment | Dedicated ML team |
| Decision volume | < 100 decisions/day | > 10,000 decisions/day |
| Value per decision | Low (< $1 each) | High (> $10 each) or high volume |

---

## Examples of ML Overkill

### Case Studies

**Case 1: Email Category Detection**
- Company built an NLP classifier (BERT fine-tuned) to route emails
- Categories: billing, technical, cancel, general
- Reality: 85% of emails contained keywords that perfectly identified the category
- Better solution: Regex/keyword matching with ML only for the ambiguous 15%

**Case 2: Dynamic Pricing** — Startup built RL for pricing 50 products. Reality: A/B testing 3 price points and picking the winner would have sufficed.

**Case 3: Employee Attrition** — XGBoost model's top feature was "updated LinkedIn recently" (correlation, not causation). Better: exit interviews + broad retention programs.

**Case 4: "AI-Powered" Loan Calculator** — The "AI" computed a deterministic amortization formula. Just implement the math directly.

### Warning Signs of ML Overkill

1. You can explain the logic in < 5 minutes to a non-technical person
2. The "training data" is actually a lookup table
3. The model implements a policy (not discovers a pattern)
4. The heuristic baseline already meets the performance bar
5. Maintenance cost exceeds the problem's business value

---

## Maintenance Cost of ML Systems

### The Hidden Tax of ML in Production

```
Traditional Software:     ML System:
                         
Code --> Deploy --> Done   Code --> Deploy --> Monitor --> Retrain --> 
                               Validate --> Deploy --> Monitor --> 
                               Debug Drift --> Retrain --> ...
```

### Ongoing Costs

- Data quality monitoring (2-5 hrs/week)
- Model drift detection and retraining (4-8 hrs/month)
- Feature store maintenance (infrastructure team)
- A/B testing (1-2 weeks per experiment)
- Debugging "why did it predict X?" (unpredictable, often urgent)
- Regulatory compliance and fairness audits (quarterly)
- Incident response for model serving failures (on-call)

### Google's ML Technical Debt

From "Hidden Technical Debt in Machine Learning Systems" (Google, 2015): The ML model code is perhaps 5% of the total system. The other 95% is data collection, verification, feature extraction, configuration, serving infrastructure, monitoring, and process management.

> **Critical Insight:** The real cost of ML isn't building the model — it's maintaining it. A model that takes 2 months to build may require 0.5 FTE permanently to maintain (monitoring, retraining, debugging, compliance). Before committing to ML, ask: "Is this problem important enough to warrant a permanent maintenance investment?" If not, use rules.

---

## Interview Talking Points

> "One of my most impactful contributions was convincing the team NOT to build an ML model. For our notification timing feature, I showed that a simple rule — 'send at the same time the user was last active' — achieved 90% of the predicted ML model performance with zero maintenance cost. We shipped in 2 days instead of 2 months."

> "I use a layered approach: rules for clear-cut cases, ML for the ambiguous middle. In our fraud system, rules handled 70% of decisions (obvious fraud and obviously safe), and the ML model only scored the remaining 30%. This reduced model load by 70% and made the system much more debuggable."

> "I always ask three questions before starting an ML project: (1) What would a domain expert do without any model? (2) How much better does ML need to be to justify the investment? (3) Who will maintain this in 2 years? If the answers are unsatisfying, I recommend a simpler approach."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Defaulting to ML for every problem | Ask: "Can rules solve 80% of this?" first |
| Not establishing a baseline heuristic | Always measure simple baseline before building ML |
| Ignoring maintenance cost in project planning | Include 0.5 FTE/year maintenance in ROI calculation |
| Using ML for deterministic problems | If the formula is known, just implement it |
| Building ML when data is insufficient | Wait until you have 10x features samples minimum |
| Choosing black-box model for regulated domain | Use interpretable models or prove compliance |
| Not considering latency at inference time | Benchmark serving latency before choosing architecture |
| Assuming ML will improve over time automatically | Models degrade without active monitoring and retraining |
| Building ML before validating the product | Prove the product works with heuristics first, then optimize |
| Underestimating the team expertise needed | ML systems need data engineers + ML engineers + domain experts |

---

## Rapid-Fire Q&A

**Q1: How do you decide between rules and ML?**
A: If domain experts can articulate the decision logic in < 20 rules, use rules. If the pattern is too complex to describe, changes over time, or requires learning from data, consider ML. Always start with rules as a baseline.

**Q2: What is a heuristic baseline and why does it matter?**
A: A heuristic is a simple, interpretable rule (e.g., "predict most common class" or "last value in time series"). It matters because if your ML model can't significantly beat the heuristic, the added complexity isn't justified.

**Q3: When is insufficient data a blocker for ML?**
A: When you have fewer than 10x samples per feature, fewer than 50 minority-class examples, or when the signal-to-noise ratio is too low for any model to learn reliably. In these cases, expert rules or data collection should come first.

**Q4: Give an example where ML is overkill.**
A: Calculating shipping cost based on weight and distance — it's a known formula. Routing support tickets when 90% can be classified by keywords. Pricing tiers based on company size. Any deterministic mapping.

**Q5: What is the maintenance cost of an ML system?**
A: Typically 0.5-1 FTE permanently for monitoring, retraining, debugging, and compliance. Plus infrastructure costs (feature store, model serving, monitoring tools). The ML code itself is perhaps 5% of the total system.

**Q6: When would you replace an existing ML model with rules?**
A: When the model's top 3-5 features account for 90%+ of predictions, when regulatory requirements change to require full interpretability, when maintenance cost exceeds value, or when the model has decayed and nobody is available to retrain it.

**Q7: What is Google's "Rule #1 of ML"?**
A: "Don't be afraid to launch a product without machine learning." Validate the product with heuristics first, establish data collection, then add ML to optimize. Premature ML optimization is a common startup failure mode.

**Q8: How do you estimate if ML ROI is positive?**
A: Calculate: (ML improvement over baseline) * (number of decisions) * (value per decision) - (development cost + annual maintenance). If this is negative or marginal, don't use ML.

**Q9: What is the role of rules in a production ML system?**
A: Rules serve as guardrails (safety constraints), handle obvious cases (reducing model load), provide explainability (auditable decisions), and act as fallbacks when the model is unavailable or uncertain.

**Q10: How do you communicate to stakeholders that ML isn't needed?**
A: Frame it positively: "We can ship this in 2 days with rules that cover 90% of cases, vs 3 months for ML that covers 95%. Let's launch fast, measure the gap, and add ML only if that 5% matters enough to justify the investment."

---

## ASCII Cheat Sheet

```
+------------------------------------------------------------------+
|                    SHOULD YOU USE ML? FLOWCHART                    |
+------------------------------------------------------------------+
|                                                                    |
|  Is the input->output mapping deterministic/known?                 |
|  +-- YES --> Implement as code. NOT an ML problem.                 |
|  +-- NO  --> Continue                                              |
|                                                                    |
|  Do you have >= 500 labeled examples?                              |
|  +-- NO  --> Use rules/heuristics. Collect data.                   |
|  +-- YES --> Continue                                              |
|                                                                    |
|  Does a simple heuristic meet the performance bar?                 |
|  +-- YES --> Ship the heuristic. Monitor if gap matters.           |
|  +-- NO  --> Continue                                              |
|                                                                    |
|  Is the expected ML lift worth the cost?                           |
|  (lift * decisions * value) > (dev_cost + maintenance)             |
|  +-- NO  --> Use heuristic. Revisit when scale changes.            |
|  +-- YES --> Continue                                              |
|                                                                    |
|  Can your team maintain an ML system long-term?                    |
|  +-- NO  --> Use managed/AutoML service, or hire first.            |
|  +-- YES --> BUILD ML. Start simple, iterate.                      |
|                                                                    |
+------------------------------------------------------------------+
|                    ALTERNATIVES TO ML                              |
+------------------------------------------------------------------+
|                                                                    |
|  Instead of...          Try...                                     |
|  ML classification      Keyword rules + regex                      |
|  ML recommendation      Most popular + recency                     |
|  ML forecasting         Seasonal naive (same day last week)        |
|  ML anomaly detection   Z-score > 3 sigma                          |
|  ML personalization     Segment-based rules (3-5 segments)         |
|  ML pricing             A/B test a few price points                |
|  ML routing             Decision tree with <10 nodes               |
|                                                                    |
+------------------------------------------------------------------+
|                    ML SYSTEM TRUE COST                             |
+------------------------------------------------------------------+
|                                                                    |
|  VISIBLE COSTS:                                                    |
|  - Model development (2-6 months)                                  |
|  - Infrastructure (GPU, storage, serving)                          |
|                                                                    |
|  HIDDEN COSTS (often 3-5x visible):                                |
|  - Data pipeline maintenance                                       |
|  - Model monitoring and drift detection                            |
|  - Retraining cadence (weekly/monthly)                             |
|  - Debugging "why did it predict X?"                               |
|  - Feature store and freshness                                     |
|  - A/B testing infrastructure                                      |
|  - Regulatory compliance and audits                                |
|  - On-call for model-related incidents                             |
|  - Documentation and knowledge transfer                            |
|                                                                    |
|  TOTAL: Plan for 0.5-1 FTE permanent maintenance per model        |
|                                                                    |
+------------------------------------------------------------------+
|                    THE GOLDEN RULE                                 |
+------------------------------------------------------------------+
|                                                                    |
|  "Start with the simplest thing that could possibly work.          |
|   Measure. Only add complexity when simple fails AND the gap       |
|   justifies the investment."                                       |
|                                                                    |
+------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 35 of 45*
