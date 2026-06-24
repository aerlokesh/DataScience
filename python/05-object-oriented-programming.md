# 🎯 Topic 5: Object-Oriented Programming for Data Science

> **Data Science Interview — OOP Deep Dive**  
> Classes, inheritance, polymorphism, decorators, dunder methods, dataclasses, and ABCs — focused on how OOP actually shows up in DS pipelines, custom transformers, and production ML code. Targeted at 5 YOE candidates interviewing at Google, Meta, Amazon, Netflix, Spotify, and Uber.

---

## Table of Contents

1. [Why OOP Matters in Data Science](#why-oop-matters-in-data-science)
2. [Classes & Core Mechanics](#classes--core-mechanics)
3. [Decorators: @property, @classmethod, @staticmethod](#decorators-property-classmethod-staticmethod)
4. [Dunder Methods That Matter](#dunder-methods-that-matter)
5. [Inheritance & Polymorphism](#inheritance--polymorphism)
6. [Abstract Base Classes (ABC)](#abstract-base-classes-abc)
7. [Dataclasses](#dataclasses)
8. [Composition vs Inheritance](#composition-vs-inheritance)
9. [Practical DS Example: Custom Sklearn Transformer](#practical-ds-example-custom-sklearn-transformer)
10. [Interview Talking Points & Scripts](#interview-talking-points--scripts)
11. [Common Interview Mistakes](#common-interview-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## Why OOP Matters in Data Science

OOP is not just software engineering ceremony — it solves real DS problems:

- **ML Pipelines**: sklearn's entire API is built on OOP (`.fit()`, `.transform()`, `.predict()`)
- **Custom Transformers**: Feature engineering steps that plug into `Pipeline`
- **Experiment Tracking**: Config objects that encapsulate hyperparameters
- **Data Loaders**: Iteratable dataset classes (PyTorch `Dataset`)
- **Production ML**: Model serving classes with consistent interfaces

> **Critical Insight**: At Meta/Google-level interviews, you're expected to write classes that fit into existing frameworks — not just standalone scripts. Knowing *when* to reach for OOP vs a simple function is the signal they look for.

---

## Classes & Core Mechanics

```python
class Model:
    """A simple model wrapper demonstrating core OOP."""
    
    version = "1.0"  # Class attribute — shared across ALL instances
    
    def __init__(self, name: str, hyperparams: dict):
        self.name = name                # Instance attribute
        self._hyperparams = hyperparams # Convention: "private"
        self._is_fitted = False
    
    def fit(self, X, y):
        # Training logic here
        self._is_fitted = True
        return self
    
    def predict(self, X):
        if not self._is_fitted:
            raise RuntimeError("Model must be fitted before prediction")
        # Prediction logic
        return X  # placeholder
```

> **Critical Insight**: Python has no true private attributes. A single underscore `_attr` is a convention meaning "internal use." Double underscore `__attr` triggers name mangling (`_ClassName__attr`) — use it to avoid accidental collisions in subclasses, not for "security."

---

## Decorators: @property, @classmethod, @staticmethod

| Feature | Receives | Use Case | Can Modify Instance? | Can Modify Class? |
|---------|----------|----------|---------------------|-------------------|
| Instance method | `self` | Operations on instance data | Yes | Yes (via `self.__class__`) |
| `@classmethod` | `cls` | Factory methods, alternative constructors | No | Yes |
| `@staticmethod` | Nothing | Utility functions logically grouped with class | No | No |
| `@property` | `self` | Computed attributes, validation, lazy access | Getter only (unless setter defined) | No |

```python
class ExperimentResult:
    def __init__(self, y_true, y_pred):
        self._y_true = y_true
        self._y_pred = y_pred
    
    @property
    def accuracy(self):
        """Computed on access — always up to date."""
        correct = sum(t == p for t, p in zip(self._y_true, self._y_pred))
        return correct / len(self._y_true)
    
    @property
    def error_rate(self):
        return 1.0 - self.accuracy
    
    @classmethod
    def from_csv(cls, filepath):
        """Factory method — alternative constructor."""
        import csv
        with open(filepath) as f:
            reader = csv.reader(f)
            y_true, y_pred = zip(*[(int(r[0]), int(r[1])) for r in reader])
        return cls(list(y_true), list(y_pred))
    
    @staticmethod
    def is_valid_prediction(value):
        """Pure utility — no access to instance or class needed."""
        return isinstance(value, (int, float)) and not math.isnan(value)
```

> **Critical Insight**: Use `@property` when an attribute is derived from other state. This avoids stale data — `self.accuracy` is always consistent with `self._y_true` and `self._y_pred`. Interviewers love seeing this pattern for model metrics.

---

## Dunder Methods That Matter

| Method | Purpose | DS Use Case |
|--------|---------|-------------|
| `__repr__` | Developer-friendly string | Debugging model configs in notebooks |
| `__len__` | `len(obj)` support | Dataset size |
| `__getitem__` | `obj[key]` indexing | Batch access in data loaders |
| `__iter__` | `for item in obj` | Streaming over large datasets |
| `__eq__` | `obj1 == obj2` | Comparing experiment configs |

```python
class Dataset:
    def __init__(self, features, labels):
        self._features = features
        self._labels = labels
    
    def __repr__(self):
        return f"Dataset(n_samples={len(self)}, n_features={len(self._features[0])})"
    
    def __len__(self):
        return len(self._features)
    
    def __getitem__(self, idx):
        """Supports integer indexing and slicing."""
        if isinstance(idx, slice):
            return Dataset(self._features[idx], self._labels[idx])
        return self._features[idx], self._labels[idx]
    
    def __iter__(self):
        for i in range(len(self)):
            yield self[i]
    
    def __eq__(self, other):
        return (isinstance(other, Dataset) and 
                self._features == other._features and 
                self._labels == other._labels)
```

---

## Inheritance & Polymorphism

```python
class BaseFeatureExtractor:
    def fit(self, X, y=None):
        return self
    
    def transform(self, X):
        raise NotImplementedError("Subclasses must implement transform()")
    
    def fit_transform(self, X, y=None):
        return self.fit(X, y).transform(X)

class TextLengthExtractor(BaseFeatureExtractor):
    def transform(self, X):
        return [[len(text)] for text in X]

class WordCountExtractor(BaseFeatureExtractor):
    def transform(self, X):
        return [[len(text.split())] for text in X]
```

**Polymorphism in action** — any extractor works interchangeably:

```python
extractors = [TextLengthExtractor(), WordCountExtractor()]
for ext in extractors:
    result = ext.fit_transform(["hello world", "foo"])
    # Same interface, different behavior
```

---

## Abstract Base Classes (ABC)

ABCs enforce that subclasses implement required methods — fail-fast at instantiation, not at runtime.

```python
from abc import ABC, abstractmethod

class BaseTransformer(ABC):
    """Interface contract: subclasses MUST implement these."""
    
    @abstractmethod
    def fit(self, X, y=None):
        pass
    
    @abstractmethod
    def transform(self, X):
        pass
    
    def fit_transform(self, X, y=None):
        """Concrete method — shared by all subclasses."""
        return self.fit(X, y).transform(X)

# This FAILS at instantiation — not at method call time:
# obj = BaseTransformer()  # TypeError: Can't instantiate abstract class
```

> **Critical Insight**: Use ABCs when building a framework or library where other developers (or your future self) will extend your classes. In DS, this is common for defining pipeline step interfaces, custom metric calculators, or data connector contracts.

---

## Dataclasses

Dataclasses eliminate boilerplate for classes that primarily hold data — perfect for configs and experiment tracking.

```python
from dataclasses import dataclass, field
from typing import List, Optional

@dataclass
class ExperimentConfig:
    model_name: str
    learning_rate: float = 0.001
    batch_size: int = 32
    epochs: int = 10
    tags: List[str] = field(default_factory=list)
    description: Optional[str] = None
    
    def __post_init__(self):
        if self.learning_rate <= 0:
            raise ValueError("learning_rate must be positive")

# Auto-generates __init__, __repr__, __eq__
config = ExperimentConfig("bert-base", learning_rate=3e-5, tags=["nlp", "prod"])
print(config)  # ExperimentConfig(model_name='bert-base', learning_rate=3e-05, ...)

# Equality comparison works out of the box
config2 = ExperimentConfig("bert-base", learning_rate=3e-5, tags=["nlp", "prod"])
assert config == config2  # True — compares field by field
```

**When to use dataclasses in DS:**

- Hyperparameter configs (replaces loose dicts)
- Feature store metadata
- Experiment results that get logged/compared
- Pipeline step parameters

> **Critical Insight**: Use `@dataclass(frozen=True)` for immutable configs — prevents accidental mutation during experiments and makes objects hashable (usable as dict keys or in sets).

---

## Composition vs Inheritance

| Aspect | Inheritance | Composition |
|--------|-------------|-------------|
| Relationship | "is-a" (Dog is an Animal) | "has-a" (Car has an Engine) |
| Coupling | Tight — child depends on parent internals | Loose — components are interchangeable |
| Flexibility | Fixed at class definition | Swap components at runtime |
| When to use | Shared interface + some shared logic | Combining behaviors, configurable pieces |
| DS example | `CustomScaler(BaseEstimator)` | `Pipeline([scaler, model])` |
| Danger | Deep hierarchies become fragile | More initial setup code |

```python
# INHERITANCE — use when you genuinely extend a framework class
class LogScaler(BaseFeatureExtractor):
    def transform(self, X):
        import math
        return [[math.log1p(v) for v in row] for row in X]

# COMPOSITION — preferred for assembling behavior
class FeaturePipeline:
    """Composes multiple transformers without inheriting from any."""
    
    def __init__(self, steps: list):
        self.steps = steps  # List of transformer objects
    
    def fit_transform(self, X, y=None):
        result = X
        for step in self.steps:
            result = step.fit_transform(result, y)
        return result

# Swap components freely — no class hierarchy needed
pipeline = FeaturePipeline([TextLengthExtractor(), LogScaler()])
```

> **Critical Insight**: "Favor composition over inheritance" is the golden rule. Sklearn's `Pipeline` itself is composition. Only inherit when you're implementing a specific interface (like `BaseEstimator`/`TransformerMixin`) — never just to share code.

---

## Practical DS Example: Custom Sklearn Transformer

This is the type of code you might write in a Meta or Google interview when asked to build a reusable feature engineering step:

```python
from sklearn.base import BaseEstimator, TransformerMixin
import numpy as np

class OutlierClipper(BaseEstimator, TransformerMixin):
    """Clips features to [lower, upper] percentiles learned during fit.
    
    Fully compatible with sklearn Pipeline, GridSearchCV, and cross_val_score.
    """
    
    def __init__(self, lower_percentile=1, upper_percentile=99):
        self.lower_percentile = lower_percentile
        self.upper_percentile = upper_percentile
    
    def fit(self, X, y=None):
        X = np.asarray(X)
        self.lower_bounds_ = np.percentile(X, self.lower_percentile, axis=0)
        self.upper_bounds_ = np.percentile(X, self.upper_percentile, axis=0)
        return self
    
    def transform(self, X):
        X = np.asarray(X)
        return np.clip(X, self.lower_bounds_, self.upper_bounds_)

# Usage in a pipeline
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression

pipe = Pipeline([
    ('clip', OutlierClipper(lower_percentile=5, upper_percentile=95)),
    ('scale', StandardScaler()),
    ('model', LogisticRegression())
])
# pipe.fit(X_train, y_train) — works with GridSearchCV, cross_val_score, etc.
```

> **Critical Insight**: The sklearn contract requires: (1) `__init__` params match `get_params()` output exactly, (2) `fit()` returns `self`, (3) learned attributes end with underscore (e.g., `lower_bounds_`). Violating these breaks `GridSearchCV`.

---

## Interview Talking Points & Scripts

> *"I use OOP in data science primarily for two things: building reusable pipeline components that conform to sklearn's fit/transform interface, and creating configuration objects with dataclasses that make experiment tracking reproducible. I favor composition over inheritance — my pipelines compose steps rather than building deep class hierarchies."*

> *"The key dunder methods I implement most often are `__repr__` for debuggability in notebooks, `__len__` and `__getitem__` for dataset classes that need to support batching, and `__eq__` for comparing experiment configs. These make objects behave like native Python containers."*

> *"I reach for ABCs when defining contracts in a team setting — for example, a base DataConnector class that forces all implementations to have `connect()`, `read()`, and `close()` methods. This catches missing implementations at import time rather than in production at 2 AM."*

> *"@property is my go-to for computed metrics on model result objects. Instead of storing accuracy as a stale attribute, I compute it on access — so it's always consistent with the underlying predictions. It also lets me add validation in setters without changing the API."*

---

## Common Interview Mistakes

| Mistake | Why It's Wrong | Correct Approach |
|---------|---------------|-----------------|
| Mutable class attribute as default | Shared across ALL instances — one instance mutates everyone's data | Use `None` default + assign in `__init__`, or `field(default_factory=list)` |
| Forgetting `super().__init__()` | Parent's initialization is skipped — attributes missing, MRO broken | Always call `super().__init__()` in subclass `__init__` |
| Overusing inheritance for code reuse | Creates fragile coupling, hard to test | Use composition or mixins for shared behavior |
| Storing computed values as attributes | Values go stale when inputs change | Use `@property` for derived values |
| `__init__` params not matching attributes | Breaks sklearn's `get_params()`/`set_params()` | Keep `__init__` signature and stored attributes in sync |
| Using `type(x) == MyClass` for checks | Fails for subclasses | Use `isinstance(x, MyClass)` |
| Writing `__eq__` without `__hash__` | Object becomes unhashable (can't use in sets/dict keys) | Implement both, or use `@dataclass(frozen=True)` |

---

## Rapid-Fire Q&A

**Q1: What's the difference between `@classmethod` and `@staticmethod`?**  
A: `@classmethod` receives the class as `cls` — use for factory methods. `@staticmethod` receives nothing — it's just a namespaced utility function.

**Q2: When would you use `__slots__`?**  
A: When you have millions of instances (e.g., data records) and need to reduce memory. It prevents `__dict__` creation per instance.

**Q3: What is MRO and why does it matter?**  
A: Method Resolution Order (C3 linearization) determines which parent's method gets called in diamond inheritance. Check with `ClassName.__mro__`.

**Q4: Why use `@dataclass` over a plain dict for configs?**  
A: Type safety, IDE autocompletion, `__repr__` for free, immutability with `frozen=True`, and `__post_init__` validation.

**Q5: How does sklearn's `TransformerMixin` work?**  
A: It provides `fit_transform()` for free — it just calls `self.fit(X, y).transform(X)`. You only need to implement `fit()` and `transform()`.

**Q6: What does `__repr__` vs `__str__` mean?**  
A: `__repr__` is for developers (unambiguous, ideally `eval`-able). `__str__` is for users (pretty). If only one is defined, `__repr__` is the fallback for both.

**Q7: Can you have multiple inheritance in Python?**  
A: Yes, but use it carefully. Mixins (small, single-purpose classes) are the safe pattern. Deep diamond hierarchies are a code smell.

**Q8: What's the point of `ABC` if Python has duck typing?**  
A: ABCs give fail-fast behavior — you get `TypeError` at instantiation if a method is missing, not a `NotImplementedError` in production at runtime.

**Q9: How do you make a class iterable?**  
A: Implement `__iter__` (returning an iterator) or `__getitem__` (Python will auto-iterate by index starting at 0).

**Q10: When should you NOT use OOP in DS?**  
A: For one-off analysis scripts, EDA notebooks, or simple data transformations — a function is simpler. OOP shines when you need reusable, testable, composable components.

---

## Summary Cheat Sheet

```
+-------------------------------------------------------------------+
|  OOP FOR DATA SCIENCE — QUICK REFERENCE                           |
+-------------------------------------------------------------------+
|                                                                   |
|  WHEN TO USE OOP:                                                 |
|    - Custom sklearn transformers (fit/transform interface)         |
|    - Dataset classes (PyTorch-style __getitem__/__len__)           |
|    - Config objects (dataclass with validation)                    |
|    - Framework extensions (ABC-defined contracts)                  |
|                                                                   |
|  GOLDEN RULES:                                                    |
|    1. Favor composition over inheritance                          |
|    2. Use @property for computed/derived attributes                |
|    3. Use dataclasses for data-holding objects                    |
|    4. Use ABC when building extensible frameworks                 |
|    5. Keep __init__ params and stored attrs in sync (sklearn)     |
|                                                                   |
|  KEY DECORATORS:                                                  |
|    @property      → computed attribute (lazy/derived)             |
|    @classmethod   → factory/alternative constructor               |
|    @staticmethod  → namespaced utility                            |
|                                                                   |
|  ESSENTIAL DUNDERS:                                               |
|    __repr__     → debuggable representation                       |
|    __len__      → len(obj)                                        |
|    __getitem__  → obj[i], enables iteration + slicing             |
|    __iter__     → explicit iteration protocol                     |
|    __eq__       → comparison (pair with __hash__)                 |
|                                                                   |
|  COMMON TRAPS:                                                    |
|    - Mutable default class attrs → shared state bug               |
|    - Missing super().__init__() → broken MRO                      |
|    - Stale attributes → use @property instead                     |
|    - Deep inheritance → use composition                           |
|                                                                   |
+-------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 5 of 25*
