# 🎯 Topic 21: Coding Patterns & Algorithms

> **Data Science Interview — Coding Patterns & Algorithms Deep Dive**  
> The essential algorithmic patterns every DS/DA must know for big tech interviews — Big-O analysis, sliding windows, hash maps, binary search, dynamic programming, and DS-specific problems like moving averages, Top-K elements, and anomaly detection in sequences.

---

## Table of Contents

1. [Big-O Notation & Complexity Analysis](#big-o-notation--complexity-analysis)
2. [Two Pointers Technique](#two-pointers-technique)
3. [Sliding Window Pattern](#sliding-window-pattern)
4. [Hash Maps for O(1) Lookups](#hash-maps-for-o1-lookups)
5. [Binary Search — Standard & Variations](#binary-search--standard--variations)
6. [Sorting Algorithms & When to Use Which](#sorting-algorithms--when-to-use-which)
7. [Stacks & Queues — Applications](#stacks--queues--applications)
8. [Recursion Fundamentals](#recursion-fundamentals)
9. [Dynamic Programming Basics](#dynamic-programming-basics)
10. [DS-Specific Problems & Solutions](#ds-specific-problems--solutions)
11. [Interview Talking Points & Scripts](#interview-talking-points--scripts)
12. [Common Interview Mistakes](#common-interview-mistakes)
13. [Rapid-Fire Q&A](#rapid-fire-qa)
14. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## Big-O Notation & Complexity Analysis

> **Critical Insight:** Interviewers don't want you to just say "O(n)." They want you to explain **why** — identify the dominant operation, show you understand constants drop out, and reason about best/average/worst cases.

| Big-O | Name | Example | n=1M Runtime* |
|-------|------|---------|---------------|
| O(1) | Constant | Hash lookup, array index | ~1 ns |
| O(log n) | Logarithmic | Binary search | ~20 ns |
| O(n) | Linear | Single pass over array | ~1 ms |
| O(n log n) | Linearithmic | Merge sort, Tim sort | ~20 ms |
| O(n^2) | Quadratic | Nested loops, bubble sort | ~16 min |
| O(2^n) | Exponential | Recursive Fibonacci (naive) | Heat death of universe |

### Big-O for Common Data Structure Operations

| Operation | Array | Hash Map | BST (balanced) | Heap |
|-----------|-------|----------|----------------|------|
| Search | O(n) | O(1) avg | O(log n) | O(n) |
| Insert | O(n) | O(1) avg | O(log n) | O(log n) |
| Delete | O(n) | O(1) avg | O(log n) | O(log n) |
| Min/Max | O(n) | O(n) | O(log n) | O(1) |

---

## Two Pointers Technique

> **Critical Insight:** Two pointers reduce O(n^2) brute-force nested loops to O(n) by exploiting sorted order or opposite-end convergence.

### Problem 1: Pair Sum in Sorted Array — O(n) time, O(1) space

```python
def two_sum_sorted(nums, target):
    left, right = 0, len(nums) - 1
    while left < right:
        s = nums[left] + nums[right]
        if s == target: return [left, right]
        elif s < target: left += 1
        else: right -= 1
    return []
```
### Problem 2: Merge Two Sorted Arrays — O(n+m) time, O(n+m) space

```python
def merge_sorted(a, b):
    result, i, j = [], 0, 0
    while i < len(a) and j < len(b):
        if a[i] <= b[j]: result.append(a[i]); i += 1
        else: result.append(b[j]); j += 1
    result.extend(a[i:])
    result.extend(b[j:])
    return result
```
### Problem 3: Remove Duplicates In-Place — O(n) time, O(1) space

```python
def remove_duplicates(nums):
    if not nums: return 0
    write = 1
    for read in range(1, len(nums)):
        if nums[read] != nums[read - 1]:
            nums[write] = nums[read]
            write += 1
    return write
```

---

## Sliding Window Pattern

> **Critical Insight:** Use sliding window when the problem asks about contiguous subarrays/substrings with some constraint. Fixed-size windows slide; variable-size windows expand right and shrink left.

### Problem 4: Moving Average (DS Classic) — O(1) per call

```python
from collections import deque

class MovingAverage:
    def __init__(self, size):
        self.window, self.total, self.size = deque(), 0, size

    def next(self, val):
        self.window.append(val)
        self.total += val
        if len(self.window) > self.size:
            self.total -= self.window.popleft()
        return self.total / len(self.window)
```
### Problem 5: Maximum Sum Subarray of Size K — O(n) time, O(1) space

```python
def max_sum_subarray(nums, k):
    window_sum = sum(nums[:k])
    max_sum = window_sum
    for i in range(k, len(nums)):
        window_sum += nums[i] - nums[i - k]
        max_sum = max(max_sum, window_sum)
    return max_sum
```
### Problem 6: Longest Substring Without Repeats — O(n) time

```python
def longest_unique_substring(s):
    char_index, max_len, left = {}, 0, 0
    for right, char in enumerate(s):
        if char in char_index and char_index[char] >= left:
            left = char_index[char] + 1
        char_index[char] = right
        max_len = max(max_len, right - left + 1)
    return max_len
```

---

## Hash Maps for O(1) Lookups

> **Critical Insight:** When you see "find if X exists," "count occurrences," or "group by" — think hash map first. It turns O(n) nested searches into O(1) lookups.

### Problem 7: Two Sum (Unsorted — The Classic) — O(n) time, O(n) space

```python
def two_sum(nums, target):
    seen = {}
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen: return [seen[complement], i]
        seen[num] = i
    return []
```
### Problem 8: Frequency Counter — O(n) time

```python
from collections import Counter

def find_duplicates(data):
    return [item for item, count in Counter(data).items() if count > 1]

def first_unique(data):
    counts = Counter(data)
    return next((x for x in data if counts[x] == 1), None)
```
### Problem 9: Group Anagrams — O(n * k log k) time

```python
from collections import defaultdict
def group_anagrams(words):
    groups = defaultdict(list)
    for word in words:
        groups[tuple(sorted(word))].append(word)
    return list(groups.values())
```

---

## Binary Search — Standard & Variations

> **Critical Insight:** Binary search applies whenever you can define a condition that partitions the space into two halves. This includes searching for thresholds, insertion points, and optimal values — not just finding elements.

### Problem 10: Standard Binary Search — O(log n) time, O(1) space

```python
def binary_search(nums, target):
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = left + (right - left) // 2
        if nums[mid] == target: return mid
        elif nums[mid] < target: left = mid + 1
        else: right = mid - 1
    return -1
```
### Problem 11: Find Percentile Rank — O(log n)

```python
def bisect_left(nums, target):
    left, right = 0, len(nums)
    while left < right:
        mid = (left + right) // 2
        if nums[mid] < target: left = mid + 1
        else: right = mid
    return left

def percentile_rank(sorted_data, value):
    pos = bisect_left(sorted_data, value)
    return (pos / len(sorted_data)) * 100
```

---

## Sorting Algorithms & When to Use Which

| Algorithm | Time (Avg) | Space | Stable? | Best Use Case |
|-----------|-----------|-------|---------|---------------|
| Tim Sort (Python) | O(n log n) | O(n) | Yes | General — always use `sorted()` |
| Quick Sort | O(n log n) | O(log n) | No | In-memory, cache-friendly |
| Merge Sort | O(n log n) | O(n) | Yes | External sort, linked lists |
| Counting Sort | O(n + k) | O(k) | Yes | Small integer range |

> **Critical Insight:** In Python you almost never implement sorting — you use `sorted()`. The value is knowing **when sorting helps** and understanding stable sort for multi-key ordering.

```python
# Multi-key stable sort: sort by secondary key first, then primary
records = sorted(records, key=lambda x: x["department"])
records = sorted(records, key=lambda x: x["salary"], reverse=True)
```

---

## Stacks & Queues — Applications

| Structure | Key Operations | Use Case |
|-----------|---------------|----------|
| Stack (LIFO) | push/pop O(1) | Undo, expression parsing, DFS |
| Queue (FIFO) | enqueue/dequeue O(1) | BFS, task scheduling, buffering |
| Priority Queue | insert/extract O(log n) | Top-K, scheduling, Dijkstra |

### Problem 13: Valid Parentheses — O(n) time, O(n) space

```python
def is_valid_parens(s):
    stack, pairs = [], {')': '(', ']': '[', '}': '{'}
    for char in s:
        if char in '([{': stack.append(char)
        elif not stack or stack.pop() != pairs[char]: return False
    return len(stack) == 0
```
### Problem 14: Top-K Frequent Elements — O(n log k) time

```python
import heapq
from collections import Counter

def top_k_frequent(nums, k):
    counts = Counter(nums)
    return heapq.nlargest(k, counts.keys(), key=counts.get)
```

---

## Recursion Fundamentals

> **Critical Insight:** Every recursive solution needs a **base case**, **recursive case**, and **progress toward base case**. Python's default recursion limit is 1000 — always mention iterative alternatives for large inputs.

### Problem 15: Flatten Nested List — O(n) time

```python
def flatten(nested):
    result = []
    for item in nested:
        if isinstance(item, list): result.extend(flatten(item))
        else: result.append(item)
    return result
```

---

## Dynamic Programming Basics

> **Critical Insight:** DP is "smart recursion." If your recursive solution recalculates the same subproblems, optimize with memoization (top-down cache) or tabulation (bottom-up array).

| Approach | Direction | Pro | Con |
|----------|-----------|-----|-----|
| Memoization | Top-down | Natural from recursion | Stack depth limit |
| Tabulation | Bottom-up | No stack overhead | Must define iteration order |

### Problem 16: Fibonacci — O(n) time, O(1) space (tabulation)

```python
def fib(n):
    if n <= 1: return n
    prev, curr = 0, 1
    for _ in range(2, n + 1):
        prev, curr = curr, prev + curr
    return curr
```
### Problem 17: Maximum Subarray (Kadane's) — O(n) time, O(1) space

```python
def max_subarray(nums):
    max_sum = current_sum = nums[0]
    for num in nums[1:]:
        current_sum = max(num, current_sum + num)
        max_sum = max(max_sum, current_sum)
    return max_sum
```
### Problem 18: Coin Change — O(amount * coins) time, O(amount) space

```python
def coin_change(coins, amount):
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0
    for i in range(1, amount + 1):
        for coin in coins:
            if coin <= i: dp[i] = min(dp[i], dp[i - coin] + 1)
    return dp[amount] if dp[amount] != float('inf') else -1
```

---

## DS-Specific Problems & Solutions

### Problem 19: Running Median — O(log n) insert, O(1) query

```python
import heapq

class RunningMedian:
    """Two-heap approach: max-heap (low) and min-heap (high)."""
    def __init__(self):
        self.low, self.high = [], []  # low is negated for max-heap

    def add(self, num):
        heapq.heappush(self.low, -num)
        heapq.heappush(self.high, -heapq.heappop(self.low))
        if len(self.high) > len(self.low):
            heapq.heappush(self.low, -heapq.heappop(self.high))

    def median(self):
        if len(self.low) > len(self.high): return -self.low[0]
        return (-self.low[0] + self.high[0]) / 2
```
### Problem 20: Anomaly Detection (Rolling Z-Score) — O(1) per observation

```python
from collections import deque
import math

class StreamAnomalyDetector:
    def __init__(self, window_size=50, threshold=3.0):
        self.window, self.size, self.threshold = deque(), window_size, threshold
        self.sum = self.sq_sum = 0

    def is_anomaly(self, value):
        if len(self.window) < self.size:
            self._add(value)
            return False
        mean = self.sum / self.size
        std = math.sqrt(max(self.sq_sum / self.size - mean**2, 1e-10))
        z = abs(value - mean) / std
        self._add(value)
        if len(self.window) > self.size:
            old = self.window.popleft()
            self.sum -= old
            self.sq_sum -= old ** 2
        return z > self.threshold

    def _add(self, v):
        self.window.append(v)
        self.sum += v
        self.sq_sum += v ** 2
```
### Problem 21: Merge K Sorted Streams — O(n log k) time, O(k) space

```python
def merge_k_sorted(streams):
    heap = []
    for i, s in enumerate(streams):
        it = iter(s)
        val = next(it, None)
        if val is not None: heapq.heappush(heap, (val, i, it))
    while heap:
        val, idx, it = heapq.heappop(heap)
        yield val
        nxt = next(it, None)
        if nxt is not None: heapq.heappush(heap, (nxt, idx, it))
```
### Problem 22: EMA & Deduplication — O(n) time

```python
def compute_ema(values, alpha=0.3):
    """Exponential Moving Average. Higher alpha = more responsive."""
    ema = [values[0]]
    for v in values[1:]:
        ema.append(alpha * v + (1 - alpha) * ema[-1])
    return ema

def deduplicate_sorted(sorted_iter):
    """Yield unique elements from sorted stream. O(1) extra space."""
    prev = None
    for item in sorted_iter:
        if item != prev:
            yield item
            prev = item
```

---

## Interview Talking Points & Scripts

### "Walk me through your approach to this problem"

> *"I start by clarifying constraints — input size, sorted?, edge cases. Then I identify which pattern applies: sorted input suggests binary search or two pointers; tracking seen elements means hash map; contiguous subarray constraint means sliding window. I state the brute-force baseline, then explain how the pattern reduces complexity. I always declare time and space complexity before coding."*

### "How do you decide which data structure to use?"

> *"I match the structure to the dominant operations. O(1) lookups by key means hash map. Sorted order with efficient insert means heap or BST. FIFO means queue. Nested matching means stack. For streaming with fixed windows, a deque gives O(1) on both ends. The principle: define what operations you need most, pick the structure that makes those cheapest."*

### "Why this approach over alternatives?"

> *"I considered brute-force O(n^2) but recognized the sliding window pattern applies — contiguous subarrays with a constraint. The window maintains a running computation and updates incrementally, turning quadratic into linear. The trade-off is slightly more complex code, but the performance gain is essential at scale."*

---

## Common Interview Mistakes

| # | Mistake ❌ | Correction ✅ |
|---|-----------|--------------|
| 1 | Jumping to code without clarifying | Ask: "Input size? Sorted? Duplicates?" |
| 2 | O(n^2) when O(n) exists | Hash map for Two Sum, not nested loop |
| 3 | Forgetting edge cases | Handle empty input, single element, all-same |
| 4 | Off-by-one in binary search | Template: `left <= right` with `mid +/- 1` |
| 5 | Ignoring space complexity | Always state both: "O(n) time, O(1) space" |
| 6 | Modifying input without asking | Ask: "Can I sort in-place or preserve input?" |
| 7 | Not recognizing DP patterns | If you recalculate subproblems, add memoization |
| 8 | Recursion without base case guard | Define base case first, ensure progress toward it |
| 9 | Wrong window type | Fixed = known size K; Variable = condition-based |
| 10 | Not dry-running with examples | Trace through with [1,2,3] and edge case [] |

---

## Rapid-Fire Q&A

**Q1: Time of `in` for list vs. set?** List O(n), Set O(1) avg. Convert to set for repeated checks.

**Q2: When deque over list?** When you need O(1) on both ends. `list.pop(0)` is O(n); `deque.popleft()` is O(1).

**Q3: Memoization vs. tabulation?** Memo = top-down + cache. Tab = bottom-up + array. Tab avoids stack depth issues.

**Q4: Binary search on unsorted data?** Not for finding elements, but works on any monotonic function.

**Q5: Best approach for Top-K from a stream?** Min-heap of size K. O(n log k) time, O(k) space.

**Q6: What is Kadane's algorithm?** O(n) max subarray sum. Track current_sum (extend or restart) and global max.

**Q7: When is sorting overkill?** For k-th element use quickselect O(n). Bounded range: counting sort O(n+k).

**Q8: Sliding window vs. two pointers?** Window = contiguous range with constraint. Pointers can converge or fast/slow.

**Q9: Space of recursive DFS on tree?** O(h) where h = height. Balanced: O(log n). Skewed: O(n).

**Q10: Running stats without storing all data?** Welford's: track count, running mean, M2. O(1) per update.

**Q11: "Find subarrays with sum = K" pattern?** Prefix sum + hash map. Check if `prefix_sum - K` seen. O(n).

**Q12: Python's sorted() stability matters when?** Multi-key sort. Secondary key first, primary second. Equal elements keep order.

---

## Summary Cheat Sheet

```
+----------------------------------------------------------------------+
|            CODING PATTERNS & ALGORITHMS — CHEAT SHEET                 |
+----------------------------------------------------------------------+
| PATTERN SELECTION:                                                    |
|   Sorted + find pair  -> Two Pointers    | Contiguous sub -> Window  |
|   Find/count/group    -> Hash Map        | Sorted + find  -> BinSrch |
|   Top-K / stream min  -> Heap            | Nested pairs   -> Stack   |
|   Overlapping subprob -> DP              | O(n^2) to O(n) -> HashMap |
+----------------------------------------------------------------------+
| PYTHON BUILT-INS: sorted(), heapq.nlargest, bisect.bisect_left,     |
|   Counter, deque, defaultdict, lru_cache, itertools.groupby          |
+----------------------------------------------------------------------+
| DS PATTERNS:                                                         |
|   Moving Average -> Window+deque   | Running Median -> Two Heaps    |
|   Top-K -> Min-heap(size K)        | Anomaly -> Rolling Z-score     |
|   Merge Streams -> K-way heap      | Percentile -> Binary search    |
+----------------------------------------------------------------------+
| INTERVIEW FLOW:                                                      |
|   1. Clarify  2. Pattern  3. Complexity  4. Code  5. Test  6. Edge  |
+----------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 21 of 25*
