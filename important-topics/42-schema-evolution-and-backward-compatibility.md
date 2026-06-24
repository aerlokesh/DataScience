# 🎯 Topic 42: Schema Evolution and Backward Compatibility

> *"Data schemas WILL change — that's inevitable. What separates mature data systems from fragile ones is whether those changes break downstream consumers or gracefully evolve in place."*

---

## 📑 Table of Contents

1. [Why Schemas Change](#why-schemas-change)
2. [Additive vs Breaking Changes](#additive-vs-breaking-changes)
3. [Forward and Backward Compatibility](#forward-and-backward-compatibility)
4. [Schema Registries](#schema-registries)
5. [Versioning Strategies](#versioning-strategies)
6. [Nullable vs Required Fields](#nullable-vs-required-fields)
7. [Migration Patterns](#migration-patterns)
8. [Handling Deprecated Columns](#handling-deprecated-columns)
9. [Protobuf/Avro Schema Evolution](#protobufavro-schema-evolution)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Why Schemas Change

### Common Reasons for Schema Evolution

| Category | Example | Frequency |
|----------|---------|-----------|
| New features | Add `subscription_tier` column | Monthly |
| Bug fixes | Change `price` from INT to DECIMAL | Rare but impactful |
| Business changes | Rename `gender` to `identity` | Occasional |
| Performance | Denormalize frequently joined columns | Quarterly |
| Compliance | Add `gdpr_consent_date`, remove `ssn` | As needed |
| Upstream changes | Source API adds/removes fields | Unpredictable |
| Refactoring | Split `address` into components | During migrations |

### The Pain of Schema Changes

```
Producer changes schema
    |
    +-- ETL pipeline breaks (unexpected column type)
    |
    +-- Dashboard shows NULL/errors (missing column)
    |
    +-- ML model fails (feature vector shape changed)
    |
    +-- Downstream team gets paged (SLA breach)
    |
    +-- Trust in data erodes (people stop using the table)
```

> **Critical Insight:** Schema changes are the #1 source of "data pipeline fires" in most organizations. They're predictable in the sense that they WILL happen, but unpredictable in timing. The solution is designing for evolution from day one — not hoping schemas stay static.

---

## Additive vs Breaking Changes

### Additive (Non-Breaking) Changes

These can be made safely without coordinating with consumers:

```
SAFE:
  + Add a new OPTIONAL column (with default NULL)
  + Add a new table
  + Add a new enum value (at the END of the list)
  + Widen a column (VARCHAR(50) → VARCHAR(100))
  + Add a new index
  + Add new optional fields to a message/record
```

### Breaking Changes

These WILL break downstream consumers unless coordinated:

```
BREAKING:
  - Remove a column
  - Rename a column
  - Change a column's data type (incompatible)
  - Change NULL → NOT NULL constraint
  - Change column semantics (same name, different meaning)
  - Remove an enum value
  - Change primary key structure
  - Change partitioning scheme
```

### Gray Area (Context-Dependent)

```
MAYBE BREAKING:
  ? Add NOT NULL column without default (breaks INSERT without the column)
  ? Change column from NOT NULL to NULL (existing code might not handle NULLs)
  ? Narrow a column (VARCHAR(100) → VARCHAR(50)) — data might truncate
  ? Add a new required field (breaks old producers)
  ? Change sort/clustering key (breaks queries relying on order)
```

### Compatibility Matrix

| Change Type | Backward Compatible? | Forward Compatible? | Safe Without Coordination? |
|-------------|---------------------|--------------------|-----------------------------|
| Add optional column | Yes | Yes | Yes |
| Add required column with default | Yes | No | Mostly |
| Remove unused column | No | Yes | NO — verify no consumers |
| Rename column | No | No | NO — always breaking |
| Widen type (INT → BIGINT) | Yes | No | Mostly |
| Narrow type (BIGINT → INT) | No | No | NO — data loss risk |
| Change semantics | No | No | NO — silent corruption |

> **Critical Insight:** The most dangerous schema change is a SEMANTIC change — same column name, but the meaning changes (e.g., `revenue` switches from USD to local currency). This won't trigger any technical error but will silently corrupt all downstream analyses. Always change the column name when semantics change.

---

## Forward and Backward Compatibility

### Definitions

```
BACKWARD COMPATIBILITY (new reader, old data):
  "Can new code read data written by old code?"
  New consumer can process old-format messages/rows.
  
  Example: New dashboard added column "subscription_tier"
           Can still read old rows where that column is NULL.

FORWARD COMPATIBILITY (old reader, new data):
  "Can old code read data written by new code?"
  Old consumer can process new-format messages/rows.
  
  Example: Producer adds new field "loyalty_score"
           Old consumer safely ignores unknown field.

FULL COMPATIBILITY:
  Both backward AND forward compatible.
  New and old producers/consumers coexist without issues.
```

### Visual Explanation

```
Timeline:  V1 ──────────────── V2 ──────────────── V3
           (old schema)         (new schema)

Backward: V2 reader can read V1 data? → YES = backward compatible
Forward:  V1 reader can read V2 data? → YES = forward compatible

        ┌──────────────────┐
        │ FULL COMPATIBLE  │ Both directions work
        ├──────────────────┤
        │ BACKWARD ONLY    │ New reads old, but old can't read new
        ├──────────────────┤
        │ FORWARD ONLY     │ Old reads new, but new can't read old
        ├──────────────────┤
        │ INCOMPATIBLE     │ Neither direction works
        └──────────────────┘
```

### Why Both Matter

```
Backward Compatibility needed when:
  - Rolling out new consumer gradually (canary/blue-green)
  - Historical data must remain queryable
  - Reprocessing old data with new code

Forward Compatibility needed when:
  - Rolling out new producer gradually
  - Multiple producer versions coexist
  - Consumers can't be updated simultaneously
```

---

## Schema Registries

### What Is a Schema Registry?

A centralized service that stores, validates, and enforces schema versions and compatibility rules.

```
Producer → Schema Registry → "Is this schema compatible with v3?"
                                    |
                              YES: Allow publish
                              NO:  Reject (breaking change blocked)
```

### Key Components

| Component | Purpose |
|-----------|---------|
| Schema Store | Versioned history of all schema definitions |
| Compatibility Checker | Validates new schema against rules |
| Subject | Logical grouping (e.g., per topic, per table) |
| Version | Incremental schema version within a subject |
| Compatibility Mode | BACKWARD, FORWARD, FULL, NONE |

### Confluent Schema Registry (Kafka Ecosystem)

```
Compatibility Modes:
  BACKWARD:       New schema can read old data
  BACKWARD_TRANSITIVE: New can read ALL previous versions
  FORWARD:        Old schema can read new data  
  FORWARD_TRANSITIVE:  Old can read ALL future versions
  FULL:           Both backward and forward
  FULL_TRANSITIVE:     Full across all versions
  NONE:           No compatibility check (dangerous)
```

### Example Workflow

```
1. Producer wants to add field "loyalty_tier"
2. Producer submits new schema to Registry
3. Registry checks compatibility:
   - Mode: BACKWARD
   - New field has default value? YES
   - Passes! Schema registered as v4.
4. Producer starts writing v4 messages
5. Consumers upgrade at their own pace
   (old consumers safely ignore new field)
```

> **Critical Insight:** A schema registry is to data schemas what version control is to code. Without it, schema changes are untracked, unvalidated, and uncoordinated. With it, you get an audit trail, automated compatibility checks, and safe rollback capability.

---

## Versioning Strategies

### Strategy 1: Schema Versioning (Explicit Version Field)

```json
{
  "schema_version": 3,
  "user_id": "abc123",
  "email": "user@example.com",
  "loyalty_tier": "gold"  // Added in v3
}
```

**Pros:** Explicit, easy to route processing
**Cons:** Consumers must handle all versions, code complexity grows

### Strategy 2: API Versioning (URL/Header)

```
/api/v1/users  → Returns schema v1 format
/api/v2/users  → Returns schema v2 format (with loyalty_tier)
```

**Pros:** Clean separation, consumers choose version
**Cons:** Maintaining multiple versions is expensive

### Strategy 3: Table Versioning

```sql
-- Approach A: New table per major version
CREATE TABLE users_v1 (id, name, email);
CREATE TABLE users_v2 (id, name, email, loyalty_tier);

-- Approach B: View-based versioning
CREATE VIEW users_v1 AS SELECT id, name, email FROM users;
CREATE VIEW users_v2 AS SELECT id, name, email, loyalty_tier FROM users;
```

**Pros:** No migration needed, backward compatible by default
**Cons:** Proliferation of tables/views, confusing ownership

### Strategy 4: Additive-Only (Append Columns)

```sql
-- Never remove, never rename — only add
ALTER TABLE users ADD COLUMN loyalty_tier VARCHAR(20) DEFAULT NULL;
ALTER TABLE users ADD COLUMN created_at_v2 TIMESTAMP;  -- If semantics change, new column
```

**Pros:** Always backward compatible, simple
**Cons:** Tables accumulate dead columns, can become unwieldy

### Choosing a Strategy

| Scenario | Recommended Strategy |
|----------|---------------------|
| Streaming (Kafka) | Schema registry + forward/backward compatibility |
| Data warehouse | Additive-only + deprecation process |
| REST APIs | URL versioning or content negotiation |
| ML feature stores | Explicit version + feature registry |
| Event sourcing | Schema version field in each event |

---

## Nullable vs Required Fields

### The Fundamental Tradeoff

```
NULLABLE (optional):
  + Easy to add new columns (won't break existing producers)
  + Backward compatible by default
  + Flexible for partial data
  - Consumers must handle NULLs everywhere
  - Less data quality enforcement
  - Can mask data pipeline bugs

REQUIRED (NOT NULL):
  + Guarantees data completeness
  + Simpler consumer logic (no null checks)
  + Catches pipeline bugs early
  - Breaking to add retroactively
  - Must provide defaults for historical data
  - Less flexible for evolution
```

### Decision Framework

```
Make it REQUIRED when:
  - Business logic cannot function without it (order_id, timestamp)
  - Missing values indicate a bug, not valid state
  - All producers CAN always provide the value

Make it NULLABLE when:
  - Field is new and historical data doesn't have it
  - Field is optional by nature (middle_name, promo_code)
  - Multiple producers exist with different capabilities
  - You need schema evolution flexibility
```

### The Default Value Pattern

```sql
-- Adding a new required column safely
-- Step 1: Add as nullable
ALTER TABLE orders ADD COLUMN fulfillment_type VARCHAR(20) NULL;

-- Step 2: Backfill historical data
UPDATE orders SET fulfillment_type = 'standard' WHERE fulfillment_type IS NULL;

-- Step 3: Set NOT NULL constraint (once all data is populated)
ALTER TABLE orders ALTER COLUMN fulfillment_type SET NOT NULL;

-- Step 4: Add default for new rows
ALTER TABLE orders ALTER COLUMN fulfillment_type SET DEFAULT 'standard';
```

> **Critical Insight:** In a data warehouse, bias toward NULLABLE for new columns. In a transactional system, bias toward REQUIRED. The warehouse is read-heavy and needs flexibility; the transactional system is write-heavy and needs guarantees.

---

## Migration Patterns

### Pattern 1: Expand-and-Contract (Zero Downtime)

```
Phase 1 — EXPAND: Add new column alongside old
  orders: [old_status VARCHAR(10), status_v2 VARCHAR(20)]
  Both old and new code writes to both columns.

Phase 2 — MIGRATE: Backfill old data into new column
  UPDATE orders SET status_v2 = old_status WHERE status_v2 IS NULL;

Phase 3 — TRANSITION: Switch consumers to new column
  Consumers read status_v2 instead of old_status.
  Verify all consumers migrated.

Phase 4 — CONTRACT: Remove old column
  ALTER TABLE orders DROP COLUMN old_status;
```

### Pattern 2: Shadow Table

```
1. Create new table with new schema
2. Dual-write to both old and new table
3. Verify new table correctness
4. Switch consumers to new table
5. Drop old table after grace period
```

### Pattern 3: View Migration

```sql
-- Original table
CREATE TABLE users_raw (id INT, first_name TEXT, last_name TEXT);

-- Phase 1: Create view with new schema  
CREATE VIEW users AS 
SELECT id, first_name, last_name, 
       first_name || ' ' || last_name AS full_name  -- New derived column
FROM users_raw;

-- Phase 2: Once consumers use the view, change underlying table freely
-- Consumers never see the raw table change
```

### Pattern 4: Blue-Green Tables

```
1. Build new table (green) with new schema
2. Populate green table with transformed data
3. Run validation: compare blue and green
4. Swap pointer (view/config) from blue to green
5. Keep blue as rollback for N days
6. Drop blue after confidence period
```

### Migration Decision Matrix

| Factor | Expand-Contract | Shadow Table | View | Blue-Green |
|--------|----------------|--------------|------|------------|
| Downtime | Zero | Zero | Zero | Near-zero |
| Complexity | Medium | High | Low | Medium |
| Storage cost | Low (temp extra column) | High (full copy) | None | High (full copy) |
| Rollback ease | Medium | High | High | High |
| Best for | Column changes | Major restructuring | Adding derived | Table replacement |

---

## Handling Deprecated Columns

### Deprecation Lifecycle

```
ACTIVE          → Column is used by producers and consumers
  |
DEPRECATED      → Column still populated but consumers should migrate
  |                (Add to documentation, log usage, notify teams)
  |                Duration: 2-4 weeks minimum
  |
ZOMBIE          → Column still exists but no longer populated
  |                (NULL for new rows, historical values preserved)
  |                Duration: 2-4 weeks
  |
REMOVED         → Column physically dropped from table
                   (After confirming zero active queries)
```

### Implementation

```sql
-- Step 1: Mark as deprecated (documentation + column comment)
COMMENT ON COLUMN users.legacy_score IS 
    'DEPRECATED since 2024-03-01. Use engagement_score instead. Removal target: 2024-04-01';

-- Step 2: Add logging to detect usage
-- (In your query audit system, flag queries referencing deprecated columns)

-- Step 3: Notify consumers
-- (Automated alerts when deprecated columns are queried)

-- Step 4: Stop populating (make zombie)
-- (Update pipeline to SET NULL instead of populating)

-- Step 5: Remove after grace period
ALTER TABLE users DROP COLUMN legacy_score;
```

### Tracking Deprecations

```yaml
# deprecated_columns.yml
deprecations:
  - table: users
    column: legacy_score
    deprecated_date: 2024-03-01
    removal_date: 2024-04-01
    replacement: engagement_score
    owner: analytics-team
    consumers_notified: true
    active_queries: 3  # from audit log
    
  - table: orders
    column: old_status
    deprecated_date: 2024-02-15
    removal_date: 2024-03-15
    replacement: status_v2
    owner: platform-team
    consumers_notified: true
    active_queries: 0  # safe to remove!
```

> **Critical Insight:** Never remove a column without evidence that no consumer uses it. Query audit logs are essential. A "no queries in 30 days" threshold gives confidence. Without this, you're gambling on not breaking an obscure monthly job.

---

## Protobuf/Avro Schema Evolution

### Apache Avro

```json
// Schema v1
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "id", "type": "string"},
    {"name": "email", "type": "string"},
    {"name": "created_at", "type": "long"}
  ]
}

// Schema v2 — BACKWARD COMPATIBLE (new optional field with default)
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "id", "type": "string"},
    {"name": "email", "type": "string"},
    {"name": "created_at", "type": "long"},
    {"name": "loyalty_tier", "type": ["null", "string"], "default": null}
  ]
}
```

**Avro Evolution Rules:**
- Add field with default → Backward compatible
- Remove field with default → Forward compatible
- Add field WITHOUT default → BREAKING (old data can't be read)
- Rename field → BREAKING (use aliases instead)

### Protocol Buffers (Protobuf)

```protobuf
// v1
message User {
  string id = 1;
  string email = 2;
  int64 created_at = 3;
}

// v2 — COMPATIBLE (new optional field)
message User {
  string id = 1;
  string email = 2;
  int64 created_at = 3;
  string loyalty_tier = 4;  // New field, tag 4
}
```

**Protobuf Evolution Rules:**
- NEVER change a field's tag number
- NEVER reuse a deleted field's tag number
- Adding new fields is always safe (old readers skip unknown tags)
- Removing fields is safe (mark as `reserved` to prevent reuse)
- Changing field types requires compatible type pairs (int32 ↔ int64)

### Comparison

| Feature | Avro | Protobuf | JSON Schema |
|---------|------|----------|-------------|
| Schema required for read | Yes | Yes (or self-describing) | No |
| Default values | Yes (in schema) | Yes (zero values) | Yes |
| Forward compatible | With defaults | Always (skip unknown) | Depends |
| Backward compatible | With defaults | Always (defaults for missing) | Depends |
| Human readable | JSON schema | .proto file | JSON |
| Schema registry | Confluent SR | Buf Schema Registry | JSON Schema Store |
| Best for | Kafka/Hadoop | gRPC/microservices | REST APIs |

---

## Interview Talking Points

> "When I joined, schema changes were causing 3-4 pipeline incidents per month. I introduced a data contract system where producers register schema changes in advance, and a CI check validates backward compatibility before merge. Incidents dropped to zero within two months."

> "I follow the expand-and-contract pattern for all column migrations. We add the new column as nullable, backfill, switch consumers, then remove the old column. This ensures zero-downtime migrations and gives us a clean rollback path at every step."

> "For our event streaming platform, I enforced BACKWARD_TRANSITIVE compatibility in our schema registry. This means any new schema version must be readable by ALL previous consumer versions, not just the immediately prior one. It's stricter but gives us confidence that no consumer, regardless of version, will break."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Renaming columns without coordination | Use expand-and-contract: add new name, migrate consumers, drop old |
| Changing column semantics without renaming | ALWAYS rename when meaning changes (revenue_usd not revenue) |
| Removing columns without usage verification | Check query audit logs; require 30 days of zero usage |
| Making columns NOT NULL without backfill | Add as NULL first, backfill, then add constraint |
| No schema registry for streaming data | Register all Kafka/event schemas; enforce compatibility |
| Reusing deleted field tags in Protobuf | Mark deleted tags as `reserved` permanently |
| Assuming all consumers update simultaneously | Design for version skew: old and new coexist for weeks |
| No deprecation grace period | Minimum 2-4 weeks between deprecated and removed |

---

## Rapid-Fire Q&A

**Q1: What's the difference between backward and forward compatibility?**
Backward: new code can read old data. Forward: old code can read new data. Full compatibility is both.

**Q2: What's an additive schema change?**
Adding a new optional column/field with a default value. It's safe because existing code ignores it and existing data gets the default.

**Q3: What's the most dangerous type of schema change?**
A semantic change — same column name, different meaning. It causes silent data corruption with no technical error signal.

**Q4: What's a schema registry?**
A centralized service that stores schema versions and validates compatibility before allowing new schemas to be registered. Acts as a gatekeeper for schema changes.

**Q5: How do you safely remove a column?**
Deprecate (document + notify), stop writing (zombie), verify zero queries via audit log, then physically drop after grace period.

**Q6: What's the expand-and-contract pattern?**
Add new column alongside old (expand), migrate data and consumers, then remove old column (contract). Provides zero-downtime migration.

**Q7: When should you prefer Avro over Protobuf?**
Avro for Hadoop/Kafka data pipelines (schema in data file, splittable). Protobuf for gRPC microservices (smaller wire format, strong typing).

**Q8: Why should field tags never be reused in Protobuf?**
Old serialized messages still contain the old tag. If reused with a different type/meaning, old messages will be deserialized incorrectly — silent corruption.

**Q9: What's a data contract?**
A formal agreement specifying schema, quality thresholds, SLAs, and change management processes between data producer and consumer.

**Q10: How do you handle a column type change (e.g., INT to BIGINT)?**
For widening (INT → BIGINT): usually safe in most databases (ALTER TABLE). For incompatible changes (INT → STRING): use expand-and-contract with a new column name.

---

## ASCII Cheat Sheet

```
+============================================================+
|      SCHEMA EVOLUTION & COMPATIBILITY — CHEAT SHEET         |
+============================================================+

CHANGE CLASSIFICATION:
  SAFE (Additive):
    + Add optional column (with default/NULL)
    + Add new table or view
    + Widen column type (INT → BIGINT)
    + Add new enum value at end
  
  BREAKING:
    - Remove column
    - Rename column  
    - Change type (incompatible)
    - Change semantics (same name, different meaning!)
    - Add required column without default

COMPATIBILITY TYPES:
  Backward:  New reader reads old data      ✓
  Forward:   Old reader reads new data      ✓
  Full:      Both directions work           ✓✓
  None:      No guarantees                  ✗

MIGRATION PATTERN (Expand-and-Contract):
  1. EXPAND:     Add new column alongside old
  2. MIGRATE:    Backfill historical data
  3. TRANSITION: Switch consumers to new column
  4. CONTRACT:   Remove old column

DEPRECATION LIFECYCLE:
  ACTIVE → DEPRECATED (2-4 weeks) → ZOMBIE (2-4 weeks) → REMOVED
           (still written)          (NULL for new rows)   (dropped)
  
  REQUIRED: Zero queries for 30 days before REMOVED

PROTOBUF RULES:
  - NEVER reuse tag numbers
  - NEVER change a tag's type (incompatibly)
  - Adding fields: ALWAYS safe
  - Removing fields: mark as `reserved`

AVRO RULES:
  - New field WITH default: backward compatible
  - New field WITHOUT default: BREAKING
  - Remove field WITH default: forward compatible
  - Rename: Use aliases, don't rename

SCHEMA REGISTRY MODES:
  BACKWARD:              New reads old (1 version)
  BACKWARD_TRANSITIVE:   New reads ALL old versions
  FORWARD:               Old reads new (1 version)
  FORWARD_TRANSITIVE:    Old reads ALL new versions
  FULL_TRANSITIVE:       All directions, all versions

NULLABLE vs REQUIRED:
  Warehouse columns:    Bias toward NULLABLE (flexibility)
  Transactional:        Bias toward REQUIRED (guarantees)
  New columns:          ALWAYS start nullable, constrain later

+============================================================+
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 42 of 45*
