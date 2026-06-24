# 🎯 Topic 47: GenAI and LLM Applications in Analytics

> *"Generative AI is not a replacement for data science — it's an accelerant. The data scientists who thrive in 2025 aren't those who fear LLMs but those who know exactly when to deploy them, when to avoid them, and how to evaluate their outputs with the same rigor they'd apply to any model."*

---

## 📑 Table of Contents

1. [Embeddings for Classification, Clustering, and Similarity](#embeddings-for-classification-clustering-and-similarity)
2. [RAG: Retrieval-Augmented Generation](#rag-retrieval-augmented-generation)
3. [Prompt Engineering for Data Tasks](#prompt-engineering-for-data-tasks)
4. [LLM Evaluation Frameworks](#llm-evaluation-frameworks)
5. [Fine-Tuning vs Few-Shot vs RAG Decision Framework](#fine-tuning-vs-few-shot-vs-rag-decision-framework)
6. [Vector Databases and Embedding Infrastructure](#vector-databases-and-embedding-infrastructure)
7. [Practical Applications in Analytics](#practical-applications-in-analytics)
8. [Cost, Latency, and Operational Tradeoffs](#cost-latency-and-operational-tradeoffs)
9. [When NOT to Use LLMs](#when-not-to-use-llms)
10. [Interview Talking Points](#interview-talking-points)
11. [Common Mistakes](#common-mistakes)
12. [Rapid-Fire Q&A](#rapid-fire-qa)
13. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Embeddings for Classification, Clustering, and Similarity

### What Are Embeddings?

Embeddings are dense vector representations of text (or other data) that capture semantic meaning in a numerical format suitable for mathematical operations.

```
TEXT:                           EMBEDDING (simplified):
"customer complained about      → [0.82, -0.15, 0.34, ..., 0.91]
 late delivery"                    (1536 dimensions for OpenAI ada-002)
                                   (768 dimensions for sentence-transformers)

"shipment arrived too slow"     → [0.79, -0.12, 0.38, ..., 0.88]
                                   ↑ Similar vector! High cosine similarity

"loved the product quality"     → [-0.21, 0.67, -0.45, ..., 0.12]
                                   ↑ Very different vector — different topic
```

### Using Embeddings for Classification

```python
from sentence_transformers import SentenceTransformer
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split
import numpy as np

# Step 1: Generate embeddings
model = SentenceTransformer('all-MiniLM-L6-v2')  # Fast, good quality
texts = df['support_ticket_text'].tolist()
embeddings = model.encode(texts, show_progress_bar=True)

# Step 2: Use embeddings as features for classification
X_train, X_test, y_train, y_test = train_test_split(
    embeddings, df['category'], test_size=0.2, random_state=42
)

# Step 3: Simple classifier on top
clf = LogisticRegression(max_iter=1000)
clf.fit(X_train, y_train)
print(f"Accuracy: {clf.score(X_test, y_test):.3f}")

# WHY this works:
# - Embeddings capture semantic meaning without manual feature engineering
# - No need for TF-IDF, n-grams, or handcrafted features
# - Often outperforms traditional NLP with minimal labeled data
```

### Clustering with Embeddings

```python
from sklearn.cluster import KMeans
from sklearn.metrics import silhouette_score
import umap

# Cluster customer feedback into themes
embeddings = model.encode(feedback_texts)

# Reduce dimensionality for visualization
reducer = umap.UMAP(n_components=2, random_state=42)
embeddings_2d = reducer.fit_transform(embeddings)

# Cluster
kmeans = KMeans(n_clusters=8, random_state=42)
clusters = kmeans.fit_predict(embeddings)

# Evaluate cluster quality
print(f"Silhouette Score: {silhouette_score(embeddings, clusters):.3f}")

# Interpret clusters: find representative texts per cluster
for cluster_id in range(8):
    mask = clusters == cluster_id
    centroid = embeddings[mask].mean(axis=0)
    distances = np.linalg.norm(embeddings[mask] - centroid, axis=1)
    closest_idx = np.where(mask)[0][distances.argsort()[:3]]
    print(f"\nCluster {cluster_id} ({mask.sum()} items):")
    for idx in closest_idx:
        print(f"  - {feedback_texts[idx][:100]}")
```

### Semantic Similarity Search

```python
from scipy.spatial.distance import cosine

# Find similar support tickets to a new one
new_ticket = "my order hasn't arrived and it's been 2 weeks"
new_embedding = model.encode([new_ticket])[0]

# Compute cosine similarity with all historical tickets
similarities = [1 - cosine(new_embedding, emb) for emb in historical_embeddings]
top_similar = np.argsort(similarities)[-5:][::-1]

# Use for: duplicate detection, routing, suggested solutions
```

> **Critical Insight:** Embeddings are the "feature engineering for free" of the LLM era. Before reaching for a full LLM to classify text, try: embed text with a sentence transformer, then train a logistic regression on top. This gives you 80-90% of LLM performance at 1% of the cost, with full interpretability of the classifier layer.

---

## RAG: Retrieval-Augmented Generation

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    RAG PIPELINE                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  USER QUERY: "What was our Q3 churn rate for enterprise?"  │
│                                                             │
│  Step 1: EMBED the query                                   │
│     query → [0.45, -0.22, 0.78, ...]                       │
│                                                             │
│  Step 2: RETRIEVE relevant documents from vector DB        │
│     ┌──────────────────────────────────────────────┐       │
│     │  Vector DB (Pinecone/Weaviate/pgvector)      │       │
│     │  Search: top-k most similar chunks           │       │
│     │  Result: 3-5 relevant document chunks        │       │
│     └──────────────────────────────────────────────┘       │
│                                                             │
│  Step 3: AUGMENT the prompt with retrieved context         │
│     "Given these documents: [chunk1, chunk2, chunk3]       │
│      Answer: What was our Q3 churn rate for enterprise?"   │
│                                                             │
│  Step 4: GENERATE answer with LLM                          │
│     LLM reads context + question → grounded answer         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Implementation Skeleton

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.vectorstores import Pinecone
from langchain.embeddings import OpenAIEmbeddings
from langchain.chains import RetrievalQA
from langchain.llms import ChatOpenAI

# Step 1: Chunk documents
splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,        # Characters per chunk
    chunk_overlap=50,      # Overlap for context continuity
    separators=["\n\n", "\n", ". ", " "]
)
chunks = splitter.split_documents(internal_docs)

# Step 2: Embed and store in vector DB
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Pinecone.from_documents(chunks, embeddings, index_name="analytics-kb")

# Step 3: Build retrieval chain
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})
qa_chain = RetrievalQA.from_chain_type(
    llm=ChatOpenAI(model="gpt-4o-mini", temperature=0),
    retriever=retriever,
    return_source_documents=True  # For citation tracking
)

# Step 4: Query
result = qa_chain({"query": "What was Q3 enterprise churn?"})
print(result["result"])
print("Sources:", [doc.metadata["source"] for doc in result["source_documents"]])
```

### Why RAG Over Fine-Tuning for Knowledge

```
FINE-TUNING for knowledge:
  ❌ Expensive to retrain when docs change
  ❌ Can hallucinate "learned" facts
  ❌ No source attribution
  ❌ Stale the moment training ends

RAG for knowledge:
  ✅ Update docs anytime (no retraining)
  ✅ Grounded in actual source text
  ✅ Full source attribution (citations)
  ✅ Always up-to-date with latest docs
  ✅ Easier to debug (check retrieved chunks)
```

> **Critical Insight:** RAG is the go-to pattern for enterprise data science Q&A systems — internal wikis, report archives, Slack history. The key quality lever is not the LLM model but the RETRIEVAL quality: chunk size, embedding model choice, and metadata filtering. A mediocre LLM with excellent retrieval beats a frontier LLM with poor retrieval every time.

---

## Prompt Engineering for Data Tasks

### Text-to-SQL

```python
SYSTEM_PROMPT = """You are a SQL expert. Given the database schema below, 
write a SQL query to answer the user's question.

SCHEMA:
- orders(order_id, user_id, amount, created_at, status)
- users(user_id, name, segment, signup_date, country)
- products(product_id, name, category, price)
- order_items(order_id, product_id, quantity)

RULES:
1. Use only the tables and columns listed above
2. Always use explicit JOINs (never implicit comma joins)
3. Use date functions compatible with PostgreSQL
4. Include comments explaining complex logic
5. Return only the SQL query, no explanation
"""

USER_PROMPT = "What's the average order value by customer segment for Q3 2024?"

# Expected output:
# SELECT u.segment, AVG(o.amount) as avg_order_value
# FROM orders o
# JOIN users u ON o.user_id = u.user_id
# WHERE o.created_at >= '2024-07-01' AND o.created_at < '2024-10-01'
#   AND o.status = 'completed'
# GROUP BY u.segment
# ORDER BY avg_order_value DESC
```

### Data Summarization Prompt

```python
SUMMARIZATION_PROMPT = """Analyze this data summary and provide a 3-paragraph 
executive briefing:

DATA:
- Monthly revenue: $4.2M (down 8% MoM, up 12% YoY)  
- New users: 45K (down 15% MoM)
- Churn rate: 4.8% (up from 3.2% last month)
- NPS: 42 (down from 51)

INSTRUCTIONS:
1. Lead with the most critical finding and its business impact
2. Identify the likely root cause connecting these metrics
3. Recommend 1-2 specific actions with expected impact
4. Use confident language where data supports it, hedged language otherwise
"""
```

### Structured Extraction Prompt

```python
EXTRACTION_PROMPT = """Extract structured data from this customer review.
Return ONLY valid JSON matching this schema:

{
  "sentiment": "positive|negative|neutral|mixed",
  "topics": ["list of mentioned topics"],
  "urgency": "low|medium|high|critical",
  "product_mentioned": "string or null",
  "action_required": "boolean",
  "summary": "one sentence summary"
}

REVIEW: "{review_text}"
"""
```

### Prompt Engineering Best Practices for Data Tasks

```
1. SCHEMA GROUNDING: Always include actual table/column names
2. FEW-SHOT EXAMPLES: Show 2-3 input/output pairs for complex formats
3. OUTPUT FORMAT: Specify exact format (JSON, SQL, markdown)
4. CONSTRAINTS: State what NOT to do (no hallucinated columns, no CTEs if not needed)
5. CHAIN-OF-THOUGHT: For complex logic, ask LLM to "think step by step"
6. VALIDATION HOOK: Include "If you're unsure, say 'UNCERTAIN: [reason]'"
```

> **Critical Insight:** Text-to-SQL is one of the highest-ROI LLM applications for data teams — it democratizes data access. But it needs guardrails: validate generated SQL against schema, run EXPLAIN before execution, and NEVER auto-execute write queries (SELECT only). The biggest failure mode is hallucinated column names that produce valid-looking but incorrect queries.

---

## LLM Evaluation Frameworks

### Evaluation Taxonomy

```
┌─────────────────────────────────────────────────────────────┐
│              LLM EVALUATION METHODS                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  AUTOMATED METRICS (fast, cheap, limited):                 │
│  ├── BLEU: n-gram overlap with reference (translation)     │
│  ├── ROUGE: recall of n-grams from reference (summary)     │
│  ├── BERTScore: semantic similarity via embeddings          │
│  ├── Exact Match: for structured outputs (SQL, JSON)       │
│  └── Execution Accuracy: does generated SQL return right?  │
│                                                             │
│  LLM-AS-JUDGE (medium cost, scalable):                     │
│  ├── Pairwise comparison: "Which response is better?"      │
│  ├── Rubric scoring: "Rate 1-5 on accuracy, relevance"    │
│  ├── Reference-based: "Does this match the gold answer?"   │
│  └── Self-consistency: "Rate your confidence 1-10"         │
│                                                             │
│  HUMAN EVALUATION (expensive, gold standard):              │
│  ├── Expert rating: domain specialists score outputs       │
│  ├── A/B preference: which of two outputs is better?       │
│  ├── Error taxonomy: classify failure modes                │
│  └── Task success: did the output achieve the goal?        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Metric Deep Dive

```python
# BLEU Score (bilingual evaluation understudy)
from nltk.translate.bleu_score import sentence_bleu
reference = [["the", "cat", "sat", "on", "the", "mat"]]
candidate = ["the", "cat", "is", "on", "the", "mat"]
score = sentence_bleu(reference, candidate)  # 0-1, higher = more overlap
# Best for: translation, paraphrase detection
# Weakness: doesn't capture semantic similarity ("couch" vs "sofa" scores 0)

# ROUGE Score (recall-oriented understudy for gisting evaluation)
from rouge_score import rouge_scorer
scorer = rouge_scorer.RougeScorer(['rouge1', 'rouge2', 'rougeL'])
scores = scorer.score(reference_summary, generated_summary)
# ROUGE-1: unigram overlap, ROUGE-2: bigram overlap, ROUGE-L: longest common subsequence
# Best for: summarization quality
# Weakness: doesn't penalize factual errors if words overlap

# BERTScore (semantic similarity via contextual embeddings)
from bert_score import score
P, R, F1 = score(candidates, references, lang="en")
# Best for: when paraphrasing is acceptable (captures meaning, not just words)

# Execution Accuracy (for text-to-SQL)
def eval_sql(generated_sql, gold_sql, db_connection):
    gen_result = db_connection.execute(generated_sql).fetchall()
    gold_result = db_connection.execute(gold_sql).fetchall()
    return set(gen_result) == set(gold_result)  # Order-agnostic comparison
```

### LLM-as-Judge Pattern

```python
JUDGE_PROMPT = """You are evaluating an AI-generated data analysis summary.

CRITERIA (score each 1-5):
1. ACCURACY: Are all stated numbers and trends correct given the data?
2. COMPLETENESS: Are all key findings mentioned?
3. ACTIONABILITY: Does it suggest clear next steps?
4. CONCISENESS: Is it appropriately brief without losing meaning?

DATA PROVIDED TO THE AI:
{original_data}

AI'S RESPONSE:
{ai_response}

GOLD STANDARD RESPONSE (for reference):
{gold_response}

Score each criterion 1-5 with one-sentence justification.
Output as JSON: {"accuracy": {"score": X, "reason": "..."}, ...}
"""

# KEY: Use a stronger model as judge (GPT-4 judging GPT-3.5 outputs)
# BIAS MITIGATION: Randomize order in pairwise comparisons
# CALIBRATION: Include known-good and known-bad examples
```

> **Critical Insight:** For data science LLM applications, execution-based evaluation is the gold standard. Don't just check if the generated SQL looks right — run it and compare results. For summarization, don't just check ROUGE — verify factual claims against source data. Automated metrics are screening tools, not final arbiters.

---

## Fine-Tuning vs Few-Shot vs RAG Decision Framework

### Decision Matrix

| Criterion | Few-Shot (Prompt) | RAG | Fine-Tuning |
|-----------|------------------|-----|-------------|
| **Setup time** | Minutes | Hours-days | Days-weeks |
| **Cost per query** | High (long prompt) | Medium (retrieval + shorter prompt) | Low (shorter prompt, custom model) |
| **Knowledge freshness** | N/A (no knowledge) | Real-time (update docs) | Stale (retrain needed) |
| **Task adaptation** | Good for formatting | Good for knowledge | Best for style/behavior |
| **Data requirement** | 0 examples needed | Documents to index | 100-10K labeled examples |
| **Hallucination risk** | High | Low (grounded) | Medium |
| **Best for** | Format, reasoning | Knowledge Q&A | Specialized behavior |

### Decision Flowchart

```
START: What do you need the LLM to do?

Q1: Does it need proprietary/internal knowledge?
  YES → Is the knowledge base > 100K tokens?
    YES → RAG (can't fit in context window)
    NO  → Few-shot with full context (simpler)
  NO ↓

Q2: Do you need a specific output format/style?
  YES → Is few-shot (3-5 examples in prompt) sufficient?
    YES → Few-shot prompting (cheapest, fastest)
    NO  → Fine-tune for consistent style
  NO ↓

Q3: Do you need domain-specific reasoning?
  YES → Do you have 500+ labeled examples?
    YES → Fine-tune (best quality)
    NO  → Few-shot with chain-of-thought
  NO → Few-shot prompting (start here always)
```

### When to Combine Approaches

```python
# RAG + Fine-Tuning: Best of both worlds
# - Fine-tune for: output format, domain terminology, reasoning style
# - RAG for: factual grounding, up-to-date knowledge, citations

# Example: Internal analytics chatbot
# Fine-tuned to: speak in company terminology, output formatted tables
# RAG provides: actual metric values, report contents, wiki knowledge

# Few-Shot + RAG: Most common practical pattern
# - Few-shot examples in system prompt define format/behavior
# - RAG provides relevant context per query
```

> **Critical Insight:** Start with few-shot prompting. Always. It's free to experiment, takes minutes to set up, and establishes a baseline. Only move to RAG when you need knowledge grounding, and only fine-tune when you've proven few-shot can't achieve the required quality. Most teams over-invest in fine-tuning before exhausting prompt engineering.

---

## Vector Databases and Embedding Infrastructure

### Comparison of Vector Databases

| Database | Type | Best For | Limitations |
|----------|------|----------|-------------|
| **Pinecone** | Managed SaaS | Production, zero-ops | Cost at scale, vendor lock-in |
| **Weaviate** | Open-source | Hybrid search (vector + keyword) | Self-hosting complexity |
| **pgvector** | PostgreSQL extension | Already using Postgres, < 10M vectors | Slower at very large scale |
| **ChromaDB** | Open-source | Prototyping, local dev | Not production-proven at scale |
| **Qdrant** | Open-source | Performance-critical, filtering | Newer ecosystem |
| **FAISS** | Library (Meta) | Offline batch similarity, research | Not a database (no CRUD, no persistence) |
| **Milvus** | Open-source | Very large scale (billions of vectors) | Complex deployment |

### pgvector Example (Most Accessible)

```sql
-- Enable extension
CREATE EXTENSION vector;

-- Create table with embedding column
CREATE TABLE document_chunks (
    id SERIAL PRIMARY KEY,
    content TEXT,
    metadata JSONB,
    embedding vector(1536),  -- OpenAI dimension
    created_at TIMESTAMP DEFAULT NOW()
);

-- Create index for fast similarity search
CREATE INDEX ON document_chunks 
    USING ivfflat (embedding vector_cosine_ops) 
    WITH (lists = 100);  -- Tune lists based on data size

-- Similarity search
SELECT content, metadata,
       1 - (embedding <=> query_embedding) AS similarity
FROM document_chunks
WHERE metadata->>'department' = 'analytics'  -- Pre-filter by metadata
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector  -- Cosine distance
LIMIT 5;
```

### Embedding Model Selection

```
MODEL SELECTION (as of 2025):

  High Quality (best accuracy):
    - OpenAI text-embedding-3-large (3072 dim, $0.13/1M tokens)
    - Cohere embed-v3 (1024 dim)
    - Voyage AI voyage-large-2 (1536 dim)

  Balanced (good quality, reasonable cost):
    - OpenAI text-embedding-3-small (1536 dim, $0.02/1M tokens)
    - sentence-transformers/all-MiniLM-L6-v2 (384 dim, FREE, local)

  Fast/Cheap (prototyping, high volume):
    - sentence-transformers/all-MiniLM-L6-v2 (FREE, runs locally)
    - TF-IDF + SVD (no API needed, interpretable)

  RULE: Start with a free local model for prototyping.
  Switch to API-based only when quality delta is proven.
```

> **Critical Insight:** For most analytics use cases (semantic search over internal docs, ticket clustering), `all-MiniLM-L6-v2` running locally gives 85-90% of the quality of expensive API embeddings at zero marginal cost. Only upgrade to OpenAI/Cohere embeddings when you've measured a meaningful quality gap on YOUR data with YOUR evaluation metrics.

---

## Practical Applications in Analytics

### 1. Automated Report Generation

```python
# Turn structured data into narrative insights
DATA_TO_NARRATIVE_PROMPT = """Convert this weekly metrics data into a 
3-paragraph executive summary. Follow the pyramid principle: lead with 
the most important finding.

METRICS:
{metrics_json}

COMPARISON PERIOD: Week over week
AUDIENCE: VP of Product
TONE: Concise, action-oriented, flag concerns prominently
"""

# Production pattern:
# 1. SQL query generates metrics → JSON
# 2. LLM generates narrative from JSON
# 3. Human reviews and approves before distribution
# 4. Feedback loop: corrections improve prompt over time
```

### 2. Anomaly Explanation

```python
# When an anomaly detector fires, LLM explains the likely cause
ANOMALY_PROMPT = """An anomaly was detected in our metrics:

ANOMALY: {metric_name} dropped {percent_change}% on {date}
NORMAL RANGE: {normal_range}
OBSERVED VALUE: {observed_value}

CORRELATED CHANGES (same time window):
{correlated_metrics}

RECENT EVENTS:
{deployment_log}
{incident_log}

Based on the correlated changes and recent events, provide:
1. Most likely root cause (1 sentence)
2. Confidence level (high/medium/low) with reasoning
3. Recommended investigation steps (3 bullets)
"""
```

### 3. Data Quality Narration

```python
# Explain data quality issues in plain English for stakeholders
DQ_PROMPT = """Translate these data quality check results into a 
stakeholder-friendly summary:

CHECKS FAILED:
- null_rate('email'): 12.4% (threshold: 5%)
- uniqueness('order_id'): 99.2% (threshold: 100%)
- freshness('events'): 4 hours (threshold: 1 hour)

For each failure, explain:
1. What it means in business terms (not technical)
2. What downstream impact it could have
3. Urgency (block analysis vs. proceed with caution)
"""
```

### 4. Semantic Ticket Routing

```python
# Automatically route support tickets using embeddings
categories = {
    "billing": "Issues with charges, refunds, payment methods, invoices",
    "technical": "App crashes, bugs, error messages, functionality issues",
    "shipping": "Delivery delays, tracking, lost packages, address changes",
    "account": "Login problems, password reset, account settings, deletion"
}

# Pre-embed category descriptions
category_embeddings = {cat: model.encode(desc) for cat, desc in categories.items()}

def route_ticket(ticket_text):
    ticket_emb = model.encode(ticket_text)
    similarities = {cat: cosine_similarity(ticket_emb, emb) 
                   for cat, emb in category_embeddings.items()}
    return max(similarities, key=similarities.get)
```

---

## Cost, Latency, and Operational Tradeoffs

### Cost Comparison (as of 2025)

```
TASK: Classify 1M customer reviews into 5 categories

APPROACH 1: LLM (GPT-4o-mini)
  Tokens: ~200 per review (input + output)
  Cost: 1M × 200 tokens × $0.15/1M tokens = $30
  Latency: ~500ms per request (parallel: 2 hours with rate limits)
  Accuracy: ~92%

APPROACH 2: Embeddings + Classifier
  Embedding cost: 1M × 50 tokens × $0.02/1M = $1
  Training: free (sklearn on your machine)
  Inference: <1ms per prediction (local)
  Total: $1 + compute time
  Accuracy: ~88%

APPROACH 3: Fine-tuned small model (distilbert)
  Training: ~$5 (GPU time for 10K examples)
  Inference: <5ms per prediction (local GPU)
  Total: $5 one-time + compute
  Accuracy: ~90%

APPROACH 4: Traditional ML (TF-IDF + LogReg)
  Cost: $0 (local compute)
  Inference: <1ms per prediction
  Accuracy: ~82%

DECISION FACTORS:
  Budget constrained → Approach 4 or 3
  Need highest accuracy → Approach 1
  High volume + good accuracy → Approach 2 or 3
  Rapidly changing categories → Approach 1 (no retraining)
```

### Latency Profiles

```
OPERATION                              TYPICAL LATENCY
─────────────────────────────────────────────────────
Local embedding (MiniLM)               5-20ms
API embedding (OpenAI)                 50-200ms
Vector DB query (top-5)                10-50ms
LLM generation (GPT-4o-mini, short)    300-800ms
LLM generation (GPT-4o, long)          2-10s
Full RAG pipeline                      500ms-3s
Fine-tuned small model inference       5-50ms

IMPLICATION:
  Interactive chat → LLM acceptable (user expects typing delay)
  Batch processing → use embeddings + classifiers (1000x faster)
  Real-time scoring → local models only (< 50ms budget)
```

> **Critical Insight:** The most common mistake in GenAI applications is using a sledgehammer (GPT-4) where a scalpel (embeddings + logistic regression) would do. For batch classification, embedding-based approaches are 100x cheaper and 1000x faster with only a small accuracy tradeoff. Reserve LLM calls for tasks that genuinely need generation: summarization, explanation, and novel text creation.

---

## When NOT to Use LLMs

### The Anti-Pattern Checklist

```
DO NOT use LLMs when:

1. DETERMINISTIC ANSWER EXISTS
   ❌ "What's the average of this column?" → Use SQL/pandas
   ❌ "Is this email valid?" → Use regex
   ❌ "Convert Celsius to Fahrenheit" → Use a formula

2. NUMERICAL PRECISION MATTERS
   ❌ Financial calculations → LLMs can't reliably multiply
   ❌ Statistical tests → Use scipy/statsmodels
   ❌ Metric computation → SQL aggregate functions

3. LATENCY IS CRITICAL (< 50ms)
   ❌ Real-time bidding → Traditional ML models
   ❌ Feature serving → Pre-computed lookups
   ❌ Fraud detection → Gradient boosted trees

4. EXPLAINABILITY IS REQUIRED
   ❌ Regulated decisions (credit, hiring) → Interpretable models
   ❌ Root cause analysis → Causal methods + domain knowledge

5. TRAINING DATA IS ABUNDANT
   ❌ 100K+ labeled examples → Fine-tuned small model wins
   ❌ Well-defined classification → BERT or even TF-IDF + LogReg

6. COST AT SCALE IS PROHIBITIVE
   ❌ Scoring 1B records daily → $150K/day for GPT-4
   ❌ Embedding 100M docs monthly → Consider local models

7. REPRODUCIBILITY IS MANDATORY
   ❌ Audit-trail decisions → LLMs are non-deterministic
   ❌ Regression testing → Outputs vary across calls
```

### The "LLM or Not" Decision Tree

```
Q: Can I write explicit rules for this task?
  YES → Don't use LLM (rules are faster, cheaper, deterministic)
  NO ↓

Q: Do I have > 10K labeled examples?
  YES → Train a task-specific model (cheaper, faster, private)
  NO ↓

Q: Does the task require generating novel text?
  YES → LLM is appropriate
  NO ↓

Q: Does the task require understanding nuance/context?
  YES → LLM or embeddings-based approach
  NO → Traditional ML or rule-based system
```

> **Critical Insight:** In interviews, showing restraint about LLM usage is MORE impressive than enthusiasm. Saying "I'd use GPT-4 for that" for every question signals inexperience. Saying "For this classification task, I'd start with embeddings + logistic regression because we have 50K labeled examples and need sub-100ms latency — LLM would be overkill" signals maturity and cost-awareness.

---

## Interview Talking Points

> "I built a RAG system over our internal analytics wiki and past quarterly reports. The key challenge wasn't the LLM — it was chunking strategy. I tested chunk sizes from 200 to 2000 characters and found 500 characters with 50-character overlap gave the best retrieval precision on our evaluation set of 100 historical questions. The system reduced time-to-answer for recurring stakeholder questions from 30 minutes (searching Confluence) to 30 seconds."

> "For our customer feedback classification pipeline processing 500K tickets monthly, I deliberately chose NOT to use an LLM. Instead, I embedded tickets with sentence-transformers and trained a gradient-boosted classifier on 5K labeled examples. This gives us 89% accuracy at $0 marginal cost per prediction and 2ms latency — versus $75/month and 500ms latency with GPT-4o-mini. The 3% accuracy gap wasn't worth the 1000x cost increase for our use case."

> "I evaluate our text-to-SQL system using execution accuracy, not just syntactic similarity. The generated SQL might look different from the gold standard but return identical results. Our evaluation suite has 200 questions covering joins, aggregations, window functions, and edge cases. Current accuracy is 78% — the failures cluster around ambiguous questions where the system can't infer which date column to filter on, which we've mitigated with a clarification prompt."

---

## Common Mistakes

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Using GPT-4 for simple classification with abundant labels | Train a task-specific model (cheaper, faster, private) |
| Evaluating only with BLEU/ROUGE (surface metrics) | Use execution-based eval + human review + LLM-as-judge |
| Skipping retrieval evaluation in RAG systems | Measure retrieval precision/recall separately from generation quality |
| One giant chunk per document in RAG | Chunk at 300-500 chars with overlap, experiment with size |
| Fine-tuning for factual knowledge | Use RAG for knowledge (updatable, citable, verifiable) |
| No guardrails on text-to-SQL outputs | Validate schema, run EXPLAIN, restrict to SELECT, add user confirmation |
| Using LLMs for numerical computation | LLMs can't reliably do math — use code/SQL for calculations |
| Ignoring cost at scale (prototype trap) | Calculate cost at production volume BEFORE committing to LLM approach |
| Not versioning prompts | Treat prompts like code: version control, A/B test, monitor drift |
| Assuming LLM outputs are deterministic | Same prompt can give different outputs; design for non-determinism |

---

## Rapid-Fire Q&A

**Q1: What is RAG and when would you use it?**
Retrieval-Augmented Generation retrieves relevant documents from a vector store and includes them in the LLM prompt for grounded answers. Use when you need LLM responses based on proprietary/internal knowledge that wasn't in training data. Preferred over fine-tuning for factual knowledge because it's updatable and citable.

**Q2: How do you evaluate a text-to-SQL system?**
Execution accuracy (run both generated and gold SQL, compare result sets) is the gold standard. Supplement with: exact match rate, partial credit for correct structure, error taxonomy (wrong table, wrong filter, wrong aggregation), and human review of edge cases.

**Q3: When would you fine-tune vs use few-shot prompting?**
Start with few-shot always. Fine-tune when: you need consistent specialized behavior that few-shot can't achieve, you have 500+ examples, you need lower latency/cost at scale, or the task requires domain-specific reasoning that prompting alone can't induce.

**Q4: What's the difference between embeddings and LLM generation?**
Embeddings map text to fixed-size vectors (numerical representations) useful for similarity, search, and classification. Generation produces new text. Embeddings are cheap, fast, and deterministic. Generation is expensive, slow, and non-deterministic. Use embeddings when you need to compare/classify; use generation when you need to create/explain.

**Q5: How would you build a system to auto-generate weekly reports?**
Pipeline: SQL computes metrics (deterministic), LLM generates narrative from structured data (format + context), human reviews before distribution. Evaluation: compare generated narratives against manually written ones, track stakeholder satisfaction, flag factual errors. Key: LLM only for narration, never for computation.

**Q6: What are the main failure modes of RAG?**
(1) Retrieval failure — relevant docs not returned (bad chunking, wrong embedding model). (2) Context window overflow — too many chunks dilute relevance. (3) Hallucination despite context — LLM ignores retrieved docs. (4) Stale index — docs updated but embeddings not refreshed. (5) Wrong granularity — chunks too large (noise) or too small (no context).

**Q7: How do you handle LLM hallucination in production?**
(1) Ground with RAG (cite sources). (2) Constrain output format (JSON schema, SQL with schema validation). (3) Add verification step (run generated code, check against source). (4) Temperature=0 for factual tasks. (5) Ask LLM to flag uncertainty. (6) Human-in-the-loop for high-stakes outputs.

**Q8: What's a vector database and when do you need one?**
A database optimized for storing and querying high-dimensional vectors (embeddings). You need one when: doing similarity search at scale (>100K vectors), building RAG systems, or doing real-time nearest-neighbor lookups. For <100K vectors, FAISS in-memory or pgvector extension may suffice.

**Q9: How do you choose between OpenAI, open-source, and local models?**
OpenAI/Anthropic: best quality, highest cost, data leaves your infra. Open-source hosted (Llama via Bedrock/Together): good quality, moderate cost, more control. Local models: cheapest at scale, full privacy, worst quality. Decision factors: data sensitivity, volume, quality requirement, and team ML ops capability.

**Q10: How is GenAI changing the data scientist role in 2025?**
Data scientists now need: prompt engineering skills, evaluation methodology for non-deterministic systems, cost-aware architecture decisions, and the judgment to know when LLMs are overkill. The role shifts from "build ML models" to "orchestrate AI systems" — choosing the right tool (LLM, embedding, traditional ML, rules) for each sub-problem in a pipeline.

---

## ASCII Cheat Sheet

```
+============================================================+
|   GenAI & LLM APPLICATIONS IN ANALYTICS — CHEAT SHEET      |
+============================================================+

EMBEDDING USE CASES:
  Classification:  embed text → train classifier on vectors
  Clustering:      embed text → KMeans/HDBSCAN on vectors
  Similarity:      cosine_similarity(embed(A), embed(B))
  Search:          embed query → find nearest in vector DB
  Deduplication:   high similarity → likely duplicate

RAG PIPELINE:
  1. CHUNK documents (300-500 chars, with overlap)
  2. EMBED chunks → store in vector DB
  3. EMBED user query → retrieve top-k similar chunks
  4. AUGMENT prompt with retrieved context
  5. GENERATE answer grounded in context
  6. CITE sources from metadata

DECISION FRAMEWORK:
  Need knowledge?     → RAG
  Need style/format?  → Fine-tune (or few-shot first)
  Need reasoning?     → Few-shot with chain-of-thought
  Need classification? → Embeddings + simple classifier
  Need computation?   → NEVER LLM — use code/SQL

PROMPT ENGINEERING FOR DATA:
  Text-to-SQL:    schema + rules + examples + question
  Summarization:  data + audience + format + constraints
  Extraction:     text + output schema + examples
  Always:         constrain output format, include examples

EVALUATION METHODS:
  Automated:  BLEU, ROUGE, BERTScore, Exact Match
  Execution:  Run generated code, compare results (BEST)
  LLM-Judge:  Stronger model scores weaker model output
  Human:      Expert rating, A/B preference, error taxonomy
  
  GOLDEN RULE: Match eval to task success metric

COST HIERARCHY (cheapest → most expensive per prediction):
  1. Rules/regex               ($0, <1ms)
  2. TF-IDF + LogReg           ($0, <1ms)
  3. Local embedding + clf     ($0, ~10ms)
  4. API embedding + clf       (~$0.001, ~100ms)
  5. Small fine-tuned LLM      (~$0.01, ~200ms)
  6. Large LLM (GPT-4o-mini)   (~$0.03, ~500ms)
  7. Frontier LLM (GPT-4o)     (~$0.30, ~3s)

WHEN NOT TO USE LLMs:
  ✗ Deterministic answers exist (SQL, formulas, rules)
  ✗ Numerical precision required (math, stats)
  ✗ Latency < 50ms needed (real-time systems)
  ✗ Explainability mandatory (regulated decisions)
  ✗ Abundant labeled data (train task-specific model)
  ✗ Cost prohibitive at scale (billions of predictions)
  ✗ Reproducibility required (audit trails)

VECTOR DB SELECTION:
  Prototyping:        ChromaDB or FAISS
  Already on Postgres: pgvector (< 10M vectors)
  Production SaaS:    Pinecone (zero-ops)
  Hybrid search:      Weaviate (vector + keyword)
  Massive scale:      Milvus (billions of vectors)

+============================================================+
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 47 of 51*
