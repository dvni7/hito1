# mitigations.md — Hito 2
**F1 Race Strategy Advisor | IIT414W Capstone**

Risks and mitigations tied directly to the failure modes identified in `error_analysis.md` and the confounds documented in `leakage_audit.md`.

---

## Risk 1 — one_stop × wet: Brier = 0.264 (worst prediction cell)

**What fails:** The model is systematically unreliable for 1-stop strategies in wet races. It applies dry-race priors to a context where rain randomises outcomes — drivers on a 1-stop in wet conditions are often there because the team gambled on drying conditions, and the outcome is driven by whether the bet was right, not by pre-race features.

**What would break in production:** An engineer asking "what if I run a 1-stop at Spa if it rains?" gets a probability estimate with ECE likely > 0.15. The number is misleading because the model has no rain-onset signal.

**Mitigation:**
1. Integrate a rain-probability forecast feature (available from the Meteorological Consortium used by FIA). Flag the prediction as "high-uncertainty" when `P(rain) > 0.3` at race start.
2. Add `wet_laps` as a feature once the dataset is cleaned — currently `weather_actual` = "wet" but `wet_laps` has variance that would help the model distinguish a fully wet race from a transitional one.
3. Short-term: add a UI warning when the engineer sets a 1-stop scenario and the circuit/date has historically high rain probability. The warning does not block the query — it anchors the engineer's interpretation.

---

## Risk 2 — Safety-car blind spot (semi-street circuits, Brier = 0.141–0.159)

**What fails:** `safety_car_periods` is all zero in this dataset. Semi-street circuits (Baku, Singapore, Jeddah) have historical SC rates of 40–60%. The model cannot distinguish a 2-stop race caused by a safety-car pit window from a planned 2-stop. This inflates error at these circuits.

**What would break in production:** Strategy recommendations at Baku or Singapore are the least reliable outputs of this model. A team following the tool at Singapore without knowing this could make a worse call than a human engineer relying on domain knowledge alone.

**Mitigation:**
1. Rebuild the dataset with safety-car lap data from the lap-level file (`f1_strategy_lap_level.csv` is already in the repo) — join `safety_car_laps` correctly and re-train.
2. Until then, add a circuit-specific reliability flag: circuits with historical SC rate > 35% get a "SC-blind" warning in the output.
3. In the model: add a binary feature `circuit_is_high_sc` (Baku, Singapore, Jeddah, Monaco, Sochi) as a proxy until the SC column is fixed. This captures the distributional shift even without lap-by-lap SC data.

---

## Risk 3 — Strategy confounding: associative estimates masked as causal recommendations

**What fails:** As documented in `leakage_audit.md` §2, the model conflates "teams that historically ran 2-stop strategies at this context" with "what would happen if you forced a 2-stop." The output is associative, not causal.

**What would break in production:** An engineer at a midfield team could see that front-tier teams who ran 2-stops had higher P(top10) and conclude "2-stop is better" — but the front-tier teams chose 2-stop because their car allowed it, not because the strategy caused the result.

**Mitigation:**
1. Add a propensity model: estimate P(team chooses 2-stop | context). Use the propensity score as an input feature to partially control for selection bias.
2. Add explicit language to the UI: "This estimate reflects historical outcomes for teams with similar context who chose this strategy. It is not a causal prediction." (Non-negotiable if this tool goes to a real team.)
3. Run the what-if only for strategies that the team's car profile actually supports — a mid-field constructor running a 1-stop where front-tier teams ran 2-stops is outside the tool's training distribution.

---

## Risk 4 — is_top3 calibration above P = 0.5

**What fails:** The is_top3 model has thin calibration data above predicted P = 0.5 (fewer than 25 test rows). Platt scaling may not generalise reliably in this regime.

**What would break in production:** For top-grid drivers (P1–P3 starters with front constructors), the model often predicts P(top3) > 0.5. The calibration curve above 0.5 is based on sparse data — the probability values may be systematically high or low.

**Mitigation:**
1. Use isotonic regression calibration for is_top3 on a longer calibration window (e.g., 2021–2022 combined instead of 2022 only). Isotonic regression becomes preferable over Platt when the score distribution is non-monotonic, which it is for GBT on a 15%-positive target.
2. Report confidence intervals alongside the point estimate using bootstrap sampling over the calibration set. Flag predictions above 0.5 as "extrapolation zone."

---

## Summary table

| Risk | Current Brier impact | Mitigation complexity | Priority |
|------|---------------------|----------------------|----------|
| one_stop × wet | +0.143 above mean | Medium (needs weather API) | High |
| Safety-car blind spot | +0.030 at semi-street | Low (lap-level file exists) | High |
| Strategy confounding | Unquantified but structural | High (causal modelling) | Medium |
| is_top3 calibration > 0.5 | Unquantified | Low (change calibration method) | Medium |
