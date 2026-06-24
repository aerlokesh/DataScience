# 🎯 Topic 22: A/B Test Design End-to-End

> **Data Science Interview — Deep Dive**
> The most-asked topic in DS interviews at Meta, Google, Netflix, Uber, and Spotify. Walk through experiment design from hypothesis to launch decision — with the quantified rigor interviewers demand.

---

## Table of Contents

1. [The Full Experiment Lifecycle](#the-full-experiment-lifecycle)
2. [Randomization Unit Selection](#randomization-unit-selection)
3. [Control vs Treatment & Multi-Arm Experiments](#control-vs-treatment--multi-arm-experiments)
4. [Pre-Experiment Checklist](#pre-experiment-checklist)
5. [Duration Planning](#duration-planning)
6. [Analysis Framework](#analysis-framework)
7. [Decision Framework](#decision-framework)
8. [Ramp-Up Strategy](#ramp-up-strategy)
9. [Edge Cases & What-If Scenarios](#edge-cases--what-if-scenarios)
10. [Quantified Worked Example](#quantified-worked-example)
11. [Interview Walkthrough: Search Ranking A/B Test](#interview-walkthrough-search-ranking-ab-test)
12. [Interview Talking Points](#interview-talking-points)
13. [Common Interview Mistakes](#common-interview-mistakes)
14. [Rapid-Fire Q&A](#rapid-fire-qa)
15. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## The Full Experiment Lifecycle

The end-to-end A/B testing pipeline has seven distinct phases. Interviewers expect you to walk through each with awareness of what can go wrong.

### Phase 1: Hypothesis Formation

- **Structure**: "If we [change X], then [metric Y] will [increase/decrease] by [Z%], because [causal mechanism]."
- **Example**: "If we simplify the checkout flow from 4 steps to 2 steps, then purchase completion rate will increase by 5%, because reduced cognitive load decreases abandonment."
- A good hypothesis is falsifiable, specific, and tied to a business mechanism.

### Phase 2: Metric Selection

| Metric Type | Purpose | Examples |
|---|---|---|
| **Primary (OEC)** | Single metric the experiment optimizes for | Conversion rate, revenue per user |
| **Secondary** | Supporting metrics that explain the "why" | Click-through rate, time-to-convert |
| **Guardrail** | Metrics that must NOT degrade | Page load time, crash rate, revenue |
| **Counter-metrics** | Canary for unintended harm | Unsubscribe rate, support tickets |

**Key principle**: Define ONE Overall Evaluation Criterion (OEC) before the experiment starts. Multiple primary metrics invite p-hacking.

### Phase 3: Power Analysis & Sample Size

Determine how many users you need and for how long. This is where you show quantitative chops (see [Quantified Worked Example](#quantified-worked-example)).

### Phase 4: Randomization & Assignment

Choose the unit, method (hashing, stratification), and ensure independence across experiments.

### Phase 5: Duration & Execution

Run for at least 1-2 full business cycles. Monitor for instrumentation bugs, SRM, and traffic imbalances.

### Phase 6: Analysis

Apply intent-to-treat, variance reduction (CUPED), and multiple testing corrections. Examine heterogeneous treatment effects.

### Phase 7: Decision & Rollout

Combine statistical significance, practical significance, and guardrail health into a launch/no-launch/iterate recommendation.

---

## Randomization Unit Selection

Choosing the wrong randomization unit is one of the most common experiment design failures. The unit must match the level at which the treatment is experienced.

| Randomization Unit | When to Use | Pros | Cons | Company Example |
|---|---|---|---|---|
| **User (user_id)** | Most product changes | Consistent UX, no spillover within user | Needs logged-in users | Meta (News Feed), Netflix |
| **Session** | Logged-out experiments | Works without auth | Same user sees different variants across sessions | Google Search (logged-out) |
| **Page view** | Layout/UI micro-tests | High sample volume | Inconsistency within session, user confusion | E-commerce pricing display |
| **Device** | Cross-platform features | Handles multi-device users | Cookies can be cleared | Spotify (desktop vs mobile) |
| **Cluster (geo/market)** | Network effects, marketplace | Controls interference/spillover | Very few units → low power | Uber (city-level), Airbnb |
| **Cookie** | Anonymous web traffic | Simple to implement | Cookie churn inflates sample | Ad-tech experiments |

### When to Cluster-Randomize

Use cluster randomization when **spillover** between units is likely:
- **Marketplace experiments** (Uber pricing): driver supply affects all riders in a city
- **Social network experiments** (Meta): a user's behavior affects their friends
- **Content recommendation** (Netflix): changing one user's recs can shift inventory for others

**Trade-off**: Cluster randomization dramatically reduces effective sample size. A 10-city experiment with 1M users per city has an effective N closer to 10 (clusters), not 10M (users).

---

## Control vs Treatment & Multi-Arm Experiments

### Standard Two-Arm Design

- **Control**: Current experience (status quo)
- **Treatment**: The single change being tested
- Split: typically 50/50 for maximum power

### Multi-Arm Experiments

When you want to test multiple variants simultaneously:

| Design | Structure | Correction Needed | Use Case |
|---|---|---|---|
| **A/B/C** | 1 control + 2 treatments | Bonferroni or Holm | Testing 2 UI designs |
| **Factorial** | 2+ factors crossed | Interaction analysis | Button color × copy text |
| **Multi-armed bandit** | Dynamic allocation | Thompson sampling | Revenue optimization with regret cost |

**Multi-arm pitfalls**:
- Each additional arm reduces power against control (traffic splits thinner)
- Must correct for multiple comparisons (Bonferroni: α/k, Holm: step-down)
- Factorial designs require checking for interaction effects before interpreting main effects

**Traffic allocation for multi-arm** (k treatments + 1 control):
- Equal split: 1/(k+1) each
- Optimal (Dunnett): allocate √k more traffic to control → control gets √k/(√k + k) share

---

## Pre-Experiment Checklist

Before launching any experiment, run this validation checklist. Skipping it leads to uninterpretable results.

### 1. Sample Ratio Mismatch (SRM) Check

- Verify that randomization assigns expected proportions (e.g., 50/50 ± noise)
- Run a chi-squared test on the observed split: χ² = Σ (observed - expected)² / expected
- **Red flag**: If p < 0.001 for the SRM test, STOP. Your randomization is broken.
- Common causes: bot filtering, redirect failures, trigger condition bugs

### 2. A/A Test Validation

- Run an A/A test (both arms get identical experience) for 1 week
- Confirms that your randomization produces no false positives above α rate
- Check that metrics are balanced across arms (t-test p-values should be uniformly distributed)
- Validates the entire instrumentation pipeline

### 3. Pre-Experiment Covariate Balance

- Compare demographics, historical behavior, and platform mix across arms
- Use a joint F-test or Hotelling's T² for multivariate balance
- Imbalance suggests randomization failure or trigger condition bias

### 4. Novelty/Primacy User Filtering

- **New users**: Include only users who joined AFTER experiment start (avoids primacy bias)
- **Existing users**: Consider a burn-in period (first 2-3 days) where novelty-seeking behavior settles
- Netflix approach: compare effect on new-to-treatment vs. experienced users after 4 weeks

### 5. Instrumentation Audit

- Confirm logging fires correctly for both arms
- Verify metric definitions match between experiment platform and reporting
- Check for data delay/latency (events arriving after analysis window)

---

## Duration Planning

### Why 1-2 Full Weeks Minimum

| Factor | Why It Matters | Minimum Duration |
|---|---|---|
| **Day-of-week effects** | Weekday vs weekend behavior differs 20-40% | 7 days (1 full week) |
| **Payroll/billing cycles** | Monthly patterns in spending | 14-28 days |
| **Novelty effect** | Users explore new features then revert | 14+ days |
| **Sample accumulation** | Need sufficient N for power | Depends on traffic |
| **Seasonal events** | Holidays, promotions skew behavior | Avoid or extend past them |

### Duration Formula

```
Duration (days) = Required Sample Size / (Daily Active Users × Fraction in Experiment)
```

**Example**: Need 400,000 users per arm, DAU = 2M, experiment gets 100% traffic, 50/50 split:
- Each arm gets 1M users/day → need 400K/1M = 0.4 days for sample size
- But still run for 14 days to capture weekly cycles and stabilize effects

**Rule of thumb**: max(power-based duration, 14 days)

### Peeking Problem

- Looking at results before the planned end date inflates false positive rate
- If you check daily for 14 days at α=0.05, actual false positive rate ≈ 25%
- **Solutions**: sequential testing (alpha spending), always-valid confidence intervals, group-sequential designs

---

## Analysis Framework

### Intent-to-Treat (ITT) vs Per-Protocol

| Approach | Definition | When to Use | Bias Risk |
|---|---|---|---|
| **Intent-to-Treat** | Analyze everyone assigned to arm, regardless of compliance | Default for A/B tests | Conservative (diluted effect) |
| **Per-Protocol** | Analyze only those who actually saw the treatment | Measuring efficacy of exposure | Selection bias (non-random subset) |
| **CACE/LATE** | IV estimate of effect on compliers | When non-compliance is high | Requires valid instrument |

**Interview answer**: "Always start with ITT because it preserves randomization. If the ITT effect is null but you suspect non-compliance, report the per-protocol as a sensitivity analysis, noting the selection bias caveat."

### CUPED (Controlled-experiment Using Pre-Experiment Data)

CUPED reduces metric variance by 20-50%, effectively increasing power without more users.

**How it works**:
```
Y_adjusted = Y - θ × (X_pre - E[X_pre])

where:
  Y = post-experiment metric
  X_pre = same metric measured pre-experiment
  θ = Cov(Y, X_pre) / Var(X_pre)
```

**Variance reduction**:
```
Var(Y_adjusted) = Var(Y) × (1 - ρ²)

where ρ = correlation between pre and post metric
```

If ρ = 0.7, variance drops by 51% → equivalent to doubling sample size.

**Key requirements**:
- Pre-experiment data must be from BEFORE randomization (no contamination)
- Works best for metrics with high temporal autocorrelation (e.g., revenue, sessions)
- Used at Meta, Microsoft, Uber as standard practice

### Multiple Testing Corrections

| Method | Approach | When |
|---|---|---|
| **Bonferroni** | α/m for each of m tests | Conservative, independent tests |
| **Holm-Bonferroni** | Step-down sequential | Less conservative than Bonferroni |
| **Benjamini-Hochberg** | Controls FDR at q | Many secondary metrics |
| **Pre-registration** | Declare primary metric upfront | Best practice — no correction needed for OEC |

---

## Decision Framework

A statistically significant result is necessary but NOT sufficient for a launch decision. Use this three-pillar framework:

### Pillar 1: Statistical Significance

- Primary metric p-value < α (typically 0.05)
- Confidence interval excludes zero
- No evidence of SRM or instrumentation issues

### Pillar 2: Practical Significance

- Is the effect size large enough to matter for the business?
- Define the Minimum Detectable Effect (MDE) BEFORE the experiment
- Example: "We will only launch if conversion lifts by at least 0.5pp (10% relative)"
- Even if p < 0.01, a +0.01% lift may not justify engineering maintenance cost

### Pillar 3: Guardrail Health

- ALL guardrail metrics must be non-degraded (typically: CI upper bound for degradation < tolerance)
- Common guardrails: latency (p95), crash rate, revenue, long-term retention
- If primary metric improves but a guardrail regresses → ITERATE, do not launch

### Decision Matrix

| Statistical Sig? | Practical Sig? | Guardrails OK? | Decision |
|---|---|---|---|
| Yes | Yes | Yes | **LAUNCH** |
| Yes | Yes | No | **Iterate** (fix guardrail regression) |
| Yes | No | Yes | **No launch** (effect too small) |
| No | — | Yes | **No launch** (insufficient evidence) |
| No | — | No | **No launch** (potential harm) |

---

## Ramp-Up Strategy

Never go from 0% to 100% in one step. A staged rollout protects against catastrophic failures.

### Standard Ramp-Up Cadence

```
Stage 1:  1% traffic  →  2-3 days  →  Check instrumentation, SRM, crashes
Stage 2:  5% traffic  →  3-5 days  →  Validate directional signal, guardrails
Stage 3: 20% traffic  →  5-7 days  →  Statistical power for primary metric
Stage 4: 50% traffic  →  7-14 days →  Full-powered analysis, segment deep-dive
Stage 5: 100% rollout →  Monitor for 7+ days post-launch
```

### Why Ramp Gradually

1. **Safety**: Catch crashes/regressions early with minimal user impact
2. **Instrumentation**: Validate logging at small scale before committing
3. **Interaction effects**: At larger allocations, interactions with concurrent experiments emerge
4. **Stakeholder confidence**: Early directional signals build organizational buy-in

### Ramp-Down Triggers (Kill Criteria)

- Guardrail metric regresses beyond pre-defined threshold
- SRM detected (randomization failure)
- System instability (error rate spikes, latency > SLA)
- Ethical concern surfaces (e.g., discriminatory impact on a subgroup)

---

## Edge Cases & What-If Scenarios

Interviewers love to test your judgment on ambiguous results. Here are the most common scenarios:

### Scenario 1: p = 0.06 (Borderline Significance)

**What to do**:
- Do NOT round down to "significant." Report honestly as inconclusive.
- Check effect size: if large and practically meaningful, consider extending the test for more power.
- Examine the confidence interval: does it include your MDE?
- Run CUPED to reduce variance — the result might cross the threshold.
- **Never**: Lower α post-hoc, remove outliers to achieve significance, or cherry-pick subgroups.

### Scenario 2: Guardrail Metric Regresses

**What to do**:
- Investigate the mechanism: is the regression causal or coincidental?
- Segment the regression: does it affect all users or a specific subgroup?
- Quantify the trade-off: primary metric gain vs. guardrail loss in dollar terms.
- Decision: iterate on the treatment to preserve the gain while fixing the regression.
- **Example**: New recommendation algorithm lifts engagement +3% but increases page load by 200ms → optimize model inference speed, retest.

### Scenario 3: Segments Disagree

**What to do**:
- Check if segments were pre-registered or discovered post-hoc.
- Apply multiple testing correction for the number of segments examined.
- Verify the segment has sufficient sample size for reliable estimation.
- Consider: is the segment difference directional (effect stronger in power users) or contradictory (helps new users, hurts old)?
- **Interview answer**: "I'd report the overall ITT result as primary. Segment differences are hypothesis-generating for the next experiment, not actionable from this one — unless pre-registered."

### Scenario 4: Novelty Effect Suspected

**What to do**:
- Plot the treatment effect over time (day 1, day 7, day 14, day 21).
- If effect decays monotonically → likely novelty.
- Compare "new-to-treatment" users vs. users who have been in treatment for 2+ weeks.
- Netflix approach: wait for effect stabilization plateau before making decisions.
- **Rule**: If the effect at day 14 is less than 50% of the day-1 effect, flag as novelty-driven.

### Scenario 5: Interaction with Concurrent Experiments

**What to do**:
- Check if traffic overlaps with other running experiments.
- Look for interaction effects using 2×2 contingency tables.
- If significant interaction found: either mutex the experiments or model the interaction.
- Uber/Meta approach: experiment platforms flag potential interactions automatically.

---

## Quantified Worked Example

**Problem**: An e-commerce platform wants to test a new product recommendation widget on the homepage. Current purchase conversion rate is 5%. They want to detect a 3% relative lift (from 5.0% to 5.15%) with α = 0.05 and power = 0.80.

### Step 1: Calculate Sample Size Per Arm

Using the formula for two-proportion z-test:

```
n = (Z_α/2 + Z_β)² × [p₁(1-p₁) + p₂(1-p₂)] / (p₂ - p₁)²

Where:
  p₁ = 0.050 (control conversion rate)
  p₂ = 0.0515 (treatment: 5% × 1.03 = 5.15%)
  Z_α/2 = 1.96 (for α = 0.05, two-sided)
  Z_β = 0.84 (for power = 0.80)
  Δ = p₂ - p₁ = 0.0015

n = (1.96 + 0.84)² × [0.050×0.950 + 0.0515×0.9485] / (0.0015)²
n = (2.80)² × [0.0475 + 0.04885] / 0.00000225
n = 7.84 × 0.09635 / 0.00000225
n ≈ 335,502 per arm
```

**Total sample needed**: ~671,000 users (335,502 per arm)

### Step 2: Determine Duration

```
Given:
  DAU = 500,000
  Experiment allocation = 100% of traffic
  Split = 50/50

Users per arm per day = 500,000 × 0.50 = 250,000
Days needed for sample = 335,502 / 250,000 ≈ 1.35 days

But: minimum 14 days for weekly cycles

Duration = max(1.35, 14) = 14 days
```

### Step 3: Apply CUPED Reduction

If pre-experiment purchase data has ρ = 0.6 correlation with post-experiment purchases:

```
Adjusted variance = Var × (1 - 0.6²) = Var × 0.64
Adjusted n needed = 335,502 × 0.64 ≈ 214,721 per arm

New duration for sample = 214,721 / 250,000 ≈ 0.86 days
Still run 14 days for cycle coverage.
```

### Step 4: Results Interpretation

After 14 days, suppose you observe:
- Control: 5.02% conversion (n = 3,500,000 user-days)
- Treatment: 5.19% conversion (n = 3,500,000 user-days)
- Relative lift: +3.4%
- 95% CI for difference: [+0.05pp, +0.29pp]
- p-value: 0.006
- Guardrail (page load): +12ms (within 50ms tolerance)

**Decision**: LAUNCH — statistically significant, practically significant (exceeds 3% relative MDE), guardrails healthy.

---

## Interview Walkthrough: Search Ranking A/B Test

> **Interviewer**: "Design an A/B test for a new search ranking algorithm at a large tech company."

### Step 1: Clarify the Context

"Before diving in, I'd ask a few clarifying questions:
- Is this web search, in-app search, or e-commerce product search?
- What's the current search engagement baseline (CTR, queries per session)?
- Are there specific user segments we're targeting or is this platform-wide?
- What's the expected traffic volume?"

*Assume*: In-app product search for an e-commerce platform, 10M DAU, current search-to-purchase conversion is 8%.

### Step 2: Hypothesis

"If we deploy the new ML-based ranking algorithm, search-to-purchase conversion will increase by at least 2% relative (from 8.0% to 8.16%), because better relevance reduces the number of queries needed to find the desired product."

### Step 3: Metrics

| Type | Metric | Why |
|---|---|---|
| **Primary (OEC)** | Search-to-purchase conversion | Directly measures if better ranking drives purchases |
| **Secondary** | Queries per session, CTR@position1, time-to-first-click | Explain mechanism |
| **Guardrail** | Revenue per user, search latency (p95), null-result rate | Must not degrade |
| **Counter-metric** | Return rate within 7 days | Ensure quality purchases |

### Step 4: Randomization

- **Unit**: User-level (user_id hash)
- **Why not query-level**: Same user seeing different rankings across queries creates confusion
- **Why not session-level**: Cross-session consistency matters for learning effects

### Step 5: Sample Size & Duration

```
Baseline p = 0.08, MDE = 2% relative → Δ = 0.0016
n per arm ≈ 240,000 users (with CUPED, ρ=0.65)
DAU = 10M, 50/50 split → 5M per arm per day
Duration = max(power_days=0.05, 14) = 14 days
```

### Step 6: Execution Plan

- Run A/A test for 3 days to validate instrumentation
- Ramp: 1% (2 days) → 10% (5 days) → 50% (14 days full analysis)
- Monitor SRM daily, guardrails daily, primary metric at end only (avoid peeking)

### Step 7: Analysis Plan

- ITT analysis on all randomized users
- CUPED adjustment using 28-day pre-experiment search conversion
- Segment analysis: new vs. returning users, mobile vs. desktop, high vs. low search frequency
- Check for novelty by plotting daily treatment effect over the 14 days

### Step 8: Decision Criteria

"I'd recommend launch if: primary metric CI lower bound > 0 (stat sig), effect > 1% relative (practical sig), latency guardrail p95 increase < 100ms, and no revenue regression."

---

## Interview Talking Points

> **On explaining your approach to an interviewer:**
>
> "My framework for A/B test design has three non-negotiable steps before any code is written: first, I nail down a single OEC with stakeholders — this prevents goal-post shifting. Second, I run a power analysis to set the sample size and minimum duration, which I communicate as a calendar date, not a user count, so product managers understand the timeline. Third, I define kill criteria and guardrails upfront so we have a pre-committed decision framework when results come in."

> **On handling ambiguous results:**
>
> "When I see a borderline result — say p = 0.06 with a meaningful effect size — I resist the temptation to call it significant. Instead, I present the confidence interval and frame it as 'the data is consistent with both no effect and a meaningful positive effect.' I then recommend either extending the test if the CI includes our MDE, or applying CUPED to squeeze more precision from the existing data. What I never do is change the significance threshold after seeing the data."

> **On communicating experiment results to non-technical stakeholders:**
>
> "I translate statistical results into business impact. Instead of saying 'the treatment effect is 0.15 percentage points with p = 0.003,' I say 'if we launch this to all users, we'd expect approximately 15,000 additional purchases per month based on our traffic, and we're confident this isn't due to random chance.' I always pair the point estimate with the confidence interval translated into dollar terms: 'somewhere between $200K and $800K additional monthly revenue.'"

> **On when NOT to A/B test:**
>
> "Not everything should be A/B tested. I'd skip experimentation when: the change is legally required or ethically necessary, when the feature has obvious network effects that make randomization meaningless, when the user base is too small for statistical power within a reasonable timeframe, or when the change is trivially reversible and the cost of a wrong decision is near-zero. In those cases, I'd propose alternatives: quasi-experimental methods like diff-in-diff, interrupted time series, or simply a staged rollout with pre/post monitoring."

---

## Common Interview Mistakes

| # | Mistake (❌) | Correct Approach (✅) |
|---|---|---|
| 1 | ❌ "I'd check results daily and stop when significant" | ✅ "I'd pre-commit to a fixed duration based on power analysis, or use sequential testing with alpha-spending" |
| 2 | ❌ "Randomize at the page-view level for a UI redesign" | ✅ "Randomize at user level to ensure consistent experience and avoid learning/novelty contamination" |
| 3 | ❌ "If p > 0.05, the treatment has no effect" | ✅ "Failure to reject H₀ means insufficient evidence. The CI may still include meaningful effects — check if the test was underpowered" |
| 4 | ❌ "I'd use a one-sided test to get more power" | ✅ "Default to two-sided unless there's a strong prior that the effect can only go one direction (rare in practice)" |
| 5 | ❌ "We'll look at 20 subgroups and report whichever is significant" | ✅ "Pre-register primary segments. Apply Benjamini-Hochberg correction for exploratory subgroups" |
| 6 | ❌ "The test is significant so we should launch" | ✅ "Statistical significance is necessary but not sufficient. I also need practical significance and healthy guardrails" |
| 7 | ❌ "We need millions of users so let's run for just 2 days" | ✅ "Even with sufficient sample size, run at least 14 days to capture day-of-week effects and let novelty wear off" |
| 8 | ❌ "I'll use total revenue as the metric" | ✅ "Use revenue per user (ratio metric) to avoid confounding with sample size differences between arms" |
| 9 | ❌ "Let's split 90/10 to minimize risk" | ✅ "90/10 splits reduce power by 2.8× vs 50/50. Use ramp-up stages (1% → 50%) instead to balance risk and power" |
| 10 | ❌ "We can just remove outliers that look weird" | ✅ "Pre-register outlier handling rules (e.g., Winsorize at 99th percentile) before seeing results" |

---

## Rapid-Fire Q&A

**Q1: What is the minimum sample size formula for a proportion metric?**
> n = (Z_α/2 + Z_β)² × 2p(1-p) / Δ² for equal-variance approximation, where Δ = p₂ - p₁.

**Q2: Why is 50/50 the optimal split for a two-arm test?**
> The variance of the difference is minimized when both arms have equal sample sizes. Power is proportional to 1/√(1/n₁ + 1/n₂), maximized at n₁ = n₂.

**Q3: What is an SRM and why does it matter?**
> Sample Ratio Mismatch — when observed assignment proportions deviate significantly from expected (e.g., 51.2%/48.8% instead of 50/50). It indicates a randomization bug that invalidates causal inference.

**Q4: How does CUPED work in one sentence?**
> CUPED subtracts a scaled version of each user's pre-experiment metric from their post-experiment metric, reducing variance by the squared correlation between pre and post periods.

**Q5: What's the difference between statistical and practical significance?**
> Statistical significance means the observed effect is unlikely under H₀ (p < α). Practical significance means the effect is large enough to justify the business cost of implementation and maintenance.

**Q6: When would you use a cluster-randomized design?**
> When units interact (marketplace, social network, shared resources) and treating one user affects others — cluster on the unit where interference is contained (city, friend group, market).

**Q7: How do you handle multiple primary metrics?**
> Don't. Pick one OEC. If stakeholders insist on two, either combine them into a composite metric or apply Bonferroni correction (α/2 for each).

**Q8: What is the peeking problem and how do you solve it?**
> Checking results repeatedly inflates Type I error (α=0.05 with daily checks for 14 days → ~25% false positive rate). Solutions: sequential testing, alpha-spending functions (O'Brien-Fleming), or always-valid confidence intervals.

**Q9: How long should you wait after a significant result before declaring novelty has worn off?**
> Monitor for 2-4 weeks. Compare the treatment effect in week 1 vs. weeks 3-4. If the effect stabilizes (< 20% decay from peak), novelty is resolved.

**Q10: What is intent-to-treat and why is it the default?**
> ITT analyzes all users in their assigned arm regardless of whether they actually experienced the treatment. It preserves randomization and gives an unbiased estimate of the policy effect (assigning users vs. forcing exposure).

**Q11: How do you size a test for a ratio metric (e.g., revenue per user) with high variance?**
> Use the empirical variance from historical data: n = (Z_α/2 + Z_β)² × 2σ² / Δ². For skewed revenue, consider log-transformation, Winsorization at 99%, or CUPED to reduce σ².

**Q12: What's the difference between a multi-armed bandit and a standard A/B test?**
> A/B tests optimize for learning (fixed allocation, precise estimates). Bandits optimize for reward during the experiment (dynamic allocation, minimize regret). Use bandits when the cost of sub-optimal assignment is high and you need less precise effect estimates (e.g., ad creative rotation).

---

## Summary Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   A/B TEST DESIGN — CHEAT SHEET                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  LIFECYCLE:  Hypothesis → Metrics → Power → Randomize → Run → Analyze  │
│              → Decide                                                   │
│                                                                         │
│  METRICS:    1 OEC (primary) + secondary + guardrails + counter-metrics │
│                                                                         │
│  POWER:      n = (Z_α/2 + Z_β)² × 2p(1-p) / Δ²                       │
│              α = 0.05, β = 0.20 → Z's = 1.96, 0.84                    │
│                                                                         │
│  UNIT:       User (default) | Session (no auth) | Cluster (spillover)  │
│                                                                         │
│  DURATION:   max(power-based days, 14 days for weekly cycles)          │
│                                                                         │
│  CUPED:      Var_reduced = Var × (1 - ρ²), ρ = pre/post correlation   │
│                                                                         │
│  DECISION:   Stat sig ∩ Practical sig ∩ Guardrails OK → LAUNCH         │
│                                                                         │
│  RAMP:       1% → 5% → 20% → 50% → 100%                              │
│                                                                         │
│  PITFALLS:   Peeking, SRM, wrong unit, no guardrails, novelty effect   │
│                                                                         │
│  NEVER:      Change α post-hoc | Remove outliers after seeing data     │
│              | Declare "no effect" from underpowered test               │
│                                                                         │
│  ALWAYS:     Pre-register OEC | Run ≥ 14 days | Check SRM first        │
│              | Report CIs, not just p-values | Define MDE upfront       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Key Formulas Quick Reference

| Formula | Use Case |
|---|---|
| `n = (1.96 + 0.84)² × 2p(1-p) / Δ²` | Sample size for proportions (80% power, α=0.05) |
| `n = (1.96 + 0.84)² × 2σ² / Δ²` | Sample size for continuous metrics |
| `Duration = n / (DAU × allocation × arm_fraction)` | Days needed |
| `Var_cuped = Var × (1 - ρ²)` | CUPED variance reduction |
| `χ² = Σ(O-E)²/E` | SRM detection |
| `Power = P(reject H₀ | H₁ true) = 1 - β` | Power definition |
| `MDE = Δ / baseline` | Minimum detectable effect (relative) |

---

## Recommended Reading

- Kohavi, Tang & Xu — *Trustworthy Online Controlled Experiments* (the bible of A/B testing)
- Deng et al. (2013) — "Improving the Sensitivity of Online Controlled Experiments by Utilizing Pre-Experiment Data" (CUPED paper)
- Johari et al. (2017) — "Peeking at A/B Tests" (always-valid inference)
- Microsoft ExP Platform documentation (public blog posts)
- Uber Engineering blog on experimentation and interference

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 22 of 45*
