# 🎯 Topic 26: Causal Inference and Confounding

> *"Correlation is not causation — but what IS causation, and how do we identify it when randomization isn't possible?"*

---

## Table of Contents

1. [Correlation vs. Causation](#correlation-vs-causation)
2. [Confounders and Bias](#confounders)
3. [Directed Acyclic Graphs (DAGs)](#dags)
4. [Backdoor Criterion](#backdoor-criterion)
5. [Instrumental Variables](#instrumental-variables)
6. [Propensity Score Matching](#propensity-score-matching)
7. [Regression Discontinuity](#regression-discontinuity)
8. [Natural Experiments](#natural-experiments)
9. [When RCTs Aren't Possible](#when-rcts-impossible)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#cheat-sheet)

---

## Correlation vs. Causation {#correlation-vs-causation}

### Three Explanations for Correlation

When X and Y are correlated, there are three possible causal structures:

```
1. X CAUSES Y (direct causation)
   X ──→ Y
   Example: Smoking → Lung cancer

2. Y CAUSES X (reverse causation)
   X ←── Y
   Example: Fire trucks at scene ← Fire (trucks don't cause fires)

3. Z CAUSES BOTH (confounding)
   X ←── Z ──→ Y
   Example: Ice cream sales & drownings ← Summer heat
```

### The Fundamental Problem of Causal Inference

```
For individual i:
  - Factual outcome:       Y_i(T=1) if treated, Y_i(T=0) if not
  - Counterfactual outcome: Y_i(T=0) if treated, Y_i(T=1) if not
  
  Individual Treatment Effect = Y_i(1) - Y_i(0)
  
  PROBLEM: We can NEVER observe both for the same individual!
  
  Solution: Average Treatment Effect (ATE)
  ATE = E[Y(1) - Y(0)] = E[Y(1)] - E[Y(0)]
  
  RCTs solve this by making E[Y|T=1] = E[Y(1)] via randomization
```

> **Critical Insight** 💡
> The reason RCTs are the "gold standard" is simple: random assignment makes treatment INDEPENDENT of all confounders (observed AND unobserved). Every observational method is essentially trying to approximate what randomization gives you for free — but none can guarantee control over unobserved confounders.

---

## Confounders and Bias {#confounders}

### What Is a Confounder?

A confounder Z is a variable that:
1. Causes (or is associated with) the treatment X
2. Causes (or is associated with) the outcome Y
3. Is NOT on the causal path from X to Y

```
       Z (confounder)
      / \
     ↓   ↓
     X    Y
     
If we ignore Z:
  E[Y|X=1] - E[Y|X=0] ≠ E[Y(1)] - E[Y(0)]
  (Observed association ≠ Causal effect)
```

### Types of Bias in Observational Studies

| Bias Type | Mechanism | Example |
|-----------|-----------|---------|
| Confounding bias | Common cause of treatment and outcome | Sicker patients get more treatment AND die more |
| Selection bias | Conditioning on a collider | Analyzing only hospitalized patients |
| Measurement bias | Differential measurement | Treatment group measured more carefully |
| Survivorship bias | Conditioning on survival | Only studying successful companies |
| Attrition bias | Differential dropout | Sicker patients leave study |
| Time-varying confounding | Confounders change over time | Past outcomes affect future treatment |

### Observed vs. Unobserved Confounders

```
OBSERVED confounders:
  → Can control via: regression, matching, stratification, weighting
  → Assumption: "conditional ignorability" / "no unmeasured confounding"

UNOBSERVED confounders:
  → Cannot control directly
  → Need: instrumental variables, diff-in-diff, regression discontinuity
  → Or: sensitivity analysis (how strong would unobserved confounder need to be?)
```

---

## Directed Acyclic Graphs (DAGs) {#dags}

### What Are DAGs?

DAGs are graphical representations of causal assumptions. Nodes = variables, directed edges = causal relationships.

```
BASIC DAG ELEMENTS:

Chain (mediation):     X → M → Y
Fork (confounding):    X ← Z → Y  
Collider:             X → C ← Y
```

### The Three Fundamental Structures

| Structure | Diagram | Unconditional | Conditional on middle |
|-----------|---------|--------------|----------------------|
| Chain | X → M → Y | X and Y associated | X and Y INDEPENDENT |
| Fork | X ← Z → Y | X and Y associated | X and Y INDEPENDENT |
| Collider | X → C ← Y | X and Y INDEPENDENT | X and Y ASSOCIATED |

> **Critical Insight** 💡
> The collider structure is counter-intuitive and critically important. Conditioning on a collider (or its descendant) CREATES a spurious association between its causes. This is "selection bias" or "Berkson's paradox." Example: conditioning on "admitted to hospital" creates a negative association between two independent diseases.

### Example DAG: Does Education Cause Higher Income?

```
        Ability
       /       \
      ↓         ↓
 Education → Income ← Experience
      ↑                    ↑
      └── Family SES ──────┘
      
Confounders of Education → Income:
  • Ability (causes both more education AND higher income)
  • Family SES (causes both education access AND income opportunities)
  
To identify causal effect of Education on Income:
  → Must block backdoor paths through Ability and Family SES
  → Either condition on them (if measurable) or find an instrument
```

### DAG Rules (d-separation)

```
A path is BLOCKED if:
  1. It contains a chain X → M → Y and you condition on M
  2. It contains a fork X ← Z → Y and you condition on Z
  3. It contains a collider X → C ← Y and you DON'T condition on C
     (or any descendant of C)

Two variables are d-separated if ALL paths between them are blocked.
d-separation → conditional independence (under faithfulness)
```

---

## Backdoor Criterion {#backdoor-criterion}

### Definition

A set of variables Z satisfies the backdoor criterion relative to (X, Y) if:
1. No node in Z is a descendant of X
2. Z blocks every path between X and Y that contains an arrow INTO X

### Why It Matters

If Z satisfies the backdoor criterion, then:
```
P(Y | do(X)) = Σ_z P(Y | X, Z=z) × P(Z=z)

This means: conditioning on Z gives the causal effect!
```

### Example Application

```
DAG:
    Z₁ → X → Y
    Z₁ → Y
    Z₂ → Y
    X → M → Y

Q: What do we need to condition on to get causal effect of X on Y?

Backdoor paths from X to Y:
  X ← Z₁ → Y  (the fork through Z₁)

Sufficient adjustment sets:
  • {Z₁}  ← blocks the backdoor path
  
DO NOT condition on:
  • M (mediator — would block part of the causal effect)
  • Z₂ (not a confounder — doesn't cause X)
```

### Practical Steps for Using DAGs

```python
# Using the dowhy/pgmpy library (conceptual)
# Step 1: Draw your DAG based on domain knowledge
# Step 2: Identify all backdoor paths
# Step 3: Find minimal adjustment set
# Step 4: Verify no colliders are being conditioned on
# Step 5: Condition on the adjustment set in your regression/matching

# Pseudocode for causal effect estimation
import dowhy
from dowhy import CausalModel

model = CausalModel(
    data=df,
    treatment='education',
    outcome='income',
    graph="""
    digraph {
        ability -> education;
        ability -> income;
        family_ses -> education;
        family_ses -> income;
        education -> income;
        experience -> income;
    }
    """
)

# Identify causal effect
identified_estimand = model.identify_effect()
# Estimate using backdoor adjustment
estimate = model.estimate_effect(
    identified_estimand,
    method_name="backdoor.linear_regression"
)
```

---

## Instrumental Variables {#instrumental-variables}

### The IV Idea

When you have UNOBSERVED confounders, find a variable Z (the "instrument") that:
1. Causes variation in the treatment (relevance)
2. Affects the outcome ONLY through the treatment (exclusion restriction)
3. Is not confounded with the outcome (independence)

```
Unobserved       
Confounder (U)   
    ↓       ↓    
    X   →   Y    
    ↑             
    Z (instrument)
    
Z → X → Y  (the causal path we exploit)
Z ⊥ U       (instrument is independent of confounders)
Z → Y only through X (exclusion restriction)
```

### Classic Examples

| Setting | Instrument | Treatment | Outcome |
|---------|-----------|-----------|---------|
| Returns to education | Quarter of birth | Years of schooling | Wages |
| Effect of military service | Draft lottery number | Service | Lifetime earnings |
| Effect of family size | Twin births | Number of children | Mother's labor supply |
| Price elasticity | Weather shocks | Supply/Price | Demand |

### Two-Stage Least Squares (2SLS)

```
Stage 1: X_hat = α + γ × Z + ε₁    (predict treatment from instrument)
Stage 2: Y = β₀ + β₁ × X_hat + ε₂  (use predicted treatment to estimate effect)

The β₁ from Stage 2 is the IV estimate of the causal effect.
```

```python
from linearmodels.iv import IV2SLS
import pandas as pd

# Example: Effect of education on income
# Instrument: distance to nearest college
model = IV2SLS.from_formula(
    'income ~ 1 + experience + [education ~ distance_to_college]',
    data=df
)
results = model.fit()
print(results.summary)
```

### IV Assumptions and Pitfalls

| Assumption | What It Means | How to Check |
|-----------|---------------|-------------|
| Relevance | Z predicts X | First-stage F-statistic > 10 |
| Exclusion | Z → Y only through X | Cannot test! Domain knowledge |
| Independence | Z ⊥ U | Cannot test fully; argue from design |
| Monotonicity | Z affects X in same direction for all | Needed for LATE interpretation |

> **Critical Insight** 💡
> IV gives you the Local Average Treatment Effect (LATE) — the causal effect for "compliers" (those whose treatment status is actually affected by the instrument). This is NOT the ATE for the entire population. A draft lottery IV tells you the effect of military service for those who served BECAUSE of the draft, not for volunteers.

---

## Propensity Score Matching {#propensity-score-matching}

### The Propensity Score

```
e(X) = P(T=1 | X) = probability of receiving treatment given covariates

Key Theorem (Rosenbaum & Rubin, 1983):
  If (Y(0), Y(1)) ⊥ T | X    (ignorability given X)
  Then (Y(0), Y(1)) ⊥ T | e(X)  (ignorability given propensity score)

→ Instead of matching on ALL covariates, match on ONE scalar: e(X)
```

### PSM Step by Step

```python
from sklearn.linear_model import LogisticRegression
from sklearn.neighbors import NearestNeighbors
import numpy as np
import pandas as pd

# Step 1: Estimate propensity scores
model = LogisticRegression()
X = df[['age', 'income', 'education', 'prior_purchases']]
T = df['received_treatment']
model.fit(X, T)
df['propensity_score'] = model.predict_proba(X)[:, 1]

# Step 2: Check overlap (common support)
# Treatment and control propensity score distributions should overlap
treated = df[df['received_treatment'] == 1]['propensity_score']
control = df[df['received_treatment'] == 0]['propensity_score']
# Trim: remove units outside common support
min_ps = max(treated.min(), control.min())
max_ps = min(treated.max(), control.max())
df_trimmed = df[(df['propensity_score'] >= min_ps) & 
                (df['propensity_score'] <= max_ps)]

# Step 3: Match treated to nearest control
treated_ps = df_trimmed[df_trimmed['received_treatment']==1]['propensity_score'].values
control_ps = df_trimmed[df_trimmed['received_treatment']==0]['propensity_score'].values

nn = NearestNeighbors(n_neighbors=1)
nn.fit(control_ps.reshape(-1, 1))
distances, indices = nn.kneighbors(treated_ps.reshape(-1, 1))

# Step 4: Estimate ATT (Average Treatment Effect on Treated)
treated_outcomes = df_trimmed[df_trimmed['received_treatment']==1]['outcome'].values
matched_control_outcomes = df_trimmed[df_trimmed['received_treatment']==0]['outcome'].values[indices.flatten()]
att = np.mean(treated_outcomes - matched_control_outcomes)
print(f"ATT estimate: {att:.4f}")

# Step 5: Check balance (covariates should be balanced after matching)
# Standardized mean differences should be < 0.1 for all covariates
```

### PSM vs. Other Methods

| Method | Handles Unobserved Confounders? | Assumption |
|--------|-------------------------------|-----------|
| PSM | NO | No unobserved confounders |
| IPW (weighting) | NO | No unobserved confounders |
| IV | YES | Valid instrument exists |
| Diff-in-Diff | Partially | Parallel trends |
| RDD | YES (locally) | Continuity at threshold |
| RCT | YES | Proper randomization |

> **Critical Insight** 💡
> PSM's fatal weakness: it ONLY controls for OBSERVED confounders. If there's an unobserved variable that affects both treatment assignment and outcome, PSM will give biased results — and you won't know it. Always perform sensitivity analysis (e.g., Rosenbaum bounds) to assess how robust your findings are to potential hidden bias.

---

## Regression Discontinuity {#regression-discontinuity}

### The Core Idea

When treatment is assigned based on whether a "running variable" crosses a threshold, units just above and just below the threshold are essentially randomized.

```
Treatment assignment:  T = 1 if Score ≥ c, T = 0 if Score < c

Near the cutoff c:
  • Units at c - ε and c + ε are nearly identical
  • Only difference: one got treatment, other didn't
  • → Quasi-random assignment AT THE CUTOFF
```

### Visual Intuition

```
Outcome
    │         ●  ●
    │        ●  ●     Treatment group
    │       ●  ●
    │      ● ●
    │     ●│← JUMP (causal effect)
    │    ● │
    │   ●  │  Control group
    │  ●   │
    │ ●    │
    │●     │
    └──────┼──────────→ Running Variable (Score)
           c (cutoff)
```

### Sharp vs. Fuzzy RDD

| Type | Rule | Example | Estimation |
|------|------|---------|-----------|
| Sharp | T = 1(Score ≥ c) perfectly | Scholarship if GPA ≥ 3.5 | Local regression at cutoff |
| Fuzzy | P(T=1) jumps at c but ≠ 100% | Eligibility ≠ enrollment | IV with cutoff as instrument |

```python
# Sharp RDD estimation using local linear regression
import numpy as np
from sklearn.linear_model import LinearRegression

# Bandwidth selection (simplified — use rdrobust in practice)
bandwidth = 0.5  # units of running variable
cutoff = 3.5

# Select observations near cutoff
mask = (df['score'] >= cutoff - bandwidth) & (df['score'] <= cutoff + bandwidth)
df_local = df[mask].copy()
df_local['above'] = (df_local['score'] >= cutoff).astype(int)
df_local['score_centered'] = df_local['score'] - cutoff

# Local linear regression with different slopes
df_local['interaction'] = df_local['above'] * df_local['score_centered']
X = df_local[['above', 'score_centered', 'interaction']]
y = df_local['outcome']

model = LinearRegression().fit(X, y)
rdd_effect = model.coef_[0]  # coefficient on 'above'
print(f"RDD estimate of causal effect: {rdd_effect:.4f}")
```

### RDD Assumptions and Validation

| Assumption | Check | Method |
|-----------|-------|--------|
| No manipulation of running variable | Density test at cutoff | McCrary test (check for bunching) |
| Continuity of potential outcomes | Covariates smooth at cutoff | Plot covariates against running var |
| Local to cutoff | Effect is for units AT threshold | Acknowledge limited generalizability |

---

## Natural Experiments {#natural-experiments}

### What Makes a Natural Experiment?

An event or policy that creates quasi-random variation in treatment assignment without researcher intervention.

### Famous Examples

| Natural Experiment | Treatment | Question Answered |
|-------------------|-----------|-------------------|
| Vietnam draft lottery | Military service | Effect on lifetime earnings |
| Oregon Medicaid lottery | Health insurance | Effect on health outcomes |
| Hurricane Katrina displacement | Neighborhood change | Effect of neighborhood on outcomes |
| COVID lockdowns | Remote work | Effect on productivity |
| Minimum wage increase (NJ vs. PA) | Higher minimum wage | Effect on employment |
| School enrollment cutoff dates | Extra year of school | Effect of age on academic performance |

### Evaluating Natural Experiments

```
Criteria for a good natural experiment:
  1. Assignment is "as-if random" — not chosen by individuals
  2. Clear treated/control groups identifiable
  3. No other simultaneous changes confound the comparison
  4. Sufficient sample size for statistical power
  5. External validity — can results generalize?
```

> **Critical Insight** 💡
> Natural experiments are found, not designed. The key skill is recognizing when a real-world event creates quasi-random variation. Data scientists should develop an eye for situations where some policy, threshold, or event split similar people into different groups — these are natural experiment opportunities.

---

## When RCTs Aren't Possible {#when-rcts-impossible}

### Reasons RCTs May Be Infeasible

| Reason | Example | Alternative Method |
|--------|---------|-------------------|
| Ethical constraints | Can't deny treatment to sick patients | Natural experiment, RDD |
| Scale limitations | Can't randomize an entire market | Diff-in-diff, synthetic control |
| Political/legal | Can't randomly assign tax rates | RDD, IV |
| Technical | Feature requires full rollout | Pre/post with CUPED, interrupted time series |
| Network effects | Users interact, violate SUTVA | Cluster randomization, switchback |
| Already launched | Want to evaluate past decision | Observational + DAG + sensitivity analysis |

### Decision Framework: Choosing a Method

```
Can you randomize?
├── YES → A/B test (RCT)
│
├── NO → Is there a threshold/cutoff for treatment?
│         ├── YES → Regression Discontinuity Design
│         │
│         └── NO → Is there a valid instrument?
│                   ├── YES → Instrumental Variables
│                   │
│                   └── NO → Is there a comparable control group over time?
│                             ├── YES → Difference-in-Differences
│                             │
│                             └── NO → Can you measure all confounders?
│                                       ├── YES → Propensity Score / Regression
│                                       │         (+ sensitivity analysis)
│                                       │
│                                       └── NO → Descriptive analysis only
│                                                 (state limitations clearly)
```

### Strength of Causal Evidence (Hierarchy)

```
STRONGEST EVIDENCE
    │
    │  1. Well-designed RCT
    │  2. Regression Discontinuity (sharp)
    │  3. Instrumental Variables (strong instrument)
    │  4. Difference-in-Differences (parallel trends validated)
    │  5. Propensity Score Methods (rich covariate set)
    │  6. Simple regression with controls
    │  7. Unadjusted observational comparison
    │
WEAKEST EVIDENCE
```

---

## Interview Talking Points {#interview-talking-points}

> **"Explain confounding in simple terms with an example."**
>
> "A confounder is a lurking variable that causes BOTH the treatment and the outcome, creating a spurious association. Classic example: ice cream sales and drowning deaths are positively correlated, but buying ice cream doesn't cause drowning — summer heat drives both. In tech: users who use premium features also have higher retention, but this might be because engaged users both adopt features AND stay longer, not because the features cause retention. The confounder is 'inherent engagement level.'"

> **"When would you use propensity score matching vs. an instrumental variable?"**
>
> "The choice depends on whether I can credibly assume 'no unobserved confounders.' If I have a rich set of covariates and domain knowledge suggests they capture all relevant differences between treated and untreated groups, PSM is appropriate — it's simpler and gives me the ATE. But if there's likely an unobserved confounder (like motivation or ability that I can't measure), I need an IV that creates exogenous variation in treatment. The trade-off: IV handles unobserved confounding but gives me only a local effect (LATE) and requires a valid instrument, which is often hard to find and impossible to fully verify."

> **"How would you evaluate the causal effect of a feature that was already launched without an A/B test?"**
>
> "First, I'd draw the DAG to map out what confounders exist. Then I'd look for opportunities: Was there a phased rollout creating a natural diff-in-diff? Is there a threshold that determined eligibility (RDD)? Is there an instrument? If none of these exist, I'd use observational methods (PSM or regression) with all available covariates, but I'd be transparent about limitations — run sensitivity analysis to show how strong an unobserved confounder would need to be to explain away the effect. I'd present it as 'suggestive evidence' rather than 'causal proof.'"

---

## Common Mistakes {#common-mistakes}

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| "We controlled for everything, so it's causal" | Can never control for ALL confounders without randomization |
| Conditioning on a collider (post-treatment variable) | Only condition on pre-treatment confounders; draw a DAG |
| Using PSM and claiming it handles unobserved confounding | PSM only handles observed confounders; run sensitivity analysis |
| Weak instrument (first-stage F < 10) | Verify instrument strength; report F-statistic |
| Ignoring exclusion restriction violation | Defend why instrument affects outcome ONLY through treatment |
| "Correlation implies no causation" | Correlation CAN be causal — the issue is determining WHICH direction and ruling out confounding |
| Over-controlling (conditioning on mediators) | Don't control for variables on the causal path between X and Y |
| RDD far from cutoff | RDD is local; don't extrapolate the effect to the full population |
| No overlap check in PSM | Always verify common support; trim non-overlapping regions |
| Ignoring time-varying confounding | In longitudinal settings, use marginal structural models or g-computation |

---

## Rapid-Fire Q&A {#rapid-fire-qa}

**Q1: What is the fundamental problem of causal inference?**
> We can never observe both the factual and counterfactual outcome for the same individual. We observe Y(1) OR Y(0), never both. Causal inference methods attempt to construct valid counterfactuals.

**Q2: What is a DAG and why use one?**
> A Directed Acyclic Graph representing causal assumptions. It makes assumptions explicit, identifies which variables to control for (and which NOT to), and reveals whether a causal effect is identifiable from observational data.

**Q3: What is the backdoor criterion?**
> A set Z satisfies the backdoor criterion if (1) no variable in Z is a descendant of X, and (2) Z blocks all backdoor paths from X to Y. Conditioning on such Z gives the causal effect.

**Q4: Why is conditioning on a collider dangerous?**
> It creates a spurious association between the collider's causes. Example: conditioning on "Hollywood star" (caused by both beauty and talent) creates a negative association between beauty and talent — the "explain-away" effect.

**Q5: What is the exclusion restriction in IV?**
> The instrument Z must affect the outcome Y ONLY through the treatment X. If Z has a direct effect on Y (bypassing X), the IV estimate is biased.

**Q6: What does LATE mean in IV estimation?**
> Local Average Treatment Effect — the causal effect for "compliers" (units whose treatment changes because of the instrument), not for the full population.

**Q7: When does propensity score matching fail?**
> When there are unobserved confounders, when there's no common support (treatment and control propensity score distributions don't overlap), or when the propensity score model is misspecified.

**Q8: What is the key assumption of RDD?**
> Continuity: potential outcomes are smooth functions of the running variable at the cutoff. If individuals can precisely manipulate their score to be above/below the cutoff, this assumption is violated.

**Q9: How do you check for manipulation in RDD?**
> McCrary density test: check if there's unusual bunching of observations just above or below the cutoff. A discontinuity in the density suggests manipulation.

**Q10: What is a sensitivity analysis in causal inference?**
> Quantifying how strong an unobserved confounder would need to be to overturn your causal conclusion. E.g., Rosenbaum bounds in PSM or E-value in observational studies.

---

## ASCII Cheat Sheet {#cheat-sheet}

```
╔══════════════════════════════════════════════════════════════════════╗
║          CAUSAL INFERENCE & CONFOUNDING CHEAT SHEET                 ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  THREE REASONS FOR CORRELATION:                                      ║
║  1. X → Y  (causation)                                             ║
║  2. X ← Y  (reverse causation)                                     ║
║  3. X ← Z → Y  (confounding)                                       ║
║                                                                      ║
║  DAG STRUCTURES:                                                     ║
║  Chain:    X → M → Y    (condition on M blocks path)               ║
║  Fork:     X ← Z → Y    (condition on Z blocks path)               ║
║  Collider: X → C ← Y    (conditioning OPENS path!)                 ║
║                                                                      ║
║  BACKDOOR CRITERION:                                                 ║
║  Adjust for Z if: (1) Z not descendant of X                        ║
║                   (2) Z blocks all backdoor paths X ← ... → Y      ║
║                                                                      ║
║  METHOD SELECTION:                                                   ║
║  ┌──────────────────────┬───────────────────────┬──────────┐       ║
║  │ Method               │ Key Assumption         │ Handles  │       ║
║  │                      │                        │ Unobs?   │       ║
║  ├──────────────────────┼───────────────────────┼──────────┤       ║
║  │ RCT                  │ Proper randomization   │ YES      │       ║
║  │ RDD                  │ Continuity at cutoff   │ YES*     │       ║
║  │ IV                   │ Exclusion restriction  │ YES      │       ║
║  │ Diff-in-Diff         │ Parallel trends        │ Partial  │       ║
║  │ PSM                  │ No unobs. confounders  │ NO       │       ║
║  │ Regression           │ Correct specification  │ NO       │       ║
║  └──────────────────────┴───────────────────────┴──────────┘       ║
║  (* locally, at the cutoff)                                         ║
║                                                                      ║
║  INSTRUMENTAL VARIABLES:                                             ║
║  Three requirements: (1) Relevance (Z→X, F>10)                     ║
║                      (2) Exclusion (Z→Y only via X)                 ║
║                      (3) Independence (Z ⊥ confounders)             ║
║  Gives: LATE (not ATE)                                              ║
║                                                                      ║
║  PROPENSITY SCORE:                                                   ║
║  e(X) = P(T=1|X)                                                   ║
║  Steps: Estimate → Check overlap → Match/Weight → Check balance    ║
║  ALWAYS do: sensitivity analysis for hidden bias                    ║
║                                                                      ║
║  CAUSAL EFFECT IDENTIFICATION CHECKLIST:                             ║
║  □ Draw DAG with domain knowledge                                   ║
║  □ Identify all backdoor paths                                      ║
║  □ Find sufficient adjustment set                                   ║
║  □ Check: no colliders being conditioned on                         ║
║  □ Check: no mediators being conditioned on                         ║
║  □ Assess sensitivity to unmeasured confounding                     ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 26 of 45*
