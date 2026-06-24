# 🎯 Topic 12: Data Visualization & EDA

> A comprehensive guide to exploratory data analysis, chart selection, and visual storytelling — covering Matplotlib, Seaborn, Plotly, accessibility, and how to answer "How would you visualize X?" in big tech interviews.

---

## Table of Contents

1. [Matplotlib Architecture](#1-matplotlib-architecture)
2. [Seaborn for Statistical Visualization](#2-seaborn-for-statistical-visualization)
3. [Chart Type Selection Guide](#3-chart-type-selection-guide)
4. [EDA Patterns & Workflow](#4-eda-patterns--workflow)
5. [Color Accessibility & Colorblind-Safe Palettes](#5-color-accessibility--colorblind-safe-palettes)
6. [Plotly for Interactive Charts](#6-plotly-for-interactive-charts)
7. [Interview Talking Points & Scripts](#7-interview-talking-points--scripts)
8. [Common Interview Mistakes](#8-common-interview-mistakes)
9. [Rapid-Fire Q&A](#9-rapid-fire-qa)
10. [Summary Cheat Sheet](#10-summary-cheat-sheet)

---

## 1. Matplotlib Architecture

### Figure, Axes, and Subplots

Matplotlib uses an **object-oriented hierarchy**: `Figure` > `Axes` > `Axis` > `Tick`. Understanding this is critical for customization beyond basic plots.

> 💡 **Critical Insight:** Never use `plt.plot()` in production code or interview demos. Always use the explicit `fig, ax = plt.subplots()` pattern — it signals you understand the OO API and gives full control over multi-panel figures.

```python
import matplotlib.pyplot as plt
import numpy as np

# The professional pattern: explicit Figure + Axes
fig, axes = plt.subplots(2, 2, figsize=(12, 8), dpi=100)
fig.suptitle("Q4 User Engagement Metrics", fontsize=14, fontweight='bold')

# Each axes object is independently configurable
x = np.linspace(0, 10, 100)
axes[0, 0].plot(x, np.sin(x), color='#1f77b4', linewidth=2)
axes[0, 0].set_title("DAU Trend")
axes[0, 0].set_xlabel("Week")
axes[0, 0].set_ylabel("Users (M)")
axes[0, 0].spines[['top', 'right']].set_visible(False)  # Clean look

axes[0, 1].bar(['iOS', 'Android', 'Web'], [45, 38, 17], color=['#2196F3', '#4CAF50', '#FF9800'])
axes[0, 1].set_title("Platform Split (%)")

axes[1, 0].scatter(np.random.randn(50), np.random.randn(50), alpha=0.6, edgecolors='black', linewidth=0.5)
axes[1, 0].set_title("Feature Correlation")

axes[1, 1].hist(np.random.exponential(2, 1000), bins=30, color='#9C27B0', edgecolor='white')
axes[1, 1].set_title("Session Duration Distribution")

plt.tight_layout()
plt.savefig("engagement_dashboard.png", bbox_inches='tight', facecolor='white')
plt.show()
```

### Key Customization Patterns

```python
# Twin axes for dual-scale plots (revenue vs users on same chart)
fig, ax1 = plt.subplots(figsize=(10, 5))
ax2 = ax1.twinx()

months = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun']
revenue = [1.2, 1.5, 1.8, 2.1, 2.4, 2.9]
users = [100, 120, 135, 160, 180, 210]

ax1.bar(months, revenue, color='#4CAF50', alpha=0.7, label='Revenue ($M)')
ax2.plot(months, users, color='#F44336', marker='o', linewidth=2, label='MAU (K)')

ax1.set_ylabel("Revenue ($M)", color='#4CAF50')
ax2.set_ylabel("MAU (K)", color='#F44336')
ax1.legend(loc='upper left')
ax2.legend(loc='upper right')
plt.title("Revenue vs User Growth")
```

---

## 2. Seaborn for Statistical Visualization

Seaborn builds on Matplotlib with statistical semantics and beautiful defaults. It integrates with Pandas DataFrames natively.

> 💡 **Critical Insight:** In interviews, Seaborn signals you think statistically — it automatically adds confidence intervals, kernel density estimates, and handles categorical groupings. Use it when exploring relationships, not just plotting raw values.

```python
import seaborn as sns
import pandas as pd

# Set a clean, professional theme
sns.set_theme(style="whitegrid", palette="colorblind", font_scale=1.1)

# Distribution analysis
fig, axes = plt.subplots(1, 3, figsize=(15, 5))

# KDE + Histogram (replacement for deprecated distplot)
sns.histplot(data=df, x='session_duration', kde=True, ax=axes[0], color='steelblue')
axes[0].set_title("Session Duration Distribution")

# Boxplot with jittered points — shows outliers + distribution shape
sns.boxplot(data=df, x='platform', y='revenue_per_user', ax=axes[1], palette='Set2')
sns.stripplot(data=df, x='platform', y='revenue_per_user', ax=axes[1],
              color='black', alpha=0.3, size=3)
axes[1].set_title("Revenue by Platform")

# Violin plot — KDE + box in one
sns.violinplot(data=df, x='cohort', y='retention_d7', ax=axes[2], inner='box')
axes[2].set_title("D7 Retention by Cohort")

plt.tight_layout()
```

### Correlation Heatmap

```python
# Correlation matrix with masking (upper triangle)
corr = df[['dau', 'session_length', 'purchases', 'notifications_sent', 'retention']].corr()
mask = np.triu(np.ones_like(corr, dtype=bool))

fig, ax = plt.subplots(figsize=(8, 6))
sns.heatmap(corr, mask=mask, annot=True, fmt='.2f', cmap='RdBu_r',
            center=0, vmin=-1, vmax=1, square=True, linewidths=0.5,
            cbar_kws={"shrink": 0.8})
ax.set_title("Feature Correlation Matrix")
```

### Pairplot for Multivariate EDA

```python
# Pairplot with hue — instant multivariate overview
g = sns.pairplot(df[['age', 'income', 'spend', 'satisfaction', 'churn']],
                 hue='churn', palette={'Yes': '#E53935', 'No': '#43A047'},
                 diag_kind='kde', plot_kws={'alpha': 0.5, 'edgecolor': None})
g.fig.suptitle("Churn Drivers — Pairwise Relationships", y=1.02)
```

---

## 3. Chart Type Selection Guide

| Data Relationship | Best Chart Type | When to Use | Avoid |
|---|---|---|---|
| Trend over time | **Line chart** | Continuous time series, showing trajectory | Bars for >7 time points |
| Comparison across categories | **Bar chart** (horizontal for many) | Discrete groups, rankings | Pie charts for >5 slices |
| Distribution of one variable | **Histogram / KDE** | Understanding shape, skew, modality | Bar charts (they imply categories) |
| Distribution + outliers | **Box plot / Violin** | Comparing distributions across groups | Just showing means |
| Two-variable relationship | **Scatter plot** | Correlation, clusters, outliers | Line chart (implies ordering) |
| Part-of-whole | **Stacked bar / Treemap** | Composition breakdowns | Pie charts (hard to compare) |
| Correlation matrix | **Heatmap** | Many pairwise relationships | Individual scatter plots (too many) |
| Geospatial | **Choropleth / Bubble map** | Location-based patterns | Tables of lat/long values |
| Ranking | **Horizontal bar** | Top-N, leaderboards | Vertical bars with rotated labels |
| Flow / Funnel | **Funnel chart / Sankey** | User journeys, conversion steps | Bar charts (lose sequential meaning) |

> 💡 **Critical Insight:** In interviews, always state WHY you chose a chart type. Say: "I would use a box plot here because we need to compare distributions across groups AND identify outliers — a bar chart of means would hide the variance."

---

## 4. EDA Patterns & Workflow

### The Systematic EDA Framework (Interview-Ready)

```python
def systematic_eda(df, target_col=None):
    """
    Structured EDA workflow — use this mental framework in interviews.
    Phase 1: Shape & Schema
    Phase 2: Missing & Quality
    Phase 3: Univariate Distributions
    Phase 4: Bivariate Relationships
    Phase 5: Multivariate Patterns
    """

    # PHASE 1: Shape & Schema
    print(f"Shape: {df.shape}")
    print(f"Dtypes:\n{df.dtypes.value_counts()}")
    print(f"Memory: {df.memory_usage(deep=True).sum() / 1e6:.1f} MB")

    # PHASE 2: Missing Data & Quality
    missing = df.isnull().sum()
    missing_pct = (missing / len(df) * 100).sort_values(ascending=False)
    print(f"\nMissing (>0%):\n{missing_pct[missing_pct > 0]}")

    # Visualize missingness pattern
    fig, ax = plt.subplots(figsize=(10, 4))
    sns.heatmap(df.isnull().T, cbar=False, cmap='Reds', yticklabels=True, ax=ax)
    ax.set_title("Missing Data Pattern (red = missing)")

    # PHASE 3: Univariate — numeric distributions
    numeric_cols = df.select_dtypes(include='number').columns
    n_cols = min(len(numeric_cols), 12)
    fig, axes = plt.subplots(3, 4, figsize=(16, 10))
    for i, col in enumerate(numeric_cols[:n_cols]):
        ax = axes.flatten()[i]
        sns.histplot(df[col].dropna(), kde=True, ax=ax, color='steelblue')
        ax.axvline(df[col].median(), color='red', linestyle='--', label='median')
        ax.set_title(f"{col} (skew={df[col].skew():.2f})")
    plt.tight_layout()

    # PHASE 4: Bivariate — correlation with target
    if target_col:
        correlations = df[numeric_cols].corrwith(df[target_col]).sort_values()
        fig, ax = plt.subplots(figsize=(8, 6))
        correlations.plot(kind='barh', ax=ax, color=['#E53935' if x < 0 else '#43A047' for x in correlations])
        ax.set_title(f"Correlation with {target_col}")
        ax.axvline(0, color='black', linewidth=0.5)

    # PHASE 5: Multivariate — top correlated features pairplot
    if target_col:
        top_features = correlations.abs().nlargest(5).index.tolist()
        sns.pairplot(df[top_features + [target_col]], hue=target_col)
```

### EDA Red Flags to Call Out in Interviews

| Signal | What It Means | Action |
|---|---|---|
| Bimodal distribution | Likely mixed populations | Segment and analyze separately |
| Missing Not At Random (MNAR) | Missingness carries information | Feature-engineer the missingness |
| Extreme skew (>2) | Mean is misleading | Use median; consider log transform |
| High cardinality categoricals | Sparse encodings, overfitting risk | Group rare categories; use target encoding |
| Perfect correlation (r=1.0) | Data leakage or duplicate feature | Investigate and drop one |
| Sudden distribution shifts | Time-based data drift | Split before/after; flag to stakeholders |

---

## 5. Color Accessibility & Colorblind-Safe Palettes

> 💡 **Critical Insight:** ~8% of men and ~0.5% of women have color vision deficiency. At scale (millions of users viewing your dashboards), this is NOT optional. Interviewers at Google and Meta specifically look for accessibility awareness.

### Recommended Palettes

| Palette | Use Case | Source |
|---|---|---|
| `colorblind` (Seaborn) | Default for all categorical plots | Seaborn built-in |
| `viridis` / `plasma` | Sequential numeric data | Matplotlib perceptually uniform |
| `RdBu_r` (diverging) | Correlation matrices, deviation from center | Matplotlib |
| `cividis` | Fully colorblind-safe sequential | Matplotlib |
| IBM Design palette | Enterprise dashboards | `['#648FFF','#785EF0','#DC267F','#FE6100','#FFB000']` |

```python
# Setting colorblind-safe defaults globally
sns.set_palette("colorblind")
plt.rcParams['axes.prop_cycle'] = plt.cycler(
    color=['#0077BB', '#33BBEE', '#009988', '#EE7733', '#CC3311', '#EE3377', '#BBBBBB']
)

# Wong's colorblind-safe palette (widely cited in academic visualization)
wong_palette = ['#000000', '#E69F00', '#56B4E9', '#009E73', '#F0E442', '#0072B2', '#D55E00', '#CC79A7']

# Always supplement color with shape/pattern/label
fig, ax = plt.subplots(figsize=(8, 5))
markers = ['o', 's', '^', 'D']
for i, group in enumerate(df['segment'].unique()):
    subset = df[df['segment'] == group]
    ax.scatter(subset['x'], subset['y'], label=group,
               color=wong_palette[i], marker=markers[i], s=60, alpha=0.7)
ax.legend(title='Segment')
ax.set_title("Segments (color + shape for accessibility)")
```

### Accessibility Checklist

- Use color AND shape/pattern to encode categories
- Ensure sufficient contrast ratio (WCAG AA: 4.5:1 for text)
- Add direct labels instead of relying solely on legends
- Test with a colorblind simulator (e.g., Coblis, Sim Daltonism)
- Avoid red-green as the only differentiator

---

## 6. Plotly for Interactive Charts

Plotly is essential for dashboards, stakeholder presentations, and notebook-based EDA where interactivity adds value.

```python
import plotly.express as px
import plotly.graph_objects as go
from plotly.subplots import make_subplots

# Quick interactive scatter with hover data
fig = px.scatter(df, x='ad_spend', y='conversions', color='channel',
                 size='impressions', hover_data=['campaign_name', 'roi'],
                 title="Ad Spend vs Conversions by Channel",
                 template='plotly_white')
fig.update_traces(marker=dict(line=dict(width=1, color='DarkSlateGrey')))
fig.show()

# Interactive time series with range slider
fig = px.line(df, x='date', y='revenue', color='region',
              title="Revenue Trend by Region")
fig.update_xaxes(rangeslider_visible=True)
fig.update_layout(hovermode='x unified')
fig.show()

# Funnel chart for conversion analysis
fig = go.Figure(go.Funnel(
    y=['Homepage', 'Product Page', 'Add to Cart', 'Checkout', 'Purchase'],
    x=[10000, 6500, 3200, 1800, 1200],
    textinfo="value+percent initial",
    marker=dict(color=['#3F51B5', '#2196F3', '#03A9F4', '#00BCD4', '#4CAF50'])
))
fig.update_layout(title="Purchase Funnel (Q4 2024)")
fig.show()
```

### When to Use Plotly vs Matplotlib/Seaborn

| Criterion | Matplotlib/Seaborn | Plotly |
|---|---|---|
| Static reports / papers | Preferred | Overkill |
| Jupyter exploration | Good | Better (hover, zoom) |
| Stakeholder dashboards | Limited | Ideal |
| Publication-quality figures | Best control | Possible but harder |
| Large datasets (>100K pts) | Faster rendering | Can lag (use WebGL) |
| Reproducible scripts | Standard | Requires HTML export |

---

## 7. Interview Talking Points & Scripts

### "How would you visualize the results of an A/B test?"

> *"I would create a multi-panel visualization. First, a bar chart with confidence intervals showing the point estimates and uncertainty for control vs treatment — this immediately communicates whether the difference is statistically significant by whether the CIs overlap. Second, I would include a distribution plot (KDE or histogram) of the metric for both groups overlaid, so stakeholders can see the full shape, not just the mean. If it is a time-series metric, I would add a line chart showing the metric trajectory for both groups over the experiment duration to detect novelty effects or ramp-up periods. I always annotate the chart with the p-value, effect size, and sample sizes directly on the figure."*

### "Walk me through your EDA process for a new dataset."

> *"I follow a five-phase systematic approach. Phase one: understand shape, schema, and memory footprint. Phase two: assess data quality — missing values, duplicates, and whether missingness is random or informative. Phase three: univariate distributions for every feature, looking for skew, multimodality, and outliers. Phase four: bivariate relationships, particularly correlations with the target variable. Phase five: multivariate exploration using pairplots or dimensionality reduction like PCA/t-SNE. Throughout, I document hypotheses and anomalies. The goal is not just description but generating testable hypotheses for modeling."*

### "How would you present a complex analysis to non-technical stakeholders?"

> *"I follow three principles: lead with the insight, not the method; use no more than one chart per key finding; and annotate directly on the visualization rather than using legends that require back-and-forth reading. I would start with a single headline number or takeaway, then support it with one clean chart. I remove all chartjunk — gridlines, excessive tick marks, unnecessary borders. If showing a trend, I highlight the relevant segment with color and grey out the rest. I always include a 'so what' annotation: not just 'revenue increased 15%' but 'revenue increased 15%, driven by the new checkout flow — recommend expanding to mobile.'"*

### "What visualization would you use for high-dimensional data?"

> *"For initial exploration, I would use a correlation heatmap to identify clusters of related features. For pattern discovery, I would apply PCA or t-SNE/UMAP to reduce to 2-3 dimensions and create an interactive scatter plot colored by the target variable or a suspected grouping. For feature importance communication, a horizontal bar chart of SHAP values or feature importances sorted by magnitude. The key is matching the visualization to the audience: PCA biplots for technical peers, simplified bar charts of top drivers for executives."*

---

## 8. Common Interview Mistakes

| ❌ Mistake | ✅ Better Approach |
|---|---|
| Starting y-axis at non-zero to exaggerate differences | Always start at zero for bar charts; explain if truncating line charts |
| Using pie charts with >5 categories | Use horizontal bar chart sorted by value |
| Showing only means without variance | Add error bars, box plots, or confidence intervals |
| Using 3D charts | Stick to 2D — 3D distorts perception of values |
| Red/green encoding without alternative cues | Use colorblind-safe palette + shape/pattern redundancy |
| Overloading a single chart with too many series | Facet/small multiples or highlight one series at a time |
| No axis labels or unclear units | Always label axes with units (e.g., "Revenue ($M)") |
| Using `plt.plot()` without explicit figure/axes | Use `fig, ax = plt.subplots()` for reproducible, customizable plots |
| Presenting raw data dumps instead of insights | Lead with the finding, use chart as supporting evidence |
| Choosing chart type based on aesthetics | Choose based on the data relationship you want to communicate |
| Ignoring aspect ratio (stretching time series) | Use appropriate width:height ratio (16:9 for time series, 1:1 for scatter) |
| Not explaining what the visualization shows | Add a descriptive title that states the insight, not just the variable name |

---

## 9. Rapid-Fire Q&A

**Q1: When should you use a log scale?**
A: When data spans multiple orders of magnitude (e.g., income distribution, web traffic) or when you want to show multiplicative/percentage changes rather than absolute changes.

**Q2: What is the difference between `sns.histplot` and `sns.kdeplot`?**
A: `histplot` bins data into discrete bars (optionally with KDE overlay); `kdeplot` shows only the smooth kernel density estimate. Use histplot for actual counts, kdeplot for comparing distribution shapes across groups.

**Q3: How do you handle overplotting in scatter plots?**
A: Use alpha transparency, jittering, 2D density/hexbin plots, or sample a representative subset. For large datasets, use `datashader` or Plotly with WebGL rendering.

**Q4: Why are dual-axis charts controversial?**
A: They can mislead by implying correlation where none exists (different scales can make unrelated trends appear connected). Always label clearly and consider separate panels instead.

**Q5: What makes a good dashboard vs a good static report figure?**
A: Dashboards need interactivity (filters, drill-down), real-time data, and multiple metrics at a glance. Static figures need to be self-contained — every insight readable without clicking, with annotations and clear captions.

**Q6: How do you choose bin width for histograms?**
A: Start with Sturges' rule (`bins = 1 + log2(n)`) or Freedman-Diaconis (`bin_width = 2*IQR*n^(-1/3)`). Then adjust visually — too few bins hides shape, too many creates noise.

**Q7: When is a table better than a chart?**
A: When exact values matter more than patterns (financial reports, parameter tables), when there are fewer than ~5 data points, or when the audience needs to look up specific numbers.

**Q8: How do you visualize uncertainty?**
A: Confidence interval bands on line charts, error bars on bar charts, fan charts for forecasts, or bootstrap distribution plots. Never show point estimates alone for statistical results.

**Q9: What is small multiples (faceting) and when to use it?**
A: Repeating the same chart type across panels for different subgroups (e.g., one line chart per region). Use when you have 3-12 categories and want to compare patterns without overplotting.

**Q10: How would you visualize a confusion matrix?**
A: As an annotated heatmap with counts AND percentages, using a sequential colormap. Normalize by row (recall per class) or column (precision per class) depending on what matters most.

---

## 10. Summary Cheat Sheet

```
+------------------------------------------------------------------------+
|                   DATA VISUALIZATION CHEAT SHEET                        |
+------------------------------------------------------------------------+
|                                                                        |
|  MATPLOTLIB PATTERN:                                                   |
|    fig, ax = plt.subplots(nrows, ncols, figsize=(w, h))               |
|    ax.plot/bar/scatter/hist(...)                                       |
|    ax.set_title/xlabel/ylabel(...)                                     |
|    plt.tight_layout() -> plt.savefig(...)                              |
|                                                                        |
|  SEABORN ESSENTIALS:                                                   |
|    sns.histplot()    - distributions (replaced distplot)               |
|    sns.boxplot()     - compare distributions + outliers                |
|    sns.heatmap()     - correlation matrices                            |
|    sns.pairplot()    - multivariate overview                           |
|    sns.lmplot()      - scatter + regression line                       |
|                                                                        |
|  CHART SELECTION RULES:                                                |
|    Trend -> Line  |  Compare -> Bar  |  Distribute -> Hist/Box        |
|    Relate -> Scatter  |  Compose -> Stacked Bar  |  Correlate -> Heat |
|                                                                        |
|  EDA PHASES:                                                           |
|    1. Shape/Schema  2. Quality/Missing  3. Univariate                  |
|    4. Bivariate     5. Multivariate                                    |
|                                                                        |
|  ACCESSIBILITY:                                                        |
|    - Use colorblind palette (sns.set_palette("colorblind"))            |
|    - Encode with color AND shape/pattern                               |
|    - Direct labels > legends                                           |
|    - Test with colorblind simulator                                    |
|                                                                        |
|  PLOTLY QUICK START:                                                   |
|    px.scatter/line/bar/histogram(df, x=, y=, color=, ...)             |
|    fig.update_layout(template='plotly_white')                          |
|                                                                        |
|  INTERVIEW GOLDEN RULES:                                               |
|    1. State WHY you chose the chart type                               |
|    2. Always show uncertainty (CIs, error bars)                        |
|    3. Title = insight, not variable name                               |
|    4. Lead with finding, chart is evidence                             |
|    5. Mention accessibility unprompted                                  |
|                                                                        |
+------------------------------------------------------------------------+
```

---

*Part of the [Data Science Interview Prep](../README.md) series — Topic 12 of 25*
