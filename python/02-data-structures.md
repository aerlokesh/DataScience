# 🎯 Topic 2: Python Data Structures

> **Data Science Interview — Deep Dive**  
> Lists, Tuples, Dictionaries, Sets — operations, time complexity, and when to use which. The foundation of every coding interview question.

---

## Table of Contents

1. [The Big Four](#the-big-four)
2. [Time Complexity — Know This Cold](#time-complexity--know-this-cold)
3. [Lists — Ordered, Mutable, Most Used](#lists--ordered-mutable-most-used)
4. [Dictionaries — The Interview Workhorse](#dictionaries--the-interview-workhorse)
5. [Sets — For Uniqueness and Fast Lookup](#sets--for-uniqueness-and-fast-lookup)
6. [Tuples — Immutable and Hashable](#tuples--immutable-and-hashable)
7. [Collections Module — Power Tools](#collections-module--power-tools)
8. [Interview Talking Points & Scripts](#interview-talking-points--scripts)
9. [Common Interview Mistakes](#common-interview-mistakes)
10. [Classic Interview Problems](#classic-interview-problems)
11. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## The Big Four

| Structure | Ordered? | Mutable? | Duplicates? | Use Case |
|-----------|----------|----------|-------------|----------|
| `list` | Yes | Yes | Yes | General-purpose, sequences |
| `tuple` | Yes | No | Yes | Fixed data, dict keys, function returns |
| `dict` | Yes (3.7+) | Yes | Keys: No | Key-value mapping, O(1) lookup |
| `set` | No | Yes | No | Uniqueness, membership testing |

### The Critical Insight

> The #1 pattern in DS coding interviews: **"Can I solve this faster with a hash table?"** Dictionaries and sets give O(1) average lookup, turning O(n²) brute-force into O(n) solutions.

---

## Time Complexity — Know This Cold

| Operation | List | Dict | Set |
|-----------|------|------|-----|
| Access by index | O(1) | N/A | N/A |
| Search/`in` | **O(n)** | **O(1)** | **O(1)** |
| Insert at end | O(1) | O(1) | O(1) |
| Insert at beginning | **O(n)** | N/A | N/A |
| Delete by value | O(n) | O(1) | O(1) |
| Sort | O(n log n) | N/A | N/A |
| `len()` | O(1) | O(1) | O(1) |

> **Talking Point**: "When I see a problem that requires repeated lookups or membership checks, my instinct is to use a set or dict rather than scanning a list. That single choice often takes the solution from O(n²) to O(n)."

---

## Lists — Ordered, Mutable, Most Used

### Essential Operations

```python
nums = [3, 1, 4, 1, 5, 9]

# Slicing
nums[1:4]       # [1, 4, 1]
nums[::-1]      # [9, 5, 1, 4, 1, 3] (reverse)
nums[::2]       # [3, 4, 5] (every 2nd)

# Modification
nums.append(2)       # O(1) — add to end
nums.insert(0, 7)    # O(n) — add at position
nums.pop()           # O(1) — remove last
nums.pop(0)          # O(n) — remove first
nums.remove(4)       # O(n) — remove first occurrence

# sorted() vs .sort()
original = [3, 1, 4]
new_list = sorted(original)  # returns NEW list, original unchanged
original.sort()              # sorts IN-PLACE, returns None!
```

### List Comprehensions (Master these!)

```python
# Basic
squares = [x**2 for x in range(10)]

# With condition
evens = [x for x in range(20) if x % 2 == 0]

# With transform
labels = ["high" if x > 50 else "low" for x in scores]

# Nested (flatten)
flat = [num for row in matrix for num in row]
```

---

## Dictionaries — The Interview Workhorse

### Essential Operations

```python
user = {"name": "Anjali", "role": "DS", "yoe": 5}

# Access (safe!)
user.get("salary", 0)       # 0 if key missing (no KeyError!)
user.setdefault("dept", "Analytics")  # set only if missing

# Iteration
for key, value in user.items():
    print(f"{key}: {value}")

# Comprehension
word_lengths = {w: len(w) for w in ["hello", "world"]}

# Merging (Python 3.9+)
merged = dict1 | dict2  # dict2 values win on conflicts
```

### defaultdict — Auto-Initialize Missing Keys

```python
from collections import defaultdict

# Grouping pattern (most common in interviews!)
groups = defaultdict(list)
for item in data:
    groups[item["category"]].append(item)

# Counting pattern
counts = defaultdict(int)
for word in text.split():
    counts[word] += 1
```

---

## Sets — For Uniqueness and Fast Lookup

```python
a = {1, 2, 3, 4, 5}
b = {4, 5, 6, 7, 8}

a | b    # Union: {1, 2, 3, 4, 5, 6, 7, 8}
a & b    # Intersection: {4, 5}
a - b    # Difference: {1, 2, 3}
a ^ b    # Symmetric difference: {1, 2, 3, 6, 7, 8}

# Membership test — O(1)!
if user_id in active_users_set:  # FAST
    pass
```

### When to Use Set vs List for Lookups

```python
# BAD: O(n) lookup on each iteration = O(n*m) total
blocklist = ["spam", "bot", "fake"]
filtered = [u for u in users if u["type"] not in blocklist]

# GOOD: O(1) lookup = O(n) total
blocklist = {"spam", "bot", "fake"}  # set!
filtered = [u for u in users if u["type"] not in blocklist]
```

---

## Tuples — Immutable and Hashable

```python
point = (3, 4)

# Unpacking
x, y = point
first, *rest = [1, 2, 3, 4, 5]  # first=1, rest=[2,3,4,5]

# As dict keys (lists can't do this!)
cache = {(x, y): compute(x, y) for x, y in coordinates}

# Named tuples (readable)
from collections import namedtuple
Point = namedtuple('Point', ['x', 'y'])
p = Point(3, 4)
print(p.x, p.y)  # 3 4
```

---

## Collections Module — Power Tools

### Counter — Frequency Counting

```python
from collections import Counter

words = "the cat sat on the mat the cat".split()
counts = Counter(words)
counts.most_common(3)  # [('the', 3), ('cat', 2), ('sat', 1)]

# Set-like operations on Counters
counter_a = Counter("aabbc")
counter_b = Counter("abbcc")
counter_a & counter_b  # min of each: Counter({'a':1, 'b':2, 'c':1})
```

### OrderedDict, deque

```python
from collections import deque

# deque — O(1) operations on both ends (list is O(n) for left)
queue = deque()
queue.append("right")      # add right
queue.appendleft("left")   # add left — O(1)!
queue.popleft()            # remove left — O(1)!

# LRU Cache uses OrderedDict
from collections import OrderedDict
```

---

## Interview Talking Points & Scripts

### When asked "How would you find duplicates in a dataset?"

> *"My first thought is to use a set for O(1) lookups. I'd iterate through the data once, checking if each element is already in my 'seen' set. If yes, it's a duplicate. This gives O(n) time and O(n) space — much better than the O(n²) nested loop approach. For DataFrames specifically, I'd use pandas `duplicated()` method."*

### When asked "When would you use a tuple over a list?"

> *"Tuples when the data shouldn't change — like coordinates, database records, or function return values. They're also hashable, so they can serve as dictionary keys, which is useful for memoization. And since they're immutable, they're slightly more memory-efficient and signal intent to other developers."*

### When asked about hash tables

> *"Python's dict and set are hash tables underneath. They compute a hash of the key to determine the storage bucket, giving O(1) average lookup. Collisions are handled with open addressing. This is why only hashable (immutable) types can be dict keys — the hash must stay stable. In interviews, recognizing that a problem needs O(1) lookup is often the key insight that unlocks an efficient solution."*

---

## Common Interview Mistakes

| ❌ Mistake | Why It's Wrong | ✅ Better |
|-----------|---------------|----------|
| Using `list` for membership tests | O(n) per lookup | Use `set` for O(1) |
| `dict[key]` without checking | KeyError if missing | Use `dict.get(key, default)` |
| Modifying list while iterating | Skips elements or errors | Iterate over a copy or build new list |
| Forgetting `sorted()` returns new list | `.sort()` returns `None` | Know which one you need |
| Using list as dict key | TypeError: unhashable | Use tuple instead |
| Not knowing `defaultdict` | Verbose `if key in dict` checks | `defaultdict(list)` or `defaultdict(int)` |

---

## Classic Interview Problems

### Two Sum (Amazon, Google, Meta)

```python
def two_sum(nums, target):
    """O(n) time, O(n) space using hash map"""
    seen = {}
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i
    return []
```

### Group Anagrams (Meta)

```python
from collections import defaultdict

def group_anagrams(words):
    groups = defaultdict(list)
    for word in words:
        key = tuple(sorted(word))  # anagrams have same sorted form
        groups[key].append(word)
    return list(groups.values())
```

### First Non-Repeating Character

```python
from collections import Counter

def first_unique(s):
    counts = Counter(s)
    for i, char in enumerate(s):
        if counts[char] == 1:
            return i
    return -1
```

### Remove Duplicates Preserving Order

```python
def remove_duplicates(lst):
    seen = set()
    result = []
    for item in lst:
        if item not in seen:
            seen.add(item)
            result.append(item)
    return result

# One-liner (Python 3.7+ — dict preserves insertion order)
list(dict.fromkeys(lst))
```

---

## Summary Cheat Sheet

```
╔══════════════════════════════════════════════════════════════╗
║             DATA STRUCTURES CHEAT SHEET                      ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  LIST:  Ordered, mutable, O(1) append, O(n) search         ║
║  DICT:  Key-value, O(1) lookup, use .get() for safety      ║
║  SET:   Unique elements, O(1) membership, math operations   ║
║  TUPLE: Immutable, hashable, use as dict keys              ║
║                                                              ║
║  PATTERN: "Can I use a hash table?" = interview unlock      ║
║                                                              ║
║  TOOLS:  Counter, defaultdict, deque, namedtuple           ║
║                                                              ║
║  GOTCHAS: list for lookups (use set), mutable during iter  ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 2 of 25*
