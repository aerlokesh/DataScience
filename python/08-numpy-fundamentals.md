# 🎯 Topic 8: NumPy Fundamentals

> **Scope:** Core NumPy operations every Data Scientist must know — array creation, indexing,
> broadcasting, vectorization internals, linear algebra, and memory layout. Targeted at DS/DA
> candidates with ~5 YOE interviewing for big tech (FAANG/MAANG) roles.

---

## Table of Contents

1. [Array Creation & Data Types](#1-array-creation--data-types)
2. [Indexing & Slicing](#2-indexing--slicing)
3. [Broadcasting Rules](#3-broadcasting-rules)
4. [Vectorization vs Loops — Why NumPy Is Fast](#4-vectorization-vs-loops--why-numpy-is-fast)
5. [Memory Layout: C-Contiguous vs Fortran-Contiguous](#5-memory-layout-c-contiguous-vs-fortran-contiguous)
6. [Axis Operations](#6-axis-operations)
7. [Linear Algebra Essentials](#7-linear-algebra-essentials)
8. [Interview Talking Points & Scripts](#8-interview-talking-points--scripts)
9. [Common Interview Mistakes](#9-common-interview-mistakes)
10. [Rapid-Fire Q&A](#10-rapid-fire-qa)
11. [Summary Cheat Sheet](#11-summary-cheat-sheet)

---

## 1. Array Creation & Data Types

```python
import numpy as np

# From Python lists
a = np.array([1, 2, 3])                    # 1-D int64
b = np.array([[1.0, 2.0], [3.0, 4.0]])     # 2-D float64

# Factory functions
zeros = np.zeros((3, 4))                    # shape (3, 4), float64
ones  = np.ones((2, 3), dtype=np.int32)     # explicit dtype
eye   = np.eye(4)                           # 4x4 identity

# Ranges and linspace
seq   = np.arange(0, 10, 0.5)              # half-open interval
lin   = np.linspace(0, 1, 50)              # 50 points inclusive

# Random arrays (modern API)
rng = np.random.default_rng(seed=42)
normal_data = rng.normal(loc=0, scale=1, size=(1000,))
uniform_data = rng.uniform(0, 1, size=(5, 5))

# Structured arrays (useful for heterogeneous tabular data)
dt = np.dtype([('name', 'U10'), ('age', 'i4'), ('score', 'f8')])
records = np.array([('Alice', 30, 95.5), ('Bob', 25, 88.0)], dtype=dt)
```

> **Critical Insight:** NumPy arrays are *homogeneous* — every element shares the same dtype.
> This constraint enables contiguous memory allocation and SIMD optimization. When you pass a
> mixed list (ints and floats), NumPy *upcasts* to the broadest type. Understanding dtype
> promotion is essential for avoiding silent precision loss in production pipelines.

---

## 2. Indexing & Slicing

```python
arr = np.arange(24).reshape(4, 6)

# Basic slicing (returns VIEWS, not copies)
row_slice = arr[1:3, :]          # rows 1-2, all columns
col_slice = arr[:, ::2]          # all rows, every other column

# Integer (fancy) indexing (returns COPIES)
fancy = arr[[0, 2, 3], [1, 4, 5]]   # elements at (0,1), (2,4), (3,5)

# Boolean masking
mask = arr > 10
filtered = arr[mask]             # 1-D array of elements > 10

# Combining slicing and boolean indexing
arr[arr % 2 == 0] = 0           # zero out even numbers IN PLACE (view!)
```

> **Critical Insight:** The view vs copy distinction is the #1 source of bugs in NumPy code.
> Basic slicing creates *views* (shared memory). Fancy indexing and boolean indexing create
> *copies*. If you modify a view, the original array changes. Use `arr.copy()` explicitly
> when you need independent data. Interviewers test this regularly.

```python
# Demonstrating view vs copy
original = np.array([10, 20, 30, 40, 50])
view_slice = original[1:4]       # VIEW
view_slice[0] = 999
print(original)                  # [10, 999, 30, 40, 50] -- original mutated!

copy_slice = original[1:4].copy()
copy_slice[0] = 0
print(original)                  # [10, 999, 30, 40, 50] -- unchanged
```

---

## 3. Broadcasting Rules

Broadcasting allows NumPy to perform element-wise operations on arrays with *different* shapes
without creating intermediate copies.

### The Three Broadcasting Rules

1. If arrays differ in ndim, the shape of the smaller array is **padded with 1s on the left**.
2. Arrays with size 1 along a dimension act as if they had the size of the largest array in that dimension.
3. If sizes disagree and neither is 1, a **ValueError** is raised.

### ASCII Diagrams — Shape Alignment

```
Example 1: (3, 4) + (4,)
--------------------------------------------------
Step 1 — Pad on left:     (3, 4) + (1, 4)
Step 2 — Stretch dim 0:   (3, 4) + (3, 4)  ✓ compatible

    Array A (3,4)         Array B (4,) -> (1,4) -> (3,4)
    ┌─────────────┐       ┌─────────────┐
    │ a00 a01 a02 a03│       │ b0  b1  b2  b3 │  ← row repeated
    │ a10 a11 a12 a13│  +    │ b0  b1  b2  b3 │     3 times
    │ a20 a21 a22 a23│       │ b0  b1  b2  b3 │
    └─────────────┘       └─────────────┘


Example 2: (3, 1) + (1, 4)
--------------------------------------------------
Step 1 — Shapes already same ndim
Step 2 — Stretch:          (3, 4) + (3, 4)  ✓ compatible

    Array A (3,1)         Array B (1,4)
    ┌───┐                 ┌─────────────┐
    │ a0 │  stretched →   │ a0 a0 a0 a0 │
    │ a1 │                │ a1 a1 a1 a1 │       Result (3,4)
    │ a2 │                │ a2 a2 a2 a2 │
    └───┘                 └─────────────┘
                    +
                          │ b0 b1 b2 b3 │  stretched ↓
                          │ b0 b1 b2 b3 │
                          │ b0 b1 b2 b3 │


Example 3: (3,) + (4,)  → ERROR
--------------------------------------------------
Neither dimension is 1 and they don't match.
    (3,) vs (4,)  →  ValueError!
```

```python
# Practical broadcasting: subtract column means
data = rng.normal(0, 1, size=(100, 5))
col_means = data.mean(axis=0)             # shape (5,)
centered = data - col_means               # (100,5) - (5,) broadcasts to (100,5)

# Outer product via broadcasting
x = np.arange(5).reshape(5, 1)            # (5, 1)
y = np.arange(3).reshape(1, 3)            # (1, 3)
outer = x * y                             # (5, 3)
```

---

## 4. Vectorization vs Loops — Why NumPy Is Fast

### Why Vectorized Code Wins

| Factor | Python Loop | NumPy Vectorized |
|--------|-------------|------------------|
| **Type checking** | Every iteration checks types | Single check at call boundary |
| **Memory access** | Objects scattered in heap | Contiguous buffer (cache-friendly) |
| **SIMD instructions** | Not available | CPU applies 4-8 ops per cycle (SSE/AVX) |
| **Interpreter overhead** | Bytecode dispatch per element | Single C-level dispatch for entire array |
| **GIL pressure** | Held during Python execution | Released inside NumPy C extensions |

### Performance Comparison Table

| Operation | Python Loop (ms) | NumPy Vectorized (ms) | Speedup |
|-----------|------------------|-----------------------|---------|
| Sum 1M elements | ~120 | ~0.5 | ~240x |
| Element-wise multiply 1M | ~180 | ~0.8 | ~225x |
| Dot product 10K x 10K | ~45,000 | ~50 (BLAS) | ~900x |
| Boolean mask filter 1M | ~250 | ~3 | ~83x |
| Cumulative sum 1M | ~150 | ~1.2 | ~125x |

> **Critical Insight:** The speedup is NOT just "C vs Python." It comes from three synergistic
> factors: (1) elimination of per-element type dispatch, (2) spatial locality enabling CPU
> cache prefetching on contiguous memory, and (3) SIMD parallelism (one instruction processes
> multiple data elements). When an interviewer asks "why is NumPy fast?", hit all three.

```python
import time

n = 1_000_000
data = rng.random(n)

# Python loop
start = time.perf_counter()
total = 0
for x in data:
    total += x
loop_time = time.perf_counter() - start

# Vectorized
start = time.perf_counter()
total_np = data.sum()
vec_time = time.perf_counter() - start

print(f"Loop: {loop_time*1000:.1f} ms | Vectorized: {vec_time*1000:.3f} ms")
print(f"Speedup: {loop_time/vec_time:.0f}x")
```

---

## 5. Memory Layout: C-Contiguous vs Fortran-Contiguous

### Comparison Table

| Property | C-Order (Row-Major) | Fortran-Order (Column-Major) |
|----------|---------------------|------------------------------|
| **Storage** | Last axis varies fastest | First axis varies fastest |
| **Default in NumPy** | Yes (`order='C'`) | No (must specify `order='F'`) |
| **Row operations** | Fast (contiguous in memory) | Slow (strided access) |
| **Column operations** | Slow (strided access) | Fast (contiguous in memory) |
| **Common in** | C, C++, Python, most languages | Fortran, MATLAB, R |
| **Cache performance** | Best for row-wise iteration | Best for column-wise iteration |

```
Memory layout for a 3x4 array:

C-order (row-major):
Memory: [a00, a01, a02, a03, a10, a11, a12, a13, a20, a21, a22, a23]
         ← row 0 →         ← row 1 →         ← row 2 →

F-order (column-major):
Memory: [a00, a10, a20, a01, a11, a21, a02, a12, a22, a03, a13, a23]
         ← col 0 → ← col 1 →  ← col 2 →  ← col 3 →
```

```python
# Checking and converting order
arr_c = np.zeros((1000, 1000), order='C')
arr_f = np.zeros((1000, 1000), order='F')

print(arr_c.flags['C_CONTIGUOUS'])   # True
print(arr_f.flags['F_CONTIGUOUS'])   # True

# Performance impact: summing rows vs columns
arr = rng.random((5000, 5000))
arr_f = np.asfortranarray(arr)

# Row sum is faster on C-order
%timeit arr.sum(axis=1)    # C-order: fast
%timeit arr_f.sum(axis=1)  # F-order: slower

# Column sum is faster on F-order
%timeit arr.sum(axis=0)    # C-order: slower
%timeit arr_f.sum(axis=0)  # F-order: fast
```

> **Critical Insight:** When interfacing with Fortran-based LAPACK/BLAS routines (via
> `scipy.linalg`), converting to F-order *before* repeated linear algebra calls avoids
> internal transpositions. In ML pipelines where you repeatedly access features (columns),
> F-order can yield measurable speedups. Know this nuance — it separates senior from junior.

---

## 6. Axis Operations

```python
arr = np.arange(24).reshape(2, 3, 4)  # shape: (2, 3, 4)

# axis=0 collapses first dimension  → result shape (3, 4)
# axis=1 collapses second dimension → result shape (2, 4)
# axis=2 collapses third dimension  → result shape (2, 3)

print(arr.sum(axis=0).shape)   # (3, 4) — sum across "layers"
print(arr.sum(axis=1).shape)   # (2, 4) — sum across "rows"
print(arr.sum(axis=2).shape)   # (2, 3) — sum across "columns"

# keepdims preserves shape for broadcasting
col_means = arr.mean(axis=2, keepdims=True)  # shape (2, 3, 1)
centered = arr - col_means                    # broadcasts correctly
```

> **Critical Insight:** Think of `axis=k` as "collapse dimension k." The resulting array
> has shape with dimension k removed. Use `keepdims=True` when the result must broadcast
> back against the original array — this avoids shape mismatch errors in pipelines.

```python
# Common axis patterns in data science
data = rng.random((100, 5))   # 100 samples, 5 features

# Per-feature statistics (collapse samples axis)
feature_means = data.mean(axis=0)     # shape (5,)
feature_stds  = data.std(axis=0)      # shape (5,)

# Per-sample statistics (collapse features axis)
sample_norms = np.linalg.norm(data, axis=1)  # shape (100,)

# Standardization (z-score)
standardized = (data - feature_means) / feature_stds  # broadcasting!
```

---

## 7. Linear Algebra Essentials

```python
# Dot product / matrix multiplication
A = rng.random((3, 4))
B = rng.random((4, 5))
C = A @ B                         # (3, 5) — preferred operator (Python 3.5+)
C_alt = np.dot(A, B)             # equivalent
C_alt2 = np.matmul(A, B)        # equivalent

# Vector dot product
v1 = np.array([1, 2, 3])
v2 = np.array([4, 5, 6])
dot = v1 @ v2                    # scalar: 32

# Norms
from numpy.linalg import norm
l1 = norm(v1, ord=1)             # Manhattan: |1|+|2|+|3| = 6
l2 = norm(v1, ord=2)             # Euclidean: sqrt(1+4+9) = 3.742
linf = norm(v1, ord=np.inf)      # Max absolute: 3
fro = norm(A, ord='fro')         # Frobenius norm for matrices

# Singular Value Decomposition (SVD)
X = rng.random((100, 5))
U, s, Vt = np.linalg.svd(X, full_matrices=False)
# U: (100, 5), s: (5,), Vt: (5, 5)
# Reconstruct: X ≈ U @ np.diag(s) @ Vt

# Explained variance ratio (like PCA)
explained_var = (s ** 2) / np.sum(s ** 2)
cumulative_var = np.cumsum(explained_var)

# Eigendecomposition (symmetric matrices)
cov = X.T @ X / len(X)           # (5, 5) covariance
eigenvalues, eigenvectors = np.linalg.eigh(cov)  # eigh for symmetric

# Solving linear systems: Ax = b
A_sq = rng.random((5, 5))
b = rng.random(5)
x = np.linalg.solve(A_sq, b)    # More numerically stable than inv(A) @ b
```

> **Critical Insight:** Never compute `np.linalg.inv(A) @ b` to solve linear systems.
> Use `np.linalg.solve(A, b)` instead — it is 2x faster and numerically more stable
> (uses LU decomposition internally). Similarly, prefer SVD over eigendecomposition for
> non-square matrices. Interviewers love testing whether you know *when* to use each tool.

---

## 8. Interview Talking Points & Scripts

### On "Why not just use Python loops?"

> *"In production data pipelines, Python loops over numerical data are 100-1000x slower than
> vectorized NumPy operations. This isn't just a language-speed issue — it's architectural.
> NumPy stores data in contiguous memory buffers, enabling CPU cache prefetching and SIMD
> instructions that process 4-8 elements per clock cycle. Python loops, by contrast, require
> per-element type dispatch, boxing/unboxing of numeric objects, and scattered heap access
> that defeats the cache hierarchy. When I'm writing feature engineering code, I think in
> terms of array operations first and only fall back to loops for genuinely sequential logic
> like state machines."*

### On Broadcasting

> *"Broadcasting is NumPy's mechanism for performing element-wise operations on differently-
> shaped arrays without materializing intermediate copies. The rules are simple: dimensions
> are compared from right to left, and sizes must either match or one must be 1. When I
> center a dataset by subtracting column means, broadcasting handles the shape mismatch
> between (n_samples, n_features) and (n_features,) automatically. This is memory-efficient
> because no (n_samples, n_features)-sized temporary is created — the operation is applied
> element-by-element using the broadcasted view."*

### On Memory Layout

> *"NumPy defaults to C-contiguous (row-major) ordering, where consecutive elements of a row
> sit adjacent in memory. This matters for performance: iterating along the contiguous axis
> gets ~5-10x better cache utilization than strided access. In my experience, this becomes
> critical in two scenarios: (1) when interfacing with Fortran-based LAPACK routines where
> converting to F-order avoids internal copies, and (2) when designing feature matrices —
> since sklearn and pandas store data row-major, row-wise operations like sample-level
> normalization are inherently cache-friendly."*

### On Views vs Copies

> *"One of the most subtle aspects of NumPy is that basic slicing creates views — shared
> memory windows into the original array. This is a deliberate design choice for efficiency:
> slicing a 10GB array doesn't allocate new memory. But it means mutations propagate. In
> production code, I explicitly call .copy() when I need isolation, and I use
> np.shares_memory() to verify during debugging. Fancy indexing and boolean masks always
> return copies, which is why filtering operations are safe to mutate."*

---

## 9. Common Interview Mistakes

| # | Mistake | Why It's Wrong | Correct Approach |
|---|---------|----------------|------------------|
| 1 | Confusing `axis=0` with "operate on rows" | `axis=0` collapses the row dimension, giving per-column results | Think "axis=k removes dimension k from the output shape" |
| 2 | Modifying a slice without realizing it's a view | Mutates the original array silently | Use `.copy()` when you need independence; check with `np.shares_memory()` |
| 3 | Using `np.linalg.inv(A) @ b` to solve systems | Numerically unstable and 2x slower | Use `np.linalg.solve(A, b)` which uses LU decomposition |
| 4 | Forgetting `keepdims=True` before broadcasting back | Shape mismatch errors in normalization code | Always use `keepdims=True` when the result will be broadcast |
| 5 | Writing Python loops for element-wise operations | 100-1000x slower; shows lack of NumPy fluency | Reformulate as vectorized array expressions |
| 6 | Ignoring dtype — mixing int32 and float64 carelessly | Silent precision loss or unexpected upcasting | Specify dtype explicitly; check with `.dtype` |
| 7 | Using `==` to compare float arrays | Floating point equality fails silently | Use `np.allclose(a, b, atol=1e-8)` or `np.isclose()` |
| 8 | Reshaping without understanding C vs F order | Data reordered unexpectedly | Specify `order='C'` or `order='F'` explicitly; verify with print |
| 9 | Assuming `.flatten()` and `.ravel()` are identical | `.flatten()` always copies; `.ravel()` returns view when possible | Use `.ravel()` for efficiency; `.flatten()` when copy is needed |
| 10 | Broadcasting errors from misaligned shapes | Shapes (3,) and (4,) cannot broadcast | Reshape explicitly: `arr.reshape(-1, 1)` to add axis |

---

## 10. Rapid-Fire Q&A

**Q1: What is the difference between `np.array` and `np.asarray`?**
`np.array` always creates a new array (copy). `np.asarray` returns a view if the input is
already an ndarray with compatible dtype — no copy overhead.

**Q2: How does NumPy achieve parallelism without threads?**
Through SIMD vectorization (SSE/AVX instructions process 4-8 floats per cycle) and by linking
to optimized BLAS libraries (OpenBLAS, MKL) that use multiple threads internally.

**Q3: What does `arr.strides` tell you?**
The number of bytes to step in each dimension. For a (3,4) float64 C-order array:
strides = (32, 8) — 8 bytes per element in columns, 4*8=32 bytes to jump one row.

**Q4: When would you use `np.einsum`?**
For complex tensor contractions that are awkward to express with `@` or `np.tensordot`.
Example: batch matrix multiply `np.einsum('bij,bjk->bik', A, B)`.

**Q5: How do you add a dimension to an array?**
Three ways: `arr[np.newaxis, :]`, `arr[:, None]`, or `np.expand_dims(arr, axis=0)`.

**Q6: What is the difference between `np.concatenate` and `np.stack`?**
`concatenate` joins along an existing axis. `stack` creates a *new* axis and joins along it.
`np.vstack`/`np.hstack` are convenience wrappers.

**Q7: Why does `np.linalg.eigh` exist separately from `np.linalg.eig`?**
`eigh` exploits symmetry for 2x speed and guaranteed real eigenvalues. Always use it for
covariance matrices and Gram matrices.

**Q8: How do you handle NaN values in NumPy aggregations?**
Use `np.nanmean`, `np.nanstd`, `np.nansum` — they ignore NaN entries. Regular `np.mean`
propagates NaN (returns NaN if any element is NaN).

**Q9: What is structured array and when is it preferable to a DataFrame?**
A structured array stores heterogeneous fields in a single contiguous buffer. Preferable
when you need zero-copy memory mapping of binary files or C-struct interop.

**Q10: How does `np.where` work?**
With three args: `np.where(condition, x, y)` returns elements from `x` where condition is
True, else from `y`. With one arg: returns indices where condition is True (like `np.nonzero`).

**Q11: What is the memory overhead of a NumPy array vs a Python list?**
A Python list of 1M floats uses ~28 MB (each float is a 28-byte object + 8-byte pointer).
A NumPy float64 array uses ~8 MB (just the raw buffer + ~100 bytes of metadata).

**Q12: How does `np.vectorize` differ from true vectorization?**
`np.vectorize` is syntactic sugar — it still loops in Python. It provides broadcasting
support but NOT the speed of true NumPy ufuncs. Never use it for performance.

---

## 11. Summary Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════════╗
║                    NUMPY FUNDAMENTALS CHEAT SHEET                       ║
╠══════════════════════════════════════════════════════════════════════════╣
║                                                                          ║
║  CREATION:   np.array, zeros, ones, eye, arange, linspace, random       ║
║  INDEXING:   slice=VIEW | fancy/bool=COPY | use .copy() for safety      ║
║  BROADCAST:  pad 1s on left → stretch size-1 dims → error if mismatch  ║
║  SPEED:      vectorize > loop (100-1000x) due to SIMD + cache + no GIL ║
║  MEMORY:     C-order=row-major (default) | F-order=col-major            ║
║  AXIS:       axis=k removes dim k | keepdims=True for broadcast back    ║
║  LINALG:     @ for matmul | solve() not inv() | SVD for non-square      ║
║                                                                          ║
║  KEY RULES:                                                              ║
║  • Never use python loops for element-wise array ops                     ║
║  • Always check view vs copy when mutating slices                        ║
║  • Use np.allclose() not == for float comparison                         ║
║  • Specify dtype explicitly in production code                           ║
║  • Use np.nan* functions when data may contain NaN                       ║
║                                                                          ║
╚══════════════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 8 of 25*
