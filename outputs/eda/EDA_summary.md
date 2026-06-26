# EDA Summary — Marketing Mix Model Budget Allocator

**Dataset:** Google Meridian official demo dataset (`geo_all_channels.csv`)
**Granularity:** 40 geos x 156 weeks (2021-01-25 to 2024-01-15), aggregated to national-weekly level for the initial model (geo-level modeling deferred as a stretch goal)

## 1. Spend vs. impressions: redundant, not complementary

Spend and impressions are perfectly correlated within every channel (r = 1.0000 for all 5 channels). They carry identical information — including both in the model would create artificial collinearity.

**Decision:** use **spend** as the regressor for each channel, since the end deliverable is a dollar-based budget recommendation.

## 2. Conversions distribution

- Mean ≈ 422.6M, std ≈ 29.2M → CV ≈ 6.9% — conversions move very little relative to their own average.
- Empirical rule check: actual min/max sit just outside the 2-std band but comfortably inside 3-std — no severe outliers, mild extremes only.

**Implication:** a large, stable baseline (α) likely explains most weekly conversions. Media channels probably explain a comparatively small slice of the week-to-week movement — expect modest, not dramatic, channel coefficients.

## 3. Channel collinearity (VIF)

| Channel | VIF | Verdict |
|---|---|---|
| Channel2 | 4.09 | Clean — most trustworthy estimate |
| Channel1 | 6.80 | Moderate |
| Channel0 | 10.18–10.46 | Borderline severe |
| Channel4 | 12.17–12.18 | Severe |
| Channel3 | 20.99–22.36 | Most severe — ~95% of its variation is explainable by the other 4 channels |

**Implication:** Channel3 and Channel4's individual ROI estimates will carry wide, honest credible intervals. Plan to use **tighter, benchmark-informed priors** specifically for Channel0, Channel3, and Channel4 to partially compensate for what the data alone can't separate. Channel2's estimate can be trusted with a wider/flatter prior.

## 4. Raw impressions vs. conversions: weak/negative, as expected pre-transform

Same-week correlations ranged from 0.05 to -0.18 — weak or mildly negative. A scatter plot (Channel3) showed a loose, formless cloud — not a curved/nonlinear shape.

**Cause, most likely:** (a) no adstock/lag applied yet, (b) baseline swamping the signal, (c) possible budget-pacing (spend reacting to sales rather than only causing it). **Not** evidence of nonlinearity — that would need to show up as a visible curve, which it didn't.

**Implication:** this is expected and exactly why adstock + saturation transforms and proper regression (not raw correlation) are necessary before drawing conclusions about channel effectiveness.

## 5. Control variables

| Variable | Raw r with conversions | Notes |
|---|---|---|
| Promo | 0.301 (partial r ≈ 0.276–0.28 after removing shared time-trend) | Real, trend-robust signal. Consider channel x Promo interaction terms to capture promo amplifying ad effectiveness. |
| competitor_sales_control | -0.385 | Strongest control relationship; logical direction (competitor gains likely cost this business conversions). |
| sentiment_score_control | -0.064 | Weak at national level; keep in model, monitor at geo-level. |
| population | ~0 variation nationally (CV near zero — summing 40 geos cancels week-to-week change) | No usable signal at national level; relevant only for geo-level/per-capita modeling. |

## 6. Organic_channel0_impression — halo effect risk

- Real variation (CV = 0.54), but near-zero raw correlation with conversions (pre-transform, as expected).
- Correlates moderately with paid Channel0 (r = 0.462) — evidence of a halo effect (paid activity lifting organic visibility).
- VIF = 9.18 when included alongside the 5 paid channels — ~89% of its variation is explainable by paid spend collectively.

**Implication:** include as its own baseline-driver regressor (not a channel to optimize budget for) to avoid omitted-variable bias, while flagging the halo effect as a known attribution risk in the writeup.

## Summary: what this means for the model build

1. Use **spend**, not impressions, as the channel regressors.
2. Apply **adstock + saturation transforms** to every channel before regression — raw correlations are not representative of true effects.
3. Use **tighter, informative priors** for Channel0, Channel3, Channel4 (high VIF); a flatter prior is defensible for Channel2.
4. Include **Promo, competitor_sales_control, sentiment_score_control, and Organic_channel0_impression** as additional regressors/controls — each carries a real, defensible signal or guards against omitted-variable bias.
5. Exclude **population** from the national-level model; revisit at geo-level.
6. Document the **halo effect** between Organic_channel0_impression and paid channels as a known limitation, not an oversight.
