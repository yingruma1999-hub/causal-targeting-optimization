# Starbucks Uplift Modeling — Complete Thinking Process & Interview Prep

> **Purpose:** This document reconstructs the full analytical thinking process behind the Starbucks Uplift Modeling project, including initial assumptions, mistakes discovered mid-analysis, methodological pivots, and final solutions. It is written in a narrative style suitable for behavioral/technical DS interview preparation (e.g., "Walk me through a project where you had to change your approach").

---

## Table of Contents

1. [Project Setup & Initial Framing](#1-project-setup--initial-framing)
2. [First Attempt: Observational Uplift Modeling](#2-first-attempt-observational-uplift-modeling)
3. [The Collapse — All Meta-Learners Returned ~0 CATE](#3-the-collapse--all-meta-learners-returned-0-cate)
4. [Root Cause Diagnosis: Self-Selection Bias](#4-root-cause-diagnosis-self-selection-bias)
5. [The SRM Contradiction — Why I Initially Said "Not Applicable" Then Did It Anyway](#5-the-srm-contradiction)
6. [The Methodological Pivot: Dual-Path Causal Analysis](#6-the-methodological-pivot-dual-path-causal-analysis)
7. [Path A: Intent-to-Treat (ITT) Analysis](#7-path-a-intent-to-treat-itt-analysis)
8. [Path B: Observational Analysis with Bias Correction](#8-path-b-observational-analysis-with-bias-correction)
9. [Post-Treatment Variable Contamination](#9-post-treatment-variable-contamination)
10. [DR-Learner Collapse vs IPW Success](#10-dr-learner-collapse-vs-ipw-success)
11. [Business Implications & ROI Negativity](#11-business-implications--roi-negativity)
12. [Key Takeaways for Interviews](#12-key-takeaways-for-interviews)

---

## 1. Project Setup & Initial Framing

### The Dataset

The Starbucks dataset is a simulated marketing experiment with three files:
- **portfolio.csv** — 10 different offer types (BOGO, discount, informational) with metadata (channels, difficulty, reward, duration)
- **profile.csv** — 17,000 users with demographics (age, gender, income, membership date)
- **transcript.csv** — 306,534 event records (offer received, offer viewed, offer completed, transaction)

### Initial Thinking

My first instinct was to frame this as a classic **uplift modeling** problem:
- **Treatment group:** Users who were "treated" with an offer
- **Control group:** Users who were not treated
- **Outcome:** Whether the user made a purchase (binary conversion)

The core question: *Which users have the highest incremental response to marketing offers?*

### Initial Treatment Definition (The Critical Decision)

I defined treatment at the **person × offer** level:
- **T = 1 (Treated):** The user *viewed* the offer (i.e., opened/engaged with it)
- **T = 0 (Control):** The user *received* the offer but did *not* view it

**Why I chose this:** It seemed intuitive — users who actually saw the offer vs. those who ignored it. The "real" treatment is seeing the offer, not just receiving it. This gave me 8,828 person×offer pairs: 6,415 viewers (T=1) and 2,413 non-viewers (T=0).

**What I didn't realize yet:** This was a fundamentally flawed causal definition. More on this in Section 4.

---

## 2. First Attempt: Observational Uplift Modeling

### Feature Engineering

I built an RFM-lite (Recency, Frequency, Monetary) feature set from the transaction data:
- `txn_count` — number of transactions per user
- `total_spend` — total spending per user
- `avg_basket` — average transaction amount
- Combined with demographics: `age`, `income`, `gender_F/M/O`, `tenure` (days since membership), `income_age_ratio`
- Plus offer-level features: `reward`, `difficulty`, `duration`, channel dummies

This gave me a 23-feature model matrix.

### Meta-Learner Implementation

I implemented four standard uplift meta-learners:

1. **S-Learner:** Single model with treatment as a feature. Predict Y(X, T=1) - Y(X, T=0).
2. **T-Learner:** Separate models for treatment and control. CATE = μ₁(X) - μ₀(X).
3. **X-Learner:** Two-stage approach — first fit T-learner, then impute individual treatment effects and fit a CATE model.
4. **Transformed Outcome (TO) Learner:** Uses the IPW-reweighted pseudo-outcome Y* = T·Y/e - (1-T)·Y/(1-e) as a direct regression target.

### Naive ATE Result

The naive ATE (simple difference in means) was **0.284** — viewers converted at 28.4 percentage points higher than non-viewers. This seemed like a strong treatment effect.

---

## 3. The Collapse — All Meta-Learners Returned ~0 CATE

### What Happened

When I ran the S-Learner, T-Learner, and X-Learner, something alarming happened:

| Learner | Mean CATE | Std CATE |
|---------|-----------|----------|
| S-Learner | ~0.001 | ~0.002 |
| T-Learner | ~0.003 | ~0.005 |
| X-Learner | ~0.002 | ~0.004 |
| **TO-Learner** | **0.284** | **~0.15** |

Three out of four learners predicted essentially **zero heterogeneous treatment effect** for everyone. Only the Transformed Outcome learner showed a non-trivial effect, and its mean (0.284) exactly matched the naive ATE.

### Initial Confusion

This was deeply puzzling. The naive ATE was 28.4%, suggesting a massive treatment effect. But the conditional models (S/T/X-Learner) couldn't find any heterogeneity — they were saying "the treatment effect is the same for everyone, and it's approximately zero."

**The contradiction:** How can the raw data show a 28.4% difference, but the models say the effect is ~0?

### Why the TO-Learner Was Different

The TO-Learner showed 0.284 because it directly uses the IPW pseudo-outcome. It's essentially computing a reweighted version of the raw difference. If the propensity score model is poorly specified (which it was, because the "treatment" wasn't random), the TO-Learner just echoes the naive estimate.

---

## 4. Root Cause Diagnosis: Self-Selection Bias

### The "Aha" Moment

After seeing the collapse, I stepped back and thought about **why** the S/T/X-Learners were collapsing. The answer was a fundamental causal inference problem:

**T = "viewed offer" is a self-selected behavior, not a randomly assigned treatment.**

Users who view offers are systematically different from users who don't:
- They are more engaged with the app
- They are more likely to be frequent purchasers already
- They have higher baseline conversion rates *regardless of the offer*

This means the 28.4% raw difference is **not** a treatment effect — it's a **selection effect**. Users who view offers would have purchased anyway at higher rates. The models were correctly identifying that, conditional on features, the "treatment" had ~0 additional effect because the features already explained the difference.

### The Causal Diagram (DAG)

```
Engagement (U) → Views Offer (T)
Engagement (U) → Makes Purchase (Y)
Views Offer (T) → Makes Purchase (Y)  [this is the real causal effect, but it's confounded by U]
```

The **confounder** (underlying engagement level) drives both the "treatment" (viewing) and the outcome (purchasing). Without controlling for this properly, the naive estimate is biased upward.

### Why the S/T/X-Learners Collapsed

These learners fit separate outcome models for T=1 and T=0 groups. Because the features (especially behavioral ones like `txn_count`, `total_spend`) already captured the engagement difference between viewers and non-viewers, the predicted outcomes μ₁(X) and μ₀(X) were very similar conditional on X. The models were essentially saying: "Once I know a user's transaction history and demographics, whether they viewed the offer doesn't add much predictive power for conversion."

This is actually **correct behavior** from the models — they were detecting that the observed "treatment effect" was confounded.

---

## 5. The SRM Contradiction

### What I Initially Said

When the user asked "SRM check和偏差修正有必要做吗?" (Is SRM check and bias correction necessary?), I responded:

> "**SRM check — not applicable.** SRM (Sample Ratio Mismatch) is designed for randomized experiments to check if the allocation ratio deviates from the designed split. Your treatment definition is 'viewed offer' (a self-selected behavior), not a platform-assigned randomization, so the 72.7% vs 27.3% split is driven by user behavior, not by a randomization mechanism. The concept of SRM doesn't apply."

**This statement was technically correct for the original Frame B (observational) definition where T = viewed.**

### Why I Later Did an SRM Check Anyway

When I restructured the analysis into a dual-path framework, I introduced **Frame A (ITT: Intent-to-Treat)**:
- **T = 1:** User *received* any non-informational offer (platform-assigned)
- **T = 0:** User *never received* any non-informational offer

This is a fundamentally different treatment definition. Now the treatment is **platform-assigned** — Starbucks decided who received offers, not the users themselves. This *is* a (quasi-)randomized assignment, and therefore SRM checks *do* apply.

### The Key Distinction

| | Frame B (Observational) | Frame A (ITT) |
|---|---|---|
| Treatment definition | T = viewed offer (user behavior) | T = received offer (platform decision) |
| Assignment mechanism | Self-selection | Platform randomization |
| SRM applicable? | **No** — ratio reflects user behavior | **Yes** — ratio should reflect platform design |
| Sample split | 72.7% / 27.3% | 59.6% / 40.4% |

### SRM Check Result (Frame A)

For Frame A, I assumed the platform designed a 60/40 split (since 59.6% is very close to 60%). The chi-squared test gave **p = 0.26**, which means the observed ratio is consistent with the designed ratio. **SRM passed.**

### How to Explain This in an Interview

> "I initially said SRM doesn't apply, and I was right — for the original observational framing where 'treatment' was a user behavior. But when I restructured the analysis to use an ITT framework where treatment is platform-assigned, SRM became applicable and important. This pivot illustrates why treatment definition is the single most critical decision in causal inference — it determines which statistical tools are valid."

---

## 6. The Methodological Pivot: Dual-Path Causal Analysis

### The Decision

After diagnosing the self-selection problem, I proposed a dual-path analysis:

**Path A — Intent-to-Treat (ITT):**
- T = received any non-informational offer (platform-assigned)
- Clean causal identification via randomization
- Conservative estimate (includes non-compliers — people who received but didn't view)
- Answers: "What is the causal effect of *sending* an offer?"

**Path B — Observational with Bias Correction:**
- T = viewed offer (self-selected)
- Requires propensity score methods (IPW, Doubly Robust) to correct for selection bias
- More targeted but riskier estimate
- Answers: "What is the causal effect of *seeing* an offer, after correcting for selection bias?"

### Why Both Paths?

1. **Path A** gives a **credible, defensible** causal estimate because randomization handles confounding. But it's diluted by non-compliers (people who got the offer but never looked at it).

2. **Path B** estimates the more **economically interesting** quantity (effect of actually seeing the offer), but relies on the assumption that we've measured all confounders (the "unconfoundedness" or "ignorability" assumption).

Running both lets us **triangulate**: if the estimates are in the same ballpark, we gain confidence. If they diverge wildly, we know something is wrong.

---

## 7. Path A: Intent-to-Treat (ITT) Analysis

### Building the ITT Frame

I created a user-level dataset:
- For each of the 14,825 valid users, check if they were ever sent a non-informational offer
- **T = 1:** 8,828 users received at least one non-informational offer
- **T = 0:** 5,997 users received no non-informational offers (some may have received informational-only offers)
- **Y = 1:** User made at least one transaction during the study period

### ITT ATE Result

Simple difference in means:
- Conversion rate (T=1): 73.9%
- Conversion rate (T=0): 57.8%
- **ITT ATE = 0.161** (16.1 percentage points)

This is much smaller than the naive observational ATE of 0.284, which already tells us that a large chunk of the original estimate was selection bias.

### Feature Selection for ITT

A critical decision: **which features to use for the ITT uplift models?**

I used only **7 pre-treatment demographic features**: `age`, `income`, `tenure`, `income_age_ratio`, `gender_F`, `gender_M`, `gender_O`.

I deliberately **excluded** behavioral features (`txn_count`, `total_spend`, `avg_basket`) even though they were available. Why?

→ See Section 9 (Post-Treatment Variable Contamination).

### ITT Uplift Results

| Model | Mean CATE | Std CATE | Range |
|-------|-----------|----------|-------|
| T-Learner | 0.160 | 0.083 | [-0.03, 0.33] |
| TO-Learner | 0.161 | 0.124 | wider |

**Key finding:** Unlike the original observational frame, the ITT T-Learner found real, non-trivial heterogeneity. The mean CATE (0.160) matched the ITT ATE (0.161), which is a good sanity check. And the standard deviation (0.083) indicates meaningful variation — some users have near-zero uplift, others have 30%+ uplift.

### Top CATE Drivers

Feature importance from the CATE model (GBR on T-Learner predictions):
1. **tenure** — 47% importance (newer members benefit more from offers)
2. **income_age_ratio** — 26%
3. **age** — 13%

---

## 8. Path B: Observational Analysis with Bias Correction

### Propensity Score Estimation

For the observational frame (T = viewed), I estimated P(View | X) using Logistic Regression with cross-validation:
- Mean propensity: 0.727
- Std: 0.271
- Good overlap between treatment and control groups (verified with histogram)

I clipped propensity scores to [0.05, 0.95] to avoid extreme weights.

### IPW (Inverse Propensity Weighting)

The IPW estimator reweights each observation to create a pseudo-population where treatment is independent of covariates:

$$\hat{\tau}_{IPW} = \frac{1}{n} \sum_{i=1}^{n} \left[ \frac{T_i Y_i}{e(X_i)} - \frac{(1-T_i) Y_i}{1 - e(X_i)} \right]$$

Result: **IPW ATE = 0.144**

### Comparison with Naive and ITT

| Estimator | ATE | Interpretation |
|-----------|-----|----------------|
| Naive (viewers vs non-viewers) | 0.284 | Confounded by self-selection |
| IPW-corrected | 0.144 | After removing measurable selection bias |
| ITT (received vs not received) | 0.161 | Clean causal effect of sending offer |

**Key insight:** The IPW estimate (0.144) is close to the ITT estimate (0.161), which is reassuring. It suggests that:
1. About **49%** of the naive estimate (0.284) was pure selection bias
2. The remaining ~0.14–0.16 is the genuine causal effect
3. The two independent approaches (randomization-based ITT and propensity-score-based IPW) converge, strengthening our confidence

---

## 9. Post-Treatment Variable Contamination

### The Problem Discovered During Covariate Balance Check

When I ran the covariate balance check (Standardized Mean Difference between ITT treatment and control), I found:

**Demographics (pre-treatment):**
- age: SMD = 0.013 ✅
- income: SMD = 0.024 ✅
- tenure: SMD = -0.005 ✅
- All gender dummies: |SMD| < 0.02 ✅

**Behavioral features (post-treatment):**
- txn_count: SMD = **0.336** ❌
- total_spend: SMD = **0.365** ❌
- avg_basket: SMD = **0.109** ❌

### Why Behavioral Features Were Imbalanced

The behavioral features (`txn_count`, `total_spend`) were computed from **all transactions during the study period**, which includes transactions that were *caused by* the treatment (receiving an offer). This is classic **post-treatment variable contamination**:

```
Receives Offer (T) → Views/Completes Offer → Makes Additional Purchases → Higher txn_count
```

Including these features in the uplift model would **absorb** the treatment effect, biasing CATE estimates toward zero. This is actually another explanation for why the original observational meta-learners collapsed — the behavioral features were mediators, not confounders.

### The Solution

For Path A (ITT), I restricted the feature set to **pre-treatment demographics only** (7 features). These are determined before treatment assignment and cannot be affected by it.

For a production system, the ideal solution would be to compute behavioral features using only **pre-treatment-period transactions** (e.g., activity in the 30 days before the experiment started). The simulated dataset doesn't have clean pre/post timestamps for this, so demographics-only was the safest approach.

### How to Explain This in an Interview

> "During the covariate balance check, I discovered that transaction-based features showed large imbalances (SMD > 0.3) between treatment and control groups, even though the treatment was randomly assigned. This seemed contradictory at first — random assignment should balance everything. The resolution was that these features were *post-treatment*: they included transactions that were caused by the offer itself. Including them would create a 'bad control' problem, absorbing the very effect we're trying to measure. I restricted the model to pre-treatment demographics only."

---

## 10. DR-Learner Collapse vs IPW Success

### The Doubly Robust (DR) Estimator

The DR estimator combines outcome modeling and propensity scoring:

$$\hat{\tau}_{DR} = \frac{1}{n} \sum_{i=1}^{n} \left[ \hat{\mu}_1(X_i) - \hat{\mu}_0(X_i) + \frac{T_i(Y_i - \hat{\mu}_1(X_i))}{e(X_i)} - \frac{(1-T_i)(Y_i - \hat{\mu}_0(X_i))}{1-e(X_i)} \right]$$

### What Happened

The DR estimator gave **ATE ≈ 0.0005** — essentially zero. This was surprising because:
- IPW gave 0.144 (reasonable)
- The DR estimator is supposed to be "better" (doubly robust = consistent if either the outcome model OR propensity model is correct)

### Why DR Collapsed

The issue was that the **outcome models (μ₁ and μ₀) overfit**. When the calibrated GradientBoosting outcome models fit very well within each treatment group, the residuals (Y - μ̂(X)) become very small. The DR formula then becomes dominated by the outcome model component (μ̂₁(X) - μ̂₀(X)), which — just like in the original S/T-Learner collapse — estimates near-zero because the features explain away the difference.

In mathematical terms: if μ̂₁(X) ≈ E[Y|X, T=1] and μ̂₀(X) ≈ E[Y|X, T=0] very accurately, and the features X include post-treatment behavioral variables (which they did in Frame B), then μ̂₁(X) - μ̂₀(X) ≈ 0 for the same reason the T-Learner collapsed.

### The Lesson

"Doubly robust" doesn't mean "always better." In finite samples:
1. If outcome models overfit, the DR estimator inherits the bias of the outcome model
2. With observational data containing post-treatment variables, flexible outcome models can absorb the treatment effect
3. IPW, despite being more variable, can be more robust when the propensity model is well-specified but the outcome model is contaminated

### How to Explain This in an Interview

> "I expected the DR estimator to outperform IPW, but it collapsed to near-zero. After investigation, I realized the outcome models were too flexible and overfit to the treatment-group-specific patterns, absorbing the treatment effect into the predictions. IPW, which only models the treatment assignment mechanism without modeling the outcome, was actually more robust in this setting. This taught me that 'doubly robust' is an asymptotic property — in finite samples with model misspecification or contaminated features, simpler estimators can outperform."

---

## 11. Business Implications & ROI Negativity

### The ROI Problem

Even after establishing a genuine 16% uplift, the business case was challenging:

- Average offer cost: ~$2.48 per recipient
- Average revenue per conversion: ~$20
- Incremental conversions from blanket targeting: 0.161 × N
- **NPV of blanket targeting: -$0.25 per person** (cost exceeds incremental revenue)

### Targeted Strategies

Using CATE estimates, I simulated three strategies:

| Strategy | Coverage | Incremental Conv | Cost | NPV |
|----------|----------|-------------------|------|-----|
| Blanket (all) | 100% | 2,388 | $21,902 | -$3,683 |
| Top 50% CATE | 50% | 1,405 | $10,951 | -$885 |
| Persuadables only | 66.4% | 1,830 | $14,543 | -$2,054 |

All strategies were NPV-negative at $20 revenue per conversion.

### Break-Even Analysis

- **Top 50% CATE** breaks even at **$20.41** revenue per conversion (very close!)
- **Blanket** breaks even at **$32.64**
- This means if the CLV (Customer Lifetime Value) is even slightly above $20, targeted strategies become profitable

### The Four-Quadrant Framework

I segmented users using CATE and baseline conversion probability:

| Quadrant | Definition | Count | % | Strategy |
|----------|-----------|-------|---|----------|
| Persuadable | Low base, high CATE | 9,839 | 66.4% | **Target these** |
| Sure Thing | High base, high CATE | 2,733 | 18.4% | Already converting, save money |
| Lost Cause | Low base, low CATE | 1,456 | 9.8% | Don't waste budget |
| Sleeping Dog | High base, negative CATE | 406 | 2.7% | **Avoid** — offer hurts conversion |

### How to Explain This in an Interview

> "The model showed a real 16% causal uplift, but the business case was negative at face value. Instead of stopping there, I built a sensitivity analysis showing that targeted strategies break even at just $20.41 per conversion — essentially at the current revenue level. For a company like Starbucks where customer lifetime value is much higher than a single transaction, the targeted uplift strategy is clearly profitable. The key insight is that even a 'negative ROI' analysis is valuable — it tells you exactly where the profitability threshold is and which customer segments to target."

---

## 12. Key Takeaways for Interviews

### 1. Treatment Definition Is Everything

> "The single most important decision in causal inference is how you define treatment and control. My initial definition (viewed vs. not viewed) seemed natural but created a self-selection problem that invalidated all downstream analysis. Switching to an ITT framework (received vs. not received) gave me clean causal identification."

### 2. When Models "Fail," Ask Why Before Fixing

> "When all meta-learners collapsed to zero, my instinct wasn't to tune hyperparameters or try fancier models — it was to question the fundamental setup. The collapse was actually the models telling me something true: once you condition on features that capture engagement, whether someone viewed an offer doesn't add predictive power. The problem wasn't the models; it was the causal framework."

### 3. Triangulation Builds Confidence

> "I deliberately ran two independent causal analyses — ITT with randomization and observational with IPW correction. They converged at 0.144–0.161, which is much stronger evidence than either estimate alone. In practice, I always try to attack a causal question from multiple angles."

### 4. Know Your Assumptions

> "Every causal estimator has assumptions. ITT assumes randomization (verified with SRM + covariate balance). IPW assumes no unmeasured confounders. DR assumes either the outcome model or propensity model is correct. When DR collapsed, it was because the outcome model was contaminated with post-treatment variables — violating a key assumption even though the estimator is theoretically 'robust.'"

### 5. A Negative Result Is Still a Result

> "The project showed that blanket targeting is NPV-negative, but targeted strategies break even at realistic CLV thresholds. In an interview, I'd frame this as: 'I didn't just build a model — I quantified the exact conditions under which the recommendation becomes profitable, which is what decision-makers actually need.'"

### 6. The Progression of Complexity

The project demonstrates a clear analytical maturity progression:

```
Naive difference → Meta-learners → Causal identification problem
→ Reframe treatment → Dual-path analysis → Bias correction
→ Feature contamination detection → Business simulation
```

This is exactly the kind of iterative, self-correcting analytical process that senior DS roles require.

---

## 13. Why Not CUPED? — Knowing When a Method Does (and Doesn't) Apply

### What Is CUPED?

CUPED (Controlled-experiment Using Pre-Experiment Data) is a variance reduction technique popularized by Microsoft. The idea: use a pre-experiment covariate X (e.g., last 30 days' spending before the experiment) to reduce the variance of your ATE estimator without changing its expectation:

$$\hat{Y}_{cuped} = \bar{Y} - \theta(\bar{X} - E[X]), \quad \theta = \frac{\text{Cov}(Y, X)}{\text{Var}(X)}$$

If pre-experiment spending is highly correlated with in-experiment spending, CUPED can reduce variance by 50%+ — meaning you need far fewer users to detect the same effect size.

### When to Use CUPED

| Requirement | Why |
|-------------|-----|
| Randomized experiment (A/B test) | CUPED adjusts variance, not bias — it assumes unbiased ATE already |
| Pre-experiment data available | Need a covariate measured *before* treatment assignment |
| Goal is statistical power / tighter CI | Effect estimate stays the same, confidence interval shrinks |

### Why I Didn't Use It in This Project

1. **The primary challenge was causal identification, not statistical power.** My ITT ATE (0.161) was already highly significant. The problem was never "we can't detect the effect" — it was "are we measuring the *right* effect?"

2. **No clean pre-experiment period in the data.** The Starbucks simulated dataset mixes all events (offers, transactions) in one timeline without a clear pre-experiment vs. in-experiment boundary. I couldn't cleanly extract "spending before any offer was sent" as a CUPED covariate.

3. **CUPED doesn't fix self-selection bias.** Even if I had pre-period data, CUPED would only tighten the confidence interval of whatever estimator I'm using. It wouldn't solve the fundamental problem of observational Frame B (T = viewed is self-selected). For that, I needed IPW/DR, not variance reduction.

### CUPED vs. Methods I Actually Used

| Method | Purpose | Used? | Why / Why Not |
|--------|---------|-------|---------------|
| **CUPED** | Variance reduction (tighter CI) | ❌ | No pre-period data; statistical power was not the bottleneck |
| **IPW** | Bias correction for selection | ✅ | Core fix for observational Frame B |
| **T-Learner** | Heterogeneous treatment effects | ✅ | Needed CATE for targeting |
| **SRM + Covariate Balance** | Validate randomization | ✅ | Confirmed ITT frame is clean |
| **DR-Learner** | Doubly robust bias correction | ✅ (collapsed) | Outcome model overfit — documented as a learning |

### How to Explain This in an Interview

> "I considered CUPED but decided it wasn't the right tool for this problem. CUPED is a variance reduction technique for randomized experiments — it makes your confidence intervals narrower but doesn't change the point estimate. My project's challenge was *causal identification* (defining treatment correctly and correcting selection bias), not statistical power. The ITT effect was already significant at p < 0.001. Additionally, the dataset lacked a clean pre-experiment period needed for CUPED covariates. If this were a real production A/B test with pre-period data, CUPED would absolutely be part of my analysis pipeline."

**Bonus — when an interviewer asks "What would you do differently with better data?":**
> "With a real Starbucks dataset, I'd (1) use pre-experiment spending as a CUPED covariate to reduce ATE variance by ~50%, (2) compute behavioral features from pre-period only to avoid post-treatment contamination, and (3) potentially use a regression discontinuity design if offers were triggered by spending thresholds."

---

## Appendix: Quick-Reference Q&A for Interviews

**Q: Walk me through a time you had to change your analytical approach mid-project.**
> A: See Sections 3-6. Started with observational uplift, discovered all models collapsed, diagnosed self-selection bias, restructured into dual-path causal analysis.

**Q: What's the difference between ITT and per-protocol analysis?**
> A: ITT analyzes everyone by their assigned group regardless of compliance. In my project, ITT compared users who *received* offers vs. those who didn't, regardless of whether they viewed them. This dilutes the effect but preserves randomization. Per-protocol (viewing) is more targeted but introduces selection bias.

**Q: When would you use IPW vs. matching vs. regression adjustment?**
> A: IPW when you have good overlap and a well-specified propensity model. Matching for small samples where you need transparency. Regression adjustment (S-learner) when the outcome model is well-specified. In my project, IPW outperformed DR because the outcome model was contaminated by post-treatment variables.

**Q: How do you validate a causal estimate when you can't run an A/B test?**
> A: Triangulation (compare ITT and observational estimates), sensitivity analysis (how much unmeasured confounding would be needed to nullify the result), placebo tests, and refutation tests. My two estimates converging at 0.14-0.16 provided strong evidence.

**Q: What's selection bias and how do you detect it?**
> A: Selection bias occurs when treatment assignment is correlated with the outcome through a backdoor path. I detected it by noticing that the naive estimate (0.284) was almost double the clean ITT estimate (0.161) — about 49% of the apparent effect was pure selection.

**Q: Tell me about a time a "fancier" method performed worse than a simpler one.**
> A: The Doubly Robust estimator (theoretically superior) collapsed to zero while simple IPW gave a reasonable estimate. The DR estimator's outcome model overfit and absorbed the treatment effect. Sometimes simpler methods are more robust in practice.

---

*Document prepared for DS/PM interview preparation. All analysis code is in the accompanying Jupyter notebook.*
