# 🎯 Topic 50: Search & Ranking Systems

> *"Ranking is the most impactful ML problem in tech — every major product (Google, Amazon, Netflix, Spotify, LinkedIn) is fundamentally a ranking system. A 0.1% improvement in ranking quality can translate to billions in revenue."*

---

## 📑 Table of Contents

1. [Information Retrieval Fundamentals](#1-information-retrieval-fundamentals)
2. [Evaluation Metrics for Ranking](#2-evaluation-metrics-for-ranking)
3. [Learning-to-Rank Approaches](#3-learning-to-rank-approaches)
4. [Feature Engineering for Ranking](#4-feature-engineering-for-ranking)
5. [Position Bias and Debiasing](#5-position-bias-and-debiasing)
6. [Online vs Offline Evaluation](#6-online-vs-offline-evaluation)
7. [Interleaving Experiments](#7-interleaving-experiments)
8. [Query Understanding](#8-query-understanding)
9. [Re-Ranking and Multi-Stage Architecture](#9-re-ranking-and-multi-stage-architecture)
10. [Practical System Examples](#10-practical-system-examples)
11. [Interview Talking Points](#11-interview-talking-points)
12. [Common Mistakes](#12-common-mistakes)
13. [Rapid-Fire Q&A](#13-rapid-fire-qa)
14. [ASCII Cheat Sheet](#14-ascii-cheat-sheet)

---

## 1. Information Retrieval Fundamentals

### The Core Problem

Given a query `q` and a corpus of documents `D = {d1, d2, ..., dn}`, produce a ranked list of documents ordered by relevance to `q`.

### Classical Models

**TF-IDF (Term Frequency - Inverse Document Frequency):**
```
TF-IDF(t, d) = TF(t, d) × IDF(t)

TF(t, d) = count(t in d) / |d|
IDF(t) = log(N / df(t))

Where:
- N = total documents in corpus
- df(t) = number of documents containing term t
```

**BM25 (Best Match 25) — Industry Standard:**
```
BM25(q, d) = Σ_{t ∈ q} IDF(t) × [TF(t,d) × (k₁ + 1)] / [TF(t,d) + k₁ × (1 - b + b × |d|/avgdl)]

Where:
- k₁ = 1.2 (term frequency saturation)
- b = 0.75 (document length normalization)
- avgdl = average document length
```

> **Critical Insight:** BM25 remains a strong baseline in 2024+. Many production systems still use BM25 for initial retrieval (recall-oriented) before applying ML re-ranking (precision-oriented). Never dismiss it as "too simple."

### Vector Space Model & Semantic Search

Modern systems combine lexical (BM25) with semantic (embedding) retrieval:

```
Lexical:   query_terms ∩ doc_terms (exact match)
Semantic:  cos(embed(query), embed(doc)) (meaning match)
Hybrid:    α × BM25_score + (1-α) × semantic_score
```

---

## 2. Evaluation Metrics for Ranking

### Precision and Recall at K

```
Precision@K = |relevant docs in top K| / K
Recall@K = |relevant docs in top K| / |total relevant docs|
```

### Mean Average Precision (MAP)

```
AP(q) = (1/|relevant|) × Σ_{k=1}^{n} [Precision@k × rel(k)]

MAP = (1/|Q|) × Σ_{q ∈ Q} AP(q)
```

Where `rel(k) = 1` if document at position k is relevant, else 0.

### Normalized Discounted Cumulative Gain (NDCG)

The most important metric for graded relevance (not just binary relevant/irrelevant):

```
DCG@K = Σ_{i=1}^{K} (2^{rel_i} - 1) / log₂(i + 1)

NDCG@K = DCG@K / IDCG@K

Where IDCG@K = DCG of the ideal (perfect) ranking
```

**Properties:**
- Handles graded relevance (e.g., 0-4 scale)
- Position-discounted (top results matter more)
- Normalized to [0, 1] for cross-query comparison

### Mean Reciprocal Rank (MRR)

For navigational queries (one correct answer):

```
RR(q) = 1 / rank_of_first_relevant_result
MRR = (1/|Q|) × Σ_{q ∈ Q} RR(q)
```

> **Critical Insight:** Choose metrics based on the search task: **NDCG** for general web/product search (graded relevance), **MRR** for navigational queries (one right answer), **MAP** for recall-oriented tasks (find all relevant items), **Precision@1** for voice assistants (only show one result).

### Metric Comparison Table

| Metric | Relevance Type | Position Sensitive | Best For |
|--------|---------------|-------------------|----------|
| Precision@K | Binary | No (within K) | Quick assessment |
| Recall@K | Binary | No | Retrieval completeness |
| MAP | Binary | Yes (via AP) | Research benchmarks |
| NDCG@K | Graded | Yes (log discount) | Production ranking |
| MRR | Binary | Yes (first hit) | Navigational/QA |
| ERR | Graded | Yes (cascade model) | Web search (user stops) |

---

## 3. Learning-to-Rank Approaches

### Overview

Learning-to-Rank (LTR) uses ML to learn a scoring function that produces good rankings according to an evaluation metric.

### Pointwise Approach

**Treat ranking as regression/classification on individual documents:**

```
Input: (query, document) → features
Output: relevance score (regression) or relevance class (classification)

Models: Linear Regression, Gradient Boosted Trees, Neural Nets
Loss: MSE, Cross-Entropy

Example: Predict P(click | query, document) for each document independently
```

**Limitation:** Ignores relative ordering — optimizes per-document score, not the list.

### Pairwise Approach

**Learn which of two documents should rank higher:**

```
Input: (query, doc_i, doc_j) where rel(doc_i) > rel(doc_j)
Output: P(doc_i should rank above doc_j)

Loss: hinge loss or cross-entropy on pairs
Models: RankSVM, RankNet, LambdaRank

RankNet Loss:
L = -Σ_{(i,j): rel_i > rel_j} [log σ(s_i - s_j)]
```

**Key Algorithm — LambdaMART:**
- Combines LambdaRank (pairwise gradients weighted by NDCG delta) with MART (gradient boosted trees)
- Industry standard for years (used at Microsoft Bing, Yahoo)
- Directly optimizes NDCG via gradient approximation

> **Critical Insight:** LambdaMART/XGBoost with pairwise loss remains extremely competitive. In many practical settings it matches or beats neural approaches, especially with <1M training examples. Neural models shine at scale (billions of examples) and when you have rich unstructured features (text, images).

### Listwise Approach

**Optimize the entire ranked list directly:**

```
Input: (query, [doc_1, doc_2, ..., doc_n]) with relevance labels
Output: Permutation that maximizes metric

Models: ListNet, ListMLE, ApproxNDAP, SoftRank
Loss: Directly approximates NDCG or other list-level metrics

ListNet: minimize KL-divergence between predicted and true top-1 probabilities
P(d_i is top-1) = exp(s_i) / Σ_j exp(s_j)  (Plackett-Luce model)
```

### Comparison of LTR Approaches

| Aspect | Pointwise | Pairwise | Listwise |
|--------|-----------|----------|----------|
| Training unit | Single doc | Document pair | Full list |
| Loss function | MSE/CE | Pair preference | List metric proxy |
| Metric awareness | None | Indirect | Direct |
| Computational cost | Low | O(n²) pairs | High |
| Data efficiency | High | Medium | Low |
| Handles ties | Poorly | Well | Well |
| Industry standard | Baseline | LambdaMART | Neural (at scale) |

---

## 4. Feature Engineering for Ranking

### Feature Categories

**1. Query-Document Match Features (Relevance)**

| Feature | Description |
|---------|------------|
| BM25 score | Lexical relevance |
| TF-IDF cosine similarity | Vector space match |
| Exact title match | Query appears in title |
| Query term coverage | % of query terms in doc |
| Embedding cosine similarity | Semantic match |
| Query-doc click-through rate | Historical behavioral |

**2. Document Quality Features (Authority)**

| Feature | Description |
|---------|------------|
| PageRank / authority score | Link-based importance |
| Domain authority | Site-level credibility |
| Content freshness | Time since last update |
| Content length | Depth of coverage |
| Spam score | Inverse spam likelihood |
| Review count / rating | Social proof |

**3. User/Context Features (Personalization)**

| Feature | Description |
|---------|------------|
| User click history | Past preferences |
| User purchase history | Buying patterns |
| Geographic location | Local relevance |
| Device type | Mobile vs desktop behavior |
| Time of day / day of week | Temporal patterns |
| User segment / cohort | Collaborative signals |

**4. Click/Engagement Signals**

| Feature | Description |
|---------|------------|
| Historical CTR at position | Position-normalized clicks |
| Dwell time | Time spent on result |
| Bounce rate | Quick returns = bad |
| Add-to-cart rate | E-commerce intent |
| Long click rate | Clicks with >30s dwell |

> **Critical Insight:** The most powerful features in production ranking systems are **behavioral** (clicks, purchases, dwell time), not textual. But behavioral features create a cold-start problem for new items and a rich-get-richer feedback loop. Always include exploration mechanisms.

### BM25 Deep Dive for Interviews

```python
import numpy as np
from collections import Counter

def bm25_score(query_terms, doc_terms, corpus_stats, k1=1.2, b=0.75):
    """
    Calculate BM25 score for a query-document pair.
    
    Parameters:
    -----------
    query_terms : list of str
    doc_terms : list of str
    corpus_stats : dict with 'N', 'avgdl', 'df' (doc frequency per term)
    """
    doc_len = len(doc_terms)
    doc_tf = Counter(doc_terms)
    score = 0.0
    
    for term in query_terms:
        tf = doc_tf.get(term, 0)
        df = corpus_stats['df'].get(term, 0)
        N = corpus_stats['N']
        avgdl = corpus_stats['avgdl']
        
        # IDF component (with smoothing)
        idf = np.log((N - df + 0.5) / (df + 0.5) + 1)
        
        # TF component with saturation and length normalization
        tf_norm = (tf * (k1 + 1)) / (tf + k1 * (1 - b + b * doc_len / avgdl))
        
        score += idf * tf_norm
    
    return score
```

---

## 5. Position Bias and Debiasing

### The Problem

Users click higher-ranked results more often **regardless of relevance**. This creates a feedback loop:

```
Item at position 1 → More clicks → Higher CTR feature → Ranked higher → Position 1...

Observed CTR = P(relevant) × P(examined | position)
                ↑ what we want    ↑ position bias
```

### Position Bias Model

```
P(click | query, doc, position) = P(examine | position) × P(click | examine, query, doc)

Examination probability decreases with position:
P(examine | pos) ≈ 1/pos^α  (where α ≈ 1 for web search)
```

### Debiasing Methods

**1. Inverse Propensity Weighting (IPW):**
```
Weight each click by 1/P(examine | position)

Debiased_CTR(doc) = Σ clicks / Σ (1/P(examine|pos)) × exposures

If position 1 has P(examine)=1.0 and position 5 has P(examine)=0.3:
A click at position 5 counts as 1/0.3 = 3.3 clicks (upweighted)
```

**2. Position as a Feature:**
```
Train model with position as input feature.
At inference time, set position = 1 for all candidates (counterfactual).
Model learns to separate position effect from true relevance.
```

**3. Randomization-Based Estimation:**
```
Occasionally randomize results to collect unbiased data.
Cost: user experience degradation.
Compromise: randomize in low-value slots (positions 5-10).
```

> **Critical Insight:** If you train a ranking model on click data without debiasing, you're essentially training the model to predict "what position was this shown at" rather than "how relevant is this." This is one of the most common and damaging mistakes in production ranking systems.

### Trust Bias

Users also trust higher-ranked results more (especially from authority sources like Google):
- A result at position 1 may be perceived as more relevant even if it's not
- This means even explicit judgments (ratings) can be position-biased

---

## 6. Online vs Offline Evaluation

### The Gap Problem

```
Offline metrics (NDCG on test set) ≠ Online metrics (engagement in production)

Common causes of divergence:
1. Position bias in training data
2. Stale training data (distribution shift)
3. Feedback loops (model influences its own training data)
4. User adaptation (users change behavior with new ranking)
5. Missing features in offline setting
```

### Offline Evaluation

| Method | Description | Limitation |
|--------|-------------|-----------|
| Hold-out test set | Standard ML evaluation | Position bias, stale data |
| Replay evaluation | Counterfactual from logs | Only evaluates items previously shown |
| Counterfactual estimation | IPS-weighted offline metrics | High variance |

### Online Evaluation

| Method | Description | Advantage |
|--------|-------------|-----------|
| A/B test | Random user split | Gold standard for causality |
| Interleaving | Mix results from two rankers | 10-100x more sensitive |
| Multi-armed bandit | Adaptive allocation | Faster convergence |

> **Critical Insight:** A model that improves NDCG@10 by 2% offline typically improves online engagement by much less (often 0.1-0.5%). The offline-to-online correlation is positive but noisy. This is why interleaving and A/B tests are essential before shipping.

---

## 7. Interleaving Experiments

### Team Draft Interleaving

Combine results from two rankers (A and B) into a single interleaved list shown to users:

```
Ranker A: [a1, a2, a3, a4, a5]
Ranker B: [b1, b2, b3, b4, b5]

Team Draft (alternating picks):
Interleaved: [a1, b1, a2, b2, a3, b3, ...]
              ↑ Team A  ↑ Team B

User clicks: positions 1, 3, 5
Team A clicks: 2 (positions 1, 3)
Team B clicks: 1 (position 5 if from B)

Winner: Ranker A (more clicks attributed)
```

### Why Interleaving is More Sensitive

- A/B test: compares average engagement across user groups
- Interleaving: compares within the SAME user session

```
Statistical power comparison:
- A/B test: needs ~100K sessions to detect 1% improvement
- Interleaving: needs ~1K sessions for the same detection

Reason: within-user comparison eliminates user-level variance
```

### Interleaving Variants

| Method | Description | Handles position bias? |
|--------|-------------|----------------------|
| Team Draft | Alternating picks | Partially |
| Balanced | Ensures equal representation | Yes |
| Probabilistic | Softmax-based mixing | Yes |
| Optimized | Minimizes variance | Yes |

---

## 8. Query Understanding

### Intent Classification

```
Query Types:
├── Navigational: "facebook login" → user wants a specific page
├── Informational: "how does photosynthesis work" → wants knowledge
├── Transactional: "buy iphone 15 pro" → wants to purchase
└── Local: "pizza near me" → wants nearby results
```

### Spell Correction

```
"reccommendation systems" → "recommendation systems"

Approaches:
1. Edit distance (Levenshtein) + language model
2. Noisy channel model: P(correction|typo) ∝ P(typo|correction) × P(correction)
3. Neural seq2seq models
4. Character n-gram embeddings for fuzzy matching
```

### Query Expansion

```
Original: "ML engineer salary"
Expanded: "ML engineer salary" OR "machine learning engineer compensation" OR "ML engineer pay"

Methods:
1. Synonym expansion (thesaurus/embedding neighbors)
2. Pseudo-relevance feedback (add terms from top results)
3. Neural query expansion (T5/GPT-based rewriting)
```

### Query Relaxation

When a query is too specific and returns few results:

```
"red nike air max 90 size 11 under $100" → few results
Relax: "red nike air max 90 size 11" → more results
Relax: "nike air max 90 size 11" → even more results
```

> **Critical Insight:** Query understanding is often the highest-leverage improvement in search systems. A perfectly ranked list for the wrong interpretation of the query is useless. Invest in intent classification and query rewriting before investing in better ranking models.

---

## 9. Re-Ranking and Multi-Stage Architecture

### The Funnel Architecture

Production search systems cannot run expensive models on millions of candidates. Solution: multi-stage pipeline.

```
┌─────────────────────────────────────────────────┐
│  CORPUS (billions of documents)                  │
└───────────────────────┬─────────────────────────┘
                        │ Retrieval (BM25 + ANN)
                        │ Latency: <10ms
                        │ Candidates: ~1000
                        ▼
┌─────────────────────────────────────────────────┐
│  FIRST-STAGE RANKING (lightweight model)         │
│  Features: BM25, embedding sim, doc quality      │
│  Model: Linear / small GBDT                      │
│  Latency: <20ms, Candidates: ~100               │
└───────────────────────┬─────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────┐
│  SECOND-STAGE RE-RANKING (heavy model)           │
│  Features: All features + cross-attention        │
│  Model: Deep neural network / BERT-based         │
│  Latency: <50ms, Candidates: ~20                │
└───────────────────────┬─────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────┐
│  BUSINESS LOGIC LAYER                            │
│  Diversity, freshness boost, policy filters      │
│  Sponsored results interleaving                  │
│  Final list: 10 results shown to user            │
└─────────────────────────────────────────────────┘
```

### Why Multi-Stage?

| Stage | Candidates | Model Complexity | Latency Budget |
|-------|-----------|-----------------|---------------|
| Retrieval | 10M → 1000 | Simple (inverted index + ANN) | <10ms |
| L1 Ranking | 1000 → 100 | Medium (GBDT) | <20ms |
| L2 Re-ranking | 100 → 20 | Heavy (BERT/transformer) | <50ms |
| Business Rules | 20 → 10 | Rule-based | <5ms |

### Approximate Nearest Neighbor (ANN) for Retrieval

```python
# Embedding-based retrieval at scale
# Libraries: FAISS, ScaNN, Annoy, HNSW

import faiss

# Index 1M document embeddings (768-dim)
dimension = 768
index = faiss.IndexIVFPQ(
    faiss.IndexFlatL2(dimension),
    dimension,
    n_list=1024,    # Voronoi cells
    m=64,           # PQ sub-vectors
    bits=8          # bits per sub-vector
)

# Query: find top-100 similar documents in <5ms
distances, indices = index.search(query_embedding, k=100)
```

> **Critical Insight:** The biggest ranking improvements often come from **improving retrieval recall** (getting relevant items into the candidate set) rather than improving the re-ranker. If a relevant document is never retrieved, no amount of re-ranking can surface it. Invest in retrieval diversity.

---

## 10. Practical System Examples

### Google Web Search

```
Architecture:
1. Query Understanding: intent, entities, spell correction
2. Retrieval: Inverted index (trillion+ pages) → ~1000 candidates
3. Ranking: BERT-based model (as of 2019 BERT update)
4. Features: PageRank, content quality, freshness, E-E-A-T signals
5. Personalization: search history, location, language
6. Diversity: avoid duplicate domains, ensure topic variety
```

### Amazon Product Search

```
Key differences from web search:
1. Purchase intent dominates (transactional queries)
2. Revenue optimization alongside relevance
3. Features: sales velocity, conversion rate, reviews, price, Prime eligibility
4. Cold-start: new products have no behavioral data
5. Inventory/availability as hard constraint
6. Sponsored products interleaved with organic

Ranking Formula (simplified):
Score = Relevance × Conversion_Probability × Revenue_Weight
```

### Spotify Playlist / Music Recommendation

```
Hybrid approach:
1. Collaborative filtering (user-item interactions)
2. Content-based (audio features, genre, mood)
3. NLP on lyrics, artist bios, playlist names
4. Contextual: time of day, recent listening, activity

Exploration vs Exploitation:
- Exploit: songs similar to what user loves
- Explore: new artists/genres to expand taste
- Balance via epsilon-greedy or Thompson Sampling
```

### LinkedIn Feed Ranking

```
Multi-objective optimization:
- Engagement (likes, comments, shares)
- Session time
- Job applications
- Connection growth
- Content creator satisfaction

Two-tower architecture:
- User tower: profile, activity, network features
- Content tower: post features, author features, engagement history
- Final ranking: interaction features + cross-features
```

---

## 11. Interview Talking Points

> **"How would you design a product search ranking system for an e-commerce platform?"**
>
> "I'd build a multi-stage funnel. Retrieval stage: BM25 on product titles + embedding-based ANN on a bi-encoder trained on (query, clicked product) pairs, retrieving ~500 candidates. L1 ranking: a GBDT model using query-product text match, category match, historical CTR (position-debiased), price competitiveness, review score, and availability. L2 re-ranking: a cross-encoder neural model on the top 50 that captures query-product interaction deeply. Business layer: diversity (don't show 10 variants of the same product), freshness boost for new products, and sponsored ad interleaving. Key metrics: NDCG@10 offline, revenue per search and click-through rate online."

> **"How do you handle position bias in your click data?"**
>
> "Three approaches I'd combine: (1) Include position as a feature during training, then set position=1 at inference for all candidates — this lets the model separate position effects from relevance. (2) Use inverse propensity weighting where I estimate P(examine|position) from randomization experiments and upweight clicks at lower positions. (3) Run periodic randomization in low-traffic slots to collect unbiased relevance judgments. I'd validate the debiasing by checking that NDCG computed on randomized data correlates better with the debiased model's predictions."

> **"What's the difference between NDCG and MAP, and when would you use each?"**
>
> "MAP uses binary relevance (relevant or not) and treats each relevant document equally regardless of grade. NDCG handles graded relevance (e.g., highly relevant, somewhat relevant, irrelevant) and weights highly relevant documents more via the 2^rel - 1 gain function. I'd use NDCG for product search where a perfect match is much more valuable than a partial match. I'd use MAP for recall-oriented tasks like legal document search where finding ALL relevant documents matters and they're equally important."

---

## 12. Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Training on click data without debiasing | Apply IPW or position-as-feature debiasing |
| Using only NDCG offline to decide ship/no-ship | Run interleaving or A/B test before shipping |
| Building complex re-ranker before fixing retrieval | Audit retrieval recall first — can't rank what you don't retrieve |
| Ignoring query understanding | Invest in spell correction, intent detection, query expansion |
| Using pointwise loss when ranking matters | Use pairwise (LambdaMART) or listwise objectives |
| One model for all query types | Segment by intent (navigational vs exploratory) with different strategies |
| Ignoring cold-start items | Boost new items with exploration, use content features |
| Optimizing only relevance | Multi-objective: relevance + diversity + freshness + revenue |
| Ignoring latency constraints | Design for latency budget per stage — fast retrieval, heavier re-ranking |
| Using human labels without inter-annotator agreement | Measure and report agreement; use graded labels, not binary |

---

## 13. Rapid-Fire Q&A

**Q1: What's the difference between precision-oriented and recall-oriented retrieval?**
A: Precision-oriented returns fewer but highly relevant results (product search top-10). Recall-oriented ensures no relevant document is missed (legal discovery, medical literature). The retrieval stage should be recall-oriented; ranking stages add precision.

**Q2: Why is BM25's k1 parameter important?**
A: k1 controls term frequency saturation. With k1=0, TF doesn't matter (boolean). With k1→infinity, TF grows linearly (no saturation). k1=1.2 means repeating a term helps but with diminishing returns — the 10th occurrence adds much less than the 1st.

**Q3: How does NDCG handle queries with different numbers of relevant documents?**
A: NDCG normalizes by IDCG (ideal DCG), so a query with 3 relevant docs and a query with 30 relevant docs both have NDCG in [0,1]. This makes averaging across queries meaningful.

**Q4: What's the cascade model assumption in ranking?**
A: Users examine results top-to-bottom and stop once satisfied. This implies: (1) only top results matter, (2) examination probability decreases with rank, (3) a highly relevant result early "steals" attention from later results.

**Q5: How do two-tower models work for retrieval?**
A: One tower encodes the query into an embedding, another encodes documents. At serving time, document embeddings are pre-computed and indexed (ANN). Only the query tower runs in real-time. Relevance = dot product or cosine similarity between the two embeddings.

**Q6: Why might a model that improves NDCG offline hurt online metrics?**
A: (1) Position bias in offline data makes improvements illusory. (2) The model over-exploits popular items, reducing diversity. (3) The model breaks an implicit user contract (e.g., users expect freshness). (4) Presentation bias — layout changes affect clicks independent of ranking.

**Q7: What is the "exposure bias" problem in ranking?**
A: Items that have never been shown have no behavioral features (CTR, dwell time) and thus rank low, creating a self-reinforcing cycle. Solutions: exploration bonuses, randomization, using content-based features for cold items.

**Q8: How does interleaving detect smaller effects than A/B tests?**
A: A/B tests compare means across groups (between-user variance is high). Interleaving compares within the same user session (eliminates user-level variance). The paired comparison is statistically more powerful.

**Q9: What's the difference between hard negatives and random negatives in training?**
A: Random negatives are easy (model learns quickly). Hard negatives are documents that are similar but not relevant — they force the model to make fine-grained distinctions. Training with hard negatives significantly improves ranking quality but requires careful mining.

**Q10: How would you measure if your search system has a diversity problem?**
A: (1) Measure subtopic recall (does the top-10 cover different aspects of ambiguous queries?). (2) Track "reformulation rate" — users reformulating means they didn't find what they wanted. (3) Compute intra-list similarity — if top results are too similar, diversity is low. (4) Segment by query type and check abandonment rates.

---

## 14. ASCII Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                    SEARCH & RANKING SYSTEMS                          ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  MULTI-STAGE PIPELINE                                                ║
║  ┌──────────────────────────────────────────┐                        ║
║  │ Corpus (10⁹) ──► Retrieval (10³)         │  BM25 + ANN           ║
║  │     ──► L1 Rank (10²)                    │  GBDT                 ║
║  │     ──► L2 Re-rank (10¹)                 │  Neural/BERT          ║
║  │     ──► Business Rules (final)           │  Diversity + Ads      ║
║  └──────────────────────────────────────────┘                        ║
║                                                                      ║
║  EVALUATION METRICS                                                  ║
║  ┌──────────────────────────────────────────┐                        ║
║  │  NDCG@K = DCG@K / IDCG@K                │ Graded relevance       ║
║  │  DCG = Σ (2^rel - 1) / log₂(i+1)       │ Position-discounted    ║
║  │  MAP = mean of AveragePrecision          │ Binary relevance       ║
║  │  MRR = mean of 1/rank_first_relevant    │ One right answer       ║
║  └──────────────────────────────────────────┘                        ║
║                                                                      ║
║  LEARNING-TO-RANK                                                    ║
║  ┌──────────────────────────────────────────┐                        ║
║  │  Pointwise: score(q, d) → regression     │ Simple, ignores list   ║
║  │  Pairwise:  P(d_i > d_j) → preference    │ LambdaMART (GBDT)     ║
║  │  Listwise:  optimize full list metric     │ Neural (at scale)     ║
║  └──────────────────────────────────────────┘                        ║
║                                                                      ║
║  POSITION BIAS                                                       ║
║  ┌──────────────────────────────────────────┐                        ║
║  │  P(click) = P(examine|pos) × P(rel)     │                        ║
║  │                                          │                        ║
║  │  Pos 1: ████████████ (100% examined)     │                        ║
║  │  Pos 2: ████████░░░░ (70% examined)      │                        ║
║  │  Pos 3: ██████░░░░░░ (50% examined)      │                        ║
║  │  Pos 5: ███░░░░░░░░░ (25% examined)      │                        ║
║  │  Pos 10:█░░░░░░░░░░░ (10% examined)      │                        ║
║  │                                          │                        ║
║  │  Fix: IPW, position-as-feature, randomize│                        ║
║  └──────────────────────────────────────────┘                        ║
║                                                                      ║
║  KEY FEATURES FOR RANKING                                            ║
║  ┌──────────────────────────────────────────┐                        ║
║  │  Text Match: BM25, embeddings, coverage  │                        ║
║  │  Authority:  PageRank, domain trust      │                        ║
║  │  Behavioral: CTR, dwell time, purchases  │                        ║
║  │  Freshness:  recency, update frequency   │                        ║
║  │  Personal:   history, location, device   │                        ║
║  └──────────────────────────────────────────┘                        ║
║                                                                      ║
║  BM25 FORMULA                                                        ║
║  score = Σ IDF(t) × [TF×(k₁+1)] / [TF + k₁×(1-b+b×|d|/avgdl)]    ║
║  k₁=1.2 (saturation), b=0.75 (length norm)                          ║
║                                                                      ║
║  ONLINE vs OFFLINE                                                   ║
║  Offline NDCG ≠ Online engagement (gap exists!)                      ║
║  Always validate with interleaving or A/B test                       ║
║  Interleaving: 100x more sensitive than A/B                          ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 50 of 51*
