# 🎯 Topic 23: Data Cleaning & Preprocessing

> A comprehensive interview guide covering the full spectrum of data cleaning and preprocessing techniques — from duplicate detection and outlier treatment to pipeline patterns and large-scale data strategies. Tailored for Data Scientists and Data Analysts with ~5 years of experience interviewing at Google, Meta, Amazon, Netflix, Spotify, and Uber.

---

## Table of Contents

1. [Handling Duplicates](#1-handling-duplicates)
2. [Data Type Issues](#2-data-type-issues)
3. [Outlier Detection and Treatment](#3-outlier-detection-and-treatment)
4. [Inconsistent Data](#4-inconsistent-data)
5. [Date Parsing and Timezones](#5-date-parsing-and-timezones)
6. [Regex for Cleaning](#6-regex-for-cleaning)
7. [Handling Mixed Types in Columns](#7-handling-mixed-types-in-columns)
8. [Large Dataset Strategies](#8-large-dataset-strategies)
9. [Pipeline Patterns](#9-pipeline-patterns)
10. [Data Quality Checks](#10-data-quality-checks)
11. [Interview Talking Points and Scripts](#11-interview-talking-points-and-scripts)
12. [Common Interview Mistakes](#12-common-interview-mistakes)
13. [Rapid-Fire Q&A](#13-rapid-fire-qa)
14. [Summary Cheat Sheet](#14-summary-cheat-sheet)

---

## 1. Handling Duplicates

### Exact Duplicates

```python
import pandas as pd

# Detect exact duplicates
df.duplicated().sum()
df[df.duplicated(subset=['user_id', 'timestamp'], keep=False)]

# Remove duplicates keeping first occurrence
df_clean = df.drop_duplicates(subset=['user_id', 'event_type', 'timestamp'], keep='first')

# Aggregate duplicates instead of dropping
df_agg = df.groupby(['user_id', 'date']).agg(
    total_revenue=('revenue', 'sum'),
    num_transactions=('transaction_id', 'count'),
    last_event=('timestamp', 'max')
).reset_index()
```

### Fuzzy Matching with rapidfuzz

```python
from rapidfuzz import fuzz, process

# Deduplicate company names with fuzzy matching
def deduplicate_fuzzy(series, threshold=85):
    """Group similar strings and map to canonical form."""
    canonical_map = {}
    unique_values = series.dropna().unique()
    
    for value in unique_values:
        if value in canonical_map:
            continue
        # Find all matches above threshold
        matches = process.extract(
            value, unique_values, scorer=fuzz.token_sort_ratio, limit=None
        )
        similar = [m[0] for m in matches if m[1] >= threshold]
        # Use the shortest string as canonical (often the cleanest)
        canonical = min(similar, key=len)
        for s in similar:
            canonical_map[s] = canonical
    
    return series.map(canonical_map)

df['company_clean'] = deduplicate_fuzzy(df['company_name'], threshold=88)
```

> **Critical Insight:** At scale (millions of rows), pairwise fuzzy matching is O(n^2). Use blocking strategies — group by first letter, zip code, or phonetic encoding (Soundex/Metaphone) — then fuzzy-match only within blocks. This reduces complexity from millions of comparisons to thousands.

### Comparison: Duplicate Detection Methods

| Method | Use Case | Complexity | Accuracy |
|--------|----------|-----------|----------|
| `df.duplicated()` | Exact matches on all/subset of columns | O(n) | Perfect for exact |
| `fuzzywuzzy` / `rapidfuzz` | Typos, abbreviations, formatting differences | O(n^2) without blocking | High with tuned threshold |
| Record linkage (dedupe library) | Probabilistic matching across datasets | O(n log n) with blocking | Very high |
| Phonetic (Soundex, Metaphone) | Name matching across transliterations | O(n) | Moderate |
| Embedding similarity | Semantic deduplication of text | O(n) with ANN index | Context-dependent |

---

## 2. Data Type Issues

### Detecting and Coercing Mixed Types

```python
# Identify columns with mixed types
def detect_mixed_types(df):
    """Report columns where values have inconsistent types."""
    mixed = {}
    for col in df.columns:
        types = df[col].dropna().apply(type).value_counts()
        if len(types) > 1:
            mixed[col] = types.to_dict()
    return mixed

# Safe numeric coercion
df['revenue'] = pd.to_numeric(df['revenue'], errors='coerce')

# Coerce with logging of failures
mask = pd.to_numeric(df['price'], errors='coerce').isna() & df['price'].notna()
failed_values = df.loc[mask, 'price'].unique()
print(f"Could not convert {len(failed_values)} unique values: {failed_values[:10]}")
df['price'] = pd.to_numeric(df['price'], errors='coerce')
```

### Type Optimization for Memory

```python
def optimize_dtypes(df):
    """Downcast numeric types and convert object columns to category."""
    for col in df.select_dtypes(include=['int64']).columns:
        df[col] = pd.to_numeric(df[col], downcast='integer')
    for col in df.select_dtypes(include=['float64']).columns:
        df[col] = pd.to_numeric(df[col], downcast='float')
    for col in df.select_dtypes(include=['object']).columns:
        nunique_ratio = df[col].nunique() / len(df)
        if nunique_ratio < 0.5:  # Low cardinality -> category
            df[col] = df[col].astype('category')
    return df
```

> **Critical Insight:** In interviews, always mention that `errors='coerce'` silently produces NaN. A production pipeline should log what was coerced and why, enabling you to fix upstream issues rather than masking them.

---

## 3. Outlier Detection and Treatment

### IQR Method

```python
def detect_outliers_iqr(series, multiplier=1.5):
    """Identify outliers using the Interquartile Range method."""
    Q1 = series.quantile(0.25)
    Q3 = series.quantile(0.75)
    IQR = Q3 - Q1
    lower_bound = Q1 - multiplier * IQR
    upper_bound = Q3 + multiplier * IQR
    return (series < lower_bound) | (series > upper_bound)

outlier_mask = detect_outliers_iqr(df['transaction_amount'])
print(f"Outliers detected: {outlier_mask.sum()} ({outlier_mask.mean():.2%})")
```

### Z-Score and Modified Z-Score

```python
import numpy as np
from scipy import stats

# Standard Z-Score (assumes normality)
def detect_outliers_zscore(series, threshold=3.0):
    z_scores = np.abs(stats.zscore(series.dropna()))
    return z_scores > threshold

# Modified Z-Score (robust to outliers themselves)
def detect_outliers_modified_zscore(series, threshold=3.5):
    """Uses median and MAD instead of mean and std."""
    median = series.median()
    mad = np.median(np.abs(series - median))
    modified_z = 0.6745 * (series - median) / mad  # 0.6745 is the 0.75 quantile of N(0,1)
    return np.abs(modified_z) > threshold
```

### Treatment Strategies

```python
# Winsorization (cap at percentiles)
from scipy.stats import mstats

df['amount_winsorized'] = mstats.winsorize(df['amount'], limits=[0.01, 0.01])

# Log transformation (for right-skewed data)
df['amount_log'] = np.log1p(df['amount'])  # log1p handles zeros

# Capping at domain-specific bounds
df['age'] = df['age'].clip(lower=0, upper=120)

# Isolation Forest for multivariate outliers
from sklearn.ensemble import IsolationForest

iso_forest = IsolationForest(contamination=0.05, random_state=42)
df['is_outlier'] = iso_forest.fit_predict(df[['amount', 'frequency', 'recency']]) == -1
```

> **Critical Insight:** Never remove outliers mechanically. Ask: (1) Is this a data entry error? Fix or remove. (2) Is this a genuine extreme value? Keep it — it may be your most important signal (e.g., whale users at Spotify, high-value transactions at Uber). (3) Does the outlier break your model assumptions? Transform or use robust methods.

### Comparison: Outlier Detection Methods

| Method | Assumptions | Strengths | Weaknesses |
|--------|-------------|-----------|------------|
| IQR (1.5x) | None (non-parametric) | Simple, robust to extreme values | Fixed threshold, univariate only |
| Z-Score | Normal distribution | Intuitive, standard | Sensitive to outliers in mean/std |
| Modified Z-Score | Symmetric distribution | Robust (uses median/MAD) | Still univariate |
| Isolation Forest | None | Multivariate, handles complex patterns | Black box, needs tuning |
| DBSCAN | Density-based clusters | Finds clusters and noise together | Sensitive to epsilon parameter |
| Mahalanobis Distance | Multivariate normal | Accounts for correlations | Assumes MVN, needs n > p |

---

## 4. Inconsistent Data

### Categorical Normalization

```python
# Standardize categorical values
def normalize_categorical(series):
    """Strip, lowercase, and map common variations."""
    cleaned = series.str.strip().str.lower().str.replace(r'\s+', ' ', regex=True)
    return cleaned

# Mapping dictionaries for known variations
gender_map = {
    'm': 'male', 'f': 'female', 'male': 'male', 'female': 'female',
    'man': 'male', 'woman': 'female', 'nb': 'non-binary',
    'non-binary': 'non-binary', 'nonbinary': 'non-binary'
}
df['gender'] = df['gender'].str.strip().str.lower().map(gender_map)

# Country name standardization
country_map = {
    'us': 'United States', 'usa': 'United States', 'united states': 'United States',
    'uk': 'United Kingdom', 'united kingdom': 'United Kingdom',
    'great britain': 'United Kingdom', 'gb': 'United Kingdom'
}
df['country'] = df['country'].str.strip().str.lower().map(country_map).fillna(df['country'])
```

### Encoding Issues

```python
# Detect and fix encoding problems
import chardet

def detect_encoding(file_path, sample_size=10000):
    """Detect file encoding from a sample."""
    with open(file_path, 'rb') as f:
        result = chardet.detect(f.read(sample_size))
    return result['encoding'], result['confidence']

# Read with correct encoding
encoding, confidence = detect_encoding('data.csv')
df = pd.read_csv('data.csv', encoding=encoding)

# Fix mojibake (double-encoded UTF-8)
def fix_mojibake(text):
    """Attempt to fix common double-encoding issues."""
    try:
        return text.encode('latin-1').decode('utf-8')
    except (UnicodeDecodeError, UnicodeEncodeError, AttributeError):
        return text

df['name'] = df['name'].apply(fix_mojibake)
```

> **Critical Insight:** At companies like Uber and Spotify operating globally, encoding issues are extremely common. Always specify encoding explicitly when reading files, and validate that string lengths and character distributions make sense after loading.

---

## 5. Date Parsing and Timezones

### Robust Date Parsing

```python
import pytz
from datetime import datetime

# Parse mixed date formats
df['date'] = pd.to_datetime(df['date_str'], infer_datetime_format=True, errors='coerce')

# Handle specific ambiguous formats
df['date'] = pd.to_datetime(df['date_str'], format='mixed', dayfirst=False)

# Multiple format fallback
def parse_dates_multi(series, formats=None):
    """Try multiple date formats in order."""
    if formats is None:
        formats = ['%Y-%m-%d', '%m/%d/%Y', '%d-%b-%Y', '%Y-%m-%dT%H:%M:%S', '%Y%m%d']
    
    result = pd.Series([pd.NaT] * len(series), index=series.index)
    remaining = series.notna()
    
    for fmt in formats:
        mask = remaining & result.isna()
        if not mask.any():
            break
        parsed = pd.to_datetime(series[mask], format=fmt, errors='coerce')
        result[mask] = parsed
    
    return result
```

### Timezone Handling

```python
# Make naive timestamps timezone-aware
df['event_time'] = pd.to_datetime(df['event_time'])
df['event_time_utc'] = df['event_time'].dt.tz_localize('UTC')

# Convert between timezones
df['event_time_pacific'] = df['event_time_utc'].dt.tz_convert('US/Pacific')

# Handle mixed timezones (e.g., user events from different regions)
def normalize_to_utc(row):
    """Convert local time to UTC based on user timezone."""
    local_tz = pytz.timezone(row['user_timezone'])
    local_time = local_tz.localize(row['event_time'])
    return local_time.astimezone(pytz.UTC)

df['event_time_utc'] = df.apply(normalize_to_utc, axis=1)

# DST-safe operations
df['date'] = df['event_time_utc'].dt.tz_convert('US/Eastern').dt.date
```

> **Critical Insight:** For analytics at global companies, always store timestamps in UTC and convert to local time only for display. When computing daily metrics, define "day" relative to the user's timezone — a user in Tokyo and a user in New York experience different calendar days at the same UTC instant.

---

## 6. Regex for Cleaning

### Phone Number Standardization

```python
import re

def clean_phone_number(phone):
    """Normalize phone numbers to E.164 format (+1XXXXXXXXXX for US)."""
    if pd.isna(phone):
        return None
    # Remove all non-digit characters
    digits = re.sub(r'\D', '', str(phone))
    # Handle country code
    if len(digits) == 10:  # US number without country code
        return f'+1{digits}'
    elif len(digits) == 11 and digits[0] == '1':
        return f'+{digits}'
    elif len(digits) > 11:
        return f'+{digits}'
    return None  # Invalid

df['phone_clean'] = df['phone'].apply(clean_phone_number)
```

### Email Validation and Cleaning

```python
def clean_email(email):
    """Validate and normalize email addresses."""
    if pd.isna(email):
        return None
    email = str(email).strip().lower()
    # Basic email pattern
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    if re.match(pattern, email):
        return email
    return None

# Extract domain for analysis
df['email_domain'] = df['email'].str.extract(r'@([a-zA-Z0-9.-]+\.[a-zA-Z]{2,})')
```

### Address Cleaning

```python
def standardize_address(address):
    """Normalize common address abbreviations."""
    if pd.isna(address):
        return None
    address = str(address).strip()
    # Standard abbreviation replacements
    replacements = {
        r'\bSt\.?\b': 'Street', r'\bAve\.?\b': 'Avenue',
        r'\bBlvd\.?\b': 'Boulevard', r'\bDr\.?\b': 'Drive',
        r'\bLn\.?\b': 'Lane', r'\bRd\.?\b': 'Road',
        r'\bApt\.?\b': 'Apartment', r'\bSte\.?\b': 'Suite',
        r'\b#\b': 'Unit'
    }
    for pattern, replacement in replacements.items():
        address = re.sub(pattern, replacement, address, flags=re.IGNORECASE)
    # Normalize whitespace
    address = re.sub(r'\s+', ' ', address)
    return address.title()
```

---

## 7. Handling Mixed Types in Columns

```python
# Diagnose mixed-type columns
def diagnose_column(series):
    """Detailed breakdown of values in a mixed-type column."""
    report = {
        'total_rows': len(series),
        'null_count': series.isna().sum(),
        'type_distribution': series.dropna().apply(type).value_counts().to_dict(),
        'sample_non_numeric': [],
    }
    # Try numeric conversion and report failures
    numeric = pd.to_numeric(series, errors='coerce')
    failed = series[numeric.isna() & series.notna()]
    report['non_numeric_count'] = len(failed)
    report['sample_non_numeric'] = failed.head(10).tolist()
    return report

# Strategy: Split mixed column into typed sub-columns
def split_mixed_column(df, col):
    """Separate numeric and text values from a mixed column."""
    numeric_mask = pd.to_numeric(df[col], errors='coerce').notna()
    df[f'{col}_numeric'] = pd.to_numeric(df[col], errors='coerce')
    df[f'{col}_text'] = df.loc[~numeric_mask & df[col].notna(), col]
    return df

# Handle columns with sentinel values
sentinel_values = ['N/A', 'n/a', 'NA', '-', '--', 'null', 'None', '?', '']
df = pd.read_csv('data.csv', na_values=sentinel_values)
```

> **Critical Insight:** Mixed types in a column often signal an upstream schema issue. Document the root cause and communicate it to the data engineering team rather than only patching it in your pipeline. At Amazon and Google, data quality is treated as a shared responsibility between producers and consumers.

---

## 8. Large Dataset Strategies

### Chunked Processing

```python
# Process large CSV in chunks
def process_large_csv(filepath, chunksize=100_000):
    """Process a large CSV file in memory-efficient chunks."""
    results = []
    for chunk in pd.read_csv(filepath, chunksize=chunksize, dtype_backend='pyarrow'):
        # Apply cleaning per chunk
        chunk = chunk.dropna(subset=['user_id'])
        chunk['amount'] = pd.to_numeric(chunk['amount'], errors='coerce')
        chunk = chunk[chunk['amount'] > 0]
        results.append(chunk)
    return pd.concat(results, ignore_index=True)
```

### Memory Optimization Techniques

```python
def reduce_memory_usage(df, verbose=True):
    """Reduce DataFrame memory usage by optimizing dtypes."""
    start_mem = df.memory_usage(deep=True).sum() / 1024**2
    
    for col in df.columns:
        col_type = df[col].dtype
        
        if col_type == 'object':
            # Convert low-cardinality strings to categorical
            if df[col].nunique() / len(df) < 0.5:
                df[col] = df[col].astype('category')
        elif col_type in ['int64', 'int32']:
            c_min, c_max = df[col].min(), df[col].max()
            if c_min >= 0:
                if c_max < 255:
                    df[col] = df[col].astype(np.uint8)
                elif c_max < 65535:
                    df[col] = df[col].astype(np.uint16)
                elif c_max < 4294967295:
                    df[col] = df[col].astype(np.uint32)
            else:
                if c_min > -128 and c_max < 127:
                    df[col] = df[col].astype(np.int8)
                elif c_min > -32768 and c_max < 32767:
                    df[col] = df[col].astype(np.int16)
                elif c_min > -2147483648 and c_max < 2147483647:
                    df[col] = df[col].astype(np.int32)
        elif col_type == 'float64':
            df[col] = pd.to_numeric(df[col], downcast='float')
    
    end_mem = df.memory_usage(deep=True).sum() / 1024**2
    if verbose:
        print(f"Memory: {start_mem:.1f} MB -> {end_mem:.1f} MB ({100*(start_mem-end_mem)/start_mem:.1f}% reduction)")
    return df
```

### Alternative Engines for Scale

| Tool | Best For | Memory Model | Speed vs Pandas |
|------|----------|--------------|-----------------|
| pandas | < 1 GB datasets | In-memory, eager | Baseline |
| Polars | 1-50 GB datasets | In-memory, lazy eval | 5-10x faster |
| Dask | > RAM datasets | Out-of-core, lazy | Parallel, distributed |
| PySpark | > 100 GB datasets | Distributed cluster | Horizontal scale |
| DuckDB | Analytical queries on files | Vectorized, out-of-core | 10-50x for SQL workloads |
| PyArrow | Columnar I/O, type system | Zero-copy | Fast I/O, interop layer |

> **Critical Insight:** At Netflix and Spotify, most data cleaning happens in SQL (dbt/Spark) before data reaches Python. Interviewers want to know you can clean data at scale — mention that your pandas code is often a prototype for a Spark/dbt transformation that runs on the full dataset.

---

## 9. Pipeline Patterns

### sklearn Pipeline with Custom Transformers

```python
from sklearn.base import BaseEstimator, TransformerMixin
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.impute import SimpleImputer
from sklearn.preprocessing import StandardScaler, OneHotEncoder

class OutlierClipper(BaseEstimator, TransformerMixin):
    """Custom transformer to clip outliers at IQR boundaries."""
    
    def __init__(self, multiplier=1.5):
        self.multiplier = multiplier
    
    def fit(self, X, y=None):
        Q1 = np.percentile(X, 25, axis=0)
        Q3 = np.percentile(X, 75, axis=0)
        IQR = Q3 - Q1
        self.lower_ = Q1 - self.multiplier * IQR
        self.upper_ = Q3 + self.multiplier * IQR
        return self
    
    def transform(self, X):
        return np.clip(X, self.lower_, self.upper_)


class DateFeatureExtractor(BaseEstimator, TransformerMixin):
    """Extract temporal features from datetime columns."""
    
    def __init__(self, date_col='date'):
        self.date_col = date_col
    
    def fit(self, X, y=None):
        return self
    
    def transform(self, X):
        X = X.copy()
        dt = pd.to_datetime(X[self.date_col])
        X['day_of_week'] = dt.dt.dayofweek
        X['month'] = dt.dt.month
        X['hour'] = dt.dt.hour
        X['is_weekend'] = (dt.dt.dayofweek >= 5).astype(int)
        X = X.drop(columns=[self.date_col])
        return X


# Full preprocessing pipeline
numeric_features = ['age', 'income', 'transaction_amount']
categorical_features = ['country', 'device_type', 'subscription_tier']

numeric_pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='median')),
    ('outlier_clip', OutlierClipper(multiplier=2.0)),
    ('scaler', StandardScaler())
])

categorical_pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='constant', fill_value='unknown')),
    ('encoder', OneHotEncoder(handle_unknown='ignore', sparse_output=False))
])

preprocessor = ColumnTransformer([
    ('numeric', numeric_pipeline, numeric_features),
    ('categorical', categorical_pipeline, categorical_features)
])

# Use in full ML pipeline
from sklearn.ensemble import GradientBoostingClassifier

full_pipeline = Pipeline([
    ('preprocessor', preprocessor),
    ('classifier', GradientBoostingClassifier())
])

full_pipeline.fit(X_train, y_train)
```

> **Critical Insight:** Sklearn Pipelines ensure that preprocessing is fitted only on training data and applied consistently to test/production data. This prevents data leakage — one of the most common silent errors in production ML systems.

---

## 10. Data Quality Checks

### The Four Dimensions of Data Quality

```python
class DataQualityChecker:
    """Comprehensive data quality assessment framework."""
    
    def __init__(self, df):
        self.df = df
        self.report = {}
    
    def check_completeness(self, critical_columns=None):
        """Measure null rates and flag columns below threshold."""
        null_rates = self.df.isnull().mean()
        self.report['completeness'] = {
            'null_rates': null_rates.to_dict(),
            'columns_above_50pct_null': null_rates[null_rates > 0.5].index.tolist()
        }
        if critical_columns:
            critical_nulls = null_rates[critical_columns]
            self.report['completeness']['critical_column_nulls'] = critical_nulls[critical_nulls > 0].to_dict()
        return self
    
    def check_validity(self, rules=None):
        """Validate values against business rules."""
        violations = {}
        if rules is None:
            rules = {}
        for col, rule in rules.items():
            if col not in self.df.columns:
                continue
            mask = ~self.df[col].apply(rule)
            if mask.any():
                violations[col] = {
                    'violation_count': int(mask.sum()),
                    'violation_rate': float(mask.mean()),
                    'sample_violations': self.df.loc[mask, col].head(5).tolist()
                }
        self.report['validity'] = violations
        return self
    
    def check_consistency(self, cross_field_rules=None):
        """Check cross-field logical consistency."""
        issues = {}
        if cross_field_rules:
            for name, rule_func in cross_field_rules.items():
                violations = ~self.df.apply(rule_func, axis=1)
                if violations.any():
                    issues[name] = {
                        'violation_count': int(violations.sum()),
                        'violation_rate': float(violations.mean())
                    }
        self.report['consistency'] = issues
        return self
    
    def check_uniqueness(self, key_columns):
        """Verify primary key uniqueness."""
        dup_count = self.df.duplicated(subset=key_columns).sum()
        self.report['uniqueness'] = {
            'key_columns': key_columns,
            'duplicate_count': int(dup_count),
            'duplicate_rate': float(dup_count / len(self.df)),
            'is_unique': dup_count == 0
        }
        return self
    
    def summarize(self):
        """Generate human-readable summary."""
        return self.report


# Usage
checker = DataQualityChecker(df)
checker.check_completeness(critical_columns=['user_id', 'event_type', 'timestamp'])
checker.check_validity(rules={
    'age': lambda x: 0 <= x <= 120 if pd.notna(x) else True,
    'email': lambda x: '@' in str(x) if pd.notna(x) else True,
    'revenue': lambda x: x >= 0 if pd.notna(x) else True
})
checker.check_consistency(cross_field_rules={
    'end_after_start': lambda row: row['end_date'] >= row['start_date'] if pd.notna(row['end_date']) else True,
    'discount_leq_price': lambda row: row['discount'] <= row['price'] if pd.notna(row['discount']) else True
})
checker.check_uniqueness(key_columns=['user_id', 'timestamp', 'event_type'])

report = checker.summarize()
```

### Automated Quality Gates

```python
def quality_gate(df, config):
    """Pass/fail gate for pipeline integration."""
    failures = []
    
    # Completeness gate
    for col, max_null_rate in config.get('max_null_rates', {}).items():
        actual = df[col].isnull().mean()
        if actual > max_null_rate:
            failures.append(f"FAIL: {col} null rate {actual:.2%} > {max_null_rate:.2%}")
    
    # Row count gate
    min_rows = config.get('min_row_count', 0)
    if len(df) < min_rows:
        failures.append(f"FAIL: Row count {len(df)} < minimum {min_rows}")
    
    # Freshness gate
    if 'timestamp_col' in config:
        max_age_hours = config.get('max_age_hours', 24)
        latest = pd.to_datetime(df[config['timestamp_col']]).max()
        age_hours = (pd.Timestamp.now() - latest).total_seconds() / 3600
        if age_hours > max_age_hours:
            failures.append(f"FAIL: Data is {age_hours:.1f} hours old > {max_age_hours}h threshold")
    
    if failures:
        raise ValueError("Data quality gate FAILED:\n" + "\n".join(failures))
    return True
```

> **Critical Insight:** At Meta and Google, data pipelines include automated quality gates that block downstream jobs when quality degrades. This is not optional — it prevents cascading failures where bad data corrupts models, dashboards, and business decisions.

---

## 11. Interview Talking Points and Scripts

### "Describe your typical data cleaning workflow"

> *"My data cleaning workflow follows a systematic approach. First, I do an initial profiling pass — shape, dtypes, null rates, distributions, and a random sample of rows. This gives me the lay of the land. Next, I check structural integrity: are primary keys unique? Are there unexpected duplicates? Do join keys match across tables?*
>
> *Then I address data types — coercing numerics, parsing dates, and identifying mixed-type columns. For categoricals, I normalize casing, strip whitespace, and map known variations to canonical forms. For outliers, I never remove them without domain context — I first ask whether they are errors or genuine extreme values.*
>
> *Finally, I build reproducible cleaning as a pipeline — either sklearn Pipeline for ML workflows or dbt transformations for analytics. I always include data quality assertions that will alert me if upstream data changes unexpectedly. The key principle is: document your assumptions, validate them explicitly, and make your cleaning auditable."*

### "How do you handle outliers?"

> *"My approach to outliers depends entirely on context. First, I separate data errors from genuine extreme values. A negative age is clearly wrong — I will fix or null it. A transaction of 50,000 dollars at an e-commerce company is unusual but might be a legitimate bulk purchase.*
>
> *For detection, I typically start with the IQR method for exploratory analysis because it makes no distributional assumptions. For modeling, if I need to detect multivariate outliers, I use Isolation Forest or Mahalanobis distance.*
>
> *For treatment, my decision tree is: (1) If it is a data error, fix at source or impute. (2) If it is a genuine value that would distort analysis, I winsorize rather than remove — this preserves the directionality while capping influence. (3) If the model is sensitive to outliers, I consider robust methods — median-based statistics, tree-based models, or log transformations. (4) In some cases, outliers are the signal — fraud detection is literally the task of finding outliers.*
>
> *I always document the outlier treatment decision and its business justification, and I report how many observations were affected."*

### "How do you ensure data quality in production?"

> *"I think about data quality in four dimensions: completeness, validity, consistency, and freshness. In production, I implement automated quality gates in the pipeline — these check null rates against thresholds, validate business rules, verify referential integrity, and ensure data is fresh.*
>
> *When a gate fails, it blocks downstream consumers and sends an alert. This is critical because silent data quality degradation can corrupt weeks of model predictions or dashboards before anyone notices. I also maintain data contracts with upstream producers — explicit schemas and SLAs on quality metrics. At scale, tools like Great Expectations or dbt tests codify these checks."*

---

## 12. Common Interview Mistakes

| Mistake | Better Approach |
|---------|-----------------|
| Dropping all rows with any null value | Analyze null patterns first; use imputation strategies appropriate to the mechanism (MCAR/MAR/MNAR) |
| Removing outliers without domain context | Ask whether the outlier is a data error or genuine signal; document the business justification for any removal |
| Using `infer_datetime_format` without validation | Explicitly specify format when known; validate parsed dates fall within expected range |
| Applying cleaning to full dataset before train/test split | Fit cleaning parameters (e.g., imputation median, scaler stats) only on training data |
| Hardcoding cleaning rules without tests | Write assertions that validate assumptions; break loudly when upstream data changes |
| Ignoring the "why" behind messy data | Trace data quality issues to their source; fix the root cause with the producing team |
| Cleaning in Jupyter notebooks with no reproducibility | Encapsulate cleaning in functions/pipelines that can be version-controlled and unit-tested |
| Using `apply` with row-wise functions on large data | Use vectorized operations (pandas string methods, numpy where) for 10-100x speedup |
| Treating all missing data the same | Distinguish between structurally missing (not applicable) and genuinely missing (applicable but unknown) |
| Reporting cleaned data counts without documenting what was removed | Always log the volume and nature of removed/modified records for audit trail |

---

## 13. Rapid-Fire Q&A

**Q1: What is the difference between `dropna()` and imputation? When do you choose each?**
> `dropna()` removes entire rows/columns; imputation fills missing values. Drop when missingness is minimal (<5%) and MCAR; impute when data is valuable, when missingness is informative (MAR/MNAR), or when you cannot afford to lose samples.

**Q2: How does `pd.to_numeric(errors='coerce')` differ from `astype(float)`?**
> `errors='coerce'` converts unparseable values to NaN silently; `astype(float)` raises a ValueError on non-numeric strings. Use coerce for exploratory cleaning, astype for strict validation.

**Q3: Why use Modified Z-Score instead of standard Z-Score?**
> Standard Z-Score uses mean and std, which are themselves distorted by outliers (masking effect). Modified Z-Score uses median and MAD — robust statistics that are not influenced by the extreme values you are trying to detect.

**Q4: What is data leakage in the context of preprocessing?**
> Fitting preprocessing steps (imputation, scaling, encoding) on the full dataset including test data. Information from test data "leaks" into training, inflating performance estimates. Solution: fit only on train, transform both.

**Q5: How do you handle a column where 40% of values are null?**
> First investigate why — is it structurally missing (e.g., "spouse_name" for unmarried people) or truly missing? If structural, encode as a separate category. If truly missing, consider: missingness indicator feature, model-based imputation (KNN, MICE), or whether the column is even usable given the rate.

**Q6: What is the advantage of `category` dtype in pandas?**
> Stores strings as integer codes with a lookup table. Reduces memory by 90%+ for low-cardinality columns, speeds up groupby/merge operations, and enables ordered categoricals for ordinal data.

**Q7: When would you use chunked processing vs. switching to Spark/Polars?**
> Chunked processing works when each chunk can be processed independently (row-wise cleaning, filtering). Switch to Spark/Polars when you need global operations (sorting, groupby over all data, joins across large tables) that require seeing the full dataset.

**Q8: How do you validate that your cleaning pipeline produces correct results?**
> Write unit tests with known inputs and expected outputs. Add data quality assertions at each pipeline stage. Compare distributions before/after cleaning. Monitor key statistics (row counts, null rates, value distributions) in production with alerting.

**Q9: What is a data contract and why does it matter for cleaning?**
> A data contract is an explicit agreement between data producer and consumer on schema, semantics, quality thresholds, and freshness SLAs. It shifts quality responsibility upstream and lets consumers build reliable pipelines without defensive over-cleaning.

**Q10: How do you handle timezone-naive timestamps from multiple regions?**
> Never assume a timezone. Trace the data lineage to determine what timezone the source system used. If mixed, require a timezone field alongside the timestamp, localize each correctly, then convert all to UTC for storage. Document the timezone semantics explicitly.

---

## 14. Summary Cheat Sheet

```
+-------------------------------------------------------------------+
|            DATA CLEANING & PREPROCESSING CHEAT SHEET              |
+-------------------------------------------------------------------+
|                                                                   |
|  WORKFLOW ORDER:                                                  |
|  1. Profile (shape, dtypes, nulls, distributions)                |
|  2. Structural checks (keys, duplicates, joins)                  |
|  3. Type coercion (numerics, dates, categoricals)                |
|  4. Deduplication (exact + fuzzy)                                |
|  5. Outlier analysis (detect -> classify -> treat)               |
|  6. Normalization (case, whitespace, encodings)                  |
|  7. Pipeline encapsulation (sklearn / dbt / Spark)               |
|  8. Quality gates (assertions + monitoring)                      |
|                                                                   |
+-------------------------------------------------------------------+
|                                                                   |
|  OUTLIER DECISION TREE:                                          |
|  Is it a data error? --> Fix/null it                             |
|  Is it genuine but distorting? --> Winsorize/transform           |
|  Is it genuine and informative? --> Keep it                      |
|  Is the outlier the signal? --> Model it (anomaly detection)     |
|                                                                   |
+-------------------------------------------------------------------+
|                                                                   |
|  MISSING DATA STRATEGY:                                          |
|  MCAR (random) --> Drop if <5%, mean/median impute              |
|  MAR (depends on observed) --> Model-based impute (MICE, KNN)   |
|  MNAR (depends on unobserved) --> Domain knowledge, Heckman     |
|                                                                   |
+-------------------------------------------------------------------+
|                                                                   |
|  SCALE DECISION:                                                 |
|  < 1 GB        --> pandas (optimize dtypes)                      |
|  1-50 GB       --> Polars or DuckDB                              |
|  > 50 GB       --> Spark / Dask / BigQuery                       |
|                                                                   |
+-------------------------------------------------------------------+
|                                                                   |
|  KEY PRINCIPLES:                                                 |
|  - Never clean before train/test split                           |
|  - Document every removal/transformation                         |
|  - Fix root causes upstream, not just symptoms                   |
|  - Make cleaning reproducible and testable                       |
|  - Outliers are not always bad -- context is everything          |
|  - Silent failures are worse than loud errors                    |
|                                                                   |
+-------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 23 of 25*
