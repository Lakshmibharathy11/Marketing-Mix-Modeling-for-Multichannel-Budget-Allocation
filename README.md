# Marketing Mix Model — Multi-Channel Budget Allocator

[![Dashboard](https://img.shields.io/badge/Live%20Dashboard-Looker%20Studio-blue)](https://datastudio.google.com/reporting/4b8a253d-cede-481e-bd2c-54a4517b0cd9)

## Project Goal

Most businesses running ads across multiple channels face the same problem: platform dashboards (Google Ads, Meta Ads Manager) each claim credit for the same conversions, making it impossible to know which channel genuinely drove incremental growth — and therefore, where to actually put next quarter's budget.

This project builds a **channel-agnostic Bayesian Marketing Mix Model (MMM)** that solves this problem by:
- Estimating each channel's true incremental contribution using statistical inference, not platform-reported attribution
- Quantifying the uncertainty around every estimate (not just a single number, but a defensible confidence range)
- Recommending an optimized budget reallocation with explicit, documented reasoning for every constraint applied

The output is a business-ready budget allocator with a stakeholder-facing Looker Studio dashboard — mirroring the decision-support layer Google's own Meridian/Scenario Planner and Amazon's MMM portal both ship around their core models.

---

## Dataset

**Source:** Google Meridian official demo dataset (`geo_all_channels.csv`)
**Granularity:** 40 geographic markets × 156 weeks (2021-01-25 to 2024-01-15)
**Modeled at:** National-weekly level (aggregated across all 40 geos as the primary model; geo-level modeling documented as a future improvement)

**Variables:**
- 5 paid media channels: spend + impressions per channel per week
- 1 organic media channel (Organic_channel0_impression)
- 2 control variables: competitor_sales_control (standardized), sentiment_score_control (standardized)
- 1 non-media treatment: Promo intensity (weekly promotional activity index)
- KPI: weekly conversions (national total)

---

## Methodology

### Step 1 — Exploratory Data Analysis

Full EDA conducted before any modeling, with documented findings driving every subsequent modeling decision. Key findings:

**Spend vs. impressions are perfectly correlated (r = 1.0000 for all 5 channels)**
Spend and impressions carry identical information within each channel — including both as predictors would introduce perfect collinearity. Decision: use spend as the sole channel regressor, since the deliverable is a dollar-denominated budget recommendation.

**Conversions have a low coefficient of variation (CV = 6.9%)**
Mean conversions = 422.6M, std = 29.2M. Most weekly conversions come from a large, stable baseline — media channels explain only a portion of week-to-week movement. This set the expectation early: raw channel-conversions correlations would be weak pre-transform, which is expected and not a data quality issue.

**Same-week channel-conversions correlations: weak to mildly negative (r = 0.05 to -0.18)**
Confirmed as expected — three causes identified: (1) no adstock/lag applied yet, (2) baseline swamping the signal, (3) possible budget-pacing (spend reacting to sales rather than causing it). A scatter plot confirmed a formless cloud rather than any visible curved/nonlinear pattern, ruling out saturation as the sole explanation.

**Control variable findings:**
- Promo: r = 0.301, partial r = 0.276 after removing shared time-trend — real, trend-robust signal worth including
- Competitor sales control: r = -0.385 — strongest control relationship, logically directional (competitor gains cost this business conversions)
- Sentiment score control: r = -0.064 — weak at national level, retained to guard against omitted-variable bias
- Population: CV ≈ 0 at national level (summing 40 fixed-population geos produces a constant) — excluded from national model

**Organic_channel0_impression halo effect:**
Organic impressions correlate moderately with paid Channel0 spend (r = 0.462) — evidence that paid advertising lifts organic visibility. VIF = 9.18 when included alongside paid channels (~89% of its variation explainable by paid spend). Retained as a baseline-driver regressor to prevent omitted-variable bias, documented as a known attribution risk.

---

### Step 2 — Collinearity Diagnosis (VIF)

VIF (Variance Inflation Factor) measures how much unique information each channel provides, independent of the others. Formula: **Unique information % = (1/VIF) × 100**

| Channel | VIF | Unique Info | Verdict |
|---|---|---|---|
| Channel2 | 4.09 | 24.4% | Clean — most trustworthy estimate |
| Channel1 | 6.80 | 14.7% | Moderate |
| Channel0 | 10.18 | 9.8% | Borderline severe |
| Channel4 | 12.17 | 8.2% | Severe |
| Channel3 | 20.99 | 4.8% | Most severe — ~95% of its variation is explainable by the other 4 channels |

**Why VIF directly determines prior tightness:**
In Bayesian inference, the likelihood term (data-driven update) is weak for high-VIF channels — the data genuinely cannot cleanly isolate that channel's individual contribution because its spend always moved alongside the others. When the likelihood is weak, the prior dominates the posterior. Setting a tighter prior for high-VIF channels prevents the model from producing wild, uninformative estimates precisely where the data gives the least guidance — not by constraining the truth, but by anchoring the estimate to plausible industry norms when the data alone can't provide the anchor.

---

### Step 3 — Bayesian MMM with Google Meridian

**Tool:** Google Meridian v1.7.0 (open-source Bayesian MMM library, successor to LightweightMMM)
**Model type:** National-level, non-revenue KPI
**Sampling:** 4 MCMC chains, 1,000 adapt + 500 burn-in + 1,000 kept samples per chain (4,000 total posterior samples)

**Prior configuration (VIF-informed):**

All channels use LogNormal priors on ROI — same loc (mean belief), different scale (tightness):

| Channel | Prior Scale | Reasoning |
|---|---|---|
| Channel0 | 0.9 (default) | Moderate VIF |
| Channel1 | 0.9 (default) | Moderate VIF |
| Channel2 | 1.1 (widest) | Cleanest VIF — let data decide |
| Channel3 | 0.6 (tightest) | Severe VIF — prior guards against collinearity inflation |
| Channel4 | 0.7 | Severe VIF |

**Convergence diagnostics:**
R-hat < 1.002 for all parameters — excellent convergence across all 4 independent chains. The MCMC random walk reliably explored the full posterior space.

**Prior vs. posterior comparison:**
All five channels showed meaningful posterior updating (narrower distributions than priors), confirming the data contained real signal. Channel3's tight prior successfully anchored its estimate without the posterior wandering into implausible territory — the collinearity mitigation worked as designed.

---

### Step 4 — Model Results

| Channel | Spend Share | ROI (mean) | 90% Credible Interval | mROI | Action |
|---|---|---|---|---|---|
| Channel2 | 6% | 2.0 | (0.2, 5.9) | 0.9 | Increase — highest ROI, underfunded |
| Channel0 | 18% | 1.4 | (0.3, 3.5) | 0.6 | Maintain |
| Channel4 | 22% | 1.3 | (0.4, 2.8) | 0.6 | Slight reduction |
| Channel3 | 40% | 1.2 | (0.5, 2.3) | 0.6 | Maintain — largest budget, near saturation |
| Channel1 | 14% | 1.2 | (0.2, 3.0) | 0.5 | Reduce — lowest mROI |

**Total media contribution:** ~21.8% of conversions (14.4%, 31.1%) — remainder is stable baseline. This is consistent with the EDA finding (conversions CV = 6.9%) that most conversions come from a large, stable baseline characteristic of a mature, established business.

**Key insight — high baseline + saturation:**
This dataset represents an established business with a large organic baseline — most conversions happen regardless of ad spend (repeat customers, brand equity, organic demand). Three years of consistent spending has pushed most channels into the flat portion of their saturation curves, where mROI < 1.0 for all channels. This is not a modeling failure — it's an accurate diagnosis of a mature media mix already close to its efficient frontier.

---

### Step 5 — Budget Optimization

**Constrained optimization** (VIF-informed bounds):
- Channel2: ±40% (most trustworthy estimate, wider room to move)
- Channel0/Channel1: ±30% (default)
- Channel4: ±25%
- Channel3: ±15% (least trustworthy estimate, conservative constraint)

**Unconstrained optimization** (default ±30% all channels) run as comparison scenario.

| Scenario | Projected Lift |
|---|---|
| Constrained (VIF-informed) | +0.5% |
| Unconstrained (default) | +0.44% |

**Finding:** The two scenarios produced nearly identical recommendations, with Channel3 barely moving in either case. This revealed that the optimizer's allocation was driven primarily by each channel's saturation curve position — Channel3 was already near its optimal spend level regardless of constraint width. This validates that the collinearity mitigation worked upstream (during model fitting via tight priors) rather than needing to be re-applied as a hard constraint at the optimization stage.

**Recommended reallocation:**
- Channel2: +$4.8M (+39.7%) — highest ROI, most underfunded relative to efficiency
- Channel1: -$4.9M (-15.7%) — lowest mROI, trim first
- Channel4: -$2.3M (-4.8%) — modest trim
- Channel0: +$1.5M (+3.7%) — modest increase
- Channel3: +$0.8M (+0.9%) — essentially unchanged

---

## What I Learned

1. **Raw correlations between spend and conversions are almost always misleading** — same-week correlations don't capture adstock (lagged effects) and get swamped by a stable baseline. Statistical modeling with proper transforms is necessary, not optional.

2. **VIF is more than a collinearity warning — it directly informs prior setting** in a Bayesian framework. The formula `unique information % = (1/VIF) × 100` translates a diagnostic number into a concrete modeling decision.

3. **mROI, not ROI, should drive reallocation decisions** — ROI tells you historical average performance; mROI tells you where you currently sit on the saturation curve, which is the number that answers "should I spend more here right now?"

4. **A mature business with high baseline and saturated channels should expect modest optimization lift** — not because the model failed, but because the media mix is already close to its efficient frontier. Modest lift with high confidence is a better outcome than dramatic lift with wide uncertainty.

5. **Constrained optimization outperformed unconstrained in our test** — confirming that domain-knowledge-informed constraints (VIF-aware bounds) produce more defensible recommendations than unconstrained optimization, even when the lift difference is small.

---

## Limitations & Future Improvements

1. **Geo-level hierarchical modeling** — we deliberately simplified to national-weekly level for the first build. Running the full geo-level model (40 markets × 156 weeks) would give Meridian more statistical power to separate collinear channels, reducing the uncertainty in Channel3/Channel4's estimates. Meridian's GeoX feature supports this directly.

2. **Holdout experiment calibration** — our priors were set using VIF as a proxy for confidence. The more rigorous approach is calibrating priors against actual geo holdout experiments (running/not running media in different markets), which provides genuine causal evidence rather than a collinearity diagnostic. This is the industry gold standard.

3. **Promo interaction terms** — our model included Promo as an additive control, but didn't capture the likely interaction effect (promotions amplifying channel effectiveness during sale periods). Adding channel × Promo interaction terms would surface this mechanism.

4. **Organic channel modeling** — Organic_channel0_impression was included as a baseline driver but not decomposed further. Building a separate model for organic lift (driven by brand/SEO investment) would close the halo-effect attribution gap we identified in EDA.

5. **LLM-powered insight translation layer** — planned next phase: a grounded agent that translates posterior outputs into plain-language executive summaries, with a mechanical verification pass confirming every numeric claim in the narrative matches the actual model output (hallucination guard).

6. **Real business data** — this project used Google's simulated demo dataset. Applying the same pipeline to real advertiser data (with real spend variation, real geo experiments, and real promotional calendars) would allow validation of the model's recommendations against actual business outcomes.

---

## Project Structure
├── data/
│   └── geo_all_channels.csv          # Google Meridian demo dataset
├── outputs/
│   ├── eda/                          # EDA charts + summary
│   │   ├── weekly_spend_by_channel.png
│   │   ├── weekly_conversions.png
│   │   ├── conversions_vs_channel3.png
│   │   ├── control_variables_over_time.png
│   │   ├── channel3_impressions_vs_conversions_scatter.png
│   │   └── EDA_summary.md
│   ├── model/                        # Model outputs
│   │   ├── channel_summary.csv
│   │   ├── channel_summary_clean.csv
│   │   └── budget_reallocation.csv
│   └── dashboard/                    # Looker Studio data sources
│       ├── channel_performance.csv
│       ├── budget_reallocation_long.csv
│       └── headline_summary.csv
├── .gitignore
└── README.md

## Live Dashboard

[View the interactive budget allocation dashboard →](https://datastudio.google.com/reporting/4b8a253d-cede-481e-bd2c-54a4517b0cd9)

---

*Built with Google Meridian v1.7.0, Python 3.12, TensorFlow Probability, Pandas, Statsmodels, and Looker Studio.*
*Dataset: Google Meridian official simulated demo dataset (geo_all_channels.csv)*
