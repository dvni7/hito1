# Hito 1 — Framing Document
**Capstone: F1 Race Strategy Advisor | Module 5 — Unit IV**

---

## 1. Decision Context

**What decision are we supporting?**
A race engineer must decide, on the Saturday evening before race day, which pit-stop strategy (number of stops and tire compound sequence) maximises the probability that their driver finishes in the top 10. The tool produces a calibrated probability — P(top10 | strategy scenario) — allowing engineers to compare alternatives with concrete, quantified risk.

**Who is the decision-maker?**
The primary user is the **race engineer** (the person who directly controls in-race calls). A secondary user is the **sporting director**, who validates the recommendation against constructor-points priorities.

**When in the race weekend?**
The decision window opens after qualifying (Saturday ~18:00 local time) and closes at race formation lap (Sunday ~13:50 local time). The model is queried with *scenario inputs* set by the engineer — the race has not yet happened, so strategy features (`n_stops`, `compound_sequence`) are intentionally injected as "what-if" values, not observed signals.

---

## 2. Target and Primary Metric

**Target variable:** `is_top10` — binary flag equal to 1 if the driver finishes in positions 1–10, 0 otherwise. This directly encodes the points-scoring threshold in F1 (only the top 10 score constructor points).

**Primary metric: Brier Score**
We optimise for Brier score because:
- The product outputs a **calibrated probability**, not a ranking. Brier score penalises both poor discrimination *and* poor calibration.
- A race engineer acting on P(top10) = 0.85 must trust that ~85% of similar scenarios actually result in top-10 finishes. Miscalibrated scores destroy trust.
- Reference floor: Grid-rule baseline Brier = 0.208; calibrated docent model Brier = 0.132.

**Secondary metric: ROC-AUC** — measures discrimination ability across all thresholds; used to compare model variants during Hito 2.

**Calibration is a graded deliverable.** We use Platt scaling trained on 2022 calibration data (never on the test set).

---

## 3. Baseline Plan

**Heuristic baseline — F1-defendable rationale:**

> P(top10) = 0.87 if `grid_position` ≤ 5
> P(top10) = 0.68 if 5 < `grid_position` ≤ 10
> P(top10) = 0.25 if `grid_position` > 10

*Rationale:* In F1, drivers starting in the top 5 nearly always have the pace to score points unless they suffer a mechanical failure or collision. Positions 6–10 are within the points zone but face overtaking pressure. Beyond P10, scoring requires significant attrition. Thresholds are grounded in F1 domain knowledge and validated against 2019–2021 base rates (actual: 0.874 / 0.679 / 0.254), not test data.

Note: `qualifying_position == grid_position` in 100% of rows in this dataset (no grid penalties recorded), so both columns are interchangeable. We use `grid_position` as the canonical name.

**Simple model baseline:**
Logistic regression with three features: `grid_position` (continuous), `constructor_tier` (ordinal-encoded: front=2, midfield=1, backmarker=0), and `n_stops` (SCENARIO INPUT — declared explicitly in leakage audit; engineer sets this value to ask "what-if"). Trained on 2019–2021, calibrated with Platt scaling on 2022.

Both baselines are evaluated on the 2023–2024 test set with Brier score, log loss, and calibration curve.

---

## 4. What-If Comparison Plan

The model is a **scenario comparison tool**: the engineer fixes pre-race context (driver, circuit, grid position) and varies strategy inputs (`n_stops`, `compound_sequence`) to compare P(top10) across scenarios.

**Scenario A — Monaco 2024, Norris (McLaren):**
- `grid_position` = 4, `constructor_tier` = **front** (McLaren was front-tier at Monaco 2024 per dataset)
- Scenario A1: `n_stops=1`, `compound_sequence=M-H` → see notebook for P(top10)
- Scenario A2: `n_stops=2`, `compound_sequence=S-M-H` → see notebook for P(top10)
- **Engineer question:** Does a two-stop undercut justify the track-position risk at Monaco, where overtaking is nearly impossible?

**Scenario B — Monza 2023, Russell (Mercedes):**
- `grid_position` = 8, `constructor_tier` = **midfield** (Mercedes was midfield-tier at Italian GP 2023 per dataset)
- Scenario B1: `n_stops=1`, `compound_sequence=H-M` → see notebook for P(top10)
- Scenario B2: `n_stops=2`, `compound_sequence=S-H-M` → see notebook for P(top10)
- **Engineer question:** At a DRS-heavy circuit where overtaking is easier, does a two-stop allow for a faster final stint to recover positions?

