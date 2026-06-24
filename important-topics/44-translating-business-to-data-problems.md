# 🎯 Topic 44: Translating Business Problems to Data Problems

> *"VERY IMPORTANT for senior roles. The most valuable data scientists don't just answer questions — they figure out the RIGHT question to ask. Translating vague business needs into crisp, answerable data problems is the core senior skill."*

---

## 📑 Table of Contents

1. [Why This Skill Matters](#why-this-skill-matters)
2. [Framework for Requirement Gathering](#framework-for-requirement-gathering)
3. [Ambiguity Resolution](#ambiguity-resolution)
4. [Translating "Should We Launch X?" Into an Analysis Plan](#translating-should-we-launch-x-into-an-analysis-plan)
5. [Scoping: What Questions Do We Need to Answer?](#scoping-what-questions-do-we-need-to-answer)
6. [Feasibility Assessment](#feasibility-assessment)
7. [Stakeholder Communication Patterns](#stakeholder-communication-patterns)
8. [Communicating Tradeoffs](#communicating-tradeoffs)
9. [Saying "No" with Data](#saying-no-with-data)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Why This Skill Matters

### The Senior Data Scientist's Real Job

```
Junior DS:  "Here are the results of the query you asked for."
Mid DS:     "Here are the results, and here's what they mean."
Senior DS:  "I heard your business problem, designed the right analysis,
             executed it, and here's my recommendation with tradeoffs."
```

At 5 YOE, you're expected to:
1. **Hear** an ambiguous business need
2. **Translate** it into a precise analytical question
3. **Scope** what data/methods are needed
4. **Execute** the analysis (or delegate it)
5. **Communicate** findings as actionable recommendations

### The Translation Gap

```
Business stakeholder says:          Data scientist must hear:
─────────────────────────────────────────────────────────────
"We need to grow faster"            → What's limiting growth? Where's 
                                       the biggest lever?

"Is our new feature working?"       → Define "working" — engagement? 
                                       retention? revenue? For whom?

"Should we expand to Germany?"      → What's the market size? Can our 
                                       unit economics work there?

"Customers are unhappy"             → Which customers? About what? 
                                       How do we measure happiness?

"We're losing to competitors"       → Where specifically? Which segment?
                                       What's their advantage?
```

> **Critical Insight:** Stakeholders almost never give you a well-defined analytical problem. They give you a business anxiety or aspiration. YOUR job is to translate that into something answerable. If you just execute what they literally asked for, you'll often answer the wrong question.

---

## Framework for Requirement Gathering

### The CRISP Framework

```
C - Context:    What's the business situation? Why now?
R - Result:     What decision will this analysis inform?
I - Input:      What data do we have? What constraints exist?
S - Scope:      How deep should we go? What's out of scope?
P - Precision:  How accurate does the answer need to be?
```

### Questions to Always Ask

**Understanding the Decision:**
```
1. "What decision are you trying to make?"
2. "What would you do differently if the answer is X vs Y?"
3. "When do you need this by? What's the cost of waiting?"
4. "Who else is involved in this decision?"
5. "What have you already tried or ruled out?"
```

**Understanding the Metric:**
```
6. "How do you define 'success' here?"
7. "Over what time period?"
8. "For which user segment / product / geography?"
9. "What would a 'good enough' answer look like?"
10. "Is this a one-time question or ongoing monitoring?"
```

**Understanding Constraints:**
```
11. "Do we have the data to answer this?"
12. "Are there budget/resource constraints?"
13. "Are there legal/privacy considerations?"
14. "What level of rigor is needed? (Directional vs precise?)"
15. "Is there a hypothesis you're already leaning toward?"
```

### Example Translation Session

```
STAKEHOLDER: "We need to understand if our premium tier is worth it."

DATA SCIENTIST'S CLARIFYING QUESTIONS:
├── "Worth it for whom — for us (revenue) or for users (value)?"
│    → "Both, but start with whether users who upgrade retain better."
├── "How do you define 'upgrade'? Trial→Paid? Basic→Premium?"
│    → "Basic→Premium specifically."
├── "What time horizon? 1 month? 6 months? Lifetime?"
│    → "Let's look at 6-month retention and LTV."
├── "Should we account for self-selection bias?"
│    → "What do you mean?" → [Explain: premium users may already be
│      more engaged; upgrading doesn't necessarily CAUSE retention]
├── "What action would you take with this analysis?"
│    → "If premium users retain better, we'll invest in conversion.
│       If not, we'll reconsider the tier structure."
└── "When do you need this?"
     → "Board meeting in 3 weeks."

TRANSLATED DATA PROBLEM:
"Compare 6-month retention and LTV between users who upgrade to 
premium vs. comparable users who don't, controlling for pre-upgrade 
engagement level. Deliver recommendation in 2 weeks."
```

---

## Ambiguity Resolution

### Types of Ambiguity

| Type | Example | Resolution Strategy |
|------|---------|-------------------|
| **Metric Ambiguity** | "Improve engagement" | Define: DAU? Session time? Feature usage? |
| **Scope Ambiguity** | "Analyze our users" | Clarify: all users? New? US only? Last 90 days? |
| **Temporal Ambiguity** | "Is it getting worse?" | Define: compared to when? Last week? Last year? |
| **Causal Ambiguity** | "Did the feature work?" | Clarify: correlation or causation needed? |
| **Action Ambiguity** | "Help us grow" | Ask: grow what? Users? Revenue? Which lever? |
| **Precision Ambiguity** | "Give me the numbers" | Ask: directional estimate or precise measurement? |

### The Ambiguity Resolution Ladder

```
Level 1: CLARIFY THE OBJECTIVE
  "What business outcome are we trying to achieve?"
  
Level 2: DEFINE THE METRIC
  "How would we measure success numerically?"
  
Level 3: SET THE SCOPE
  "For which users, time period, and geography?"
  
Level 4: DETERMINE THE METHOD
  "Do we need causation or correlation? How rigorous?"
  
Level 5: AGREE ON DELIVERABLE
  "What format and when? Deck? Dashboard? Quick answer?"
```

### Handling "I Don't Know What I Need"

```
STAKEHOLDER: "I just want to understand our users better."

BAD RESPONSE: "OK, I'll pull some user data and make charts."
  (Unscoped, will waste weeks producing unused deliverables)

GOOD RESPONSE: "Let me propose three angles we could explore:
  1. Segmentation: Who are our users and how do they differ?
  2. Journey: What does a typical user path look like?
  3. Health: Which users are at risk of churning?
  
  Which of these would be most valuable for decisions 
  you're making in the next month?"
```

> **Critical Insight:** When stakeholders can't articulate what they need, propose 2-3 concrete options. This is faster than open-ended exploration and gives them something to react to. People are better at evaluating options than generating them from scratch.

---

## Translating "Should We Launch X?" Into an Analysis Plan

### The Launch Decision Framework

```
"Should we launch feature X?" decomposes into:

1. SIZING: How big is the opportunity?
   - How many users would be affected?
   - What's the potential revenue/engagement impact?
   - What's our confidence in these estimates?

2. EVIDENCE: What does existing data tell us?
   - Did the A/B test show significant improvement?
   - On which metrics? For which segments?
   - Are there tradeoffs (improved metric A, degraded metric B)?

3. RISK: What could go wrong?
   - Cannibalization of existing features?
   - Increased support load?
   - Technical debt or scalability concerns?
   - Regulatory/legal considerations?

4. COST: What does it take to launch?
   - Engineering effort to productionize?
   - Ongoing maintenance?
   - Opportunity cost (what won't we build instead)?

5. REVERSIBILITY: Can we undo this?
   - Is it a one-way door (hard to reverse) or two-way door?
   - What's the rollback plan?
```

### Analysis Plan Template

```
ANALYSIS PLAN: Should we launch [Feature X]?

OBJECTIVE:
  Determine if Feature X should launch to all users based on 
  A/B test results and projected business impact.

KEY QUESTIONS:
  1. What is the measured impact on primary metric (conversion)?
  2. Are there negative impacts on secondary metrics (revenue, retention)?
  3. Does the effect vary by user segment?
  4. Is the effect sustained over time (novelty vs lasting)?
  5. What is the projected annual revenue impact at full scale?

DATA SOURCES:
  - A/B test results (Experiment Platform, test #1234)
  - Revenue data (data warehouse, orders table)
  - User segments (user_dimensions table)
  - Historical feature launches (for baseline comparison)

METHODOLOGY:
  - Statistical significance testing (frequentist, alpha=0.05)
  - Segment heterogeneity analysis (interaction effects)
  - Long-run effect estimation (extrapolate from test duration)
  - Cannibalization check (did other features decline?)

TIMELINE:
  - Week 1: Pull and validate data, run primary analysis
  - Week 2: Segment analysis, sensitivity checks
  - Week 3: Write recommendation, present to stakeholders

DELIVERABLE:
  1-page recommendation memo with data appendix
  Recommendation: Launch / Don't Launch / Launch with modifications

DECISION MAKER: VP of Product
DECISION DATE: March 30, 2024
```

---

## Scoping: What Questions Do We Need to Answer?

### The Scoping Funnel

```
ALL POSSIBLE QUESTIONS (infinite)
         │
    ┌────┴────┐
    │ Filter  │  "Which questions actually matter for the decision?"
    └────┬────┘
         │
RELEVANT QUESTIONS (many)
         │
    ┌────┴────┐
    │ Filter  │  "Which ones can we answer with available data?"
    └────┬────┘
         │
ANSWERABLE QUESTIONS (some)
         │
    ┌────┴────┐
    │ Filter  │  "Which ones can we answer within the timeline?"
    └────┬────┘
         │
IN-SCOPE QUESTIONS (few)
         │
    ANALYSIS PLAN
```

### Prioritization Matrix

| Question | Decision Impact | Data Available | Effort | Priority |
|----------|----------------|----------------|--------|----------|
| What's the conversion lift? | Critical | Yes (A/B test) | Low | P0 |
| Does it cannibalize feature Y? | High | Yes (usage data) | Medium | P1 |
| What's the 12-month revenue impact? | High | Partial (need assumptions) | Medium | P1 |
| How does it affect NPS? | Medium | Not yet (survey needed) | High | P2 (defer) |
| Will it work in APAC markets? | Medium | No (no APAC test) | Very High | P3 (out of scope) |

### Explicit Out-of-Scope Statement

```
OUT OF SCOPE (and why):
- Long-term (12-month+) retention effects: Test was only 4 weeks.
  RISK ACCEPTED: We'll monitor post-launch.
- APAC market impact: No users in test from APAC. 
  MITIGATION: Staged rollout to APAC with monitoring.
- Competitive response: No data available.
  MITIGATION: Monitor market share post-launch.
```

> **Critical Insight:** Defining what's OUT of scope is as important as defining what's IN scope. Unstated scope leads to scope creep ("can you also check...?") and unmet expectations ("I thought you'd cover market impact"). Be explicit about boundaries and the reasoning behind them.

---

## Feasibility Assessment

### The Feasibility Checklist

```
DATA FEASIBILITY:
  [ ] Do we have the necessary data?
  [ ] Is it at the right granularity?
  [ ] Is it of sufficient quality?
  [ ] Do we have enough historical data?
  [ ] Are there privacy/legal restrictions?

METHODOLOGICAL FEASIBILITY:
  [ ] Can we establish causation (or only correlation)?
  [ ] Is the sample size sufficient?
  [ ] Are there confounders we can't control for?
  [ ] Do standard methods apply, or do we need custom approaches?

RESOURCE FEASIBILITY:
  [ ] Can we do this within the timeline?
  [ ] Do we have the compute resources?
  [ ] Do we have the expertise?
  [ ] Is this the best use of our team's time?

IMPACT FEASIBILITY:
  [ ] Will the answer actually change a decision?
  [ ] Can the organization act on findings?
  [ ] Is the timing right (decision hasn't already been made)?
```

### Communicating Feasibility Constraints

```
STAKEHOLDER: "Can you tell me the causal impact of our brand 
campaign on revenue?"

HONEST RESPONSE:
"I can give you a directional estimate, but here's what limits 
our precision:

What I CAN do:
- Before/after comparison with seasonal adjustment (~1 week)
- Geo-holdout analysis if we have a control market (~2 weeks)
- Marketing mix model estimate with confidence interval (~3 weeks)

What I CAN'T do:
- True causal measurement (would need randomized experiment)
- Isolate brand vs. performance marketing effects perfectly
- Account for competitor actions during the same period

My recommendation: Start with the geo-holdout analysis. It gives 
us the best causal estimate within our data constraints. We'll 
have a directional answer in 2 weeks with a confidence interval 
around +/- 15%.

Acceptable?"
```

---

## Stakeholder Communication Patterns

### The BLUF Pattern (Bottom Line Up Front)

```
BAD (burying the lead):
"I analyzed 6 months of data across 15 segments using 3 different 
models and found that after controlling for seasonality and 
channel mix..."

GOOD (BLUF):
"Recommendation: Launch the feature. It increases conversion by 
3.2% with no measurable negative impact on revenue or retention.

Supporting evidence:
- A/B test: +3.2% conversion (p<0.01, n=500K)
- No cannibalization detected in adjacent features
- Effect consistent across all user segments
- Projected annual revenue impact: +$2.4M

Details and methodology in appendix."
```

### Adapting to Audience

| Audience | Communication Style |
|----------|-------------------|
| **Executive** | 1-sentence recommendation + impact number. "Launch it: +$2.4M/year." |
| **Product Manager** | Recommendation + key tradeoffs + segment details |
| **Engineering** | Technical approach + data requirements + confidence level |
| **Data Team** | Methodology details + assumptions + limitations |
| **Finance** | ROI framing + confidence intervals + scenario analysis |

### Managing Expectations

```
AT THE START:
"I expect to have a directional answer by Friday and a 
rigorous analysis by end of next week. If I hit blockers 
with data quality, I'll flag by Wednesday."

DURING:
"Quick update: The data looks clean and the primary analysis 
confirms our hypothesis. I'm running sensitivity checks now. 
On track for the Friday deliverable."

IF SCOPE CHANGES:
"You mentioned wanting APAC analysis too. That's 3-5 additional 
days because we'd need to pull separate data. Should I:
A) Include APAC and deliver next Wednesday, or
B) Deliver US-only on Friday, APAC as follow-up?"
```

---

## Communicating Tradeoffs

### The Tradeoff Matrix

```
OPTION A: Launch to all users immediately
  + Full revenue capture ($2.4M/year)
  + Fastest time to market
  - Risk: 5% chance the effect is smaller than measured
  - No APAC validation
  
OPTION B: Staged rollout (25% → 50% → 100% over 6 weeks)
  + Validates sustained effect
  + Catches edge cases early
  - Delays full revenue by 6 weeks (~$275K opportunity cost)
  - More engineering complexity
  
OPTION C: Don't launch, run extended test
  + Higher confidence in long-term effect
  - 4-week delay minimum
  - $460K opportunity cost
  - Test fatigue in user base

MY RECOMMENDATION: Option B (staged rollout)
REASONING: De-risks without material revenue delay. The 
$275K opportunity cost is worth the confidence gained.
```

### Quantifying Uncertainty

```
Instead of: "We think revenue will increase."

Say: "Our best estimate is $2.4M additional annual revenue.
      - Optimistic scenario (90th percentile): $3.1M
      - Conservative scenario (10th percentile): $1.8M
      - Worst case (if effect decays): $1.2M
      
      Even in the worst case, the ROI is positive given 
      $200K implementation cost."
```

> **Critical Insight:** Never present a single point estimate without uncertainty. Stakeholders need to understand the range of outcomes to make informed decisions. A $2.4M estimate with +/- $1.5M range is a very different situation from $2.4M with +/- $200K.

---

## Saying "No" with Data

### When to Push Back

```
PUSH BACK WHEN:
1. The question can't be answered with available data
2. The timeline is impossible for rigorous analysis
3. The proposed metric is misleading
4. The analysis would confirm a decision already made (vanity)
5. The approach would cause harm (biased, unethical)
6. Resources would be better spent elsewhere
```

### How to Say No Constructively

```
BAD: "No, we can't do that."
BAD: "Sure, I'll try." (then deliver garbage)

GOOD: "I can't give you what you asked for, but here's what 
I CAN give you that would be more useful:"

PATTERN:
1. Acknowledge the underlying need
2. Explain why the specific request is problematic
3. Propose an alternative that serves the same need
4. Get alignment on the alternative
```

### Examples

```
REQUEST: "Can you prove our product is better than competitor's?"
PUSHBACK: "I can't 'prove' superiority because we don't have 
comparable data. But I can quantify our strengths:
- Retention rates vs. industry benchmarks
- NPS relative to category average
- Feature adoption vs. competitor public disclosures
Would that serve the same purpose?"

REQUEST: "Run this A/B test on 500 users."
PUSHBACK: "With 500 users and your expected 2% lift, we'd need 
to run for 8 months to reach significance. Three alternatives:
1. Increase sample to 50K users → result in 2 weeks
2. Accept a larger minimum detectable effect (10%) → 500 users works
3. Use a different metric with higher variance → faster convergence
Which constraint can we relax?"

REQUEST: "I need this by tomorrow."
PUSHBACK: "I can give you a directional answer by tomorrow 
(rough estimate based on historical patterns). A rigorous 
analysis with confidence intervals needs a week. Which serves 
your immediate need better? If the decision can't wait, I'd 
caveat the quick estimate heavily."
```

---

## Interview Talking Points

> "The most impactful project I led started with a vague ask: 'Help us understand why growth is slowing.' I spent two days in requirement gathering — interviewing the VP of Growth, the marketing lead, and the product team. I narrowed it from 'understand growth' to three specific hypotheses: (1) acquisition channel saturation, (2) activation rate decline, (3) increased churn in a specific cohort. Each became a 1-week workstream with a clear deliverable. Without that scoping, I would have spent months producing an unfocused 'growth report.'"

> "I once pushed back on a stakeholder who wanted a 50-metric dashboard covering everything. I asked 'What decisions will you make differently based on each metric?' and found that only 8 metrics actually informed decisions. The rest were 'nice to know.' We built an 8-metric dashboard that became the most-used in the company, and created a separate ad-hoc analysis process for the rest."

> "My approach to feasibility is upfront and honest. When our GM asked for causal attribution of a branding campaign, I explained that true causal measurement wasn't possible without a geo-holdout (which we hadn't set up). Instead of pretending correlation was causation, I proposed a marketing mix model with explicit confidence intervals. The GM appreciated the honesty and approved a geo-holdout design for the next campaign."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Accepting the first ask at face value | Ask "what decision will this inform?" to uncover true need |
| Starting analysis before scoping | Spend 20% of time on scoping, 60% on execution, 20% on communication |
| Answering the literal question instead of the real question | Translate business anxiety into analytical question |
| Delivering results without recommendation | Always include: "Based on this, I recommend X because..." |
| Saying yes to everything | Evaluate feasibility, push back constructively, propose alternatives |
| Presenting findings without uncertainty | Always quantify confidence intervals and assumptions |
| Not defining what's out of scope | Explicitly state scope boundaries to prevent creep |
| Over-engineering when a quick estimate suffices | Match rigor to decision importance (80/20 rule) |
| Letting stakeholders define the methodology | You're the expert — propose the approach, explain why |
| Working in isolation then surprising stakeholders | Check in at 30% and 70% completion to course-correct |

---

## Rapid-Fire Q&A

**Q1: What's the most important question to ask a stakeholder?**
"What decision will this analysis inform, and what would you do differently based on the outcome?" This reveals whether the work is actually actionable.

**Q2: How do you handle "I need this yesterday"?**
Offer a tiered response: "I can give you a rough directional answer today, a validated estimate by Wednesday, or a rigorous analysis by next Friday. Which level of rigor does this decision require?"

**Q3: How do you scope an ambiguous problem?**
Use the funnel: all possible questions → filter by decision relevance → filter by data availability → filter by timeline → focused analysis plan.

**Q4: When should you push back on a request?**
When it can't be answered with available data, when the timeline is impossible for rigor, when the metric is misleading, or when resources would be better spent elsewhere.

**Q5: How do you translate "should we launch X?" into analysis?**
Decompose into: sizing (how big is the opportunity), evidence (what does data show), risk (what could go wrong), cost (engineering + opportunity), and reversibility (can we undo it).

**Q6: What's the difference between a senior and junior approach to stakeholder requests?**
Junior executes the literal request. Senior understands the underlying business need, translates it into the right analytical question, and delivers a recommendation — not just data.

**Q7: How precise does an analysis need to be?**
Match precision to the decision. A "should we invest $50M?" decision needs rigorous analysis with confidence intervals. A "which color for the button?" decision needs a quick A/B test result.

**Q8: How do you handle conflicting stakeholder priorities?**
Make the tradeoffs explicit. "If we optimize for engagement, revenue might decline 5%. If we optimize for revenue, engagement drops 8%. Here are the numbers — which tradeoff does the business prefer?"

**Q9: What if the data doesn't support the stakeholder's hypothesis?**
Present the findings honestly: "The data shows X, which contradicts our hypothesis of Y. Here are three possible explanations: [1], [2], [3]. I recommend we investigate [1] further because..."

**Q10: How do you know when an analysis is "done enough"?**
When additional analysis would not change the recommendation. If your confidence interval is tight enough to make the decision, stop refining.

---

## ASCII Cheat Sheet

```
+============================================================+
|   TRANSLATING BUSINESS TO DATA PROBLEMS — CHEAT SHEET       |
+============================================================+

THE TRANSLATION CHAIN:
  Business Anxiety → Business Question → Data Question
  → Analysis Plan → Execution → Recommendation → Decision

CRISP FRAMEWORK:
  C - Context     (Why now? What's the situation?)
  R - Result      (What decision does this inform?)
  I - Input       (What data/constraints exist?)
  S - Scope       (How deep? What's out of scope?)
  P - Precision   (How accurate must the answer be?)

QUESTIONS TO ALWAYS ASK:
  1. "What decision will this inform?"
  2. "How do you define success?"
  3. "For which segment/period/geo?"
  4. "When do you need this?"
  5. "What would you do differently if answer is X vs Y?"

SCOPING FUNNEL:
  All Questions → Decision-Relevant → Data-Available
  → Time-Feasible → IN-SCOPE (the actual plan)

FEASIBILITY CHECK:
  [ ] Data exists at right granularity?
  [ ] Method achieves needed rigor?
  [ ] Timeline realistic?
  [ ] Answer will actually change a decision?

"SHOULD WE LAUNCH?" DECOMPOSITION:
  1. SIZING:        How big is the opportunity?
  2. EVIDENCE:      What does the data show?
  3. RISK:          What could go wrong?
  4. COST:          What does it take?
  5. REVERSIBILITY: Can we undo it?

COMMUNICATION HIERARCHY:
  Executive:  1 sentence + impact number
  PM:         Recommendation + tradeoffs + segments
  Engineering: Technical approach + requirements
  Finance:    ROI + scenarios + confidence intervals

SAYING NO CONSTRUCTIVELY:
  1. Acknowledge the underlying need
  2. Explain why the specific ask is problematic
  3. Propose alternative that serves same need
  4. Get alignment

MATURITY LEVELS:
  Junior:  Executes literal request
  Mid:     Clarifies, then executes well
  Senior:  Translates, scopes, recommends, influences

+============================================================+
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 44 of 45*
