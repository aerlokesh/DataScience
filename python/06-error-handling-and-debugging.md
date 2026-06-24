# 🎯 Topic 6: Error Handling, Debugging & Defensive Coding

> Master Python's error handling machinery — try/except/else/finally, custom exceptions, context managers, logging, and assertions — with a focus on building resilient data pipelines that fail gracefully and loudly.

---

## Table of Contents

1. [Exception Hierarchy & Fundamentals](#1-exception-hierarchy--fundamentals)
2. [try/except/else/finally Deep Dive](#2-tryexceptelsefinally-deep-dive)
3. [Custom Exceptions for Data Pipelines](#3-custom-exceptions-for-data-pipelines)
4. [Context Managers (with Statement)](#4-context-managers-with-statement)
5. [Logging Best Practices](#5-logging-best-practices)
6. [Assertions vs. Raise](#6-assertions-vs-raise)
7. [Defensive Coding in Data Pipelines](#7-defensive-coding-in-data-pipelines)
8. [Interview Talking Points & Scripts](#8-interview-talking-points--scripts)
9. [Common Interview Mistakes](#9-common-interview-mistakes)
10. [Rapid-Fire Q&A](#10-rapid-fire-qa)
11. [Summary Cheat Sheet](#11-summary-cheat-sheet)

---

## 1. Exception Hierarchy & Fundamentals

| Level | Class | Use Case |
|-------|-------|----------|
| Root | `BaseException` | System-exit events (`KeyboardInterrupt`, `SystemExit`) |
| Common Base | `Exception` | All user/application errors inherit from here |
| Arithmetic | `ArithmeticError` → `ZeroDivisionError` | Division, overflow |
| Lookup | `LookupError` → `KeyError`, `IndexError` | Dict/list access |
| Type/Value | `TypeError`, `ValueError` | Wrong type or invalid value |
| I/O | `OSError` → `FileNotFoundError` | File/network operations |
| Custom | Your classes inheriting `Exception` | Domain-specific failures |

> **Critical Insight:** Never catch `BaseException` unless you intend to handle `KeyboardInterrupt` and `SystemExit`. Catching `Exception` is the correct ceiling for application-level error handling.

---

## 2. try/except/else/finally Deep Dive

```python
def load_dataset(path: str) -> pd.DataFrame:
    """Demonstrates when each block executes."""
    try:
        # Runs first — wrap ONLY the risky operation
        df = pd.read_csv(path)
    except FileNotFoundError:
        # Runs ONLY if the specific exception fires
        logger.error(f"Dataset not found: {path}")
        raise
    except pd.errors.ParserError as e:
        # Catch more specific exceptions BEFORE general ones
        logger.error(f"Malformed CSV: {e}")
        raise DataQualityError(f"Cannot parse {path}") from e
    else:
        # Runs ONLY if try succeeded — keep success logic here
        # to avoid accidentally catching exceptions from this code
        logger.info(f"Loaded {len(df)} rows from {path}")
        return df
    finally:
        # ALWAYS runs — cleanup, regardless of success or failure
        logger.debug(f"Finished load attempt for {path}")
```

| Block | Runs When | Typical Use |
|-------|-----------|-------------|
| `try` | Always (first) | Wrap the minimal risky operation |
| `except` | Only if matching exception raised in `try` | Log, transform, re-raise |
| `else` | Only if `try` completed without exception | Success-path logic |
| `finally` | Always, even after `return` or unhandled exception | Resource cleanup |

> **Critical Insight:** The `else` block exists to separate "what might fail" from "what to do on success." Code in `else` is NOT protected by the `except` — bugs there propagate naturally, which is what you want.

---

## 3. Custom Exceptions for Data Pipelines

```python
class DataPipelineError(Exception):
    """Base exception for all pipeline failures."""
    pass

class DataQualityError(DataPipelineError):
    """Raised when data fails validation checks."""
    def __init__(self, message: str, column: str = None, row_count: int = None):
        self.column = column
        self.row_count = row_count
        super().__init__(message)

class SchemaValidationError(DataPipelineError):
    """Raised when incoming data doesn't match expected schema."""
    def __init__(self, expected: list, actual: list):
        self.missing = set(expected) - set(actual)
        self.extra = set(actual) - set(expected)
        msg = f"Schema mismatch — missing: {self.missing}, unexpected: {self.extra}"
        super().__init__(msg)

class RetryableError(DataPipelineError):
    """Raised for transient failures that warrant a retry."""
    pass

# Usage in validation
def validate_schema(df: pd.DataFrame, expected_cols: list) -> None:
    actual_cols = df.columns.tolist()
    if set(expected_cols) != set(actual_cols):
        raise SchemaValidationError(expected=expected_cols, actual=actual_cols)
```

> **Critical Insight:** Custom exceptions carry structured context (column name, row counts, missing fields) that generic exceptions cannot. Interviewers love seeing domain-aware error design — it signals production maturity.

---

## 4. Context Managers (with Statement)

### Database Connection

```python
from contextlib import contextmanager

@contextmanager
def db_connection(connection_string: str):
    """Guarantees connection cleanup even on failure."""
    conn = psycopg2.connect(connection_string)
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()

# Usage
with db_connection(DSN) as conn:
    df = pd.read_sql("SELECT * FROM events", conn)
```

### Timer for Pipeline Stages

```python
import time
from contextlib import contextmanager

@contextmanager
def pipeline_timer(stage_name: str):
    """Logs wall-clock time for any pipeline stage."""
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        logger.info(f"[{stage_name}] completed in {elapsed:.2f}s")

# Usage
with pipeline_timer("feature_engineering"):
    df = compute_features(raw_df)
```

### Temporary File Handling

```python
import tempfile
from contextlib import contextmanager

@contextmanager
def temp_parquet(df: pd.DataFrame):
    """Write df to a temp parquet file, auto-cleanup on exit."""
    tmp = tempfile.NamedTemporaryFile(suffix=".parquet", delete=False)
    try:
        df.to_parquet(tmp.name)
        yield tmp.name
    finally:
        os.unlink(tmp.name)
```

> **Critical Insight:** Context managers encode the "acquire → use → release" pattern. The `__enter__`/`__exit__` protocol (or `@contextmanager`) guarantees cleanup runs even if the body raises — essential for DB connections, locks, and temp files in production pipelines.

---

## 5. Logging Best Practices

```python
import logging

# Proper setup — do this ONCE at module/application level
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(name)s | %(levelname)s | %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
    handlers=[
        logging.StreamHandler(),                          # Console
        logging.FileHandler("pipeline.log", mode="a"),    # Persistent log
    ]
)
logger = logging.getLogger(__name__)
```

| Level | When to Use | Example |
|-------|-------------|---------|
| `DEBUG` | Detailed diagnostic info during development | `logger.debug(f"Row {i}: feature vector = {vec}")` |
| `INFO` | Confirmation that things work as expected | `logger.info(f"Loaded {n} rows from {src}")` |
| `WARNING` | Something unexpected but not breaking | `logger.warning(f"Column 'age' has {pct}% nulls")` |
| `ERROR` | A function failed but app continues | `logger.error(f"Failed to fetch partition {p}", exc_info=True)` |
| `CRITICAL` | App-level failure, likely shutting down | `logger.critical("Database unreachable — aborting pipeline")` |

```python
# Anti-pattern vs. correct pattern
# BAD: print statements (not captured, no levels, no timestamps)
print(f"Error: {e}")

# GOOD: structured logging with exception info
logger.error("Transform failed for batch %s", batch_id, exc_info=True)
```

> **Critical Insight:** Always pass `exc_info=True` (or use `logger.exception()`) when logging inside an `except` block. This captures the full traceback without swallowing it — invaluable for post-mortem debugging in production.

---

## 6. Assertions vs. Raise

| Aspect | `assert` | `raise` |
|--------|----------|---------|
| Purpose | Developer sanity check / invariant | Runtime validation & control flow |
| Disabled by | `python -O` (optimized mode) | Never — always executes |
| Production use | **No** — stripped in optimized builds | **Yes** — the correct choice |
| Appropriate for | Tests, debug-mode invariants | Input validation, business rules |
| Performance | Zero cost when optimized away | Minimal (exception creation) |

```python
# WRONG — assert for runtime validation (gets stripped with -O flag)
assert len(df) > 0, "Empty dataframe"

# CORRECT — explicit raise for production validation
if len(df) == 0:
    raise DataQualityError("Received empty dataframe", row_count=0)
```

---

## 7. Defensive Coding in Data Pipelines

```python
import time
from functools import wraps

def retry(max_attempts: int = 3, backoff: float = 2.0):
    """Retry decorator with exponential backoff for transient failures."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except RetryableError as e:
                    if attempt == max_attempts:
                        raise
                    wait = backoff ** attempt
                    logger.warning(f"Attempt {attempt} failed: {e}. Retrying in {wait}s")
                    time.sleep(wait)
        return wrapper
    return decorator

@retry(max_attempts=3)
def fetch_from_api(endpoint: str) -> dict:
    response = requests.get(endpoint, timeout=30)
    if response.status_code == 503:
        raise RetryableError(f"Service unavailable: {endpoint}")
    response.raise_for_status()
    return response.json()

def defensive_transform(df: pd.DataFrame) -> pd.DataFrame:
    """Shows null checks, schema validation, and safe defaults."""
    # Schema validation
    required = ["user_id", "event_ts", "amount"]
    validate_schema(df, required)

    # Null checks with clear logging
    null_pct = df["amount"].isna().mean()
    if null_pct > 0.5:
        raise DataQualityError(
            f"Column 'amount' is {null_pct:.0%} null — exceeds 50% threshold",
            column="amount"
        )

    # Safe type coercion with error tracking
    df["amount"] = pd.to_numeric(df["amount"], errors="coerce")
    coerced_nulls = df["amount"].isna().sum()
    if coerced_nulls > 0:
        logger.warning(f"Coerced {coerced_nulls} non-numeric values to NaN in 'amount'")

    return df
```

---

## 8. Interview Talking Points & Scripts

> *"In production data pipelines, I follow the principle of 'fail fast, fail loud.' I use custom exception hierarchies — like DataQualityError and SchemaValidationError — so that upstream monitoring can distinguish a bad data source from a code bug. Bare excepts are never acceptable because they mask the real failure and make debugging in production nearly impossible."*

> *"I always wrap resource acquisition in context managers — database connections, temp files, API sessions. The with-statement guarantees cleanup runs even during exceptions, preventing connection leaks that silently degrade system health over time."*

> *"For retry logic, I separate transient failures (network timeouts, 503s) from permanent ones (schema mismatches, auth failures). Retrying a permanent error wastes time and masks the root cause. I use exponential backoff to avoid thundering-herd effects on recovering services."*

> *"I never use assertions for runtime validation because Python's -O flag strips them entirely. Assertions are for developer invariants during testing — production validation must use explicit raise statements that cannot be optimized away."*

---

## 9. Common Interview Mistakes

| Mistake | Why It's Wrong | Correct Approach |
|---------|---------------|-----------------|
| Bare `except:` | Catches `SystemExit`, `KeyboardInterrupt`, hides bugs | Catch specific exceptions: `except ValueError as e:` |
| Swallowing errors silently | `except: pass` — failure is invisible | Log the error + re-raise or return a sentinel with logging |
| Using `assert` in production | Stripped by `python -O`, validation disappears | Use `if ... : raise` for runtime checks |
| Catching too broadly | `except Exception:` around 50 lines of code | Wrap only the minimal risky operation |
| Not chaining exceptions | `raise NewError(msg)` — loses original traceback | `raise NewError(msg) from original_exception` |
| print() instead of logging | No levels, no timestamps, not captured in prod | Use `logging` module with proper handlers |
| Not re-raising after logging | Error is logged but caller thinks success occurred | `logger.error(...); raise` |
| `finally` for return values | Returning from `finally` silently swallows exceptions | Use `finally` only for cleanup, never for returns |

---

## 10. Rapid-Fire Q&A

**Q1: What's the difference between `except Exception` and bare `except:`?**
> Bare `except:` catches everything including `SystemExit` and `KeyboardInterrupt`. `except Exception:` only catches application-level errors, letting system signals propagate normally.

**Q2: When does `else` run in a try/except/else block?**
> Only when `try` completes without raising any exception. It separates success-path logic from error-prone code.

**Q3: Can `finally` swallow an exception?**
> Yes — if `finally` contains a `return` statement, it silently suppresses any active exception. Never return from `finally`.

**Q4: What does `raise ... from e` do?**
> It sets `__cause__` on the new exception, creating an explicit chain. The traceback shows "The above exception was the direct cause of..." rather than "During handling... another exception occurred."

**Q5: How do you make a class work with `with` statements?**
> Implement `__enter__` (returns the resource) and `__exit__` (handles cleanup). `__exit__` receives exc_type, exc_val, exc_tb; returning `True` suppresses the exception.

**Q6: Why not catch exceptions around large blocks of code?**
> You might accidentally catch an exception from code you didn't intend to protect, masking bugs and making debugging extremely difficult.

**Q7: What's `logger.exception()` shorthand for?**
> It's equivalent to `logger.error(msg, exc_info=True)` — logs the message at ERROR level with the full traceback attached.

**Q8: Should data pipeline validation use assert or raise?**
> Always `raise`. Assertions are stripped in optimized mode (`python -O`), so production validation would silently disappear.

**Q9: How does exponential backoff differ from fixed-interval retry?**
> Exponential backoff (e.g., 2s, 4s, 8s) reduces load on a struggling service, while fixed intervals can create a thundering herd that prevents recovery.

**Q10: What is exception chaining and why does it matter?**
> Using `raise X from Y` preserves the original traceback. Without it, you lose the context of what actually went wrong, making production debugging significantly harder.

---

## 11. Summary Cheat Sheet

```
+------------------------------------------------------------------------+
|  ERROR HANDLING & DEFENSIVE CODING — CHEAT SHEET                       |
+------------------------------------------------------------------------+
|                                                                        |
|  try/except/else/finally:                                              |
|    try     → wrap ONLY the risky operation                             |
|    except  → catch specific exceptions, log, re-raise or transform     |
|    else    → success-path logic (not protected by except)              |
|    finally → cleanup ALWAYS runs (never put return here)               |
|                                                                        |
|  Custom Exceptions:                                                    |
|    - Inherit from Exception (never BaseException)                      |
|    - Carry structured context (column, row_count, missing fields)      |
|    - Use hierarchy: PipelineError → DataQualityError, SchemaError      |
|    - Chain with `raise X from original` to preserve traceback          |
|                                                                        |
|  Context Managers:                                                     |
|    - Use for any acquire/use/release pattern                           |
|    - @contextmanager or __enter__/__exit__                              |
|    - DB connections, temp files, timers, locks                         |
|                                                                        |
|  Logging:                                                              |
|    - DEBUG < INFO < WARNING < ERROR < CRITICAL                         |
|    - Always exc_info=True in except blocks                             |
|    - Never print() in production code                                  |
|                                                                        |
|  Defensive Coding:                                                     |
|    - Validate schema at pipeline boundaries                            |
|    - Null-check with thresholds, not just existence                    |
|    - Retry only transient failures with exponential backoff            |
|    - Fail fast on permanent errors (schema, auth, logic)               |
|                                                                        |
|  Golden Rules:                                                         |
|    1. Never bare except                                                |
|    2. Never swallow errors silently                                    |
|    3. Never assert in production validation                            |
|    4. Always chain exceptions (raise X from Y)                         |
|    5. Always use context managers for resources                        |
|                                                                        |
+------------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 6 of 25*
