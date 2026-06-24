# 🎯 Topic 4: Functions, Closures & Decorators

> **Data Science Interview — Functions, Scope & Metaprogramming Deep Dive**  
> Covers first-class functions, *args/**kwargs, closures, decorators, and functools — the building blocks of clean, reusable data pipelines and the Python patterns that separate senior candidates from junior ones.

---

## Table of Contents

1. [First-Class Functions & Arguments](#first-class-functions--arguments)
2. [Lambda Functions](#lambda-functions)
3. [Scope & the LEGB Rule](#scope--the-legb-rule)
4. [Closures & Factory Functions](#closures--factory-functions)
5. [Decorators — Theory & Practical Patterns](#decorators--theory--practical-patterns)
6. [functools — The Power Tools](#functools--the-power-tools)
7. [Interview Talking Points & Scripts](#interview-talking-points--scripts)
8. [Common Interview Mistakes](#common-interview-mistakes)
9. [Rapid-Fire Q&A](#rapid-fire-qa)
10. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## First-Class Functions & Arguments

In Python, functions are **first-class objects** — they can be assigned to variables, passed as arguments, returned from other functions, and stored in data structures.

> **Critical Insight**: This is what makes decorators, callbacks, and functional-style DS pipelines possible. When an interviewer asks "Why are functions first-class citizens?", they want to hear that functions have the same status as integers or strings — they're objects with a type, an id, and attributes.

### *args and **kwargs

```python
def flexible_aggregator(*args, **kwargs):
    """Accepts any positional and keyword arguments."""
    strategy = kwargs.get('strategy', 'mean')
    data = args  # tuple of positional arguments

    if strategy == 'mean':
        return sum(data) / len(data)
    elif strategy == 'sum':
        return sum(data)

# Usage
flexible_aggregator(1, 2, 3, 4, strategy='mean')  # 2.5
```

**Unpacking in function calls:**

```python
params = {'learning_rate': 0.01, 'epochs': 100, 'batch_size': 32}
train_model(**params)  # Equivalent to train_model(learning_rate=0.01, ...)
```

---

## Lambda Functions

| Feature | `lambda` | `def` |
|---------|----------|-------|
| Body | Single expression only | Multiple statements |
| Name | Anonymous (or assigned) | Named |
| Docstring | Not supported | Supported |
| Readability | Concise for simple ops | Better for complex logic |
| Debugging | Stack trace shows `<lambda>` | Shows function name |
| Use case | Inline callbacks, `sort(key=...)` | Everything else |

```python
# Common DS use cases for lambda
import pandas as pd

df = pd.DataFrame({'revenue': [100, 200, 150]})

# Transform columns
df['log_revenue'] = df['revenue'].apply(lambda x: np.log1p(x))

# Custom sort key
features = ['age', 'income_log', 'tenure_months']
sorted(features, key=lambda f: len(f))  # Sort by name length

# filter + lambda
outliers = list(filter(lambda x: abs(x) > 3, z_scores))
```

> **Critical Insight**: In interviews, always mention that `lambda` is syntactic sugar — it creates the same function object as `def`. Prefer `def` for anything non-trivial. The readability argument matters at Google/Meta scale.

---

## Scope & the LEGB Rule

Python resolves variable names using the **LEGB** lookup order:

| Level | Name | Description | Example |
|-------|------|-------------|---------|
| **L** | Local | Inside the current function | Function parameters, local vars |
| **E** | Enclosing | Inside enclosing (outer) functions | Closure variables |
| **G** | Global | Module-level | Top-level assignments |
| **B** | Built-in | Python's built-in namespace | `len`, `print`, `range` |

```python
x = "global"  # G

def outer():
    x = "enclosing"  # E

    def inner():
        x = "local"  # L
        print(x)     # "local" — L wins

    inner()
    print(x)  # "enclosing" — E level

outer()
print(x)  # "global" — G level
```

### `nonlocal` and `global` keywords

```python
def counter():
    count = 0  # Enclosing scope

    def increment():
        nonlocal count   # Without this, assignment creates LOCAL count
        count += 1
        return count

    return increment

c = counter()
c()  # 1
c()  # 2
```

> **Critical Insight**: The `nonlocal` keyword is what makes mutable closures possible. Without it, `count += 1` would raise `UnboundLocalError` because Python sees the assignment and treats `count` as local.

---

## Closures & Factory Functions

A **closure** is a function that remembers the variables from its enclosing scope even after that scope has finished executing.

```python
def make_normalizer(mean, std):
    """Factory: returns a normalizer function bound to specific stats."""
    def normalize(x):
        return (x - mean) / std  # mean and std are "closed over"
    return normalize

# Create specialized normalizers for different features
age_normalizer = make_normalizer(mean=35.2, std=12.1)
income_normalizer = make_normalizer(mean=55000, std=20000)

age_normalizer(40)  # 0.396...
```

**Why closures matter in Data Science:**

- **Feature pipelines**: Factory functions create reusable transformers bound to fitted statistics.
- **Callback factories**: Create experiment-specific logging callbacks.
- **Configuration injection**: Avoid global state by closing over config.

### The Late-Binding Closure Trap

```python
# BUG: All functions reference the SAME variable i
funcs = [lambda: i for i in range(5)]
[f() for f in funcs]  # [4, 4, 4, 4, 4] — NOT [0, 1, 2, 3, 4]

# FIX: Capture value at definition time via default argument
funcs = [lambda i=i: i for i in range(5)]
[f() for f in funcs]  # [0, 1, 2, 3, 4]
```

---

## Decorators — Theory & Practical Patterns

A decorator is a function that takes a function and returns a modified function. It's syntactic sugar for `func = decorator(func)`.

### Pattern Comparison Table

| Decorator Type | Use Case | Takes Arguments? | Structure |
|----------------|----------|-----------------|-----------|
| Simple | Timer, logging | No | `def decorator(func)` |
| Parameterized | Retry with N attempts | Yes | `def decorator(n): def wrapper(func): ...` |
| Class-based | Stateful (call counting) | Optional | `class Decorator: def __call__` |
| Stacked | Compose behaviors | N/A | Applied bottom-up |

### Timer Decorator

```python
import time
from functools import wraps

def timer(func):
    """Measure execution time — essential for profiling DS pipelines."""
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper

@timer
def train_model(X, y):
    # ... expensive training
    pass
```

### Retry with Exponential Backoff

```python
import time
from functools import wraps

def retry(max_attempts=3, backoff_factor=2):
    """Parameterized decorator — retries on failure with exponential backoff."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts:
                        raise
                    wait = backoff_factor ** attempt
                    print(f"Attempt {attempt} failed: {e}. Retrying in {wait}s...")
                    time.sleep(wait)
        return wrapper
    return decorator

@retry(max_attempts=5, backoff_factor=2)
def fetch_training_data(url):
    """Fetch data from flaky API endpoint."""
    response = requests.get(url, timeout=10)
    response.raise_for_status()
    return response.json()
```

### Memoize / Cache Decorator

```python
from functools import wraps

def memoize(func):
    """Cache results — critical for expensive feature computations."""
    cache = {}

    @wraps(func)
    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]
    return wrapper

@memoize
def compute_distance_matrix(n_points):
    """O(n^2) computation we only want to run once per input size."""
    # ... expensive computation
    pass
```

> **Critical Insight**: Decorators execute at **import time** (when the module loads), not at call time. The `@decorator` line runs immediately and replaces the function. This is why `@wraps` matters — without it, introspection tools (pytest, Sphinx, debuggers) see the wrapper, not the original function.

---

## functools — The Power Tools

### `lru_cache` — Built-in Memoization

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def expensive_feature_engineering(customer_id: int) -> dict:
    """Cache expensive DB lookups during batch processing."""
    # Simulates expensive computation
    return query_database(customer_id)

# Check cache performance
expensive_feature_engineering.cache_info()
# CacheInfo(hits=45, misses=10, maxsize=128, currsize=10)

# Clear cache when underlying data changes
expensive_feature_engineering.cache_clear()
```

**DS application**: Memoize recursive algorithms (dynamic programming), expensive feature lookups, and repeated statistical computations during cross-validation.

### `partial` — Currying / Argument Freezing

```python
from functools import partial

def train_with_hyperparams(X, y, lr, batch_size, epochs):
    pass

# Freeze hyperparameters, vary only data
quick_train = partial(train_with_hyperparams, lr=0.001, batch_size=32, epochs=10)
full_train = partial(train_with_hyperparams, lr=0.0001, batch_size=64, epochs=100)

# Now just pass data
quick_train(X_train, y_train)
full_train(X_train, y_train)
```

### `wraps` — Metadata Preservation

```python
from functools import wraps

# WITHOUT @wraps
def bad_decorator(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@bad_decorator
def my_func():
    """Important docstring."""
    pass

print(my_func.__name__)  # "wrapper"  <-- WRONG
print(my_func.__doc__)   # None       <-- LOST

# WITH @wraps — always use this
def good_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@good_decorator
def my_func():
    """Important docstring."""
    pass

print(my_func.__name__)  # "my_func"            <-- CORRECT
print(my_func.__doc__)   # "Important docstring" <-- PRESERVED
```

---

## Interview Talking Points & Scripts

> *"Functions in Python are first-class objects — they can be assigned to variables, stored in collections, and passed as arguments. This is what enables the decorator pattern and functional-style data pipelines. In my work, I use closures to create factory functions that bind fitted parameters — like creating a normalizer that remembers the training set's mean and standard deviation."*

> *"Decorators are my go-to for cross-cutting concerns in ML pipelines. I typically implement a @timer for profiling, a @retry with exponential backoff for API calls during data collection, and @lru_cache for memoizing expensive feature computations. The key implementation detail is always using @wraps from functools to preserve the decorated function's metadata for debugging and documentation."*

> *"The LEGB rule governs Python's name resolution — Local, Enclosing, Global, Built-in. This matters most when working with closures. A common trap is late binding: if you create lambdas in a loop, they all share the loop variable's final value. The fix is capturing the value at definition time through a default argument."*

> *"For memoization in Data Science, I prefer functools.lru_cache for pure functions with hashable arguments. It gives you cache_info() for monitoring hit rates and cache_clear() for invalidation. For more complex caching needs — like DataFrames that aren't hashable — I implement a custom decorator with a dictionary-based cache keyed on a hash of the input."*

---

## Common Interview Mistakes

| Mistake | Why It's Wrong | Correct Approach |
|---------|---------------|-----------------|
| Mutable default argument `def f(lst=[])` | The default list is shared across ALL calls — it's created once at function definition | Use `def f(lst=None): lst = lst or []` |
| Forgetting `@wraps(func)` in decorators | Loses `__name__`, `__doc__`, `__module__` — breaks debugging, testing, documentation | Always apply `@wraps(func)` to the inner wrapper |
| Late binding in closures / loop lambdas | Closures capture the **variable**, not its **value** — all lambdas see the final loop value | Use default argument `lambda i=i: i` to capture by value |
| Using `global` instead of proper design | Creates hidden dependencies, makes testing impossible, breaks parallelism | Pass state explicitly or use closures |
| Caching functions with mutable args | `lru_cache` requires hashable arguments — lists and dicts will raise `TypeError` | Convert to tuples/frozensets, or use a custom cache with hashed keys |
| Returning lambda from a function without testing | Hard to debug since stack traces show `<lambda>` | Return a named inner function for better traceability |
| Applying decorators in wrong order | Stacked decorators apply **bottom-up** — `@a @b def f` means `a(b(f))` | Think of it as wrapping: innermost decorator touches `f` first |

---

## Rapid-Fire Q&A

**Q1: What's the difference between `*args` and `**kwargs`?**  
A: `*args` collects extra positional arguments into a tuple. `**kwargs` collects extra keyword arguments into a dictionary. Together they let a function accept any signature — essential for decorator wrappers.

**Q2: Can a lambda have multiple statements?**  
A: No. A lambda is limited to a single expression. For anything requiring statements (assignments, loops, try/except), use `def`.

**Q3: What is a closure?**  
A: A function that retains access to variables from its enclosing scope after that scope has finished executing. The variables are stored in the function's `__closure__` attribute.

**Q4: Why does `lru_cache` require hashable arguments?**  
A: It uses a dictionary internally — keys must be hashable. Lists and dicts fail. Convert to tuples or use `@lru_cache` only on functions with immutable parameters.

**Q5: What happens if you stack `@decorator_a` above `@decorator_b`?**  
A: `decorator_a(decorator_b(func))` — `b` wraps first, then `a` wraps the result. Execution order when calling: `a`'s wrapper runs first (outermost), then `b`'s, then the original function.

**Q6: How do you make a decorator that accepts arguments?**  
A: Add an outer function: `def my_decorator(arg): def decorator(func): @wraps(func) def wrapper(...): ... return wrapper; return decorator`. Three levels of nesting.

**Q7: What's `functools.partial` used for?**  
A: It freezes some arguments of a function, returning a new callable with fewer parameters. Useful for creating specialized versions of generic functions without subclassing.

**Q8: What does `nonlocal` do?**  
A: It tells Python that a variable assignment in an inner function should modify the variable in the enclosing scope, not create a new local variable.

**Q9: When would you choose a class-based decorator over a function-based one?**  
A: When you need to maintain state across calls (like counting invocations) or when the decorator logic is complex enough to benefit from methods and attributes.

**Q10: What's the performance cost of decorators?**  
A: One extra function call per invocation (the wrapper). Usually negligible, but in tight loops over millions of iterations, it can matter. Profile before optimizing.

---

## Summary Cheat Sheet

```
+------------------------------------------------------------------+
|           FUNCTIONS, CLOSURES & DECORATORS CHEAT SHEET            |
+------------------------------------------------------------------+
| FIRST-CLASS FUNCTIONS                                            |
|   - Functions are objects: assignable, passable, storable        |
|   - *args → tuple of positional args                             |
|   - **kwargs → dict of keyword args                              |
|   - lambda → single-expression anonymous function                |
+------------------------------------------------------------------+
| LEGB SCOPE RULE                                                  |
|   Local → Enclosing → Global → Built-in                         |
|   - nonlocal: modify enclosing scope variable                    |
|   - global: modify module-level variable (avoid!)                |
+------------------------------------------------------------------+
| CLOSURES                                                         |
|   - Function + captured enclosing variables                      |
|   - Use case: factory functions, stateful callbacks              |
|   - Trap: late binding in loops → fix with default args          |
+------------------------------------------------------------------+
| DECORATORS                                                       |
|   - @decorator = func = decorator(func)                          |
|   - ALWAYS use @wraps(func) on the wrapper                       |
|   - Parameterized = 3 levels of nesting                          |
|   - Stacked: bottom decorator applies first                      |
|   - Common: @timer, @retry, @memoize, @validate                 |
+------------------------------------------------------------------+
| FUNCTOOLS ESSENTIALS                                              |
|   - lru_cache(maxsize): memoize pure functions (hashable args)   |
|   - partial(func, *args, **kwargs): freeze arguments             |
|   - wraps(func): preserve __name__, __doc__, __module__          |
+------------------------------------------------------------------+
| GOLDEN RULES                                                     |
|   1. Never use mutable defaults → use None + conditional         |
|   2. Always @wraps in decorators                                 |
|   3. Closures capture variables, not values                      |
|   4. Prefer def over lambda for anything non-trivial             |
|   5. lru_cache only works with hashable arguments                |
+------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 4 of 25*
