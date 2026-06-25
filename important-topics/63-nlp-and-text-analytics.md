# 🎯 Topic 63: NLP & Text Analytics

> *"From bag-of-words to transformers — understanding how machines process human language, explained for interview confidence."*

---

## 📋 Table of Contents

1. [Text Preprocessing Pipeline](#1-text-preprocessing-pipeline)
2. [Bag of Words vs TF-IDF](#2-bag-of-words-vs-tf-idf)
3. [Word Embeddings](#3-word-embeddings)
4. [Modern NLP: Transformers, BERT, GPT](#4-modern-nlp-transformers-bert-gpt)
5. [Text Classification Approaches](#5-text-classification-approaches)
6. [Sentiment Analysis](#6-sentiment-analysis)
7. [Topic Modeling](#7-topic-modeling)
8. [Document Similarity & Cosine Similarity](#8-document-similarity--cosine-similarity)
9. [Traditional NLP vs LLMs Decision Framework](#9-traditional-nlp-vs-llms-decision-framework)
10. [Practical Applications](#10-practical-applications)
11. [Interview Talking Points](#11-interview-talking-points)
12. [Common Mistakes](#12-common-mistakes)
13. [Rapid-Fire Q&A](#13-rapid-fire-qa)
14. [ASCII Cheat Sheet](#14-ascii-cheat-sheet)

---

## 1. Text Preprocessing Pipeline

### The Standard Pipeline

```
Raw Text
  │
  ├── 1. Lowercasing          → "Hello World" → "hello world"
  ├── 2. Remove HTML/URLs     → Strip tags, links
  ├── 3. Remove punctuation   → "hello!" → "hello"
  ├── 4. Tokenization         → "hello world" → ["hello", "world"]
  ├── 5. Stop word removal    → Remove "the", "is", "a"
  ├── 6. Stemming/Lemmatization → "running" → "run"
  └── 7. Vectorization        → Convert to numbers
```

### Stemming vs Lemmatization

| Aspect | Stemming | Lemmatization |
|--------|----------|---------------|
| Method | Chop word endings (rules-based) | Dictionary lookup for base form |
| Speed | Fast | Slower |
| Accuracy | Crude (can produce non-words) | Accurate (real dictionary words) |
| Example | "better" → "better"/"bett" | "better" → "good" |
| Example | "running" → "run" | "running" → "run" |
| Example | "studies" → "studi" | "studies" → "study" |
| When to use | Speed matters, approximate is fine | Accuracy matters, downstream NLU |

### When to Skip Preprocessing Steps

| Step | Skip When |
|------|-----------|
| Lowercasing | Case carries meaning (NER, sentiment: "US" vs "us") |
| Stop words | Using n-grams, phrase detection, or deep learning |
| Stemming/Lemma | Using subword tokenization (BERT, GPT) |
| Punctuation removal | Sentiment ("!!!" = emphasis), code parsing |

> **Critical Insight**: Modern deep learning models (BERT, GPT) do their OWN tokenization (subword/BPE). Heavy preprocessing can actually HURT performance with these models. The full pipeline above is primarily for traditional ML approaches (BoW, TF-IDF + classifier).

---

## 2. Bag of Words vs TF-IDF

### Bag of Words (BoW)

**Concept**: Represent each document as a vector of word counts.

```
Doc 1: "the cat sat on the mat"
Doc 2: "the dog sat on the log"

Vocabulary: [cat, dog, log, mat, on, sat, the]
Doc 1 vector: [1, 0, 0, 1, 1, 1, 2]
Doc 2 vector: [0, 1, 1, 0, 1, 1, 2]
```

### TF-IDF (Term Frequency - Inverse Document Frequency)

**Concept**: Weight words by how important they are to a specific document relative to the corpus.

```
TF-IDF(word, doc) = TF(word, doc) × IDF(word)

TF = frequency of word in document / total words in document
IDF = log(total documents / documents containing word)
```

**Intuition**: Words that appear often in ONE document but rarely across ALL documents are the most informative.

### Comparison Table

| Aspect | Bag of Words | TF-IDF |
|--------|-------------|--------|
| Weighting | Raw counts (or binary) | Frequency × rarity |
| Common words | Dominate the vector | Down-weighted by IDF |
| Rare, informative words | Same weight as common | Up-weighted |
| Sparsity | High (most entries = 0) | High (same structure) |
| Order preserved | No (bag = unordered) | No |
| N-grams support | Yes (bigrams, trigrams) | Yes |
| Computation | Faster | Slightly more expensive |
| Typical accuracy | Lower | Higher (usually) |

### N-grams: Capturing Word Order

```
Unigrams: ["I", "love", "New", "York"]
Bigrams:  ["I love", "love New", "New York"]
Trigrams: ["I love New", "love New York"]
```

N-grams partially address BoW's inability to capture word order and phrases.

> **Critical Insight**: TF-IDF is still the go-to for many practical applications (search engines, document classification, feature engineering). It's simple, interpretable, fast, and surprisingly competitive. Don't dismiss it just because deep learning exists. Many production systems use TF-IDF + logistic regression as a strong baseline that's easy to maintain and debug.

---

## 3. Word Embeddings

### The Problem with BoW/TF-IDF

- Vectors are HUGE (dimension = vocabulary size, often 50K+)
- No semantic meaning: "happy" and "joyful" are as different as "happy" and "table"
- No generalization: Each word is an independent dimension

### Word2Vec — Two Architectures

**Skip-gram**: Given a word, predict its context words
```
Input: "cat"  →  Predict: "the", "sat", "on", "mat"
```

**CBOW (Continuous Bag of Words)**: Given context words, predict the center word
```
Input: "the", "___", "sat"  →  Predict: "cat"
```

### Word2Vec Intuition

- Words that appear in similar contexts get similar vectors
- Produces dense vectors (typically 100-300 dimensions)
- Captures semantic relationships through vector arithmetic

```
king - man + woman ≈ queen
Paris - France + Italy ≈ Rome
```

### GloVe (Global Vectors)

- Combines global co-occurrence statistics with local context
- Builds a co-occurrence matrix, then factorizes it
- Often performs similarly to Word2Vec
- Key difference: uses GLOBAL corpus statistics vs Word2Vec's local window

### Embedding Comparison

| Aspect | Word2Vec | GloVe | FastText |
|--------|----------|-------|----------|
| Training | Neural network (local context) | Matrix factorization (global stats) | Word2Vec + subwords |
| OOV words | Can't handle | Can't handle | Handles via subwords |
| Speed | Moderate | Fast (pre-computed) | Slower (subwords) |
| Morphology | Ignores | Ignores | Captures (un-happy) |
| Dimension | 100-300 | 100-300 | 100-300 |

### Static vs Contextual Embeddings

| Aspect | Static (Word2Vec/GloVe) | Contextual (BERT/GPT) |
|--------|------------------------|----------------------|
| Word meaning | One vector per word | Different vector per context |
| Polysemy | "bank" = same vector always | "river bank" != "bank account" |
| Training | Unsupervised, one-time | Pre-trained on massive corpora |
| Size | 100-300 dims | 768-1024+ dims |
| Use in 2024+ | Feature engineering, baselines | State-of-the-art for most tasks |

> **Critical Insight**: The revolution from Word2Vec to BERT is the shift from STATIC to CONTEXTUAL embeddings. "I went to the bank" gives "bank" a different vector depending on whether we're talking about rivers or finance. This is why BERT crushed Word2Vec on every benchmark — language is fundamentally about context.

---

## 4. Modern NLP: Transformers, BERT, GPT

### The Transformer Architecture (High-Level)

```
┌─────────────────────────────┐
│          OUTPUT              │
├─────────────────────────────┤
│     Feed-Forward Network    │
├─────────────────────────────┤
│     Multi-Head Attention    │  ← The key innovation!
├─────────────────────────────┤
│   Positional Encoding       │  ← Adds word order info
├─────────────────────────────┤
│     Input Embeddings        │
├─────────────────────────────┤
│          INPUT              │
└─────────────────────────────┘
```

### Self-Attention — The Core Idea

**Intuition**: When processing a word, look at ALL other words in the sentence and determine which ones are relevant.

```
"The animal didn't cross the street because it was too tired"

When processing "it":
  Attention focuses on → "animal" (high weight)
  Not on → "street" (low weight)
```

**Why it matters**: Captures long-range dependencies without the sequential bottleneck of RNNs.

### BERT vs GPT — Fundamental Difference

| Aspect | BERT | GPT |
|--------|------|-----|
| Architecture | Encoder only | Decoder only |
| Direction | Bidirectional (sees left AND right) | Left-to-right only (autoregressive) |
| Training task | Masked Language Model (fill in blanks) | Next token prediction |
| Best for | Understanding (classification, NER, QA) | Generation (text creation, chat) |
| Fine-tuning | Add task-specific head | Prompt engineering or fine-tune |
| Parameters | 110M (base) - 340M (large) | 117M (GPT-1) to 1.7T (GPT-4) |

### BERT Pre-training Tasks

1. **Masked Language Model (MLM)**: Randomly mask 15% of tokens, predict them
   ```
   Input:  "The [MASK] sat on the mat"
   Output: "The cat sat on the mat"
   ```

2. **Next Sentence Prediction (NSP)**: Is sentence B the actual next sentence after A?

### Transfer Learning in NLP

```
Phase 1: Pre-training (expensive, done once by big labs)
  └── Learn language understanding from billions of words

Phase 2: Fine-tuning (cheap, done by practitioners)
  └── Adapt to specific task with small labeled dataset
       ├── Text classification
       ├── Named Entity Recognition
       ├── Question Answering
       └── Sentiment Analysis
```

> **Critical Insight**: The transformer's self-attention mechanism processes all words simultaneously (O(n^2) with sequence length), unlike RNNs which process sequentially. This parallelism enabled training on massive datasets, leading to the "scaling laws" that drove GPT's success. The cost is quadratic memory with sequence length — which is why context windows matter.

---

## 5. Text Classification Approaches

### Evolution of Approaches

| Era | Method | Typical Accuracy | Effort |
|-----|--------|-----------------|--------|
| 2000s | BoW + Naive Bayes | Moderate | Low |
| 2010s | TF-IDF + SVM/LR | Good | Low |
| 2015s | Word2Vec + CNN/LSTM | Better | Medium |
| 2018+ | Fine-tuned BERT | Best | Medium |
| 2022+ | LLM zero/few-shot | Competitive | Very Low |

### When to Use What (Decision Framework)

```
Labeled data available?
├── < 100 examples → LLM zero/few-shot prompting
├── 100-1000 examples → Fine-tuned BERT or TF-IDF + LR
├── 1000-10000 examples → Fine-tuned BERT (best bet)
└── 10000+ examples → Any approach works; consider cost/speed

Latency requirements?
├── Real-time (< 50ms) → TF-IDF + LR (fastest)
├── Near real-time (< 500ms) → Distilled BERT
└── Batch processing → Full BERT/LLM

Interpretability needed?
├── Yes → TF-IDF + LR (coefficients are interpretable)
└── No → BERT/LLM for max accuracy
```

### Multi-Class vs Multi-Label

| Type | Definition | Example | Approach |
|------|-----------|---------|----------|
| Multi-class | Exactly one label per document | News category | Softmax output |
| Multi-label | Zero or more labels per document | Tags on a blog post | Sigmoid per label |
| Hierarchical | Labels have parent-child structure | Product taxonomy | Cascade classifiers |

> **Critical Insight**: In production, TF-IDF + Logistic Regression is still the champion when you need speed, interpretability, and easy maintenance. It's the first model you should try. Only move to BERT if (a) you need the accuracy boost AND (b) you can afford the inference cost. Many companies run TF-IDF in production because it's 100x faster and only slightly less accurate.

---

## 6. Sentiment Analysis

### Levels of Sentiment Analysis

| Level | Granularity | Example |
|-------|-------------|---------|
| Document-level | Overall sentiment of entire doc | "This movie review is positive" |
| Sentence-level | Sentiment per sentence | "The food was great but service was terrible" |
| Aspect-level | Sentiment per topic/aspect | "Battery: positive, Screen: negative" |

### Approaches Comparison

| Approach | How It Works | Pros | Cons |
|----------|-------------|------|------|
| Lexicon-based (VADER) | Dictionary of sentiment words | No training needed, interpretable | Misses context, sarcasm |
| ML (TF-IDF + classifier) | Supervised learning | Fast, decent accuracy | Needs labeled data |
| Deep Learning (BERT) | Fine-tuned on sentiment data | Best accuracy, captures context | Expensive, slower |
| LLM prompting | Ask GPT/Claude to analyze | Zero-shot, flexible | Expensive at scale, latency |

### Challenges in Sentiment Analysis

| Challenge | Example | Why It's Hard |
|-----------|---------|---------------|
| Sarcasm | "Oh great, another meeting" | Positive words, negative meaning |
| Negation | "Not bad at all" | Negative word, positive sentiment |
| Context | "This is sick!" | Slang positive vs literal negative |
| Comparison | "Better than terrible" | Relative, not absolute |
| Mixed | "Great food, awful service" | Multiple sentiments in one text |

> **Critical Insight**: Sentiment analysis sounds simple but is deceptively hard at scale. The gap between "works on examples" and "works in production" is massive. Real user text is messy — typos, slang, sarcasm, multilingual mixing. Always validate with domain-specific test sets, not just general benchmarks.

---

## 7. Topic Modeling

### LDA (Latent Dirichlet Allocation)

**Intuition**: Every document is a mixture of topics, and every topic is a mixture of words.

```
Document → [30% Sports, 50% Politics, 20% Business]

Topic "Sports" → {game: 0.05, player: 0.04, team: 0.03, score: 0.02, ...}
Topic "Politics" → {vote: 0.04, election: 0.03, policy: 0.03, ...}
```

### How LDA Works (Simplified)

1. Choose number of topics K
2. Randomly assign each word to a topic
3. Iteratively improve assignments based on:
   - How often this word appears in this topic (across all docs)
   - How much this document likes this topic (proportion)
4. Converge to stable topic-word and document-topic distributions

### LDA Parameters

| Parameter | What It Controls | Tuning |
|-----------|-----------------|--------|
| K (num_topics) | Number of topics | Coherence score, business needs |
| Alpha | Document-topic density | Low = fewer topics per doc |
| Beta (Eta) | Topic-word density | Low = fewer words per topic |

### Evaluating Topic Models

| Metric | What It Measures | Good Value |
|--------|-----------------|-----------|
| Coherence (C_v) | Semantic coherence of top words | Higher is better (> 0.5) |
| Perplexity | How surprised the model is | Lower is better |
| Human judgment | Do topics make sense? | The ultimate test |

### Alternatives to LDA

| Method | Key Difference | When to Use |
|--------|---------------|-------------|
| NMF (Non-negative Matrix Factorization) | Linear algebra based, faster | Shorter texts, speed needed |
| BERTopic | Uses BERT embeddings + clustering | Modern, captures semantics better |
| Top2Vec | Document/word embeddings jointly | Automatic topic count |
| LLM-based | Prompt LLM to identify themes | Small datasets, exploratory |

> **Critical Insight**: LDA is unsupervised — there's no "correct" answer. The number of topics is a choice, not a discovery. In interviews, demonstrate you understand this subjectivity: "I'd try multiple K values, evaluate with coherence scores, but ultimately validate with domain experts. Topic modeling is exploratory — it generates hypotheses, not conclusions."

---

## 8. Document Similarity & Cosine Similarity

### Cosine Similarity

**Formula:**
```
cosine_similarity(A, B) = (A · B) / (||A|| × ||B||)

Range: [-1, 1] for general vectors, [0, 1] for TF-IDF vectors
1 = identical direction (most similar)
0 = orthogonal (no similarity)
-1 = opposite (with embeddings)
```

**Why cosine over Euclidean?**
- Cosine measures DIRECTION (angle), not magnitude
- A long document and short document can be equally similar in topic
- Euclidean would say they're different just because one has more words

### Document Similarity Approaches

| Method | Vector Representation | Similarity Metric | Best For |
|--------|----------------------|-------------------|----------|
| TF-IDF + Cosine | Sparse high-dimensional | Cosine similarity | Keyword matching |
| Doc2Vec | Dense, fixed-size | Cosine/Euclidean | Semantic similarity |
| BERT embeddings | Dense, contextual | Cosine similarity | Deep semantic matching |
| Sentence-BERT | Optimized for similarity | Cosine similarity | Fast semantic search |
| BM25 | Term frequency based | BM25 score | Search/retrieval |

### Practical Application: Document Search

```
Query: "machine learning algorithms"

Step 1: Vectorize query → q_vector
Step 2: Vectorize all documents → doc_vectors
Step 3: Compute cosine_similarity(q_vector, each doc_vector)
Step 4: Rank by similarity score
Step 5: Return top-K results
```

### Scaling Similarity Search

| Scale | Approach | Speed |
|-------|----------|-------|
| < 10K docs | Brute force cosine | Fast enough |
| 10K - 1M docs | Approximate NN (FAISS, Annoy) | Very fast |
| > 1M docs | FAISS + quantization + sharding | Production-grade |

> **Critical Insight**: Cosine similarity with TF-IDF captures LEXICAL similarity (same words). BERT/Sentence-BERT captures SEMANTIC similarity (same meaning, different words). "Car" vs "automobile" scores low with TF-IDF but high with BERT. Choose based on whether you need keyword matching or meaning matching.

---

## 9. Traditional NLP vs LLMs Decision Framework

### When to Use Traditional NLP

| Scenario | Why Traditional Wins |
|----------|---------------------|
| High throughput (millions of docs/day) | 100-1000x faster, cheaper |
| Latency-critical (< 10ms) | TF-IDF + LR is near-instant |
| Simple, well-defined task | Don't need a sledgehammer |
| Full interpretability required | Can inspect feature weights |
| Offline/edge deployment | Small model size |
| Labeled data is abundant | Supervised ML matches or beats |
| Regulatory constraints | Deterministic, auditable |

### When to Use LLMs

| Scenario | Why LLMs Win |
|----------|-------------|
| Very few labeled examples | Zero/few-shot capability |
| Complex reasoning needed | Multi-step understanding |
| Generative tasks | Summarization, paraphrasing |
| Rapid prototyping | Works out-of-the-box |
| Multi-task | One model, many tasks |
| Nuanced understanding | Sarcasm, implicit meaning |
| Interactive/conversational | Chat, Q&A, dialogue |

### The Hybrid Approach (Often Best in Practice)

```
┌─────────────────────────────────────────────────────────┐
│ Stage 1: Fast Filter (Traditional)                       │
│   TF-IDF + simple rules → Filter 1M docs to 1000       │
├─────────────────────────────────────────────────────────┤
│ Stage 2: Deep Analysis (LLM)                            │
│   BERT/LLM → Re-rank, classify, extract from 1000     │
├─────────────────────────────────────────────────────────┤
│ Stage 3: Human Review (if needed)                       │
│   Humans verify edge cases flagged by model             │
└─────────────────────────────────────────────────────────┘
```

> **Critical Insight**: The interview-winning answer is NEVER "just use GPT for everything" or "traditional NLP is dead." It's always about trade-offs: "For this specific problem with these constraints (latency, cost, accuracy, data volume), here's why I'd choose X." Show you think about production realities, not just accuracy benchmarks.

---

## 10. Practical Applications

### Spam Detection System

```
Approach: TF-IDF + Logistic Regression (fast, interpretable)

Features:
- TF-IDF of email body
- Presence of known spam phrases
- Sender reputation score
- URL count, attachment types
- ALL CAPS ratio

Pipeline:
Raw email → Preprocess → Feature extract → Classify → Action
                                              │
                                    ├── Ham → Inbox
                                    └── Spam → Spam folder (with confidence)
```

### Customer Feedback Classification

```
Multi-label classification:
  Input: "The delivery was late and the product was damaged"
  Output: [shipping_issue, product_quality]

Approach:
  1. Baseline: TF-IDF + One-vs-Rest Logistic Regression
  2. If accuracy insufficient: Fine-tuned BERT
  3. For new categories: LLM few-shot classification
```

### Content Moderation System

```
Tiered approach for scale + accuracy:

Tier 1 (Instant): Keyword blocklist + regex patterns
  → Catches 60% of violations immediately

Tier 2 (Fast): TF-IDF + ML classifier
  → Catches 30% more with context

Tier 3 (Accurate): BERT/LLM for edge cases
  → Handles ambiguity, context-dependent violations

Tier 4 (Human): Review queue for uncertain cases
  → Continuous labeling for model improvement
```

> **Critical Insight**: Production NLP systems are almost always multi-stage pipelines, not single models. Fast/cheap models handle the easy cases; expensive models handle the hard cases. This is cost-efficient AND more accurate than using one approach for everything.

---

## 11. Interview Talking Points

### "Build a content moderation system for user-generated content"

> "I'd design a tiered system for cost efficiency at scale. First, a fast keyword/regex filter catches obvious violations. Second, a TF-IDF + classifier handles the bulk with low latency. Third, BERT-based models handle nuanced cases like sarcasm or coded language. Finally, uncertain cases go to human review. I'd implement active learning — human decisions feed back as training data. For the ML components, I'd start with multi-class (harassment, hate speech, spam, explicit content) and evaluate per-class precision/recall because false negatives for hate speech are worse than false positives for spam."

### "How would you classify customer feedback into categories?"

> "I'd start by understanding the label taxonomy — is it flat or hierarchical? How many categories? Then I'd examine the data: How much labeled data exists? Are categories balanced? My baseline would be TF-IDF + Logistic Regression because it's fast to iterate, interpretable, and often surprisingly strong. If that doesn't meet accuracy requirements, I'd fine-tune BERT. For the cold-start problem with new categories, I'd use LLM few-shot classification with just 5-10 examples. I'd always set up a feedback loop where low-confidence predictions get human review, which generates more training data."

### "How would you build a search system for internal documents?"

> "I'd use a two-stage retrieval approach. Stage 1: BM25 or TF-IDF for fast candidate retrieval — get the top 100 documents matching keywords. Stage 2: Re-rank with a cross-encoder (BERT-based) that scores query-document relevance more accurately but is too slow for the full corpus. I'd also generate dense embeddings (Sentence-BERT) for semantic search — finding documents that are conceptually relevant even without keyword overlap. The final ranking would blend keyword match score with semantic similarity. For the index, I'd use Elasticsearch for BM25 and FAISS for vector similarity."

---

## 12. Common Mistakes

| ❌ Wrong | ✅ Right |
|----------|----------|
| Heavy preprocessing before BERT | BERT has its own tokenizer; minimal preprocessing |
| Using Word2Vec for polysemous words | Use contextual embeddings (BERT) for ambiguous words |
| "Just use GPT" without considering cost | Evaluate cost/latency trade-offs; TF-IDF might suffice |
| Ignoring class imbalance in text classification | Stratified splits, class weights, or oversampling |
| Training on ALL data including test set (leakage) | Strict train/test split; fit vectorizer on train only |
| One-size-fits-all preprocessing | Adapt pipeline to task (NER needs case, sentiment needs punctuation) |
| Ignoring out-of-vocabulary words | Use subword tokenization or FastText |
| Evaluating with accuracy on imbalanced text data | Use F1, precision, recall per class |
| Treating text classification as solved | Domain shift, temporal drift degrade models over time |
| Not establishing a simple baseline first | Always start with TF-IDF + LR before complex models |

---

## 13. Rapid-Fire Q&A

**Q1: What's the difference between stemming and lemmatization?**
A: Stemming chops word endings with rules (fast, crude — "studies" → "studi"). Lemmatization uses dictionary lookup for proper base forms (slower, accurate — "studies" → "study"). Use lemmatization when meaning matters, stemming when speed matters.

**Q2: Why is TF-IDF better than raw word counts?**
A: Raw counts favor common words like "the," "is." TF-IDF downweights words that appear everywhere (low IDF) and upweights words unique to specific documents. It captures what makes each document DISTINCTIVE.

**Q3: How does Word2Vec capture semantics?**
A: By training on the distributional hypothesis: words appearing in similar contexts have similar meanings. "King" and "queen" appear near similar words ("royal," "throne"), so they get similar vectors. The training optimizes to predict context from words (or vice versa).

**Q4: What's the key innovation of Transformers?**
A: Self-attention — allowing each word to "look at" every other word simultaneously, weighted by relevance. This replaced sequential processing (RNNs) with parallel computation and enabled capturing long-range dependencies without information bottlenecks.

**Q5: When would you use BERT vs GPT?**
A: BERT for understanding/classification tasks (sentiment, NER, similarity) because it's bidirectional. GPT for generation tasks (summarization, chatbots, text creation) because it's autoregressive. BERT sees full context; GPT only sees what came before.

**Q6: What is cosine similarity and why use it for text?**
A: It measures the angle between two vectors, ranging 0-1 for positive vectors. For text, it captures topical similarity regardless of document length. A 100-word and 10,000-word document about the same topic score high because direction (topic) matters, not magnitude (length).

**Q7: How do you handle multiple languages?**
A: Multilingual BERT (mBERT) or XLM-RoBERTa for cross-lingual understanding. For traditional approaches: language detection first, then language-specific pipelines. For LLMs: most handle multiple languages natively but with varying quality.

**Q8: What's the cold start problem in text classification?**
A: When you have a new category with few/no labeled examples. Solutions: zero-shot classification (using label descriptions), few-shot learning with LLMs, transfer from related categories, or active learning to efficiently gather labels.

**Q9: How do you evaluate a text generation system?**
A: Automatic metrics (BLEU, ROUGE, BERTScore) give rough estimates. Human evaluation is gold standard — rate fluency, relevance, factuality. For specific tasks: task-completion rate, user satisfaction. No single metric captures "good text."

**Q10: What's the difference between NER and text classification?**
A: Text classification assigns labels to entire documents (spam/not-spam). NER identifies and classifies specific SPANS within text (person names, organizations, dates). Classification is document-level; NER is token-level.

---

## 14. ASCII Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                   NLP & TEXT ANALYTICS CHEAT SHEET                    ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  TEXT PREPROCESSING PIPELINE:                                        ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │ Raw Text → Lowercase → Remove noise → Tokenize →           │    ║
║  │ Stop words → Stem/Lemmatize → Vectorize                     │    ║
║  │                                                             │    ║
║  │ NOTE: Skip most steps for BERT/GPT (own tokenizer)          │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  VECTORIZATION EVOLUTION:                                            ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │ BoW       → Sparse, no semantics, no order                  │    ║
║  │ TF-IDF    → Sparse, weighted importance, no order           │    ║
║  │ Word2Vec  → Dense, semantic, static (one vec per word)      │    ║
║  │ BERT      → Dense, semantic, CONTEXTUAL (dynamic)           │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  MODEL SELECTION:                                                    ║
║                                                                      ║
║  Labeled data?                                                       ║
║  ├── None/few → LLM zero/few-shot                                   ║
║  ├── <1000 → Fine-tune BERT or TF-IDF + LR                         ║
║  └── >1000 → TF-IDF+LR baseline, then BERT if needed               ║
║                                                                      ║
║  Latency?                                                            ║
║  ├── <10ms → TF-IDF + LR (production workhorse)                    ║
║  ├── <100ms → DistilBERT                                            ║
║  └── >100ms → Full BERT/LLM                                         ║
║                                                                      ║
║  BERT vs GPT:                                                        ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │ BERT: Bidirectional → Understanding                         │    ║
║  │   Best for: classification, NER, similarity, QA             │    ║
║  │                                                             │    ║
║  │ GPT: Left-to-right → Generation                             │    ║
║  │   Best for: text creation, summarization, dialogue          │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  COSINE SIMILARITY:                                                  ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │ cos(A,B) = (A·B) / (||A|| × ||B||)                         │    ║
║  │                                                             │    ║
║  │ 1.0 = identical    0.5 = somewhat similar    0.0 = unrelated│    ║
║  │                                                             │    ║
║  │ Why cosine? → Measures direction (topic), ignores           │    ║
║  │               magnitude (length). Perfect for text.         │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  TOPIC MODELING (LDA):                                               ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │ Document = mixture of topics                                │    ║
║  │ Topic = mixture of words                                    │    ║
║  │                                                             │    ║
║  │ Doc1: [40% Tech, 30% Business, 30% Science]                │    ║
║  │ "Tech" topic: {AI: 0.08, data: 0.05, algorithm: 0.04}     │    ║
║  │                                                             │    ║
║  │ Choose K: coherence score + domain expertise                │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  PRODUCTION NLP (TIERED ARCHITECTURE):                               ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │ Tier 1: Rules/Keywords    → Fast, catches obvious cases     │    ║
║  │ Tier 2: TF-IDF + ML      → Handles bulk, low latency       │    ║
║  │ Tier 3: BERT/LLM         → Handles edge cases, nuance      │    ║
║  │ Tier 4: Human review     → Uncertain cases, generates data  │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  SENTIMENT ANALYSIS CHALLENGES:                                      ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │ Sarcasm:   "Oh great, another meeting" → Negative           │    ║
║  │ Negation:  "Not bad at all" → Positive                      │    ║
║  │ Mixed:     "Great food, awful service" → Aspect-level       │    ║
║  │ Context:   "This is sick!" → Depends on domain              │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 63 of 65*
