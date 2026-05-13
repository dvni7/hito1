# baseline_comparison.md — Hito 2
**F1 Race Strategy Advisor | IIT414W Capstone**

---

## 1. Metrics on test set (2023–2024)

### is_top10 (primary target — locked from Hito 1)

| Model | Brier ↓ | Log Loss ↓ | ROC-AUC ↑ | vs Docent |
|-------|---------|-----------|----------|-----------|
| Grid-rule reference (Hito 1 floor) | 0.2080 | — | — | — |
| LR+Platt (Hito 1 baseline) | 0.1396 | 0.4436 | 0.8766 | +0.0076 above docent |
| **LR+Platt (Hito 2, richer features)** | **0.1322** | **0.4173** | **0.8920** | ties docent |
| RF+Platt | 0.1284 | 0.4105 | 0.8950 | beats docent |
| **GBT+Platt (production)** | **0.1255** | **0.4006** | **0.9022** | beats docent |
| Docent reference | 0.1320 | — | 0.8920 | — |

GBT+Platt beats the docent baseline on both Brier and ROC-AUC. The improvement over the Hito 1 LR baseline comes from adding `circuit_type`, `driver_circuit_prior_avg`, and encoded strategy types as features, plus the non-linear capacity of the gradient boosting model.

### is_top3 (expansion target — podium classification)

| Model | Brier ↓ | Log Loss ↓ | ROC-AUC ↑ |
|-------|---------|-----------|----------|
| Grid-rule reference (P1–P3 baseline) | ~0.130 | — | — |
| LR+Platt | 0.0789 | 0.2522 | 0.9212 |
| RF+Platt | 0.0769 | 0.2422 | 0.9259 |
| **GBT+Platt (production)** | **0.0704** | **0.2347** | **0.9279** |

No docent reference exists for `is_top3` — we use the is_top10 docent as an upper bound on difficulty and note that the lower positive rate (15.5% vs 51.7%) mechanically reduces the Brier score range.

---

## 2. Calibration quality

Both GBT+Platt models are well-calibrated on the 2022 block. Key observations from `calibration_curves.png`:

**is_top10:** Platt scaling reduces miscalibration at both tails. The model slightly overestimates probabilities in the 0.6–0.8 range (typical for gradient boosting with imbalanced classes). Expected Calibration Error (ECE) ≈ 0.04.

**is_top3:** Calibration is tighter in the low-probability regime (most drivers have P(top3) < 0.20). The model is well-calibrated below 0.5. Above 0.5, sample size is thin — Platt scaling provides a reasonable but uncertain mapping. ECE ≈ 0.05.

**Probability quality note for is_top3:** Because is_top3 is a 15.5%-positive binary target, the model's probability outputs are concentrated in the 0–0.4 range. A predicted P(top3) = 0.80 corresponds to a very strong prior belief. Engineers should treat `is_top3` probabilities as relative comparisons (A > B) rather than absolute calibrated frequencies above 0.5.

---

## 3. Model selection rationale

GBT+Platt is selected as the production model for both targets because:
1. It achieves the lowest Brier on test — primary metric per framing.md §2.
2. It captures non-linear interactions (grid_position × constructor_tier, strategy_type × circuit_type) that logistic regression misses.
3. Platt calibration on 2022 prevents test-set leakage in the calibration step.

The LR+Platt baseline is retained in the notebook for direct comparison to Hito 1.

---

## 4. What the expansion target adds

`is_top3` cannot replace `is_top10` — they measure different things. The comparison is useful because:

- When `P(top10)` is near-certain (e.g., P2 grid, front constructor), `is_top10` saturates and loses discrimination between strategies. `is_top3` remains sensitive.
- A strategy that marginally favours 2-stop on `is_top10` (+0.002) can massively disfavour it on `is_top3` (−0.286) — see `whatif_comparison.md` for the Qatar 2024 example.
- The disagree rate across the test set is ~17.5% of driver-race pairs when comparing 1-stop vs 2-stop. These are the cases where an advisor using only `is_top10` would be silent or misleading.
