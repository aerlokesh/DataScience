# 🎯 Topic 11: Event-Based Data Modeling

> *Modern analytics is built on events — every click, purchase, page view, and status change tells a story. Event-based modeling captures the full narrative of user behavior rather than just the final state.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [Event Sourcing vs State-Based Modeling](#event-sourcing-vs-state-based-modeling)
3. [Event Schema Design](#event-schema-design)
4. [The Actor-Action-Object-Timestamp Pattern](#actor-action-object-timestamp-pattern)
5. [Wide vs Tall Event Tables](#wide-vs-tall-event-tables)
6. [The Activity Schema Pattern](#the-activity-schema-pattern)
7. [Session Reconstruction](#session-reconstruction)
8. [Clickstream Modeling](#clickstream-modeling)
9. [Real-World Examples](#real-world-examples)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Introduction

Event-based data modeling records **what happened** rather than **what is**. Instead of storing "user's current plan is Premium," you store "user upgraded from Basic to Premium at 2024-03-15T14:22:00Z." The current state becomes a derivation of the event history, not the source of truth.

This approach powers:
- Product analytics (Mixpanel, Amplitude, Heap)
- Real-time streaming (Kafka, Kinesis)
- Behavioral cohort analysis
- Attribution modeling
- Funnel and retention analysis

> **Critical Insight:** Event-based models trade query simplicity for analytical power. You can always derive the current state from events (replay), but you cannot recover event history from a state-based model. This asymmetry makes events the more powerful primitive.

---

## Event Sourcing vs State-Based Modeling

### State-Based (Traditional)

```sql
-- State table: only shows current reality
CREATE TABLE users (
    user_id         INT PRIMARY KEY,
    plan            VARCHAR(20),      -- Current plan
    email           VARCHAR(200),     -- Current email
    last_login      TIMESTAMP,        -- Most recent
    total_orders    INT               -- Running count
);
-- What happened between signup and now? Unknown.
```

### Event-Sourced

```sql
-- Event log: every state change is recorded
CREATE TABLE user_events (
    event_id        BIGINT PRIMARY KEY,
    user_id         INT NOT NULL,
    event_type      VARCHAR(50),
    event_timestamp TIMESTAMP NOT NULL,
    properties      JSONB              -- Flexible payload
);
-- Current state = replay all events for that user
```

### Side-by-Side Comparison

| Aspect | State-Based | Event-Sourced |
|--------|-------------|---------------|
| Storage of "what happened" | Lost on update | Preserved forever |
| Current state query | Direct SELECT | Aggregate/replay events |
| Storage size | Compact (1 row per entity) | Large (1 row per event) |
| Schema evolution | ALTER TABLE (risky) | New event types (additive) |
| Debugging/audit | Difficult | Full audit trail |
| Time-travel queries | Not possible | Natural |
| Write pattern | UPDATE in place | Append-only INSERT |
| Concurrent writes | Lock contention | No conflicts (append) |
| Analytics flexibility | Limited to stored aggregates | Unlimited re-analysis |
| Undo operations | Complex | Replay without unwanted event |

> **Critical Insight:** Most modern data platforms use a hybrid: the operational system may be state-based for simplicity, but fires events to a log (Kafka, event bus) that feeds the analytics warehouse. You get operational simplicity AND analytical power.

---

## Event Schema Design

### Core Event Fields

Every event should contain these foundational fields:

```sql
CREATE TABLE events (
    -- Identity
    event_id            UUID PRIMARY KEY,        -- Globally unique
    event_type          VARCHAR(100) NOT NULL,   -- 'page_view', 'purchase', 'signup'
    
    -- Temporal
    event_timestamp     TIMESTAMP NOT NULL,      -- When it happened
    received_timestamp  TIMESTAMP NOT NULL,      -- When we received it
    processed_timestamp TIMESTAMP,               -- When ETL processed it
    
    -- Actor (who did it)
    user_id             BIGINT,                  -- NULL for anonymous
    anonymous_id        VARCHAR(100),            -- Cookie/device ID
    session_id          VARCHAR(100),
    
    -- Context (where/how)
    platform            VARCHAR(20),             -- 'web', 'ios', 'android'
    device_type         VARCHAR(20),             -- 'desktop', 'mobile', 'tablet'
    country             VARCHAR(5),
    
    -- Payload (what specifically)
    properties          JSONB,                   -- Event-specific attributes
    
    -- Metadata
    schema_version      VARCHAR(10),             -- '1.0', '1.1', '2.0'
    source              VARCHAR(50)              -- 'client_sdk', 'server', 'etl'
);
```

### Event Type Taxonomy

```
user_events/
├── lifecycle/
│   ├── user_signed_up
│   ├── user_activated
│   ├── user_deactivated
│   └── user_deleted
├── authentication/
│   ├── login_attempted
│   ├── login_succeeded
│   └── login_failed
├── engagement/
│   ├── page_viewed
│   ├── feature_used
│   ├── search_performed
│   └── content_consumed
├── commerce/
│   ├── product_viewed
│   ├── added_to_cart
│   ├── checkout_started
│   ├── order_placed
│   └── order_refunded
└── subscription/
    ├── trial_started
    ├── plan_upgraded
    ├── plan_downgraded
    └── subscription_cancelled
```

---

## The Actor-Action-Object-Timestamp Pattern

Every event answers four questions:

| Component | Question | Example |
|-----------|----------|---------|
| **Actor** | Who did it? | user_id=12345 |
| **Action** | What did they do? | added_to_cart |
| **Object** | What was it done to? | product_id='SKU-789' |
| **Timestamp** | When? | 2024-03-15T14:22:33Z |

### Schema Implementation

```sql
-- Explicit actor-action-object columns
CREATE TABLE activity_log (
    event_id        UUID PRIMARY KEY,
    
    -- ACTOR
    actor_type      VARCHAR(20),         -- 'user', 'system', 'admin'
    actor_id        VARCHAR(50),
    
    -- ACTION
    action          VARCHAR(100),        -- 'viewed', 'purchased', 'shared'
    
    -- OBJECT
    object_type     VARCHAR(50),         -- 'product', 'page', 'document'
    object_id       VARCHAR(50),
    
    -- TIMESTAMP
    occurred_at     TIMESTAMP NOT NULL,
    
    -- CONTEXT
    context         JSONB                -- Additional metadata
);
```

### Real Example

```sql
INSERT INTO activity_log VALUES (
    'evt-a1b2c3',
    'user', '12345',                           -- Actor
    'added_to_cart',                            -- Action
    'product', 'SKU-789',                      -- Object
    '2024-03-15T14:22:33Z',                    -- Timestamp
    '{"quantity": 2, "source": "search_results", "session_id": "sess-xyz"}'
);
```

> **Critical Insight:** The Actor-Action-Object pattern provides a universal schema that can capture ANY business event. This uniformity enables cross-domain analysis: "Which users who viewed a product AND opened a support ticket eventually purchased?" becomes a simple self-join.

---

## Wide vs Tall Event Tables

### Tall (Long) Format — One Row Per Event

```sql
-- Every event is a row with a generic payload
SELECT * FROM events WHERE user_id = 12345 ORDER BY event_timestamp;

-- event_type          | event_timestamp      | properties
-- page_viewed         | 2024-03-15 14:20:00  | {"page": "/products/shoes"}
-- product_viewed      | 2024-03-15 14:20:15  | {"product_id": "SKU-123"}
-- added_to_cart       | 2024-03-15 14:21:00  | {"product_id": "SKU-123", "qty": 1}
-- checkout_started    | 2024-03-15 14:22:00  | {"cart_total": 89.99}
-- order_placed        | 2024-03-15 14:23:30  | {"order_id": "ORD-456", "total": 94.99}
```

### Wide Format — One Row Per Entity/Session with Columns Per Event

```sql
-- Pivoted: one row per user-session with boolean/count columns
CREATE TABLE session_summary (
    session_id          VARCHAR(100),
    user_id             BIGINT,
    session_start       TIMESTAMP,
    session_end         TIMESTAMP,
    pages_viewed        INT,
    products_viewed     INT,
    added_to_cart       BOOLEAN,
    checkout_started    BOOLEAN,
    order_placed        BOOLEAN,
    order_total         DECIMAL(10,2),
    search_count        INT,
    total_events        INT
);
```

### Comparison

| Aspect | Tall (Event Log) | Wide (Pivoted) |
|--------|-----------------|----------------|
| Schema flexibility | High — new events don't change schema | Low — new columns needed |
| Query for sequences | Natural (ORDER BY timestamp) | Impossible |
| Aggregation queries | Requires GROUP BY/PIVOT | Direct column access |
| ML feature engineering | Needs pivot/aggregation | Ready for models |
| Storage efficiency | Sparse (many NULLs in properties) | Dense (purpose-built) |
| Funnel analysis | Sequential query on tall | Pre-computed in wide |
| Data freshness | Real-time append | Batch computation |

> **Critical Insight:** In practice, you need BOTH. The tall event log is your source of truth (immutable, append-only). Wide tables are derived artifacts built for specific use cases (ML features, dashboards, session analytics). Think of wide tables as materialized views over the tall log.

---

## The Activity Schema Pattern

Popularized by the modern data stack, the Activity Schema standardizes event modeling around a single wide table with a fixed set of columns:

```sql
CREATE TABLE activity_stream (
    activity_id         UUID PRIMARY KEY,
    
    -- WHO
    entity_id           VARCHAR(100) NOT NULL,  -- The primary entity
    entity_type         VARCHAR(50),            -- 'customer', 'account'
    
    -- WHAT
    activity            VARCHAR(200) NOT NULL,  -- 'completed_order', 'viewed_page'
    
    -- WHEN
    ts                  TIMESTAMP NOT NULL,
    
    -- RELATIONSHIPS (standardized link columns)
    feature_1           VARCHAR(500),           -- Most important attribute
    feature_2           VARCHAR(500),           -- Second attribute
    feature_3           VARCHAR(500),           -- Third attribute
    revenue_impact      DECIMAL(12,2),          -- Monetary value if applicable
    link                VARCHAR(500),           -- FK to related entity
    
    -- METADATA
    activity_occurrence INT,                    -- Nth time entity did this
    activity_repeated_at TIMESTAMP              -- Next time entity does this
);
```

### Advantages of Activity Schema

1. **Self-join friendly** — All activities share the same schema
2. **Feature engineering** — Standardized columns enable reusable transforms
3. **Cross-domain joins** — Any activity can be correlated with any other
4. **Tool-friendly** — BI tools work well with a single wide table

```sql
-- "Users who completed an order within 7 days of signing up"
SELECT 
    signup.entity_id,
    signup.ts AS signup_date,
    order_act.ts AS first_order_date,
    DATEDIFF(day, signup.ts, order_act.ts) AS days_to_first_order
FROM activity_stream signup
JOIN activity_stream order_act
    ON signup.entity_id = order_act.entity_id
    AND order_act.activity = 'completed_order'
    AND order_act.activity_occurrence = 1
WHERE signup.activity = 'signed_up';
```

---

## Session Reconstruction

Sessions group events into meaningful user interaction windows. Since raw events are independent, sessions must be reconstructed.

### Sessionization Logic

```sql
-- Standard 30-minute inactivity timeout sessionization
WITH event_gaps AS (
    SELECT 
        user_id,
        event_timestamp,
        event_type,
        LAG(event_timestamp) OVER (
            PARTITION BY user_id ORDER BY event_timestamp
        ) AS prev_event_time,
        EXTRACT(EPOCH FROM (
            event_timestamp - LAG(event_timestamp) OVER (
                PARTITION BY user_id ORDER BY event_timestamp
            )
        )) / 60.0 AS minutes_since_last
    FROM events
),
session_boundaries AS (
    SELECT 
        *,
        CASE 
            WHEN minutes_since_last > 30 OR minutes_since_last IS NULL 
            THEN 1 ELSE 0 
        END AS is_new_session
    FROM event_gaps
),
session_ids AS (
    SELECT 
        *,
        SUM(is_new_session) OVER (
            PARTITION BY user_id ORDER BY event_timestamp
        ) AS session_number
    FROM session_boundaries
)
SELECT 
    user_id,
    session_number,
    MIN(event_timestamp) AS session_start,
    MAX(event_timestamp) AS session_end,
    COUNT(*) AS event_count,
    EXTRACT(EPOCH FROM (MAX(event_timestamp) - MIN(event_timestamp))) / 60.0 
        AS session_duration_minutes
FROM session_ids
GROUP BY user_id, session_number;
```

### Session Metrics

| Metric | Definition | Query Pattern |
|--------|-----------|---------------|
| Session duration | Last event - First event | MAX(ts) - MIN(ts) per session |
| Bounce rate | Sessions with 1 page view | COUNT(*) = 1 / total sessions |
| Pages per session | Count of page_view events | COUNT where type='page_view' |
| Conversion rate | Sessions with purchase / total | SUM(has_purchase) / COUNT(*) |
| Avg time between events | Mean gap within session | AVG of inter-event gaps |

---

## Clickstream Modeling

Clickstream data captures every interaction a user has with a digital product.

### Raw Clickstream Table

```sql
CREATE TABLE clickstream (
    event_id            UUID,
    user_id             BIGINT,
    session_id          VARCHAR(100),
    event_type          VARCHAR(50),      -- 'pageview','click','scroll','form_submit'
    page_url            VARCHAR(2000),
    page_title          VARCHAR(500),
    referrer_url        VARCHAR(2000),
    element_id          VARCHAR(200),     -- What was clicked
    element_text        VARCHAR(500),
    scroll_depth_pct    INT,
    viewport_width      INT,
    viewport_height     INT,
    event_timestamp     TIMESTAMP,
    
    -- UTM attribution
    utm_source          VARCHAR(100),
    utm_medium          VARCHAR(100),
    utm_campaign        VARCHAR(200)
);
```

### Funnel Analysis from Clickstream

```sql
-- E-commerce conversion funnel
WITH funnel AS (
    SELECT 
        session_id,
        MAX(CASE WHEN event_type = 'page_view' 
                 AND page_url LIKE '%/products/%' THEN 1 ELSE 0 END) AS viewed_product,
        MAX(CASE WHEN event_type = 'click' 
                 AND element_id = 'add-to-cart' THEN 1 ELSE 0 END) AS added_to_cart,
        MAX(CASE WHEN event_type = 'page_view' 
                 AND page_url LIKE '%/checkout%' THEN 1 ELSE 0 END) AS reached_checkout,
        MAX(CASE WHEN event_type = 'form_submit' 
                 AND page_url LIKE '%/order-confirm%' THEN 1 ELSE 0 END) AS completed_order
    FROM clickstream
    WHERE event_timestamp >= CURRENT_DATE - INTERVAL '7 days'
    GROUP BY session_id
)
SELECT 
    COUNT(*) AS total_sessions,
    SUM(viewed_product) AS viewed_product,
    SUM(added_to_cart) AS added_to_cart,
    SUM(reached_checkout) AS reached_checkout,
    SUM(completed_order) AS completed_order,
    ROUND(100.0 * SUM(added_to_cart) / NULLIF(SUM(viewed_product), 0), 1) 
        AS view_to_cart_pct,
    ROUND(100.0 * SUM(completed_order) / NULLIF(SUM(reached_checkout), 0), 1) 
        AS checkout_to_order_pct
FROM funnel;
```

---

## Real-World Examples

### Example 1: User Journey (SaaS Product)

```sql
-- Modeling user lifecycle events
INSERT INTO events (event_id, user_id, event_type, event_timestamp, properties) VALUES
('e1', 100, 'signed_up',        '2024-01-10 09:00:00', '{"plan": "free", "source": "google"}'),
('e2', 100, 'onboarding_started','2024-01-10 09:05:00', '{"step": 1}'),
('e3', 100, 'onboarding_completed','2024-01-10 09:15:00','{"steps_completed": 5}'),
('e4', 100, 'feature_used',     '2024-01-10 09:20:00', '{"feature": "dashboard_create"}'),
('e5', 100, 'invite_sent',      '2024-01-12 14:00:00', '{"invitee_email": "team@co.com"}'),
('e6', 100, 'plan_upgraded',    '2024-01-15 11:00:00', '{"from": "free", "to": "pro", "mrr": 49.00}'),
('e7', 100, 'feature_used',     '2024-01-16 10:00:00', '{"feature": "api_integration"}');

-- Derive activation: signed_up -> feature_used within 24h?
SELECT 
    s.user_id,
    s.event_timestamp AS signup_time,
    MIN(f.event_timestamp) AS first_feature_use,
    CASE WHEN MIN(f.event_timestamp) <= s.event_timestamp + INTERVAL '24 hours'
         THEN TRUE ELSE FALSE END AS activated_24h
FROM events s
LEFT JOIN events f 
    ON s.user_id = f.user_id 
    AND f.event_type = 'feature_used'
    AND f.event_timestamp > s.event_timestamp
WHERE s.event_type = 'signed_up'
GROUP BY s.user_id, s.event_timestamp;
```

### Example 2: Product Analytics (E-commerce)

```sql
-- Product interaction events for recommendation engine
CREATE TABLE product_events (
    event_id        UUID,
    user_id         BIGINT,
    product_id      VARCHAR(50),
    event_type      VARCHAR(30),     -- 'view','wishlist','cart','purchase','review'
    event_timestamp TIMESTAMP,
    properties      JSONB
);

-- Build user-product affinity matrix
SELECT 
    user_id,
    product_id,
    SUM(CASE event_type 
        WHEN 'view' THEN 1
        WHEN 'wishlist' THEN 3
        WHEN 'cart' THEN 5
        WHEN 'purchase' THEN 10
        WHEN 'review' THEN 7
    END) AS affinity_score,
    COUNT(*) AS interaction_count,
    MAX(event_timestamp) AS last_interaction
FROM product_events
WHERE event_timestamp >= CURRENT_DATE - INTERVAL '90 days'
GROUP BY user_id, product_id;
```

### Example 3: Multi-Touch Attribution

```sql
-- Track touchpoints leading to conversion
WITH user_touchpoints AS (
    SELECT 
        user_id,
        event_type,
        event_timestamp,
        properties->>'channel' AS channel,
        LEAD(event_type) OVER (
            PARTITION BY user_id ORDER BY event_timestamp
        ) AS next_event
    FROM events
    WHERE event_type IN ('ad_clicked', 'email_opened', 'organic_visit', 
                         'social_click', 'order_placed')
),
conversion_paths AS (
    SELECT 
        user_id,
        ARRAY_AGG(channel ORDER BY event_timestamp) AS path,
        COUNT(*) - 1 AS touchpoints_before_conversion
    FROM user_touchpoints
    WHERE user_id IN (
        SELECT user_id FROM events WHERE event_type = 'order_placed'
    )
    GROUP BY user_id
)
SELECT 
    touchpoints_before_conversion,
    COUNT(*) AS conversions
FROM conversion_paths
GROUP BY touchpoints_before_conversion
ORDER BY touchpoints_before_conversion;
```

---

## Interview Talking Points

> "We modeled our product analytics using the Actor-Action-Object-Timestamp pattern. Every user interaction — views, clicks, purchases, reviews — became an event in a single unified table. This allowed us to answer questions that would have been impossible with separate tables: 'What percentage of users who viewed a product 3+ times eventually purchased it within 7 days?'"

> "I designed our sessionization logic using a 30-minute inactivity window, but we validated this threshold by plotting the distribution of inter-event gaps. We found a bimodal distribution with a clear valley at 25 minutes, confirming that 30 minutes was a sensible cutoff for our product."

> "The biggest tradeoff with event-based modeling is query complexity for current-state questions. 'What plan is user X on?' requires scanning all plan_changed events and taking the most recent. We solved this with a materialized view that maintains current state, refreshed every 5 minutes — giving us the best of both worlds."

> "We chose the tall event format for our raw data lake because schema flexibility was critical — our product team ships new event types weekly. Wide tables are computed downstream for specific ML models and dashboard views. The tall-to-wide transformation happens in our dbt models."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Storing only current state, discarding events | Keep the full event log; derive state from events |
| No schema versioning on events | Include schema_version field; handle schema evolution |
| Using mutable event tables (UPDATE/DELETE) | Events are immutable; append correction events instead |
| Inconsistent event naming (camelCase + snake_case) | Establish convention early: verb_noun (past tense) |
| Missing timestamp fields (only event_time) | Track event_time, received_time, processed_time |
| No anonymous-to-known identity stitching | Implement identity resolution (anonymous_id -> user_id) |
| Assuming events arrive in order | Always sort by event_timestamp; handle late arrivals |
| Enormous JSONB payloads without documentation | Schema registry or at minimum a documented contract |
| Session timeout too short or too long | Analyze inter-event gap distribution to validate threshold |
| Mixing business events with system events | Separate user actions from system/infra events |

---

## Rapid-Fire Q&A

**Q1: What's the difference between event sourcing and event-driven architecture?**
Event sourcing = storing all state changes as events (the event log IS the database). Event-driven architecture = components communicate via events (messages trigger actions). You can have one without the other.

**Q2: How do you handle late-arriving events?**
Use event_timestamp (when it happened) vs received_timestamp (when it arrived). Process based on event_timestamp. For streaming, use watermarks with allowed lateness windows.

**Q3: How do you handle event schema changes?**
Use a schema registry (Confluent Schema Registry, AWS Glue Schema Registry). Version your events. Support both forward and backward compatibility. Never remove required fields.

**Q4: What's identity resolution in event modeling?**
Mapping anonymous events (cookie-based) to known users (post-login). When user 123 logs in from device ABC, retroactively associate all device ABC events with user 123.

**Q5: How do you size an event table?**
Estimate: users x events_per_user_per_day x days_retained x avg_row_size. A SaaS product with 100K daily users at 50 events/day = 5M rows/day = 1.8B rows/year.

**Q6: When should events be in a data lake vs warehouse?**
Raw events in the lake (cheap, schema-on-read, unlimited retention). Transformed/sessionized events in the warehouse (fast queries, governed schema). Lakehouse blurs this line.

**Q7: What's the difference between CDC and event-based modeling?**
CDC (Change Data Capture) captures database mutations as events (INSERT, UPDATE, DELETE on a table). Event modeling captures business-level events (user_signed_up, order_placed). CDC is infrastructure; events are domain.

**Q8: How do you build funnels from event data?**
Define ordered steps, check if each user completed step N before step N+1, with optional time constraints. Use window functions (LEAD/LAG) or self-joins with timestamp ordering.

**Q9: What's the "wide event" anti-pattern?**
Putting hundreds of optional columns on a single event table instead of using a JSONB payload or separate event types. Leads to tables that are 95% NULL.

**Q10: How do you partition event tables for performance?**
Partition by date (event_timestamp) as the primary strategy. Optionally cluster/sort by user_id or event_type for common access patterns. In BigQuery, partition by day + cluster by user_id, event_type.

---

## ASCII Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────┐
│              EVENT-BASED DATA MODELING GUIDE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  STATE vs EVENTS                                                 │
│  ═══════════════                                                 │
│                                                                  │
│  State:  [Current Snapshot] ← Only "now"                         │
│           user.plan = 'premium'                                   │
│                                                                  │
│  Events: [e1]→[e2]→[e3]→[e4]→[e5] ← Full history               │
│          signed_up → trial → upgraded → feature_used → ...       │
│                                                                  │
│  Current State = f(all events) — replay to reconstruct           │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  ACTOR-ACTION-OBJECT-TIMESTAMP                                   │
│  ═════════════════════════════                                    │
│                                                                  │
│  ┌────────┐   ┌────────────┐   ┌──────────┐   ┌───────────┐    │
│  │ ACTOR  │   │   ACTION   │   │  OBJECT  │   │ TIMESTAMP │    │
│  │ user_id│──▶│added_to_cart│──▶│product_id│   │ 14:22:33Z │    │
│  │ "1234" │   │            │   │ "SKU-78" │   │           │    │
│  └────────┘   └────────────┘   └──────────┘   └───────────┘    │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  TALL vs WIDE                                                    │
│  ════════════                                                    │
│                                                                  │
│  TALL (Event Log):           WIDE (Session Summary):             │
│  ┌──────┬─────────────┐     ┌──────┬───┬───┬───┬───┐           │
│  │ user │ event_type  │     │ user │ v │ c │ p │ $ │           │
│  ├──────┼─────────────┤     ├──────┼───┼───┼───┼───┤           │
│  │  A   │ page_view   │     │  A   │ 3 │ 1 │ 1 │ Y │           │
│  │  A   │ page_view   │     │  B   │ 5 │ 2 │ 0 │ N │           │
│  │  A   │ add_to_cart │     └──────┴───┴───┴───┴───┘           │
│  │  A   │ purchase    │     (v=views, c=cart, p=purchase)        │
│  │  B   │ page_view   │                                         │
│  │  B   │ page_view   │     Tall = source of truth               │
│  │  B   │ add_to_cart │     Wide = derived for specific use      │
│  └──────┴─────────────┘                                         │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  SESSIONIZATION                                                  │
│  ══════════════                                                  │
│                                                                  │
│  Events: ●──●──●──●─────────────────────●──●──●                 │
│          |  2m  3m  1m |     45 min gap     | 2m  5m |           │
│          |← Session 1 →|                    |← Sess 2→|          │
│                                                                  │
│  Rule: gap > 30 min = new session                                │
│  Implement: LAG() + SUM(is_new_session) OVER (ORDER BY ts)      │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  FUNNEL FROM EVENTS                                              │
│  ══════════════════                                              │
│                                                                  │
│  product_viewed  ████████████████████████  100% (10,000)         │
│  added_to_cart   ████████████             45%  (4,500)           │
│  checkout_start  ████████                 30%  (3,000)           │
│  order_placed    ████                     15%  (1,500)           │
│                                                                  │
│  Drop-off analysis: WHERE did users leave?                       │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  SCHEMA DESIGN PRINCIPLES                                        │
│  ════════════════════════                                         │
│                                                                  │
│  1. Events are IMMUTABLE — never UPDATE or DELETE                 │
│  2. Include event_time AND received_time (late arrivals)         │
│  3. Use JSONB/MAP for flexible properties                        │
│  4. Version your schemas (schema_version field)                  │
│  5. Consistent naming: past_tense_verb_noun                      │
│  6. Partition by date, cluster by user_id + event_type           │
│  7. Derive state from events, not the reverse                    │
│  8. Implement identity stitching (anonymous → known)             │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  ACTIVITY SCHEMA (Standardized Pattern)                          │
│  ══════════════════════════════════════                           │
│                                                                  │
│  entity_id | activity        | ts         | f1     | f2    | rev │
│  user_100  | signed_up       | 2024-01-10 | google | free  |  0  │
│  user_100  | plan_upgraded   | 2024-01-15 | pro    | free  | 49  │
│  user_100  | feature_used    | 2024-01-16 | api    | -     |  0  │
│                                                                  │
│  Self-join any activity with any other — unified schema!         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 11 of 45*
