# 🎯 Topic 39: Metric Decomposition and Debugging Drops

> *"CRITICAL INTERVIEW TOPIC. 'Revenue dropped 10% week-over-week — what would you investigate?' is the most common product analytics interview question. Your framework for systematic diagnosis is what separates senior from junior analysts."*

---

## 📑 Table of Contents

1. [Why This Matters So Much](#why-this-matters-so-much)
2. [Metric Trees and Decomposition](#metric-trees-and-decomposition)
3. [The Systematic Debugging Framework](#the-systematic-debugging-framework)
4. [Step-by-Step Root Cause Analysis](#step-by-step-root-cause-analysis)
5. [Time-Based vs Segment-Based Decomposition](#time-based-vs-segment-based-decomposition)
6. [Interview Walkthrough: Revenue Dropped 10%](#interview-walkthrough-revenue-dropped-10)
7. [Communication Framework for Findings](#communication-framework-for-findings)
8. [SQL Patterns for Debugging](#sql-patterns-for-debugging)
9. [Interview Talking Points](#interview-talking-points)
10. [Common Mistakes](#common-mistakes)
11. [Rapid-Fire Q&A](#rapid-fire-qa)
12. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Why This Matters So Much

This question tests:
1. **Structured thinking** — Can you be systematic under pressure?
2. **Business intuition** — Do you know what drives metrics?
3. **Technical depth** — Can you actually query and investigate?
4. **Communication** — Can you explain your findings clearly?

It appears in interviews at: Google, Meta, Amazon, Netflix, Uber, Airbnb, Spotify, and virtually every data science role involving product analytics.

> **Critical Insight:** Interviewers are NOT looking for the right answer (there often isn't one). They're evaluating your PROCESS. A structured, systematic approach that considers multiple hypotheses and eliminates them methodically will always beat a lucky guess.

---

## Metric Trees and Decomposition

### The Core Idea

Every metric can be decomposed into a product of sub-metrics. When the metric changes, exactly one (or more) of its components must have changed.

### Common Metric Trees

**Revenue:**
```
Revenue = Users x Conversion Rate x Average Order Value (AOV)
        = Users x Orders/User x Revenue/Order
        = Traffic x CTR x Add-to-Cart Rate x Checkout Rate x AOV
```

**Engagement:**
```
DAU = New Users + Returning Users
    = MAU x (DAU/MAU ratio)
    = Sessions / (Sessions per User)

Total Time Spent = DAU x Sessions/User x Time/Session
```

**Marketplace:**
```
GMV = Buyers x Orders/Buyer x Items/Order x Price/Item
    = Supply (Listings) x Demand (Views/Listing) x Conversion x AOV
```

**Subscriptions:**
```
MRR = Subscribers x ARPU
    = (Existing - Churned + New + Reactivated) x (Base Price + Upgrades - Downgrades)

MRR Change = New MRR + Expansion - Contraction - Churn
```

**Ad Revenue:**
```
Ad Revenue = Impressions x CPM / 1000
           = DAU x Sessions/User x Pages/Session x Ads/Page x CPM/1000
           = (Reach) x (Engagement Depth) x (Monetization Density) x (Ad Value)
```

### Building Your Own Metric Tree

```
Step 1: Start with the top-level metric
Step 2: Ask "what mathematical components make this up?"
Step 3: For each component, ask "can this be further decomposed?"
Step 4: Stop when you reach directly measurable/actionable quantities
```

> **Critical Insight:** The metric tree is your debugging MAP. When a metric drops, trace through the tree to find which branch(es) moved. This converts an overwhelming "revenue dropped" into a manageable "conversion rate on mobile in Germany dropped."

---

## The Systematic Debugging Framework

### The 6-Question Framework

When told "Metric X dropped Y%," work through these in order:

```
1. IS THE DATA CORRECT?
   - Logging issues? Pipeline delays? Duplicate removal changed?
   - Did the metric definition change?
   - Is this a data freshness / ETL failure?

2. IS THIS EXPECTED?
   - Seasonality? (Day of week, holiday, end of month)
   - Planned changes? (Price increase, feature removal, marketing pause)
   - External events? (Competitor launch, news, weather)

3. WHEN DID IT START?
   - Sudden cliff or gradual decline?
   - Correlates with a deploy/release?
   - Started at a specific hour or date?

4. WHERE IS IT HAPPENING?
   - One platform? (iOS vs Android vs Web)
   - One geography? (US, EU, APAC)
   - One user segment? (New vs existing, paid vs organic)
   - One product/category?

5. WHAT COMPONENT CHANGED?
   - Decompose the metric tree
   - Which sub-metric drove the change?
   - Is it volume (fewer users) or rate (lower conversion)?

6. WHY DID THAT COMPONENT CHANGE?
   - Feature change? Bug? Policy change?
   - Competitive action? Market shift?
   - Correlation with other metrics?
```

### Decision Tree Visualization

```
Metric Dropped
    |
    +-- Is data valid?
    |       |
    |       +-- NO --> Fix data pipeline / logging
    |       +-- YES
    |              |
    +-- Is it expected (seasonality/planned)?
    |       |
    |       +-- YES --> Document, no action needed
    |       +-- NO
    |              |
    +-- Sudden or gradual?
    |       |
    |       +-- SUDDEN --> Check releases, outages, external events
    |       +-- GRADUAL --> Check trends, market shifts, user mix
    |              |
    +-- Which segment?
    |       |
    |       +-- ALL segments --> Systemic issue (infra, algo, pricing)
    |       +-- ONE segment --> Segment-specific issue
    |              |
    +-- Which metric tree component?
            |
            +-- Volume (fewer users) --> Acquisition or retention problem
            +-- Rate (lower conversion) --> Product or experience problem
            +-- Value (lower AOV) --> Pricing or mix shift
```

---

## Step-by-Step Root Cause Analysis

### Step 1: Validate the Data

```sql
-- Check for logging gaps
SELECT DATE(event_timestamp) AS event_date,
       COUNT(*) AS event_count
FROM events
WHERE event_timestamp BETWEEN '2024-01-01' AND '2024-01-31'
GROUP BY DATE(event_timestamp)
ORDER BY event_date;

-- Check for sudden volume drops (pipeline issue)
-- Look for dates with < 80% of expected volume
```

### Step 2: Check Seasonality

```sql
-- Compare same day of week, previous weeks
SELECT 
    DATE(event_timestamp) AS event_date,
    DAYOFWEEK(event_timestamp) AS dow,
    COUNT(DISTINCT user_id) AS dau
FROM events
WHERE event_timestamp >= CURRENT_DATE - INTERVAL '8 weeks'
GROUP BY DATE(event_timestamp), DAYOFWEEK(event_timestamp)
ORDER BY dow, event_date;
```

### Step 3: Segment Decomposition

```sql
-- Break metric by platform
SELECT 
    platform,
    DATE(event_timestamp) AS event_date,
    COUNT(DISTINCT user_id) AS users,
    COUNT(DISTINCT CASE WHEN converted THEN user_id END) AS converters,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN converted THEN user_id END) / 
          COUNT(DISTINCT user_id), 2) AS conversion_rate
FROM events
WHERE event_timestamp >= CURRENT_DATE - INTERVAL '14 days'
GROUP BY platform, DATE(event_timestamp)
ORDER BY platform, event_date;
```

### Step 4: Component Analysis

```sql
-- Revenue decomposition
WITH daily_metrics AS (
    SELECT 
        DATE(order_timestamp) AS order_date,
        COUNT(DISTINCT user_id) AS unique_buyers,
        COUNT(*) AS total_orders,
        SUM(revenue) AS total_revenue,
        AVG(revenue) AS avg_order_value,
        COUNT(*) * 1.0 / COUNT(DISTINCT user_id) AS orders_per_buyer
    FROM orders
    WHERE order_timestamp >= CURRENT_DATE - INTERVAL '30 days'
    GROUP BY DATE(order_timestamp)
)
SELECT 
    order_date,
    unique_buyers,
    orders_per_buyer,
    avg_order_value,
    total_revenue,
    -- Week-over-week changes
    LAG(unique_buyers, 7) OVER (ORDER BY order_date) AS buyers_wow,
    LAG(avg_order_value, 7) OVER (ORDER BY order_date) AS aov_wow,
    ROUND(100.0 * (unique_buyers - LAG(unique_buyers, 7) OVER (ORDER BY order_date)) / 
          LAG(unique_buyers, 7) OVER (ORDER BY order_date), 1) AS buyers_pct_change
FROM daily_metrics
ORDER BY order_date;
```

---

## Time-Based vs Segment-Based Decomposition

### Time-Based Decomposition

Asks: **WHEN** did the change happen?

```
Day-over-Day:   Did it change yesterday? Gradually over weeks?
Hour-over-Hour: Did it happen at a specific time (deploy)?
Week-over-Week: Is this a weekly pattern (seasonality)?
YoY:            Is this a yearly cycle?
```

**Best for:** Identifying sudden changes, correlating with events/deploys

### Segment-Based Decomposition

Asks: **WHERE** in the user/product space did the change happen?

```
Platform:   iOS / Android / Web / API
Geography:  Country / Region / City
User Type:  New / Existing / Paid / Free
Channel:    Organic / Paid / Email / Social
Product:    Category / SKU / Price tier
Device:     Model / OS version / Browser
```

**Best for:** Isolating the root cause to a specific group

### Mix Shift vs Rate Change

A crucial distinction. Revenue can drop because:

```
Rate Change:  Same user mix, but conversion rates dropped
              (Something broke for everyone)

Mix Shift:    Conversion rates unchanged, but the MIX of users shifted
              toward a lower-converting segment
              (More low-intent traffic, fewer high-intent users)
```

```python
# Simpson's Paradox Detection
# Overall conversion dropped, but every segment's conversion is stable!
# --> Mix shifted toward lower-converting segments

segments = ['Mobile', 'Desktop']
for period in ['Last Week', 'This Week']:
    for seg in segments:
        print(f"{period} - {seg}: CVR = {cvr[period][seg]:.1%}, Share = {share[period][seg]:.1%}")

# Last Week - Mobile: CVR = 2.0%, Share = 40%
# Last Week - Desktop: CVR = 5.0%, Share = 60%
# Overall: 3.8%

# This Week - Mobile: CVR = 2.0%, Share = 60%  <-- More mobile!
# This Week - Desktop: CVR = 5.0%, Share = 40%
# Overall: 3.2%  <-- "Drop" is entirely mix shift!
```

> **Critical Insight:** Always check for mix shift before concluding that something is "broken." A viral TikTok video might drive a surge of low-intent mobile traffic, making your conversion rate look terrible while nothing is actually wrong with your product.

---

## Interview Walkthrough: Revenue Dropped 10%

### The Prompt

*"You're a data scientist at an e-commerce company. You come in Monday morning and revenue is down 10% week-over-week. Walk me through how you'd investigate."*

### Model Answer (Structured)

**Opening (Frame the problem):**
"I'd approach this systematically, starting with data validation, then checking for expected causes, then segmenting to isolate the issue. Let me walk through my framework."

**Step 1 — Data Validation (30 seconds):**
"First, I'd verify the data is correct. Is the reporting pipeline fully loaded? Are there any known ETL issues? Did a tracking pixel fail? I'd check event volumes to make sure we're not comparing a partial day."

**Step 2 — Expected Causes (30 seconds):**
"Was there a holiday, planned promotion ending, or known seasonality? If last week had a big sale event and this week didn't, the drop is expected."

**Step 3 — Timing (1 minute):**
"I'd look at hourly or daily granularity. Did it drop suddenly (suggesting a bug or outage) or gradually (suggesting a trend)? If sudden, I'd correlate with deployment timestamps."

**Step 4 — Metric Decomposition (2 minutes):**
"Revenue = Users x Conversion Rate x AOV. I'd decompose:
- Did we have fewer users? (traffic/acquisition problem)
- Did conversion rate drop? (product/UX problem)
- Did AOV drop? (pricing/assortment problem)

Let's say I find that users are stable, AOV is stable, but conversion rate dropped from 3.5% to 3.1%."

**Step 5 — Segment (2 minutes):**
"I'd segment the conversion drop by:
- Platform: Is it iOS, Android, or Web?
- Let's say it's only mobile web.
- Geography: All countries or one?
- Let's say it's global mobile web.
- Funnel stage: Where do users drop off?
- Let's say the add-to-cart to checkout step dropped.

This suggests a mobile web checkout issue."

**Step 6 — Root Cause (1 minute):**
"I'd check if there was a mobile web deploy last week. I'd look at error rates on the checkout page. If there's a new bug causing checkout failures on certain mobile browsers, that's our root cause."

**Step 7 — Quantify and Communicate:**
"The 10% revenue drop is caused by a 12% conversion rate decline on mobile web, specifically at the checkout step, likely due to the v2.4 deploy on Thursday. Estimated daily impact: $150K. Recommendation: rollback and hotfix."

---

## Communication Framework for Findings

### The SCQA Framework

```
S - Situation:  "Revenue dropped 10% WoW last Thursday"
C - Complication: "This is not seasonal — same period last year was flat"
Q - Question:   "What caused it and what should we do?"
A - Answer:     "Mobile checkout bug from Thursday deploy; 
                 rollback would recover ~$150K/day"
```

### The Pyramid Structure for Presenting

```
        ┌─────────────────────────┐
        │   ANSWER (1 sentence)   │  <-- Lead with this
        └───────────┬─────────────┘
                    │
    ┌───────────────┼───────────────┐
    │               │               │
┌───┴───┐   ┌──────┴──────┐   ┌────┴────┐
│Evidence│   │  Evidence   │   │Evidence │  <-- Supporting data
│ Point 1│   │  Point 2    │   │Point 3  │
└───┬────┘   └──────┬──────┘   └────┬────┘
    │               │               │
  Details        Details          Details    <-- Only if asked
```

### Template for Communicating Metric Drops

```
SUBJECT: [Metric] [dropped/spiked] [X%] — Root Cause: [one-liner]

SUMMARY:
- What: [Metric] dropped [X%] between [date] and [date]
- Impact: Estimated $[X]/day in lost [revenue/conversions/etc.]
- Cause: [Root cause in one sentence]
- Status: [Investigating / Resolved / Mitigation in progress]
- Action: [Rollback / Fix deployed / Monitoring]

DETAILS:
1. Timeline: First observed [date/time], caused by [event]
2. Scope: Affects [segment] only — [X%] of total traffic
3. Decomposition: [metric component] drove [Y%] of the drop
4. Evidence: [2-3 data points supporting root cause]

NEXT STEPS:
- [ ] [Immediate action]
- [ ] [Short-term fix]
- [ ] [Prevention for future]
```

---

## SQL Patterns for Debugging

### Compare Periods Side-by-Side

```sql
WITH this_week AS (
    SELECT 
        platform,
        COUNT(DISTINCT user_id) AS users,
        SUM(revenue) AS revenue,
        COUNT(DISTINCT order_id) AS orders
    FROM orders
    WHERE order_date BETWEEN '2024-03-11' AND '2024-03-17'
    GROUP BY platform
),
last_week AS (
    SELECT 
        platform,
        COUNT(DISTINCT user_id) AS users,
        SUM(revenue) AS revenue,
        COUNT(DISTINCT order_id) AS orders
    FROM orders
    WHERE order_date BETWEEN '2024-03-04' AND '2024-03-10'
    GROUP BY platform
)
SELECT 
    COALESCE(t.platform, l.platform) AS platform,
    l.users AS users_lw,
    t.users AS users_tw,
    ROUND(100.0 * (t.users - l.users) / l.users, 1) AS users_pct_change,
    l.revenue AS revenue_lw,
    t.revenue AS revenue_tw,
    ROUND(100.0 * (t.revenue - l.revenue) / l.revenue, 1) AS revenue_pct_change
FROM this_week t
FULL OUTER JOIN last_week l ON t.platform = l.platform;
```

### Identify the Biggest Contributor to a Drop

```sql
-- Which segment contributed most to revenue decline?
WITH segment_revenue AS (
    SELECT 
        country,
        platform,
        SUM(CASE WHEN week = 'this_week' THEN revenue ELSE 0 END) AS rev_this_week,
        SUM(CASE WHEN week = 'last_week' THEN revenue ELSE 0 END) AS rev_last_week
    FROM (
        SELECT *, 
            CASE WHEN order_date >= '2024-03-11' THEN 'this_week' 
                 ELSE 'last_week' END AS week
        FROM orders
        WHERE order_date >= '2024-03-04'
    ) t
    GROUP BY country, platform
)
SELECT 
    country, platform,
    rev_last_week, rev_this_week,
    (rev_this_week - rev_last_week) AS absolute_change,
    ROUND(100.0 * (rev_this_week - rev_last_week) / rev_last_week, 1) AS pct_change,
    -- Contribution to total change
    ROUND(100.0 * (rev_this_week - rev_last_week) / 
          (SUM(rev_this_week) OVER() - SUM(rev_last_week) OVER()), 1) AS contribution_pct
FROM segment_revenue
ORDER BY absolute_change ASC;  -- Most negative first
```

### Funnel Drop-Off Analysis

```sql
WITH funnel AS (
    SELECT 
        DATE(session_start) AS session_date,
        COUNT(DISTINCT session_id) AS sessions,
        COUNT(DISTINCT CASE WHEN viewed_product THEN session_id END) AS viewed,
        COUNT(DISTINCT CASE WHEN added_to_cart THEN session_id END) AS added,
        COUNT(DISTINCT CASE WHEN started_checkout THEN session_id END) AS checkout,
        COUNT(DISTINCT CASE WHEN purchased THEN session_id END) AS purchased
    FROM sessions
    WHERE session_start >= CURRENT_DATE - INTERVAL '14 days'
    GROUP BY DATE(session_start)
)
SELECT 
    session_date,
    sessions,
    ROUND(100.0 * viewed / sessions, 1) AS view_rate,
    ROUND(100.0 * added / viewed, 1) AS atc_rate,
    ROUND(100.0 * checkout / added, 1) AS checkout_rate,
    ROUND(100.0 * purchased / checkout, 1) AS purchase_rate,
    ROUND(100.0 * purchased / sessions, 2) AS overall_conversion
FROM funnel
ORDER BY session_date;
```

---

## Interview Talking Points

> "My approach to debugging metric drops follows a strict hierarchy: validate data first, check for expected causes, then systematically decompose. In a recent case, GMV appeared to drop 8% but the first thing I found was that our data pipeline had a 4-hour delay — the metric caught up by end of day. Always validate before investigating."

> "The most impactful debugging I've done was identifying a mix shift that masqueraded as a conversion rate decline. Our overall conversion dropped 0.5pp, and the product team was scrambling to find bugs. I showed through segment decomposition that every individual segment's conversion was stable — we just had 20% more traffic from a low-intent social campaign. The 'fix' was to either accept the lower blended rate or adjust the campaign targeting."

> "I structure my findings using the pyramid principle — answer first, then supporting evidence. When I reported to leadership, I said: 'Revenue dropped $150K/day due to a mobile checkout bug deployed Thursday. Rollback will restore it within 24 hours.' Then I provided the decomposition evidence only when they asked for details."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Jumping to conclusions ("must be a bug") | Follow systematic framework — validate, expected?, when?, where?, what?, why? |
| Not checking data validity first | ALWAYS verify logging, pipeline health, and metric definition first |
| Looking only at the top-level metric | Decompose into components using the metric tree |
| Forgetting about mix shift (Simpson's Paradox) | Check both segment-level rates AND segment share changes |
| Presenting findings without a clear recommendation | Always end with: impact quantification + recommended action |
| Investigating every segment equally | Use 80/20 — find the largest contributor first, then drill deeper |
| Comparing inappropriate time periods | Account for day-of-week, holidays, and promotional calendars |
| Not considering external factors | Check competitor launches, weather, macro events, news |
| Saying "I'd look at the data" without specifics | Name the exact query/analysis — "I'd segment conversion by platform and compare WoW" |
| Spending too long on one hypothesis | Time-box each hypothesis to 2-3 minutes in an interview, then move on |

---

## Rapid-Fire Q&A

**Q1: What's the first thing you check when a metric drops?**
Data validity — is the pipeline fully loaded? Is there a logging issue? Did the metric definition change?

**Q2: How do you distinguish a bug from seasonality?**
Compare to the same day-of-week in previous weeks and same period last year. If historical patterns show similar dips at this time, it's likely seasonal.

**Q3: What's a metric tree?**
A mathematical decomposition of a metric into multiplicative components. Revenue = Users x Conversion x AOV. Each branch can be further decomposed.

**Q4: What's mix shift and why does it matter?**
When the composition of your user base changes (e.g., more mobile users) causing overall metrics to shift even though per-segment metrics are stable. It's Simpson's Paradox in action.

**Q5: How do you tell if a drop is due to fewer users vs lower conversion?**
Decompose: check if user count changed (volume) or if conversion rate changed (rate). If both, quantify the contribution of each using: total change = volume effect + rate effect + interaction.

**Q6: What's the difference between a sudden cliff and a gradual decline?**
A cliff suggests a discrete event (deploy, outage, policy change). A gradual decline suggests a trend (competition, market shift, degrading user mix).

**Q7: How do you prioritize which segments to investigate first?**
Start with the largest segments by revenue/volume. A 5% drop in a segment that's 80% of revenue matters more than a 50% drop in a 2% segment.

**Q8: What if every segment dropped equally?**
Then it's likely a systemic issue — data problem, site-wide bug, external event affecting everyone, or a change to a core shared component.

**Q9: How do you quantify the contribution of each factor?**
Use additive decomposition: total_change = sum of (segment_share x segment_change) across all segments. Or use Shapley values for complex interactions.

**Q10: How do you communicate findings to non-technical stakeholders?**
Lead with the answer (what happened and what to do), provide one supporting data point, and offer details only if asked. Use the SCQA or pyramid framework.

---

## ASCII Cheat Sheet

```
+============================================================+
|       METRIC DEBUGGING FRAMEWORK — CHEAT SHEET             |
+============================================================+

THE 6 QUESTIONS (IN ORDER):
  1. Is the DATA correct?        (Pipeline, logging, definition)
  2. Is this EXPECTED?           (Seasonality, planned change)
  3. WHEN did it start?          (Sudden vs gradual)
  4. WHERE is it happening?      (Platform, geo, segment)
  5. WHAT component changed?     (Metric tree decomposition)
  6. WHY did it change?          (Root cause)

COMMON METRIC TREES:
  Revenue = Users x CVR x AOV
  Engagement = DAU x Sessions/User x Time/Session
  Ad Rev = Impressions x CPM/1000
  GMV = Buyers x Frequency x Basket Size x Price

DECOMPOSITION TYPES:
  Time-Based:    When did it change? (hourly/daily/weekly)
  Segment-Based: Where? (platform/geo/user type/channel)
  Funnel-Based:  What step? (view/click/add/checkout/buy)
  Component:     Which factor? (volume/rate/value)

MIX SHIFT vs RATE CHANGE:
  Rate Change:  Segment conversion rates dropped
  Mix Shift:    Segment rates stable, but proportions changed
  TEST:         Check each segment independently!

COMMUNICATION TEMPLATE:
  1. ANSWER:     "[Metric] dropped [X%] due to [cause]"
  2. IMPACT:     "$[Y]/day lost, affecting [segment]"
  3. ACTION:     "[Rollback/Fix/Monitor] — ETA [time]"
  4. EVIDENCE:   Only if asked (decomposition data)

INTERVIEW TIME ALLOCATION (10 min):
  Clarify & Frame:       1 min
  Data Validation:       1 min
  Seasonality Check:     1 min
  Metric Decomposition:  3 min
  Segment Isolation:     2 min
  Root Cause & Action:   2 min

RED FLAGS (things to always check):
  - Day-of-week effect (Mon vs Sun)
  - Promotion ending / starting
  - App update / deploy timestamp
  - Third-party dependency (payments, CDN)
  - Bot traffic spike/removal
  - Definition change in upstream table

+============================================================+
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 39 of 45*
