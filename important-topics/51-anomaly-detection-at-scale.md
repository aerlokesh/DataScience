# 🎯 Topic 51: Anomaly Detection at Scale

> *"The art of anomaly detection is not finding anomalies — it's finding the RIGHT anomalies. At production scale, every system generates noise. The challenge is separating signal from noise without drowning your on-call engineers in false alerts."*

---

## 📑 Table of Contents

1. [Foundations of Anomaly Detection](#1-foundations-of-anomaly-detection)
2. [Statistical Methods](#2-statistical-methods)
3. [Machine Learning Methods](#3-machine-learning-methods)
4. [Time-Series Anomaly Detection](#4-time-series-anomaly-detection)
5. [Streaming Anomaly Detection](#5-streaming-anomaly-detection)
6. [Alert Fatigue and Threshold Tuning](#6-alert-fatigue-and-threshold-tuning)
7. [False Positive Management](#7-false-positive-management)
8. [Production Systems and Use Cases](#8-production-systems-and-use-cases)
9. [Comparison Table of Methods](#9-comparison-table-of-methods)
10. [Python Implementation Examples](#10-python-implementation-examples)
11. [Interview Talking Points](#11-interview-talking-points)
12. [Common Mistakes](#12-common-mistakes)
13. [Rapid-Fire Q&A](#13-rapid-fire-qa)
14. [ASCII Cheat Sheet](#14-ascii-cheat-sheet)

---

## 1. Foundations of Anomaly Detection

### What is an Anomaly?

An anomaly (outlier) is a data point that deviates significantly from the expected pattern. Three types:

```
1. Point Anomaly:     Single data point is abnormal
                      Example: $50K transaction from a user who spends $50/day

2. Contextual Anomaly: Point is abnormal given its context
                       Example: 80°F is normal in July, anomalous in January

3. Collective Anomaly: A sequence of points is abnormal together
                       Example: 5 failed logins is normal; 500 in 1 minute is not
```

### The Fundamental Challenge

```
                    ┌─────────────────────────────┐
                    │    ANOMALY DETECTION         │
                    │    IMPOSSIBILITY TRILEMMA    │
                    └─────────────────────────────┘
                    
     High Recall ──────── Pick 2 ──────── Low False Positives
          \                                      /
           \                                    /
            \                                  /
             ────────── Low Latency ──────────
```

You cannot simultaneously achieve: (1) catch all anomalies, (2) have zero false alarms, (3) detect in real-time. Every system makes trade-offs.

### Supervised vs Unsupervised

| Setting | Description | When to Use |
|---------|-------------|------------|
| **Supervised** | Labeled normal + anomaly examples | Fraud detection with historical labels |
| **Semi-supervised** | Only normal data for training | System health (anomalies are rare/novel) |
| **Unsupervised** | No labels at all | Exploratory, finding unknown patterns |

> **Critical Insight:** In practice, most anomaly detection is **semi-supervised** — you have abundant normal data but very few labeled anomalies. This means you're really learning "what normal looks like" and flagging deviations. The choice of "how much deviation = anomaly" is always a business decision, not a purely statistical one.

---

## 2. Statistical Methods

### Z-Score Method

```
z_score = (x - μ) / σ

Anomaly if |z_score| > threshold (typically 3)

Assumptions:
- Data is approximately normally distributed
- Stationary (mean and variance don't change over time)
```

**Modified Z-Score (Robust):**
```
MAD = median(|x_i - median(x)|)
Modified_Z = 0.6745 × (x_i - median(x)) / MAD

More robust to outliers themselves affecting μ and σ
```

### IQR (Interquartile Range) Method

```
Q1 = 25th percentile
Q3 = 75th percentile
IQR = Q3 - Q1

Lower bound = Q1 - 1.5 × IQR
Upper bound = Q3 + 1.5 × IQR

Anomaly if x < Lower bound OR x > Upper bound
```

**Advantages:** Non-parametric, robust to non-normal distributions
**Limitation:** Only detects univariate point anomalies

### Grubbs' Test

```
Hypothesis test for a single outlier in univariate data:

G = max|x_i - x̄| / s

Compare G to critical value from Grubbs' table.
Reject H₀ (no outlier) if G > G_critical(α, n)

Limitation: Tests for ONE outlier at a time. Must apply iteratively.
Assumption: Data is normally distributed.
```

### Mahalanobis Distance (Multivariate)

```
D²(x) = (x - μ)ᵀ Σ⁻¹ (x - μ)

Where:
- μ = mean vector
- Σ = covariance matrix

Accounts for correlations between features.
Under normality, D² follows a Chi-squared distribution with p degrees of freedom.
Anomaly if D² > χ²(p, 1-α)
```

> **Critical Insight:** Statistical methods are **interpretable and fast** — perfect for dashboards and simple alerting. But they assume stationarity and parametric distributions. For production metrics with daily/weekly seasonality, trends, and heteroscedasticity, they need preprocessing (detrending, seasonal decomposition) to work reliably.

---

## 3. Machine Learning Methods

### Isolation Forest

**Key Intuition:** Anomalies are "easy to isolate" — they require fewer random splits to separate from the rest.

```
Algorithm:
1. Build an ensemble of random trees (iForest)
2. Each tree: randomly select a feature and split point
3. Anomaly score = average path length across trees
4. Short path length → easy to isolate → likely anomaly

Score normalization:
s(x, n) = 2^(-E(h(x)) / c(n))

Where:
- E(h(x)) = average path length for point x
- c(n) = average path length in a random BST
- Score ∈ (0, 1): closer to 1 = more anomalous
```

**Hyperparameters:**
- `n_estimators`: 100-300 trees (more = stable)
- `max_samples`: 256 (subsampling for efficiency)
- `contamination`: expected fraction of anomalies (0.01-0.1)

### Local Outlier Factor (LOF)

**Key Intuition:** An anomaly has a substantially lower density than its neighbors.

```
LOF(x) = average local reachability density of neighbors / local reachability density of x

LOF ≈ 1: similar density to neighbors (normal)
LOF >> 1: much lower density than neighbors (anomaly)

Steps:
1. Find k-nearest neighbors of x
2. Compute reachability distance: max(k-dist(neighbor), d(x, neighbor))
3. Compute local reachability density (LRD) = inverse of avg reachability distance
4. LOF = mean(LRD of neighbors) / LRD(x)
```

**Strengths:** Detects local anomalies (outlier in its own region but not globally)
**Weakness:** O(n²) computation, struggles with high dimensions

### One-Class SVM

```
Learns a boundary around normal data in feature space:

Objective: Find hyperplane that maximizes margin from origin in feature space
min (1/2)||w||² + (1/νn)Σ max(0, -f(x_i)) - ρ

Where:
- ν = upper bound on fraction of outliers (similar to contamination)
- Kernel trick maps to high-dimensional space (RBF common)
- Points with f(x) < 0 are anomalies
```

### Autoencoders for Anomaly Detection

```
Architecture:
Input → Encoder → Bottleneck (compressed representation) → Decoder → Reconstruction

Training: Only on normal data
Inference: High reconstruction error → anomaly

Loss = ||x - x̂||²  (reconstruction error)

Anomaly score = reconstruction_error(x)
Threshold: percentile of training reconstruction errors (e.g., 99th)
```

**Variational Autoencoder (VAE) extension:**
```
Instead of reconstruction error alone:
Anomaly score = -ELBO = reconstruction_error + KL_divergence

Advantage: Also detects points that are unlikely under the learned latent distribution
```

> **Critical Insight:** Autoencoders are the workhorse of modern anomaly detection for high-dimensional data (images, sensor data, logs). The key insight is that the model learns to reconstruct NORMAL patterns. Anomalies are patterns it has never seen and thus reconstructs poorly. The bottleneck forces the model to learn a compressed representation of "normal."

---

## 4. Time-Series Anomaly Detection

### STL Decomposition + Residual Thresholding

```
Time series = Trend + Seasonal + Residual
                T_t  +   S_t    +   R_t

Anomaly detection on residuals:
1. Decompose with STL (Seasonal-Trend decomposition using LOESS)
2. Compute residuals R_t = Y_t - T_t - S_t
3. Flag if |R_t| > k × σ_R (typically k = 3)

Advantages: Handles seasonality naturally
            Interpretable ("this spike is not explained by trend or season")
```

### Prophet-Based Anomaly Detection

```python
from prophet import Prophet

# Train Prophet model on historical data
model = Prophet(interval_width=0.99)  # 99% prediction interval
model.fit(df_train)

# Predict
forecast = model.predict(df_test)

# Anomaly: actual value outside prediction interval
anomalies = df_test[
    (df_test['y'] < forecast['yhat_lower']) | 
    (df_test['y'] > forecast['yhat_upper'])
]
```

**Prophet handles:** Multiple seasonality (daily, weekly, yearly), holidays, trend changepoints, missing data.

### Changepoint Detection

Detect when statistical properties change: mean shift, variance shift, or trend change.

Algorithms: CUSUM (mean shifts), PELT (optimal segmentation), Bayesian Online Changepoint Detection (streaming-friendly, maintains run-length probability distribution).

### CUSUM (Cumulative Sum Control Chart)

```
S_t = max(0, S_{t-1} + (x_t - μ₀ - k))

Where:
- μ₀ = expected mean (target)
- k = allowance (sensitivity, typically σ/2)
- Signal alarm when S_t > h (decision threshold)

Two-sided: track both positive (upward shift) and negative (downward shift)
```

> **Critical Insight:** For production metrics, the most common pattern is **STL decomposition → residual analysis → dynamic thresholding**. This handles seasonality (daily/weekly patterns), trends (growth), and adapts thresholds to recent volatility. Static thresholds on raw metrics generate excessive false positives during expected seasonal changes.

---

## 5. Streaming Anomaly Detection

### Requirements for Real-Time Detection

Constraints: process each point in O(1) or O(log n), bounded memory, adapt to concept drift, detect within seconds/minutes.

### Exponentially Weighted Moving Average (EWMA)

`mu_t = alpha * x_t + (1-alpha) * mu_{t-1}` and `sigma^2_t = alpha * (x_t - mu_t)^2 + (1-alpha) * sigma^2_{t-1}`. Anomaly if `|x_t - mu_t| > k * sigma_t`. Alpha close to 1 reacts quickly (more FP); alpha close to 0 adapts slowly (misses sudden shifts).

### Sliding Window Approaches

Maintain a window of last N points with rolling mean/std/percentiles. Window size trade-off: small = sensitive but noisy, large = stable but slow. Typical production setting: short window (5 min) for spike detection, long window (7 days) for baseline comparison. Alert if short_window_metric > long_window_baseline + threshold.

### Twitter's S-H-ESD (Seasonal Hybrid ESD)

Combines STL decomposition (seasonality removal) with Generalized ESD test on residuals. Key innovation: uses median-based decomposition so anomalies don't corrupt seasonal/trend estimates.

---

## 6. Alert Fatigue and Threshold Tuning

### The Alert Fatigue Problem

Scenario: 500 microservices x 10 metrics = 5000 time series. Static threshold at 3-sigma generates ~15 false alerts/day. On-call stops trusting alerts by day 3. Real anomaly gets ignored on day 7 leading to production incident. The paradox: more sensitive alerts lead to more false positives, less trust, and slower response to real issues.

### Dynamic Thresholding

Instead of `alert if metric > FIXED_VALUE`, use `alert if metric > f(recent_history, day_of_week, seasonality)`. Approaches: (1) Percentile-based: alert if value > 99.9th percentile of same hour/day. (2) Prediction-based: alert if |actual - predicted| > k x recent_error_std. (3) Adaptive: threshold = mu_recent + k x sigma_recent (EWMA-based).

### Alert Prioritization

`Priority = Severity x Confidence x Novelty`. Severity: business impact (revenue > latency > error rate). Confidence: multiple correlated signals > single spike. Novelty: first-time failure mode > recurring pattern.

> **Critical Insight:** The best anomaly detection system is one that engineers **trust and act on**. This means optimizing for precision (low false positive rate) even at the cost of some recall. A system that catches 95% of issues but has a 50% false positive rate will be ignored. A system that catches 70% of issues with a 5% false positive rate will be trusted and acted upon.

---

## 7. False Positive Management

### Root Causes of False Positives

| Cause | Example | Solution |
|-------|---------|----------|
| Seasonal patterns | Traffic drops every Sunday | Model seasonality explicitly |
| Planned events | Black Friday spike | Event calendar integration |
| Deploy artifacts | Latency spike during deploy | Suppress during deploys |
| Metric gaps | Missing data looks like zero | Distinguish null vs zero |
| Correlated alerts | One root cause, 50 alerts | Alert grouping/dedup |
| Threshold rot | Business grew, old threshold irrelevant | Auto-adaptive thresholds |

### Strategies for Reducing False Positives

1. **Multi-Signal Confirmation:** Single metric spike = warning; multiple correlated metrics anomalous = page. Example: latency spike alone could be one slow request, but latency + error rate + throughput drop = real issue.

2. **Minimum Duration:** Require anomaly to persist for N consecutive points before alerting. Eliminates instantaneous blips.

3. **Anomaly Score Smoothing:** Apply EMA to raw scores; alert only if smoothed score exceeds threshold for >3 minutes.

4. **Feedback Loop:** Engineers mark alerts as FP/TP; system learns which patterns are acceptable and auto-tunes thresholds.

### Measuring Alert Quality

Alert Precision = True Alerts / (True + False Alerts). Alert Recall = True Alerts / (True Alerts + Missed Incidents). MTTD = Mean Time To Detect. Target: Precision > 80%, MTTD < 5 minutes for critical metrics.

---

## 8. Production Systems and Use Cases

### Monitoring Metrics (Site Reliability)

Metrics: request latency (p50/p95/p99), error rate (5xx/4xx), throughput (rps), saturation (CPU/memory/disk).

```
Architecture: Metrics → Kafka → Anomaly Detector → Alert Router → PagerDuty
                                      │                               │
                                Model Registry                  On-call Engineer
                              (per-metric thresholds)            (with context)
```

### Pipeline Health Monitoring

Monitor: row counts (drop = upstream failure), null rates (spike = schema change), value distributions (shift = quality problem), freshness (SLA violation = pipeline stuck), schema changes. Challenge: pipelines have natural variability (weekday/weekend, holidays).

### Fraud Detection

Unique challenges: extreme class imbalance (0.1% fraud), adversarial adaptation, cost asymmetry (FN >>> FP cost), real-time requirement (<100ms), label delay (days/weeks).

Architecture: Transaction -> Feature Extraction -> Rule Engine (fast, high precision) -> ML Model (uncertain cases) -> Score -> Threshold -> Block/Allow/Review queue.

> **Critical Insight:** At scale (millions of metrics), the anomaly detection problem becomes a **meta-problem**: you need algorithms to decide which algorithm and parameters to use for each metric. Netflix and Uber both implement automated model selection based on metric characteristics (seasonality strength, trend, noise level, history length).

---

## 9. Comparison Table of Methods

| Method | Type | Strengths | Weaknesses | Best For |
|--------|------|-----------|------------|----------|
| **Z-Score** | Statistical | Simple, fast, interpretable | Assumes normality, not robust | Quick checks, Gaussian data |
| **IQR** | Statistical | Non-parametric, robust | Univariate only | Box-plot alerting |
| **Grubbs** | Statistical | Formal hypothesis test | One outlier at a time, assumes normal | Small clean datasets |
| **Mahalanobis** | Statistical | Multivariate, accounts for correlation | Assumes multivariate normal | Tabular, low-dim |
| **Isolation Forest** | ML (tree) | Fast, scalable, no distribution assumption | Contamination param needed | General purpose, high-dim |
| **LOF** | ML (density) | Finds local anomalies | O(n^2), high-dim struggles | Clusters with varying density |
| **One-Class SVM** | ML (boundary) | Works in high-dim with kernel | Slow to train, kernel selection | Medium datasets |
| **Autoencoder** | ML (neural) | Handles complex patterns, high-dim | Needs tuning, less interpretable | Images, sequences, rich data |
| **STL + threshold** | Time-series | Handles seasonality, interpretable | Assumes additive decomposition | Seasonal metrics |
| **Prophet** | Time-series | Multiple seasonality, holidays, robust | Slower than simple methods | Business metrics |
| **CUSUM** | Streaming | Low latency, detects shifts | Only mean shifts, needs known μ₀ | Manufacturing, SPC |
| **EWMA** | Streaming | Adaptive, simple, fast | Single parameter tradeoff | Real-time monitoring |
| **S-H-ESD** | Hybrid | Handles seasonality + multiple anomalies | Batch (not streaming) | Twitter-scale metrics |

---

## 10. Python Implementation Examples

### Isolation Forest

```python
import numpy as np
from sklearn.ensemble import IsolationForest
import pandas as pd

def detect_anomalies_iforest(data, contamination=0.05):
    """
    Detect anomalies using Isolation Forest.
    
    Parameters:
    -----------
    data : pd.DataFrame
        Feature matrix (each row is an observation)
    contamination : float
        Expected proportion of anomalies
    
    Returns:
    --------
    pd.Series with -1 (anomaly) or 1 (normal)
    """
    model = IsolationForest(
        n_estimators=200,
        max_samples='auto',
        contamination=contamination,
        random_state=42,
        n_jobs=-1
    )
    
    predictions = model.fit_predict(data)
    scores = model.decision_function(data)  # Lower = more anomalous
    
    return pd.DataFrame({
        'prediction': predictions,
        'anomaly_score': -scores  # Negate so higher = more anomalous
    })

# Example usage
np.random.seed(42)
normal_data = np.random.randn(1000, 3)
anomalies = np.random.uniform(low=-4, high=4, size=(50, 3))
data = pd.DataFrame(
    np.vstack([normal_data, anomalies]),
    columns=['feature_1', 'feature_2', 'feature_3']
)

results = detect_anomalies_iforest(data, contamination=0.05)
print(f"Anomalies detected: {(results['prediction'] == -1).sum()}")
```

### STL-Based Time Series Anomaly Detection

```python
import numpy as np
import pandas as pd
from statsmodels.tsa.seasonal import STL

def detect_timeseries_anomalies(ts, period=7, threshold_sigma=3):
    """
    Detect anomalies in time series using STL decomposition.
    
    Parameters:
    -----------
    ts : pd.Series with DatetimeIndex
        Time series data
    period : int
        Seasonal period (7 for weekly, 24 for hourly-daily, etc.)
    threshold_sigma : float
        Number of standard deviations for anomaly threshold
    
    Returns:
    --------
    pd.DataFrame with decomposition and anomaly flags
    """
    # STL decomposition
    stl = STL(ts, period=period, robust=True)
    result = stl.fit()
    
    # Extract residuals
    residuals = result.resid
    
    # Robust threshold using MAD
    mad = np.median(np.abs(residuals - np.median(residuals)))
    threshold = threshold_sigma * 1.4826 * mad  # 1.4826 converts MAD to σ equivalent
    
    # Flag anomalies
    anomalies = np.abs(residuals) > threshold
    
    return pd.DataFrame({
        'observed': ts,
        'trend': result.trend,
        'seasonal': result.seasonal,
        'residual': residuals,
        'threshold_upper': result.trend + result.seasonal + threshold,
        'threshold_lower': result.trend + result.seasonal - threshold,
        'is_anomaly': anomalies
    })

# Example: detect anomalies in daily revenue
dates = pd.date_range('2024-01-01', periods=365, freq='D')
np.random.seed(42)

# Simulate: trend + weekly seasonality + noise + injected anomalies
trend = np.linspace(100, 150, 365)
seasonal = 10 * np.sin(2 * np.pi * np.arange(365) / 7)
noise = np.random.randn(365) * 5
revenue = trend + seasonal + noise

# Inject anomalies
revenue[50] = 250   # Spike
revenue[200] = 50   # Drop

ts = pd.Series(revenue, index=dates)
results = detect_timeseries_anomalies(ts, period=7)
print(f"Anomalies detected: {results['is_anomaly'].sum()}")
```

### Streaming Anomaly Detection with EWMA

```python
import numpy as np
from collections import deque

class StreamingAnomalyDetector:
    """
    Real-time anomaly detection using EWMA with dynamic thresholds.
    Suitable for monitoring production metrics.
    """
    
    def __init__(self, alpha=0.1, threshold_sigma=3, warmup_period=100):
        """
        Parameters:
        -----------
        alpha : float
            EWMA smoothing factor (0.01-0.3 typical)
        threshold_sigma : float
            Number of standard deviations for threshold
        warmup_period : int
            Points to observe before alerting
        """
        self.alpha = alpha
        self.threshold_sigma = threshold_sigma
        self.warmup_period = warmup_period
        
        self.ewma_mean = None
        self.ewma_var = None
        self.count = 0
        self.recent_anomalies = deque(maxlen=10)
    
    def update(self, value):
        """
        Process a new data point. Returns anomaly score and alert decision.
        
        Returns:
        --------
        dict with 'is_anomaly', 'score', 'threshold', 'ewma_mean'
        """
        self.count += 1
        
        # Initialize
        if self.ewma_mean is None:
            self.ewma_mean = value
            self.ewma_var = 0.0
            return {'is_anomaly': False, 'score': 0.0, 
                    'threshold': float('inf'), 'ewma_mean': value}
        
        # Update EWMA statistics
        deviation = value - self.ewma_mean
        self.ewma_mean = self.alpha * value + (1 - self.alpha) * self.ewma_mean
        self.ewma_var = self.alpha * deviation**2 + (1 - self.alpha) * self.ewma_var
        
        ewma_std = np.sqrt(self.ewma_var)
        
        # Calculate anomaly score
        if ewma_std > 0:
            score = abs(deviation) / ewma_std
        else:
            score = 0.0
        
        # Determine threshold (dynamic)
        threshold = self.threshold_sigma
        
        # Don't alert during warmup
        is_anomaly = (score > threshold) and (self.count > self.warmup_period)
        
        if is_anomaly:
            self.recent_anomalies.append(self.count)
        
        return {
            'is_anomaly': is_anomaly,
            'score': score,
            'threshold': threshold,
            'ewma_mean': self.ewma_mean,
            'ewma_std': ewma_std
        }

# Example: monitor a metric stream
detector = StreamingAnomalyDetector(alpha=0.05, threshold_sigma=3.5)

np.random.seed(42)
# Simulate normal metric with occasional anomalies
normal_stream = np.random.randn(500) * 10 + 100
normal_stream[250] = 200  # Inject spike
normal_stream[400] = 30   # Inject drop

anomaly_count = 0
for i, value in enumerate(normal_stream):
    result = detector.update(value)
    if result['is_anomaly']:
        anomaly_count += 1
        print(f"ANOMALY at t={i}: value={value:.1f}, "
              f"score={result['score']:.2f}, mean={result['ewma_mean']:.1f}")

print(f"\nTotal anomalies: {anomaly_count}")
```

### Autoencoder-Based Detection

```python
import numpy as np
from tensorflow import keras

def build_anomaly_autoencoder(input_dim, encoding_dim=8):
    """Build autoencoder for anomaly detection. Train on normal data only."""
    encoder = keras.Sequential([
        keras.layers.Dense(64, activation='relu', input_shape=(input_dim,)),
        keras.layers.Dense(32, activation='relu'),
        keras.layers.Dense(encoding_dim, activation='relu'),
    ])
    decoder = keras.Sequential([
        keras.layers.Dense(32, activation='relu', input_shape=(encoding_dim,)),
        keras.layers.Dense(64, activation='relu'),
        keras.layers.Dense(input_dim, activation='sigmoid'),
    ])
    autoencoder = keras.Sequential([encoder, decoder])
    autoencoder.compile(optimizer='adam', loss='mse')
    return autoencoder

def detect_with_autoencoder(model, data, threshold_percentile=99):
    """Score data points by reconstruction error."""
    reconstructions = model.predict(data, verbose=0)
    errors = np.mean((data - reconstructions)**2, axis=1)
    threshold = np.percentile(errors, threshold_percentile)
    return {'errors': errors, 'threshold': threshold, 'anomalies': errors > threshold}

# Usage: train on NORMAL data, score new data — high error = anomaly
```

---

## 11. Interview Talking Points

> **"Design an anomaly detection system for monitoring 10,000 microservice metrics at Uber/Netflix."**
>
> "I'd build a streaming pipeline: metrics flow through Kafka, each processed by a per-metric detector. For model selection, I'd cluster metrics by behavior (seasonal-periodic, trending, stationary, sparse) and assign appropriate algorithms — STL+residual for seasonal metrics, EWMA for stationary, and Prophet for complex patterns. Key design decisions: (1) Dynamic thresholds adapt to each metric's volatility. (2) Multi-signal confirmation — require 2+ correlated metrics to anomaly before paging. (3) Deploy-aware suppression using integration with the deployment system. (4) Feedback loop where engineers mark false/true positives to auto-tune thresholds. Target: >80% precision, <5 minute MTTD for critical metrics."

> **"How would you handle the false positive problem in fraud detection?"**
>
> "False positives in fraud are costly — blocking legitimate customers hurts revenue and trust. I'd implement a tiered approach: (1) Hard rules for obvious fraud (block immediately). (2) ML model outputs a continuous risk score. (3) Multiple decision boundaries: score > 0.95 → auto-block, 0.7-0.95 → manual review queue, < 0.7 → allow. (4) Cost-sensitive optimization: the threshold is set where marginal cost of FP (lost legitimate transaction) equals marginal cost of FN (fraud loss). (5) Feedback: confirmed fraud labels are fed back within 24h to retrain. (6) Feature velocity tracking — I'd monitor how fast the fraud patterns evolve and retrain accordingly."

> **"Compare Isolation Forest vs Autoencoder for anomaly detection. When would you use each?"**
>
> "Isolation Forest: fast, works out-of-the-box, handles tabular data well, interpretable (you can trace which features caused isolation). Use when you have structured/tabular data, need quick deployment, or have limited labeled data. Autoencoder: handles complex, high-dimensional data (images, sequences, sensor streams), captures non-linear patterns, but requires more tuning and training data. Use when your data is unstructured, high-dimensional, or has complex temporal dependencies. In practice, I'd prototype with Isolation Forest first — if it achieves target precision/recall, ship it. Only move to autoencoders if the data complexity demands it."

---

## 12. Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Static thresholds on seasonal metrics | Use STL decomposition or dynamic thresholds |
| Training anomaly detector on data containing anomalies | Clean training data or use robust methods (Isolation Forest) |
| Alerting on every statistical anomaly | Filter by business impact and confidence |
| Ignoring the base rate (too sensitive) | Tune for precision > 80% before increasing recall |
| One algorithm for all metrics | Cluster metrics by pattern, assign appropriate detectors |
| Not accounting for metric correlation | Group correlated alerts to reduce redundancy |
| Evaluating without ground truth labels | Create a labeled evaluation set from past incidents |
| Ignoring concept drift | Periodically retrain or use online-learning approaches |
| Setting contamination=0.5 in Isolation Forest | Use domain knowledge; typical contamination is 0.01-0.05 |
| Reporting anomaly detection accuracy like classification | Use precision@k, recall@k, or time-to-detect as metrics |

---

## 13. Rapid-Fire Q&A

**Q1: Why is Isolation Forest preferred over LOF for high-dimensional data?**
A: Isolation Forest has O(n log n) complexity and handles high dimensions well because random splits still work. LOF requires distance computation in high-D where distances become meaningless (curse of dimensionality) and has O(n^2) complexity.

**Q2: How do you evaluate an anomaly detector without labels?**
A: (1) Inject synthetic anomalies and measure detection rate. (2) Use known past incidents as positives. (3) Domain expert review of top-scored points. (4) Downstream impact: did acting on alerts prevent incidents? (5) A/B test alert configurations against incident MTTR.

**Q3: What's the difference between an outlier and a changepoint?**
A: An outlier is a single point deviating from the pattern (temporary). A changepoint is a persistent shift in the time series behavior (new normal). Treating a changepoint as a sequence of outliers causes continuous false alerts.

**Q4: How does contamination parameter work in Isolation Forest?**
A: It sets the decision threshold on anomaly scores. contamination=0.05 means the top 5% of points by anomaly score are labeled as anomalies. It does NOT affect the model itself (tree structure), only the threshold on scores.

**Q5: Why use MAD instead of standard deviation for thresholding?**
A: Standard deviation is sensitive to outliers (outliers inflate sigma, making them harder to detect — a masking effect). MAD (Median Absolute Deviation) is robust: even 49% contamination doesn't affect it. This is critical when anomalies are already present in the training window.

**Q6: What is the "inductive" vs "transductive" distinction in anomaly detection?**
A: Transductive: scores depend on the full dataset (adding a new point changes all scores). Example: LOF. Inductive: model is trained once, new points scored independently. Example: Isolation Forest (once trees are built). Inductive is needed for production/streaming.

**Q7: How would you handle seasonality in streaming anomaly detection?**
A: Maintain per-time-slot statistics (e.g., separate EWMA for each hour-of-day and day-of-week). Compare current value to the expected value for "this hour on this day of week." This requires more memory but properly handles recurring patterns.

**Q8: What's the cold-start problem in anomaly detection?**
A: A new metric/service has no history to establish "normal." Solutions: (1) Use generic thresholds from similar metrics. (2) Warmup period where detection is disabled. (3) Transfer learning from related metrics. (4) Conservative (wide) thresholds that tighten as data accumulates.

**Q9: How do you handle missing data in time-series anomaly detection?**
A: (1) Distinguish between "missing" (system failure) and "zero" (genuine). Missing data itself may be an anomaly (pipeline broken). (2) For modeling: interpolate short gaps, flag long gaps. (3) Never impute and then alert on the imputed values. (4) Track a "data freshness" metric separately.

**Q10: Why do autoencoders sometimes fail to detect anomalies?**
A: (1) Overcapacity: model is too powerful and can reconstruct anything including anomalies. Fix: reduce bottleneck size. (2) Training data contamination: anomalies in training teach the model to reconstruct them. (3) Mode collapse: encoder maps everything to same representation. (4) Anomaly type differs from expected: trained on spike detection but anomaly is a slow drift.

---

## 14. ASCII Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                   ANOMALY DETECTION AT SCALE                         ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  ANOMALY TYPES                                                       ║
║  ┌──────────────────────────────────────────┐                        ║
║  │  Point:      ──────────·──────────       │ Single outlier         ║
║  │  Contextual: ──Summer:OK──Winter:BAD──   │ Context-dependent      ║
║  │  Collective: ──────[█████]───────────    │ Sequence is abnormal   ║
║  └──────────────────────────────────────────┘                        ║
║                                                                      ║
║  METHOD SELECTION GUIDE                                              ║
║  ┌──────────────────────────────────────────┐                        ║
║  │  Tabular + Low-dim → Z-score, IQR        │                        ║
║  │  Tabular + High-dim → Isolation Forest    │                        ║
║  │  Time-series + Seasonal → STL + threshold │                        ║
║  │  Time-series + Streaming → EWMA, CUSUM    │                        ║
║  │  Complex/Images → Autoencoder             │                        ║
║  │  Local clusters → LOF                     │                        ║
║  │  Fraud (labeled) → Supervised + rules     │                        ║
║  └──────────────────────────────────────────┘                        ║
║                                                                      ║
║  PRODUCTION ARCHITECTURE                                             ║
║  ┌──────────────────────────────────────────┐                        ║
║  │  Metrics → Kafka → Detector → Router     │                        ║
║  │                        │         │        │                        ║
║  │                   Model Store  PagerDuty  │                        ║
║  │                        │         │        │                        ║
║  │                   Feedback ← Engineer     │                        ║
║  └──────────────────────────────────────────┘                        ║
║                                                                      ║
║  KEY FORMULAS                                                        ║
║  • Z-score: z = (x - μ) / σ                                         ║
║  • IQR bounds: Q1 - 1.5×IQR, Q3 + 1.5×IQR                          ║
║  • EWMA: μ_t = α·x_t + (1-α)·μ_{t-1}                               ║
║  • Isolation score: s = 2^(-E(h(x))/c(n))                           ║
║  • LOF: density(neighbors) / density(point)                          ║
║  • MAD: median(|x_i - median(x)|)                                   ║
║                                                                      ║
║  ALERT FATIGUE PREVENTION                                            ║
║  ┌──────────────────────────────────────────┐                        ║
║  │  1. Dynamic thresholds (not static)      │                        ║
║  │  2. Multi-signal confirmation            │                        ║
║  │  3. Minimum duration before alert        │                        ║
║  │  4. Correlation-based grouping           │                        ║
║  │  5. Deploy-aware suppression             │                        ║
║  │  6. Engineer feedback loop               │                        ║
║  │  Target: Precision > 80%                 │                        ║
║  └──────────────────────────────────────────┘                        ║
║                                                                      ║
║  FALSE POSITIVE vs FALSE NEGATIVE TRADEOFF                           ║
║  ┌──────────────────────────────────────────┐                        ║
║  │           ← More Sensitive               │                        ║
║  │  FP ████████████████░░░░ High FP Rate    │                        ║
║  │  FN ░░░░░░░░░░░░░░░░████ Low FN Rate    │                        ║
║  │           → More Specific                │                        ║
║  │  FP ░░░░████░░░░░░░░░░░░ Low FP Rate    │                        ║
║  │  FN ░░░░░░░░████████████ High FN Rate   │                        ║
║  │                                          │                        ║
║  │  Fraud: prefer sensitivity (catch fraud)  │                        ║
║  │  Monitoring: prefer specificity (trust)   │                        ║
║  └──────────────────────────────────────────┘                        ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 51 of 51*