These scenarios use concrete, dataset-verified feature values. Results are computed in `hito1_baseline.ipynb` §9 and will be extended in Hito 2 with compound_sequence as an additional encoded scenario input.

---

## 5. Dataset Limitations (acknowledged)

We acknowledge the following limitations from the five known issues, and their consequences for our recommendations:

**Limitation 1 — `qualifying_time_s` is entirely NULL; `qualifying_position == grid_position` for all rows.**
In this dataset, no grid penalties are recorded — qualifying_position and grid_position are identical for all 2,447 entries. This means the dataset cannot capture the ~8–12% of race weekends where engine changes or DSQs cause a grid-position shift. We use `grid_position` as the canonical feature name and do not build any story around qualifying lap time.

**Limitation 2 — Strategy features are post-race observations used as scenario inputs.**
`n_stops`, `compound_sequence`, and `stint_lengths` are not known before the race. In this capstone they are *intentionally* injected as user-controlled scenario inputs — the model is a "what-if" comparator, not a pre-race predictor. We declare this distinction explicitly: any model that treats these as pre-race signals would be a target-leakage error in any other context. All scenario-input features must be set by the engineer, not predicted.

**Limitation 3 — `safety_car_periods` is all zero in this dataset version.**
The column exists but contains no positive values across all 2,447 rows and all six seasons. This is a known data-recovery limitation: SC intervals were not successfully joined to this race-level artifact. Consequence: we cannot use safety car as a predictor or stress-test slice on this dataset. We use `weather_actual` (dry/wet) as the available audit slice instead. A future version of the dataset should incorporate SC data from the lap-level file.

**Limitation 4 — Strategy choice is not independent of car pace, driver, weather, and race incidents.**
A team that chooses a 2-stop strategy may do so because their car degrades faster — meaning `n_stops` is confounded with underlying pace. The what-if tool should be interpreted as "holding pace constant, what is the effect of strategy?", not as a causal estimate.

**Limitation 5 — Coverage starts in 2019.**
Six seasons is a modest sample. The 2022 ground-effect regulation reset may create non-stationarity that the temporal split partially captures but cannot eliminate.

---

## 6. Three Experiments Planned for Hito 2

| # | Experiment | Hypothesis | Primary metric target |
|---|-----------|-----------|----------------------|
| **E1** | Random Forest with pre-race features (`grid_position`, `constructor_tier`, `driver_prior3_avg_finish`, `constructor_prior3_avg_finish`, `driver_circuit_prior_avg`) | RF will better capture non-linear interactions (e.g., front constructor × front-row start) than logistic regression; expected Brier < 0.160 | Brier ≤ 0.160 |
| **E2** | Gradient Boosting (XGBoost) adding encoded scenario inputs (`n_stops`, `compound_sequence` one-hot encoded, `weather_actual`) | Explicitly modelling strategy choices will widen the P(top10) gap between 1-stop and 2-stop scenarios, making the what-if tool more actionable | Brier ≤ 0.140; scenario delta ≥ 0.08 |
| **E3** | Calibration method comparison — Platt scaling vs. isotonic regression (both trained on 2022 only) | Isotonic regression will produce a better-calibrated curve for the RF/XGB models because the score distributions are non-monotonic; Platt will suffice for logistic regression | Expected Calibration Error ≤ 0.05 |

All experiments follow the locked temporal split and use only 2022 calibration data for calibration fitting.

---

## 7. Team Workflow

| Period | Task | Owner |
|--------|------|-------|
| Mon → Tue | Draft framing.md (all 7 sections), define leakage audit categories | Daniela |
| Mon → Tue | Set up GitHub repo, create data/ folder, verify CSV loads cleanly | Daniela |
| Tue → Wed AM | Build and run hito1_baseline.ipynb end-to-end; verify "Run All" works | Daniela |
| Tue → Wed AM | Document PROMPTS.md (≥2 AI interactions with 6-field standard) | Daniela |
| Wed class | Final review, push to GitHub, submit Canvas URL | Daniela |

*Note: submission is a single-member team contribution for this Hito.*
