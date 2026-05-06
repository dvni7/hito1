# PROMPTS.md — AI Interaction Log
**Capstone Hito 1 | F1 Race Strategy Advisor**

---

## Interaction 1 — Baseline choice and leakage classification

**Context:**
Working on Hito 1 for the F1 Race Strategy Advisor capstone. The dataset `f1_strategy_race_level.csv` includes both pre-race features (qualifying_position, constructor_tier, driver stats) and post-race strategy observations (n_stops, compound_sequence, stint_lengths). The assignment requires a baseline that is "F1-defensible" and implemented before seeing test results. I need to decide (a) what the heuristic baseline should be, and (b) how to correctly classify features as pre-race vs. scenario inputs vs. audit columns.

**Prompts used:**
1. "In F1, what is the empirical probability that a driver starting in positions 1–5 finishes in the points (top 10)? What about positions 6–10 and 11+? Give me domain-grounded estimates I can use as a heuristic baseline."
2. "For an F1 race strategy advisor that answers 'what-if I choose strategy X vs Y', which of these features are pre-race signals, which are scenario inputs (intentionally set by the engineer), and which are post-race leakage I should not use: qualifying_position, n_stops, compound_sequence, safety_car_periods, driver_recent_dnf_rate, stint_lengths, constructor_tier."

**Output received:**
1. AI confirmed historical rates: ~80–90% top-10 rate from P1–P5, ~45–55% from P6–P10, ~10–15% from P11+. Suggested thresholds: 0.85 / 0.50 / 0.12. Noted that these are pre-2022 regulation estimates and the 2022 reset may shift distributions.
2. AI classified: pre-race = {qualifying_position, constructor_tier, driver_recent_dnf_rate, driver_experience_races}; scenario inputs = {n_stops, compound_sequence, stint_lengths}; audit/excluded = {safety_car_periods, weather outcome} — flagged that safety_car_periods is a race-outcome feature that should not appear as a predictor unless framed explicitly as a stress-test slice.

**Validation:**
- Checked heuristic thresholds against 2019–2021 training data: P(top10 | qpos ≤ 5) = 0.83, P(top10 | qpos 6–10) = 0.51, P(top10 | qpos > 10) = 0.14. Close to AI estimates, confirming the heuristic is data-consistent.
- Verified leakage classification against assignment instructions: "scenario inputs allowed because product is a scenario comparison tool" — consistent with AI output.

**Adaptations made:**
- Threshold for group 2 adjusted from 0.50 to 0.50 (unchanged — data agreed).
- Threshold for group 1 set to 0.85 (AI said 0.80–0.90; I chose 0.85 as midpoint, conservative for calibration).
- `safety_car_periods` kept in dataset as a binary audit column and scenario stress-test, NOT as a model predictor — consistent with AI recommendation and assignment leakage rules.

**Final Decision:**
Heuristic baseline: P(top10) = 0.85 / 0.50 / 0.12 based on qualifying_position buckets. Feature leakage classification: pre-race features for the baseline, scenario inputs declared explicitly in the notebook leakage audit cell. Safety car excluded from model.

---

## Interaction 2 — Calibration approach for small calibration set

**Context:**
The temporal split uses 2022 as the calibration block (approximately 440 driver-race entries across ~22 rounds × 20 drivers). I need to fit a calibration mapping on this set. I'm choosing between Platt scaling (logistic regression on raw scores) and isotonic regression (non-parametric). The calibration set is small, which makes isotonic regression prone to overfitting.

**Prompts used:**
1. "I have ~440 samples in my calibration set (2022 F1 season). I want to calibrate a logistic regression model's probability output. Should I use Platt scaling or isotonic regression? What are the risks of each with this sample size?"
2. "Show me the sklearn code to fit CalibratedClassifierCV with method='sigmoid' on a separate calibration set (not cross-val) after the model is already trained on training data."

**Output received:**
1. AI recommended Platt scaling (sigmoid) for small calibration sets because isotonic regression requires more data to avoid overfitting the calibration curve — it can produce a step function that memorises the calibration set. With N ≈ 440 and a binary target, Platt scaling is standard practice. Isotonic regression becomes preferable when N > 1000 and the score distribution is strongly non-monotonic (e.g., tree ensembles).
2. AI provided code using `CalibratedClassifierCV(base_estimator=model, method='sigmoid', cv='prefit')` fitted on the calibration X, y.

**Validation:**
- Ran calibration curve on 2022 set: Platt scaling reduced ECE from 0.09 (uncalibrated LR) to 0.04. Isotonic regression on same set gave ECE 0.03 but visually overfit in the high-probability tail (only 12 samples above predicted 0.80).
- Decision confirmed: Platt scaling is correct choice for this set size.

**Adaptations made:**
- AI suggested `cv='prefit'` — verified this requires the model to be already fitted before passing to CalibratedClassifierCV. Adjusted code to fit logistic regression separately on train set first, then wrap with `cv='prefit'`.
- AI's code used deprecated `base_estimator` parameter in newer sklearn — replaced with `estimator=` parameter.

**Final Decision:**
Use `CalibratedClassifierCV(estimator=lr, method='sigmoid', cv='prefit')` fitted on 2022 calibration set. Isotonic regression is noted as an experiment for Hito 2 (Experiment E3) when we have more complex models with larger effective calibration samples via cross-validation.
