# 🎯 Topic 7: File Handling & I/O Operations

> **Scope:** Reading and writing files in Python, serialization formats (CSV, JSON, Pickle, Parquet, JSONL), pathlib for modern path manipulation, memory-efficient techniques for large files (chunking, generators), and choosing the right format for data science workflows. Targeted at DS/DA roles with 5 YOE interviewing at big tech.

---

## Table of Contents

1. [Core File I/O Concepts](#core-file-io-concepts)
2. [The pathlib Module](#the-pathlib-module)
3. [Reading & Writing CSV Files](#reading--writing-csv-files)
4. [JSON & JSONL (JSON Lines)](#json--jsonl-json-lines)
5. [Pickle Serialization](#pickle-serialization)
6. [Parquet & Columnar Storage](#parquet--columnar-storage)
7. [File Format Comparison Table](#file-format-comparison-table)
8. [Working with Large Files](#working-with-large-files)
9. [Context Managers & Resource Safety](#context-managers--resource-safety)
10. [Interview Talking Points & Scripts](#interview-talking-points--scripts)
11. [Common Interview Mistakes](#common-interview-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## Core File I/O Concepts

Python's built-in `open()` function is the foundation of all file operations. Understanding modes, encoding, and buffering is essential.

### File Modes

| Mode | Description | Creates File? | Truncates? |
|------|-------------|---------------|------------|
| `'r'` | Read (text, default) | No | No |
| `'w'` | Write (text) | Yes | Yes |
| `'a'` | Append (text) | Yes | No |
| `'x'` | Exclusive create | Yes (fails if exists) | N/A |
| `'rb'` | Read binary | No | No |
| `'wb'` | Write binary | Yes | Yes |

> **Critical Insight:** Always specify `encoding='utf-8'` explicitly when working with text files. The default encoding is platform-dependent (`cp1252` on Windows vs `utf-8` on macOS/Linux), which causes silent data corruption in cross-platform data pipelines.

```python
# Best practice: explicit encoding
with open('data.csv', 'r', encoding='utf-8') as f:
    content = f.read()

# Reading line by line (memory efficient for large files)
with open('large_log.txt', 'r', encoding='utf-8') as f:
    for line in f:  # Iterator — only one line in memory at a time
        process(line.strip())
```

---

## The pathlib Module

`pathlib` (Python 3.4+) replaces `os.path` with an object-oriented API for filesystem paths.

```python
from pathlib import Path

# Constructing paths (OS-agnostic)
data_dir = Path('/data/raw')
file_path = data_dir / 'sales_2024.csv'  # Overloaded / operator

# Common operations
file_path.exists()          # True/False
file_path.is_file()         # True if file (not directory)
file_path.suffix            # '.csv'
file_path.stem              # 'sales_2024'
file_path.parent            # PosixPath('/data/raw')
file_path.stat().st_size    # File size in bytes

# Globbing for batch processing
csv_files = list(data_dir.glob('**/*.csv'))  # Recursive glob

# Reading/writing directly
content = file_path.read_text(encoding='utf-8')
file_path.write_text('hello', encoding='utf-8')

# Creating directories safely
output_dir = Path('results/figures')
output_dir.mkdir(parents=True, exist_ok=True)
```

> **Critical Insight:** In interviews, using `pathlib` signals modern Python fluency. It eliminates entire categories of bugs (path separator issues, missing directory errors) and is the recommended approach in PEP 428.

---

## Reading & Writing CSV Files

### Standard Library (`csv` module)

```python
import csv

# Reading with DictReader (column-name access)
with open('data.csv', 'r', encoding='utf-8') as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row['revenue'], row['region'])

# Writing with DictWriter
fieldnames = ['date', 'revenue', 'region']
with open('output.csv', 'w', encoding='utf-8', newline='') as f:
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerow({'date': '2024-01-01', 'revenue': 50000, 'region': 'US'})
```

### Pandas (Production Data Science)

```python
import pandas as pd

# Reading with type optimization
df = pd.read_csv(
    'sales.csv',
    dtype={'region': 'category', 'store_id': 'int32'},
    parse_dates=['order_date'],
    usecols=['order_date', 'revenue', 'region', 'store_id'],  # Only needed columns
    na_values=['', 'NULL', 'N/A']
)

# Writing with compression
df.to_csv('output.csv.gz', index=False, compression='gzip')
```

> **Critical Insight:** Specifying `dtype` and `usecols` in `pd.read_csv()` can reduce memory usage by 50-80% on real datasets. This is a common follow-up question: "How would you load a 10GB CSV on a 16GB RAM machine?"

---

## JSON & JSONL (JSON Lines)

### Standard JSON

```python
import json

# Reading JSON
with open('config.json', 'r', encoding='utf-8') as f:
    data = json.load(f)  # Parse entire file into dict/list

# Writing JSON (human-readable)
with open('results.json', 'w', encoding='utf-8') as f:
    json.dump(data, f, indent=2, ensure_ascii=False, default=str)

# Handling non-serializable types (datetime, numpy)
import datetime
import numpy as np

class CustomEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime.datetime):
            return obj.isoformat()
        if isinstance(obj, np.integer):
            return int(obj)
        if isinstance(obj, np.floating):
            return float(obj)
        if isinstance(obj, np.ndarray):
            return obj.tolist()
        return super().default(obj)

json.dumps(data, cls=CustomEncoder)
```

### JSONL (JSON Lines) — Streaming Format

```python
# Writing JSONL — one JSON object per line
def write_jsonl(records, filepath):
    with open(filepath, 'w', encoding='utf-8') as f:
        for record in records:
            f.write(json.dumps(record, ensure_ascii=False) + '\n')

# Reading JSONL — memory efficient (streaming)
def read_jsonl(filepath):
    with open(filepath, 'r', encoding='utf-8') as f:
        for line in f:
            if line.strip():  # Skip empty lines
                yield json.loads(line)

# Processing large JSONL files lazily
for record in read_jsonl('events.jsonl'):
    if record['event_type'] == 'purchase':
        process_purchase(record)
```

> **Critical Insight:** JSONL is the de facto standard for ML training data, logging pipelines, and streaming data (e.g., Kinesis, Kafka exports). Unlike JSON arrays, JSONL allows append-without-rewrite and line-by-line processing without loading the full file.

---

## Pickle Serialization

```python
import pickle

# Saving Python objects (models, preprocessors)
model = train_model(X_train, y_train)
with open('model.pkl', 'wb') as f:
    pickle.dump(model, f, protocol=pickle.HIGHEST_PROTOCOL)

# Loading
with open('model.pkl', 'rb') as f:
    model = pickle.load(f)

# Multiple objects in one file
with open('pipeline.pkl', 'wb') as f:
    pickle.dump({'model': model, 'scaler': scaler, 'features': feature_list}, f)
```

> **Critical Insight:** Pickle is NOT safe for untrusted data — it can execute arbitrary code on load. In interviews, always mention this security concern and note that pickle files are Python-version-dependent and not cross-language. For ML model deployment, prefer ONNX or joblib.

---

## Parquet & Columnar Storage

```python
import pandas as pd
import pyarrow.parquet as pq

# Writing Parquet (with compression)
df.to_parquet('sales.parquet', engine='pyarrow', compression='snappy', index=False)

# Reading Parquet (column pruning — only reads needed columns from disk)
df = pd.read_parquet(
    'sales.parquet',
    columns=['revenue', 'region', 'order_date'],
    filters=[('region', '==', 'US')]  # Predicate pushdown
)

# Reading partitioned datasets
df = pd.read_parquet('data/sales/', filters=[('year', '>=', 2023)])

# PyArrow for schema inspection without loading data
schema = pq.read_schema('sales.parquet')
print(schema)  # Column names, types, metadata
metadata = pq.read_metadata('sales.parquet')
print(f"Rows: {metadata.num_rows}, Row groups: {metadata.num_row_groups}")
```

> **Critical Insight:** Parquet achieves 60-90% compression vs CSV while being 10-100x faster to read for analytical queries due to columnar storage and predicate pushdown. It is the standard format for data lakes (S3, HDFS) and tools like Spark, Athena, and BigQuery natively read it.

---

## File Format Comparison Table

| Feature | CSV | JSON | JSONL | Pickle | Parquet |
|---------|-----|------|-------|--------|---------|
| **Human Readable** | Yes | Yes | Yes | No | No |
| **Schema Enforcement** | No | No | No | No | Yes |
| **Type Preservation** | No (all strings) | Partial | Partial | Full (Python) | Full |
| **Nested Data** | No | Yes | Yes | Yes | Yes (struct) |
| **Compression** | External | External | External | No | Built-in |
| **Columnar Access** | No (row-based) | No | No | No | Yes |
| **Streaming/Append** | Yes | No (must rewrite) | Yes | No | No (immutable) |
| **Cross-Language** | Universal | Universal | Universal | Python only | Universal |
| **Security** | Safe | Safe | Safe | UNSAFE | Safe |
| **Typical Use Case** | Data exchange, Excel | APIs, configs | ML data, logs | Model artifacts | Data lakes, analytics |
| **Read Speed (large)** | Slow | Slow | Medium | Fast | Very Fast |
| **File Size (10M rows)** | ~1 GB | ~1.5 GB | ~1.2 GB | ~0.8 GB | ~0.2 GB |

### When to Use Which Format

| Scenario | Best Format | Reason |
|----------|-------------|--------|
| Sharing with non-technical stakeholders | CSV | Universal Excel/Sheets compatibility |
| REST API response | JSON | Standard web interchange format |
| ML training data pipeline | JSONL | Streamable, appendable, line-countable |
| Saving sklearn model to disk | Pickle/Joblib | Preserves Python object graph |
| Analytics data lake on S3 | Parquet | Compression, columnar queries, schema |
| Logging/event streaming | JSONL | Append-only, structured, parseable |
| Intermediate pipeline cache | Parquet | Fast I/O, typed, compact |
| Config files | JSON/YAML | Human readable, editable |

---

## Working with Large Files

### Chunked Reading with Pandas

```python
import pandas as pd

# Process a 50GB CSV in manageable chunks
chunk_size = 100_000
results = []

for chunk in pd.read_csv('huge_file.csv', chunksize=chunk_size, dtype={'id': 'int32'}):
    # Filter and aggregate per chunk
    filtered = chunk[chunk['status'] == 'completed']
    agg = filtered.groupby('region')['revenue'].sum()
    results.append(agg)

# Combine chunk results
final = pd.concat(results).groupby(level=0).sum()
```

### Generator-Based Processing

```python
from pathlib import Path
import csv

def stream_csv(filepath, batch_size=10000):
    """Yield batches of rows as lists of dicts."""
    batch = []
    with open(filepath, 'r', encoding='utf-8') as f:
        reader = csv.DictReader(f)
        for row in reader:
            batch.append(row)
            if len(batch) >= batch_size:
                yield batch
                batch = []
        if batch:  # Don't forget the last partial batch
            yield batch

# Memory-constant processing regardless of file size
total_revenue = 0
for batch in stream_csv('transactions.csv'):
    total_revenue += sum(float(row['amount']) for row in batch)
```

### Memory-Mapped Files for Random Access

```python
import mmap
import os

# Memory-mapped file for random access to large files
with open('large_binary.dat', 'rb') as f:
    mm = mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ)
    # Read bytes 1000-2000 without loading entire file
    chunk = mm[1000:2000]
    mm.close()
```

> **Critical Insight:** The three strategies for large files are: (1) chunked reading for aggregation tasks, (2) generators for streaming ETL pipelines, and (3) memory-mapped files for random access patterns. In interviews, articulate which pattern fits which use case rather than defaulting to "just use Spark."

---

## Context Managers & Resource Safety

```python
from contextlib import contextmanager
import tempfile
from pathlib import Path

# Custom context manager for atomic writes (prevents corruption)
@contextmanager
def atomic_write(filepath, mode='w', encoding='utf-8'):
    """Write to temp file, then rename — prevents partial writes on crash."""
    filepath = Path(filepath)
    tmp_path = filepath.with_suffix('.tmp')
    try:
        with open(tmp_path, mode, encoding=encoding) as f:
            yield f
        tmp_path.replace(filepath)  # Atomic on POSIX
    except:
        tmp_path.unlink(missing_ok=True)
        raise

# Usage
with atomic_write('results.json') as f:
    json.dump(large_results, f)

# Multiple file handling
from contextlib import ExitStack

def merge_sorted_files(input_paths, output_path):
    with ExitStack() as stack:
        files = [stack.enter_context(open(p, 'r')) for p in input_paths]
        out = stack.enter_context(open(output_path, 'w'))
        # All files guaranteed closed even if exception occurs
        for line in merge_iterators(files):
            out.write(line)
```

---

## Interview Talking Points & Scripts

> *"When loading data for analysis, I always consider the file format as a design decision rather than an afterthought. For exploratory work on moderate datasets, I use CSV with explicit dtypes and usecols in pandas to control memory. For production pipelines where the same data is queried repeatedly, I convert to Parquet immediately — the columnar format gives me predicate pushdown and 80%+ compression, which means faster reads and lower storage costs on S3."*

> *"For large files that exceed available RAM, I reach for chunked processing first — pandas read_csv with chunksize lets me aggregate across chunks without loading everything. If the pipeline needs to be truly streaming and framework-agnostic, I write generator functions that yield batches. This pattern composes well with multiprocessing.Pool for CPU-bound transforms."*

> *"I'm deliberate about format choices in data pipelines. JSONL for event logs and ML training data because it supports append-without-rewrite and line-level parallelism. Parquet for analytical tables because of schema enforcement and columnar efficiency. I avoid pickle for anything shared between services due to security and versioning risks — ONNX or standard serialization formats are safer."*

> *"One pattern I've found invaluable is atomic writes — writing to a temporary file and renaming on success. This prevents data corruption from partial writes during pipeline failures, which matters when your job writes outputs that downstream systems consume."*

---

## Common Interview Mistakes

| Mistake | Why It's Wrong | Correct Approach |
|---------|---------------|------------------|
| Reading entire large file with `f.read()` | Loads full file into memory; crashes on large inputs | Use iteration (`for line in f`) or chunked reading |
| Not specifying encoding | Platform-dependent behavior; silent corruption | Always use `encoding='utf-8'` explicitly |
| Using `os.path.join()` in new code | Works but outdated; less readable | Use `pathlib.Path` with `/` operator |
| Storing DataFrames as pickle for sharing | Version-dependent, insecure, not cross-language | Use Parquet for tabular data interchange |
| Not closing files (no `with` statement) | Resource leaks, especially in long-running processes | Always use context managers (`with open(...)`) |
| Loading full JSON array for streaming data | Cannot append without rewriting; memory-heavy | Use JSONL for streaming/append patterns |
| Ignoring `newline=''` in CSV writing | Double newlines on Windows | Always pass `newline=''` to `open()` for CSV |
| Saying "I'd use Spark" for any large file | Overkill for single-machine problems; shows lack of range | First try chunked pandas/generators; Spark for truly distributed needs |
| Not handling the last partial batch in generators | Silently drops trailing records | Always yield remainder after loop |
| Using default pickle protocol | Older protocols are slower and produce larger files | Use `protocol=pickle.HIGHEST_PROTOCOL` |

---

## Rapid-Fire Q&A

**Q1: What's the difference between `json.load()` and `json.loads()`?**
`json.load()` reads from a file object; `json.loads()` parses from a string. Same for `dump()` vs `dumps()`.

**Q2: How does Parquet achieve faster reads than CSV for analytical queries?**
Columnar storage means only requested columns are read from disk. Row-group metadata enables predicate pushdown (skipping irrelevant data blocks entirely).

**Q3: Why is JSONL preferred over JSON arrays for ML training data?**
JSONL supports streaming (one record per line), append-without-rewrite, line-level parallelism (split by line count), and easy `wc -l` record counting.

**Q4: How would you process a 50GB CSV on a machine with 16GB RAM?**
Use `pd.read_csv(chunksize=100000)`, process each chunk (filter/aggregate), accumulate partial results, and combine at the end. Alternatively, use generators or convert to Parquet first.

**Q5: What's the security risk with pickle?**
Pickle can execute arbitrary code during deserialization (`__reduce__` method). Never unpickle data from untrusted sources. Prefer JSON, Parquet, or ONNX for cross-service data.

**Q6: When would you use `pathlib.Path.glob()` vs `os.listdir()`?**
`glob()` supports pattern matching (wildcards, recursive `**`), returns Path objects, and is more expressive. `os.listdir()` only lists immediate contents without filtering.

**Q7: What does `newline=''` do when opening a CSV file for writing?**
It prevents the csv module from adding extra `\r` characters on Windows, which would result in blank lines between rows.

**Q8: How does chunked reading differ from memory-mapped files?**
Chunked reading processes sequential batches (good for full-file scans). Memory-mapped files provide random byte-offset access without loading the full file (good for indexed lookups).

**Q9: What's the advantage of Parquet's row groups?**
Row groups allow parallel processing and predicate pushdown. Statistics (min/max) stored per row group let readers skip entire groups that don't match filters.

**Q10: How do you handle a CSV where some columns have mixed types?**
Specify `dtype` explicitly in `pd.read_csv()`, use `na_values` for missing markers, and set `low_memory=False` to let pandas infer types from the full column rather than per-chunk.

**Q11: What is an atomic write and why does it matter in data pipelines?**
Writing to a temp file then renaming ensures readers never see partial/corrupted output. If the process crashes mid-write, the original file remains intact.

**Q12: When should you prefer `joblib.dump()` over `pickle.dump()` for ML models?**
Joblib is optimized for objects containing large NumPy arrays (common in sklearn models) — it uses memory-mapping and compression for faster serialization of array-heavy objects.

---

## Summary Cheat Sheet

```
+------------------------------------------------------------------+
|              FILE HANDLING & I/O — QUICK REFERENCE                |
+------------------------------------------------------------------+
| FORMAT SELECTION:                                                 |
|   CSV     → Universal exchange, Excel compat, no types           |
|   JSON    → APIs, configs, nested structures                     |
|   JSONL   → Streaming, ML data, logs, append-friendly            |
|   Pickle  → Python-only model artifacts (INSECURE)               |
|   Parquet → Analytics, data lakes, columnar, compressed          |
+------------------------------------------------------------------+
| LARGE FILE STRATEGIES:                                           |
|   Chunked  → pd.read_csv(chunksize=N) for aggregation           |
|   Generator→ yield batches for streaming ETL                     |
|   Mmap     → Random access without full load                    |
|   Parquet  → Column pruning + predicate pushdown                 |
+------------------------------------------------------------------+
| BEST PRACTICES:                                                   |
|   ✓ Always use context managers (with statement)                 |
|   ✓ Explicit encoding='utf-8'                                    |
|   ✓ pathlib over os.path                                         |
|   ✓ Specify dtype/usecols in pd.read_csv()                       |
|   ✓ Atomic writes for pipeline outputs                           |
|   ✓ HIGHEST_PROTOCOL for pickle                                  |
|   ✗ Never pickle untrusted data                                  |
|   ✗ Never f.read() on unknown-size files                         |
+------------------------------------------------------------------+
| MEMORY ESTIMATION:                                               |
|   CSV in pandas ≈ 5-10x file size (string overhead)              |
|   Parquet in pandas ≈ 2-3x file size                             |
|   dtype optimization → 50-80% reduction                          |
|   category dtype → 90%+ reduction for low-cardinality columns    |
+------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 7 of 25*
