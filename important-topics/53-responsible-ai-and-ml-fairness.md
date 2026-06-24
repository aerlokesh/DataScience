# 🎯 Topic 53: Responsible AI and ML Fairness

> *Building models that are not only accurate but equitable — understanding bias, measuring fairness, and implementing safeguards that ensure AI serves all people, not just the majority.*

---

## Table of Contents

1. [Why Responsible AI Matters](#why-responsible-ai-matters)
2. [Microsoft's Responsible AI Principles](#microsofts-responsible-ai-principles)
3. [Types of Bias in Machine Learning](#types-of-bias-in-machine-learning)
4. [Fairness Metrics](#fairness-metrics)
5. [Impossibility Theorem](#impossibility-theorem)
6. [Bias Detection in Datasets and Models](#bias-detection)
7. [Bias Mitigation Strategies](#bias-mitigation-strategies)
8. [Model Cards and Datasheets](#model-cards-and-datasheets)
9. [Regulatory Context](#regulatory-context)
10. [Practical Implementation with Fairlearn and AIF360](#practical-implementation)
11. [Interview Talking Points](#interview-talking-points)
12. [Common Mistakes](#common-mistakes)
13. [Rapid-Fire Q&A](#rapid-fire-qa)
14. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Why Responsible AI Matters

Responsible AI is the practice of developing, deploying, and maintaining AI systems that are fair, transparent, accountable, and aligned with human values. It is not merely ethical window-dressing — it is a business imperative and increasingly a legal requirement.

**Real-world failures that motivated the field:**

| Case | Year | Issue | Impact |
|------|------|-------|--------|
| Amazon hiring tool | 2018 | Penalized resumes containing "women's" | Scrapped internally |
| COMPAS recidivism | 2016 | Higher false positive rate for Black defendants | Criminal justice reform debate |
| Apple Card | 2019 | Gender disparity in credit limits | Regulatory investigation |
| Healthcare algorithm | 2019 | Used cost as proxy for health needs; Black patients deprioritized | 46% reduction in Black patients identified as needing care |
| Facial recognition | 2018 | Error rate 34.7% for darker-skinned women vs 0.8% for lighter-skinned men | Moratoriums and bans |

> **Critical Insight:** Bias in ML is not always intentional. A model trained on historically biased data will faithfully reproduce and potentially amplify those biases. The model does exactly what you train it to do — which is precisely the problem.

---

## Microsoft's Responsible AI Principles

Microsoft's RAI framework is one of the most comprehensive industry frameworks and commonly referenced in interviews:

| Principle | Definition | Practical Application |
|-----------|-----------|----------------------|
| **Fairness** | AI systems should treat all people fairly | Test for disparate impact across protected groups |
| **Reliability & Safety** | AI should perform reliably and safely | Stress testing, failure modes, monitoring |
| **Privacy & Security** | AI should be secure and respect privacy | Differential privacy, data minimization |
| **Inclusiveness** | AI should empower and engage everyone | Accessibility, diverse representation in training data |
| **Transparency** | AI should be understandable | Model cards, documentation, explainability |
| **Accountability** | People should be accountable for AI | Human oversight, governance processes, audit trails |

> **Critical Insight:** These principles can conflict. Maximizing privacy (by collecting less data) can hurt fairness (by making it impossible to measure disparities). Responsible AI requires *balancing* principles, not maximizing any single one. A mature organization has a framework for resolving these tensions.

**Other notable frameworks:**
- Google's AI Principles (2018)
- OECD AI Principles (2019)
- IEEE Ethically Aligned Design
- Singapore's Model AI Governance Framework
- NIST AI Risk Management Framework (2023)

---

## Types of Bias in Machine Learning

| Bias Type | Definition | Example | Stage |
|-----------|-----------|---------|-------|
| **Historical Bias** | Bias existing in the real world, reflected in data | Past lending discrimination encoded in approval records | Data |
| **Representation Bias** | Underrepresentation of certain groups in training data | Face datasets with 80% lighter skin tones | Data |
| **Measurement Bias** | Features measured differently across groups | Arrest rates as proxy for crime rates (policing bias) | Data |
| **Aggregation Bias** | Single model used for groups with different relationships | Diabetes model that ignores racial differences in biomarkers | Modeling |
| **Learning Bias** | Model amplifies patterns from biased training | Word embeddings associating "nurse" with "female" | Modeling |
| **Evaluation Bias** | Benchmarks not representative of all populations | Accuracy evaluated only on majority group | Evaluation |
| **Deployment Bias** | System used in contexts it was not designed for | Hiring tool trained on one job applied to all roles | Deployment |
| **Feedback Loop Bias** | Model predictions influence future training data | Predictive policing: more police --> more arrests --> "more crime" | Post-deployment |
| **Label Bias** | Ground truth labels reflect human prejudice | Performance reviews rated lower for underrepresented employees | Data |
| **Selection Bias** | Training data not randomly sampled | Survivorship bias: only modeling customers who were approved | Data |

> **Critical Insight:** Removing protected attributes (race, gender) from features does NOT eliminate bias. This "fairness through unawareness" approach fails because other features serve as proxies — zip code correlates with race, name correlates with gender, school correlates with socioeconomic status. You must actively test for disparate impact.

---

## Fairness Metrics

### Core Metrics Comparison Table

| Metric | Definition | Formula | Intuition |
|--------|-----------|---------|-----------|
| **Demographic Parity** (Statistical Parity) | P(Y_hat=1 \| A=0) = P(Y_hat=1 \| A=1) | Selection rates equal across groups | Equal acceptance rates regardless of group |
| **Equalized Odds** | P(Y_hat=1 \| Y=1, A=0) = P(Y_hat=1 \| Y=1, A=1) AND same for Y=0 | TPR and FPR equal across groups | Equal error rates across groups |
| **Equal Opportunity** | P(Y_hat=1 \| Y=1, A=0) = P(Y_hat=1 \| Y=1, A=1) | TPR equal across groups | Qualified individuals equally likely to be selected |
| **Predictive Parity** | P(Y=1 \| Y_hat=1, A=0) = P(Y=1 \| Y_hat=1, A=1) | Precision equal across groups | Positive predictions equally reliable across groups |
| **Calibration** | P(Y=1 \| score=s, A=0) = P(Y=1 \| score=s, A=1) | Same score means same probability across groups | Scores mean the same thing for everyone |
| **Treatment Equality** | FN/FP ratio equal across groups | Error ratio balance | Same ratio of error types across groups |

### Choosing the Right Metric

| Context | Recommended Metric | Reasoning |
|---------|--------------------|-----------|
| Hiring/admissions | Equal Opportunity or Demographic Parity | Qualified candidates should have equal chances |
| Criminal justice (bail) | Equalized Odds | Both error types (false arrest, false release) matter |
| Lending decisions | Predictive Parity + Calibration | Score should mean same risk regardless of group |
| Healthcare screening | Equal Opportunity (TPR parity) | Sick patients must be equally likely to be detected |
| Advertising (benign) | Demographic Parity (loose) | Equal exposure to opportunities |

> **Critical Insight:** The choice of fairness metric is a *normative* decision, not a technical one. Different metrics encode different values about what "fair" means. A data scientist's job is to (1) articulate the tradeoffs clearly, (2) present options to decision-makers, and (3) implement the chosen standard rigorously. Never pick a metric just because it is easy to satisfy.

---

## Impossibility Theorem

**Choquet's Impossibility (Kleinberg-Mullainathan-Raghavan, 2016):**

It is mathematically impossible to simultaneously satisfy all of:
1. Calibration (within groups)
2. False positive rate parity (across groups)
3. False negative rate parity (across groups)

...unless the base rates are identical across groups OR the model is perfect.

```
If P(Y=1|A=0) != P(Y=1|A=1)    [different base rates]
Then: Calibration + Equalized Odds = IMPOSSIBLE

You MUST choose which fairness guarantee matters most.
```

> **Critical Insight:** This is perhaps the most important theoretical result in ML fairness. It means you cannot have a model that is simultaneously "fair" by all definitions. This is why fairness is a conversation about values, not just a technical optimization problem. Be prepared to explain this tradeoff in interviews.

---

## Bias Detection

### In Datasets

```python
import pandas as pd

def dataset_bias_audit(df, target_col, protected_col):
    """Quick bias audit: representation, base rates, disparate impact."""
    # Representation and base rates by group
    repr_pct = df[protected_col].value_counts(normalize=True)
    base_rates = df.groupby(protected_col)[target_col].mean()
    # Disparate impact ratio (<0.8 = concern per four-fifths rule)
    di_ratio = base_rates.min() / base_rates.max()

    return {
        'representation': repr_pct.to_dict(),
        'base_rates': base_rates.to_dict(),
        'disparate_impact_ratio': di_ratio
    }
```

### In Model Predictions

```python
from sklearn.metrics import confusion_matrix

def model_fairness_audit(y_true, y_pred, sensitive_attribute):
    """Compute fairness metrics across groups."""
    metrics = {}
    for group in sensitive_attribute.unique():
        mask = sensitive_attribute == group
        tn, fp, fn, tp = confusion_matrix(y_true[mask], y_pred[mask]).ravel()
        metrics[group] = {
            'selection_rate': y_pred[mask].mean(),
            'tpr': tp / (tp + fn) if (tp + fn) > 0 else 0,
            'fpr': fp / (fp + tn) if (fp + tn) > 0 else 0,
            'precision': tp / (tp + fp) if (tp + fp) > 0 else 0,
            'base_rate': y_true[mask].mean()
        }
    # Compute disparities (for binary sensitive attribute)
    groups = list(metrics.keys())
    if len(groups) == 2:
        g0, g1 = groups
        metrics['disparities'] = {
            'dp_diff': abs(metrics[g0]['selection_rate'] - metrics[g1]['selection_rate']),
            'eq_odds_tpr': abs(metrics[g0]['tpr'] - metrics[g1]['tpr']),
            'eq_odds_fpr': abs(metrics[g0]['fpr'] - metrics[g1]['fpr']),
            'disparate_impact': min(metrics[g0]['selection_rate'],
                metrics[g1]['selection_rate']) / max(metrics[g0]['selection_rate'],
                metrics[g1]['selection_rate'])
        }
    return metrics
```

---

## Bias Mitigation Strategies

### Overview by Stage

| Stage | Strategy | Method | Tradeoff |
|-------|----------|--------|----------|
| **Pre-processing** | Reweighting | Assign higher weights to underrepresented group-label combos | May reduce overall accuracy |
| **Pre-processing** | Resampling | Oversample minority, undersample majority per group | Can introduce noise or lose data |
| **Pre-processing** | Disparate Impact Remover | Transform features to remove correlation with protected attr | Loses some signal |
| **Pre-processing** | Learning Fair Representations | Learn latent representation that is independent of protected attr | Complex, may not generalize |
| **In-processing** | Fairness Constraints | Add fairness penalty to loss function | Accuracy-fairness tradeoff |
| **In-processing** | Adversarial Debiasing | Train adversary to predict protected attr from predictions; penalize | Harder to train, unstable |
| **In-processing** | Exponentiated Gradient | Reduce fair classification to cost-sensitive classification | Well-studied, good guarantees |
| **Post-processing** | Threshold Adjustment | Use different classification thresholds per group | Simple, effective; requires group labels at inference |
| **Post-processing** | Reject Option Classification | Defer uncertain predictions near boundary | Reduces automation rate |
| **Post-processing** | Calibrated Equalized Odds | Adjust predictions to equalize FPR and TPR | Small accuracy cost |

> **Critical Insight:** Post-processing is often the most practical starting point because it does not require retraining the model and can be applied as a policy layer. However, some jurisdictions prohibit using different thresholds per group (treating people differently based on protected characteristics). Know your legal context before implementing.

### Pre-processing: Reweighting Example

```python
def compute_reweighting(df, target_col, protected_col):
    """Compute sample weights to achieve demographic parity in labels."""
    total = len(df)
    weights = pd.Series(index=df.index, dtype=float)

    for group in df[protected_col].unique():
        for label in df[target_col].unique():
            mask = (df[protected_col] == group) & (df[target_col] == label)
            n_group = (df[protected_col] == group).sum()
            n_label = (df[target_col] == label).sum()
            n_both = mask.sum()

            # Expected proportion under independence
            expected = (n_group / total) * (n_label / total) * total
            weight = expected / n_both if n_both > 0 else 1.0
            weights[mask] = weight

    return weights
```

### In-processing: Fairness Constraints with Fairlearn

```python
from fairlearn.reductions import ExponentiatedGradient, DemographicParity
from sklearn.linear_model import LogisticRegression

# Base estimator
base_model = LogisticRegression(max_iter=1000)

# Constrained model: optimize accuracy subject to demographic parity
mitigator = ExponentiatedGradient(
    estimator=base_model,
    constraints=DemographicParity(),
    eps=0.01  # fairness tolerance
)

mitigator.fit(X_train, y_train, sensitive_features=A_train)
y_pred_fair = mitigator.predict(X_test)
```

### Post-processing: Threshold Adjustment

```python
from fairlearn.postprocessing import ThresholdOptimizer

# Optimize thresholds to satisfy equalized odds
postprocessor = ThresholdOptimizer(
    estimator=trained_model,
    constraints="equalized_odds",
    objective="accuracy_score",
    prefit=True
)

postprocessor.fit(X_val, y_val, sensitive_features=A_val)
y_pred_fair = postprocessor.predict(X_test, sensitive_features=A_test)
```

---

## Model Cards and Datasheets

### Model Cards (Mitchell et al., 2019)

A standardized document that accompanies a trained model:

| Section | Contents |
|---------|----------|
| Model Details | Architecture, training algorithm, version |
| Intended Use | Primary use cases and out-of-scope uses |
| Factors | Groups, instrumentation, environment |
| Metrics | Performance measures with confidence intervals |
| Evaluation Data | Datasets, motivation, preprocessing |
| Training Data | Same as above for training |
| Ethical Considerations | Sensitive data, known limitations |
| Caveats and Recommendations | Known failure modes, guidance |

### Datasheets for Datasets (Gebru et al., 2021)

| Section | Key Questions |
|---------|--------------|
| Motivation | Why was this dataset created? Who funded it? |
| Composition | What does each instance represent? Is it a sample? |
| Collection | How was data acquired? Who collected it? |
| Preprocessing | What cleaning was done? |
| Uses | What tasks has it been used for? What should it NOT be used for? |
| Distribution | How is it distributed? Under what license? |
| Maintenance | Who maintains it? How can errors be reported? |

> **Critical Insight:** Model cards and datasheets are not just bureaucratic exercises — they force you to think about failure modes, misuse potential, and affected populations *before* deployment. Companies that skip documentation often discover bias only after public embarrassment. Treat documentation as a first-class engineering artifact.

---

## Regulatory Context

| Regulation | Jurisdiction | Key Requirements for AI/ML |
|-----------|-------------|---------------------------|
| **EU AI Act** (2024) | EU | Risk-based classification; high-risk AI requires conformity assessment, documentation, human oversight |
| **GDPR Article 22** | EU | Right not to be subject to solely automated decisions; right to explanation |
| **ECOA / FCRA** | US | Must provide specific adverse action reasons for credit denials |
| **NYC Local Law 144** | US (NYC) | Bias audits required for automated employment decision tools |
| **Colorado AI Act** | US (CO) | Algorithmic impact assessments for high-risk AI |
| **EEOC Guidelines** | US | Four-fifths rule for disparate impact in employment |
| **Canada AIDA** | Canada | Proposed: obligations for high-impact AI systems |

**The Four-Fifths Rule (US employment law):**
```
If selection_rate(minority_group) / selection_rate(majority_group) < 0.8
--> Prima facie evidence of disparate impact
```

> **Critical Insight:** The EU AI Act classifies AI systems into four risk levels: Unacceptable (banned), High-risk (heavy regulation), Limited risk (transparency), Minimal risk (no rules). Most hiring, lending, and healthcare AI falls under "high-risk" and requires conformity assessments, technical documentation, human oversight mechanisms, and ongoing monitoring. Know where your models sit in this framework.

---

## Practical Implementation

### Fairness Pipeline with Fairlearn

```python
import pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score
from sklearn.datasets import fetch_openml
from fairlearn.metrics import MetricFrame, demographic_parity_difference, selection_rate
from fairlearn.reductions import ExponentiatedGradient, EqualizedOdds

# Load Adult Income dataset
data = fetch_openml(data_id=1590, as_frame=True)
X, y = data.data, (data.target == '>50K').astype(int)
sensitive = X['sex']
X_encoded = pd.get_dummies(X.drop(columns=['sex']), drop_first=True)

X_train, X_test, y_train, y_test, A_train, A_test = train_test_split(
    X_encoded, y, sensitive, test_size=0.3, random_state=42, stratify=y
)

# Train unconstrained model and evaluate fairness
model = GradientBoostingClassifier(n_estimators=100, random_state=42)
model.fit(X_train, y_train)
y_pred = model.predict(X_test)

metric_frame = MetricFrame(
    metrics={'accuracy': accuracy_score, 'selection_rate': selection_rate},
    y_true=y_test, y_pred=y_pred, sensitive_features=A_test
)
print(metric_frame.by_group)
print(f"Demographic Parity Diff: "
      f"{demographic_parity_difference(y_test, y_pred, sensitive_features=A_test):.4f}")

# Mitigate with ExponentiatedGradient (equalized odds constraint)
mitigator = ExponentiatedGradient(
    estimator=LogisticRegression(max_iter=1000),
    constraints=EqualizedOdds(), eps=0.01
)
mitigator.fit(X_train, y_train, sensitive_features=A_train)
y_pred_fair = mitigator.predict(X_test)

# Compare: accuracy tradeoff
print(f"Accuracy: {accuracy_score(y_test, y_pred):.4f} -> "
      f"{accuracy_score(y_test, y_pred_fair):.4f}")
print(f"DP Diff:  {demographic_parity_difference(y_test, y_pred, sensitive_features=A_test):.4f} -> "
      f"{demographic_parity_difference(y_test, y_pred_fair, sensitive_features=A_test):.4f}")
```

### Using AIF360 for Bias Detection

```python
from aif360.datasets import BinaryLabelDataset
from aif360.metrics import BinaryLabelDatasetMetric
from aif360.algorithms.preprocessing import Reweighing

# Create AIF360 dataset and measure bias
dataset = BinaryLabelDataset(
    df=train_df, label_names=['income'],
    protected_attribute_names=['sex'], favorable_label=1, unfavorable_label=0
)
metric = BinaryLabelDatasetMetric(
    dataset, unprivileged_groups=[{'sex': 0}], privileged_groups=[{'sex': 1}]
)
print(f"Disparate Impact: {metric.disparate_impact():.4f}")

# Apply reweighting and retrain
reweigher = Reweighing(
    unprivileged_groups=[{'sex': 0}], privileged_groups=[{'sex': 1}]
)
dataset_reweighed = reweigher.fit_transform(dataset)
model.fit(X_train, y_train, sample_weight=dataset_reweighed.instance_weights)
```

---

## Interview Talking Points

> **"How would you check if your model is fair?"**
>
> "I follow a systematic approach. First, I define *who* we are checking fairness for — which groups and which protected attributes. Then I choose the appropriate fairness metric based on the context — for hiring, I would use equal opportunity (equal TPR across groups); for lending, calibration plus disparate impact ratio. I compute these metrics on a held-out test set, broken down by group. I look for the four-fifths rule as a starting threshold: if any group's selection rate is below 80% of the highest group's rate, that is a red flag. Beyond aggregate metrics, I examine confusion matrices per group, calibration curves per group, and SHAP value distributions per group to understand *why* disparities exist. Critically, I do this analysis *before* deployment, not after."

> **"What would you do if you found your model discriminates?"**
>
> "I would follow a structured remediation process. First, *diagnose the source*: Is it a data problem (representation, labeling) or a modeling problem (proxy features, aggregation bias)? This determines the fix. If it is representation bias, I would explore resampling or reweighting. If proxy features are the issue, I would consider removing them or using adversarial debiasing. For quick wins, post-processing threshold adjustment is effective — different thresholds per group to equalize TPR/FPR. However, I would also escalate to stakeholders: fairness is not purely a technical decision. I would present the accuracy-fairness tradeoff clearly — 'We can reduce the disparity from 12% to 2% with a 1.5% accuracy drop' — and let decision-makers choose. Finally, I would implement ongoing monitoring because fairness can degrade over time as populations shift."

> **"Can you remove bias by just removing protected attributes from features?"**
>
> "No — this is called 'fairness through unawareness' and it is a common misconception. Protected attributes are correlated with many other features through historical and structural factors. Zip code correlates with race, name correlates with gender, occupation correlates with both. The model will simply learn these proxies. In fact, removing protected attributes can make things *worse* because you can no longer even measure whether the model is biased — you have lost the ability to audit. The correct approach is to *keep* protected attributes for measurement and auditing purposes, while using mitigation techniques (constraints, reweighting, threshold adjustment) to ensure equitable outcomes."

> **"How do you balance accuracy and fairness?"**
>
> "The impossibility theorem tells us we cannot satisfy all fairness criteria simultaneously when base rates differ across groups. So the question is really about acceptable tradeoffs. I frame this as a Pareto frontier: I train multiple models with varying fairness constraints and present the accuracy-fairness curve to stakeholders. Often the tradeoff is small — in my experience, you can reduce disparities by 80-90% with only a 1-3% accuracy loss. But the choice of *how much* accuracy to sacrifice for *how much* fairness is a business and ethical decision, not a technical one. I provide the analysis and options; leadership decides the operating point."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Removing protected attributes to "ensure fairness" | Keep them for auditing; use mitigation techniques instead |
| Using only one fairness metric | Report multiple metrics; acknowledge impossibility theorem tradeoffs |
| Testing fairness only on aggregate test set | Slice by group, intersectionally (e.g., Black women, not just Black + women) |
| Assuming equal base rates across groups | Check base rates first; unequal rates change which metrics are achievable |
| Applying US regulatory frameworks globally | Fairness definitions and legal requirements vary by jurisdiction |
| Treating fairness as a one-time check | Implement ongoing monitoring; distributions and bias shift over time |
| Optimizing for a fairness metric without understanding its meaning | Choose metric based on normative values appropriate to the context |
| Ignoring intersectionality | A model can be fair across gender AND race but unfair for specific intersections |
| Assuming correlation with protected attribute means discrimination | Some correlations are legitimate (insurance actuarial tables in some jurisdictions) |
| Deferring all fairness decisions to "the ethics team" | Every ML practitioner shares responsibility; fairness is part of model development |

---

## Rapid-Fire Q&A

**Q1: What is the four-fifths rule?**
A: If the selection rate for a protected group is less than 80% (four-fifths) of the rate for the group with the highest rate, there is prima facie evidence of adverse/disparate impact (US employment law).

**Q2: Can a model satisfy both demographic parity and equalized odds?**
A: Only if base rates are identical across groups or the model is perfect. Otherwise, the impossibility theorem (Kleinberg et al., 2016) says no.

**Q3: What is the difference between disparate treatment and disparate impact?**
A: Disparate treatment = intentionally using protected attributes (explicit discrimination). Disparate impact = facially neutral policy that disproportionately affects a protected group (structural discrimination).

**Q4: What is "fairness gerrymandering"?**
A: A model that appears fair on each protected attribute individually but is unfair for intersectional subgroups (e.g., fair for women and fair for Black individuals, but unfair for Black women).

**Q5: What does the EU AI Act classify as "high-risk" AI?**
A: AI used in critical infrastructure, education, employment, essential services, law enforcement, migration, and administration of justice/democratic processes.

**Q6: What is differential privacy and how does it relate to fairness?**
A: Differential privacy adds noise to protect individual data. It can worsen fairness because noise disproportionately affects smaller groups (less data to average over), creating a privacy-fairness tension.

**Q7: What is a proxy variable in the context of bias?**
A: A feature that is correlated with a protected attribute and allows the model to indirectly discriminate. Example: zip code as proxy for race, first name as proxy for gender/ethnicity.

**Q8: How does feedback loop bias work?**
A: Model predictions influence real-world outcomes which become future training data. Example: predictive policing sends more officers to an area, leading to more arrests, "confirming" the prediction.

**Q9: What is the difference between individual and group fairness?**
A: Group fairness: statistical metrics hold across groups. Individual fairness: similar individuals receive similar predictions (Lipschitz condition on model output with respect to a similarity metric).

**Q10: What is an algorithmic impact assessment?**
A: A structured evaluation of an AI system's potential effects on individuals and communities, including risk identification, bias testing, and mitigation planning — increasingly required by regulation (EU AI Act, Colorado AI Act).

---

## ASCII Cheat Sheet

```
+============================================================================+
|              RESPONSIBLE AI DECISION FRAMEWORK                               |
+============================================================================+

  DATA --> MODEL --> DEPLOY --> MONITOR
   |        |         |          |
   v        v         v          v
  Check    Choose    Model      Track metrics
  repr.    metric    card       by group
  Audit    Apply     Human      Detect drift
  labels   constr.   oversight   Watch feedback
  Doc in   Test      Appeal      loops
  sheet    intersect. process

+============================================================================+
|                    FAIRNESS METRICS AT A GLANCE                              |
+============================================================================+

  Metric               What must be equal?      Group A    Group B
  ─────────────────────────────────────────────────────────────────
  Demographic Parity   P(Y_hat=1)               30%   =   30%
  Equal Opportunity    P(Y_hat=1 | Y=1) [TPR]   85%   =   85%
  Equalized Odds       TPR AND FPR              85%/10% = 85%/10%
  Predictive Parity    P(Y=1 | Y_hat=1) [PPV]   70%   =   70%
  Calibration          P(Y=1 | score=s)         score means same thing

+============================================================================+
|                    IMPOSSIBILITY THEOREM                                     |
+============================================================================+

  If base rates differ:  P(Y=1|A=0) = 20%  vs  P(Y=1|A=1) = 40%

  Then you CANNOT have all three:
    [Calibration] + [FPR Parity] + [FNR Parity] = IMPOSSIBLE

  Pick your priority:
    ├── Calibration + Equal Opp?  --> FPR will differ
    ├── Equalized Odds?           --> Calibration will differ
    └── Demographic Parity?       --> Neither calibration nor eq. odds

+============================================================================+
|                    BIAS MITIGATION PIPELINE                                  |
+============================================================================+

  [Raw Data] --> PRE-PROCESSING --> [Debiased Data]
                 (Reweighting, Resampling, Fair Representations)
                        |
                        v
              IN-PROCESSING --> [Trained Model]
              (Constraints, Adversarial, Exponentiated Gradient)
                        |
                        v
              POST-PROCESSING --> [Fair Predictions]
              (Threshold Adjust, Reject Option, Calibrated Eq. Odds)

  Rule of thumb: POST first (quick) --> IN (best tradeoff) --> PRE (root cause)

+============================================================================+
|                    FOUR-FIFTHS RULE QUICK CHECK                              |
+============================================================================+

  Group A rate: 60%  |  Group B rate: 40%
  Ratio = 40/60 = 0.667  <  0.8  --> ADVERSE IMPACT DETECTED

+============================================================================+
```

*Part of the [Data Science Interview Topics](README.md) collection — Topic 53 of 53*
