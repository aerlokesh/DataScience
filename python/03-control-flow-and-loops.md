# 🎯 Topic 3: Control Flow, Loops & Iteration

> **Scope:** for/while loops, list/dict/set comprehensions, generators vs lists, itertools essentials
> (chain, product, combinations, groupby), enumerate/zip, map/filter/reduce, lazy evaluation,
> memory efficiency patterns, and common iteration pitfalls tested at Google, Meta, Amazon, Netflix, and Spotify.

---

## Table of Contents

1. [For & While Loops — Beyond the Basics](#1-for--while-loops--beyond-the-basics)
2. [Comprehensions — Lists, Dicts, Sets](#2-comprehensions--lists-dicts-sets)
3. [Generators vs Lists — Lazy Evaluation](#3-generators-vs-lists--lazy-evaluation)
4. [itertools Essentials for Data Science](#4-itertools-essentials-for-data-science)
5. [enumerate, zip, map, filter, reduce](#5-enumerate-zip-map-filter-reduce)
6. [Common Interview Mistakes](#6-common-interview-mistakes)
7. [Interview Talking Points & Scripts](#7-interview-talking-points--scripts)
8. [Rapid-Fire Q&A](#8-rapid-fire-qa)
9. [Summary Cheat Sheet](#9-summary-cheat-sheet)

---

## 1. For & While Loops — Beyond the Basics

```python
# for-else pattern (rarely seen but asked at Google)
def find_prime_factor(n):
    for i in range(2, int(n**0.5) + 1):
        if n % i == 0:
            return i
    else:
        # Only runs if loop completes without break
        return n  # n is prime

# while with walrus operator (Python 3.8+)
import random
while (value := random.randint(1, 10)) != 7:
    print(f"Got {value}, trying again...")
```

> **Critical Insight:** The `for-else` clause executes only when the loop finishes *without* hitting a `break`. Interviewers at Meta love asking this because most candidates get it wrong.

---

## 2. Comprehensions — Lists, Dicts, Sets

```python
# List comprehension with condition
scores = [85, 92, 78, 95, 60, 88]
high_scores = [s for s in scores if s >= 80]

# Nested comprehension — flattening a matrix
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = [x for row in matrix for x in row]

# Dict comprehension — feature engineering
raw_features = {"age": 25, "income": 75000, "score": 680}
normalized = {k: v / max(raw_features.values()) for k, v in raw_features.items()}

# Set comprehension — unique categories
records = [("user1", "click"), ("user2", "view"), ("user1", "click")]
unique_events = {event for _, event in records}

# Conditional expression inside comprehension
labels = [f"high_{col}" if val > 0.5 else f"low_{col}"
          for col, val in normalized.items()]
```

### Comprehension vs Loop Performance

| Approach | 1M items (time) | Readability | Memory |
|----------|----------------|-------------|--------|
| List comprehension | ~45ms | High | O(n) — stores all |
| Equivalent for-loop | ~65ms | Medium | O(n) — stores all |
| Generator expression | ~0ms (lazy) | High | O(1) — yields one |

> **Critical Insight:** Comprehensions are ~30% faster than equivalent loops because they are optimized at the bytecode level (single `LIST_APPEND` instruction vs `LOAD_ATTR` + `CALL_FUNCTION` for `.append()`).

---

## 3. Generators vs Lists — Lazy Evaluation

```python
# Generator expression — O(1) memory
sum_squares = sum(x**2 for x in range(10_000_000))  # No list created!

# Generator function with yield
def read_large_csv(filepath):
    """Process 10GB file without loading into memory."""
    with open(filepath) as f:
        header = next(f).strip().split(",")
        for line in f:
            yield dict(zip(header, line.strip().split(",")))

# Chaining generators — pipeline pattern (common in DS)
def parse_records(rows):
    for row in rows:
        yield {k: float(v) if v.replace(".", "").isdigit() else v
               for k, v in row.items()}

def filter_valid(records):
    for rec in records:
        if rec.get("score", 0) > 0:
            yield rec

# Entire pipeline is lazy — processes one record at a time
pipeline = filter_valid(parse_records(read_large_csv("data.csv")))
```

### Why Lazy Evaluation Matters for Large Datasets

| Scenario | List Approach | Generator Approach |
|----------|--------------|-------------------|
| 10M row CSV processing | 8GB RAM, OOM risk | ~50KB constant |
| Streaming API responses | Wait for all, then process | Process as they arrive |
| Feature pipeline (5 transforms) | 5 intermediate lists | Single pass, O(1) extra |
| Early termination (find first match) | Compute all, then pick | Stop immediately |

```python
# Memory demonstration
import sys

list_version = [x**2 for x in range(1_000_000)]
gen_version = (x**2 for x in range(1_000_000))

print(sys.getsizeof(list_version))  # ~8,448,728 bytes
print(sys.getsizeof(gen_version))   # 112 bytes (!)
```

> **Critical Insight:** Generators are *single-use*. Once exhausted, they yield nothing. You cannot index, slice, or `len()` a generator. If you need multiple passes over data, either re-create the generator or materialize into a list.

---

## 4. itertools Essentials for Data Science

```python
from itertools import chain, product, combinations, groupby, islice, accumulate

# chain — merge multiple iterables (combine train/val/test sets)
train = [1, 2, 3]
val = [4, 5]
test = [6, 7]
all_data = list(chain(train, val, test))  # [1, 2, 3, 4, 5, 6, 7]

# product — hyperparameter grid search
learning_rates = [0.001, 0.01, 0.1]
batch_sizes = [32, 64, 128]
for lr, bs in product(learning_rates, batch_sizes):
    print(f"Training with lr={lr}, batch_size={bs}")

# combinations — feature interaction pairs
features = ["age", "income", "tenure", "score"]
interaction_pairs = list(combinations(features, 2))
# [('age','income'), ('age','tenure'), ('age','score'), ...]

# groupby — aggregate (data MUST be sorted by key first!)
from operator import itemgetter
data = sorted([("A", 10), ("B", 20), ("A", 30), ("B", 40)], key=itemgetter(0))
for key, group in groupby(data, key=itemgetter(0)):
    values = [v for _, v in group]
    print(f"{key}: mean={sum(values)/len(values)}")

# islice — take first N from generator without materializing
first_100 = list(islice(read_large_csv("huge.csv"), 100))
```

### itertools Function Reference Table

| Function | Use Case in DS | Time Complexity | Returns |
|----------|---------------|-----------------|---------|
| `chain(*iterables)` | Merge datasets | O(total items) | Iterator |
| `product(*iterables)` | Grid search params | O(n1 * n2 * ...) | Iterator |
| `combinations(it, r)` | Feature pairs | O(C(n,r)) | Iterator |
| `permutations(it, r)` | Ordered arrangements | O(P(n,r)) | Iterator |
| `groupby(it, key)` | Split-apply-combine | O(n) | Iterator of (key, group) |
| `islice(it, stop)` | Sample from stream | O(stop) | Iterator |
| `accumulate(it, func)` | Running totals/max | O(n) | Iterator |
| `starmap(func, it)` | Apply func to tuples | O(n) | Iterator |

> **Critical Insight:** `groupby` requires data to be **pre-sorted** by the grouping key. Forgetting this is the #1 `groupby` bug in interviews. Unlike pandas `groupby`, itertools' version only groups *consecutive* identical keys.

---

## 5. enumerate, zip, map, filter, reduce

```python
# enumerate — avoid index tracking anti-pattern
features = ["age", "income", "score"]
for idx, feat in enumerate(features, start=1):
    print(f"Feature {idx}: {feat}")

# zip — parallel iteration (stops at shortest)
predictions = [0.9, 0.3, 0.7, 0.8]
actuals = [1, 0, 1, 1]
errors = [abs(p - a) for p, a in zip(predictions, actuals)]

# zip_longest for unequal lengths
from itertools import zip_longest
list(zip_longest([1,2,3], [4,5], fillvalue=0))  # [(1,4), (2,5), (3,0)]

# map — apply transform (returns iterator in Python 3)
raw_inputs = ["  Hello ", " World  ", "Python  "]
cleaned = list(map(str.strip, raw_inputs))

# filter — select elements matching predicate
values = [0, 1, None, 3, "", 5, False]
truthy = list(filter(None, values))  # [1, 3, 5]

# reduce — cumulative computation
from functools import reduce
nums = [1, 2, 3, 4, 5]
product_val = reduce(lambda a, b: a * b, nums)  # 120
```

### Comparison: Comprehension vs map/filter vs Loop

| Pattern | Syntax | Returns | Best When |
|---------|--------|---------|-----------|
| List comprehension | `[f(x) for x in it if cond]` | list | Transform + filter together |
| `map(func, it)` | `map(f, it)` | iterator (lazy) | Single existing function to apply |
| `filter(pred, it)` | `filter(f, it)` | iterator (lazy) | Simple predicate, no transform |
| Generator expression | `(f(x) for x in it)` | generator | Large data, single pass needed |
| For-loop | `for x in it: ...` | None (side effects) | Complex logic, multiple statements |

> **Critical Insight:** In modern Python, list comprehensions are preferred over `map`/`filter` for readability when a lambda is needed. Use `map` only when you already have a named function (e.g., `map(int, strings)`).

---

## 6. Common Interview Mistakes

| # | Mistake (Wrong) | Correct Approach | Why It Matters |
|---|----------------|------------------|----------------|
| 1 | Modifying a list while iterating over it | Iterate over a copy: `for x in items[:]` or build a new list | Causes skipped elements or `RuntimeError` with dicts |
| 2 | Using `range(len(lst))` to access index | Use `enumerate(lst)` | Pythonic, less error-prone |
| 3 | Forgetting generators are single-use | Re-create or use `itertools.tee()` | Silent bugs: second loop produces nothing |
| 4 | `groupby` on unsorted data | Sort by key first, then `groupby` | Groups only consecutive identical keys |
| 5 | Infinite `while True` without break condition | Always ensure a reachable exit | Interview red flag: shows lack of rigor |
| 6 | Nested comprehension too deep (3+ levels) | Switch to explicit loops or helper functions | Readability matters in production code |
| 7 | `map(lambda x: x+1, lst)` instead of comprehension | `[x+1 for x in lst]` | Lambda + map is slower and less readable |
| 8 | Not using `itertools` for combinatorial problems | Import and use `combinations`/`product` | Reinventing wheels wastes interview time |

```python
# WRONG: Modifying list while iterating
numbers = [1, 2, 3, 4, 5, 6]
for n in numbers:
    if n % 2 == 0:
        numbers.remove(n)  # Bug! Skips elements
# Result: [1, 3, 5] sometimes, [1, 3, 6] other times

# CORRECT: Filter into new list
numbers = [1, 2, 3, 4, 5, 6]
odds = [n for n in numbers if n % 2 != 0]  # Always [1, 3, 5]
```

---

## 7. Interview Talking Points & Scripts

> *"When I work with large datasets — say processing 100M rows of event logs — I default to generator pipelines rather than list comprehensions. Each stage in the pipeline yields one record at a time, which keeps memory constant regardless of dataset size. I've used this pattern extensively in production ETL jobs where we process terabytes of clickstream data without hitting memory limits."*

> *"For hyperparameter tuning, I rely on `itertools.product` to generate the full grid lazily. Combined with early stopping logic, this means we don't materialize the entire search space upfront. For feature engineering, `combinations(features, 2)` gives me all interaction pairs without manual enumeration — critical when working with 50+ features."*

> *"I think of Python iteration as a spectrum: lists when you need random access or multiple passes, generators when you need memory efficiency or streaming, and itertools when you need combinatorial logic. The choice directly impacts whether your notebook can handle the full dataset or crashes on the 10% sample."*

> *"One pattern I always follow is never modifying a collection while iterating over it. In data pipelines, this means I build new filtered collections rather than removing items in-place. It eliminates an entire class of subtle bugs that are painful to debug in production."*

---

## 8. Rapid-Fire Q&A

**Q1: What's the difference between `range(10)` in Python 2 vs Python 3?**
Python 2 returns a list; Python 3 returns a lazy range object (iterator-like). This is why Python 3 doesn't need `xrange`.

**Q2: Can you restart a generator?**
No. Once exhausted, you must create a new generator instance. Alternatively, use `itertools.tee()` for limited reuse (but it buffers consumed items, so memory advantage is lost).

**Q3: When would you choose a while loop over a for loop?**
When the number of iterations isn't known in advance — e.g., convergence loops in optimization (gradient descent until loss < threshold).

**Q4: What does `yield from` do?**
It delegates to a sub-generator, flattening nested yields. Equivalent to `for item in sub_gen: yield item` but more efficient.

**Q5: Why is list comprehension faster than a for-loop with append?**
The comprehension uses a specialized bytecode instruction (`LIST_APPEND`) and avoids repeated attribute lookups for the `.append` method.

**Q6: What happens if you `zip` iterables of different lengths?**
`zip` stops at the shortest. Use `itertools.zip_longest(fillvalue=...)` to pad the shorter ones.

**Q7: How does `reduce` differ from `accumulate`?**
`reduce` returns a single final value. `accumulate` returns an iterator of all intermediate results (useful for running totals).

**Q8: What's the memory complexity of `itertools.combinations(range(1000), 2)`?**
O(1) for the iterator object itself. It generates each combination on-the-fly without storing all 499,500 pairs simultaneously.

**Q9: Can a dictionary comprehension have duplicate keys?**
Yes syntactically, but later keys overwrite earlier ones. Only the last value for each key survives.

**Q10: What's the Pythonic way to loop over two lists with their indices?**
`for idx, (a, b) in enumerate(zip(list1, list2)):`

---

## 9. Summary Cheat Sheet

```
+------------------------------------------------------------------+
|           CONTROL FLOW & ITERATION — CHEAT SHEET                  |
+------------------------------------------------------------------+
| COMPREHENSIONS                                                    |
|   List:  [expr for x in it if cond]                              |
|   Dict:  {k: v for k, v in it}                                   |
|   Set:   {expr for x in it}                                      |
|   Gen:   (expr for x in it)  <-- lazy, O(1) memory               |
+------------------------------------------------------------------+
| GENERATORS                                                        |
|   - Use yield in functions for custom iterators                   |
|   - Single-use: cannot rewind or index                            |
|   - Chain them for memory-efficient pipelines                     |
|   - sys.getsizeof(gen) ≈ 112 bytes regardless of data size       |
+------------------------------------------------------------------+
| ITERTOOLS TOP 5 FOR DS                                            |
|   chain(*its)        → merge multiple datasets                    |
|   product(*its)      → hyperparameter grids                       |
|   combinations(it,r) → feature interaction pairs                  |
|   groupby(it, key)   → split-apply (SORT FIRST!)                 |
|   islice(it, n)      → sample from stream without loading all     |
+------------------------------------------------------------------+
| PATTERNS                                                          |
|   enumerate(it, start=0)   → index + value                       |
|   zip(a, b)                → parallel iteration                   |
|   map(func, it)            → lazy transform                       |
|   filter(pred, it)         → lazy selection                       |
|   reduce(func, it, init)   → single accumulated value            |
+------------------------------------------------------------------+
| GOLDEN RULES                                                      |
|   1. Never modify a collection while iterating over it            |
|   2. Prefer generators for large data (>1M rows)                  |
|   3. Use comprehensions over map+lambda                           |
|   4. Sort before groupby                                          |
|   5. Generators are single-use — plan accordingly                 |
+------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 3 of 25*
