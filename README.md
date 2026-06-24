# Data Science & Analytics — Interview Knowledge Base

> A comprehensive, interview-ready knowledge base for **Data Scientist / Data Analyst** roles at big tech (Google, Meta, Amazon, Netflix, Spotify, Uber, Apple, Microsoft).  
> Built for 5+ YOE candidates. Not generic prep — deep domain knowledge that separates senior from junior.

---

## Repository Structure

```
DataScience/
├── python/                    → 25 guides: Python for DS interviews
│   ├── 01 to 10              → Core Python (data types → advanced pandas)
│   ├── 11 to 16             → Stats, ML, A/B testing, visualization
│   └── 17 to 25             → NLP, SQL, deep learning, deployment, case studies
│
├── important-topics/          → 51 deep-dive topics (the "System Design" of DS)
│   ├── 01-07                 → SQL mastery
│   ├── 08-14                 → Data modeling & warehousing
│   ├── 15-21                 → Pipelines & analytics engineering
│   ├── 22-28                 → Experimentation & causal inference
│   ├── 29-35                 → Applied machine learning
│   ├── 36-45                 → Metrics, data quality, communication
│   └── 46-51                 → Advanced: Spark, GenAI, survival, attribution, search, anomaly detection
│
├── sql-practice/              → 50 graded SQL problems (Easy → Hard)
└── README.md                  → This file
```

---

## 6-Week Study Plan

### Week 1: SQL & Data Manipulation
| Day | Focus | Resources |
|-----|-------|-----------|
| 1-2 | Joins, subqueries, CTEs | `important-topics/01-03` |
| 3-4 | Window functions, aggregation | `important-topics/02, 04` |
| 5 | Cohort & funnel queries | `important-topics/06` |
| 6-7 | SQL practice problems (Easy + Medium) | `sql-practice/` |

### Week 2: Statistics & Experimentation
| Day | Focus | Resources |
|-----|-------|-----------|
| 1-2 | Probability, distributions, hypothesis testing | `python/11` |
| 3-4 | A/B test design, power analysis, metric selection | `important-topics/22-24` |
| 5-6 | Pitfalls, causal inference, quasi-experiments | `important-topics/25-28` |
| 7 | Practice: design an A/B test end-to-end | `important-topics/22` |

### Week 3: Data Modeling & Pipelines
| Day | Focus | Resources |
|-----|-------|-----------|
| 1-2 | Star/snowflake, fact/dim, SCD | `important-topics/08-10` |
| 3-4 | OLTP/OLAP, warehouse architecture, lakehouse | `important-topics/12-14` |
| 5-6 | ETL/ELT, DAGs, dbt, batch vs streaming | `important-topics/15-18` |
| 7 | Practice: design a data model for Uber trips | `important-topics/08` |

### Week 4: Machine Learning & Product Metrics
| Day | Focus | Resources |
|-----|-------|-----------|
| 1-2 | Supervised ML, model selection, evaluation | `python/14`, `important-topics/30-31` |
| 3-4 | Feature engineering, imbalanced data, regularization | `important-topics/29, 32-33` |
| 5-6 | KPIs, retention, metric decomposition | `important-topics/36-39` |
| 7 | Practice: "Revenue dropped 10% — investigate" | `important-topics/39` |

### Week 5: Python Coding & Advanced Topics
| Day | Focus | Resources |
|-----|-------|-----------|
| 1-2 | Pandas deep dive, data cleaning | `python/09-10, 23` |
| 3-4 | Coding patterns, algorithms for DS | `python/21` |
| 5-6 | Spark/PySpark, GenAI for analytics | `important-topics/46-47` |
| 7 | SQL practice problems (Hard) | `sql-practice/` |

### Week 6: Case Studies & Communication
| Day | Focus | Resources |
|-----|-------|-----------|
| 1-2 | End-to-end case studies | `python/25` |
| 3-4 | Recommendation systems, search ranking | `important-topics/34, 50` |
| 5-6 | Data storytelling, stakeholder communication | `important-topics/44-45` |
| 7 | Mock run: pick 3 random topics, explain in 5 min each | All |

---

## How This Maps to Interview Rounds

| Interview Round | What They Test | Study |
|----------------|---------------|-------|
| **SQL Coding** | Live SQL problems (45-60 min) | `important-topics/01-07` + `sql-practice/` |
| **Python/Coding** | Pandas, data manipulation, algorithms | `python/01-25` |
| **Statistics/Experimentation** | A/B testing, hypothesis testing, causal | `important-topics/22-28` + `python/11, 16` |
| **Product Sense / Metrics** | Define metrics, debug drops, measure success | `important-topics/36-40` |
| **Data Modeling / System Design** | Design data warehouse, pipeline architecture | `important-topics/08-21` |
| **Machine Learning** | Applied ML, model selection, evaluation | `important-topics/29-35` + `python/14-15` |
| **Case Study / Take-Home** | End-to-end project | `python/25` |

---

## Comparison to SDE Prep

| SDE Track | Data Track | This Repo |
|-----------|-----------|-----------|
| DSA (LeetCode) | SQL + Query Logic | `important-topics/01-07` + `sql-practice/` |
| System Design (HLD) | Data Modeling + Warehouse | `important-topics/08-14` |
| Low-Level Design (LLD) | Pipelines + dbt + Transforms | `important-topics/15-21` |
| Production Systems | Experimentation + Metrics + ML | `important-topics/22-45` |

---

## Key Principles

1. **Each file is self-contained** — read any topic independently
2. **Interview scripts included** — rehearsable paragraphs you can adapt
3. **Code is embedded** — Python/SQL in markdown blocks, not separate files
4. **Mistakes section in every file** — learn what NOT to say
5. **Quantified reasoning** — numbers, not hand-waving

---

*Inspired by [System-Design-Resources](https://github.com/aerlokesh/System-Design-Resources) — adapted for Data Science.*
