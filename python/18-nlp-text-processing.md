# 🎯 Topic 18: NLP & Text Processing

> **Data Science Interview — Natural Language Processing & Text Processing Deep Dive**  
> From raw text to machine-readable features — tokenization, TF-IDF, embeddings, topic modeling, and classification pipelines. The complete toolkit for turning unstructured text into signal, with interview scripts and production-ready code.

---

## Table of Contents

1. [Text Preprocessing Pipeline](#text-preprocessing-pipeline)
2. [Regular Expressions for Text Cleaning](#regular-expressions-for-text-cleaning)
3. [Bag of Words Model](#bag-of-words-model)
4. [TF-IDF — The Math and Intuition](#tf-idf--the-math-and-intuition)
5. [N-grams](#n-grams)
6. [Word Embeddings](#word-embeddings)
7. [Cosine Similarity](#cosine-similarity)
8. [Topic Modeling — LDA](#topic-modeling--lda)
9. [Text Classification Pipeline](#text-classification-pipeline)
10. [sklearn Vectorizers](#sklearn-vectorizers)
11. [Interview Talking Points & Scripts](#interview-talking-points--scripts)
12. [Common Interview Mistakes](#common-interview-mistakes)
13. [Rapid-Fire Q&A (Top 12)](#rapid-fire-qa-top-12)
14. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## Text Preprocessing Pipeline

Text data is messy. Before any modeling, you must clean and normalize it. Here is the standard pipeline:

### Step-by-Step Preprocessing

| Step | Purpose | Example |
|------|---------|---------|
| Lowercasing | Normalize case | `"NLP"` -> `"nlp"` |
| Punctuation removal | Remove noise | `"hello!"` -> `"hello"` |
| Tokenization | Split into units | `"I love NLP"` -> `["I", "love", "NLP"]` |
| Stopword removal | Remove common words | Remove "the", "is", "a" |
| Stemming | Reduce to root form (crude) | `"running"` -> `"run"` |
| Lemmatization | Reduce to dictionary form | `"better"` -> `"good"` |

> **Critical Insight:** Stemming is faster but cruder (Porter Stemmer chops suffixes blindly: "university" -> "univers"). Lemmatization uses vocabulary and POS tags to return real words ("better" -> "good"). For search engines, stemming is often enough. For text understanding tasks, lemmatization is preferred.

```python
import re
import nltk
from nltk.corpus import stopwords
from nltk.stem import PorterStemmer, WordNetLemmatizer
from nltk.tokenize import word_tokenize

text = "The runners were running quickly through 3 different cities!"

# Full pipeline in sequence
text_lower = text.lower()                                    # Step 1: Lowercase
text_clean = re.sub(r'[^a-z\s]', '', text_lower)           # Step 2: Remove noise
tokens = word_tokenize(text_clean)                           # Step 3: Tokenize
stop_words = set(stopwords.words('english'))
tokens_filtered = [t for t in tokens if t not in stop_words] # Step 4: Stopwords
# Result: ['runners', 'running', 'quickly', 'different', 'cities']

# Step 5a: Stemming (crude but fast)
stemmer = PorterStemmer()
stemmed = [stemmer.stem(t) for t in tokens_filtered]
# ['runner', 'run', 'quickli', 'differ', 'citi']

# Step 5b: Lemmatization (precise, real words)
lemmatizer = WordNetLemmatizer()
lemmatized = [lemmatizer.lemmatize(t) for t in tokens_filtered]
# ['runner', 'running', 'quickly', 'different', 'city']
```

### Stemming vs Lemmatization Comparison

| Feature | Stemming | Lemmatization |
|---------|----------|---------------|
| Speed | Fast | Slower |
| Output | May not be real word | Always a real word |
| Algorithm | Rule-based suffix stripping | Dictionary + POS lookup |
| Example | "studies" -> "studi" | "studies" -> "study" |
| Use case | Search, IR systems | Chatbots, text understanding |
| Library | `PorterStemmer`, `SnowballStemmer` | `WordNetLemmatizer`, spaCy |

---

## Regular Expressions for Text Cleaning

Regex is essential for text preprocessing in production pipelines.

```python
import re

text = "Call me at 555-123-4567 or email bob@mail.com! Visit https://example.com"

# Production pattern — compile once, use many times (significant speedup at scale)
URL_PATTERN = re.compile(r'https?://\S+')
EMAIL_PATTERN = re.compile(r'\S+@\S+')
PHONE_PATTERN = re.compile(r'\d{3}[-.\s]?\d{3}[-.\s]?\d{4}')

def clean_text(text):
    """Order matters: remove structured patterns BEFORE stripping special chars."""
    text = URL_PATTERN.sub('', text)
    text = EMAIL_PATTERN.sub('', text)
    text = PHONE_PATTERN.sub('', text)
    text = re.sub(r'[^a-z\s]', '', text.lower())  # Keep only letters + spaces
    return re.sub(r'\s+', ' ', text).strip()       # Collapse whitespace

print(clean_text(text))  # "call me at  or email  visit"
```

> **Critical Insight:** In production, the order of regex operations matters. Remove URLs and emails BEFORE removing special characters, otherwise you destroy the patterns you need to match. Always use `re.compile()` when applying patterns to millions of documents.

---

## Bag of Words Model

The simplest text representation: count word occurrences per document.

```python
from sklearn.feature_extraction.text import CountVectorizer

corpus = ["I love machine learning", "I love NLP and text processing",
          "machine learning and NLP are related"]

vectorizer = CountVectorizer()
X = vectorizer.fit_transform(corpus)

print(vectorizer.get_feature_names_out())
# ['and', 'are', 'learning', 'love', 'machine', 'nlp', 'processing', 'related', 'text']
print(X.toarray())  # Sparse matrix of word counts per document
```

> **Critical Insight:** Bag of Words completely ignores word order. "Dog bites man" and "Man bites dog" have identical BoW representations. This is a fundamental limitation — if word order matters for your task (sentiment, translation), you need n-grams or sequence models.

**BoW Limitations:** No word order, sparse high-dimensional vectors, no semantic similarity (synonyms are unrelated), common words dominate without weighting.

---

## TF-IDF — The Math and Intuition

**TF-IDF = Term Frequency x Inverse Document Frequency**

The intuition: a word is important to a document if it appears frequently in THAT document (high TF) but rarely across ALL documents (high IDF).

### The Math

```
TF(t, d) = count(t in d) / total_terms_in_d

IDF(t) = log(N / df(t))       where N = total docs, df(t) = docs containing t

TF-IDF(t, d) = TF(t, d) * IDF(t)
```

> **Critical Insight:** IDF penalizes words that appear everywhere (like "the", "is"). A word appearing in every document gets IDF = log(1) = 0, making its TF-IDF score zero regardless of frequency. This is why TF-IDF often eliminates the need for explicit stopword removal.

```python
from sklearn.feature_extraction.text import TfidfVectorizer

corpus = ["the cat sat on the mat", "the dog sat on the log", "cats and dogs are friends"]

tfidf = TfidfVectorizer()  # sklearn uses smoothed IDF: log((1+N)/(1+df)) + 1
X = tfidf.fit_transform(corpus)

# Inspect IDF values — 'the' (common) gets low IDF, 'friends' (rare) gets high IDF
for word, idx in sorted(tfidf.vocabulary_.items()):
    print(f"{word:12s} IDF={tfidf.idf_[idx]:.3f}")
```

### TF-IDF Variants

| Variant | TF Component | IDF Component | Used In |
|---------|-------------|---------------|---------|
| Raw count | `count(t,d)` | `log(N/df)` | Basic textbooks |
| sklearn default | `count` | `log((1+N)/(1+df)) + 1` | scikit-learn |
| Sublinear TF | `1 + log(count)` | `log((1+N)/(1+df)) + 1` | `sublinear_tf=True` |

---

## N-grams

N-grams capture local word order by considering sequences of N consecutive tokens.

| Type | N | Example from "I love machine learning" |
|------|---|----------------------------------------|
| Unigram | 1 | "I", "love", "machine", "learning" |
| Bigram | 2 | "I love", "love machine", "machine learning" |
| Trigram | 3 | "I love machine", "love machine learning" |

```python
from sklearn.feature_extraction.text import CountVectorizer

# Unigrams + Bigrams (most common in practice)
vec = CountVectorizer(ngram_range=(1, 2))
X = vec.fit_transform(["I love machine learning"])
print(vec.get_feature_names_out())
# ['love', 'love machine', 'learning', 'machine', 'machine learning']
```

> **Critical Insight:** Adding bigrams/trigrams dramatically increases feature dimensionality but captures phrases like "not good" (vs "good" alone losing negation). The sweet spot for most classification tasks is `ngram_range=(1, 2)` with TF-IDF weighting and a `max_features` cap.

---

## Word Embeddings

Embeddings represent words as dense, low-dimensional vectors where semantic similarity is captured by geometric proximity.

### Comparison: BoW vs TF-IDF vs Embeddings

| Feature | Bag of Words | TF-IDF | Word Embeddings |
|---------|-------------|--------|-----------------|
| Vector type | Sparse, high-dim | Sparse, high-dim | Dense, low-dim (100-300d) |
| Semantics | None | None | Captured |
| Word order | Ignored | Ignored | Depends on method |
| Vocabulary | Fixed to corpus | Fixed to corpus | Pretrained or trainable |
| "king - man + woman" | Impossible | Impossible | = "queen" |
| Training data needed | Just your corpus | Just your corpus | Massive external corpora |
| Interpretability | High (word counts) | High (weighted counts) | Low (latent dimensions) |
| Best for | Simple classifiers | Search, IR, baselines | Deep learning, transfer |

### Key Embedding Methods

| Method | Year | Mechanism | Context-aware? |
| ------ | ---- | --------- | -------------- |
| Word2Vec (CBOW) | 2013 | Predict center word from context | No (static) |
| Word2Vec (Skip-gram) | 2013 | Predict context from center word | No (static) |
| GloVe | 2014 | Global co-occurrence matrix factorization | No (static) |
| BERT | 2018 | Transformer, bidirectional context | Yes (contextual) |

```python
# Conceptual Word2Vec usage (gensim)
from gensim.models import Word2Vec

sentences = [["I", "love", "NLP"], ["NLP", "is", "fun"]]
model = Word2Vec(sentences, vector_size=100, window=5, min_count=1)
vector = model.wv["NLP"]           # 100-dimensional dense vector
model.wv.most_similar("NLP")       # Semantically similar words
```

> **Critical Insight:** For a Data Scientist in industry, the decision framework is: start with TF-IDF as your baseline (fast, interpretable, often surprisingly good), then try pretrained embeddings (GloVe, fastText) if you need semantic understanding, and only move to contextual embeddings (BERT) if you have enough data and compute to justify the complexity.

---

## Cosine Similarity

Measures the angle between two vectors — standard metric for comparing text representations.

```
cosine_similarity(A, B) = (A . B) / (||A|| * ||B||)
```

Range: -1 (opposite) to 1 (identical direction). For TF-IDF vectors, always 0 to 1.

```python
from sklearn.metrics.pairwise import cosine_similarity
from sklearn.feature_extraction.text import TfidfVectorizer

docs = ["machine learning is great",
        "deep learning is a subset of machine learning",
        "I love cooking pasta"]
tfidf = TfidfVectorizer()
X = tfidf.fit_transform(docs)

# Pairwise cosine similarity
sim_matrix = cosine_similarity(X)
print(sim_matrix)
# doc0 vs doc1: high similarity (both about ML)
# doc0 vs doc2: low similarity (different topics)
```

> **Critical Insight:** Cosine similarity is preferred over Euclidean distance for text because it is invariant to document length. A 10-word document and a 1000-word document about the same topic will have high cosine similarity but very different Euclidean distances simply due to magnitude differences.

### Why Cosine Over Euclidean for Text?

| Metric | Length-invariant? | Range | Best For |
|--------|------------------|-------|----------|
| Cosine similarity | Yes | [-1, 1] | Text similarity, recommendations |
| Euclidean distance | No | [0, inf) | Dense, normalized vectors |
| Jaccard similarity | Yes (set-based) | [0, 1] | Binary presence/absence |

---

## Topic Modeling — LDA

**Latent Dirichlet Allocation** discovers hidden topics in a document collection. It is an unsupervised method.

### Intuition

- Each **document** is a mixture of topics (e.g., 70% sports, 30% politics)
- Each **topic** is a mixture of words (e.g., sports topic: "game" 0.05, "team" 0.04, ...)
- LDA works backward from observed words to infer these latent structures

```python
from sklearn.decomposition import LatentDirichletAllocation
from sklearn.feature_extraction.text import CountVectorizer

docs = [
    "The team won the championship game",
    "The player scored three goals in the match",
    "The senator proposed a new tax bill",
    "Congress voted on healthcare reform",
    "The goalkeeper made an incredible save",
    "The president signed the executive order"
]

# LDA works with raw counts, NOT TF-IDF
count_vec = CountVectorizer(stop_words='english')
X = count_vec.fit_transform(docs)

# Fit LDA with 2 topics
lda = LatentDirichletAllocation(n_components=2, random_state=42)
lda.fit(X)

# Print top words per topic
feature_names = count_vec.get_feature_names_out()
for topic_idx, topic in enumerate(lda.components_):
    top_words = [feature_names[i] for i in topic.argsort()[:-6:-1]]
    print(f"Topic {topic_idx}: {', '.join(top_words)}")
# Topic 0: game, team, scored, goals, player  (sports)
# Topic 1: new, tax, congress, voted, president  (politics)
```

> **Critical Insight:** LDA requires you to specify the number of topics (k) beforehand. Use coherence score or perplexity to choose k. Also, LDA uses raw word counts — do NOT feed it TF-IDF vectors. The generative model assumes integer word counts.

---

## Text Classification Pipeline

End-to-end pipeline for building a text classifier (e.g., spam detection).

```python
from sklearn.model_selection import train_test_split
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.pipeline import Pipeline
from sklearn.metrics import classification_report

# Assume X_text (list of strings) and y (labels) are loaded
X_train, X_test, y_train, y_test = train_test_split(X_text, y, test_size=0.3, random_state=42)

# Build pipeline: TF-IDF + Logistic Regression
pipeline = Pipeline([
    ('tfidf', TfidfVectorizer(ngram_range=(1, 2), max_features=5000, stop_words='english')),
    ('clf', LogisticRegression(max_iter=1000))
])

pipeline.fit(X_train, y_train)
print(classification_report(y_test, pipeline.predict(X_test)))
```

> **Critical Insight:** `sklearn.pipeline.Pipeline` is not optional in interviews — it shows you understand proper ML engineering. Pipelines prevent data leakage (TF-IDF vocabulary is fit only on training data), make cross-validation correct, and simplify deployment. Always mention pipelines when discussing text classification.

---

## sklearn Vectorizers

### Key Parameters

| Parameter | CountVectorizer | TfidfVectorizer | Purpose |
|-----------|-----------------|-----------------|---------|
| `max_features` | Yes | Yes | Limit vocabulary size |
| `ngram_range` | Yes | Yes | Include n-grams |
| `stop_words` | Yes | Yes | Remove stopwords |
| `min_df` | Yes | Yes | Min document frequency |
| `max_df` | Yes | Yes | Max document frequency (remove too-common) |
| `sublinear_tf` | No | Yes | Apply log(1+tf) scaling |
| `norm` | No | Yes | L1 or L2 normalization |
| `binary` | Yes | Yes | Binary presence vs counts |

```python
# Production-ready vectorizer configuration
tfidf = TfidfVectorizer(
    max_features=10000,        # Cap vocabulary
    ngram_range=(1, 2),        # Unigrams + bigrams
    min_df=5,                  # Ignore very rare terms
    max_df=0.95,               # Ignore terms in >95% of docs
    sublinear_tf=True,         # Dampens high-frequency term impact
    stop_words='english'
)
```

> **Critical Insight:** `min_df` and `max_df` are your best tools for controlling vocabulary size in production. `min_df=5` removes typos and very rare terms. `max_df=0.95` acts like automatic stopword removal. Together they can reduce a 100K vocabulary to 10K with minimal information loss.

---

## Interview Talking Points & Scripts

### When asked "How would you build a spam classifier?"

> *"I would approach this as a binary text classification problem. First, I would preprocess the text — lowercase, remove URLs and special characters, tokenize. Then I would represent the text using TF-IDF with unigrams and bigrams, capping features at maybe 10K with min_df=5 to remove noise. For the model, I would start with Logistic Regression — it is fast, interpretable, and works surprisingly well for text. I would use sklearn Pipeline to bundle vectorization and modeling together, preventing data leakage during cross-validation. Evaluation would focus on precision and recall separately because the costs are asymmetric — false positives (blocking real email) are worse than false negatives (letting some spam through). If the baseline is not sufficient, I would try an SVM with linear kernel, or move to a pretrained transformer model. I would also engineer domain-specific features: presence of urgency words, excessive caps, suspicious URLs, or known spam phrases."*

### When asked "TF-IDF vs Embeddings — when to use which?"

> *"I use TF-IDF as my default starting point for any text classification or search task. It is fast, interpretable, requires no external data, and gives you a strong baseline. TF-IDF excels at keyword-matching tasks like document retrieval, spam detection, or topic classification where the exact words matter. I switch to embeddings when I need semantic understanding — when synonyms matter, when the vocabulary is diverse, or when I have limited labeled data and want to leverage transfer learning. For example, if 'automobile' and 'car' should be treated as similar, embeddings handle this naturally while TF-IDF treats them as completely unrelated. The decision also depends on data size: with thousands of documents, TF-IDF plus a linear model often matches or beats embeddings. With smaller data or more nuanced tasks, pretrained embeddings provide a crucial boost."*

### When asked "What is cosine similarity and why use it for text?"

> *"Cosine similarity measures the angle between two vectors, ignoring their magnitude. For text, this is crucial because we want to compare documents by topic, not length. A 100-word abstract and a 5000-word paper on the same topic should be similar, and cosine achieves this by normalizing out length. The output ranges from 0 to 1 for TF-IDF vectors — 1 meaning identical, 0 meaning unrelated."*

---

## Common Interview Mistakes

| ❌ Mistake | ✅ Correction |
|-----------|--------------|
| Feeding TF-IDF vectors to LDA | LDA needs raw counts — use CountVectorizer |
| Fitting TF-IDF on test data | Fit on train only, transform test — use Pipeline |
| Removing stopwords AND using TF-IDF | TF-IDF already downweights common words; doing both is redundant (though not harmful) |
| Saying "Word2Vec captures word order" | Word2Vec uses local context windows but the final vectors are static per word — no sentence-level order |
| Using Euclidean distance for sparse TF-IDF | Cosine similarity is standard — Euclidean is dominated by document length |
| Not mentioning n-grams for negation | "not good" is lost with unigrams; bigrams capture it |
| Confusing stemming with lemmatization | Stemming = heuristic chopping; lemmatization = dictionary lookup returning real words |
| Ignoring class imbalance in text classification | Spam/fraud are rare — mention stratified splits, class weights, or SMOTE |
| Saying embeddings are always better than TF-IDF | TF-IDF often wins on keyword-heavy tasks with enough data; always baseline first |
| Not knowing what IDF does when df equals N | IDF = log(1) = 0, so the term gets zero weight — it appears everywhere and is uninformative |

---

## Rapid-Fire Q&A (Top 12)

**Q1: What does TF-IDF of zero mean?**  
The word appears in every document (IDF=0) or does not appear in this document (TF=0). Either way, it carries no discriminative information for that document.

**Q2: Why is text data represented as sparse matrices?**  
Most documents use only a tiny fraction of the total vocabulary. A 50K-vocabulary TF-IDF matrix for a single document has >99% zeros. Sparse format stores only non-zero values, saving orders of magnitude in memory.

**Q3: CountVectorizer vs TfidfVectorizer — when to pick which?**  
CountVectorizer for LDA topic modeling and when downstream model handles weighting (e.g., Naive Bayes with built-in priors). TfidfVectorizer for everything else — classifiers, similarity search, clustering.

**Q4: What is sublinear TF and why use it?**  
Replaces raw count with `1 + log(count)`. Prevents a word appearing 100 times from being 100x more important than one appearing once. Diminishing returns on repeated occurrence.

**Q5: How do you handle out-of-vocabulary words at inference time?**  
TF-IDF/BoW: OOV words are simply ignored (not in the fitted vocabulary). Embeddings: use subword models (fastText) or UNK token. This is a real production concern.

**Q6: Word2Vec CBOW vs Skip-gram — when to use each?**  
CBOW is faster, works better for frequent words. Skip-gram is slower but better for rare words and small datasets. Skip-gram is more commonly used in practice.

**Q7: Can you use TF-IDF for document clustering?**  
Yes — TF-IDF + K-Means with cosine distance is a classic baseline for document clustering. Use `normalize()` or set `norm='l2'` so Euclidean distance in normalized space equals cosine distance.

**Q8: How do you choose the number of topics in LDA?**  
Use coherence score (higher is better) or perplexity (lower is better, but can overfit). Plot both against number of topics and look for the elbow. Domain expertise should validate the output.

**Q9: What is the curse of dimensionality for text?**  
A 50K vocabulary means 50K features per document. Most are zero. Models overfit easily unless you reduce dimensionality (max_features, SVD/LSA, or embeddings).

**Q10: How does BERT differ from Word2Vec?**  
Word2Vec produces one static vector per word regardless of context. BERT produces different vectors for the same word depending on surrounding words. "Bank" near "river" vs "bank" near "money" get different BERT embeddings but identical Word2Vec embeddings.

**Q11: What is the relationship between BoW and TF-IDF?**  
BoW (CountVectorizer) gives raw counts. TF-IDF (TfidfVectorizer) reweights those counts by IDF and normalizes. TF-IDF is literally a weighted and normalized version of BoW.

**Q12: How do you handle multilingual text?**  
Options: train separate models per language, use multilingual embeddings (mBERT, XLM-R), or translate to a single language first. For simple tasks like classification, language-specific TF-IDF often works fine.

---

## Summary Cheat Sheet

```
+==================================================================+
|              NLP & TEXT PROCESSING CHEAT SHEET                     |
+==================================================================+
|                                                                    |
|  PREPROCESSING:  lowercase -> remove noise -> tokenize ->         |
|                  stopwords -> stem/lemmatize                       |
|                                                                    |
|  REPRESENTATIONS:                                                  |
|    BoW:       Sparse, counts, no semantics, fast                  |
|    TF-IDF:    Sparse, weighted counts, downweights common words   |
|    Embeddings: Dense, semantic, pretrained, transfer learning      |
|                                                                    |
|  TF-IDF MATH:  TF(t,d) * log(N / df(t))                          |
|    - High TF + Low DF = important discriminative term             |
|    - Word in every doc => IDF=0 => TF-IDF=0                      |
|                                                                    |
|  SIMILARITY:   Cosine (angle-based, length-invariant)             |
|                Range [0,1] for TF-IDF vectors                     |
|                                                                    |
|  TOPIC MODEL:  LDA — docs=mix of topics, topics=mix of words     |
|                Uses CountVectorizer (NOT TF-IDF)                  |
|                                                                    |
|  PIPELINE:     TfidfVectorizer(ngram_range=(1,2)) + LogReg       |
|                Always use sklearn Pipeline to prevent leakage     |
|                                                                    |
|  SKLEARN:      CountVectorizer -> raw counts (for LDA, NB)       |
|                TfidfVectorizer -> weighted (for classif, search)  |
|                Key params: max_features, min_df, max_df           |
|                                                                    |
|  DECISION:     Start TF-IDF -> try embeddings if semantics       |
|                needed -> BERT only if compute allows              |
|                                                                    |
+==================================================================+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 18 of 25*
