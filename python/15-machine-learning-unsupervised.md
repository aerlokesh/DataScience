# 🎯 Topic 15: Unsupervised Machine Learning

> **Scope:** Clustering algorithms (K-Means, Hierarchical, DBSCAN, GMM), dimensionality reduction (PCA, t-SNE), anomaly detection, and evaluation metrics — with emphasis on practical applications in customer segmentation, feature engineering, and outlier detection at scale.

---

## Table of Contents

1. [Clustering Algorithms](#1-clustering-algorithms)
   - [K-Means](#k-means)
   - [Hierarchical Clustering](#hierarchical-clustering)
   - [DBSCAN](#dbscan)
   - [Gaussian Mixture Models (GMM)](#gaussian-mixture-models-gmm)
2. [Dimensionality Reduction](#2-dimensionality-reduction)
   - [PCA](#principal-component-analysis-pca)
   - [t-SNE](#t-sne)
3. [Anomaly Detection](#3-anomaly-detection)
4. [Evaluation Metrics](#4-evaluation-metrics)
5. [Clustering Algorithm Comparison](#5-clustering-algorithm-comparison)
6. [Python Implementation](#6-python-implementation)
7. [Interview Talking Points & Scripts](#7-interview-talking-points--scripts)
8. [Common Interview Mistakes](#8-common-interview-mistakes)
9. [Rapid-Fire Q&A](#9-rapid-fire-qa)
10. [Summary Cheat Sheet](#10-summary-cheat-sheet)

---

## 1. Clustering Algorithms

### K-Means

**Algorithm Steps:**
1. Initialize K centroids (randomly or via K-Means++)
2. Assign each data point to the nearest centroid
3. Recompute centroids as the mean of assigned points
4. Repeat steps 2-3 until convergence (centroids stop moving)

**Initialization Methods:**
- **Random:** Pick K random points — prone to poor local optima
- **K-Means++:** First centroid random, subsequent centroids chosen proportional to squared distance from nearest existing centroid — dramatically improves convergence

**Limitations:**
- Assumes spherical, equally-sized clusters
- Sensitive to outliers (centroids get pulled)
- Must specify K in advance
- Converges to local optima — run multiple times with different seeds
- Fails on non-convex shapes (e.g., concentric rings)

> 💡 **Critical Insight:** K-Means minimizes within-cluster sum of squares (inertia), which is NOT the same as maximizing separation. Two clusters can have low inertia but heavily overlap in feature space. Always validate with silhouette scores.

---

### Hierarchical Clustering

**Agglomerative (Bottom-Up) Algorithm:**
1. Start with each point as its own cluster
2. Merge the two closest clusters
3. Repeat until one cluster remains (or desired K reached)

**Linkage Methods:**

| Linkage | Distance Metric | Best For | Weakness |
|---------|----------------|----------|----------|
| Single | Min distance between clusters | Elongated clusters | Chaining effect |
| Complete | Max distance between clusters | Compact clusters | Sensitive to outliers |
| Average | Mean pairwise distance | Balanced approach | Computationally expensive |
| Ward | Minimizes variance increase | Equal-size clusters | Assumes spherical shapes |

**Dendrograms:** Visual tree showing merge history. Cut at different heights to get different numbers of clusters. The longest vertical line without a horizontal crossing suggests optimal K.

> 💡 **Critical Insight:** Hierarchical clustering is O(n^3) in naive implementation, making it impractical for datasets above ~10K points. Use it for exploratory analysis, then switch to K-Means or DBSCAN for production.

---

### DBSCAN

**Core Concepts:**
- **Epsilon (eps):** Maximum distance between two points to be considered neighbors
- **min_samples:** Minimum points within epsilon to form a dense region
- **Core point:** Has >= min_samples within eps
- **Border point:** Within eps of a core point but doesn't meet min_samples
- **Noise point:** Neither core nor border — labeled as -1

**Strengths:**
- No need to specify K
- Finds arbitrary-shaped clusters
- Robust to outliers (labels them as noise)
- Deterministic (unlike K-Means)

**Weaknesses:**
- Struggles with varying density clusters
- Sensitive to eps and min_samples choices
- Poor performance in high dimensions (curse of dimensionality)

> 💡 **Critical Insight:** Use a k-distance plot to select epsilon. Sort the distance to the k-th nearest neighbor (where k = min_samples) for all points, plot it, and look for the "elbow." The y-value at the elbow is your eps.

---

### Gaussian Mixture Models (GMM)

**Soft Clustering via EM Algorithm:**
1. **Initialize:** Set initial means, covariances, and mixing weights for K Gaussians
2. **E-step:** Compute probability each point belongs to each Gaussian (responsibilities)
3. **M-step:** Update means, covariances, and weights using responsibilities
4. **Repeat** until log-likelihood converges

**Key Differences from K-Means:**
- Produces probability of membership (soft assignment) vs. hard assignment
- Models elliptical clusters via covariance matrices
- Uses BIC/AIC for model selection instead of elbow method

> 💡 **Critical Insight:** GMM is the probabilistic generalization of K-Means. When covariances are constrained to be identity matrices and mixing weights are equal, GMM reduces to K-Means. This is a powerful interview answer that shows depth.

---

## 2. Dimensionality Reduction

### Principal Component Analysis (PCA)

**What It Does:** Projects data onto orthogonal axes (principal components) that maximize variance.

**Steps:**
1. Standardize features (zero mean, unit variance)
2. Compute covariance matrix
3. Compute eigenvalues and eigenvectors
4. Sort by eigenvalue magnitude (descending)
5. Select top-k components that capture desired variance (e.g., 95%)
6. Project data onto new subspace

**When to Use PCA:**
- Feature reduction before modeling (reduce multicollinearity)
- Visualization (project to 2-3 dims)
- Noise reduction (discard low-variance components)
- Speed up training of downstream models

**When NOT to Use:**
- When features have non-linear relationships (use kernel PCA or autoencoders)
- When interpretability of original features matters
- When feature importance is needed (PCA creates linear combinations)

> 💡 **Critical Insight:** PCA components are ranked by variance explained, NOT by predictive power. The first PC captures the most variance but might be irrelevant noise. Always validate that reduced features improve your actual task metric.

---

### t-SNE

**Purpose:** Non-linear dimensionality reduction for VISUALIZATION only.

**Key Parameters:**
- **Perplexity (5-50):** Roughly the number of effective neighbors. Low = local structure, High = global structure.
- **Learning rate (10-1000):** Too low = points bunch, too high = uniform blob.
- **Iterations:** Usually 1000+ needed for convergence.

**Critical Rules:**
- NEVER use t-SNE for feature reduction before modeling — it's non-parametric, can't transform new data
- Distances between clusters are meaningless — only within-cluster structure is preserved
- Different runs give different results (stochastic)
- Perplexity must be less than number of points

> 💡 **Critical Insight:** t-SNE is a visualization tool, NOT a preprocessing step. If an interviewer asks "why not use t-SNE for feature reduction?", explain: (1) it has no inverse transform, (2) it can't handle new data points, (3) distances are not preserved globally, (4) it's O(n^2) in memory.

---

## 3. Anomaly Detection

### Isolation Forest

**Intuition:** Anomalies are easier to isolate. Build random trees; anomalies have shorter average path lengths.

- Randomly select a feature and a split value
- Points that require fewer splits to isolate are more anomalous
- Anomaly score based on average path length across all trees
- No distance computation needed — works well in high dimensions

### Local Outlier Factor (LOF)

**Intuition:** Compare local density of a point to its neighbors. If significantly less dense, it's an outlier.

- Computes k-distance neighborhood for each point
- Calculates local reachability density
- LOF > 1 = less dense than neighbors (potential outlier)
- Captures local anomalies that global methods miss

**When to Use Which:**

| Method | Best For | Limitation |
|--------|----------|-----------|
| Isolation Forest | High-dimensional, large datasets | Struggles with local anomalies |
| LOF | Local density-based anomalies | O(n^2), sensitive to k |
| Z-score | Univariate, Gaussian data | Misses multivariate anomalies |
| Mahalanobis | Multivariate Gaussian | Assumes elliptical distribution |

---

## 4. Evaluation Metrics

### Silhouette Score
- Range: [-1, 1]. Higher = better defined clusters.
- For each point: (b - a) / max(a, b) where a = avg intra-cluster distance, b = avg nearest-cluster distance.
- Score near 0 = overlapping clusters. Negative = likely misassigned.

### Elbow Method
- Plot inertia (within-cluster sum of squares) vs. K.
- Look for the "elbow" where marginal improvement drops.
- Subjective — often unclear. Use silhouette as confirmation.

### Davies-Bouldin Index
- Lower = better. Measures avg similarity between each cluster and its most similar cluster.
- Does not require ground truth labels.
- Biased toward convex clusters (like K-Means).

### Calinski-Harabasz Index
- Higher = better. Ratio of between-cluster dispersion to within-cluster dispersion.
- Fast to compute but biased toward spherical clusters.

> 💡 **Critical Insight:** No single metric is definitive. In interviews, say: "I use the elbow method as a starting point, validate with silhouette scores across multiple K values, and ultimately assess whether the clusters are actionable for the business problem."

---

## 5. Clustering Algorithm Comparison

| Algorithm | Cluster Shape | Handles Noise | Needs K | Scalability | Soft Assignment |
|-----------|--------------|---------------|---------|-------------|-----------------|
| K-Means | Spherical | No | Yes | O(nKt) — excellent | No |
| Hierarchical | Any (linkage-dependent) | No | Optional | O(n^3) — poor | No |
| DBSCAN | Arbitrary | Yes | No | O(n log n) w/ index | No |
| GMM | Elliptical | No | Yes | O(nK^2d) — moderate | Yes |
| Mean Shift | Arbitrary | Yes | No | O(n^2) — poor | No |
| Spectral | Arbitrary | No | Yes | O(n^3) — poor | No |

---

## 6. Python Implementation

```python
import numpy as np
import pandas as pd
from sklearn.cluster import KMeans, DBSCAN, AgglomerativeClustering
from sklearn.mixture import GaussianMixture
from sklearn.decomposition import PCA
from sklearn.manifold import TSNE
from sklearn.ensemble import IsolationForest
from sklearn.neighbors import LocalOutlierFactor
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import silhouette_score, davies_bouldin_score
import matplotlib.pyplot as plt

# ============================================================
# DATA PREPARATION (Always standardize for distance-based methods)
# ============================================================
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)

# ============================================================
# K-MEANS WITH ELBOW METHOD
# ============================================================
inertias = []
silhouettes = []
K_range = range(2, 11)

for k in K_range:
    km = KMeans(n_clusters=k, init='k-means++', n_init=10, random_state=42)
    labels = km.fit_predict(X_scaled)
    inertias.append(km.inertia_)
    silhouettes.append(silhouette_score(X_scaled, labels))

# Plot elbow
fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 4))
ax1.plot(K_range, inertias, 'bo-')
ax1.set_xlabel('K'); ax1.set_title('Elbow Method')
ax2.plot(K_range, silhouettes, 'ro-')
ax2.set_xlabel('K'); ax2.set_title('Silhouette Score')
plt.tight_layout()
plt.show()

# Final model
best_k = 4  # chosen from elbow + silhouette
kmeans = KMeans(n_clusters=best_k, init='k-means++', n_init=10, random_state=42)
cluster_labels = kmeans.fit_predict(X_scaled)

# ============================================================
# DBSCAN WITH K-DISTANCE PLOT FOR EPS SELECTION
# ============================================================
from sklearn.neighbors import NearestNeighbors

min_samples = 5
nn = NearestNeighbors(n_neighbors=min_samples)
nn.fit(X_scaled)
distances, _ = nn.kneighbors(X_scaled)
k_distances = np.sort(distances[:, -1])

plt.plot(k_distances)
plt.xlabel('Points (sorted)'); plt.ylabel(f'{min_samples}-th NN distance')
plt.title('K-Distance Plot for eps selection')
plt.show()

# Apply DBSCAN
eps_value = 0.5  # read from k-distance elbow
dbscan = DBSCAN(eps=eps_value, min_samples=min_samples)
db_labels = dbscan.fit_predict(X_scaled)
n_clusters = len(set(db_labels)) - (1 if -1 in db_labels else 0)
n_noise = list(db_labels).count(-1)
print(f"Clusters: {n_clusters}, Noise points: {n_noise}")

# ============================================================
# GMM (SOFT CLUSTERING)
# ============================================================
gmm = GaussianMixture(n_components=4, covariance_type='full', random_state=42)
gmm.fit(X_scaled)
gmm_labels = gmm.predict(X_scaled)
gmm_probs = gmm.predict_proba(X_scaled)  # soft assignment probabilities

# Model selection via BIC
bics = [GaussianMixture(n_components=k, random_state=42).fit(X_scaled).bic(X_scaled)
        for k in range(2, 11)]
optimal_k = range(2, 11)[np.argmin(bics)]

# ============================================================
# PCA — DIMENSIONALITY REDUCTION
# ============================================================
pca = PCA(n_components=0.95)  # retain 95% variance
X_pca = pca.fit_transform(X_scaled)
print(f"Reduced from {X_scaled.shape[1]} to {X_pca.shape[1]} components")
print(f"Explained variance ratios: {pca.explained_variance_ratio_}")

# Scree plot
plt.bar(range(1, len(pca.explained_variance_ratio_) + 1),
        pca.explained_variance_ratio_, alpha=0.7)
plt.step(range(1, len(pca.explained_variance_ratio_) + 1),
         np.cumsum(pca.explained_variance_ratio_), where='mid', color='red')
plt.xlabel('Component'); plt.ylabel('Variance Explained')
plt.title('PCA Scree Plot')
plt.show()

# ============================================================
# t-SNE — VISUALIZATION ONLY
# ============================================================
tsne = TSNE(n_components=2, perplexity=30, learning_rate=200,
            n_iter=1000, random_state=42)
X_tsne = tsne.fit_transform(X_scaled)  # cannot transform new data!

plt.scatter(X_tsne[:, 0], X_tsne[:, 1], c=cluster_labels, cmap='viridis', s=10)
plt.title('t-SNE Visualization (colored by K-Means labels)')
plt.show()

# ============================================================
# ANOMALY DETECTION
# ============================================================
# Isolation Forest
iso_forest = IsolationForest(contamination=0.05, random_state=42)
anomaly_labels = iso_forest.fit_predict(X_scaled)  # -1 = anomaly, 1 = normal
anomaly_scores = iso_forest.decision_function(X_scaled)

# Local Outlier Factor
lof = LocalOutlierFactor(n_neighbors=20, contamination=0.05)
lof_labels = lof.fit_predict(X_scaled)  # -1 = anomaly
lof_scores = lof.negative_outlier_factor_

# ============================================================
# HIERARCHICAL CLUSTERING WITH DENDROGRAM
# ============================================================
from scipy.cluster.hierarchy import dendrogram, linkage

Z = linkage(X_scaled, method='ward')
plt.figure(figsize=(12, 5))
dendrogram(Z, truncate_mode='lastp', p=20)
plt.title('Dendrogram (Ward Linkage)')
plt.xlabel('Cluster size'); plt.ylabel('Distance')
plt.show()

agg = AgglomerativeClustering(n_clusters=4, linkage='ward')
agg_labels = agg.fit_predict(X_scaled)
```

---

## 7. Interview Talking Points & Scripts

### How Do You Choose K?

> *"I take a multi-pronged approach. First, I use the elbow method on inertia to get a rough range. Then I compute silhouette scores across that range to find the K with the best-defined cluster boundaries. But critically, I also validate against the business context — if I'm segmenting customers for marketing, the segments need to be actionable and differentiable. Sometimes K=3 is statistically optimal but K=5 maps better to the product team's campaign strategy. I always present both the statistical and business rationale."*

### PCA vs. t-SNE?

> *"They serve completely different purposes. PCA is a linear transformation that preserves global variance structure — I use it for actual feature reduction before modeling, because it's deterministic, invertible, and can transform new data. t-SNE is non-linear and optimized for 2D visualization — it reveals local neighborhood structure but destroys global distances. I'd never use t-SNE output as features for a model because it's stochastic, non-parametric, and O(n-squared). In practice, I often do PCA first to reduce to 50 dimensions, then t-SNE on top for visualization."*

### When Would You Use Unsupervised Learning in Production?

> *"Three main use cases come to mind. First, customer segmentation — clustering behavioral features to discover natural user groups that inform product and marketing strategy. Second, dimensionality reduction as a preprocessing step — PCA before a classifier when dealing with hundreds of correlated features, which reduces overfitting and speeds up training. Third, anomaly detection — Isolation Forest on transaction data to flag potential fraud, where labeled examples are scarce but we know anomalies are rare. The key insight is that unsupervised methods don't replace supervised ones; they complement them by discovering structure we didn't know to look for."*

### How Do You Handle the 'No Ground Truth' Problem?

> *"This is the fundamental challenge of unsupervised learning. I evaluate on three axes: internal metrics like silhouette score tell me about cluster cohesion and separation. Stability analysis — if I bootstrap or subsample the data and clusters remain consistent, I have more confidence. And most importantly, domain validation — do the clusters tell a coherent story? Can a product manager look at cluster profiles and say 'yes, these are real user archetypes'? If the clusters don't drive different actions, they're not useful regardless of metrics."*

---

## 8. Common Interview Mistakes

| ❌ Mistake | ✅ Better Answer |
|-----------|-----------------|
| "Use t-SNE to reduce features before training" | "t-SNE is for visualization only; use PCA or autoencoders for feature reduction" |
| "K-Means works for any shaped cluster" | "K-Means assumes spherical clusters; use DBSCAN or spectral clustering for arbitrary shapes" |
| "Higher silhouette always means better clustering" | "Silhouette is one signal; always validate against business usefulness and stability" |
| "DBSCAN doesn't need hyperparameters" | "DBSCAN requires careful tuning of eps and min_samples via k-distance plots" |
| "PCA components are features" | "PCA components are linear combinations of original features and lose interpretability" |
| "More clusters is always better (lower inertia)" | "Inertia always decreases with more K; the goal is finding the inflection point" |
| "GMM and K-Means are completely different" | "GMM generalizes K-Means; with spherical covariance constraints, GMM reduces to K-Means" |
| "Anomaly detection replaces manual rules" | "It complements rules — use domain rules as baseline, ML for novel patterns" |

---

## 9. Rapid-Fire Q&A

**Q1: Why standardize before K-Means?**
K-Means uses Euclidean distance. Without standardization, features with larger scales dominate the distance metric, biasing cluster assignments.

**Q2: What happens if you set min_samples=1 in DBSCAN?**
Every point becomes a core point — no noise detected. You effectively get single-linkage clustering, which defeats the purpose of DBSCAN.

**Q3: Can PCA handle categorical features?**
No. PCA requires continuous data. For mixed data, use MCA (Multiple Correspondence Analysis) or encode categoricals first, but interpret with caution.

**Q4: How does the EM algorithm avoid getting stuck?**
It doesn't guarantee a global optimum. Like K-Means, run multiple restarts. Use BIC across runs to select the best model.

**Q5: What's the time complexity of t-SNE?**
O(n^2) for the exact algorithm due to pairwise similarity computation. Barnes-Hut approximation brings it to O(n log n).

**Q6: When would silhouette score be misleading?**
With non-convex clusters (e.g., crescent shapes). DBSCAN might find them perfectly, but silhouette penalizes the irregular geometry.

**Q7: How do you deploy a clustering model in production?**
Train K-Means/GMM offline, save centroids/parameters, then at inference compute nearest centroid or highest probability component. For DBSCAN, typically pre-compute clusters and use approximate nearest neighbor for assignment.

**Q8: What's the relationship between eigenvalues and explained variance?**
Each eigenvalue represents the variance captured by its corresponding eigenvector. Explained variance ratio = eigenvalue / sum(all eigenvalues).

**Q9: Can DBSCAN find clusters of different densities?**
Poorly. A single eps can't capture both dense and sparse regions. Use HDBSCAN (hierarchical DBSCAN) or OPTICS for varying-density clusters.

**Q10: What's contamination in Isolation Forest?**
The expected proportion of anomalies in the dataset. It sets the decision threshold. If unknown, start with 0.01-0.05 and tune based on precision/recall on validated samples.

---

## 10. Summary Cheat Sheet

```
+===========================================================================+
|                    UNSUPERVISED ML — CHEAT SHEET                           |
+===========================================================================+
|                                                                           |
| CLUSTERING DECISION TREE:                                                 |
|   Know K? ──Yes──> Spherical? ──Yes──> K-Means                           |
|     │                  └──No──> GMM (elliptical) or Spectral              |
|     └──No──> Varying density? ──Yes──> HDBSCAN                           |
|                    └──No──> DBSCAN                                        |
|                                                                           |
| DIMENSIONALITY REDUCTION:                                                 |
|   Goal = Feature reduction ──> PCA (linear) or Autoencoder (non-linear)  |
|   Goal = Visualization ──> t-SNE (local) or UMAP (global + local)        |
|                                                                           |
| CHOOSING K:                                                               |
|   1. Elbow method (inertia)        — rough range                         |
|   2. Silhouette score              — statistical validation              |
|   3. BIC/AIC (for GMM)             — model selection                     |
|   4. Domain knowledge              — business actionability              |
|                                                                           |
| ANOMALY DETECTION:                                                        |
|   High-dim, large data ──> Isolation Forest                              |
|   Local density matters ──> LOF                                           |
|   Gaussian assumption ok ──> Mahalanobis distance                        |
|                                                                           |
| KEY FORMULAS:                                                             |
|   Silhouette = (b - a) / max(a, b)     range: [-1, 1]                   |
|   Inertia = sum of squared distances to nearest centroid                 |
|   Explained Variance Ratio = eigenvalue_i / sum(eigenvalues)             |
|                                                                           |
| GOLDEN RULES:                                                             |
|   - Always standardize before distance-based methods                     |
|   - t-SNE is NEVER for feature engineering                               |
|   - No single metric replaces domain validation                          |
|   - GMM generalizes K-Means (soft vs. hard assignment)                   |
|   - PCA before t-SNE is standard practice (reduce to 50 dims first)     |
|                                                                           |
+===========================================================================+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 15 of 25*
