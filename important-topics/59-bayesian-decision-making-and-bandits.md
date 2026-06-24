# 🎯 Topic 59: Bayesian Decision Making & Bandits

> *"Frequentist methods tell you whether to reject a hypothesis. Bayesian methods tell you what to believe — and what to do next."*

---

## Table of Contents

1. [Beta-Binomial Model for Conversion Rates](#beta-binomial-model)
2. [Posterior Updating](#posterior-updating)
3. [Thompson Sampling for Multi-Armed Bandits](#thompson-sampling)
4. [Explore-Exploit Tradeoff](#explore-exploit-tradeoff)
5. [Algorithm Comparison](#algorithm-comparison)
6. [Contextual Bandits](#contextual-bandits)
7. [Bayesian Optimization for Hyperparameters](#bayesian-optimization)
8. [When Bayesian Beats Frequentist](#when-bayesian-beats-frequentist)
9. [Bayesian A/B Testing](#bayesian-ab-testing)
10. [Practical Applications](#practical-applications)
11. [Interview Talking Points](#interview-talking-points)
12. [Common Mistakes](#common-mistakes)
13. [Rapid-Fire Q&A](#rapid-fire-qa)
14. [ASCII Cheat Sheet](#ascii-cheat-sheet)

---

## Beta-Binomial Model for Conversion Rates {#beta-binomial-model}

The Beta-Binomial is the workhorse for binary outcomes (click/no-click, convert/no-convert).

```
Likelihood:  Binomial(k | n, p)
Prior:       Beta(p | alpha, beta)
Posterior:   Beta(p | alpha + k, beta + n - k)   [conjugate!]
```

Beta is "conjugate" to Binomial — posterior is the same family as prior, giving closed-form updates.

| Prior | alpha | beta | Meaning |
|-------|-------|------|---------|
| Uniform | 1 | 1 | All rates equally likely |
| Weak | 2 | 20 | "I think ~10% but unsure" |
| Strong | 20 | 200 | "Fairly confident ~10%" |
| Jeffrey's | 0.5 | 0.5 | Non-informative |

**Interpretation:** alpha = "pseudo-successes", beta = "pseudo-failures" from prior experience.

> **Critical Insight:** Prior "strength" = alpha + beta. A Beta(1,1) with n_total=2 is easily overwhelmed by data. A Beta(100,900) requires ~1000 new observations to shift meaningfully. Choose strength based on how much historical data you want represented.

---

## Posterior Updating {#posterior-updating}

```python
import numpy as np
from scipy import stats

class BayesianConversionTracker:
    def __init__(self, alpha_prior=1, beta_prior=1):
        self.alpha = alpha_prior
        self.beta = beta_prior
    
    def update(self, successes, failures):
        self.alpha += successes
        self.beta += failures
    
    @property
    def mean(self):
        return self.alpha / (self.alpha + self.beta)
    
    @property
    def credible_interval(self):
        return (stats.beta.ppf(0.025, self.alpha, self.beta),
                stats.beta.ppf(0.975, self.alpha, self.beta))
    
    def prob_greater_than(self, threshold):
        return 1 - stats.beta.cdf(threshold, self.alpha, self.beta)
    
    def sample(self, n=10000):
        return np.random.beta(self.alpha, self.beta, size=n)
```

| Property | Formula | Implication |
|----------|---------|------------|
| Posterior mean | (alpha+k)/(alpha+beta+n) | Weighted avg of prior and data |
| Shrinkage | As n->inf, posterior->MLE | Prior washes out |
| CI width | ~1/sqrt(n) | Shrinks with more data |

---

## Thompson Sampling for Multi-Armed Bandits {#thompson-sampling}

**Problem:** K arms with unknown reward probabilities. Choose one arm per round. Maximize total reward over T rounds.

```python
class ThompsonSampling:
    """At each step: sample from posteriors, pull highest sample."""
    
    def __init__(self, n_arms):
        self.alphas = np.ones(n_arms)
        self.betas = np.ones(n_arms)
    
    def select_arm(self):
        samples = [np.random.beta(self.alphas[i], self.betas[i])
                   for i in range(len(self.alphas))]
        return np.argmax(samples)
    
    def update(self, arm, reward):
        if reward == 1:
            self.alphas[arm] += 1
        else:
            self.betas[arm] += 1

def simulate(true_rates, n_rounds=10000):
    ts = ThompsonSampling(len(true_rates))
    best = max(true_rates)
    regret = 0
    for _ in range(n_rounds):
        arm = ts.select_arm()
        reward = np.random.binomial(1, true_rates[arm])
        ts.update(arm, reward)
        regret += best - true_rates[arm]
    return regret
```

> **Critical Insight:** Thompson Sampling is **probability matching** — pulls each arm proportional to probability it's optimal. Naturally balances exploration (uncertain arms have wide posteriors, sometimes sample high) and exploitation (good arms usually sample high). Near-optimal regret.

---

## Explore-Exploit Tradeoff {#explore-exploit-tradeoff}

```
EXPLORATION: Try uncertain options to learn → reduces long-term regret
EXPLOITATION: Pull best-known option → maximizes immediate reward
```

| Algorithm | Regret | Interpretation |
|-----------|--------|----------------|
| Random | O(T) | Linear — terrible |
| Greedy | O(T) worst case | Gets stuck on suboptimal |
| Epsilon-greedy | O(T) | Fixed exploration waste |
| UCB1 | O(K * log T) | Logarithmic — good |
| Thompson | O(sqrt(K*T*log T)) | Near-optimal |

---

## Algorithm Comparison {#algorithm-comparison}

| Feature | Epsilon-Greedy | UCB1 | Thompson Sampling |
|---------|---------------|------|-------------------|
| Mechanism | Random explore with prob epsilon | Optimism under uncertainty | Posterior sampling |
| Exploration | Fixed rate | Decreases with confidence | Automatic via posterior |
| Parameters | epsilon (needs tuning) | None | Prior (usually uninformative) |
| Adapts? | No | Yes | Yes |
| Contextual? | Easy extension | LinUCB | Contextual Thompson |
| Best for | Quick prototypes | Worst-case guarantees | Production systems |

```python
class EpsilonGreedy:
    def __init__(self, n_arms, epsilon=0.1):
        self.epsilon = epsilon
        self.counts = np.zeros(n_arms)
        self.rewards = np.zeros(n_arms)
    
    def select_arm(self):
        if np.random.random() < self.epsilon:
            return np.random.randint(len(self.counts))
        return np.argmax(self.rewards / np.maximum(self.counts, 1))

class UCB1:
    def __init__(self, n_arms):
        self.counts = np.zeros(n_arms)
        self.rewards = np.zeros(n_arms)
        self.t = 0
    
    def select_arm(self):
        self.t += 1
        if 0 in self.counts:
            return np.where(self.counts == 0)[0][0]
        means = self.rewards / self.counts
        ucb = means + np.sqrt(2 * np.log(self.t) / self.counts)
        return np.argmax(ucb)
```

> **Critical Insight:** Thompson Sampling dominates in practice: (1) no tuning needed, (2) easily extends to contextual, (3) works in batch mode, (4) posteriors give interpretable uncertainty. UCB preferred for worst-case guarantees.

---

## Contextual Bandits {#contextual-bandits}

Incorporates user features for personalized arm selection:

```python
class LinearThompsonSampling:
    """E[reward | context x, arm a] = x^T * theta_a"""
    
    def __init__(self, n_arms, n_features, lambda_reg=1.0):
        self.B = [lambda_reg * np.eye(n_features) for _ in range(n_arms)]
        self.f = [np.zeros(n_features) for _ in range(n_arms)]
    
    def select_arm(self, context):
        x = np.array(context)
        best_reward, best_arm = -np.inf, 0
        for a in range(len(self.B)):
            B_inv = np.linalg.inv(self.B[a])
            mu = B_inv @ self.f[a]
            theta = np.random.multivariate_normal(mu, B_inv)
            reward = x @ theta
            if reward > best_reward:
                best_reward, best_arm = reward, a
        return best_arm
    
    def update(self, arm, context, reward):
        x = np.array(context)
        self.B[arm] += np.outer(x, x)
        self.f[arm] += reward * x
```

**Use case:** Ad personalization — context = user features, arms = ad creatives, reward = click.

---

## Bayesian Optimization for Hyperparameters {#bayesian-optimization}

Uses Gaussian Process surrogate to efficiently search parameter spaces:

```
1. Evaluate f(x) at random points
2. Fit GP posterior over f(x)
3. Find x_next maximizing acquisition function
4. Evaluate f(x_next), update GP, repeat
```

| Acquisition Function | Behavior |
|---------------------|----------|
| Expected Improvement (EI) | Balanced — good default |
| GP-UCB | Optimistic exploration |
| Probability of Improvement | Exploitation-heavy |

```python
from sklearn.gaussian_process import GaussianProcessRegressor
from sklearn.gaussian_process.kernels import Matern

class BayesianOptimizer:
    def __init__(self, bounds):
        self.bounds = np.array(bounds)
        self.gp = GaussianProcessRegressor(kernel=Matern(nu=2.5))
        self.X_obs, self.y_obs = [], []
    
    def expected_improvement(self, X, y_best):
        mu, sigma = self.gp.predict(X.reshape(-1, len(self.bounds)), return_std=True)
        Z = (mu - y_best) / np.maximum(sigma, 1e-8)
        return (mu - y_best) * stats.norm.cdf(Z) + sigma * stats.norm.pdf(Z)
    
    def suggest_next(self):
        if len(self.X_obs) < 5:
            return np.random.uniform(self.bounds[:, 0], self.bounds[:, 1])
        self.gp.fit(np.array(self.X_obs), np.array(self.y_obs))
        # Optimize EI via random restarts
        best_x, best_ei = None, -1
        for _ in range(100):
            x = np.random.uniform(self.bounds[:, 0], self.bounds[:, 1])
            ei = self.expected_improvement(x, max(self.y_obs))
            if ei > best_ei:
                best_ei, best_x = ei, x
        return best_x
```

> **Critical Insight:** BO is ideal when evaluations are expensive, space is low-dimensional (<20 params), and you need solutions in <100 evaluations. For high-dimensional spaces, random search or Hyperband is more practical.

---

## When Bayesian Beats Frequentist {#when-bayesian-beats-frequentist}

| Condition | Bayesian Advantage |
|-----------|-------------------|
| Small samples | Prior prevents overfitting |
| Sequential decisions | Natural updating, no peeking problem |
| Prior knowledge | Incorporates historical data formally |
| Decision-oriented | Direct P(A>B), not p-values |
| Multi-armed | Thompson > Bonferroni correction |
| Stopping rules | Stop when posterior concentrated |
| Communication | Credible intervals more intuitive |

| Aspect | Frequentist | Bayesian |
|--------|-------------|----------|
| Output | p-value, CI | Posterior, P(A>B) |
| Peeking | Invalid (inflates FPR) | Valid anytime |
| Stopping | Fixed N or sequential bounds | When posterior concentrated |
| Prior info | Cannot use | Formally incorporated |
| Interpretation | "Reject or fail to reject" | "Probability of improvement = X%" |
| Multiple comparisons | Requires Bonferroni/FDR correction | Hierarchical models handle naturally |
| Small data | Wide CIs, often "not significant" | Prior stabilizes estimates |

### Practical Example: When Prior Matters

```python
# New feature, similar to one tested last quarter (8% conversion, n=5000)
# Frequentist: starts from scratch, needs full sample size again
# Bayesian: use informative prior Beta(400, 4600) encoding prior knowledge

# With only 100 new observations (6 conversions):
# Frequentist CI: [2.2%, 12.6%] — too wide to decide
# Bayesian posterior with prior: [6.8%, 8.9%] — useful for decision
```

> **Critical Insight:** The Bayesian advantage is most dramatic in early stages of data collection and when you have relevant prior information. As sample size grows large, Bayesian and frequentist results converge — the prior gets "washed out." Choose Bayesian when decisions must be made quickly with limited data.

---

## Bayesian A/B Testing {#bayesian-ab-testing}

```python
def bayesian_ab_test(ctrl_conv, ctrl_total, treat_conv, treat_total,
                     prior_a=1, prior_b=1, n_sim=100000):
    """Complete Bayesian A/B analysis."""
    # Posteriors
    a_c, b_c = prior_a + ctrl_conv, prior_b + ctrl_total - ctrl_conv
    a_t, b_t = prior_a + treat_conv, prior_b + treat_total - treat_conv
    
    # Sample
    ctrl_samples = np.random.beta(a_c, b_c, n_sim)
    treat_samples = np.random.beta(a_t, b_t, n_sim)
    diff = treat_samples - ctrl_samples
    lift = diff / ctrl_samples
    
    return {
        'prob_better': (diff > 0).mean(),
        'expected_lift': lift.mean(),
        'lift_95_ci': (np.percentile(lift, 2.5), np.percentile(lift, 97.5)),
        'expected_loss': np.maximum(-diff, 0).mean(),  # Risk if wrong
    }

# Decision rule: ship when P(better) > 0.95 AND expected_loss < threshold
```

> **Critical Insight:** The "expected loss" framework is superior to P(A>B) alone. A variant with P(better)=0.85 but loss=0.0001 is safe to ship (tiny downside). Conversely, P(better)=0.96 might not suffice if expected loss is catastrophic.

---

## Practical Applications {#practical-applications}

| Application | Arms | Context | Reward | Update |
|-------------|------|---------|--------|--------|
| **Ad selection** | 5 creatives | User demographics | Click | Real-time |
| **Recommendations** | Content items | User embedding | Engagement | Hourly batch |
| **Dynamic pricing** | Price points | Demand signals | Revenue | Daily |
| **Feature rollout** | New vs old | User segment | Conversion | Daily |

### Ad Selection (Thompson Sampling)

Quickly converges to the best ad per user context while minimizing wasted impressions on poor creatives. With 5 ads, Thompson typically identifies the winner within 200-500 impressions, far faster than sequential A/B testing each pair.

### Dynamic Pricing

Arms are discrete price points. Reward is **revenue** (price * purchase_probability), not just conversion. Higher prices have lower conversion but potentially higher per-unit revenue. Update daily since prices shouldn't change too rapidly (trust considerations).

### Feature Rollout (Bayesian A/B)

Monitor P(new > old) daily. Ship when expected loss < acceptable threshold. Pause immediately if P(harm > MDE) exceeds 50%. Advantage over frequentist: can peek without inflating error rates and stop early when evidence is overwhelming in either direction.

### Batch Thompson Sampling (Production Pattern)

In production, you often cannot update after every single impression. Instead, sample parameters once, serve a batch of users, then update:

```python
class BatchThompsonSampling:
    """Production pattern: allocate traffic in batches."""
    
    def __init__(self, n_arms):
        self.alphas = np.ones(n_arms)
        self.betas = np.ones(n_arms)
    
    def compute_allocation(self, n_simulations=10000, min_traffic=0.05):
        """Compute traffic split for next batch via repeated sampling."""
        win_counts = np.zeros(len(self.alphas))
        for _ in range(n_simulations):
            samples = [np.random.beta(self.alphas[i], self.betas[i])
                       for i in range(len(self.alphas))]
            win_counts[np.argmax(samples)] += 1
        
        allocation = win_counts / n_simulations
        allocation = np.maximum(allocation, min_traffic)  # Floor
        return allocation / allocation.sum()
    
    def batch_update(self, successes_per_arm, failures_per_arm):
        """Update with accumulated batch results."""
        self.alphas += successes_per_arm
        self.betas += failures_per_arm
```

---

## Interview Talking Points {#interview-talking-points}

> **"When Thompson Sampling instead of standard A/B?"**
> "When exploration cost is high and I need to minimize regret during learning. A/B serves a bad variant to 50% for weeks. Thompson shifts traffic toward the winner automatically. Best for: many variants, can't wait weeks per test, user experience degrades with bad options."

> **"Explain Beta-Binomial to a PM."**
> "It's a belief system for rates. Start with a prior ('probably around 5%'). Each observation updates the belief. After 1000 observations, belief concentrates around truth. We can always answer 'what's the probability rate is above X?' directly — no confusing p-values."

> **"How handle peeking in A/B tests?"**
> "Frequentist: peeking inflates false positives. Bayesian: posterior is always valid — represents 'what we believe given all data so far.' I can check daily. Trade-off: need a pre-defined decision threshold to avoid running forever."

---

## Common Mistakes {#common-mistakes}

| # | Mistake (❌) | Correct Approach (✅) |
|---|-------------|----------------------|
| 1 | ❌ Flat prior when historical data exists | ✅ Set prior from previous experiments |
| 2 | ❌ Interpret credible interval as frequentist CI | ✅ CI = "95% probability parameter is in range" |
| 3 | ❌ Thompson Sampling with delayed rewards | ✅ Batch updates after rewards materialize |
| 4 | ❌ Optimize P(A>B) alone | ✅ Also check expected loss |
| 5 | ❌ No minimum exploration floor | ✅ 5-10% minimum per arm prevents premature lock-in |
| 6 | ❌ BO for >20 dimensions | ✅ Use random search or Hyperband |
| 7 | ❌ Ship on single posterior sample | ✅ Use P(better) from many samples |
| 8 | ❌ Ignore non-stationarity | ✅ Discounted TS or sliding window for changing rates |

---

## Rapid-Fire Q&A {#rapid-fire-qa}

**Q1: Why Beta for conversion rates?**
> Conjugate to Binomial (closed-form updates), defined on [0,1] matching valid probability range.

**Q2: What does Thompson Sampling do each step?**
> Sample from each arm's posterior, pull arm with highest sample. Naturally balances explore/exploit.

**Q3: Thompson Sampling regret?**
> O(sqrt(K*T*log T)) — near-optimal. Converges quickly for 2-5 arms.

**Q4: Credible interval vs confidence interval?**
> Credible: "95% probability parameter is here." Confidence: "95% of repeated intervals contain truth." Bayesian is more natural for decisions.

**Q5: When UCB beats Thompson?**
> Worst-case guarantees needed. Adversarial settings. O(K*log T) bound regardless of prior.

**Q6: Explore-exploit in simple terms?**
> Trying new things might find better options but wastes time. Sticking with best-known maximizes now but might miss better. Balance depends on time remaining.

**Q7: Bayesian optimization vs grid/random search?**
> BO uses GP surrogate to predict promising points. Finds good solutions in 10-50 evaluations vs hundreds.

**Q8: Can you peek at Bayesian A/B?**
> Yes — posterior always valid. But need pre-specified decision rule to avoid running forever.

**Q9: What's a contextual bandit?**
> Observes user features before choosing arm. Learns best arm per user type. Enables personalization.

**Q10: Expected loss criterion?**
> E[max(control - treatment, 0)] = average downside if wrong. Ship when this is acceptably small.

---

## ASCII Cheat Sheet {#ascii-cheat-sheet}

```
╔════════════════════════════════════════════════════════════╗
║        BAYESIAN DECISION MAKING & BANDITS                  ║
╠════════════════════════════════════════════════════════════╣
║                                                            ║
║  BETA-BINOMIAL:                                            ║
║  Prior: Beta(a, b) + Data: k success, n-k fail            ║
║  Posterior: Beta(a+k, b+n-k)                              ║
║  Mean = (a+k)/(a+b+n), Strength = a+b                     ║
║                                                            ║
║  THOMPSON SAMPLING:                                        ║
║  1. Sample theta_i ~ Beta(a_i, b_i) for each arm          ║
║  2. Pull arm with highest theta_i                          ║
║  3. Observe reward, update a_i or b_i                      ║
║                                                            ║
║  ALGORITHM CHOICE:                                         ║
║  Simple/explainable  → Epsilon-Greedy                      ║
║  Worst-case bounds   → UCB1                                ║
║  Best empirical      → Thompson Sampling                   ║
║  With features       → Contextual Thompson                 ║
║  Expensive evals     → Bayesian Optimization               ║
║                                                            ║
║  BAYESIAN A/B DECISION:                                    ║
║  Ship if: P(treat > ctrl) > 0.95                           ║
║    AND: Expected_loss < threshold                          ║
║  Stop if: P(harm > MDE) > 0.95                             ║
║                                                            ║
║  WHEN BAYESIAN WINS:                                       ║
║  [x] Small samples    [x] Sequential decisions             ║
║  [x] Prior available  [x] Need P(X better)                 ║
║  [x] Many variants    [x] Natural stopping                 ║
║                                                            ║
║  REGRET: Random O(T) > EpsGreedy O(T) >                   ║
║          UCB O(KlogT) > Thompson O(sqrt(KT logT))          ║
║                                                            ║
╚════════════════════════════════════════════════════════════╝
```

---

*Part of the [Data Science Interview Topics](README.md) collection — Topic 59 of 60*
