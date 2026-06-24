# 🎯 Topic 36: Defining KPIs & North Star Metrics

> **Data Science Interview — Deep Dive**
> The "product sense" question that separates senior from junior data scientists. How to define, decompose, and defend metrics that drive product decisions at scale.

---

## Table of Contents

1. [Why This Topic Matters](#why-this-topic-matters)
2. [What Makes a Good Metric](#what-makes-a-good-metric)
3. [North Star Metric: Definition and Examples](#north-star-metric-definition-and-examples)
4. [Input Metrics vs Output Metrics](#input-metrics-vs-output-metrics)
5. [Leading vs Lagging Indicators](#leading-vs-lagging-indicators)
6. [Metric Hierarchy](#metric-hierarchy)
7. [Counter Metrics and Guardrail Metrics](#counter-metrics-and-guardrail-metrics)
8. [Metric Decomposition Trees](#metric-decomposition-trees)
9. [The HEART Framework](#the-heart-framework)
10. [Handling Metric Conflicts](#handling-metric-conflicts)
11. [Interview Scenario Walkthroughs](#interview-scenario-walkthroughs)
12. [Framework for Answering Metric Questions](#framework-for-answering-metric-questions)
13. [Interview Talking Points](#interview-talking-points)
14. [Common Interview Mistakes](#common-interview-mistakes)
15. [Rapid-Fire Q&A](#rapid-fire-qa)
16. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## Why This Topic Matters

Defining KPIs and North Star metrics is arguably the **highest-signal question** in a data science interview. It tests:

- **Product intuition** — Can you think like a PM while reasoning like a statistician?
- **Business acumen** — Do you understand what drives revenue, growth, and retention?
- **Structured thinking** — Can you break complex goals into measurable components?
- **Trade-off reasoning** — How do you handle metric conflicts and unintended consequences?

At Meta, Google, Netflix, Uber, and Spotify, data scientists are expected to own the metric layer. You propose metrics, defend them, detect when they're being gamed, and sunset them when they no longer serve the product.

---

## What Makes a Good Metric

A metric is only useful if it passes the **SMART-G** test (extended SMART framework for metrics):

| Property | Definition | Bad Example | Good Example |
|----------|-----------|-------------|--------------|
| **Measurable** | Can be quantified with available data | "User happiness" | NPS score, CSAT rating |
| **Actionable** | Team can move the needle through their work | Total GDP | Feature adoption rate |
| **Relevant** | Directly connected to business objective | Page load time for a social app's growth team | DAU/MAU ratio for a social app's growth team |
| **Timely** | Updates frequently enough to inform decisions | Annual revenue (for weekly sprints) | Weekly active users |
| **Not Gameable** | Resistant to manipulation or Goodhart's Law | Total clicks (incentivizes clickbait) | Time spent reading after click |
| **Sensitive** | Moves detectably when product changes | Total registered users (never decreases) | L7 active users |

### The Goodhart's Law Problem

> "When a measure becomes a target, it ceases to be a good measure."

**Real examples of gaming:**
- YouTube optimized for **watch time** → creators made artificially long videos with filler
- Facebook optimized for **engagement** → algorithm amplified outrage content
- Uber optimized for **driver acceptance rate** → penalized drivers unfairly in low-demand areas

**Mitigation:** Always pair a primary metric with guardrail metrics that detect degenerate behavior.

---

## North Star Metric: Definition and Examples

### Definition

A **North Star Metric (NSM)** is the single metric that best captures the core value your product delivers to customers. It serves as the focal point for the entire company's product strategy.

**Characteristics of a good NSM:**
1. Reflects customer value received (not just company revenue)
2. Correlates with long-term business success
3. All teams can influence it through their specific levers
4. Simple enough to communicate across the organization

### North Star Metrics by Company

| Company | North Star Metric | Why It Works |
|---------|------------------|--------------|
| **Meta (Facebook)** | Daily Active Users (DAU) | Captures habitual engagement with platform value |
| **Meta (Instagram)** | Time spent per DAU | Measures depth of engagement with content |
| **Netflix** | Monthly viewing hours per subscriber | Reflects content value delivery, predicts retention |
| **Spotify** | Time spent listening per user | Captures core audio streaming value |
| **Uber (Rides)** | Completed trips per week | Measures marketplace liquidity and rider satisfaction |
| **Uber (Eats)** | Gross bookings | Captures total food delivery marketplace value |
| **Google (Search)** | Queries per user per day | Reflects trust and habit formation |
| **Airbnb** | Nights booked | Captures marketplace value for hosts and guests |
| **Slack** | Messages sent per team per day | Indicates communication habit formation |
| **DoorDash** | First orders per new cohort (growth); Orders per active user (retention) | Distinguishes growth vs retention phases |
| **Pinterest** | Weekly active pinners | Reflects curation and discovery behavior |
| **LinkedIn** | Monthly active sessions with meaningful interactions | Captures professional network value |
| **YouTube** | Responsible watch time | Balances engagement with content quality |
| **Duolingo** | Daily Active Learners (DAL) | Captures habit formation in learning |

### When to Change Your North Star

- Company shifts from **growth** to **monetization** phase
- Product expands to new verticals (Uber: rides → eats → freight)
- External factors make the metric misleading (e.g., COVID inflating streaming hours)

---

## Input Metrics vs Output Metrics

| Dimension | Input Metrics | Output Metrics |
|-----------|--------------|----------------|
| **Definition** | Actions teams can directly control | Results of those actions |
| **Timeframe** | Short-term, daily/weekly | Medium to long-term |
| **Controllability** | High — teams own these levers | Low — influenced by many factors |
| **Example (Uber)** | Driver supply hours, ETA accuracy, surge pricing frequency | Completed trips, revenue, rider retention |
| **Example (Netflix)** | Content catalog size, recommendation accuracy, load time | Viewing hours, subscriber retention, NPS |
| **Use in planning** | Set targets for teams | Set targets for the company |

### The Input-Output Chain

```
Input Metrics → Intermediate Metrics → Output Metrics
     ↓                    ↓                    ↓
 (Controllable)      (Observable)         (Goal)

Example (Spotify):
Playlist updates → Discovery sessions → Time spent listening → Subscriber retention
```

**Key insight for interviews:** When asked "how would you improve X?", always identify the input metrics (levers) that drive the output metric (goal). This shows you think in terms of actionable strategy, not just measurement.

---

## Leading vs Lagging Indicators

| Aspect | Leading Indicators | Lagging Indicators |
|--------|-------------------|-------------------|
| **Timing** | Predict future outcomes | Confirm past outcomes |
| **Speed** | Change quickly | Change slowly |
| **Actionability** | High — can intervene early | Low — outcome already happened |
| **Certainty** | Less certain (predictive) | More certain (confirmed) |
| **Examples** | Feature adoption rate, NPS, onboarding completion | Revenue, churn rate, LTV |

### Real-World Pairing

| Lagging Indicator | Leading Indicator(s) |
|-------------------|---------------------|
| Subscriber churn (Netflix) | Viewing hours decline, title abandonment rate, login frequency drop |
| Revenue decline (Uber) | Completed trip frequency drop, new rider signup rate, driver supply hours |
| DAU decline (Instagram) | Story creation rate drop, DM sent frequency drop, session frequency drop |
| Conversion drop (Airbnb) | Search-to-view ratio decline, saved listings decline, message response time increase |

**Interview tip:** Always propose at least one leading indicator. It shows you think about **early warning systems**, not just retrospective reporting.

---

## Metric Hierarchy

### The Four-Level Metric Pyramid

```
                    ┌─────────────────┐
                    │   NORTH STAR    │  ← Company-level, single metric
                    │  (1 metric)     │
                    └────────┬────────┘
                             │
                 ┌───────────┼───────────┐
                 ▼           ▼           ▼
          ┌──────────┐ ┌──────────┐ ┌──────────┐
          │ L1 Metric│ │ L1 Metric│ │ L1 Metric│  ← Team-level, 3-5 metrics
          └─────┬────┘ └─────┬────┘ └─────┬────┘
                │             │             │
           ┌────┼────┐  ┌────┼────┐  ┌────┼────┐
           ▼    ▼    ▼  ▼    ▼    ▼  ▼    ▼    ▼
         ┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐
         │L2 ││L2 ││L2 ││L2 ││L2 ││L2 ││L2 ││L2 │  ← Feature-level
         └───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        ┌──────────┐┌──────────┐┌──────────┐
        │Operational││Operational││Operational│  ← Debugging/monitoring
        └──────────┘└──────────┘└──────────┘
```

### Concrete Example: Instagram

| Level | Metrics | Owner |
|-------|---------|-------|
| **North Star** | Time spent per DAU | VP of Product |
| **L1** | Stories engagement rate, Reels watch time, Feed scroll depth, DM messages sent | Team Leads |
| **L2** | Reels completion rate, Reels creation rate, Reels share rate, Audio usage in Reels | Feature PMs |
| **Operational** | Video encode latency, CDN cache hit rate, recommendation model freshness | Engineering |

### Why the Hierarchy Matters

1. **Accountability** — Each team knows which metric they own
2. **Debugging** — When North Star drops, trace down the tree to find the cause
3. **Prioritization** — Invest in L2 metrics with highest leverage on L1
4. **Alignment** — All teams row in the same direction

---

## Counter Metrics and Guardrail Metrics

### Counter Metrics

A **counter metric** is tracked to ensure a primary metric improvement doesn't come at an unacceptable cost.

| Primary Metric | Counter Metric | Why |
|----------------|---------------|-----|
| Notification CTR | Unsubscribe rate | Aggressive notifications boost CTR but annoy users |
| Ads revenue per user | User session duration | Too many ads drive users away |
| Content recommendation engagement | Content diversity score | Echo chambers increase engagement short-term |
| Feature adoption rate | Support ticket volume | Confusing features get "adopted" but cause issues |
| Signup conversion rate | 7-day retention | Dark patterns boost signup but users don't stay |

### Guardrail Metrics

**Guardrails** are hard constraints that must not be violated, regardless of primary metric improvement.

| Category | Guardrail Examples |
|----------|-------------------|
| **User Safety** | Hate speech prevalence < 0.01%, Minor safety violations = 0 |
| **Platform Health** | P99 latency < 500ms, Error rate < 0.1%, Crash rate < 0.5% |
| **Business Sustainability** | CAC < LTV, Gross margin > 30%, Revenue per user not declining |
| **Regulatory** | GDPR compliance, Data deletion SLA < 30 days |
| **Fairness** | Metric disparity across demographic groups < threshold |

### Interview Framework

When proposing any metric, always state:
1. **Primary metric** — What you're optimizing
2. **Counter metric** — What you're watching doesn't degrade
3. **Guardrail** — Hard line that triggers a rollback if crossed

---

## Metric Decomposition Trees

### The Power of Multiplicative Decomposition

Decomposition trees let you express a high-level metric as a **product or sum** of sub-metrics, enabling precise diagnosis and targeted optimization.

### Revenue Decomposition

```
Revenue = Users × Conversion Rate × Average Order Value × Purchase Frequency

         Revenue
            │
    ┌───────┼───────────────┐
    │       │               │
  Users   Conv Rate     Revenue/User
    │       │               │
 ┌──┴──┐  ┌┴────┐     ┌────┴────┐
New  Return  CTR  Checkout  AOV  Frequency
Users Users       Rate
```

### Engagement Decomposition (Social Platform)

```
Total Time Spent = DAU × Sessions per DAU × Time per Session

DAU = New Users + Returning Users - Churned Users

Sessions per DAU = f(Notifications effectiveness, Content freshness, Habit strength)

Time per Session = f(Content quality, Feed algorithm, Session depth)
```

### Marketplace Decomposition (Uber/Airbnb)

```
Gross Bookings = Trips × Average Trip Value

Trips = Supply × Demand × Match Rate

Supply = Active Drivers × Online Hours per Driver
Demand = Active Riders × Request Frequency
Match Rate = f(ETA, Surge pricing, Cancellation rate)
```

### E-Commerce Funnel Decomposition

```
Revenue = Visitors × Browse-to-Cart × Cart-to-Purchase × AOV

Visitors = Organic + Paid + Direct + Referral
Browse-to-Cart = f(Search relevance, Recommendation quality, Product page UX)
Cart-to-Purchase = f(Checkout friction, Payment options, Shipping cost/speed)
AOV = f(Cross-sell effectiveness, Bundle offers, Price optimization)
```

### How to Use in Interviews

1. Start with the **highest-level metric** the interviewer asks about
2. Decompose it into **2-4 components** (multiplicative if possible)
3. Identify which component has the **most room for improvement**
4. Propose the experiment targeting that specific lever

---

## The HEART Framework

### Overview

Developed by Google's UX research team, **HEART** provides a structured way to define metrics for any product or feature across five dimensions:

| Dimension | What It Measures | Signal Type |
|-----------|-----------------|-------------|
| **H** — Happiness | User satisfaction, perceived ease of use | Attitudinal (surveys) |
| **E** — Engagement | Depth and frequency of interaction | Behavioral |
| **A** — Adoption | New users or new use of a feature | Behavioral |
| **R** — Retention | Users who come back over time | Behavioral |
| **T** — Task Success | Efficiency and effectiveness of user tasks | Behavioral |

### HEART Applied: Google Maps Navigation

| Dimension | Goal | Signal | Metric |
|-----------|------|--------|--------|
| **Happiness** | Users trust navigation | Survey after trip | % rating "accurate" 4+/5 |
| **Engagement** | Users rely on nav daily | Navigation sessions | Avg nav sessions per active user per week |
| **Adoption** | New users try navigation | First navigation event | % of new installers who navigate within 7 days |
| **Retention** | Users keep navigating | Return navigation | % of navigators active again in next 28 days |
| **Task Success** | Users reach destination efficiently | Arrival events | % of trips completed vs abandoned; avg reroutes per trip |

### HEART Applied: Spotify Discover Weekly

| Dimension | Goal | Signal | Metric |
|-----------|------|--------|--------|
| **Happiness** | Users enjoy recommendations | Saves, skips | Save rate per recommendation; skip rate within 30s |
| **Engagement** | Users listen to full playlist | Listening events | % of playlist listened; avg listen duration |
| **Adoption** | Users try Discover Weekly | First listen event | % of WAU who listen to Discover Weekly at least once |
| **Retention** | Users come back weekly | Return listens | % of listeners who return next week |
| **Task Success** | Users find new music to add | Library additions | Songs added to library from Discover Weekly per user |

### When to Use HEART

- When the interviewer asks a broad "how would you measure success for X?"
- When you need to **show breadth** before diving deep into one dimension
- When the product spans multiple user journeys

**Interview tip:** Mention HEART by name, apply 2-3 dimensions relevant to the feature, then focus on the one most aligned with the product's current strategic priority.

---

## AARRR (Pirate Metrics) Framework

The **AARRR framework** (Dave McClure) maps the entire user lifecycle into 5 stages — useful for growth-stage products and startups:

| Stage | Question | Example Metric |
|-------|----------|---------------|
| **A**cquisition | How do users find us? | New signups, app installs, landing page visits |
| **A**ctivation | Do they have a great first experience? | Completed onboarding, first action taken, "aha moment" reached |
| **R**etention | Do they come back? | D7/D30 retention, WAU/MAU ratio |
| **R**evenue | Do they pay? | Conversion to paid, ARPU, LTV |
| **R**eferral | Do they tell others? | Invite sent, viral coefficient (k-factor) |

### When to Use AARRR vs HEART

| AARRR | HEART |
|-------|-------|
| Growth-stage products | Established products |
| Full lifecycle view | Single feature evaluation |
| Business outcomes focus | User experience focus |
| "Where is the funnel leaking?" | "How good is this experience?" |

### Vanity Metrics vs Actionable Metrics

| Vanity Metric | Why It's Dangerous | Actionable Alternative |
|---------------|-------------------|----------------------|
| Total registered users | Only goes up, never down | DAU or MAU |
| Page views | Inflated by bots, auto-refresh | Unique engaged users |
| App downloads | Doesn't mean usage | Activated users (completed onboarding) |
| Total revenue (cumulative) | Always increasing | Revenue per user, MoM growth rate |
| Followers/likes | Doesn't correlate with business value | Engagement rate, conversion from followers |

> **Critical Insight**: A vanity metric makes you feel good but doesn't inform decisions. An actionable metric changes your behavior — if it goes down, you know what to investigate and fix.

**Interview test**: "If this metric doubles, would you change anything?" If no → it's a vanity metric.

---

## Handling Metric Conflicts

### Common Conflict Patterns

| Conflict Type | Example | Resolution Approach |
|---------------|---------|-------------------|
| **Engagement vs Revenue** | More ads increase revenue but decrease time spent | Optimize for long-term LTV; cap ad load with diminishing returns model |
| **Growth vs Quality** | Lowering onboarding friction increases signups but reduces activation | Gate quality at a downstream checkpoint (7-day retention) |
| **Speed vs Accuracy** | Faster recommendations reduce latency but lower relevance | A/B test to find the Pareto-optimal point |
| **Short-term vs Long-term** | Aggressive notifications boost DAU now but increase unsubscribes | Use holdout groups to measure 90-day impact |
| **User vs Business** | Free tier engagement up, but paid conversion flat | Redefine success to include upgrade path metrics |

### Resolution Framework (for Interviews)

**Step 1: Acknowledge the tension**
> "This is a classic engagement-revenue trade-off..."

**Step 2: Define the time horizon**
> "In the short term, we'd see X. But over 6 months, Y is more likely..."

**Step 3: Propose a unified metric or constraint**
> "I'd optimize for engagement subject to the constraint that revenue per user doesn't drop more than 5%."

**Step 4: Recommend measurement approach**
> "We'd run a long-term holdout experiment to measure the true long-term effect."

### The OEC Approach (Overall Evaluation Criterion)

When metrics conflict, some companies define a **weighted composite metric**:

```
OEC = w1 × Metric_1 + w2 × Metric_2 + ... + wn × Metric_n

Example (Uber):
OEC = 0.4 × Completed Trips + 0.3 × Revenue per Trip + 0.2 × Rider NPS + 0.1 × Driver Satisfaction
```

**Caveat:** Weights are subjective and require leadership alignment. In interviews, acknowledge this trade-off explicitly.

---

## Interview Scenario Walkthroughs

### Scenario 1: "Define metrics for Instagram Reels"

**Step 1: Clarify**
> "Is the goal to measure the success of Reels as a feature overall, or a specific change to Reels? Are we in the growth phase (acquiring Reels users) or the maturity phase (deepening engagement)?"

**Step 2: Identify stakeholders and their goals**
- **Viewers:** Want entertaining, relevant short videos
- **Creators:** Want reach, engagement, and monetization
- **Instagram (platform):** Want to increase time spent, compete with TikTok, and retain users

**Step 3: Propose metric hierarchy**

| Level | Metric | Rationale |
|-------|--------|-----------|
| **North Star** | Daily Reels watch time per DAU | Captures core value delivery for viewers |
| **L1 — Creator** | Monthly active Reels creators | Healthy supply drives content quality |
| **L1 — Viewer** | Reels sessions per DAU | Measures habit formation |
| **L1 — Retention** | D7 retention of Reels viewers | Leading indicator of long-term stickiness |
| **L2** | Reels completion rate (% watched >75%) | Content quality signal |
| **L2** | Reels share rate | Viral growth and content resonance |
| **L2** | Audio reuse rate | Trend participation and creator ecosystem health |
| **Counter** | Feed time spent (non-Reels) | Ensure Reels doesn't cannibalize core feed |
| **Guardrail** | Reported content rate | Safety and content quality floor |

**Step 4: State trade-offs**
> "Watch time is the primary metric, but I'd watch completion rate as a quality signal — it's possible to inflate watch time with longer videos that users don't actually enjoy."

---

### Scenario 2: "Define metrics for Uber Eats search"

**Step 1: Clarify**
> "Is this the search experience for finding restaurants, or also for searching within a restaurant's menu? I'll focus on restaurant search as that's the primary use case."

**Step 2: Apply HEART framework selectively**

| Dimension | Metric | Definition |
|-----------|--------|------------|
| **Task Success** | Search-to-order conversion rate | % of searches that result in a completed order within the session |
| **Engagement** | Searches per session | How often users search vs browse categories |
| **Happiness** | Zero-result rate | % of searches returning no results (inverse satisfaction) |

**Step 3: Full metric set**

| Level | Metric | Rationale |
|-------|--------|-----------|
| **Primary** | Search-to-order conversion | Directly measures if search helped users find what they want |
| **Secondary** | Time from search to order | Efficiency — faster is better |
| **Secondary** | Result click-through rate (position-weighted) | Relevance of ranking |
| **Counter** | Order cancellation rate post-search | Ensure relevance isn't just CTR-driven |
| **Guardrail** | Average delivery time of ordered restaurants | Don't surface far-away restaurants just because they match |
| **Operational** | P50 search latency, query autocomplete usage rate | Technical health |

**Step 4: Decomposition**
```
Search Value = Searches × Relevance × Conversion × Order Value
                 ↓           ↓            ↓            ↓
           (Volume)    (Ranking)    (UX/Price)    (Menu depth)
```

---

### Scenario 3: "How would you measure success of a new notification feature?"

**Step 1: Clarify**
> "What type of notification — push, in-app, or email? What's the goal — re-engagement, transactional, or social? Let me assume it's a new push notification type aimed at re-engaging lapsed users."

**Step 2: Define success at multiple time horizons**

| Time Horizon | Metric | Target |
|-------------|--------|--------|
| **Immediate (hours)** | Notification open rate | > 5% (benchmark for re-engagement pushes) |
| **Short-term (days)** | Reactivation rate (opened app within 24h) | > 15% of notified lapsed users |
| **Medium-term (weeks)** | D7 retention of reactivated users | > 30% return again within 7 days |
| **Long-term (months)** | Incremental DAU lift (treatment vs control) | Statistically significant positive lift |

**Step 3: Counter metrics (critical for notifications)**

| Counter Metric | Threshold | Action if Breached |
|---------------|-----------|-------------------|
| Notification opt-out rate | < 0.5% per campaign | Reduce frequency immediately |
| App uninstall rate | No significant lift vs control | Halt experiment |
| Other notification CTR | No significant decline | Ensure not creating notification fatigue |

**Step 4: Experimentation design**
> "I'd use a holdout group (10% of eligible users receive no notification) to measure true incremental impact. I'd also segment by lapse duration — users inactive 7 days vs 30 days likely respond differently."

---

## Framework for Answering Metric Questions

### The CDPT Framework (Clarify → Decompose → Prioritize → Trade-offs)

```
┌─────────────────────────────────────────────────────────────┐
│  Step 1: CLARIFY                                            │
│  - What's the product/feature?                              │
│  - What lifecycle stage? (launch, growth, maturity)         │
│  - Who are the users? (sides of marketplace, segments)      │
│  - What's the business objective? (growth, retention, $)    │
├─────────────────────────────────────────────────────────────┤
│  Step 2: DECOMPOSE                                          │
│  - Break the goal into measurable components                │
│  - Apply HEART or metric tree                               │
│  - Identify 5-8 candidate metrics                           │
├─────────────────────────────────────────────────────────────┤
│  Step 3: PRIORITIZE                                         │
│  - Select 1 primary metric (North Star for this feature)    │
│  - Select 2-3 secondary metrics                             │
│  - Identify 1-2 counter/guardrail metrics                   │
│  - Justify WHY this primary metric over others              │
├─────────────────────────────────────────────────────────────┤
│  Step 4: TRADE-OFFS                                         │
│  - What could go wrong with this metric?                    │
│  - How might it be gamed?                                   │
│  - What's the sensitivity and latency?                      │
│  - Would you change the metric as the feature matures?      │
└─────────────────────────────────────────────────────────────┘
```

### Time Budget for a 30-Minute Interview

| Phase | Time | What to Do |
|-------|------|-----------|
| Clarify | 2-3 min | Ask 2-3 questions, state assumptions |
| Decompose | 5-7 min | Draw the metric tree, list candidates |
| Prioritize | 5-7 min | Defend your primary metric choice |
| Trade-offs | 3-5 min | Counter metrics, gaming risks, evolution |
| Follow-ups | 5-10 min | Interviewer drills into your choices |

---

## Interview Talking Points

> **On choosing a North Star metric:**
> "I think about three criteria when selecting a North Star: first, does it capture the core value the user receives — not just what benefits the company? Second, is it sensitive enough that our team's work can move it within a quarter? And third, is it resistant to gaming — if someone optimized purely for this number, would the product genuinely get better? For this product, I'd choose [metric] because it satisfies all three: it reflects user value by measuring [explanation], it's actionable because our team directly controls [lever], and it's hard to game because [reason]."

> **On handling metric conflicts:**
> "When I see engagement going up but revenue going down, my first instinct isn't to pick one — it's to understand the mechanism. I'd segment the data: is this driven by a specific user cohort, a particular feature, or a temporal effect? Then I'd ask what the long-term relationship looks like. Often, short-term engagement gains compound into revenue over a longer horizon — Netflix proved this by prioritizing viewing hours over immediate upsell. I'd propose running a 90-day holdout to measure the true long-run revenue impact of the engagement-optimizing change."

> **On defending against 'why not just use revenue?':**
> "Revenue is the ultimate output metric, but it's a terrible day-to-day optimization target for three reasons: it's a lagging indicator with high latency, it's influenced by too many external factors (pricing, macro economy, seasonality) for any single team to own, and optimizing revenue directly often leads to extractive short-term decisions — more ads, higher prices, dark patterns — that destroy long-term value. I prefer a metric that's a proven leading indicator of revenue but directly reflects user value."

> **On metric decomposition in practice:**
> "When I join a new team, the first thing I do is build the metric tree. I take the team's primary goal, decompose it multiplicatively into components, and then map each component to a team or initiative. This makes prioritization trivial — you invest in the component with the most room for improvement and the most feasible lever. For example, if we decompose Uber Eats revenue into orders times AOV, and we see that order frequency has plateaued while AOV has headroom through better cross-selling, that tells us exactly where to invest."

---

## Common Interview Mistakes

| ❌ Mistake | ✅ Better Approach |
|-----------|-------------------|
| Jumping to metrics without clarifying the product context | Spend 2-3 minutes asking about lifecycle stage, user segments, and business goals |
| Proposing only one metric | Always propose a primary + 2-3 secondary + 1-2 counter metrics |
| Choosing revenue as the North Star | Revenue is an output; choose a metric that reflects user value and leads to revenue |
| Ignoring counter metrics / gaming risks | For every primary metric, state how it could be gamed and what you'd watch |
| Listing metrics without hierarchy | Organize as North Star → L1 → L2 → Operational |
| Using vanity metrics (total signups, page views) | Use ratio metrics (DAU/MAU, retention rate, conversion rate) that normalize for growth |
| Forgetting about marketplace dynamics | For two-sided marketplaces, define metrics for BOTH sides |
| Not connecting metrics to experiments | End by saying "I'd validate this metric choice by running an A/B test on X" |
| Treating all users as one segment | Acknowledge that metrics may differ by user cohort (new vs returning, power vs casual) |
| Proposing metrics you can't actually measure | Always sanity-check: "We can track this because the event fires when..." |

---

## Rapid-Fire Q&A

**Q1: What's the difference between a KPI and a North Star metric?**
> A KPI is any key performance indicator tracked by a team. A North Star is specifically the ONE metric that captures the product's core value proposition. All KPIs should ladder up to the North Star.

**Q2: How many metrics should a team track?**
> One primary metric, 3-5 secondary/supporting metrics, and 1-2 guardrails. More than 8 total metrics creates "metric fatigue" where nothing gets optimized.

**Q3: When should you change your North Star metric?**
> When the company shifts strategic phase (growth → monetization), when the product fundamentally changes (new verticals), or when the metric becomes gameable and no longer correlates with true user value.

**Q4: How do you handle a metric that's too slow to move?**
> Find a leading indicator that's proven to correlate with the slow metric. Example: if subscriber retention takes 6 months to observe, use 14-day viewing frequency as a proxy.

**Q5: What's an OEC and when would you use one?**
> Overall Evaluation Criterion — a weighted composite metric used to make A/B test decisions when multiple metrics matter. Used when you need a single go/no-go signal but care about multiple outcomes.

**Q6: How do you set metric targets?**
> Three approaches: (1) Historical trend + improvement rate, (2) Benchmark against competitors or best-in-class, (3) Work backward from business goals (revenue target → required conversion rate).

**Q7: What's a ratio metric and why prefer it?**
> A ratio metric normalizes a count by a denominator (e.g., revenue per user, messages per DAU). It removes the confound of user growth, making it a purer signal of product improvement.

**Q8: How do you validate that a metric actually matters?**
> Retrospective correlation analysis: does the metric predict long-term outcomes (retention, LTV)? Run a causal analysis: when past experiments moved this metric, did downstream business outcomes also move?

**Q9: What's the difference between a metric and a goal?**
> A goal is a desired outcome ("increase user retention"). A metric is the specific, measurable quantity that operationalizes that goal ("28-day retention rate of new users from 35% to 40%").

**Q10: How do you present metric recommendations to stakeholders?**
> Start with the business objective, show the decomposition tree, present the recommended primary metric with justification, show what it would have looked like historically, and propose the first experiment to validate it.

---

## Summary Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════════╗
║                   KPIs & NORTH STAR METRICS CHEAT SHEET                 ║
╠══════════════════════════════════════════════════════════════════════════╣
║                                                                          ║
║  GOOD METRIC PROPERTIES:  Measurable | Actionable | Relevant |          ║
║                            Timely | Not Gameable | Sensitive             ║
║                                                                          ║
║  METRIC HIERARCHY:                                                       ║
║    North Star (1) → L1 Metrics (3-5) → L2 Metrics → Operational         ║
║                                                                          ║
║  ALWAYS INCLUDE:                                                         ║
║    • Primary metric (what you optimize)                                  ║
║    • Counter metric (what you protect)                                   ║
║    • Guardrail (hard constraint, triggers rollback)                      ║
║                                                                          ║
║  DECOMPOSITION FORMULA:                                                  ║
║    Revenue = Users × Conversion × AOV × Frequency                       ║
║    Engagement = DAU × Sessions/DAU × Time/Session                        ║
║    Marketplace = Supply × Demand × Match Rate × Value/Match              ║
║                                                                          ║
║  HEART FRAMEWORK (Google):                                               ║
║    Happiness | Engagement | Adoption | Retention | Task Success           ║
║                                                                          ║
║  INTERVIEW FRAMEWORK (CDPT):                                             ║
║    Clarify → Decompose → Prioritize → Trade-offs                         ║
║                                                                          ║
║  METRIC CONFLICTS — RESOLUTION:                                          ║
║    1. Acknowledge the tension                                            ║
║    2. Define the time horizon                                            ║
║    3. Propose unified metric or constraint                               ║
║    4. Recommend long-term holdout experiment                             ║
║                                                                          ║
║  KEY DISTINCTIONS:                                                       ║
║    • Input (controllable) vs Output (goal)                               ║
║    • Leading (predictive) vs Lagging (confirmatory)                      ║
║    • Ratio metrics > Count metrics (normalize for growth)                ║
║                                                                          ║
║  RED FLAGS — NEVER DO:                                                   ║
║    ✗ Choose revenue as North Star                                        ║
║    ✗ Propose only one metric                                             ║
║    ✗ Ignore gaming/Goodhart's Law                                        ║
║    ✗ Skip the clarification step                                         ║
║    ✗ Use vanity metrics (total signups, total page views)                ║
║                                                                          ║
╚══════════════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 36 of 45*
