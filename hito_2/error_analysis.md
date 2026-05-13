# error_analysis.md — Hito 2
**F1 Race Strategy Advisor | IIT414W Capstone**

Error analysis for GBT+Platt on test set (2023–2024). Sliced by three dimensions: strategy type, circuit type, and weather. Both targets analysed. Visualisations in `error_analysis_slices.png` and `error_heatmaps.png`.

---

## 1. Slice 1 — Strategy type

| Strategy | is_top10 Brier | is_top3 Brier | N (test) |
|----------|---------------|--------------|----------|
| one_stop | **0.1411** | **0.0795** | ~420 |
| three_plus_stop | 0.1315 | 0.0634 | ~55 |
| two_stop | **0.1125** | 0.0671 | ~390 |
| no_stop | 0.0144 | 0.0080 | ~24 |

**Failure mode — one_stop strategies have the highest error on both targets.** The model struggles most with 1-stop races. The pattern is asymmetric: drivers who ran a 1-stop and failed to score points (e.g., by being undercut at a pit window) look almost identical to drivers who ran a 1-stop and finished P8. The model cannot distinguish because the outcome driver is pre-race context, not strategy.

**Specific failure context:** one-stop strategies at permanent circuits in wet conditions produce Brier = 0.264 — more than double the overall mean. Wet 1-stop races are the single most unreliable prediction cell (see cross-slice below).

**is_top3 asymmetry:** one_stop also has the worst is_top3 Brier, but the gap to two_stop is larger proportionally (0.0795 vs 0.0671). Two-stop strategies in the training data are more strongly associated with podium-fighting teams, so the model has a cleaner signal.

---

## 2. Slice 2 — Circuit type

| Circuit type | is_top10 Brier | is_top3 Brier | N (test) |
|-------------|---------------|--------------|----------|
| semi-street | **0.1406** | **0.0517** | ~115 |
| street | 0.1263 | 0.0707 | ~70 |
| permanent | 0.1221 | 0.0740 | ~704 |

**Semi-street circuits are the hardest for is_top10.** Circuits like Baku, Singapore, and Jeddah have high safety-car rates and unpredictable attrition. Our dataset has `safety_car_periods = 0` for all rows (documented limitation in Hito 1 framing.md §5), so the model has no signal for the race disruptions that most affect outcomes at these tracks.

**Interesting inversion:** is_top3 Brier is *lowest* at semi-street circuits (0.0517), while is_top10 Brier is highest there. This is a class-imbalance artefact: at semi-street circuits, very few drivers realistically podium (only top-tier starters do), so the model's conservative low predictions for most drivers are correct for is_top3. The model fails on is_top10 at semi-street because mid-field attrition creates unexpected points finishes.

---

## 3. Slice 3 — Weather

| Weather | is_top10 Brier | is_top3 Brier | N (test) |
|---------|---------------|--------------|----------|
| wet | **0.1484** | 0.0714 | ~57 |
| dry | 0.1219 | **0.0702** | ~832 |

**Wet races produce 22% higher is_top10 error than dry races** (0.1484 vs 0.1219). Wet races scramble the usual grid→finish relationship. The model trained on dry-race patterns predicts too confidently for wet conditions because it has never seen the full outcome randomness.

**is_top3 Brier is approximately equal in wet and dry** (0.0714 vs 0.0702). Podium positions in wet races are still dominated by front-grid drivers — the underlying pre-race signal (grid_position, constructor_tier) is sufficient for is_top3 even when points-scoring becomes more random.

---

## 4. Cross-slice analysis — strategy × weather (is_top10 Brier)

| Strategy | dry | wet |
|----------|-----|-----|
| one_stop | 0.1288 | **0.2644** |
| two_stop | 0.1135 | 0.1066 |
| three_plus_stop | 0.1378 | 0.1103 |
| no_stop | 0.0098 | 0.0438 |

**The worst cell is one_stop × wet: Brier = 0.264.** This is 2.17× the overall is_top10 Brier. The concrete failure mode: a 1-stop strategy in a wet race typically means the team gambled on dry conditions from the start. If rain intensifies, the driver needs an unplanned switch — a scenario that looks like a scheduled 1-stop in the raw data but has very different dynamics. The model predicts a mid-range P(top10) and is systematically wrong in either direction.

**two_stop × wet is surprisingly robust** (Brier = 0.1066, below the overall mean). This likely reflects survivorship: teams that chose 2-stop in wet conditions were usually reacting to rain — they switched compounds earlier and managed the race. The strategy signal in wet 2-stops is more reliable.

---

## 5. Cross-slice — strategy × circuit type (is_top10 Brier)

| Strategy | permanent | semi-street | street |
|----------|-----------|-------------|--------|
| one_stop | 0.1388 | **0.1481** | 0.1411 |
| two_stop | 0.1074 | **0.1589** | 0.0941 |
| three_plus_stop | 0.1378 | 0.0352 | 0.1201 |

**two_stop × semi-street is the second worst cell** (0.1589). Semi-street circuits with safety cars often produce unexpected 2-stop races where the first stop is a safety-car stop and the second is a tyre-change — the model sees "2-stop" and applies a normal two-stop prior, missing the safety-car dynamics.

---

## 6. Summary of concrete failure modes

| Priority | Failure mode | Brier | Hypothesis |
|----------|-------------|-------|-----------|
| 1 | one_stop × wet | 0.264 | No safety-car / rain signal; 1-stop in wet is an outlier strategy |
| 2 | two_stop × semi-street | 0.159 | Safety-car-induced 2-stops look like planned 2-stops in raw data |
| 3 | one_stop × semi-street | 0.148 | Attrition randomness; no SC column in dataset |
| 4 | Wet races overall (is_top10) | 0.148 | Model trained on dry-race patterns underestimates wet variance |

The top-2 failure modes share a common root cause: the dataset has no safety-car information (`safety_car_periods = 0` for all rows). Both would improve substantially if SC data were available.

---

## 7. What the error analysis says about the expansion target

`is_top3` has qualitatively different failure patterns from `is_top10`:
- is_top3 does not fail at semi-street circuits (Brier = 0.052) — podium at these tracks is predictable from pre-race context.
- is_top3 fails equally in wet and dry — because podium stays concentrated among top-grid drivers regardless of weather.
- is_top3 fails hardest on one-stop strategies — same root cause as is_top10 but with less severity.

This confirms that the two targets are measuring different things and have different reliability profiles. An engineer should use is_top3 with higher confidence at semi-street circuits (where is_top10 is least reliable) and with caution for one-stop wet scenarios (where both targets struggle).
