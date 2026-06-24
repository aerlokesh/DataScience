# 🎯 Topic 10: Pandas Advanced Techniques

> **Scope:** Advanced pandas patterns for production data science — MultiIndex manipulation, memory optimization, time series operations, string vectorization, and analytical patterns (cohort, funnel, RFM). Targeted at DS/DA roles with 5 YOE interviewing at big tech companies.

---

## Table of Contents

1. [MultiIndex & Hierarchical Indexing](#1-multiindex--hierarchical-indexing)
2. [pipe() and Method Chaining](#2-pipe-and-method-chaining)
3. [eval() and query() for Performance](#3-eval-and-query-for-performance)
4. [Categorical Dtype & Memory Optimization](#4-categorical-dtype--memory-optimization)
5. [Datetime Operations & Time Series](#5-datetime-operations--time-series)
6. [String Operations (.str accessor)](#6-string-operations-str-accessor)
7. [merge_asof for Inexact Joins](#7-merge_asof-for-inexact-joins)
8. [Practical Patterns: Cohort, Funnel, RFM, Outliers](#8-practical-patterns-cohort-funnel-rfm-outliers)
9. [Interview Talking Points & Scripts](#9-interview-talking-points--scripts)
10. [Common Interview Mistakes](#10-common-interview-mistakes)
11. [Rapid-Fire Q&A](#11-rapid-fire-qa)
12. [Summary Cheat Sheet](#12-summary-cheat-sheet)

---

## 1. MultiIndex & Hierarchical Indexing

MultiIndex enables N-dimensional data representation in a 2D DataFrame. It is essential for panel data, grouped aggregations, and pivot operations.

```python
import pandas as pd
import numpy as np

# Creating MultiIndex from tuples
arrays = [['US', 'US', 'UK', 'UK'], ['2023', '2024', '2023', '2024']]
index = pd.MultiIndex.from_arrays(arrays, names=['country', 'year'])
df = pd.DataFrame({'revenue': [100, 120, 80, 95]}, index=index)

# Cross-section slicing
df.xs('US', level='country')           # All years for US
df.xs('2024', level='year')            # All countries for 2024

# Swapping and sorting levels
df.swaplevel().sort_index()

# Resetting specific levels back to columns
df.reset_index(level='year')

# stack/unstack for reshaping
wide = df.unstack(level='year')        # year becomes columns
long = wide.stack()                     # back to long format
```

> **Critical Insight:** Always call `.sort_index()` after creating or modifying a MultiIndex. Unsorted MultiIndex leads to `PerformanceWarning` and breaks slicing with `loc[('A','B'):('C','D')]` because pandas relies on sorted order for efficient label-based lookups.

---

## 2. pipe() and Method Chaining

`pipe()` enables composable, readable data transformations by passing the entire DataFrame into a function.

```python
def remove_outliers(df, col, n_std=3):
    mean, std = df[col].mean(), df[col].std()
    return df[(df[col] - mean).abs() <= n_std * std]

def add_log_feature(df, col):
    return df.assign(**{f'log_{col}': np.log1p(df[col])})

def label_segments(df, col, bins, labels):
    return df.assign(segment=pd.cut(df[col], bins=bins, labels=labels))

# Clean pipeline — each step is testable independently
result = (
    raw_df
    .pipe(remove_outliers, col='revenue', n_std=3)
    .pipe(add_log_feature, col='revenue')
    .pipe(label_segments, col='revenue',
          bins=[0, 100, 500, np.inf],
          labels=['small', 'mid', 'large'])
)
```

> **Critical Insight:** `pipe()` functions should be pure — no side effects, no mutation of the input DataFrame. This makes pipelines reproducible and testable. In interviews, using `pipe()` signals production-quality thinking versus ad-hoc scripting.

---

## 3. eval() and query() for Performance

`eval()` and `query()` use Numexpr under the hood, reducing memory overhead for large DataFrames by avoiding intermediate arrays.

```python
# Standard approach (creates 3 temporary arrays)
mask = (df['A'] > 0) & (df['B'] < 100) & (df['C'] == 'active')
result = df[mask]

# eval/query approach (single pass, less memory)
result = df.query("A > 0 and B < 100 and C == 'active'")

# eval for computed columns
df.eval('margin = (revenue - cost) / revenue', inplace=False)

# Using local variables with @
threshold = 50
df.query("revenue > @threshold")
```

> **Critical Insight:** `eval()`/`query()` provide meaningful speedups only for DataFrames with 10,000+ rows. For small frames, the parsing overhead dominates. In interviews, mention this nuance to demonstrate awareness of performance tradeoffs.

---

## 4. Categorical Dtype & Memory Optimization

### Dtype Memory Comparison

| Dtype | Bytes per Value | Best For | Example |
|-------|----------------|----------|---------|
| `int64` | 8 | Large integers, IDs | user_id |
| `int32` | 4 | Moderate integers | age, count |
| `int16` | 2 | Small integers (< 32K) | quantity |
| `int8` | 1 | Tiny integers (< 128) | rating (1-5) |
| `float64` | 8 | Full precision decimals | lat/lon |
| `float32` | 4 | Reduced precision OK | prices |
| `object` (string) | 50-100+ | Unique text | names |
| `category` | 1-8 (codes) + dict | Low-cardinality strings | country, status |
| `bool` | 1 | Binary flags | is_active |

### Memory Optimization Techniques

```python
def optimize_dtypes(df):
    """Downcast numeric columns and convert low-cardinality strings to category."""
    optimized = df.copy()
    
    for col in optimized.select_dtypes(include=['int']).columns:
        optimized[col] = pd.to_numeric(optimized[col], downcast='integer')
    
    for col in optimized.select_dtypes(include=['float']).columns:
        optimized[col] = pd.to_numeric(optimized[col], downcast='float')
    
    for col in optimized.select_dtypes(include=['object']).columns:
        n_unique = optimized[col].nunique()
        n_total = len(optimized[col])
        if n_unique / n_total < 0.5:  # < 50% unique → categorical
            optimized[col] = optimized[col].astype('category')
    
    return optimized

# Measure savings
before = df.memory_usage(deep=True).sum() / 1e6
df_opt = optimize_dtypes(df)
after = df_opt.memory_usage(deep=True).sum() / 1e6
print(f"Memory: {before:.1f}MB → {after:.1f}MB ({(1 - after/before)*100:.0f}% reduction)")
```

> **Critical Insight:** Categorical dtype preserves ordering when created with `ordered=True`, enabling comparison operators (`<`, `>`) on categories like 'low' < 'medium' < 'high'. This is critical for ordinal features in ML pipelines.

---

## 5. Datetime Operations & Time Series

### The .dt Accessor

```python
df['date'] = pd.to_datetime(df['date_str'])

# Extract components
df['year'] = df['date'].dt.year
df['quarter'] = df['date'].dt.quarter
df['day_of_week'] = df['date'].dt.day_name()
df['is_weekend'] = df['date'].dt.dayofweek >= 5
df['days_since_epoch'] = (df['date'] - pd.Timestamp('2020-01-01')).dt.days

# Business day operations
df['next_bday'] = df['date'] + pd.offsets.BDay(1)
```

### resample() vs groupby() for Time Series

| Feature | `resample()` | `groupby(pd.Grouper())` |
|---------|-------------|-------------------------|
| **Requires** | DatetimeIndex | Any datetime column |
| **Handles gaps** | Fills missing periods automatically | Skips missing periods |
| **Upsampling** | Supported (ffill, interpolate) | Not designed for this |
| **Multiple group keys** | Not directly | Combine with other columns |
| **Use case** | Pure time aggregation | Time + category aggregation |

```python
# resample: requires DatetimeIndex
ts = df.set_index('date')
monthly = ts.resample('M')['revenue'].agg(['sum', 'mean', 'count'])

# groupby with Grouper: works with any column + datetime
quarterly = df.groupby([
    pd.Grouper(key='date', freq='Q'),
    'region'
])['revenue'].sum().unstack()

# Rolling windows on time series
ts['revenue_7d_ma'] = ts['revenue'].rolling('7D').mean()
ts['revenue_expanding_max'] = ts['revenue'].expanding().max()
```

> **Critical Insight:** `resample()` automatically inserts NaN rows for missing time periods, making it ideal for detecting data gaps. `groupby` silently skips missing periods, which can hide data quality issues in production.

---

## 6. String Operations (.str accessor)

```python
# Vectorized string operations — avoid apply() with lambdas
df['email_domain'] = df['email'].str.split('@').str[1].str.lower()
df['name_clean'] = df['name'].str.strip().str.title()
df['has_promo'] = df['description'].str.contains(r'promo|discount|sale', case=False, regex=True)

# Extract structured data from messy strings
df['amount'] = df['price_str'].str.extract(r'\$(\d+\.?\d*)').astype(float)

# Efficient categorical string matching
df['category'] = np.select(
    [
        df['product'].str.contains('phone|mobile', case=False),
        df['product'].str.contains('laptop|computer', case=False),
        df['product'].str.contains('tablet|ipad', case=False),
    ],
    ['mobile', 'computing', 'tablet'],
    default='other'
)
```

> **Critical Insight:** `.str` methods are 5-50x faster than `.apply(lambda x: ...)` for string operations because they use optimized C/Cython paths. In interviews, always prefer vectorized `.str` over row-wise apply — it signals you understand pandas performance model.

---

## 7. merge_asof for Inexact Joins

`merge_asof` performs a left join on the nearest key rather than exact matches — essential for aligning time series that don't share exact timestamps.

```python
# Stock trades matched to most recent quote
trades = pd.DataFrame({
    'time': pd.to_datetime(['10:00:01', '10:00:03', '10:00:05']),
    'ticker': ['AAPL', 'AAPL', 'AAPL'],
    'quantity': [100, 200, 150]
})
quotes = pd.DataFrame({
    'time': pd.to_datetime(['10:00:00', '10:00:02', '10:00:04']),
    'ticker': ['AAPL', 'AAPL', 'AAPL'],
    'bid': [149.5, 149.7, 149.6]
})

# Each trade gets the most recent quote (backward direction)
result = pd.merge_asof(
    trades.sort_values('time'),
    quotes.sort_values('time'),
    on='time',
    by='ticker',
    direction='backward',
    tolerance=pd.Timedelta('2s')  # max lookback window
)
```

> **Critical Insight:** Both DataFrames must be sorted by the `on` key before calling `merge_asof`. The `tolerance` parameter prevents stale matches — crucial in production to avoid joining data that is too old to be relevant.

---

## 8. Practical Patterns: Cohort, Funnel, RFM, Outliers

### Cohort Analysis

```python
def cohort_analysis(df, user_col='user_id', date_col='order_date', value_col='revenue'):
    """Monthly cohort retention analysis."""
    df = df.copy()
    df[date_col] = pd.to_datetime(df[date_col])
    df['order_month'] = df[date_col].dt.to_period('M')
    
    # Assign cohort = first purchase month per user
    df['cohort'] = df.groupby(user_col)[date_col].transform('min').dt.to_period('M')
    
    # Cohort age in months
    df['cohort_age'] = (df['order_month'] - df['cohort']).apply(lambda x: x.n)
    
    # Pivot: cohort x age → unique users
    cohort_table = df.groupby(['cohort', 'cohort_age'])[user_col].nunique().unstack(fill_value=0)
    
    # Convert to retention rates
    cohort_sizes = cohort_table.iloc[:, 0]
    retention = cohort_table.divide(cohort_sizes, axis=0)
    
    return retention
```

### Funnel Analysis

```python
def funnel_analysis(df, user_col='user_id', event_col='event', stage_order=None):
    """Calculate conversion funnel with drop-off rates."""
    if stage_order is None:
        stage_order = df[event_col].unique().tolist()
    
    funnel = []
    for stage in stage_order:
        users_at_stage = df[df[event_col] == stage][user_col].nunique()
        funnel.append({'stage': stage, 'users': users_at_stage})
    
    funnel_df = pd.DataFrame(funnel)
    funnel_df['conversion_rate'] = funnel_df['users'] / funnel_df['users'].iloc[0]
    funnel_df['stage_dropoff'] = 1 - funnel_df['users'] / funnel_df['users'].shift(1)
    funnel_df['stage_dropoff'].iloc[0] = 0
    
    return funnel_df
```

### RFM Segmentation

```python
def rfm_segmentation(df, user_col='user_id', date_col='order_date',
                     value_col='revenue', ref_date=None):
    """RFM segmentation with quintile scoring."""
    if ref_date is None:
        ref_date = df[date_col].max() + pd.Timedelta(days=1)
    
    rfm = df.groupby(user_col).agg(
        recency=(date_col, lambda x: (ref_date - x.max()).days),
        frequency=(date_col, 'count'),
        monetary=(value_col, 'sum')
    ).reset_index()
    
    # Score 1-5 (5 is best)
    rfm['R_score'] = pd.qcut(rfm['recency'], 5, labels=[5, 4, 3, 2, 1])
    rfm['F_score'] = pd.qcut(rfm['frequency'].rank(method='first'), 5, labels=[1, 2, 3, 4, 5])
    rfm['M_score'] = pd.qcut(rfm['monetary'].rank(method='first'), 5, labels=[1, 2, 3, 4, 5])
    
    rfm['RFM_score'] = (rfm['R_score'].astype(int) * 100 +
                        rfm['F_score'].astype(int) * 10 +
                        rfm['M_score'].astype(int))
    
    # Segment mapping
    def segment(row):
        r, f, m = int(row['R_score']), int(row['F_score']), int(row['M_score'])
        if r >= 4 and f >= 4:
            return 'Champions'
        elif r >= 3 and f >= 3:
            return 'Loyal'
        elif r >= 4 and f <= 2:
            return 'New Customers'
        elif r <= 2 and f >= 3:
            return 'At Risk'
        elif r <= 2 and f <= 2:
            return 'Lost'
        else:
            return 'Need Attention'
    
    rfm['segment'] = rfm.apply(segment, axis=1)
    return rfm
```

### Outlier Detection with IQR

```python
def detect_outliers_iqr(df, columns=None, factor=1.5):
    """Detect outliers using IQR method, return boolean mask and bounds."""
    if columns is None:
        columns = df.select_dtypes(include=[np.number]).columns
    
    outlier_mask = pd.DataFrame(False, index=df.index, columns=columns)
    bounds = {}
    
    for col in columns:
        Q1 = df[col].quantile(0.25)
        Q3 = df[col].quantile(0.75)
        IQR = Q3 - Q1
        lower = Q1 - factor * IQR
        upper = Q3 + factor * IQR
        
        outlier_mask[col] = (df[col] < lower) | (df[col] > upper)
        bounds[col] = {'lower': lower, 'upper': upper, 'n_outliers': outlier_mask[col].sum()}
    
    # Any row that is an outlier in at least one column
    any_outlier = outlier_mask.any(axis=1)
    
    return any_outlier, bounds
```

---

## 9. Interview Talking Points & Scripts

> *"When optimizing pandas memory in production, my first step is always profiling with `df.memory_usage(deep=True)`. I then apply a three-pronged strategy: downcast numerics to the smallest dtype that can hold the range, convert low-cardinality object columns to categorical, and for string-heavy datasets, evaluate whether PyArrow-backed string dtype is appropriate. In a recent project, this reduced memory from 12GB to 3.2GB, allowing us to process on a single machine instead of spinning up a Spark cluster."*

> *"For time series aggregation, I choose between `resample()` and `groupby(pd.Grouper())` based on the analysis requirements. If I need to detect gaps in data collection — for example, monitoring pipeline health — I use `resample()` because it explicitly surfaces missing periods as NaN. If I need to aggregate across both time and categorical dimensions, like revenue by quarter and region, `groupby` with `pd.Grouper` is more natural because it composes cleanly with other group keys."*

> *"I structure my pandas code using `pipe()` chains because it makes the transformation logic auditable and testable. Each function in the pipeline has a single responsibility, accepts and returns a DataFrame, and can be unit-tested independently. This pattern also makes it trivial to toggle steps on and off during experimentation without rewriting the pipeline."*

> *"For merge_asof, I use it whenever I need to join datasets that are aligned by time but not at identical timestamps — for example, joining user clickstream events to the most recent A/B test assignment, or aligning sensor readings from different devices that sample at different rates. The key production safeguard is always setting a tolerance to prevent stale matches."*

> *"In cohort analysis, the critical implementation detail is correctly assigning each user's cohort — their first activity period. I use `groupby().transform('min')` to vectorize this rather than a merge-based approach, which keeps the operation efficient on large datasets. The retention matrix then falls naturally out of a pivot on cohort age."*

---

## 10. Common Interview Mistakes

| Mistake | Why It's Wrong | Correct Approach |
|---------|---------------|-----------------|
| Using `apply(lambda x: x.upper())` on strings | Row-wise Python loop, 50x slower | Use `.str.upper()` vectorized accessor |
| Not sorting before `merge_asof` | Silent incorrect results or error | Always `.sort_values()` on the join key |
| Ignoring memory with `object` dtype | 10x memory waste for repetitive strings | Convert to `category` when cardinality < 50% |
| Using `iterrows()` for transformations | Python-speed iteration over C-speed data | Use vectorized ops, `np.where`, or `np.select` |
| Chaining `df[df.col > x][col2]` | Returns copy, triggers SettingWithCopyWarning | Use `.loc[mask, col]` for safe selection |
| Hardcoding timezone-naive comparisons | Breaks on DST transitions and UTC offsets | Use `pd.to_datetime(..., utc=True)` then localize |
| Forgetting `fill_value` in pivot/unstack | NaN propagates silently through calculations | Specify `fill_value=0` or handle NaN explicitly |
| Using `inplace=True` everywhere | Breaks method chaining, no performance benefit | Assign result: `df = df.dropna()` |
| Aggregating without checking group sizes | Small groups produce unreliable statistics | Filter groups with `groupby().filter(lambda x: len(x) >= n)` |

---

## 11. Rapid-Fire Q&A

**Q1: How does `pd.Categorical` reduce memory?**
A: It stores an integer code array (1-8 bytes per row) plus a small lookup dictionary of unique values, instead of repeating full strings/objects for every row.

**Q2: When would you use `eval()` over standard pandas expressions?**
A: For complex boolean masks on DataFrames with 10K+ rows where multiple intermediate arrays would be created — `eval()` avoids temporary memory allocation.

**Q3: What's the difference between `.resample('D').sum()` and `.groupby(df.index.date).sum()`?**
A: `resample` inserts rows for missing dates with NaN/0; groupby only includes dates present in data. Resample requires DatetimeIndex; groupby works on any column.

**Q4: How do you handle a MultiIndex after groupby?**
A: Use `.reset_index()` to flatten, or `.xs(key, level=n)` for cross-sections, or `.droplevel(n)` to remove unnecessary levels.

**Q5: What's the risk of `merge_asof` without `tolerance`?**
A: It will match to arbitrarily old records if no closer match exists — potentially joining data from days or weeks ago, producing silently incorrect analysis.

**Q6: How does `pipe()` differ from `apply()`?**
A: `pipe()` passes the entire DataFrame to a function (DataFrame-in, DataFrame-out); `apply()` passes individual rows or columns. Pipe is for composing transformations; apply is for element/row-level computation.

**Q7: Why prefer `np.select()` over nested `np.where()`?**
A: Readability and maintainability — `np.select` takes a list of conditions and choices, while nested `np.where` becomes unreadable beyond 2-3 levels.

**Q8: How do you detect if a column should be categorical?**
A: Check `df[col].nunique() / len(df)` — if the ratio is below 0.5 (or the column has fewer than ~1000 unique values in millions of rows), categorical saves significant memory.

**Q9: What's the `.str` accessor limitation with NaN?**
A: `.str` methods propagate NaN gracefully (return NaN for NaN inputs), but you must ensure the column is string dtype — numeric columns need `.astype(str)` first.

**Q10: How would you optimize a 50GB CSV load in pandas?**
A: Use `dtype` parameter to specify efficient types upfront, `usecols` to load only needed columns, `chunksize` for streaming processing, or switch to PyArrow/Parquet format.

**Q11: When does `category` dtype hurt performance?**
A: When cardinality is high (near-unique values), when you frequently add new categories not in the original set, or when the operation doesn't benefit from integer codes (e.g., arbitrary string concatenation).

**Q12: How do you align two time series with different frequencies?**
A: Use `merge_asof` for point-in-time alignment, or `resample` one series to match the other's frequency (e.g., upsample daily to hourly with `ffill`, or downsample hourly to daily with `mean`).

---

## 12. Summary Cheat Sheet

```
+============================================================================+
|                    PANDAS ADVANCED — CHEAT SHEET                            |
+============================================================================+
|                                                                            |
|  MULTIINDEX                                                                |
|    .xs(key, level=)    cross-section slice                                 |
|    .swaplevel()        reorder hierarchy                                   |
|    .unstack(level=)    pivot level to columns                              |
|    Always sort_index() after creation                                      |
|                                                                            |
|  PIPE & CHAINING                                                           |
|    df.pipe(func, **kw)  DataFrame → func(df, **kw) → DataFrame            |
|    Keep functions pure: no side effects, no inplace                        |
|                                                                            |
|  MEMORY OPTIMIZATION                                                       |
|    pd.to_numeric(s, downcast='integer'|'float')                            |
|    .astype('category')  when nunique/len < 0.5                             |
|    df.memory_usage(deep=True)  always profile first                        |
|                                                                            |
|  DATETIME / TIME SERIES                                                    |
|    .dt.year/month/day/dayofweek/quarter                                    |
|    .resample('M')      time bucketing (fills gaps)                         |
|    pd.Grouper(freq=)   time + other group keys                             |
|    .rolling('7D')      window by timedelta                                 |
|                                                                            |
|  STRING OPS                                                                |
|    .str.contains/extract/split/replace/strip/lower                         |
|    Always prefer .str over .apply(lambda)                                  |
|                                                                            |
|  MERGE_ASOF                                                                |
|    pd.merge_asof(left, right, on=, by=, direction=, tolerance=)            |
|    Both frames MUST be sorted on the 'on' key                              |
|                                                                            |
|  ANALYTICAL PATTERNS                                                       |
|    Cohort:  groupby().transform('min') → period arithmetic → pivot         |
|    Funnel:  sequential nunique per stage → conversion rates                |
|    RFM:     recency/frequency/monetary → qcut scores → segments            |
|    Outlier: Q1/Q3/IQR → bounds at 1.5*IQR → boolean mask                  |
|                                                                            |
+============================================================================+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 10 of 25*
