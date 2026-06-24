# 🎯 Topic 60: Unit Economics & Business Case Analysis

> *"Data scientists who can connect analysis to dollars get promoted. Those who can't stay in the analytics ghetto forever."*

---

## Table of Contents

1. [Unit Economics Framework](#unit-economics-framework)
2. [Customer Acquisition Cost (CAC)](#cac)
3. [Lifetime Value (LTV)](#ltv)
4. [LTV:CAC Ratio and Payback Period](#ltv-cac-ratio)
5. [Contribution Margin and Break-Even](#contribution-margin)
6. [NPV for Cohort Economics](#npv-cohort-economics)
7. [Sensitivity Analysis](#sensitivity-analysis)
8. [Business Case Frameworks](#business-case-frameworks)
9. [Practical Cases](#practical-cases)
10. [Structuring Business Case Interviews](#structuring-interviews)
11. [Connecting Data to Dollars](#connecting-data-to-dollars)
12. [Interview Talking Points](#interview-talking-points)
13. [Common Mistakes](#common-mistakes)
14. [Rapid-Fire Q&A](#rapid-fire-qa)
15. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Unit Economics Framework {#unit-economics-framework}

Unit economics answers: **"Do we make money on each transaction/customer?"**

```
Unit Profit = Revenue per Unit - Cost per Unit

"Unit" can be: one customer, one transaction, one order, one loan
```

| Component | Examples |
|-----------|---------|
| Revenue per unit | Subscription, commission, interest, ad revenue |
| Variable costs | Payment processing, delivery, compute, fraud |
| Fixed costs (allocated) | Engineering salaries / N users, rent / N |
| Contribution margin | Revenue - Variable costs (before fixed) |

> **Critical Insight:** The most common mistake is confusing contribution margin with profit. A business can have positive contribution margin but lose money if fixed costs aren't covered. Always clarify: at what scale are fixed costs spread?

### DoorDash Order Economics Example

```
Revenue:  Commission $4.50 + Delivery fee $3.99 + Service fee $2.70 = $11.19
Costs:    Dasher pay $7.50 + Processing $0.82 + Support $0.40 + Other $0.60 = $9.32
Contribution margin per order: $1.87 (16.7%)
```

---

## Customer Acquisition Cost (CAC) {#cac}

```
CAC = Total Acquisition Spending / New Customers Acquired
```

| Channel | Typical CAC | Notes |
|---------|-------------|-------|
| Organic/SEO | $10-30 | Low marginal cost, slow to build |
| Paid search | $50-200 | Scalable but competitive |
| Referral | $20-50 | High quality, limited scale |
| Enterprise sales | $5K-50K | High-touch, high LTV |

### Blended vs Marginal CAC

```python
def analyze_cac(channel_data):
    """Critical: blended vs marginal CAC."""
    blended = channel_data['spend'].sum() / channel_data['customers'].sum()
    
    # Marginal = cost of ONE MORE customer (always increasing)
    channel_data['cac'] = channel_data['spend'] / channel_data['customers']
    marginal = channel_data.sort_values('cac')['cac'].iloc[-1]
    
    print(f"Blended CAC: ${blended:.2f}")
    print(f"Marginal CAC: ${marginal:.2f}")
    # If marginal >> blended, growth at current economics is unsustainable
```

> **Critical Insight:** VCs focus on **marginal CAC** — the cost of the next customer. If blended is $50 but marginal is $150 (next channel you'd scale into), growth is unsustainable at current unit economics.

---

## Lifetime Value (LTV) {#ltv}

### Simple Formula

```
LTV = ARPU * Gross Margin * (1 / Churn Rate)

Example (streaming):
  $12.99/mo * 40% margin * (1/0.05 churn) = $103.92
```

### Cohort-Based LTV (More Accurate)

```python
def cohort_ltv(retention_curve, revenue_per_active, discount_rate=0.10, months=36):
    """Calculate LTV from actual cohort retention data."""
    monthly_discount = (1 + discount_rate) ** (1/12) - 1
    ltv = 0
    for month in range(months):
        if month < len(retention_curve):
            retention = retention_curve[month]
        else:
            # Project with exponential decay
            decay = retention_curve[-1] / retention_curve[-2]
            retention = retention_curve[-1] * decay ** (month - len(retention_curve) + 1)
        
        contribution = retention * revenue_per_active
        ltv += contribution / (1 + monthly_discount) ** month
    return ltv
```

| LTV Lever | Impact | How to Improve |
|-----------|--------|----------------|
| Retention | Exponential (halve churn = double LTV) | Engagement, product quality |
| ARPU | Linear | Upsell, cross-sell, price increases |
| Margin | Linear | Reduce COGS, automate support |
| Expansion | Multiplicative | Land-and-expand, usage pricing |

---

## LTV:CAC Ratio and Payback Period {#ltv-cac-ratio}

| LTV:CAC | Interpretation | Action |
|---------|---------------|--------|
| < 1.0 | Losing money per customer | Fix immediately |
| 1.0 - 3.0 | Marginally profitable | Optimize funnel |
| 3.0 - 5.0 | Healthy benchmark | Scale marketing |
| > 5.0 | Under-investing in growth | Spend more |

### Payback Period

```
Payback = CAC / (Monthly Revenue * Gross Margin)

SaaS example: $500 / ($99 * 0.80) = 6.3 months
Benchmarks: SaaS < 12mo (good), E-commerce < 6mo, Fintech < 18mo
```

> **Critical Insight:** LTV:CAC of 3.0 is the VC benchmark, but **payback period matters more for cash-constrained companies**. LTV:CAC=5.0 with 3-year payback needs massive capital. LTV:CAC=3.0 with 3-month payback self-funds growth.

---

## Contribution Margin and Break-Even {#contribution-margin}

```
Contribution Margin = Revenue - Variable Costs (per unit)
Break-Even Volume = Fixed Costs / Contribution Margin per Unit

Example: $500K monthly fixed / $1.87 per order = 267,380 orders/month
```

| Level | Includes | Purpose |
|-------|----------|---------|
| Gross margin | Revenue - COGS | Product profitability |
| Contribution margin | - Variable ops | Unit-level decisions |
| Operating margin | - Allocated fixed | Business health |
| Net margin | - Taxes, interest | True bottom line |

---

## NPV for Cohort Economics {#npv-cohort-economics}

For businesses with long payback (credit cards, insurance), discount future cash flows:

```python
def cohort_npv(initial_cost, monthly_contributions, annual_rate=0.12, months=60):
    """NPV of a customer cohort."""
    monthly_rate = (1 + annual_rate) ** (1/12) - 1
    npv = -initial_cost
    for m, cf in enumerate(monthly_contributions[:months]):
        npv += cf / (1 + monthly_rate) ** (m + 1)
    return npv
```

### Credit Card Example

```
Investment: $350 (marketing + sign-up bonus + card production)
Monthly revenue: Interest $45 + Interchange $30 + Annual fee $8.33 = $83.33
Monthly costs: Funds $12.50 + Fraud $2.50 + Rewards $22.50 + Losses $15 + Service $5 = $57.50
Monthly contribution: $25.83
Payback: $350 / $25.83 = 13.5 months
5-year NPV at 10%: ~$825 per cardholder
```

> **Critical Insight:** In credit businesses, the "J-curve" — heavy upfront investment, delayed revenue — means high-risk customers may NEVER reach payback. This is why card companies are so selective on approvals.

---

## Sensitivity Analysis {#sensitivity-analysis}

```python
def sensitivity_tornado(base_case, params_to_vary, metric_fn):
    """Vary each parameter +/-20%, rank by impact."""
    base = metric_fn(**base_case)
    results = {}
    for param, val in base_case.items():
        if param not in params_to_vary:
            continue
        low = metric_fn(**{**base_case, param: val * 0.8})
        high = metric_fn(**{**base_case, param: val * 1.2})
        results[param] = (high - low) / base * 100
    
    for p, s in sorted(results.items(), key=lambda x: -abs(x[1])):
        print(f"{p:25s}: {s:+.1f}% sensitivity")

def monte_carlo_case(n_sim=10000):
    """Simulate uncertain business case outcomes."""
    profits = []
    for _ in range(n_sim):
        revenue = np.random.normal(100000, 15000)
        cost = np.random.normal(80000, 10000)
        profits.append(revenue - cost)
    profits = np.array(profits)
    print(f"Expected: ${profits.mean():,.0f}, P(profitable): {(profits>0).mean()*100:.0f}%")
    print(f"5th pctile: ${np.percentile(profits, 5):,.0f}")
```

---

## Business Case Frameworks {#business-case-frameworks}

### The 4-Step Framework

```
1. DEFINE THE UNIT — What is one "unit"? What's the revenue model?
2. BUILD THE P&L — Revenue streams, variable costs, contribution margin
3. ANALYZE DYNAMICS — CAC scaling, LTV by segment, payback, levers
4. STRESS TEST — Key assumptions, sensitivity, downside risk
```

| Framework | When to Use |
|-----------|-------------|
| Revenue-Cost-Volume | "Should we launch X?" |
| Incremental Analysis | "Should we add/change X?" (only incremental costs) |
| Opportunity Cost | "X or Y?" (compare NPV including foregone revenue) |
| Portfolio Analysis | "Which segments?" (LTV:CAC by segment) |

---

## Practical Cases {#practical-cases}

### Case 1: "Should a Restaurant Partner with Groupon?"

```
Direct P&L (1000 deals at $50 meal for $25, Groupon takes 50%):
  Revenue: 1000 * $12.50 = $12,500
  Food cost: 1000 * $17 = -$17,000
  DIRECT LOSS: -$4,500

Incremental value:
  Extra items by group: 1000 * $15 * 30% margin = +$4,500
  Return visits (30% return, $45 spend, 30% margin): +$4,050
  Word of mouth: ~$2,500
  
  NET: +$6,550 (profitable as customer acquisition tool)

Key: Only works if return rate > 25% AND can handle capacity.
```

### Case 2: "Should We Launch a Vegan Burger?"

```
Revenue: 200/day * $14 = $2,800/day
Cannibalization: 40% from existing = true incremental $1,680/day
Variable costs: 200 * $3.50 * 0.60 = $420/day
Lost contribution (cannibalized): $268/day
Net daily contribution: $992/day
One-time cost: $20,000 (kitchen + training)
Payback: 20 days — LAUNCH IT.
```

### Case 3: "Credit Card Portfolio by Segment"

```
Super-prime (750+): LTV $1,200, CAC $400, LTV:CAC = 3.0x
Prime (700-749):    LTV $900,  CAC $200, LTV:CAC = 4.5x  ← BEST
Subprime (620-699): LTV $400,  CAC $150, LTV:CAC = 2.7x

Key: Prime is sweet spot — enough revolving for interest revenue,
     low enough charge-offs to be profitable.
```

### Case 4: "Streaming Subscription Pricing ($12.99 to $14.99)"

```
Elasticity analysis:
  LTV_old = $12.99 * 0.70 * (1/0.05) = $181.86
  LTV_new = $14.99 * 0.72 * (1/0.07) = $154.18  [churn rises 5%->7%]
  
  LTV DECREASED. Price increase destroys more through churn
  than gains in per-user revenue. Don't do it.

Key: If elasticity > -1 (inelastic), price increase works.
     If elasticity < -1 (elastic), grow subscribers instead.
```

---

## Structuring Business Case Interviews {#structuring-interviews}

### STAR-D Framework

```
S - SITUATION: "We're a [business] considering [decision]..."
T - TARGET: "Optimizing for revenue? Profit? What time horizon?"
A - ANALYSIS: "Key drivers are X, Y, Z. Let me build the P&L..."
R - RECOMMENDATION: "I recommend X because..."
D - DOWNSIDE: "If [assumption] wrong, worst case is..."
```

### Questions to Ask First

| Question | Why |
|----------|-----|
| Time horizon? | 1-year vs 5-year can flip the answer |
| Capacity constrained? | Changes whether demand is actually served |
| Competitive response? | Competitors may match your move |
| Reversible? | Higher stakes for irreversible decisions |
| Alternative use of capital? | Opportunity cost |

---

## Connecting Data to Dollars {#connecting-data-to-dollars}

| Data Metric | Dollar Translation |
|-------------|-------------------|
| +1% model accuracy | X fewer errors * $/error |
| -100ms latency | +Y% conversion * revenue/user |
| +5% retention | +Z months lifetime * margin |
| -2% fraud rate | Fewer chargebacks * avg loss |
| +10% engagement | More impressions * CPM |

```python
def quantify_improvement(current, improved, volume, cost_per_unit):
    """Translate model improvement to dollars."""
    current_errors = volume * (1 - current['precision'])
    improved_errors = volume * (1 - improved['precision'])
    monthly_savings = (current_errors - improved_errors) * cost_per_unit
    print(f"Annual savings: ${monthly_savings * 12:,.0f}")
```

> **Critical Insight:** The most impactful DS can say "my model saves $2.3M annually" instead of "AUC improved 0.03." Always translate metrics into revenue gained, cost saved, or risk reduced.

### The Executive Summary Formula

When presenting data science results to business stakeholders:

```
"By improving [metric] from [X] to [Y], we expect to
 [reduce/increase] [business outcome] by [$Z] per [time period],
 affecting [N] [units] at [$W] per unit."

Example:
"By improving fraud detection precision from 85% to 92%,
 we expect to reduce manual review costs by $840K annually,
 eliminating 14,000 false positive reviews at $60 per review."
```

### ROI Framing for ML Projects

| Project Cost | Required Annual Benefit | Reasoning |
|-------------|------------------------|-----------|
| $100K (1 DS, 2 months) | $200K+ | 2x return minimum to justify opportunity cost |
| $500K (team, 6 months) | $1M+ | Must beat alternative investments |
| $2M (platform build) | $4M+ | Higher bar for large investments |

---

## Interview Talking Points {#interview-talking-points}

> **"Startup has LTV:CAC of 2.5x but burning cash. What's wrong?"**
> "Payback period is likely too long. If CAC=$500 and monthly contribution=$20, payback is 25 months — funding 25 months before returns. I'd segment LTV:CAC by channel, calculate payback explicitly, and evaluate if retention improvements could accelerate it."

> **"How decide if a feature is worth building?"**
> "Incremental annual value vs development cost. Estimate users affected * per-user impact, translate to dollars, compare to engineering cost (salary * months). Apply 50% discount for uncertainty. Need 2-year payback minimum."

> **"Walk through credit card unit economics."**
> "Unit = one cardholder. Revenue: interest + interchange + fees. Costs: funds + losses + rewards + fraud + servicing. Key tension: highest revenue (revolvers) = highest risk. Prime revolvers are the sweet spot. Build cohort NPV because revenue is back-loaded while CAC is immediate."

---

## Common Mistakes {#common-mistakes}

| # | Mistake (❌) | Correct Approach (✅) |
|---|-------------|----------------------|
| 1 | ❌ Use blended CAC for growth decisions | ✅ Use marginal CAC |
| 2 | ❌ LTV without discounting future cash flows | ✅ Apply NPV (8-15% discount) |
| 3 | ❌ Ignore cannibalization | ✅ Net out lost revenue from existing products |
| 4 | ❌ Count sunk costs in incremental analysis | ✅ Only future/incremental costs |
| 5 | ❌ Use revenue when you mean contribution margin | ✅ Subtract variable costs first |
| 6 | ❌ Point estimates without uncertainty | ✅ Show sensitivity or confidence ranges |
| 7 | ❌ Optimize LTV:CAC alone | ✅ Also check payback and cash flow |
| 8 | ❌ Assume linear scaling | ✅ Account for diminishing returns |
| 9 | ❌ Ignore competitive response | ✅ Model likely competitor reactions |
| 10 | ❌ Report metrics without dollar translation | ✅ Always translate to revenue/cost impact |

---

## Rapid-Fire Q&A {#rapid-fire-qa}

**Q1: Unit economics in one sentence?**
> Whether you make or lose money on each individual customer/transaction, before scaling effects.

**Q2: Why LTV:CAC of 3x?**
> Earn $3 per $1 acquiring. Below 3x is thin (doesn't cover fixed costs + uncertainty). Above 5x = underinvesting.

**Q3: Contribution margin vs gross margin?**
> Gross = Revenue - COGS only. Contribution = Revenue - ALL variable costs (COGS + ops).

**Q4: When NPV vs simple payback?**
> NPV for long-horizon (credit, enterprise SaaS). Simple payback for quick-return (e-commerce, consumer).

**Q5: What makes Groupon work for a restaurant?**
> Return rate >25%, capacity to handle surge, ancillary revenue during visit. Fails if treated as revenue instead of acquisition.

**Q6: Break-even for a new product?**
> Fixed costs / Contribution margin per unit. Include launch costs in fixed, only incremental variable costs.

**Q7: What's the J-curve?**
> Heavy upfront investment (negative cash flow) before revenue ramps. Determines capital needed.

**Q8: Handling uncertainty in business cases?**
> Sensitivity on top 3-5 assumptions, Monte Carlo for complex cases, scenario analysis, break-even analysis.

**Q9: Incremental analysis principle?**
> Only costs/revenues that CHANGE because of a decision. Ignore sunk costs and unchanged fixed costs.

**Q10: Connect 1% accuracy improvement to dollars?**
> Map to outcomes: fewer false positives = less review cost, fewer missed frauds = less loss. Multiply unit change by dollar cost per unit.

---

## ASCII Cheat Sheet {#ascii-cheat-sheet}

```
╔════════════════════════════════════════════════════════════╗
║       UNIT ECONOMICS & BUSINESS CASE CHEAT SHEET           ║
╠════════════════════════════════════════════════════════════╣
║                                                            ║
║  CORE FORMULAS:                                            ║
║  Contribution = Revenue - Variable Costs                   ║
║  Break-Even = Fixed Costs / Contribution per Unit          ║
║  LTV = ARPU * Margin * (1/Churn)                          ║
║  CAC = Acquisition Spend / New Customers                   ║
║  Payback = CAC / (Monthly Revenue * Margin)                ║
║  NPV = Sum[CF_t / (1+r)^t]                                ║
║                                                            ║
║  LTV:CAC BENCHMARKS:                                       ║
║  <1x = losing money  |  1-3x = marginal                   ║
║  3-5x = healthy      |  >5x = underinvesting              ║
║                                                            ║
║  INTERVIEW STRUCTURE (STAR-D):                             ║
║  Situation → Target → Analysis → Recommend → Downside      ║
║                                                            ║
║  UNIT ECONOMICS WATERFALL:                                 ║
║  Revenue → (-COGS) → Gross Margin                          ║
║  → (-Variable Ops) → Contribution Margin [key metric]      ║
║  → (-Fixed/N) → Operating Profit                           ║
║                                                            ║
║  DATA TO DOLLARS:                                          ║
║  +1% accuracy = X fewer errors * $/error                   ║
║  -100ms latency = +Y% conversion * rev/user                ║
║  +5% retention = +Z months * monthly margin                ║
║                                                            ║
║  CASE CHECKLIST:                                           ║
║  [ ] Defined the "unit"                                    ║
║  [ ] Separated variable vs fixed                           ║
║  [ ] Contribution margin (not just revenue)                ║
║  [ ] Cannibalization accounted for                         ║
║  [ ] Incremental costs only (no sunk)                      ║
║  [ ] Discounted if long horizon                            ║
║  [ ] Sensitivity on key assumptions                        ║
║  [ ] Clear recommendation with risks                       ║
║                                                            ║
╚════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 60 of 60*
