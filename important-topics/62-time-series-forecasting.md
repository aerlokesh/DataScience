# 🎯 Topic 62: Time Series Forecasting

> *"Predicting the future from sequential data — from classical ARIMA to modern ML approaches, explained for interview mastery."*

---

## 📋 Table of Contents

1. [What Makes Time Series Special](#1-what-makes-time-series-special)
2. [Components of Time Series](#2-components-of-time-series)
3. [Stationarity](#3-stationarity)
4. [Decomposition](#4-decomposition)
5. [ARIMA Models](#5-arima-models)
6. [Seasonal Models (SARIMA)](#6-seasonal-models-sarima)
7. [Exponential Smoothing](#7-exponential-smoothing)
8. [ML Approaches to Time Series](#8-ml-approaches-to-time-series)
9. [Prophet and Modern Tools](#9-prophet-and-modern-tools)
10. [Evaluation Metrics](#10-evaluation-metrics)
11. [Cross-Validation for Time Series](#11-cross-validation-for-time-series)
12. [Interview Talking Points](#12-interview-talking-points)
13. [Common Mistakes](#13-common-mistakes)
14. [Rapid-Fire Q&A](#14-rapid-fire-qa)
15. [ASCII Cheat Sheet](#15-ascii-cheat-sheet)

---

## 1. What Makes Time Series Special

### Why Time Series != Regular ML

| Regular ML | Time Series |
|-----------|-------------|
| Observations are independent (i.i.d.) | Observations are dependent (autocorrelated) |
| Random train/test split is fine | Must respect temporal order |
| Features are static | Features evolve over time |
| Cross-validation is straightforward | Need rolling/expanding window |
| No concept of "future" leakage | Future leakage is a critical risk |

### Autocorrelation

The defining feature of time series: each observation is correlated with its past values.

- **ACF (Autocorrelation Function)**: Correlation between yt and yt-k (includes indirect effects)
- **PACF (Partial Autocorrelation Function)**: Direct correlation between yt and yt-k (removes intermediate effects)

> **Critical Insight**: If your data has no autocorrelation, it's essentially random noise — you can't forecast it. The first thing to check in any time series problem is whether meaningful autocorrelation exists. If ACF shows nothing significant, no model will help.

---

## 2. Components of Time Series

### The Four Components

1. **Trend (T)**: Long-term increase or decrease
2. **Seasonality (S)**: Regular, predictable patterns at fixed intervals
3. **Cyclical (C)**: Irregular fluctuations (not fixed period — e.g., business cycles)
4. **Residual/Noise (R)**: Random, unexplained variation

### Trend vs Seasonality vs Cycle

| Component | Period | Predictable | Example |
|-----------|--------|-------------|---------|
| Trend | Long-term | Direction yes, extent no | Company growth over years |
| Seasonality | Fixed (weekly, monthly, yearly) | Yes, highly | Holiday shopping spikes |
| Cycle | Variable (2-10+ years) | Poorly | Economic recessions |
| Noise | Random | No | Day-to-day fluctuations |

### Identifying Components Visually

```
Raw Series:  /\/\/\/\─────/\/\/\/\─────── (seasonal + trend)
Trend Only:  ─────────────────────────── (after removing seasonality)
Season Only: /\/\/\/\/\/\/\/\/\/\/\/\/\/ (fixed pattern repeats)
Residual:    ~~^~v~~^~~v~^~~v~~^~v~~^~~ (random noise)
```

> **Critical Insight**: Seasonality has a FIXED known period (every 7 days, every 12 months). Cycles have VARIABLE unknown periods. This distinction matters because seasonal models (SARIMA, Holt-Winters) need you to specify the period upfront — they can't discover it.

---

## 3. Stationarity

### What is Stationarity?

A time series is stationary if its statistical properties (mean, variance, autocorrelation) don't change over time.

### Types of Stationarity

| Type | Definition | How to Test |
|------|-----------|-------------|
| Strict stationarity | Entire distribution is time-invariant | Impractical to test |
| Weak stationarity | Mean, variance, and autocovariance are constant | ADF test, KPSS test |

### Why Stationarity Matters

- Most classical models (ARIMA) REQUIRE stationarity
- Non-stationary data has unpredictable mean/variance — forecasts diverge
- Statistical tests assume stationarity for valid inference

### Augmented Dickey-Fuller (ADF) Test

```
H0: Series has a unit root (non-stationary)
H1: Series is stationary

If p-value < 0.05 → Reject H0 → Series IS stationary
If p-value >= 0.05 → Fail to reject → Series is NOT stationary
```

### Making Data Stationary

| Technique | Removes | When to Use |
|-----------|---------|-------------|
| Differencing (d=1) | Trend | Linear trend |
| Second differencing (d=2) | Quadratic trend | Curved trend |
| Log transformation | Increasing variance | Variance grows with level |
| Seasonal differencing | Seasonality | yt - yt-m |

> **Critical Insight**: In interviews, explain stationarity as: "A stationary series is one where the rules generating it don't change over time. If you took any random chunk of the data, it would statistically look like any other chunk. This matters because forecasting assumes the patterns we learned from the past still apply to the future."

---

## 4. Decomposition

### Additive vs Multiplicative Decomposition

| Aspect | Additive | Multiplicative |
|--------|----------|----------------|
| Formula | Y = T + S + R | Y = T * S * R |
| When to use | Seasonal amplitude is constant | Seasonal amplitude grows with trend |
| Transform to additive | N/A | Take log(Y) → log(T) + log(S) + log(R) |
| Example | Temperature (same swing year-round) | Retail sales (bigger spikes as business grows) |

### How to Choose

```
If seasonal fluctuations are roughly constant regardless of level → Additive
If seasonal fluctuations are proportional to the level → Multiplicative
Quick test: Plot the data. If the "waves" get bigger over time → Multiplicative
```

### STL Decomposition

- **S**easonal and **T**rend decomposition using **L**oess
- More robust than classical decomposition
- Handles outliers better
- Can handle any type of seasonality
- Seasonal component allowed to change over time

> **Critical Insight**: In practice, many real-world series are multiplicative (sales, web traffic, stock prices). A log transformation converts multiplicative to additive, which is why log-transforming before modeling is so common.

---

## 5. ARIMA Models

### ARIMA(p, d, q) — What Each Parameter Means

| Parameter | Name | Meaning | How to Choose |
|-----------|------|---------|---------------|
| p | AR order | Number of past values used | Look at PACF (significant lags) |
| d | Differencing | Times to difference for stationarity | Number of differences until ADF passes |
| q | MA order | Number of past forecast errors used | Look at ACF (significant lags) |

### Intuition for Each Component

**AR (AutoRegressive) — p**: "Tomorrow's value depends on today's and yesterday's values"
```
yt = c + φ1*yt-1 + φ2*yt-2 + ... + εt
```
Like a weighted average of recent values.

**I (Integrated) — d**: "We difference the data to make it stationary"
```
d=1: y't = yt - yt-1  (first difference)
d=2: y''t = y't - y't-1  (difference of differences)
```

**MA (Moving Average) — q**: "Tomorrow's value adjusts based on recent forecast errors"
```
yt = c + εt + θ1*εt-1 + θ2*εt-2 + ...
```
Like correcting based on how wrong you've been.

### The Box-Jenkins Method (How to Choose p, d, q)

1. **Plot** the data and identify any obvious trend/seasonality
2. **Stationarize**: Apply differencing until ADF test passes → determines d
3. **Examine ACF/PACF** of the stationary series:
   - ACF cuts off at lag q → MA(q)
   - PACF cuts off at lag p → AR(p)
   - Both tail off → ARMA(p,q) — use AIC/BIC to choose
4. **Fit model** and check residuals (should be white noise)
5. **Iterate** if residuals show patterns

### ACF/PACF Pattern Guide

| ACF Pattern | PACF Pattern | Suggested Model |
|-------------|--------------|-----------------|
| Cuts off at lag q | Tails off | MA(q) |
| Tails off | Cuts off at lag p | AR(p) |
| Tails off | Tails off | ARMA(p,q) |
| Cuts off at lag q | Cuts off at lag p | ARMA(p,q) |

> **Critical Insight**: In modern practice, many people just try multiple (p,d,q) combinations and pick the one with lowest AIC/BIC. Auto-ARIMA (like `pmdarima` in Python) automates this. But understanding the ACF/PACF logic shows interviewers you grasp the theory, not just the library.

---

## 6. Seasonal Models (SARIMA)

### SARIMA(p,d,q)(P,D,Q,m)

Extends ARIMA with seasonal components:

| Parameter | Meaning |
|-----------|---------|
| (p,d,q) | Non-seasonal ARIMA parameters |
| (P,D,Q) | Seasonal AR, differencing, MA orders |
| m | Seasonal period (12 for monthly, 7 for daily-weekly, 4 for quarterly) |

### Example: Monthly Sales with Yearly Seasonality

```
SARIMA(1,1,1)(1,1,1,12)

Non-seasonal: AR(1), diff once, MA(1)
Seasonal: Seasonal AR(1), seasonal diff once, seasonal MA(1), period=12 months
```

### When to Use SARIMA vs Other Methods

| Method | Best For | Limitations |
|--------|----------|-------------|
| SARIMA | Single strong seasonality, linear trends | Only one seasonal period, manual tuning |
| Prophet | Multiple seasonalities, holiday effects | Less precise for simple series |
| Holt-Winters | Clear trend + single seasonality | Can't handle complex patterns |
| ML methods | Non-linear patterns, many features | Need more data, harder to interpret |

> **Critical Insight**: SARIMA handles ONE seasonal period. If your data has daily AND weekly AND yearly seasonality (like web traffic), SARIMA alone won't capture it all. You'd need multiple seasonal ARIMA (complicated), or switch to Prophet/ML which handle multiple seasonalities naturally.

---

## 7. Exponential Smoothing

### The Family of Exponential Smoothing Methods

| Method | Components | Formula Concept | Use Case |
|--------|-----------|-----------------|----------|
| **SES** (Simple) | Level only | Weighted average, recent data weighted more | No trend, no seasonality |
| **Holt's** (Double) | Level + Trend | SES + trend component | Trend but no seasonality |
| **Holt-Winters** (Triple) | Level + Trend + Seasonal | Holt's + seasonal component | Trend AND seasonality |

### Simple Exponential Smoothing (SES)

```
Forecast: ŷt+1 = α*yt + (1-α)*ŷt

α (smoothing parameter): 0 < α < 1
- α close to 1: More weight on recent observations (responsive)
- α close to 0: More weight on historical average (smooth)
```

### Holt's Linear Trend

```
Level:    lt = α*yt + (1-α)*(lt-1 + bt-1)
Trend:    bt = β*(lt - lt-1) + (1-β)*bt-1
Forecast: ŷt+h = lt + h*bt
```

### Holt-Winters

**Additive seasonality:**
```
Level:    lt = α*(yt - st-m) + (1-α)*(lt-1 + bt-1)
Trend:    bt = β*(lt - lt-1) + (1-β)*bt-1
Season:   st = γ*(yt - lt) + (1-γ)*st-m
Forecast: ŷt+h = lt + h*bt + st-m+h
```

### Comparison: Exponential Smoothing vs ARIMA

| Aspect | Exponential Smoothing | ARIMA |
|--------|----------------------|-------|
| Approach | Weighted averages | Regression on lags/errors |
| Parameters | Smoothing constants (α,β,γ) | AR/MA orders (p,d,q) |
| Stationarity required | No | Yes (after differencing) |
| Interpretability | Very intuitive | Moderate |
| Flexibility | Limited to trend/season | More flexible patterns |
| Equivalence | Some ETS ≡ some ARIMA | ETS(A,N,N) ≡ ARIMA(0,1,1) |

> **Critical Insight**: Exponential smoothing and ARIMA are NOT competing approaches — they're complementary views. Some ETS models are mathematically equivalent to certain ARIMA models. The choice often comes down to: "Do I think in terms of smoothing/components (ETS) or correlations/errors (ARIMA)?"

---

## 8. ML Approaches to Time Series

### Feature Engineering for Time Series ML

| Feature Type | Examples | Why It Helps |
|-------------|----------|--------------|
| **Lag features** | yt-1, yt-7, yt-30 | Captures autocorrelation |
| **Rolling statistics** | 7-day mean, 30-day std | Captures recent trends/volatility |
| **Date features** | Day of week, month, quarter | Captures seasonality |
| **Holiday indicators** | is_holiday, days_to_christmas | Captures event effects |
| **External features** | Weather, promotions, events | Captures causal factors |

### Common ML Models for Time Series

| Model | Strengths | Weaknesses |
|-------|-----------|------------|
| **XGBoost/LightGBM** | Handles non-linearity, feature importance | Doesn't extrapolate trends |
| **Random Forest** | Robust, handles interactions | Can't predict beyond training range |
| **Linear Regression** | Simple, extrapolates | Can't capture non-linearity |
| **LSTM/GRU** | Learns long-term dependencies | Needs lots of data, slow to train |

### The Extrapolation Problem

```
Training data range: [0, 100]

Tree-based models predict:    max = 100 (can't go beyond!)
Linear models predict:        can extrapolate beyond 100
Neural networks:              depends on architecture

Solution: Use tree models for residuals after detrending,
          or add trend as explicit feature
```

### Recursive vs Direct Multi-Step Forecasting

| Approach | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Recursive** | Predict t+1, use it to predict t+2, etc. | One model, simple | Error accumulates |
| **Direct** | Separate model for each horizon | No error accumulation | Many models to maintain |
| **Multi-output** | One model predicts all horizons | Balanced | Complex architecture |

> **Critical Insight**: ML approaches excel when you have MANY features beyond the time series itself (weather, promotions, events). For pure univariate forecasting with clear patterns, ARIMA/ETS often match or beat ML with far less effort. The power of ML is incorporating external information.

---

## 9. Prophet and Modern Tools

### Facebook Prophet — Key Concepts

**Philosophy**: Decompose into interpretable components that analysts can tune.

```
y(t) = g(t) + s(t) + h(t) + εt

g(t) = growth (linear or logistic)
s(t) = seasonality (Fourier series)
h(t) = holidays/events
εt   = noise
```

### When to Use Prophet

| Good Fit | Poor Fit |
|----------|----------|
| Strong seasonal effects | High-frequency data (seconds/minutes) |
| Multiple seasonalities | Very short series (< 2 full cycles) |
| Historical holidays matter | No clear trend/seasonality |
| Missing data/outliers | Need probabilistic forecasts |
| Non-data-scientist users | Real-time streaming |

### Prophet's Key Strengths

1. **Handles multiple seasonalities** (daily, weekly, yearly) automatically
2. **Robust to missing data** and outliers
3. **Incorporates holiday effects** with custom holiday calendars
4. **Changepoints**: Automatically detects where trends change
5. **Analyst-friendly**: Interpretable components, easy parameter tuning

### Modern Alternatives

| Tool | Best For | Key Feature |
|------|----------|-------------|
| Prophet | Business forecasting, multiple seasonalities | Interpretable decomposition |
| NeuralProphet | Prophet + deep learning | Neural network components |
| DeepAR (Amazon) | Many related time series | Probabilistic, learns across series |
| N-BEATS | Pure univariate forecasting | State-of-art accuracy |
| Temporal Fusion Transformers | Complex multi-horizon | Attention-based, interpretable |

> **Critical Insight**: Prophet is fantastic for "good enough" forecasts with minimal tuning — perfect for business contexts where you need interpretable results fast. But for competition-level accuracy or very specific patterns, custom models (ARIMA or ML) often win with proper tuning.

---

## 10. Evaluation Metrics

### Metric Comparison Table

| Metric | Formula | Pros | Cons | Best For |
|--------|---------|------|------|----------|
| **MAE** | mean(\|actual - predicted\|) | Interpretable, same units | Doesn't penalize large errors | When all errors equally bad |
| **RMSE** | sqrt(mean(error²)) | Penalizes large errors | Sensitive to outliers | When large errors are costly |
| **MAPE** | mean(\|error/actual\|)*100 | Percentage, scale-free | Undefined for zero values, asymmetric | Comparing across different scales |
| **SMAPE** | Symmetric MAPE | Handles zeros better | Still somewhat asymmetric | Alternative to MAPE |
| **MASE** | MAE / naive_forecast_MAE | Scale-free, handles zeros | Less intuitive | Comparing across series |

### When to Use Which

```
Same scale, care about big errors?     → RMSE
Same scale, all errors equal?          → MAE
Comparing across different scales?     → MAPE (if no zeros) or MASE
Intermittent demand (many zeros)?      → MASE
Business reporting?                    → MAPE (people understand %)
Model selection?                       → AIC/BIC (penalizes complexity)
```

### The Naive Benchmark

Always compare your model against naive forecasts:
- **Naive**: ŷt+1 = yt (tomorrow = today)
- **Seasonal Naive**: ŷt+1 = yt-m (same day last week/year)
- **Drift**: Project the overall trend forward

> **Critical Insight**: If your fancy model can't beat the seasonal naive (yesterday's same weekday), it's not adding value. In interviews, ALWAYS mention comparing against a baseline. "My ARIMA model achieved 12% MAPE vs 18% for seasonal naive" shows practical thinking.

---

## 11. Cross-Validation for Time Series

### Why Random Splits Fail

```
Random K-Fold:          Time Series CV:
┌─┬─┬─┬─┬─┬─┬─┬─┐     ┌─┬─┬─┬─┬─┬─┬─┬─┐
│T│V│T│T│V│T│T│T│     │T│T│T│T│V│ │ │ │  Fold 1
└─┴─┴─┴─┴─┴─┴─┴─┘     │T│T│T│T│T│V│ │ │  Fold 2
                        │T│T│T│T│T│T│V│ │  Fold 3
PROBLEM: Using future    │T│T│T│T│T│T│T│V│  Fold 4
data to predict past!    └─┴─┴─┴─┴─┴─┴─┴─┘
                        
                        CORRECT: Always train on
                        past, validate on future
```

### Time Series CV Strategies

| Strategy | Description | When to Use |
|----------|-------------|-------------|
| **Expanding Window** | Train grows each fold | Default, most data for training |
| **Sliding Window** | Fixed-size training window | When old data becomes irrelevant |
| **Blocked** | Gap between train and test | When recent data leaks patterns |

### Expanding Window Example

```
Fold 1: [Train: Jan-Jun    ] [Test: Jul    ]
Fold 2: [Train: Jan-Jul    ] [Test: Aug    ]
Fold 3: [Train: Jan-Aug    ] [Test: Sep    ]
Fold 4: [Train: Jan-Sep    ] [Test: Oct    ]
Fold 5: [Train: Jan-Oct    ] [Test: Nov    ]

Average metrics across folds for final evaluation
```

### Common Leakage Pitfalls

| Pitfall | Why It Leaks | Fix |
|---------|-------------|-----|
| Using future values in features | Direct leakage | Only use past lags |
| Global normalization | Future stats inform past | Normalize with expanding window |
| Feature selection on all data | Learned from future | Select features on train only |
| Filling missing with overall mean | Future data contributes | Forward-fill or train-only stats |

> **Critical Insight**: In interviews, if someone asks "how would you validate a time series model?" and you say "random 80/20 split," it's an instant red flag. Time series requires TEMPORAL splitting. Always train on the past, test on the future. This is non-negotiable.

---

## 12. Interview Talking Points

### "How would you forecast demand for next quarter?"

> "First, I'd examine the historical data for trend, seasonality, and any structural changes. I'd check stationarity with an ADF test and look at the autocorrelation structure. For a business forecasting problem with clear seasonality, I'd start with SARIMA or Holt-Winters as a baseline, then try Prophet if there are multiple seasonal patterns or holiday effects. I'd engineer external features like marketing spend or economic indicators and try gradient boosting. I'd validate using expanding-window time series CV, never random splits. I'd compare all models against a seasonal naive baseline using MAPE, and I'd provide prediction intervals, not just point forecasts — decision-makers need to know the uncertainty."

### "How would you detect seasonality in user signups?"

> "Multiple approaches: Visually, I'd plot the data and look for repeating patterns. Statistically, I'd examine the ACF plot — peaks at regular lags indicate seasonality (lag 7 for weekly, lag 30 for monthly). I'd also use STL decomposition to isolate the seasonal component and verify its strength. For formal testing, I could use a Fourier analysis or the Kruskal-Wallis test across time periods. If I find seasonality, I'd quantify its strength using the seasonal subseries approach — comparing the variation WITHIN seasons vs BETWEEN seasons."

### "Your model's MAPE is 15% — is that good?"

> "It depends entirely on context. For volatile stock prices, 15% might be excellent. For stable utility demand, it might be poor. I'd compare against: (1) The naive forecast — if that's 20%, our model adds value; (2) Business requirements — if decisions need ±5% accuracy, 15% isn't sufficient; (3) Forecast horizon — accuracy degrades with longer horizons, so 15% for 90-day ahead might be great but poor for next-day. I'd also check if errors are biased (consistently over/under-predicting) vs random."

---

## 13. Common Mistakes

| ❌ Wrong | ✅ Right |
|----------|----------|
| Random train/test split for time series | Temporal split (train on past, test on future) |
| Applying ARIMA to non-stationary data | Difference first until stationary, then model |
| Using MAPE with zero-valued actuals | Use MASE or MAE for intermittent demand |
| Ignoring seasonality period | Identify correct m (7 for weekly, 12 for monthly) |
| Forecasting far beyond training range with trees | Detrend first or use models that extrapolate |
| Not comparing against naive baseline | Always benchmark against seasonal naive |
| Using future info in feature engineering | Only use information available at prediction time |
| Over-differencing (d too high) | Stop when ADF test passes; over-differencing adds noise |
| Assuming stationarity without testing | Always run ADF test or visual inspection |
| Reporting only point forecasts | Include prediction intervals for decision-making |

---

## 14. Rapid-Fire Q&A

**Q1: What's the difference between AR and MA components?**
A: AR uses past values of the series itself (yt depends on yt-1). MA uses past forecast errors (yt depends on how wrong we were at t-1). AR captures momentum; MA captures shock effects.

**Q2: How do you handle multiple seasonalities?**
A: Use Prophet (handles daily+weekly+yearly), Fourier terms as features in regression, or TBATS model. SARIMA only handles one seasonal period.

**Q3: What's the difference between trend and drift?**
A: Trend is a deterministic long-term direction. Drift is a stochastic trend (random walk with upward/downward tendency). Deterministic trends are predictable; stochastic trends are not.

**Q4: When would you NOT use time series methods?**
A: When observations are truly independent (no autocorrelation), when you have rich external features that explain the target better than history, or when the series is too short to identify patterns.

**Q5: How do you handle missing values in time series?**
A: Forward fill (last known value), interpolation (linear or spline), seasonal decomposition then impute, or model-based imputation. NEVER use future values to fill past gaps.

**Q6: What is a random walk and why does it matter?**
A: yt = yt-1 + noise. It's non-stationary, unpredictable, and the best forecast is just the last value. Many financial series approximate random walks — if your series IS one, no model beats naive.

**Q7: How does Prophet handle changepoints?**
A: Prophet places potential changepoints throughout the history and uses regularization (Laplace prior) to determine which ones are significant. It allows the trend to change slope at these points.

**Q8: What's the difference between forecasting and prediction?**
A: Forecasting specifically predicts future values based on temporal patterns. Prediction is broader — any estimation of unknown values. All forecasting is prediction, but not all prediction is forecasting.

**Q9: How do you handle non-stationary variance?**
A: Log transformation (stabilizes multiplicative variance), Box-Cox transformation (finds optimal power), or GARCH models (explicitly model changing variance). Choose based on the pattern.

**Q10: What's the best model for time series?**
A: No universal best. ARIMA/ETS for univariate with clear patterns, XGBoost for feature-rich problems, Prophet for multiple seasonalities with minimal tuning, LSTMs for very complex long-range dependencies with lots of data. Always benchmark against naive.

---

## 15. ASCII Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                TIME SERIES FORECASTING CHEAT SHEET                   ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  TIME SERIES COMPONENTS:                                             ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │ Y(t) = Trend + Seasonality + Cycle + Noise  (Additive)     │    ║
║  │ Y(t) = Trend × Seasonality × Cycle × Noise  (Multiplicat) │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  MODEL SELECTION FLOWCHART:                                          ║
║                                                                      ║
║  Is data stationary?                                                 ║
║  ├── No → Difference (d times) until stationary                     ║
║  └── Yes → Check ACF/PACF                                           ║
║       ├── ACF cuts off → MA(q)                                       ║
║       ├── PACF cuts off → AR(p)                                      ║
║       └── Both tail off → ARMA(p,q) (use AIC)                       ║
║                                                                      ║
║  Has seasonality?                                                    ║
║  ├── Single season → SARIMA or Holt-Winters                         ║
║  ├── Multiple seasons → Prophet or Fourier features                  ║
║  └── No season → ARIMA or ETS                                        ║
║                                                                      ║
║  ARIMA(p,d,q) PARAMETERS:                                           ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │ p = AR order (PACF cutoff)   → "memory of values"          │    ║
║  │ d = differencing order       → "times to make stationary"   │    ║
║  │ q = MA order (ACF cutoff)    → "memory of errors"          │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  EXPONENTIAL SMOOTHING FAMILY:                                       ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │ SES:          Level only          (α)                       │    ║
║  │ Holt:         Level + Trend       (α, β)                    │    ║
║  │ Holt-Winters: Level + Trend + Season  (α, β, γ)            │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  CROSS-VALIDATION (NEVER RANDOM SPLIT!):                             ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │ Fold 1: [TRAIN TRAIN TRAIN] [TEST]                          │    ║
║  │ Fold 2: [TRAIN TRAIN TRAIN TRAIN] [TEST]                    │    ║
║  │ Fold 3: [TRAIN TRAIN TRAIN TRAIN TRAIN] [TEST]              │    ║
║  │                                                             │    ║
║  │ KEY: Always train on PAST, test on FUTURE                   │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  STATIONARITY CHECK:                                                 ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │ 1. Visual: Does mean/variance change over time?             │    ║
║  │ 2. ADF Test: p < 0.05 → stationary                         │    ║
║  │ 3. If not stationary → difference → retest                  │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  EVALUATION METRICS:                                                 ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │ MAE:  Mean |error|        → interpretable, same units       │    ║
║  │ RMSE: √Mean(error²)      → penalizes big errors            │    ║
║  │ MAPE: Mean |error/actual| → percentage, scale-free          │    ║
║  │ MASE: MAE / naive MAE    → relative to baseline             │    ║
║  │                                                             │    ║
║  │ ALWAYS compare vs seasonal naive baseline!                  │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  FEATURE ENGINEERING FOR ML:                                         ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │ Lags:     yt-1, yt-7, yt-30                                 │    ║
║  │ Rolling:  mean_7d, std_30d, min_7d                          │    ║
║  │ Calendar: day_of_week, month, is_weekend, is_holiday        │    ║
║  │ External: weather, promotions, events                       │    ║
║  │                                                             │    ║
║  │ WARNING: Only use PAST information! No future leakage!      │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 62 of 65*
