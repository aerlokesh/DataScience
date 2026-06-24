# 🎯 Topic 34: Recommendation Systems

> *"The best recommendation is one the user didn't know they wanted but can't imagine living without."*

---

## Table of Contents

1. [Overview of Recommendation Approaches](#overview-of-recommendation-approaches)
2. [Collaborative Filtering](#collaborative-filtering)
3. [Matrix Factorization](#matrix-factorization)
4. [Content-Based Filtering](#content-based-filtering)
5. [Hybrid Approaches](#hybrid-approaches)
6. [The Cold-Start Problem](#the-cold-start-problem)
7. [Implicit vs Explicit Feedback](#implicit-vs-explicit-feedback)
8. [Evaluation Metrics for Recommendations](#evaluation-metrics-for-recommendations)
9. [Real-World Systems](#real-world-systems)
10. [Production Considerations](#production-considerations)
11. [Interview Talking Points](#interview-talking-points)
12. [Common Mistakes](#common-mistakes)
13. [Rapid-Fire Q&A](#rapid-fire-qa)
14. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Overview of Recommendation Approaches

| Approach | How It Works | Strengths | Weaknesses |
|----------|-------------|-----------|------------|
| Collaborative Filtering | "Users like you bought X" | No feature engineering needed | Cold-start problem |
| Content-Based | "Similar to items you liked" | Works for new items | Limited discovery |
| Hybrid | Combines collaborative + content | Best of both | Higher complexity |
| Knowledge-Based | Expert rules and constraints | Handles complex requirements | Hard to scale |
| Deep Learning | Neural embeddings of users/items | Captures complex patterns | Data-hungry, black-box |

> **Critical Insight:** In production, no major company uses a single approach. Netflix uses collaborative filtering + content features + context (time of day, device). Amazon uses item-based CF + purchase history + browsing behavior. The secret is combining signals intelligently with a clear fallback strategy for cold-start scenarios.

---

## Collaborative Filtering

### User-Based Collaborative Filtering

```python
import numpy as np
from scipy.spatial.distance import cosine
from sklearn.metrics.pairwise import cosine_similarity

# User-Item rating matrix (rows=users, cols=items)
# 0 means unrated
ratings = np.array([
    [5, 3, 0, 1, 4],  # User 0
    [4, 0, 0, 1, 5],  # User 1
    [1, 1, 0, 5, 1],  # User 2
    [0, 0, 5, 4, 0],  # User 3
    [5, 4, 0, 1, 4],  # User 4
])

def user_based_cf(ratings, target_user, target_item, k=3):
    """Predict rating using k most similar users who rated the target item."""
    n_users = ratings.shape[0]
    similarities = np.zeros(n_users)
    
    for other in range(n_users):
        if other == target_user:
            continue
        common = (ratings[target_user] > 0) & (ratings[other] > 0)
        if common.sum() > 0:
            similarities[other] = 1 - cosine(ratings[target_user][common], ratings[other][common])
    
    # Top-k similar users who rated target item -> weighted average
    candidates = sorted([(u, similarities[u]) for u in range(n_users) 
                         if u != target_user and ratings[u, target_item] > 0],
                        key=lambda x: -x[1])[:k]
    
    if not candidates:
        return ratings[target_user][ratings[target_user] > 0].mean()
    num = sum(sim * ratings[u, target_item] for u, sim in candidates)
    return num / sum(abs(sim) for _, sim in candidates)
```

### Item-Based Collaborative Filtering

```python
def item_based_cf(ratings, target_user, target_item, k=3):
    """Predict using similarity between items the user has rated."""
    item_similarities = cosine_similarity(ratings.T)
    user_rated = [i for i in np.where(ratings[target_user] > 0)[0] if i != target_item]
    
    # Top-k most similar items to target that user already rated
    candidates = sorted([(item, item_similarities[target_item, item]) 
                         for item in user_rated], key=lambda x: -x[1])[:k]
    if not candidates:
        return ratings[:, target_item][ratings[:, target_item] > 0].mean()
    num = sum(sim * ratings[target_user, item] for item, sim in candidates)
    return num / sum(abs(sim) for _, sim in candidates)
```

### User-Based vs Item-Based

| Aspect | User-Based CF | Item-Based CF |
|--------|--------------|---------------|
| Computation | User similarity | Item similarity |
| Stability | Less stable (user tastes change) | More stable (item attributes fixed) |
| Scalability | Poor (many users) | Better (fewer items usually) |
| Sparsity handling | Struggles with sparse data | Slightly better |
| Used by | Smaller systems | Amazon, Netflix |
| Update frequency | Must recompute when users change | Item similarities can be precomputed |
| Explanation | "Users like you bought..." | "Because you liked X..." |

---

## Matrix Factorization

### SVD (Singular Value Decomposition)

```python
from surprise import SVD, Dataset, Reader, accuracy
from surprise.model_selection import cross_validate
import pandas as pd

# Prepare data for Surprise library
reader = Reader(rating_scale=(1, 5))
data = Dataset.load_from_df(
    ratings_df[['user_id', 'item_id', 'rating']], reader
)

# SVD model
svd = SVD(n_factors=50, n_epochs=20, lr_all=0.005, reg_all=0.02)
cross_validate(svd, data, measures=['RMSE', 'MAE'], cv=5, verbose=True)

# Train on full dataset and predict
trainset = data.build_full_trainset()
svd.fit(trainset)

# Predict rating for user 'U1' on item 'I5'
prediction = svd.predict('U1', 'I5')
print(f"Predicted rating: {prediction.est:.2f}")
```

### How Matrix Factorization Works

```
User-Item Matrix (sparse):           Factorized:
                                     
     I1  I2  I3  I4  I5             User Matrix (n x k)    Item Matrix (k x m)
U1 [ 5   3   ?   1   4 ]           U1 [0.8  0.2  0.5]     [0.9  0.4  0.1  0.2  0.7]
U2 [ 4   ?   ?   1   5 ]    ≈      U2 [0.7  0.1  0.6]  x  [0.5  0.8  0.3  0.1  0.6]
U3 [ 1   1   ?   5   1 ]           U3 [0.1  0.9  0.3]     [0.2  0.3  0.9  0.8  0.2]
U4 [ ?   ?   5   4   ? ]           U4 [0.2  0.8  0.7]
U5 [ 5   4   ?   1   4 ]           U5 [0.8  0.2  0.4]
                                     
Rating(u,i) ≈ dot(user_vector_u, item_vector_i) + biases
```

### ALS (Alternating Least Squares)

```python
import implicit
from scipy.sparse import csr_matrix

# For implicit feedback (views, clicks, purchases)
# Create sparse user-item interaction matrix
user_item_sparse = csr_matrix(interaction_matrix)

# ALS model (implicit library)
model = implicit.als.AlternatingLeastSquares(
    factors=64,           # Latent dimensions
    regularization=0.01,  # L2 regularization
    iterations=15,        # Number of ALS iterations
    calculate_training_loss=True
)

# Fit on item-user matrix (transposed!)
model.fit(user_item_sparse.T)

# Get recommendations for user 0
user_id = 0
recommendations = model.recommend(
    user_id, user_item_sparse[user_id], N=10,
    filter_already_liked_items=True
)
# Returns: (item_ids, scores)

# Get similar items
similar = model.similar_items(item_id=42, N=10)
```

### SVD vs ALS

| Aspect | SVD | ALS |
|--------|-----|-----|
| Feedback type | Explicit (ratings) | Both (optimized for implicit) |
| Optimization | SGD (stochastic gradient descent) | Alternates user/item optimization |
| Parallelization | Limited | Highly parallelizable |
| Missing data | Treated as unknown | Can weight by confidence |
| Scale | Medium datasets | Large-scale (Spotify uses this) |
| Library | Surprise | Implicit, Spark MLlib |

> **Critical Insight:** In production, most recommendation data is implicit (clicks, views, purchases, time spent) not explicit (ratings). ALS with confidence weighting (c_ui = 1 + alpha * interaction_count) is the standard approach for implicit feedback. The key insight from the Hu et al. 2008 paper is treating all unobserved interactions as negatives with low confidence, not as missing data.

---

## Content-Based Filtering

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

# Build item-item similarity from content features (e.g., descriptions)
tfidf = TfidfVectorizer(stop_words='english')
tfidf_matrix = tfidf.fit_transform(items_df['description'])
content_similarity = cosine_similarity(tfidf_matrix)

def content_based_recommend(user_history, content_sim, n=5):
    """Recommend items most similar to what user has liked."""
    scores = sum(content_sim[idx] for idx in user_history)
    for idx in user_history:
        scores[idx] = -1  # Exclude already-seen
    return np.argsort(scores)[::-1][:n]
```

### Content Features by Domain

| Domain | Content Features |
|--------|-----------------|
| Movies | Genre, director, cast, plot keywords, year |
| Music | Genre, tempo, key, energy, artist, lyrics |
| E-commerce | Category, brand, price range, attributes |
| News | Topic, entities, sentiment, recency |
| Books | Genre, author, length, themes, writing style |

---

## Hybrid Approaches

### Hybridization Strategies

| Strategy | Method | Example |
|----------|--------|---------|
| Weighted | Combine scores: alpha*CF + (1-alpha)*CB | Simple ensemble |
| Switching | Use CB for cold-start, CF for warm users | Fallback strategy |
| Feature Augmentation | CF predictions as features for CB model | Stacking |
| Cascade | CF generates candidates, CB re-ranks | Two-stage pipeline |
| Meta-level | One model's output feeds another | Content features -> CF model |

```python
def hybrid_recommend(user_id, item_ids, cf_model, cb_model, user_history_count):
    """Dynamic weighting: more CF as user history grows."""
    alpha = 0.3 if user_history_count < 5 else 0.5 if user_history_count < 20 else 0.8
    
    # Normalize and combine scores from both models
    cf_scores = normalize(cf_model.predict_scores(user_id, item_ids))
    cb_scores = normalize(cb_model.predict_scores(user_id, item_ids))
    hybrid_scores = alpha * cf_scores + (1 - alpha) * cb_scores
    return item_ids[np.argsort(hybrid_scores)[::-1]]
```

---

## The Cold-Start Problem

### Types of Cold Start

| Type | Problem | Solutions |
|------|---------|-----------|
| New User | No interaction history | Content-based, demographics, onboarding quiz |
| New Item | No ratings/interactions | Content features, editorial curation |
| New System | No data at all | Popularity-based, editorial, knowledge-based |

### Solutions

```python
# Strategy 1: Popularity-based fallback for new users
def popular_items_fallback(interactions_df, n=10, time_window_days=7):
    """Recommend trending items when no user history exists."""
    recent = interactions_df[interactions_df['timestamp'] >= 
             pd.Timestamp.now() - pd.Timedelta(days=time_window_days)]
    return recent.groupby('item_id').size().nlargest(n).index.tolist()

# Strategy 2: Content-based for new items (find similar existing items)
# Strategy 3: Onboarding quiz to collect explicit preferences
# Strategy 4: Epsilon-greedy exploration (random items with probability epsilon)
def epsilon_greedy_recommend(user_id, model, items, epsilon=0.1):
    """Mix exploitation (best prediction) with exploration (random)."""
    if np.random.random() < epsilon:
        return np.random.choice(items)
    scores = model.predict(user_id, items)
    return items[np.argmax(scores)]
```

> **Critical Insight:** Cold-start is solved in practice through graceful degradation: (1) For brand-new users: show popular/trending items, (2) After 1-2 interactions: switch to content-based, (3) After 5-10 interactions: blend in collaborative filtering, (4) After 20+ interactions: rely primarily on CF. This "cascade of confidence" ensures users always see something reasonable.

---

## Implicit vs Explicit Feedback

| Aspect | Explicit Feedback | Implicit Feedback |
|--------|------------------|-------------------|
| What it is | Ratings, reviews, likes | Views, clicks, purchases, time spent |
| Availability | Sparse (few users rate) | Abundant (every interaction counts) |
| Noise level | Low (intentional signal) | High (accidental clicks, position bias) |
| Negative signal | Low ratings exist | Absence doesn't mean dislike |
| Scale | 1-5 stars | Continuous (count, duration) |
| Example | Netflix star rating | Netflix watch time, completion rate |

```python
# Hu et al. 2008: Confidence-weighted matrix factorization
# r_ui = 1 if interacted, else 0 (preference)
# c_ui = 1 + alpha * interaction_count (confidence: more interactions = more certain)

# In production: weight different interaction types
interaction_weights = {
    'view': 1, 'add_to_cart': 3, 'purchase': 5,
    'repeat_purchase': 8, 'review_positive': 10, 'return': -5
}
```

---

## Evaluation Metrics for Recommendations

```python
import numpy as np

def precision_at_k(actual, predicted, k):
    """What fraction of top-k recommendations are relevant?"""
    return len(set(predicted[:k]) & set(actual)) / k

def recall_at_k(actual, predicted, k):
    """What fraction of relevant items appear in top-k?"""
    return len(set(predicted[:k]) & set(actual)) / len(actual) if actual else 0

def ndcg_at_k(actual, predicted, k):
    """Normalized Discounted Cumulative Gain at k."""
    dcg = sum(1/np.log2(i+2) for i, item in enumerate(predicted[:k]) if item in actual)
    ideal_dcg = sum(1/np.log2(i+2) for i in range(min(len(actual), k)))
    return dcg / ideal_dcg if ideal_dcg > 0 else 0

def hit_rate_at_k(actual, predicted, k):
    """Does at least one relevant item appear in top-k?"""
    return 1 if len(set(predicted[:k]) & set(actual)) > 0 else 0

def coverage(all_items, all_recommendations):
    """What fraction of catalog items ever get recommended?"""
    recommended = set(item for recs in all_recommendations for item in recs)
    return len(recommended) / len(all_items)
```

### Metrics Comparison

| Metric | Measures | Perspective |
|--------|----------|-------------|
| Precision@k | Relevance of recommendations | User satisfaction |
| Recall@k | Completeness of relevant items | Coverage of interests |
| NDCG@k | Ranking quality (position matters) | Did best items appear first? |
| Hit Rate@k | Any relevant item in top-k | Basic usefulness |
| MAP | Average precision across users | Overall system quality |
| Coverage | Item catalog usage | Business value (long tail) |
| Diversity | Variety in recommendations | Exploration, serendipity |
| Novelty | How unexpected recommendations are | Discovery value |

> **Critical Insight:** Offline metrics (NDCG, Precision@k) don't always correlate with online success (clicks, purchases, engagement). Always validate with A/B tests. A model with lower NDCG might drive more revenue if it recommends higher-margin items. The best evaluation is business metric improvement measured online.

---

## Real-World Systems

### Netflix

- **Approach**: Hybrid (matrix factorization + content + contextual)
- **Key insight**: Different algorithms for different UI rows (trending, because you watched, top picks)
- **Scale**: 200M+ users, 15K+ titles
- **Personalization**: Even thumbnail images are personalized

### Spotify (Discover Weekly)

- **Approach**: Collaborative filtering (ALS) + NLP on lyrics/podcasts + audio features (CNN)
- **Key insight**: Implicit feedback (skip rate, repeat plays, playlist additions)
- **Cold start**: Music genome project + editorial playlists

### Amazon

- **Approach**: Item-based CF ("customers who bought X also bought Y")
- **Key insight**: Item similarity is more stable than user similarity
- **Scale**: 300M+ products, real-time personalization
- **Architecture**: Two-stage: candidate generation (fast, approximate) + ranking (accurate, expensive)

### Two-Stage Architecture (Industry Standard)

```python
# Stage 1: Candidate Generation (fast, recall-focused)
# Narrow millions -> ~1000 via ANN (FAISS/Annoy/ScaNN) on embeddings
def candidate_generation(user_id, model, n_candidates=1000):
    user_embedding = model.get_user_embedding(user_id)
    return ann_index.search(user_embedding, n_candidates)

# Stage 2: Ranking (slower, precision-focused)
# Score ~1000 candidates with rich features + business rules
def ranking(user_id, candidates, ranking_model):
    features = extract_features(user_id, candidates)
    scores = ranking_model.predict(features)
    scores = apply_diversity_rules(apply_freshness_boost(scores))
    return sorted(zip(candidates, scores), key=lambda x: -x[1])
```

---

## Production Considerations

Key challenges: latency (<100ms via precomputation + ANN), scale (distributed ALS on Spark), position bias (inverse propensity weighting), filter bubbles (diversity injection), popularity bias (long-tail calibration), and validation (A/B tests with interleaving).

---

## Interview Talking Points

> "I approach recommendation systems as a two-stage problem: candidate generation for recall (ANN search over embeddings) and ranking for precision (gradient boosting over rich features). This architecture scales because the expensive ranking only runs over ~1000 candidates, not millions of items."

> "For cold-start users, I implement a confidence-weighted hybrid: start with popular/trending items, blend in content-based after 2-3 interactions, and gradually shift to collaborative filtering as history builds. The transition is smooth — not a hard switch."

> "Offline metrics like NDCG are necessary but not sufficient. At [company], I ran an A/B test where Model B had lower offline NDCG than Model A but drove 8% higher engagement. The offline metric missed the value of diversity and serendipity that users actually responded to."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Using only explicit ratings (sparse!) | Leverage implicit signals (views, clicks, time spent) |
| Evaluating only on offline metrics | Validate with A/B tests measuring business KPIs |
| Ignoring the cold-start problem | Design graceful degradation strategy from day one |
| Recommending only popular items | Add diversity and long-tail exploration |
| Not filtering already-consumed items | Always remove items user has seen/bought |
| Computing user similarity in real-time | Precompute and cache, update periodically |
| Using raw ratings without bias correction | Subtract user mean and item mean biases |
| Ignoring position bias in training data | Apply inverse propensity scoring or randomize |
| Optimizing only for clicks (engagement) | Include quality signals (dwell time, completion, returns) |
| Not handling the "Harry Potter" problem | Down-weight universally popular items everyone interacts with |

---

## Rapid-Fire Q&A

**Q1: What is the difference between user-based and item-based collaborative filtering?**
A: User-based finds similar users and recommends what they liked. Item-based finds similar items to what you already liked. Item-based is preferred in production because item similarities are more stable and can be precomputed.

**Q2: How does matrix factorization work for recommendations?**
A: It decomposes the sparse user-item matrix into two low-rank matrices (user embeddings and item embeddings). The predicted rating is the dot product of user and item vectors. This captures latent factors (e.g., "action-preference" dimension) that explain the rating patterns.

**Q3: What is the cold-start problem and how do you solve it?**
A: No interaction history for new users/items means CF cannot make predictions. Solutions: content-based features for new items, popularity/demographic recommendations for new users, onboarding preferences, and explore-exploit strategies.

**Q4: Why is implicit feedback preferred over explicit in production?**
A: Implicit feedback (clicks, views, purchases) is 100-1000x more abundant than ratings. Every user interaction generates signal, while <1% of users leave explicit ratings. The challenge is that absence of interaction doesn't necessarily mean dislike.

**Q5: What is ALS and when would you use it?**
A: Alternating Least Squares alternates between fixing user factors and optimizing item factors, then vice versa. It's preferred for implicit feedback because it can incorporate confidence weighting, and it's highly parallelizable (Spotify uses it at scale).

**Q6: How do you evaluate a recommendation system?**
A: Offline: NDCG@k, Precision@k, Recall@k, Hit Rate, MAP. Online: click-through rate, conversion rate, engagement time, revenue per session. Always complement offline with A/B tests.

**Q7: What is a two-stage recommendation architecture?**
A: Stage 1 (candidate generation): fast retrieval of ~1000 candidates from millions using embeddings + ANN. Stage 2 (ranking): precise scoring of candidates using rich features and a complex model. This balances scale with accuracy.

**Q8: How do you handle popularity bias?**
A: Popular items dominate because they have more interactions. Solutions: calibration (ensure recommendations match user's genre distribution), inverse popularity weighting in training, diversity constraints in serving, and long-tail boosting.

**Q9: What is the explore-exploit tradeoff in recommendations?**
A: Exploitation shows items the model is confident about (safe, relevant). Exploration shows uncertain items to gather information (helps cold items and discovers new preferences). Balance via epsilon-greedy, Thompson sampling, or UCB.

**Q10: How does Netflix personalize recommendations?**
A: Multiple algorithms for different UI rows, personalized thumbnails, contextual signals (time, device, recent activity), and a hybrid of CF + content + deep learning. They A/B test extensively — everything from algorithms to artwork.

---

## ASCII Cheat Sheet

```
+------------------------------------------------------------------+
|               RECOMMENDATION SYSTEM DECISION TREE                 |
+------------------------------------------------------------------+
|                                                                    |
|  What data do you have?                                            |
|  |                                                                 |
|  +-- Only item features (new system)                               |
|  |   --> Content-Based Filtering                                   |
|  |                                                                 |
|  +-- User-item interactions (ratings/clicks)                       |
|  |   +-- Explicit ratings? --> SVD / Matrix Factorization          |
|  |   +-- Implicit (clicks/views)? --> ALS with confidence          |
|  |   +-- Both? --> Hybrid approach                                 |
|  |                                                                 |
|  +-- Rich features + interactions                                  |
|      --> Two-stage: Embedding retrieval + Feature-rich ranker      |
|                                                                    |
+------------------------------------------------------------------+
|               COLD-START STRATEGY                                  |
+------------------------------------------------------------------+
|                                                                    |
|  User interaction count:                                           |
|                                                                    |
|  0 interactions   --> Popular/Trending items                       |
|  1-3 interactions --> Content-based (similar to liked items)       |
|  4-10 interactions --> Blend: 50% CF + 50% Content                 |
|  10+ interactions  --> Primarily CF (80%) + Content (20%)          |
|  50+ interactions  --> Full collaborative filtering                |
|                                                                    |
+------------------------------------------------------------------+
|               TWO-STAGE ARCHITECTURE                               |
+------------------------------------------------------------------+
|                                                                    |
|  ALL ITEMS (millions)                                              |
|       |                                                            |
|       v                                                            |
|  [Candidate Generation] -- ANN / embedding similarity              |
|       |                                                            |
|       v                                                            |
|  ~1000 CANDIDATES                                                  |
|       |                                                            |
|       v                                                            |
|  [Ranking Model] -- Rich features, GBM/NN                         |
|       |                                                            |
|       v                                                            |
|  ~10-50 RESULTS                                                    |
|       |                                                            |
|       v                                                            |
|  [Business Rules] -- Diversity, freshness, filtering               |
|       |                                                            |
|       v                                                            |
|  FINAL RECOMMENDATIONS                                             |
|                                                                    |
+------------------------------------------------------------------+
|               EVALUATION METRICS                                   |
+------------------------------------------------------------------+
|                                                                    |
|  RELEVANCE:    Precision@k, Recall@k, NDCG@k, Hit Rate@k          |
|  RANKING:      MAP, MRR                                            |
|  BEYOND:       Coverage, Diversity, Novelty, Serendipity           |
|  BUSINESS:     CTR, Conversion, Revenue/Session, Retention         |
|                                                                    |
|  GOLDEN RULE: Offline metrics inform, A/B tests decide.            |
|                                                                    |
+------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 34 of 45*
