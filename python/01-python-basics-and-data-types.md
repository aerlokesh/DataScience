# 🎯 Topic 1: Python Basics & Data Types

> **Data Science Interview — Python Fundamentals Deep Dive**  
> Everything you need to know about Python's type system, operators, and core mechanics — with interview talking points, common mistakes, and code that proves you understand the "why" not just the "what."

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Data Types & Mutability](#data-types--mutability)
3. [Strings — The Most Tested Type](#strings--the-most-tested-type)
4. [Operators & Gotchas](#operators--gotchas)
5. [Type Conversion & Coercion](#type-conversion--coercion)
6. [Truthiness & Falsy Values](#truthiness--falsy-values)
7. [Interview Talking Points & Scripts](#interview-talking-points--scripts)
8. [Common Interview Mistakes](#common-interview-mistakes)
9. [Rapid-Fire Q&A (Top 10)](#rapid-fire-qa-top-10)
10. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## Core Concepts

Python is a **dynamically typed, strongly typed** language:

- **Dynamically typed**: Variables don't declare types — the object they point to has a type.
- **Strongly typed**: Python won't implicitly convert between incompatible types (`"5" + 3` raises `TypeError`, unlike JavaScript).

### The Critical Insight for Interviews

> Everything in Python is an **object**. Variables are just **names** (references) pointing to objects in memory. When you assign `a = b`, you're not copying data — you're making `a` point to the same object as `b`.

```python
a = [1, 2, 3]
b = a          # b points to SAME list object
b.append(4)
print(a)       # [1, 2, 3, 4] — a is affected!

c = a[:]       # c is a COPY (shallow)
c.append(5)
print(a)       # [1, 2, 3, 4] — a is NOT affected
```

---

## Data Types & Mutability

### The Mutable vs Immutable Distinction (Asked in 90% of interviews!)

| Type | Mutable? | Example | Can be dict key? |
|------|----------|---------|-----------------|
| `int` | No | `42` | Yes |
| `float` | No | `3.14` | Yes |
| `str` | No | `"hello"` | Yes |
| `tuple` | No | `(1, 2, 3)` | Yes |
| `frozenset` | No | `frozenset({1,2})` | Yes |
| `list` | **Yes** | `[1, 2, 3]` | **No** |
| `dict` | **Yes** | `{"a": 1}` | **No** |
| `set` | **Yes** | `{1, 2, 3}` | **No** |

### Why This Matters

> **Talking Point**: "Immutable objects can be hashed and used as dictionary keys or set elements. This is why tuples work as keys for caching results — `@lru_cache` requires hashable arguments, so you can't pass a list directly."

### Integer Caching & String Interning

Python **caches** small integers (-5 to 256) and short strings for performance:

```python
a = 256
b = 256
print(a is b)  # True — same object (cached)

a = 257
b = 257
print(a is b)  # False (may vary) — different objects!
```

**Interview Trap**: Never rely on `is` for value comparison. Always use `==` for values, `is` only for `None` checks.

---

## Strings — The Most Tested Type

### Essential String Operations

```python
s = "Data Science"

# Slicing (know this cold!)
s[0:4]      # "Data"
s[-7:]      # "Science"  
s[::-1]     # "ecneicS ataD" (reverse)
s[::2]      # "Dt cec" (every 2nd char)

# Key methods
s.split()           # ["Data", "Science"]
s.lower()           # "data science"
s.replace("Data", "Computer")  # "Computer Science"
s.strip()           # removes whitespace
s.find("Science")   # 5 (index, -1 if not found)
s.count("a")        # 2
"_".join(["a","b"]) # "a_b"
```

### F-Strings (Always use these in interviews — shows modern Python)

```python
name, accuracy = "Random Forest", 0.9456
print(f"{name}: {accuracy:.2%}")  # "Random Forest: 94.56%"
print(f"{1000000:,}")             # "1,000,000"
print(f"{'centered':^20}")        # "      centered      "
```

### String Immutability Consequence

```python
s = "hello"
# s[0] = "H"  # TypeError! Strings are immutable
s = "H" + s[1:]  # Creates a NEW string "Hello"
```

---

## Operators & Gotchas

### `==` vs `is` (The #1 Python Interview Question)

| Operator | Checks | Use For |
|----------|--------|---------|
| `==` | Value equality | Everything |
| `is` | Identity (same object in memory) | Only `None` checks |

```python
a = [1, 2, 3]
b = [1, 2, 3]
print(a == b)   # True (same value)
print(a is b)   # False (different objects!)

x = None
if x is None:   # CORRECT way to check None
    pass
```

### Floor Division & Modulo

```python
print(7 / 2)    # 3.5 (true division)
print(7 // 2)   # 3 (floor division — rounds DOWN)
print(-7 // 2)  # -4 (not -3! rounds toward negative infinity)
print(7 % 2)    # 1 (modulo)
```

### Short-Circuit Evaluation

```python
# `and` returns first falsy OR last value
0 and 5        # 0
3 and 5        # 5

# `or` returns first truthy OR last value  
0 or 5         # 5
3 or 5         # 3
"" or "default"  # "default" (common pattern!)
```

**Practical Use** — Default values:
```python
name = user_input or "Anonymous"
```

---

## Type Conversion & Coercion

### Implicit (Automatic)

```python
3 + 2.5      # 5.5 (int promoted to float)
True + True  # 2 (bool is subclass of int!)
```

### Explicit — Know the Edge Cases

```python
int(3.9)      # 3 (TRUNCATES, does not round!)
int("42")     # 42
int("3.14")   # ValueError! Can't go str→float→int directly
float("inf")  # infinity (valid!)
bool([])      # False
bool([0])     # True (non-empty list, even if contains 0!)
list("hello") # ['h', 'e', 'l', 'l', 'o']
```

---

## Truthiness & Falsy Values

### The Complete Falsy List (Memorize this!)

```python
# ALL of these evaluate to False:
bool(False)    # False
bool(None)     # False  
bool(0)        # False
bool(0.0)      # False
bool("")       # False (empty string)
bool([])       # False (empty list)
bool({})       # False (empty dict)
bool(set())    # False (empty set)
```

> **Everything else is truthy**, including: `[0]`, `" "` (space), `[None]`, `{0: 0}`

---

## Interview Talking Points & Scripts

### When asked "Tell me about Python's type system"

> *"Python is dynamically typed — we don't declare types — but strongly typed, meaning implicit conversions between incompatible types aren't allowed. Everything is an object, and variables are references to objects. The key distinction is mutability: mutable objects like lists and dicts can be modified in place, while immutable objects like strings and tuples cannot. This matters for function arguments — if I pass a list to a function, modifications inside affect the caller, which is a common source of bugs."*

### When asked about the mutable default argument trap

> *"Default arguments are evaluated once at function definition time, not at each call. So a mutable default like `def f(lst=[])` shares the same list across all calls. The fix is to use `None` as the default and create a new list inside the function. I've seen this cause bugs in production where cached results leaked between requests."*

### When asked "`is` vs `==`?"

> *"`==` checks value equality — do these objects have the same content? `is` checks identity — are these literally the same object in memory? The only time I use `is` is checking for None, because None is a singleton. Using `is` for other comparisons is unreliable due to Python's caching of small integers and interned strings."*

---

## Common Interview Mistakes

| ❌ Mistake | Why It's Wrong | ✅ Better |
|-----------|---------------|----------|
| Using `is` to compare values | `is` checks identity, not value | Use `==` for values, `is` only for `None` |
| `def f(lst=[])` | Mutable default shared across calls | `def f(lst=None): lst = lst or []` |
| `0.1 + 0.2 == 0.3` → True? | Floating point! It's `False` | Use `math.isclose(0.1+0.2, 0.3)` |
| Saying "Python is weakly typed" | Python is STRONGLY typed | Say "dynamically typed, strongly typed" |
| Not knowing `bool` subclasses `int` | `True + True = 2` is valid | Mention it to show depth |
| Confusing `//` with `/` | `-7//2 = -4` not `-3` | `//` floors toward negative infinity |

---

## Rapid-Fire Q&A (Top 10)

**Q1: Pass-by-value or pass-by-reference?**  
Neither! **Pass-by-object-reference**. The function gets a reference to the same object. Mutable objects can be changed in place; immutable ones can't.

**Q2: What's `type(type(42))`?**  
`<class 'type'>` — the type of any class is `type` (the metaclass).

**Q3: How is memory managed?**  
Reference counting + cyclic garbage collector. When ref count hits 0, immediate deallocation. GC handles circular refs.

**Q4: What's the GIL?**  
Global Interpreter Lock prevents multiple threads from executing Python bytecode simultaneously. CPU-bound → use multiprocessing. I/O-bound → threading still works.

**Q5: `sorted()` vs `.sort()`?**  
`sorted()` returns a NEW list. `.sort()` sorts IN-PLACE, returns `None`.

**Q6: List as dictionary key?**  
`TypeError: unhashable type`. Lists are mutable. Use a tuple instead.

**Q7: Swap two variables?**  
`a, b = b, a` — tuple unpacking, no temp variable.

**Q8: What's `None`?**  
Singleton of type `NoneType`. Always check with `is None`.

**Q9: Why f-strings?**  
Faster and more readable than `.format()` or `%`. Evaluate expressions inline: `f"{len(data):,} rows"`.

**Q10: `deepcopy` vs shallow copy?**  
Shallow (`list[:]`, `copy()`) copies container, shares nested objects. `deepcopy()` recursively copies everything.

---

## Summary Cheat Sheet

```
╔══════════════════════════════════════════════════════════════╗
║                PYTHON BASICS CHEAT SHEET                     ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  TYPES:     Immutable: int, float, str, tuple, frozenset    ║
║             Mutable:   list, dict, set                       ║
║                                                              ║
║  EQUALITY:  == (value)  vs  is (identity/None only)         ║
║                                                              ║
║  FALSY:     False, None, 0, 0.0, "", [], {}, set(), ()      ║
║                                                              ║
║  DIVISION:  / (true: 3.5)  // (floor: 3)  % (mod: 1)       ║
║                                                              ║
║  STRINGS:   Immutable, f-strings preferred, [::-1] reverses ║
║                                                              ║
║  GOTCHAS:   Mutable defaults, float precision, int caching  ║
║                                                              ║
║  MEMORY:    Reference counting + GC, GIL for threads        ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 1 of 25*
