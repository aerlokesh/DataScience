# 🎯 Topic 9: Pandas Data Manipulation

> **Scope:** The single most tested Python library in DS/DA interviews. Covers DataFrame creation, indexing, filtering, groupby, merges/joins, window functions, reshaping, and null handling — with emphasis on method chaining, performance awareness, and solving classic interview problems (top-N per group, YoY growth, running totals).

---

## Table of Contents

1. [DataFrame Creation & Structure](#1-dataframe-creation--structure)
2. [Selection: loc vs iloc](#2-selection-loc-vs-iloc)
3. [Filtering & Boolean Indexing](#3-filtering--boolean-indexing)
4. [GroupBy + Aggregation](#4-groupby--aggregation)
5. [Apply vs Transform vs Agg](#5-apply-vs-transform-vs-agg)
6. [Merge & Join Types](#6-merge--join-types)
7. [Window Functions: Rolling, Shift, Rank](#7-window-functions-rolling-shift-rank)
8. [Pivot & Melt (Reshaping)](#8-pivot--melt-reshaping)
9. [Handling Nulls](#9-handling-nulls)
10. [Classic Interview Problems](#10-classic-interview-problems)
11. [Interview Talking Points & Scripts](#11-interview-talking-points--scripts)
12. [Common Interview Mistakes](#12-common-interview-mistakes)
13. [Rapid-Fire Q&A](#13-rapid-fire-qa)
14. [Summary Cheat Sheet](#14-summary-cheat-sheet)

---

## 1. DataFrame Creation & Structure

```python
import pandas as pd
import numpy as np

# From dictionary
df = pd.DataFrame({
    'user_id': [1, 2, 3, 4],
    'revenue': [100, 250, 80, 320],
    'signup_date': pd.to_datetime(['2023-01-01', '2023-02-15', '2023-01-20', '2023-03-10']),
    'region': ['US', 'EU', 'US', 'APAC']
})

# From CSV with parsing
df = pd.read_csv('data.csv', parse_dates=['date_col'], dtype={'id': str})

# Key attributes
df.shape        # (rows, cols)
df.dtypes       # column types
df.info()       # memory usage + nulls
df.describe()   # summary stats for numeric columns
```

> **Critical Insight:** Always specify `dtype` and `parse_dates` at read time. Fixing types after loading is expensive and error-prone — interviewers notice when you skip this.

---

## 2. Selection: loc vs iloc

| Feature | `loc` | `iloc` |
|---------|-------|--------|
| Indexing type | Label-based | Integer position-based |
| Slice end | **Inclusive** | **Exclusive** |
| Accepts booleans | Yes | Yes |
| Accepts callables | Yes | Yes |
| Use when | Index has meaningful labels | Positional access needed |

```python
# loc — label-based (inclusive on both ends)
df.loc[0:2, 'user_id':'revenue']      # rows 0,1,2; columns user_id through revenue
df.loc[df['revenue'] > 100, 'region'] # filtered rows, single column

# iloc — integer position (exclusive end)
df.iloc[0:2, 0:2]                      # rows 0,1; columns 0,1
df.iloc[-1]                            # last row
```

> **Critical Insight:** The most common bug in interviews is confusing `loc`'s inclusive slicing with `iloc`'s exclusive slicing. When the index is a default RangeIndex, `df.loc[0:2]` returns 3 rows but `df.iloc[0:2]` returns 2.

---

## 3. Filtering & Boolean Indexing

```python
# Single condition
high_rev = df[df['revenue'] > 200]

# Multiple conditions — MUST use parentheses and bitwise operators
us_high = df[(df['region'] == 'US') & (df['revenue'] > 100)]

# isin for membership
target_regions = df[df['region'].isin(['US', 'EU'])]

# String methods
df[df['name'].str.contains('Smith', case=False, na=False)]

# query() for cleaner syntax (great for method chaining)
df.query("revenue > 200 and region == 'US'")
```

> **Critical Insight:** Using `and`/`or` instead of `&`/`|` is a runtime error in pandas. Interviewers use this to test whether you actually write pandas daily.

---

## 4. GroupBy + Aggregation

```python
# Basic aggregation
df.groupby('region')['revenue'].sum()

# Multiple aggregations
summary = df.groupby('region').agg(
    total_revenue=('revenue', 'sum'),
    avg_revenue=('revenue', 'mean'),
    user_count=('user_id', 'nunique')
).reset_index()

# Groupby with multiple keys
df.groupby(['region', 'year']).agg({'revenue': ['sum', 'mean', 'count']})
```

> **Critical Insight:** Always call `.reset_index()` after groupby if you need the grouped column back as a regular column. The named-agg syntax (`col=('source_col', 'func')`) is the modern, interview-preferred approach.

---

## 5. Apply vs Transform vs Agg

| Method | Input | Output Shape | Use Case |
|--------|-------|--------------|----------|
| `apply` | Group DataFrame or Series | Any shape (flexible) | Custom logic, multiple columns |
| `transform` | Group Series | Same shape as input | Broadcast group result back to rows |
| `agg` | Group Series | One row per group | Summary statistics |

```python
# transform — result aligns with original index
df['pct_of_group'] = df.groupby('region')['revenue'].transform(lambda x: x / x.sum())

# apply — flexible but slower
df.groupby('region').apply(lambda g: g.nlargest(2, 'revenue'))

# agg — summarize
df.groupby('region')['revenue'].agg(['mean', 'std', 'count'])
```

> **Critical Insight:** `transform` is the key to "add a group-level stat back to each row" problems. It preserves the original index, making alignment automatic. This is a top interview pattern.

---

## 6. Merge & Join Types

| Merge Type | Keeps Rows From | NULL Behavior | SQL Equivalent |
|------------|-----------------|---------------|----------------|
| `inner` | Both (matching only) | No NULLs from merge | INNER JOIN |
| `left` | All left, matching right | NULLs in right columns | LEFT OUTER JOIN |
| `right` | All right, matching left | NULLs in left columns | RIGHT OUTER JOIN |
| `outer` | All from both | NULLs on both sides | FULL OUTER JOIN |
| `cross` | Cartesian product | No NULLs | CROSS JOIN |

```python
# Standard merge
merged = pd.merge(orders, customers, on='customer_id', how='left')

# Merge on different column names
merged = pd.merge(df1, df2, left_on='emp_id', right_on='employee_id', how='inner')

# Multiple keys
merged = pd.merge(df1, df2, on=['year', 'region'], how='outer')

# Indicator to debug merge quality
merged = pd.merge(df1, df2, on='id', how='outer', indicator=True)
print(merged['_merge'].value_counts())
# both          150
# left_only      20
# right_only     10
```

> **Critical Insight:** Always check merge cardinality. A many-to-many merge silently explodes row count. Use `validate='one_to_many'` or check `len(merged)` vs `len(left)` immediately after merging.

```python
# Validate merge cardinality
merged = pd.merge(orders, customers, on='customer_id', how='left', validate='many_to_one')
```

---

## 7. Window Functions: Rolling, Shift, Rank

```python
# Rolling average (moving window)
df['revenue_7d_avg'] = df.sort_values('date').groupby('user_id')['revenue'].transform(
    lambda x: x.rolling(7, min_periods=1).mean()
)

# Shift — lag/lead for time comparisons
df['prev_month_revenue'] = df.sort_values('date').groupby('user_id')['revenue'].shift(1)
df['mom_growth'] = (df['revenue'] - df['prev_month_revenue']) / df['prev_month_revenue']

# Rank within groups
df['rank_in_region'] = df.groupby('region')['revenue'].rank(method='dense', ascending=False)

# Cumulative sum (running total)
df['cumulative_revenue'] = df.sort_values('date').groupby('user_id')['revenue'].cumsum()

# Expanding window (all prior rows)
df['lifetime_avg'] = df.sort_values('date').groupby('user_id')['revenue'].expanding().mean().reset_index(level=0, drop=True)
```

> **Critical Insight:** Window functions in pandas require explicit `sort_values` before applying — unlike SQL where `ORDER BY` is inside the window spec. Forgetting to sort is the #1 mistake in time-series interview problems.

---

## 8. Pivot & Melt (Reshaping)

```python
# Pivot — long to wide
pivot_df = df.pivot_table(
    index='region',
    columns='quarter',
    values='revenue',
    aggfunc='sum',
    fill_value=0
)

# Melt — wide to long
melted = pd.melt(
    wide_df,
    id_vars=['region'],
    value_vars=['Q1', 'Q2', 'Q3', 'Q4'],
    var_name='quarter',
    value_name='revenue'
)

# Unstack / Stack (MultiIndex reshaping)
df.groupby(['region', 'year'])['revenue'].sum().unstack(fill_value=0)
```

> **Critical Insight:** Use `pivot_table` (not `pivot`) in interviews — it handles duplicate index/column combinations via `aggfunc`. Plain `pivot` raises an error on duplicates, which is a common trap.

---

## 9. Handling Nulls

```python
# Detection
df.isnull().sum()                    # count nulls per column
df.isnull().sum() / len(df) * 100   # null percentage

# Filling strategies
df['revenue'].fillna(0)                              # constant
df['revenue'].fillna(df['revenue'].median())         # global median
df['revenue'].fillna(df.groupby('region')['revenue'].transform('median'))  # group median
df['revenue'].ffill()                                # forward fill (time series)

# Dropping
df.dropna(subset=['critical_col'])          # drop rows where critical_col is null
df.dropna(thresh=3)                         # keep rows with at least 3 non-null values

# Interpolation for time series
df['value'].interpolate(method='linear')
```

> **Critical Insight:** Never use `df.dropna()` without `subset` in production or interviews — it drops any row with ANY null, which can silently remove most of your data. Always be explicit about which columns matter.

---

## 10. Classic Interview Problems

### Problem 1: Top-N Per Group

*"Find the top 2 revenue-generating users per region."*

```python
# Method 1: nlargest within groupby
top2 = df.groupby('region').apply(lambda g: g.nlargest(2, 'revenue')).reset_index(drop=True)

# Method 2: rank + filter (more efficient for large data)
df['rank'] = df.groupby('region')['revenue'].rank(method='first', ascending=False)
top2 = df[df['rank'] <= 2].drop(columns='rank')
```

### Problem 2: Year-over-Year Growth

*"Calculate YoY revenue growth by region."*

```python
yearly = df.groupby(['region', 'year'])['revenue'].sum().reset_index()
yearly['prev_year_rev'] = yearly.sort_values('year').groupby('region')['revenue'].shift(1)
yearly['yoy_growth'] = (yearly['revenue'] - yearly['prev_year_rev']) / yearly['prev_year_rev']
```

### Problem 3: Running Total

*"Compute cumulative revenue per user, ordered by date."*

```python
df = df.sort_values(['user_id', 'date'])
df['running_total'] = df.groupby('user_id')['revenue'].cumsum()
```

### Problem 4: Sessionization (Gap-Based)

*"Group events into sessions with 30-min inactivity gaps."*

```python
df = df.sort_values(['user_id', 'timestamp'])
df['time_gap'] = df.groupby('user_id')['timestamp'].diff()
df['new_session'] = (df['time_gap'] > pd.Timedelta('30min')).astype(int)
df['session_id'] = df.groupby('user_id')['new_session'].cumsum()
```

### Problem 5: Pivot + Percentage of Total

*"Show each product's share of total revenue per region."*

```python
pivot = df.pivot_table(index='region', columns='product', values='revenue', aggfunc='sum', fill_value=0)
pct = pivot.div(pivot.sum(axis=1), axis=0) * 100
```

---

## 11. Interview Talking Points & Scripts

> *"When I work with pandas in production, I prioritize method chaining for readability and reproducibility. Chaining lets me read a transformation pipeline top-to-bottom without creating intermediate variables that clutter the namespace. I use `.pipe()` when I need to inject custom functions into the chain, and `.assign()` for adding columns mid-chain."*

> *"For the transform vs apply distinction — I reach for transform whenever I need a group-level statistic broadcast back to the original DataFrame shape, like computing each row's percentage of its group total. Apply is my fallback for complex logic that returns a different shape, but I'm aware it's slower because it doesn't benefit from the same vectorized optimizations."*

> *"On merge validation — in any production pipeline, I always pass `validate='one_to_many'` or `'many_to_one'` to catch unexpected duplicates early. A silent many-to-many merge can inflate row counts and corrupt downstream metrics without any error message."*

> *"For window functions, I always sort explicitly before applying rolling or shift operations. Unlike SQL where ORDER BY is part of the window spec, pandas relies on the physical row order of the DataFrame, so forgetting to sort produces silently wrong results."*

> *"When handling nulls, my approach depends on the data type and business context. For numeric features I typically use group-level medians to avoid introducing bias. For categorical data I use a dedicated 'Unknown' category. I never use global dropna without a subset parameter — that's a silent data destroyer."*

---

## 12. Common Interview Mistakes

| Mistake | Why It's Wrong | Correct Approach |
|---------|---------------|------------------|
| Using `and`/`or` in boolean filters | Raises ValueError; pandas needs element-wise ops | Use `&` / `|` with parentheses around each condition |
| Forgetting `reset_index()` after groupby | Grouped columns become index, breaking downstream ops | Chain `.reset_index()` or use `as_index=False` |
| Using `apply` when `transform` suffices | 10-100x slower; doesn't align with original index | Use `transform` for same-shape group operations |
| Not sorting before `shift()`/`rolling()` | Window computed on arbitrary row order | Always `sort_values()` by the time column first |
| Using `pivot` instead of `pivot_table` | Fails on duplicate index/column pairs | Use `pivot_table` with explicit `aggfunc` |
| Chaining assignments with `df['col'] = ...` inside chains | Breaks method chains, triggers SettingWithCopyWarning | Use `.assign(col=lambda x: ...)` |
| Ignoring merge cardinality | Silent row explosion on many-to-many | Pass `validate=` parameter or check row counts |
| `df.dropna()` without `subset` | Drops rows missing ANY value across ALL columns | Specify `subset=['col1', 'col2']` |
| Comparing with `== None` | Doesn't work for NaN detection | Use `.isnull()` or `.isna()` |
| Modifying a slice without `.copy()` | SettingWithCopyWarning; changes may not persist | Use `df_subset = df[condition].copy()` |

---

## 13. Rapid-Fire Q&A

**Q1: What's the difference between `loc` and `iloc`?**
A: `loc` is label-based (inclusive end), `iloc` is integer-position-based (exclusive end). Both accept boolean arrays.

**Q2: How do you add a group-level statistic back to each row?**
A: Use `groupby().transform()` — it returns a Series with the same index as the original DataFrame.

**Q3: What does `validate` do in `pd.merge()`?**
A: It raises `MergeError` if the merge keys violate the specified cardinality (e.g., `'one_to_many'` fails if left has duplicate keys).

**Q4: How do you compute a running total partitioned by user?**
A: `df.sort_values('date').groupby('user_id')['value'].cumsum()`

**Q5: What's the difference between `pivot` and `pivot_table`?**
A: `pivot_table` supports aggregation functions and handles duplicate index/column pairs; `pivot` raises an error on duplicates.

**Q6: How do you get the top-N rows per group efficiently?**
A: Use `groupby().rank()` then filter, or `groupby().apply(lambda g: g.nlargest(n, col))`. Rank is faster on large data.

**Q7: What does `shift(1)` do and why must you sort first?**
A: `shift(1)` moves values down by 1 position (lag). Without sorting by the time column, the lag references an arbitrary previous row.

**Q8: How do you handle SettingWithCopyWarning?**
A: Use `.copy()` when subsetting, or use `.assign()` / `.loc[]` for assignments. The warning means your write may silently fail.

**Q9: What's method chaining and why use it?**
A: Calling multiple DataFrame methods in sequence (`df.query().groupby().agg().reset_index()`). It improves readability, avoids intermediate variables, and makes pipelines declarative.

**Q10: How do you fill nulls differently per group?**
A: `df['col'].fillna(df.groupby('group')['col'].transform('median'))` — transform computes group medians aligned to original rows.

**Q11: When would you use `pd.melt()`?**
A: To convert wide-format data (columns as categories) into long-format (one row per observation). Essential before plotting or groupby operations on formerly-wide columns.

**Q12: How do you detect a many-to-many merge before it happens?**
A: Check `df.duplicated(subset=['key']).any()` on both sides, or pass `validate='one_to_one'` to let pandas raise the error.

---

## 14. Summary Cheat Sheet

```
+------------------------------------------------------------------+
|                  PANDAS DATA MANIPULATION CHEAT SHEET             |
+------------------------------------------------------------------+
| SELECTION                                                        |
|   loc[row_label, col_label]   — label-based, inclusive end       |
|   iloc[row_pos, col_pos]      — position-based, exclusive end    |
|   query("col > val")          — string expression, chain-friendly|
+------------------------------------------------------------------+
| FILTERING                                                        |
|   & (and), | (or), ~ (not)   — bitwise, with parentheses        |
|   .isin([])  .str.contains()  .between()  .isna()               |
+------------------------------------------------------------------+
| GROUPBY PATTERNS                                                 |
|   .agg(name=('col','func'))  — named aggregation (preferred)    |
|   .transform(func)           — broadcast back to original shape  |
|   .apply(func)               — flexible but slower               |
+------------------------------------------------------------------+
| MERGES                                                           |
|   pd.merge(left, right, on=, how=, validate=)                   |
|   how: inner | left | right | outer | cross                     |
|   Always check: len(merged) vs len(left)                         |
+------------------------------------------------------------------+
| WINDOW FUNCTIONS (always sort first!)                            |
|   .rolling(n).mean()    — moving average                         |
|   .shift(n)             — lag (positive) or lead (negative)      |
|   .cumsum()             — running total                          |
|   .rank(method=)        — dense, first, min, max, average        |
|   .expanding().mean()   — all-prior-rows aggregate               |
+------------------------------------------------------------------+
| RESHAPING                                                        |
|   pivot_table(index=, columns=, values=, aggfunc=)              |
|   pd.melt(id_vars=, value_vars=, var_name=, value_name=)        |
+------------------------------------------------------------------+
| NULLS                                                            |
|   .fillna(val)  .ffill()  .bfill()  .interpolate()              |
|   .dropna(subset=[])  — NEVER bare dropna()                     |
|   Group fill: .fillna(groupby().transform('median'))            |
+------------------------------------------------------------------+
| METHOD CHAINING TOOLKIT                                          |
|   .assign(new_col=lambda x: ...)                                 |
|   .pipe(custom_function)                                         |
|   .query()  .sort_values()  .reset_index()                       |
+------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 9 of 25*
