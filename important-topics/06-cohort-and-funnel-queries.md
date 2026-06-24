# Topic 6: Cohort & Funnel Queries

> **Data Science Interview -- Deep Dive**  
> Building retention analyses, multi-step conversion funnels, user journey sessionization, and time-to-event calculations in SQL -- the bread and butter of product analytics interviews at the 5-YOE level.

---

## Table of Contents

1. [Why Cohort and Funnel Analysis Dominate Interviews](#1-why-cohort-and-funnel-analysis-dominate-interviews)
2. [Cohort Definition in SQL](#2-cohort-definition-in-sql)
3. [Day-N Retention Queries](#3-day-n-retention-queries)
4. [Retention Triangle (Full Cohort Table)](#4-retention-triangle-full-cohort-table)
5. [Funnel Conversion Queries](#5-funnel-conversion-queries)
6. [Strict vs Loose Funnels](#6-strict-vs-loose-funnels)
7. [Sessionization](#7-sessionization)
8. [User Journey Analysis](#8-user-journey-analysis)
9. [Time-to-Event Analysis](#9-time-to-event-analysis)
10. [Interview Problems with Solutions](#10-interview-problems-with-solutions)
11. [Interview Talking Points](#11-interview-talking-points)
12. [Common Interview Mistakes](#12-common-interview-mistakes)
13. [Rapid-Fire Q&A](#13-rapid-fire-qa)
14. [Summary Cheat Sheet](#14-summary-cheat-sheet)

---

## 1. Why Cohort and Funnel Analysis Dominate Interviews

Product companies (Meta, Spotify, Uber, Airbnb) need data scientists who can answer:

- "Are we retaining the users we acquire?" (cohort retention)
- "Where do users drop off in our conversion flow?" (funnel analysis)
- "How long does it take users to reach value?" (time-to-event)

These questions require combining GROUP BY, window functions, date arithmetic, and conditional logic -- testing multiple SQL skills simultaneously.

> **Critical Insight:** Interviewers want to see you define the cohort precisely before writing SQL. Always ask: "How do we define the cohort?" (signup date? first purchase? first app open?) and "What counts as 'active'?" (any event? specific event? minimum threshold?)

---

## 2. Cohort Definition in SQL

A cohort is a group of users who share a defining characteristic at a specific time.

### Common Cohort Definitions

| Cohort Type | Definition | SQL Pattern |
|-------------|-----------|-------------|
| Signup cohort | Week/month of registration | `DATE_TRUNC('week', signup_date)` |
| First-purchase cohort | Week/month of first order | `MIN(order_date) OVER (PARTITION BY user_id)` |
| Acquisition channel cohort | Channel of first touchpoint | `FIRST_VALUE(channel) OVER (...)` |
| Feature adoption cohort | Date they first used feature X | `MIN(event_date) WHERE event = 'feature_x'` |

```sql
-- Build a signup cohort dimension
WITH user_cohorts AS (
    SELECT 
        user_id,
        DATE_TRUNC('week', signup_date) AS cohort_week,
        signup_date
    FROM users
    WHERE signup_date >= '2024-01-01'
)
SELECT 
    cohort_week,
    COUNT(*) AS cohort_size
FROM user_cohorts
GROUP BY cohort_week
ORDER BY cohort_week;
```

---

## 3. Day-N Retention Queries

Day-N retention answers: "Of users who joined on day X, what fraction returned on day X+N?"

### D1, D7, D30 Retention

```sql
WITH user_cohorts AS (
    SELECT 
        user_id,
        DATE_TRUNC('day', signup_date)::DATE AS cohort_date
    FROM users
    WHERE signup_date >= '2024-01-01'
),
activity AS (
    SELECT DISTINCT 
        user_id,
        activity_date::DATE AS active_date
    FROM user_activity
)
SELECT 
    uc.cohort_date,
    COUNT(DISTINCT uc.user_id) AS cohort_size,
    COUNT(DISTINCT CASE 
        WHEN a1.active_date = uc.cohort_date + 1 THEN uc.user_id 
    END) AS d1_retained,
    COUNT(DISTINCT CASE 
        WHEN a7.active_date = uc.cohort_date + 7 THEN uc.user_id 
    END) AS d7_retained,
    COUNT(DISTINCT CASE 
        WHEN a30.active_date = uc.cohort_date + 30 THEN uc.user_id 
    END) AS d30_retained,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN a1.active_date = uc.cohort_date + 1 THEN uc.user_id END) 
        / COUNT(DISTINCT uc.user_id), 2) AS d1_pct,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN a7.active_date = uc.cohort_date + 7 THEN uc.user_id END) 
        / COUNT(DISTINCT uc.user_id), 2) AS d7_pct,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN a30.active_date = uc.cohort_date + 30 THEN uc.user_id END) 
        / COUNT(DISTINCT uc.user_id), 2) AS d30_pct
FROM user_cohorts uc
LEFT JOIN activity a1 ON uc.user_id = a1.user_id AND a1.active_date = uc.cohort_date + 1
LEFT JOIN activity a7 ON uc.user_id = a7.user_id AND a7.active_date = uc.cohort_date + 7
LEFT JOIN activity a30 ON uc.user_id = a30.user_id AND a30.active_date = uc.cohort_date + 30
GROUP BY uc.cohort_date
ORDER BY uc.cohort_date;
```

> **Critical Insight:** The key decision is "Day-N exact" vs "Day-N onward." D7 exact means they returned precisely on day 7. D7+ (bounded) means they returned anytime on or after day 7. Most product teams use D7+ (returned within day 7 through day 13 window) which is more robust to daily variance. Always clarify this in an interview.

### Alternative: Efficient Single-Join Pattern

```sql
WITH user_cohorts AS (
    SELECT user_id, MIN(activity_date)::DATE AS cohort_date
    FROM user_activity
    GROUP BY user_id
),
retention_data AS (
    SELECT 
        uc.user_id,
        uc.cohort_date,
        a.activity_date::DATE - uc.cohort_date AS days_since_signup
    FROM user_cohorts uc
    JOIN user_activity a ON uc.user_id = a.user_id
)
SELECT 
    cohort_date,
    COUNT(DISTINCT user_id) AS cohort_size,
    COUNT(DISTINCT CASE WHEN days_since_signup = 1 THEN user_id END) AS d1,
    COUNT(DISTINCT CASE WHEN days_since_signup = 7 THEN user_id END) AS d7,
    COUNT(DISTINCT CASE WHEN days_since_signup = 30 THEN user_id END) AS d30
FROM retention_data
WHERE days_since_signup IN (0, 1, 7, 30)
GROUP BY cohort_date
ORDER BY cohort_date;
```

---

## 4. Retention Triangle (Full Cohort Table)

A retention triangle shows retention rates for every (cohort, period) combination.

```sql
WITH user_cohorts AS (
    SELECT 
        user_id,
        DATE_TRUNC('week', signup_date)::DATE AS cohort_week
    FROM users
),
weekly_activity AS (
    SELECT DISTINCT
        user_id,
        DATE_TRUNC('week', activity_date)::DATE AS active_week
    FROM user_activity
),
retention_raw AS (
    SELECT 
        uc.cohort_week,
        (wa.active_week - uc.cohort_week) / 7 AS weeks_since_signup,
        COUNT(DISTINCT uc.user_id) AS active_users
    FROM user_cohorts uc
    JOIN weekly_activity wa ON uc.user_id = wa.user_id
    WHERE wa.active_week >= uc.cohort_week
    GROUP BY uc.cohort_week, (wa.active_week - uc.cohort_week) / 7
),
cohort_sizes AS (
    SELECT cohort_week, COUNT(DISTINCT user_id) AS cohort_size
    FROM user_cohorts
    GROUP BY cohort_week
)
SELECT 
    r.cohort_week,
    cs.cohort_size,
    r.weeks_since_signup,
    r.active_users,
    ROUND(100.0 * r.active_users / cs.cohort_size, 2) AS retention_pct
FROM retention_raw r
JOIN cohort_sizes cs ON r.cohort_week = cs.cohort_week
WHERE r.weeks_since_signup <= 12
ORDER BY r.cohort_week, r.weeks_since_signup;
```

---

## 5. Funnel Conversion Queries

A funnel tracks the sequential completion of steps. The key question: at each step, what percentage of users proceed to the next?

### Basic 5-Step Funnel

```sql
-- E-commerce funnel: Visit -> Search -> View Product -> Add to Cart -> Purchase
WITH funnel AS (
    SELECT 
        user_id,
        MAX(CASE WHEN event_type = 'page_view' THEN 1 ELSE 0 END) AS step1_visit,
        MAX(CASE WHEN event_type = 'search' THEN 1 ELSE 0 END) AS step2_search,
        MAX(CASE WHEN event_type = 'product_view' THEN 1 ELSE 0 END) AS step3_view,
        MAX(CASE WHEN event_type = 'add_to_cart' THEN 1 ELSE 0 END) AS step4_cart,
        MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS step5_purchase
    FROM events
    WHERE event_date BETWEEN '2024-01-01' AND '2024-01-31'
    GROUP BY user_id
)
SELECT 
    SUM(step1_visit) AS step1_users,
    SUM(step2_search) AS step2_users,
    SUM(step3_view) AS step3_users,
    SUM(step4_cart) AS step4_users,
    SUM(step5_purchase) AS step5_users,
    ROUND(100.0 * SUM(step2_search) / NULLIF(SUM(step1_visit), 0), 2) AS visit_to_search_pct,
    ROUND(100.0 * SUM(step3_view) / NULLIF(SUM(step2_search), 0), 2) AS search_to_view_pct,
    ROUND(100.0 * SUM(step4_cart) / NULLIF(SUM(step3_view), 0), 2) AS view_to_cart_pct,
    ROUND(100.0 * SUM(step5_purchase) / NULLIF(SUM(step4_cart), 0), 2) AS cart_to_purchase_pct,
    ROUND(100.0 * SUM(step5_purchase) / NULLIF(SUM(step1_visit), 0), 2) AS overall_conversion_pct
FROM funnel;
```

> **Critical Insight:** The basic funnel above is a "loose funnel" -- a user counts for step 3 even if they never did step 2. For a "strict funnel" that requires sequential order, you need timestamp-based logic. Always clarify in interviews which type is expected.

---

## 6. Strict vs Loose Funnels

### Comparison

| Funnel Type | Definition | Use Case |
|-------------|-----------|----------|
| **Loose** | User did step N at any point (regardless of order) | Feature adoption, multi-path journeys |
| **Strict ordered** | User did steps 1, 2, 3... in chronological sequence | Checkout flow, onboarding wizard |
| **Strict ordered with time window** | Steps must happen in order AND within X minutes/hours | Session-based conversion |

### Strict Ordered Funnel

```sql
-- Strict funnel: each step must happen AFTER the previous step
WITH step1 AS (
    SELECT user_id, MIN(event_time) AS step1_time
    FROM events
    WHERE event_type = 'page_view'
      AND event_date BETWEEN '2024-01-01' AND '2024-01-31'
    GROUP BY user_id
),
step2 AS (
    SELECT s1.user_id, MIN(e.event_time) AS step2_time
    FROM step1 s1
    JOIN events e ON s1.user_id = e.user_id
    WHERE e.event_type = 'search'
      AND e.event_time > s1.step1_time
    GROUP BY s1.user_id
),
step3 AS (
    SELECT s2.user_id, MIN(e.event_time) AS step3_time
    FROM step2 s2
    JOIN events e ON s2.user_id = e.user_id
    WHERE e.event_type = 'product_view'
      AND e.event_time > s2.step2_time
    GROUP BY s2.user_id
),
step4 AS (
    SELECT s3.user_id, MIN(e.event_time) AS step4_time
    FROM step3 s3
    JOIN events e ON s3.user_id = e.user_id
    WHERE e.event_type = 'add_to_cart'
      AND e.event_time > s3.step3_time
    GROUP BY s3.user_id
),
step5 AS (
    SELECT s4.user_id, MIN(e.event_time) AS step5_time
    FROM step4 s4
    JOIN events e ON s4.user_id = e.user_id
    WHERE e.event_type = 'purchase'
      AND e.event_time > s4.step4_time
    GROUP BY s4.user_id
)
SELECT 
    (SELECT COUNT(*) FROM step1) AS step1_users,
    (SELECT COUNT(*) FROM step2) AS step2_users,
    (SELECT COUNT(*) FROM step3) AS step3_users,
    (SELECT COUNT(*) FROM step4) AS step4_users,
    (SELECT COUNT(*) FROM step5) AS step5_users;
```

### With Time Window Constraint

```sql
-- Add: AND e.event_time <= s1.step1_time + INTERVAL '1 hour'
-- to each step to enforce a session-level funnel
```

---

## 7. Sessionization

Sessionization groups a stream of events into discrete sessions based on inactivity gaps.

### Standard 30-Minute Gap Sessionization

```sql
WITH ordered_events AS (
    SELECT 
        user_id,
        event_time,
        event_type,
        LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) AS prev_event_time
    FROM events
),
session_boundaries AS (
    SELECT 
        *,
        CASE 
            WHEN prev_event_time IS NULL THEN 1  -- first event is always a new session
            WHEN EXTRACT(EPOCH FROM (event_time - prev_event_time)) > 1800 THEN 1  -- 30 min gap
            ELSE 0
        END AS is_new_session
    FROM ordered_events
),
session_numbered AS (
    SELECT 
        *,
        SUM(is_new_session) OVER (
            PARTITION BY user_id ORDER BY event_time
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS session_id
    FROM session_boundaries
)
SELECT 
    user_id,
    session_id,
    MIN(event_time) AS session_start,
    MAX(event_time) AS session_end,
    COUNT(*) AS event_count,
    EXTRACT(EPOCH FROM MAX(event_time) - MIN(event_time)) / 60.0 AS duration_minutes,
    ARRAY_AGG(event_type ORDER BY event_time) AS event_sequence
FROM session_numbered
GROUP BY user_id, session_id;
```

### Session-Level Metrics

```sql
-- After sessionization, compute session-level KPIs
WITH sessions AS (
    -- ... (sessionization CTE from above) ...
    SELECT user_id, session_id, session_start, session_end, event_count, duration_minutes
    FROM session_numbered_agg
)
SELECT 
    DATE_TRUNC('day', session_start) AS day,
    COUNT(*) AS total_sessions,
    COUNT(DISTINCT user_id) AS unique_users,
    AVG(duration_minutes) AS avg_session_duration,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY duration_minutes) AS median_duration,
    AVG(event_count) AS avg_events_per_session,
    SUM(CASE WHEN event_count = 1 THEN 1 ELSE 0 END)::FLOAT / COUNT(*) AS bounce_rate
FROM sessions
GROUP BY DATE_TRUNC('day', session_start);
```

---

## 8. User Journey Analysis

### Most Common Paths (Sequence Mining)

```sql
-- Find the top 10 most common 3-event sequences
WITH event_sequences AS (
    SELECT 
        user_id,
        event_type AS step1,
        LEAD(event_type, 1) OVER (PARTITION BY user_id ORDER BY event_time) AS step2,
        LEAD(event_type, 2) OVER (PARTITION BY user_id ORDER BY event_time) AS step3
    FROM events
    WHERE event_date = '2024-01-15'
)
SELECT 
    step1, step2, step3,
    COUNT(*) AS frequency,
    COUNT(DISTINCT user_id) AS unique_users
FROM event_sequences
WHERE step2 IS NOT NULL AND step3 IS NOT NULL
GROUP BY step1, step2, step3
ORDER BY frequency DESC
LIMIT 10;
```

### First-Touch to Conversion Path

```sql
-- What is the most common journey from first touch to purchase?
WITH user_journeys AS (
    SELECT 
        user_id,
        STRING_AGG(event_type, ' -> ' ORDER BY event_time) AS journey_path,
        COUNT(*) AS steps
    FROM events
    WHERE user_id IN (SELECT user_id FROM events WHERE event_type = 'purchase')
    GROUP BY user_id
)
SELECT 
    journey_path,
    COUNT(*) AS users_with_path,
    AVG(steps) AS avg_steps
FROM user_journeys
GROUP BY journey_path
ORDER BY users_with_path DESC
LIMIT 20;
```

---

## 9. Time-to-Event Analysis

### Time to First Purchase

```sql
WITH user_signup AS (
    SELECT user_id, signup_date FROM users
),
first_purchase AS (
    SELECT user_id, MIN(order_date) AS first_purchase_date
    FROM orders
    GROUP BY user_id
)
SELECT 
    DATE_TRUNC('week', us.signup_date) AS cohort_week,
    COUNT(*) AS cohort_size,
    COUNT(fp.user_id) AS converted,
    ROUND(100.0 * COUNT(fp.user_id) / COUNT(*), 2) AS conversion_rate,
    AVG(fp.first_purchase_date - us.signup_date) AS avg_days_to_convert,
    PERCENTILE_CONT(0.5) WITHIN GROUP (
        ORDER BY fp.first_purchase_date - us.signup_date
    ) AS median_days_to_convert
FROM user_signup us
LEFT JOIN first_purchase fp ON us.user_id = fp.user_id
GROUP BY DATE_TRUNC('week', us.signup_date)
ORDER BY cohort_week;
```

### Survival Curve (Discrete)

```sql
-- For each day N, what fraction of users have NOT yet converted?
WITH user_signup AS (
    SELECT user_id, signup_date::DATE AS signup_date FROM users
    WHERE signup_date >= '2024-01-01'
),
conversion AS (
    SELECT user_id, MIN(order_date)::DATE AS convert_date
    FROM orders
    GROUP BY user_id
),
days_to_convert AS (
    SELECT 
        us.user_id,
        COALESCE(c.convert_date - us.signup_date, 999) AS days_until_conversion
    FROM user_signup us
    LEFT JOIN conversion c ON us.user_id = c.user_id
)
SELECT 
    day_n,
    COUNT(*) AS users_not_converted,
    ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM user_signup), 2) AS survival_pct
FROM days_to_convert
CROSS JOIN generate_series(0, 30) AS day_n
WHERE days_until_conversion > day_n
GROUP BY day_n
ORDER BY day_n;
```

---

## 10. Interview Problems with Solutions

### Problem 1: Calculate D7 Retention by Platform

**Task:** For each platform (iOS, Android, Web), compute D7 retention for weekly signup cohorts.

```sql
WITH cohorts AS (
    SELECT 
        user_id,
        platform,
        DATE_TRUNC('week', signup_date)::DATE AS cohort_week
    FROM users
    WHERE signup_date >= '2024-01-01'
),
d7_active AS (
    SELECT DISTINCT user_id
    FROM user_activity a
    JOIN cohorts c ON a.user_id = c.user_id
    WHERE a.activity_date::DATE BETWEEN c.cohort_week + 7 AND c.cohort_week + 13
)
SELECT 
    c.platform,
    c.cohort_week,
    COUNT(DISTINCT c.user_id) AS cohort_size,
    COUNT(DISTINCT d.user_id) AS d7_retained,
    ROUND(100.0 * COUNT(DISTINCT d.user_id) / COUNT(DISTINCT c.user_id), 2) AS d7_retention_pct
FROM cohorts c
LEFT JOIN d7_active d ON c.user_id = d.user_id
GROUP BY c.platform, c.cohort_week
ORDER BY c.platform, c.cohort_week;
```

### Problem 2: Build a 5-Step Funnel with Drop-Off Analysis

**Task:** Show each step, users at that step, drop-off count, and drop-off percentage.

```sql
WITH funnel_steps AS (
    SELECT 
        user_id,
        MAX(CASE WHEN event_type = 'app_open' THEN 1 ELSE 0 END) AS s1,
        MAX(CASE WHEN event_type = 'search' THEN 1 ELSE 0 END) AS s2,
        MAX(CASE WHEN event_type = 'product_view' THEN 1 ELSE 0 END) AS s3,
        MAX(CASE WHEN event_type = 'add_to_cart' THEN 1 ELSE 0 END) AS s4,
        MAX(CASE WHEN event_type = 'checkout_complete' THEN 1 ELSE 0 END) AS s5
    FROM events
    WHERE event_date BETWEEN '2024-03-01' AND '2024-03-31'
    GROUP BY user_id
),
step_counts AS (
    SELECT 
        'Step 1: App Open' AS step_name, 1 AS step_num, SUM(s1) AS users FROM funnel_steps
    UNION ALL SELECT 'Step 2: Search', 2, SUM(s2) FROM funnel_steps
    UNION ALL SELECT 'Step 3: Product View', 3, SUM(s3) FROM funnel_steps
    UNION ALL SELECT 'Step 4: Add to Cart', 4, SUM(s4) FROM funnel_steps
    UNION ALL SELECT 'Step 5: Checkout', 5, SUM(s5) FROM funnel_steps
)
SELECT 
    step_name,
    users,
    LAG(users) OVER (ORDER BY step_num) - users AS drop_off,
    ROUND(100.0 * users / FIRST_VALUE(users) OVER (ORDER BY step_num), 2) AS pct_of_top,
    ROUND(100.0 * users / LAG(users) OVER (ORDER BY step_num), 2) AS step_conversion_pct
FROM step_counts
ORDER BY step_num;
```

### Problem 3: Identify Users Who Churned After Being Power Users

**Task:** Find users who were in the top 20% by activity in month M but had zero activity in month M+1.

```sql
WITH monthly_activity AS (
    SELECT 
        user_id,
        DATE_TRUNC('month', activity_date) AS month,
        COUNT(*) AS event_count
    FROM user_activity
    GROUP BY user_id, DATE_TRUNC('month', activity_date)
),
power_users AS (
    SELECT 
        user_id,
        month,
        event_count,
        NTILE(5) OVER (PARTITION BY month ORDER BY event_count DESC) AS quintile
    FROM monthly_activity
),
churned_power AS (
    SELECT 
        pu.user_id,
        pu.month AS power_month,
        pu.event_count
    FROM power_users pu
    WHERE pu.quintile = 1  -- top 20%
    AND NOT EXISTS (
        SELECT 1 FROM monthly_activity ma
        WHERE ma.user_id = pu.user_id
        AND ma.month = pu.month + INTERVAL '1 month'
    )
)
SELECT 
    power_month,
    COUNT(*) AS churned_power_users,
    AVG(event_count) AS avg_activity_before_churn
FROM churned_power
GROUP BY power_month
ORDER BY power_month;
```

---

## 11. Interview Talking Points

> "When building retention queries, I always start by precisely defining the cohort and the retention event. D7 retention can mean 'active exactly on day 7' or 'active during the day-7-to-day-13 window.' The latter is more forgiving and what most product teams actually track."

> "For funnel analysis, I distinguish between loose funnels (did they ever do each step?) and strict funnels (did they do steps in chronological order?). Loose funnels are simpler but can overcount; strict funnels give the true sequential conversion rate."

> "Sessionization is a pattern I've implemented many times. The algorithm is: order events by time, detect gaps exceeding a threshold (typically 30 minutes), flag each gap as a new session boundary, then use a cumulative sum of those flags to assign session IDs."

> "Time-to-event analysis is related to survival analysis in statistics. The SQL version uses a discrete approach -- for each day N, count users who have not yet converted. This gives a step-function approximation of a survival curve."

---

## 12. Common Interview Mistakes

| Mistake | Why It's Wrong | Correct Approach |
|---------|---------------|-----------------|
| Not clarifying retention definition | D1 exact vs D1+ gives different numbers | Ask: "exact day or window?" |
| Including signup day as "retained" | Day 0 is not retention -- they just signed up | Start counting from Day 1+ |
| Using COUNT instead of COUNT DISTINCT | Same user with multiple events inflates numbers | Always COUNT(DISTINCT user_id) for retention |
| Loose funnel when strict is needed | Overcounts conversions for ordered processes | Use timestamp ordering for sequential funnels |
| Not handling users who skip steps | Real data is messy -- users jump around | Define rules for step-skipping explicitly |
| Forgetting timezone in date math | signup_date in UTC but activity in local time | Normalize all timestamps to one timezone |
| Dividing by zero in conversion rates | Steps with 0 users cause NULL/error | Use NULLIF(denominator, 0) |
| Not filtering incomplete cohorts | Latest cohort has not had time to reach D30 | WHERE cohort_date <= CURRENT_DATE - 30 |
| Hard-coding session timeout | 30 min may not fit your product (games vs banking) | Parameterize and explain your choice |
| Ignoring the denominator shift | Denominator should be previous step, not step 1 | Track both: step-over-step AND overall conversion |

---

## 13. Rapid-Fire Q&A

**Q1: What is the difference between retention and engagement?**  
A: Retention = did the user come back (binary per period). Engagement = how much did they do when they came back (frequency/intensity within a period).

**Q2: What does "bounded retention" mean?**  
A: User is considered retained in week N if they were active any time during that week (not necessarily on the exact Nth day). More stable metric.

**Q3: How do you handle users who signed up recently in a D30 retention chart?**  
A: Exclude them. Filter: `WHERE cohort_date <= CURRENT_DATE - INTERVAL '30 days'`. Otherwise retention appears artificially low.

**Q4: What is "N-day rolling retention" vs "N-day classic retention"?**  
A: Classic = returned on exactly day N. Rolling = returned on day N OR any day after. Rolling is always >= classic.

**Q5: How would you A/B test a funnel change?**  
A: Randomly assign users to treatment/control. Build the same funnel query for each group. Compare step-over-step conversion rates with statistical significance (chi-squared or z-test for proportions).

**Q6: Why use LEAD/LAG for sessionization instead of a self-join?**  
A: Window functions are single-pass (O(n log n)). Self-join is O(n^2) and much harder to read.

**Q7: How do you define "churn" in SQL?**  
A: No activity for X consecutive days/weeks. Common: "churned if no activity in the last 28 days." Requires checking absence, not presence.

**Q8: Can you compute a funnel per segment in one query?**  
A: Yes. Add the segment column to your GROUP BY and compute conversion rates within each segment.

**Q9: What is "resurrection" in retention analysis?**  
A: A user who was inactive (churned) and then became active again. Track by finding gaps in activity followed by resumed activity.

**Q10: How do you handle multiple sessions per user per day in funnel analysis?**  
A: Decide whether the funnel is user-level (did they ever complete it) or session-level (completed within one session). Session-level requires sessionization first.

---

## 14. Summary Cheat Sheet

```
+------------------------------------------------------------------+
|  COHORT & FUNNEL QUERIES -- CHEAT SHEET                          |
+------------------------------------------------------------------+
|                                                                    |
|  COHORT RETENTION PATTERN:                                         |
|    1. Define cohort (signup week/month)                            |
|    2. Define retention event (any activity, specific action)       |
|    3. Compute days_since_signup = activity_date - cohort_date      |
|    4. COUNT(DISTINCT user) per (cohort, days_since_signup)         |
|    5. Divide by cohort_size for retention percentage               |
|                                                                    |
|  FUNNEL PATTERN (LOOSE):                                           |
|    MAX(CASE WHEN event = 'X' THEN 1 ELSE 0 END) per user          |
|    SUM each step; divide step N by step N-1 for conversion         |
|                                                                    |
|  FUNNEL PATTERN (STRICT):                                          |
|    Chain CTEs: each step joins to prior step with time > prior     |
|    COUNT users at each step for sequential conversion              |
|                                                                    |
|  SESSIONIZATION PATTERN:                                           |
|    1. LAG(event_time) to get previous timestamp                    |
|    2. Flag gaps > threshold as new_session = 1                     |
|    3. SUM(new_session) cumulative = session_id                     |
|    4. GROUP BY (user_id, session_id) for session metrics           |
|                                                                    |
|  TIME-TO-EVENT:                                                    |
|    first_event_date - cohort_date = time_to_convert                |
|    AVG and MEDIAN for typical conversion time                      |
|    Survival curve: fraction NOT converted by day N                 |
|                                                                    |
|  KEY RULES:                                                        |
|    - Always clarify definitions before writing SQL                  |
|    - Use COUNT(DISTINCT user_id) not COUNT(*)                      |
|    - Filter out immature cohorts (not enough elapsed time)         |
|    - Use NULLIF to avoid division by zero                          |
|    - Normalize timezones before date arithmetic                    |
|                                                                    |
+------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Topics](README.md) collection -- Topic 6 of 45*
