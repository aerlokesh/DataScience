# 🎯 Topic 24: Model Deployment & Productionization

> **Scope:** Everything that happens after `model.fit()` — serialization formats, serving infrastructure, containerization, experiment tracking, feature stores, monitoring, A/B testing, CI/CD for ML, and scalability patterns. Targeted at DS/DA candidates with ~5 YOE interviewing at Google, Meta, Amazon, Netflix, Spotify, and Uber.

---

## Table of Contents

1. [Saving Models: Pickle vs Joblib vs ONNX](#1-saving-models-pickle-vs-joblib-vs-onnx)
2. [Flask & FastAPI Serving](#2-flask--fastapi-serving)
3. [Docker Basics for ML](#3-docker-basics-for-ml)
4. [MLflow Experiment Tracking](#4-mlflow-experiment-tracking)
5. [Feature Stores](#5-feature-stores)
6. [Model Monitoring](#6-model-monitoring)
7. [Batch vs Real-Time Inference](#7-batch-vs-real-time-inference)
8. [A/B Testing Models in Production](#8-ab-testing-models-in-production)
9. [CI/CD for ML](#9-cicd-for-ml)
10. [Scalability Considerations](#10-scalability-considerations)
11. [Interview Talking Points & Scripts](#11-interview-talking-points--scripts)
12. [Common Interview Mistakes](#12-common-interview-mistakes)
13. [Rapid-Fire Q&A](#13-rapid-fire-qa)
14. [Summary Cheat Sheet](#14-summary-cheat-sheet)

---

## 1. Saving Models: Pickle vs Joblib vs ONNX

### Format Comparison

| Feature | Pickle | Joblib | ONNX |
|---------|--------|--------|------|
| **File size** | Large (no compression) | Smaller (compressed NumPy arrays) | Compact (optimized graph) |
| **Speed** | Moderate | Fast for large arrays | Very fast inference |
| **Language support** | Python only | Python only | Cross-language (C++, Java, JS) |
| **Framework agnostic** | No | No | Yes |
| **Security** | Vulnerable to arbitrary code execution | Same as pickle | Safer (no code execution) |
| **Versioning risk** | High (Python/sklearn version coupling) | High (same as pickle) | Low (standardized spec) |
| **Best for** | Quick prototyping | Sklearn models with large arrays | Production cross-platform serving |

> **Critical Insight:** Never use pickle/joblib for models that will be served in production across different environments. ONNX provides framework-agnostic inference, enabling you to train in Python (sklearn, PyTorch, TensorFlow) but serve in optimized C++ runtimes with 2-10x speed improvements.

### Code Examples

```python
import pickle
import joblib
import onnx
from skl2onnx import convert_sklearn
from skl2onnx.common.data_types import FloatTensorType
from sklearn.ensemble import RandomForestClassifier
import numpy as np

# Train a model
X_train = np.random.rand(1000, 10).astype(np.float32)
y_train = np.random.randint(0, 2, 1000)
model = RandomForestClassifier(n_estimators=100)
model.fit(X_train, y_train)

# --- Method 1: Pickle ---
with open("model.pkl", "wb") as f:
    pickle.dump(model, f)

with open("model.pkl", "rb") as f:
    loaded_model = pickle.load(f)

# --- Method 2: Joblib (preferred for sklearn) ---
joblib.dump(model, "model.joblib", compress=3)
loaded_model = joblib.load("model.joblib")

# --- Method 3: ONNX (production-grade) ---
initial_type = [("float_input", FloatTensorType([None, 10]))]
onnx_model = convert_sklearn(model, initial_types=initial_type)
onnx.save_model(onnx_model, "model.onnx")

# Inference with ONNX Runtime
import onnxruntime as rt
session = rt.InferenceSession("model.onnx")
input_name = session.get_inputs()[0].name
predictions = session.run(None, {input_name: X_train[:5]})[0]
```

---

## 2. Flask & FastAPI Serving

### Framework Comparison

| Aspect | Flask | FastAPI |
|--------|-------|---------|
| **Async support** | No (requires extensions) | Native async/await |
| **Auto documentation** | No | Swagger UI built-in |
| **Type validation** | Manual | Pydantic models |
| **Performance** | ~1000 req/s | ~3000+ req/s |
| **Learning curve** | Lower | Slightly higher |
| **Production readiness** | Mature ecosystem | Modern, production-ready |

> **Critical Insight:** For ML serving at scale, FastAPI is the modern standard at companies like Uber and Netflix due to native async support (critical when your model inference is I/O-bound waiting on feature store lookups) and automatic request validation via Pydantic.

### Complete FastAPI Example

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from typing import List, Optional
import joblib
import numpy as np
import logging
from datetime import datetime

# --- Setup ---
app = FastAPI(title="ML Model Serving API", version="1.0.0")
logger = logging.getLogger(__name__)

# Load model at startup (not per-request!)
model = joblib.load("model.joblib")
MODEL_VERSION = "v1.2.3"

# --- Request/Response Schemas ---
class PredictionRequest(BaseModel):
    features: List[float] = Field(..., min_length=10, max_length=10)
    request_id: Optional[str] = None

    class Config:
        json_schema_extra = {
            "example": {
                "features": [0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0],
                "request_id": "req-abc-123"
            }
        }

class PredictionResponse(BaseModel):
    prediction: int
    probability: float
    model_version: str
    latency_ms: float
    request_id: Optional[str]

class HealthResponse(BaseModel):
    status: str
    model_version: str
    timestamp: str

# --- Endpoints ---
@app.get("/health", response_model=HealthResponse)
async def health_check():
    return HealthResponse(
        status="healthy",
        model_version=MODEL_VERSION,
        timestamp=datetime.utcnow().isoformat()
    )

@app.post("/predict", response_model=PredictionResponse)
async def predict(request: PredictionRequest):
    start_time = datetime.utcnow()
    try:
        features = np.array(request.features).reshape(1, -1)
        prediction = int(model.predict(features)[0])
        probability = float(model.predict_proba(features).max())

        latency = (datetime.utcnow() - start_time).total_seconds() * 1000

        logger.info(f"Prediction: {prediction}, Latency: {latency:.2f}ms")

        return PredictionResponse(
            prediction=prediction,
            probability=probability,
            model_version=MODEL_VERSION,
            latency_ms=round(latency, 2),
            request_id=request.request_id
        )
    except Exception as e:
        logger.error(f"Prediction failed: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict/batch")
async def predict_batch(requests: List[PredictionRequest]):
    """Batch prediction for efficiency."""
    features = np.array([r.features for r in requests])
    predictions = model.predict(features).tolist()
    probabilities = model.predict_proba(features).max(axis=1).tolist()
    return [
        {"prediction": p, "probability": prob, "model_version": MODEL_VERSION}
        for p, prob in zip(predictions, probabilities)
    ]

# Run: uvicorn app:app --host 0.0.0.0 --port 8000 --workers 4
```

---

## 3. Docker Basics for ML

> **Critical Insight:** Docker solves the "it works on my machine" problem for ML by freezing the exact Python version, library versions, system dependencies, and model artifacts into a reproducible image. At big tech, every model deployment is containerized.

### Production Dockerfile

```python
# Dockerfile for ML Model Serving
# Use slim base to minimize image size (full python image is ~900MB, slim is ~150MB)
FROM python:3.10-slim as builder

WORKDIR /app

# Install system dependencies (needed for numpy, scipy)
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies first (leverages Docker layer caching)
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# --- Production stage (multi-stage build) ---
FROM python:3.10-slim

WORKDIR /app

# Copy only installed packages from builder
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH

# Copy application code and model artifacts
COPY app.py .
COPY model.joblib .

# Non-root user for security
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

EXPOSE 8000

# Use gunicorn with uvicorn workers for production
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

```python
# requirements.txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
scikit-learn==1.3.2
numpy==1.26.2
joblib==1.3.2
pydantic==2.5.2
```

```python
# Build and run commands:
# docker build -t ml-model-api:v1.2.3 .
# docker run -p 8000:8000 --memory=2g --cpus=2 ml-model-api:v1.2.3

# Docker Compose for local development with monitoring
# docker-compose.yml
"""
version: '3.8'
services:
  model-api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - MODEL_PATH=/app/model.joblib
      - LOG_LEVEL=INFO
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '2.0'
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
"""
```

---

## 4. MLflow Experiment Tracking

> **Critical Insight:** Without experiment tracking, you cannot reproduce results, compare model versions, or audit decisions. MLflow is the de-facto open-source standard used at Databricks, Microsoft, and adopted internally at many big tech companies. It tracks the full lineage: code version, data version, hyperparameters, metrics, and artifacts.

```python
import mlflow
import mlflow.sklearn
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.model_selection import cross_val_score
from sklearn.metrics import accuracy_score, f1_score, roc_auc_score
import numpy as np

# Set tracking URI (local or remote server)
mlflow.set_tracking_uri("http://mlflow-server:5000")
mlflow.set_experiment("churn-prediction-v2")

# Define hyperparameter search space
param_grid = [
    {"n_estimators": 100, "max_depth": 3, "learning_rate": 0.1},
    {"n_estimators": 200, "max_depth": 5, "learning_rate": 0.05},
    {"n_estimators": 300, "max_depth": 7, "learning_rate": 0.01},
]

best_run_id = None
best_auc = 0

for params in param_grid:
    with mlflow.start_run(run_name=f"gbm_depth{params['max_depth']}"):
        # Log parameters
        mlflow.log_params(params)
        mlflow.log_param("model_type", "GradientBoosting")
        mlflow.log_param("data_version", "v2.1-20240115")

        # Train model
        model = GradientBoostingClassifier(**params, random_state=42)
        model.fit(X_train, y_train)

        # Evaluate
        y_pred = model.predict(X_test)
        y_proba = model.predict_proba(X_test)[:, 1]

        metrics = {
            "accuracy": accuracy_score(y_test, y_pred),
            "f1_score": f1_score(y_test, y_pred),
            "roc_auc": roc_auc_score(y_test, y_proba),
            "cv_score_mean": cross_val_score(model, X_train, y_train, cv=5).mean(),
        }

        # Log metrics
        mlflow.log_metrics(metrics)

        # Log model artifact with signature
        from mlflow.models import infer_signature
        signature = infer_signature(X_train, model.predict(X_train))
        mlflow.sklearn.log_model(
            model, "model",
            signature=signature,
            registered_model_name="churn-predictor"
        )

        # Log additional artifacts
        mlflow.log_artifact("feature_importance.png")
        mlflow.log_dict(
            {"features": feature_names, "target": "churn"},
            "metadata.json"
        )

        # Track best model
        if metrics["roc_auc"] > best_auc:
            best_auc = metrics["roc_auc"]
            best_run_id = mlflow.active_run().info.run_id

# Promote best model to production
from mlflow.tracking import MlflowClient
client = MlflowClient()
client.transition_model_version_stage(
    name="churn-predictor",
    version=best_run_id,
    stage="Production"
)
```

---

## 5. Feature Stores

### Why Feature Stores Matter

| Problem Without Feature Store | Solution with Feature Store |
|-------------------------------|----------------------------|
| Training/serving feature skew | Single feature definition, dual materialization |
| Duplicated feature engineering | Shared feature registry across teams |
| Point-in-time correctness issues | Built-in time-travel for training data |
| Slow feature computation at serving | Pre-computed features in low-latency store |
| No feature discoverability | Searchable catalog with metadata |

> **Critical Insight:** Feature stores solve the #1 cause of ML production bugs: training-serving skew. By defining features once and materializing them for both offline (training) and online (serving) use, you guarantee consistency. Companies like Uber (Michelangelo), Spotify (Jukebox), and Netflix all built or adopted feature stores.

### Feast Example

```python
# feature_repo/feature_definitions.py
from feast import Entity, Feature, FeatureView, FileSource, ValueType
from feast.types import Float32, Int64, String
from datetime import timedelta

# Define data source
user_activity_source = FileSource(
    path="s3://feature-store/user_activity.parquet",
    timestamp_field="event_timestamp",
    created_timestamp_column="created_at",
)

# Define entity (the join key)
user_entity = Entity(
    name="user_id",
    value_type=ValueType.INT64,
    description="Unique user identifier",
)

# Define feature view (group of related features)
user_activity_features = FeatureView(
    name="user_activity_features",
    entities=[user_entity],
    ttl=timedelta(days=7),  # Feature freshness
    schema=[
        Feature(name="session_count_7d", dtype=Int64),
        Feature(name="avg_session_duration_min", dtype=Float32),
        Feature(name="pages_viewed_7d", dtype=Int64),
        Feature(name="last_purchase_days_ago", dtype=Int64),
        Feature(name="lifetime_spend", dtype=Float32),
    ],
    source=user_activity_source,
    online=True,  # Materialize to online store for real-time serving
)

# --- Training: Get historical features (point-in-time correct) ---
from feast import FeatureStore
import pandas as pd

store = FeatureStore(repo_path="feature_repo/")

# Entity dataframe with timestamps (avoids data leakage!)
entity_df = pd.DataFrame({
    "user_id": [1001, 1002, 1003],
    "event_timestamp": pd.to_datetime(["2024-01-15", "2024-01-15", "2024-01-15"]),
})

training_df = store.get_historical_features(
    entity_df=entity_df,
    features=[
        "user_activity_features:session_count_7d",
        "user_activity_features:avg_session_duration_min",
        "user_activity_features:lifetime_spend",
    ],
).to_df()

# --- Serving: Get online features (low-latency) ---
online_features = store.get_online_features(
    features=[
        "user_activity_features:session_count_7d",
        "user_activity_features:avg_session_duration_min",
        "user_activity_features:lifetime_spend",
    ],
    entity_rows=[{"user_id": 1001}],
).to_dict()

# Materialize features to online store (run on schedule)
# feast materialize 2024-01-01T00:00:00 2024-01-15T00:00:00
```

---

## 6. Model Monitoring

### Types of Drift

| Drift Type | What Changes | Detection Method | Example |
|-----------|--------------|------------------|---------|
| **Data drift** | Input feature distribution P(X) | KS test, PSI, KL divergence | User age distribution shifts |
| **Concept drift** | Relationship P(Y|X) | Performance monitoring | COVID changes buying patterns |
| **Prediction drift** | Output distribution P(Y_hat) | Distribution comparison | Model predicts more positives |
| **Label drift** | Target distribution P(Y) | Ground truth monitoring | Fraud rate increases |

> **Critical Insight:** Models degrade silently. Unlike software bugs that crash, ML models fail by becoming gradually less accurate. At Google, models are monitored with automated alerts that trigger retraining when performance drops below thresholds. The key insight: you often detect data drift weeks before you can measure performance degradation (because labels arrive with delay).

### Monitoring Implementation

```python
import numpy as np
from scipy import stats
from typing import Dict, List, Tuple
import logging

class ModelMonitor:
    """Production model monitoring system."""

    def __init__(self, reference_data: np.ndarray, feature_names: List[str]):
        self.reference_data = reference_data
        self.feature_names = feature_names
        self.logger = logging.getLogger(__name__)

    def detect_data_drift_ks(
        self, current_data: np.ndarray, threshold: float = 0.05
    ) -> Dict[str, dict]:
        """Kolmogorov-Smirnov test for each feature."""
        drift_report = {}
        for i, feature in enumerate(self.feature_names):
            statistic, p_value = stats.ks_2samp(
                self.reference_data[:, i],
                current_data[:, i]
            )
            is_drifted = p_value < threshold
            drift_report[feature] = {
                "statistic": round(statistic, 4),
                "p_value": round(p_value, 4),
                "is_drifted": is_drifted,
            }
            if is_drifted:
                self.logger.warning(
                    f"DRIFT DETECTED: {feature} (KS={statistic:.4f}, p={p_value:.4f})"
                )
        return drift_report

    def population_stability_index(
        self, current_data: np.ndarray, bins: int = 10
    ) -> Dict[str, float]:
        """PSI: industry-standard drift metric. PSI > 0.2 = significant drift."""
        psi_scores = {}
        for i, feature in enumerate(self.feature_names):
            # Bin based on reference distribution
            breakpoints = np.percentile(
                self.reference_data[:, i],
                np.linspace(0, 100, bins + 1)
            )
            ref_counts = np.histogram(self.reference_data[:, i], bins=breakpoints)[0]
            cur_counts = np.histogram(current_data[:, i], bins=breakpoints)[0]

            # Normalize to proportions (avoid zeros)
            ref_pct = (ref_counts + 1) / (ref_counts.sum() + bins)
            cur_pct = (cur_counts + 1) / (cur_counts.sum() + bins)

            psi = np.sum((cur_pct - ref_pct) * np.log(cur_pct / ref_pct))
            psi_scores[feature] = round(psi, 4)

        return psi_scores

    def monitor_performance(
        self,
        predictions: np.ndarray,
        actuals: np.ndarray,
        metric_fn,
        baseline_metric: float,
        degradation_threshold: float = 0.05,
    ) -> Tuple[float, bool]:
        """Alert if performance drops beyond threshold."""
        current_metric = metric_fn(actuals, predictions)
        relative_drop = (baseline_metric - current_metric) / baseline_metric
        is_degraded = relative_drop > degradation_threshold

        if is_degraded:
            self.logger.critical(
                f"PERFORMANCE DEGRADATION: {relative_drop:.1%} drop "
                f"(baseline={baseline_metric:.4f}, current={current_metric:.4f})"
            )
        return current_metric, is_degraded


# Usage
monitor = ModelMonitor(X_train, feature_names)
drift_report = monitor.detect_data_drift_ks(X_production_batch)
psi_scores = monitor.population_stability_index(X_production_batch)

# Alert if any PSI > 0.2
drifted_features = {k: v for k, v in psi_scores.items() if v > 0.2}
if drifted_features:
    trigger_retraining_pipeline(drifted_features)
```

---

## 7. Batch vs Real-Time Inference

| Dimension | Batch Inference | Real-Time Inference |
|-----------|----------------|---------------------|
| **Latency** | Minutes to hours | Milliseconds (p99 < 100ms) |
| **Throughput** | Very high (millions of records) | Per-request |
| **Infrastructure** | Spark/Airflow scheduled jobs | REST API + load balancer |
| **Cost** | Lower (scheduled compute) | Higher (always-on servers) |
| **Freshness** | Stale (hours/days old) | Up-to-the-moment |
| **Complexity** | Lower | Higher (caching, fallbacks) |
| **Use cases** | Recommendations, email campaigns, reports | Fraud detection, search ranking, pricing |
| **Failure mode** | Retry entire batch | Per-request fallback needed |

> **Critical Insight:** The choice is rarely binary. Netflix uses batch inference to pre-compute recommendations for the homepage (95% of traffic) but real-time inference for "Because you watched X" rows that depend on the most recent viewing session. The hybrid approach optimizes cost while maintaining freshness where it matters.

```python
# --- Batch Inference Pattern (Spark/Airflow) ---
from pyspark.sql import SparkSession
import mlflow

def batch_predict(date: str):
    """Daily batch prediction job."""
    spark = SparkSession.builder.appName("batch-predict").getOrCreate()

    # Load model from registry
    model = mlflow.pyfunc.load_model("models:/churn-predictor/Production")

    # Read features from data warehouse
    features_df = spark.read.parquet(
        f"s3://warehouse/user_features/dt={date}/"
    )

    # Predict in parallel across Spark executors
    predictions = features_df.toPandas()
    predictions["churn_score"] = model.predict(
        predictions[feature_columns]
    )

    # Write predictions back to serving layer
    spark.createDataFrame(predictions[["user_id", "churn_score"]]) \
        .write.mode("overwrite") \
        .parquet(f"s3://predictions/churn/dt={date}/")

    return predictions.shape[0]


# --- Real-Time with Caching Pattern ---
from functools import lru_cache
import redis
import hashlib
import json

redis_client = redis.Redis(host="cache-server", port=6379)

def get_prediction_with_cache(user_id: int, features: dict) -> dict:
    """Real-time prediction with Redis caching."""
    # Check cache first
    cache_key = f"pred:{user_id}:{hashlib.md5(json.dumps(features, sort_keys=True).encode()).hexdigest()}"
    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)

    # Cache miss: compute prediction
    prediction = model.predict(np.array(list(features.values())).reshape(1, -1))
    result = {"user_id": user_id, "score": float(prediction[0])}

    # Cache for 1 hour
    redis_client.setex(cache_key, 3600, json.dumps(result))
    return result
```

---

## 8. A/B Testing Models in Production

### Deployment Strategies

| Strategy | Risk Level | Traffic | Duration | Use Case |
|----------|-----------|---------|----------|----------|
| **Shadow mode** | Zero | 100% to both, only champion serves | 1-2 weeks | Validate latency & correctness |
| **Canary** | Very low | 1-5% to challenger | Days | Catch catastrophic failures |
| **A/B test** | Low | 50/50 (or custom split) | Weeks | Measure business metric impact |
| **Multi-armed bandit** | Low | Dynamic allocation | Ongoing | Continuous optimization |
| **Blue/green** | Medium | Full swap | Instant | Infrastructure upgrades |

> **Critical Insight:** At Meta and Google, no model goes directly to 100% traffic. The standard progression is: Shadow mode (verify no crashes, measure latency) -> Canary (1% traffic, monitor for regressions) -> Gradual ramp (5% -> 25% -> 50% -> 100%) with automated rollback triggers. This process typically takes 2-4 weeks.

```python
import random
import hashlib
from typing import Optional
from dataclasses import dataclass

@dataclass
class ExperimentConfig:
    name: str
    champion_model: str
    challenger_model: str
    traffic_split: float  # fraction to challenger
    metrics_to_track: list
    min_sample_size: int
    rollback_threshold: float  # max acceptable degradation

class ABTestRouter:
    """Production A/B test traffic router with consistent assignment."""

    def __init__(self, config: ExperimentConfig):
        self.config = config
        self.models = {
            "champion": load_model(config.champion_model),
            "challenger": load_model(config.challenger_model),
        }

    def get_variant(self, user_id: str) -> str:
        """Deterministic assignment based on user_id hash (sticky sessions)."""
        hash_value = int(hashlib.sha256(
            f"{self.config.name}:{user_id}".encode()
        ).hexdigest(), 16)
        bucket = (hash_value % 1000) / 1000.0

        if bucket < self.config.traffic_split:
            return "challenger"
        return "champion"

    def predict(self, user_id: str, features: dict) -> dict:
        """Route prediction to assigned variant."""
        variant = self.get_variant(user_id)
        model = self.models[variant]

        prediction = model.predict(features)

        # Log for analysis
        log_experiment_event(
            experiment=self.config.name,
            user_id=user_id,
            variant=variant,
            prediction=prediction,
            features=features,
        )

        return {"prediction": prediction, "variant": variant}

    def shadow_predict(self, user_id: str, features: dict) -> dict:
        """Shadow mode: serve champion, log challenger predictions."""
        champion_pred = self.models["champion"].predict(features)
        challenger_pred = self.models["challenger"].predict(features)

        # Log both for offline comparison
        log_shadow_event(
            user_id=user_id,
            champion_pred=champion_pred,
            challenger_pred=challenger_pred,
        )

        # Always serve champion in shadow mode
        return {"prediction": champion_pred, "variant": "champion"}


# Statistical significance check
from scipy.stats import chi2_contingency, mannwhitneyu

def check_experiment_significance(
    champion_conversions: int, champion_total: int,
    challenger_conversions: int, challenger_total: int,
    alpha: float = 0.05
) -> dict:
    """Check if A/B test result is statistically significant."""
    contingency_table = [
        [champion_conversions, champion_total - champion_conversions],
        [challenger_conversions, challenger_total - challenger_conversions],
    ]
    chi2, p_value, dof, expected = chi2_contingency(contingency_table)

    champion_rate = champion_conversions / champion_total
    challenger_rate = challenger_conversions / challenger_total
    lift = (challenger_rate - champion_rate) / champion_rate

    return {
        "p_value": p_value,
        "is_significant": p_value < alpha,
        "champion_rate": champion_rate,
        "challenger_rate": challenger_rate,
        "relative_lift": f"{lift:.2%}",
        "recommendation": "Deploy challenger" if (p_value < alpha and lift > 0)
                          else "Keep champion",
    }
```

---

## 9. CI/CD for ML

> **Critical Insight:** ML CI/CD is fundamentally different from software CI/CD. Software tests verify correctness (pass/fail); ML tests verify quality (metrics above thresholds). You need both code tests AND model validation tests. At Spotify and Uber, ML pipelines include automated retraining triggered by drift detection, with human-in-the-loop approval before production promotion.

### ML Pipeline Architecture

```python
# .github/workflows/ml_pipeline.yml (conceptual structure)
"""
ML CI/CD Pipeline Stages:
1. Code Quality: lint, type check, unit tests
2. Data Validation: schema checks, distribution tests
3. Model Training: train on latest data
4. Model Validation: performance thresholds, fairness checks
5. Integration Tests: API contract tests, latency benchmarks
6. Staging Deployment: shadow mode testing
7. Production Promotion: gradual rollout with monitoring
"""

# --- Data Validation (Great Expectations style) ---
import great_expectations as ge

def validate_training_data(df):
    """Gate: fail pipeline if data quality degrades."""
    ge_df = ge.from_pandas(df)

    results = ge_df.expect_column_values_to_not_be_null("user_id")
    assert results.success, "Null user_ids found!"

    results = ge_df.expect_column_values_to_be_between(
        "age", min_value=13, max_value=120
    )
    assert results.success, "Age values out of expected range!"

    results = ge_df.expect_column_mean_to_be_between(
        "session_count_7d", min_value=1.0, max_value=50.0
    )
    assert results.success, "Feature distribution shifted unexpectedly!"

    return True


# --- Model Validation Gate ---
def validate_model(model, X_test, y_test, baseline_metrics: dict) -> bool:
    """Gate: model must beat baseline on all critical metrics."""
    from sklearn.metrics import accuracy_score, f1_score, roc_auc_score

    y_pred = model.predict(X_test)
    y_proba = model.predict_proba(X_test)[:, 1]

    current_metrics = {
        "accuracy": accuracy_score(y_test, y_pred),
        "f1": f1_score(y_test, y_pred),
        "auc": roc_auc_score(y_test, y_proba),
    }

    # Must not degrade more than 1% on any metric
    for metric, value in current_metrics.items():
        baseline = baseline_metrics[metric]
        if value < baseline * 0.99:
            raise ValueError(
                f"Model FAILED validation: {metric} dropped from "
                f"{baseline:.4f} to {value:.4f}"
            )

    # Fairness check: performance parity across groups
    for group in ["male", "female"]:
        group_mask = X_test["gender"] == group
        group_auc = roc_auc_score(y_test[group_mask], y_proba[group_mask])
        if abs(group_auc - current_metrics["auc"]) > 0.05:
            raise ValueError(f"Fairness violation: AUC disparity for {group}")

    return True


# --- Automated Retraining Trigger ---
def check_retrain_trigger() -> bool:
    """Decide if model needs retraining based on monitoring signals."""
    triggers = {
        "performance_degraded": check_performance_below_threshold(),
        "data_drift_detected": check_drift_above_psi_threshold(),
        "scheduled_retrain": check_if_weekly_retrain_due(),
        "new_labeled_data": check_new_labels_available(min_count=1000),
    }

    should_retrain = any(triggers.values())
    if should_retrain:
        log_retrain_reason(triggers)
    return should_retrain
```

---

## 10. Scalability Considerations

| Strategy | When to Use | Latency Impact | Cost Impact |
|----------|------------|----------------|-------------|
| **Horizontal scaling** (more replicas) | High QPS, stateless models | Reduces via load balancing | Linear increase |
| **Model distillation** | Large models, strict latency | Major reduction | Reduces (smaller model) |
| **Quantization** (FP32 -> INT8) | Edge/mobile, cost pressure | 2-4x speedup | Reduces |
| **Caching** | Repeated inputs, slow models | Near-zero for cache hits | Low (Redis) |
| **Batching** | GPU models, throughput focus | Slight increase per-request | Better GPU utilization |
| **Model sharding** | Huge models (LLMs) | Network overhead | Multiple GPUs |
| **Async processing** | Non-blocking, event-driven | Eventual (queue-based) | Efficient |

> **Critical Insight:** The #1 scalability lever for most ML systems is not bigger servers — it is smarter caching and pre-computation. At Amazon, recommendation models pre-compute predictions for the top 1000 items per user segment and cache them. Only long-tail queries hit the real-time model. This reduces real-time inference load by 80%+.

```python
# --- Horizontal Scaling with Kubernetes ---
"""
# kubernetes deployment manifest (conceptual)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: model-serving
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero-downtime deploys
  template:
    spec:
      containers:
      - name: model-api
        image: ml-model-api:v1.2.3
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        averageUtilization: 70
"""

# --- Model Optimization: Quantization Example ---
import onnxruntime as ort
from onnxruntime.quantization import quantize_dynamic, QuantType

# Quantize ONNX model (FP32 -> INT8)
quantize_dynamic(
    model_input="model_fp32.onnx",
    model_output="model_int8.onnx",
    weight_type=QuantType.QInt8,
)

# Benchmark comparison
import time

def benchmark_model(session, input_data, n_iterations=1000):
    """Measure inference latency."""
    input_name = session.get_inputs()[0].name
    latencies = []
    for _ in range(n_iterations):
        start = time.perf_counter()
        session.run(None, {input_name: input_data})
        latencies.append((time.perf_counter() - start) * 1000)
    return {
        "p50_ms": np.percentile(latencies, 50),
        "p95_ms": np.percentile(latencies, 95),
        "p99_ms": np.percentile(latencies, 99),
    }

# Compare FP32 vs INT8
fp32_session = ort.InferenceSession("model_fp32.onnx")
int8_session = ort.InferenceSession("model_int8.onnx")

print("FP32:", benchmark_model(fp32_session, sample_input))
print("INT8:", benchmark_model(int8_session, sample_input))
# Typical result: INT8 is 2-4x faster with <1% accuracy loss
```

---

## 11. Interview Talking Points & Scripts

### "How would you deploy this model?" — Structured Framework

> *"I'd approach deployment in five phases. First, **packaging**: I'd serialize the model in ONNX format for framework-agnostic serving, wrap it in a FastAPI service with Pydantic validation, health checks, and structured logging. Second, **containerization**: Docker multi-stage build to minimize image size, pinned dependencies, non-root user. Third, **infrastructure**: deploy behind a load balancer with auto-scaling based on CPU/latency metrics, starting with 3 replicas. Fourth, **validation**: shadow mode for one week to compare latency and prediction distribution against the current system, then canary at 5% traffic with automated rollback triggers. Fifth, **monitoring**: track data drift via PSI on input features, model performance via delayed ground truth labels, and operational metrics like p99 latency and error rates. I'd set up automated retraining triggered by drift detection, with a validation gate before any model reaches production."*

### "How do you handle training-serving skew?"

> *"Training-serving skew is the silent killer of ML systems. I address it at three levels. First, **feature computation**: I use a feature store like Feast that defines features once and materializes them for both offline training and online serving — this eliminates logic duplication. Second, **data validation**: I run schema checks and distribution tests on both training data and live traffic, alerting when they diverge. Third, **integration testing**: before deployment, I send production traffic samples through the candidate model and verify predictions match offline evaluation within acceptable bounds. The key principle is: any transformation that happens during training must happen identically during serving."*

### "How would you monitor this model long-term?"

> *"I'd set up monitoring across three dimensions. **Data monitoring**: track PSI scores for each input feature weekly, alert if any feature exceeds 0.2. **Performance monitoring**: once ground truth labels arrive — which might be days or weeks later depending on the use case — compute AUC/F1 on rolling windows and alert on 5%+ degradation. **Operational monitoring**: p50/p95/p99 latency, error rates, throughput, and memory usage. I'd also monitor prediction distribution — if the model suddenly predicts 30% positive when historically it was 10%, something is wrong even before labels arrive. All of this feeds into an automated retraining decision engine."*

---

## 12. Common Interview Mistakes

| ❌ Mistake | ✅ Better Approach |
|-----------|-------------------|
| Jumping straight to model architecture without discussing serving | Start with requirements: latency SLA, throughput, freshness needs |
| Saying "I'd just use pickle to save the model" | Discuss format tradeoffs; recommend ONNX for cross-platform production |
| Ignoring training-serving skew | Proactively mention feature stores and consistent preprocessing |
| No monitoring plan | Always include drift detection, performance tracking, and alerting |
| Deploying directly to 100% traffic | Describe shadow mode -> canary -> gradual ramp progression |
| Forgetting about fallback behavior | Explain what happens when the model fails (default predictions, cached results) |
| Not mentioning versioning | Discuss model registry, artifact versioning, and rollback capabilities |
| Over-engineering for scale on day one | Start simple (single replica), explain when/how to scale |
| Treating ML deployment like software deployment | Highlight differences: data dependencies, concept drift, retraining needs |
| Ignoring cost | Discuss batch vs real-time tradeoffs, caching strategies, model optimization |
| Not considering fairness in production | Include fairness monitoring as part of the validation gate |
| Saying "the data engineers handle that" | Own the full lifecycle — a senior DS should understand the entire pipeline |

---

## 13. Rapid-Fire Q&A

**Q1: What is training-serving skew and how do you prevent it?**
A: It is when the features/preprocessing seen during training differ from serving. Prevent with feature stores (single definition), integration tests comparing training vs serving outputs, and shared preprocessing libraries.

**Q2: When would you choose batch over real-time inference?**
A: Batch when latency tolerance is high (recommendations refreshed daily), volume is massive, and freshness is not critical. Real-time when decisions must reflect current state (fraud detection, dynamic pricing).

**Q3: How do you decide when to retrain a model?**
A: Monitor three signals: (1) performance degradation on labeled data, (2) data drift exceeding PSI > 0.2 threshold, (3) scheduled cadence (weekly/monthly). Ideally, automated triggering with human approval gate.

**Q4: What is the difference between shadow mode and canary deployment?**
A: Shadow mode runs both models on all traffic but only serves the champion's predictions (zero risk). Canary routes a small percentage of real traffic to the challenger and serves its predictions (low risk, measures real impact).

**Q5: How do you ensure model reproducibility?**
A: Version everything: code (git), data (DVC/versioned storage), hyperparameters (MLflow), environment (Docker), random seeds. Use MLflow or similar to link all artifacts to a specific run.

**Q6: What metrics would you monitor for a production model?**
A: Three layers — operational (latency p99, error rate, throughput), data quality (feature drift PSI, null rates, schema violations), and model performance (AUC, precision/recall on delayed ground truth).

**Q7: How do you handle model failures gracefully?**
A: Implement fallback hierarchy: (1) cached recent prediction, (2) default/heuristic prediction, (3) previous model version, (4) graceful degradation (show non-personalized content). Never return 500 to the user.

**Q8: What is the role of a feature store?**
A: Centralized platform for managing features — ensures consistency between training and serving, provides point-in-time correctness for training data, enables feature reuse across teams, and offers low-latency serving.

**Q9: How would you scale a model serving 10,000 QPS?**
A: Horizontal scaling (auto-scaled replicas behind load balancer), request batching for GPU models, prediction caching (Redis), model optimization (quantization/distillation), and pre-computation for common queries.

**Q10: What is concept drift vs data drift?**
A: Data drift is when input distribution P(X) changes (e.g., new user demographics). Concept drift is when the relationship P(Y|X) changes (e.g., pandemic changes what features predict churn). Both degrade performance, but concept drift is harder to detect without labels.

**Q11: How do you A/B test a model without affecting user experience?**
A: Start with shadow mode (no user impact), then canary to 1-5% of traffic with close monitoring, then gradually ramp. Use consistent hashing on user_id for sticky assignments. Define clear success metrics and minimum detectable effect before starting.

**Q12: What is CI/CD for ML and how does it differ from software CI/CD?**
A: ML CI/CD adds data validation, model quality gates (metrics above thresholds), fairness checks, and automated retraining triggers. Unlike software where tests are pass/fail, ML tests are continuous quality measures that degrade gradually.

---

## 14. Summary Cheat Sheet

```
+------------------------------------------------------------------------+
|              MODEL DEPLOYMENT & PRODUCTIONIZATION CHEAT SHEET           |
+------------------------------------------------------------------------+
|                                                                        |
|  SERIALIZATION:  Pickle (prototype) | Joblib (sklearn) | ONNX (prod)  |
|                                                                        |
|  SERVING STACK:  FastAPI + Pydantic + Uvicorn + Docker + K8s           |
|                                                                        |
|  DEPLOYMENT PROGRESSION:                                               |
|    Shadow Mode --> Canary (1-5%) --> Gradual Ramp --> Full Traffic      |
|                                                                        |
|  MONITORING LAYERS:                                                    |
|    1. Operational: latency, errors, throughput                         |
|    2. Data Quality: PSI, schema, null rates                            |
|    3. Performance: AUC/F1 on delayed ground truth                      |
|                                                                        |
|  FEATURE STORE BENEFITS:                                               |
|    - Eliminates training-serving skew                                  |
|    - Point-in-time correctness                                         |
|    - Feature reuse across teams                                        |
|                                                                        |
|  DRIFT THRESHOLDS:                                                     |
|    PSI < 0.1 = stable | 0.1-0.2 = monitor | > 0.2 = retrain           |
|                                                                        |
|  SCALING LEVERS:                                                       |
|    Caching > Pre-computation > Horizontal > Optimization > Sharding    |
|                                                                        |
|  CI/CD FOR ML:                                                         |
|    Data Validation --> Train --> Model Validation --> Staging --> Prod  |
|                                                                        |
|  GOLDEN RULE: Any transform at training MUST happen at serving         |
|                                                                        |
|  FRAMEWORK FOR "How would you deploy?":                                |
|    1. Package (ONNX + API)   4. Validate (shadow + canary)            |
|    2. Containerize (Docker)  5. Monitor (drift + perf + ops)           |
|    3. Infrastructure (K8s)                                             |
|                                                                        |
+------------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 24 of 25*
