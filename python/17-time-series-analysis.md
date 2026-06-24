# 🎯 Topic 17: Time Series Analysis

> **Data Science Interview — Time Series Forecasting & Analysis Deep Dive**  
> From stationarity tests to ARIMA model selection, exponential smoothing, and ML-based forecasting — with interview talking points, Python code, and the intuition interviewers expect from a senior DS/DA at Google, Meta, Amazon, Netflix, Spotify, or Uber.

---

## Table of Contents

1. [Decomposition: Trend, Seasonality, Residual](#decomposition-trend-seasonality-residual)
2. [Stationarity & the ADF Test](#stationarity--the-adf-test)
3. [ACF & PACF Interpretation](#acf--pacf-interpretation)
4. [ARIMA & SARIMA Model Selection](#arima--sarima-model-selection)
5. [Exponential Smoothing Methods](#exponential-smoothing-methods)
6. [Lag Features for ML Models](#lag-features-for-ml-models)
7. [Resampling & Frequency Conversion](#resampling--frequency-conversion)
8. [Evaluation Metrics (MAE, MAPE, RMSE)](#evaluation-metrics-mae-mape-rmse)
9. [Model Comparison: ARIMA vs Prophet vs LSTM](#model-comparison-arima-vs-prophet-vs-lstm)
10. [Interview Talking Points & Scripts](#interview-talking-points--scripts)
11. [Common Interview Mistakes](#common-interview-mistakes)
12. [Rapid-Fire Q&A (Top 12)](#rapid-fire-qa-top-12)
13. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## Decomposition: Trend, Seasonality, Residual

| Model | Formula | When to Use |
|-------|---------|-------------|
| Additive | Y(t) = Trend + Seasonal + Residual | Seasonal amplitude is constant over time |
| Multiplicative | Y(t) = Trend * Seasonal * Residual | Seasonal amplitude grows with the level |

> **Critical Insight:** If seasonal swings grow proportionally with the level (e.g., retail sales doubling means Christmas spike doubles), use multiplicative. A log transform converts multiplicative into additive.

```python
import pandas as pd
import numpy as np
from statsmodels.tsa.seasonal import seasonal_decompose, STL

# Generate sample data
np.random.seed(42)
dates = pd.date_range('2020-01-01', periods=365*3, freq='D')
trend = np.linspace(100, 200, len(dates))
seasonal = 20 * np.sin(2 * np.pi * np.arange(len(dates)) / 365)
ts = pd.Series(trend + seasonal + np.random.normal(0, 5, len(dates)), index=dates)

# Classical decomposition
result = seasonal_decompose(ts, model='additive', period=365)
result.plot()

# STL (preferred — robust to outliers, allows evolving seasonality)
stl = STL(ts, period=365, robust=True)
res = stl.fit()  # res.trend, res.seasonal, res.resid
```

---

## Stationarity & the ADF Test

A stationary series has **constant mean, constant variance, and autocorrelation that depends only on lag** (not time).

| Type of Non-Stationarity | Cause | Fix |
| ------------------------ | ----- | --- |
| Trend (unit root) | Stochastic trend | Differencing (d=1) |
| Seasonal | Repeating patterns | Seasonal differencing (D=1) |
| Variance | Heteroscedasticity | Log or Box-Cox transform |

```python
from statsmodels.tsa.stattools import adfuller, kpss

def check_stationarity(series, name="Series"):
    """Run ADF test and interpret results."""
    result = adfuller(series.dropna(), autolag='AIC')
    print(f"ADF Test [{name}]: stat={result[0]:.4f}, p={result[1]:.6f}")
    if result[1] < 0.05:
        print("  -> Stationary (reject null of unit root)")
    else:
        print("  -> NOT stationary (fail to reject)")
    return result[1] < 0.05

# ADF: H0 = unit root (non-stationary). Low p -> stationary.
check_stationarity(ts, "Original")
ts_diff = ts.diff().dropna()
check_stationarity(ts_diff, "First Difference")

# KPSS complement: H0 = stationary. Low p -> NOT stationary.
stat, p_value, _, _ = kpss(ts_diff.dropna(), regression='c')
print(f"KPSS: stat={stat:.4f}, p={p_value:.4f}")
```

> **Critical Insight:** ADF null is "unit root exists" (non-stationary), so low p-value means stationary. KPSS null is the opposite — "series IS stationary." Always run both: if ADF rejects and KPSS fails to reject, you're confident in stationarity.

---

## ACF & PACF Interpretation

| Pattern | ACF Behavior | PACF Behavior | Model Suggested |
|---------|-------------|---------------|-----------------|
| AR(p) | Tails off (decays) | Cuts off after lag p | Use p from PACF cutoff |
| MA(q) | Cuts off after lag q | Tails off (decays) | Use q from ACF cutoff |
| ARMA(p,q) | Tails off | Tails off | Use AIC/BIC to select |
| Non-stationary | Very slow decay | Large spike at lag 1 | Difference first! |

```python
from statsmodels.graphics.tsaplots import plot_acf, plot_pacf

fig, axes = plt.subplots(1, 2, figsize=(14, 4))
plot_acf(ts_diff, lags=40, ax=axes[0], title="ACF")
plot_pacf(ts_diff, lags=40, ax=axes[1], title="PACF")
plt.tight_layout()
```

> **Critical Insight:** "Cuts off" = drops within confidence bands after a specific lag. "Tails off" = gradually decays. PACF cutoff at lag 2 -> AR(2). ACF cutoff at lag 1 -> MA(1). When both tail off, rely on information criteria.

---

## ARIMA & SARIMA Model Selection

| Parameter | Meaning | How to Determine |
|-----------|---------|-----------------|
| p | AR order | PACF cutoff |
| d | Differencing | # differences for stationarity (usually 0-2) |
| q | MA order | ACF cutoff |
| (P,D,Q,m) | Seasonal AR, diff, MA, period | Seasonal ACF/PACF + domain knowledge |

```python
from statsmodels.tsa.statespace.sarimax import SARIMAX
import pmdarima as pm

monthly_ts = ts.resample('M').mean()

# Manual SARIMA fit
model = SARIMAX(monthly_ts, order=(1,1,1), seasonal_order=(1,1,1,12))
results = model.fit(disp=False)
forecast = results.get_forecast(steps=12)
print(results.summary().tables[1])

# Automated selection with pmdarima
auto_model = pm.auto_arima(
    monthly_ts, seasonal=True, m=12,
    start_p=0, max_p=3, start_q=0, max_q=3,
    d=None, D=None,       # Auto-determine differencing
    stepwise=True, trace=True,
    information_criterion='aic'
)
print(auto_model.summary())
```

> **Critical Insight:** Use `auto_arima` for speed but always validate with residual diagnostics — residuals should be white noise. Check with `results.plot_diagnostics()` and the Ljung-Box test (p > 0.05 means no remaining autocorrelation).

---

## Exponential Smoothing Methods

| Method | Trend | Seasonality | Parameters | Best For |
|--------|-------|-------------|------------|----------|
| SES | No | No | alpha | Short-term, no trend/season |
| Holt's Linear | Yes | No | alpha, beta | Trending data, no season |
| Holt-Winters Additive | Yes | Additive | alpha, beta, gamma | Constant seasonal amplitude |
| Holt-Winters Multiplicative | Yes | Multiplicative | alpha, beta, gamma | Growing seasonal amplitude |
| Damped Trend | Yes (damped) | Optional | alpha, beta, phi | Long-horizon forecasts |

| Parameter | Controls | High Value Means |
| --------- | -------- | ---------------- |
| alpha | Level smoothing | More weight to recent values |
| beta | Trend smoothing | More responsive trend |
| gamma | Seasonal smoothing | Seasonal pattern adapts quickly |
| phi | Trend damping (0.8-1) | Less damping (phi=1 = no damping) |

```python
from statsmodels.tsa.holtwinters import ExponentialSmoothing, SimpleExpSmoothing

ses = SimpleExpSmoothing(monthly_ts).fit(smoothing_level=0.3, optimized=False)
holt = ExponentialSmoothing(monthly_ts, trend='add', seasonal=None).fit()
hw = ExponentialSmoothing(monthly_ts, trend='add', seasonal='mul',
                          seasonal_periods=12).fit()
damped = ExponentialSmoothing(monthly_ts, trend='add', damped_trend=True,
                              seasonal='add', seasonal_periods=12).fit()

print(f"HW params: alpha={hw.params['smoothing_level']:.3f}, "
      f"beta={hw.params['smoothing_trend']:.3f}, "
      f"gamma={hw.params['smoothing_seasonal']:.3f}")
```

> **Critical Insight:** For long-horizon forecasts, always use a **damped trend**. Without damping, Holt's method projects trend linearly forever. The M3/M4 competitions showed damped trends consistently outperform non-damped alternatives.

---

## Lag Features for ML Models

```python
from sklearn.ensemble import GradientBoostingRegressor
from sklearn.model_selection import TimeSeriesSplit
from sklearn.metrics import mean_absolute_error

def create_lag_features(df, target_col, lags=[1,2,3,7,14,28]):
    """Create lag features, rolling stats, and calendar features."""
    result = df.copy()
    for lag in lags:
        result[f'lag_{lag}'] = result[target_col].shift(lag)
    for window in [7, 14, 30]:
        result[f'rolling_mean_{window}'] = result[target_col].shift(1).rolling(window).mean()
        result[f'rolling_std_{window}'] = result[target_col].shift(1).rolling(window).std()
    # Calendar + cyclical encoding
    result['day_of_week'] = result.index.dayofweek
    result['month'] = result.index.month
    result['month_sin'] = np.sin(2 * np.pi * result['month'] / 12)
    result['month_cos'] = np.cos(2 * np.pi * result['month'] / 12)
    return result.dropna()

# Time-based split (NEVER random!)
df_feat = create_lag_features(pd.DataFrame({'value': ts}, index=ts.index), 'value')
train, test = df_feat.loc[:'2022-06-30'], df_feat.loc['2022-07-01':]
model = GradientBoostingRegressor(n_estimators=200, max_depth=5)
model.fit(train.drop('value', axis=1), train['value'])
preds = model.predict(test.drop('value', axis=1))

# Walk-forward cross-validation
tscv = TimeSeriesSplit(n_splits=5)
for fold, (tr_idx, te_idx) in enumerate(tscv.split(df_feat)):
    X_tr, X_te = df_feat.iloc[tr_idx], df_feat.iloc[te_idx]
    # fit/predict per fold...
```

> **Critical Insight:** Always use `shift(1)` before rolling computations to avoid **target leakage**. The rolling mean at time t must only use data up to t-1. This is the most common mistake in time series ML pipelines.

---

## Resampling & Frequency Conversion

| Method | Use Case | Example |
|--------|----------|---------|
| `.mean()` | Averaging readings | Temperature |
| `.sum()` | Accumulating totals | Revenue |
| `.last()` | Point-in-time snapshots | Stock price |
| `.ohlc()` | Open/High/Low/Close | Financial |

```python
weekly = ts.resample('W').mean()          # Downsample
monthly_sum = ts.resample('M').sum()       # Total per month
upsampled = monthly_sum.resample('D').interpolate(method='linear')  # Upsample
ffilled = monthly_sum.resample('D').ffill()  # Step-wise fill
```

> **Critical Insight:** Match aggregation to business semantics. Using `.mean()` on revenue loses totals — use `.sum()`. Using `.sum()` on temperature is nonsensical — use `.mean()`.

---

## Evaluation Metrics (MAE, MAPE, RMSE)

| Metric | Formula | Pros | Cons |
|--------|---------|------|------|
| MAE | mean(\|y - y_hat\|) | Interpretable, robust to outliers | No scale-independence |
| RMSE | sqrt(mean((y-y_hat)^2)) | Penalizes large errors | Sensitive to outliers |
| MAPE | mean(\|y-y_hat\|/\|y\|)*100 | Scale-independent (%) | Undefined at y=0, asymmetric |
| sMAPE | mean(2\|y-y_hat\| / (\|y\|+\|y_hat\|))*100 | Bounded | Still asymmetric |
| MASE | MAE / naive_MAE | Scale-free, symmetric | Less intuitive |

```python
def forecast_metrics(y_true, y_pred, y_train=None):
    y_true, y_pred = np.array(y_true), np.array(y_pred)
    mae = np.mean(np.abs(y_true - y_pred))
    rmse = np.sqrt(np.mean((y_true - y_pred)**2))
    mask = y_true != 0
    mape = np.mean(np.abs((y_true[mask] - y_pred[mask]) / y_true[mask])) * 100
    mase = mae / np.mean(np.abs(np.diff(y_train))) if y_train is not None else None
    print(f"MAE={mae:.2f}, RMSE={rmse:.2f}, MAPE={mape:.1f}%, MASE={mase:.3f}")
    return {'MAE': mae, 'RMSE': rmse, 'MAPE': mape, 'MASE': mase}
```

> **Critical Insight:** MAPE is the most requested business metric but fails at zero and penalizes over-forecasts more than under-forecasts. For intermittent demand (many zeros), use **MASE**. Always state this tradeoff in interviews.

---

## Model Comparison: ARIMA vs Prophet vs LSTM

| Criteria | ARIMA/SARIMA | Prophet (Meta) | LSTM/Deep Learning |
|----------|-------------|----------------|-------------------|
| Data needed | Small (50+) | Medium (2+ yrs daily) | Large (1000s+) |
| Multiple seasonality | Limited | Excellent | Excellent |
| Exogenous variables | SARIMAX | Regressors | Multivariate input |
| Interpretability | High | High | Low |
| Tuning effort | Moderate | Low | High |
| Uncertainty | Built-in CI | Built-in CI | MC dropout/ensemble |
| Missing data | No | Yes | Requires imputation |
| Best for | Single series | Holidays, business | Complex patterns at scale |

| Scenario | Recommended Approach |
|----------|---------------------|
| Quick baseline | ARIMA + exponential smoothing |
| Business forecasting with holidays | Prophet |
| Thousands of SKUs at scale | LightGBM with lag features |
| Complex multivariate | LSTM / Transformer |
| Anomaly detection | STL + thresholding |
| Intermittent demand | Croston's method |

---

## Interview Talking Points & Scripts

### "How would you forecast demand?"

> *"I'd approach demand forecasting in structured stages. First, EDA — plot the series, identify trend, seasonality, outliers. Run ADF to check stationarity. For a single product with clear seasonality, SARIMA or Holt-Winters gives strong baselines. But in production at scale — 10,000 SKUs — I'd use LightGBM with engineered lag features, rolling statistics, and calendar features because it handles non-linearity and scales across products. I'd validate with walk-forward CV using MASE as my primary metric and always compare against a naive seasonal baseline to quantify actual lift."*

### "How do you check stationarity?"

> *"I combine visual inspection with statistical tests. Visually, I plot rolling mean and standard deviation — if they drift, the series isn't stationary. Statistically, I run both ADF and KPSS. ADF's null is 'unit root exists' (non-stationary); KPSS's null is 'series is stationary.' If ADF rejects and KPSS doesn't, I'm confident in stationarity. If non-stationary, I apply differencing (d=1 usually suffices). For variance non-stationarity, I log-transform first. I keep differencing minimal to avoid overdifferencing."*

### "How do you handle multiple seasonalities?"

> *"SARIMA handles one seasonal period. For multiple seasonalities — daily data with weekly AND yearly patterns — I'd use Prophet which models seasonality as Fourier series. With ML models, I encode each cycle using sin/cos: sin(2*pi*day/7) for weekly. This cyclical encoding avoids the discontinuity problem where hour 23 and hour 0 appear far apart as integers."*

---

## Common Interview Mistakes

| # | Mistake ❌ | Correction ✅ |
|---|-----------|--------------|
| 1 | Using random k-fold CV for time series | Walk-forward or expanding window CV only |
| 2 | Not checking stationarity before ARIMA | Always run ADF/KPSS first |
| 3 | Confusing ACF with PACF | PACF cutoff -> p; ACF cutoff -> q |
| 4 | Using MAPE when actuals contain zeros | Use MASE or sMAPE instead |
| 5 | Forgetting shift(1) before rolling stats | Always shift to prevent target leakage |
| 6 | Over-differencing (d>2) | Check if ADF passes; d=1 usually enough |
| 7 | Not comparing against naive baseline | Naive seasonal is surprisingly hard to beat |
| 8 | Ignoring residual diagnostics | Check Ljung-Box p > 0.05 for white noise |
| 9 | Individual ARIMA for 1000s of series | Use global ML models for scale |
| 10 | Additive decomposition when variance grows | Log-transform first or use multiplicative |

---

## Rapid-Fire Q&A (Top 12)

**Q1: What makes a time series stationary?**  
Constant mean, constant variance, autocovariance depends only on lag distance.

**Q2: What does 'd' in ARIMA(p,d,q) represent?**  
Differencing order: d=1 means y'(t) = y(t) - y(t-1).

**Q3: If ACF decays very slowly, what does that indicate?**  
Non-stationarity — difference the series before modeling.

**Q4: How does exponential smoothing differ from moving average?**  
ETS uses exponentially decaying weights (all history); MA gives equal weight to last k points.

**Q5: When would you use MASE over MAPE?**  
When actuals contain zeros, when comparing across different-scale series, or when you need symmetry.

**Q6: Seasonal period 'm' for daily data with weekly patterns?**  
m=7. Monthly with yearly: m=12. Hourly with daily: m=24.

**Q7: Why log-transform before differencing?**  
Stabilizes variance first; differencing only fixes the mean.

**Q8: AR(1) vs random walk?**  
AR(1): |phi| < 1, stationary, mean-reverting. Random walk: phi=1, non-stationary, never reverts.

**Q9: How detect seasonality in ML lag features?**  
Include seasonal lags (lag_7, lag_365), cyclical sin/cos encodings, and seasonal rolling stats.

**Q10: What does the Ljung-Box test check?**  
Whether residuals have significant autocorrelation. p < 0.05 = model missed structure.

**Q11: ARIMA for multivariate data?**  
ARIMA is univariate. Use SARIMAX for exogenous regressors, VAR for multiple endogenous series.

**Q12: How does Prophet handle holidays?**  
As binary regressors with estimated effect sizes, including before/after window effects.

---

## Summary Cheat Sheet

```
+------------------------------------------------------------------+
|               TIME SERIES ANALYSIS CHEAT SHEET                    |
+------------------------------------------------------------------+
|                                                                    |
|  WORKFLOW:                                                         |
|  1. Plot series -> identify trend, seasonality, outliers          |
|  2. Check stationarity (ADF + KPSS)                              |
|  3. Transform if needed (log, Box-Cox, differencing)              |
|  4. Examine ACF/PACF -> identify p, q orders                     |
|  5. Fit model (ARIMA, ETS, or ML with lag features)              |
|  6. Check residuals (Ljung-Box, QQ-plot)                         |
|  7. Validate with walk-forward CV                                 |
|  8. Compare vs naive baseline using MASE                          |
|                                                                    |
+------------------------------------------------------------------+
|  KEY TESTS:                                                        |
|  ADF:  H0 = unit root exists     -> low p = stationary           |
|  KPSS: H0 = series is stationary -> low p = NOT stationary       |
|  Ljung-Box: H0 = no autocorrelation in residuals                 |
+------------------------------------------------------------------+
|  MODEL SELECTION:                                                  |
|  - PACF cuts off at lag p -> AR(p)                                |
|  - ACF cuts off at lag q  -> MA(q)                                |
|  - Both tail off -> ARMA, use AIC/BIC                            |
|  - Seasonal spikes at lag m, 2m -> add (P,D,Q,m)                 |
+------------------------------------------------------------------+
|  EXPONENTIAL SMOOTHING:                                           |
|  - No trend, no season  -> SES (alpha)                           |
|  - Trend, no season     -> Holt (alpha, beta)                    |
|  - Trend + season       -> Holt-Winters (alpha, beta, gamma)     |
|  - Long horizon         -> Add damping (phi)                      |
+------------------------------------------------------------------+
|  AT SCALE (1000s+ series):                                        |
|  - Global ML model (LightGBM) with lag features                  |
|  - Never random split. Always shift before rolling.               |
|  - Walk-forward CV, MASE metric, compare vs seasonal naive       |
+------------------------------------------------------------------+
|  INTERVIEW POWER PHRASES:                                         |
|  "Check stationarity first with ADF and KPSS together"           |
|  "Always compare against a naive seasonal baseline"               |
|  "For scale, use global models with cross-series features"        |
|  "Walk-forward validation prevents future data leakage"           |
|  "MAPE fails at zero; prefer MASE for heterogeneous series"      |
+------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 17 of 25*
