# leakage_audit.md — Hito 2
**F1 Race Strategy Advisor | IIT414W Capstone**

---

## 1. Feature classification (summary)

Full classification in `hito2_modeling.ipynb` §3. Summary:

| Class | Columns used in model | Allowed? |
|-------|-----------------------|----------|
| `pre_race` | grid_position, constructor_tier, circuit_type, driver_prior3_avg_finish, constructor_prior3_avg_finish, driver_circuit_prior_avg | ✅ Yes |
| `scenario_input` | n_stops, strategy_type, compound_sequence, stint1–3_length, avg_pit_stop_duration_s, total_pit_time_s, first_pit_lap, last_pit_lap | ✅ Yes — declared user-set what-if values |
| `audit` | weather_actual, safety_car_periods, wet_laps, track_status_summary | ⚠️ Slice analysis only — NOT in model |
| `outcome` | is_top10, is_top3, finish_position, points, positions_gained, dnf | ❌ Targets only |
| `id` | season, round, circuit, Driver, Team, driver_id, ... | ❌ Not fitted |

No `audit` or `outcome` columns appear as predictors. No temporal leakage: calibration is fitted only on 2022; test is 2023–2024 and is never touched during training or calibration.

---

## 2. Strategy-confounding limitation (Hito 2 primary confound)

**The key confound:** In the training data, strategy choice (`n_stops`, `compound_sequence`) is not independent of car pace, driver ability, weather, or race incidents.

Concrete examples:
- A team that chose a 2-stop strategy at a given race may have done so because their car degraded faster — faster degradation is itself correlated with finishing position. The model learns an association between "2-stop" and a certain finishing distribution, but cannot isolate the causal effect of the strategy from the underlying pace signal.
- A driver who ran a 3-stop may have done so because a safety car reset their strategy — the 3-stop is an outcome of the race, not a free pre-race choice. Our dataset has no safety-car signal, so the model cannot distinguish planned 3-stops from reactive ones.
- Strategy choice correlates with race position at the time of the pit call. A driver running P4 at lap 20 makes a different strategy call than a driver running P12 at lap 20 — but both appear as "2-stop" in the data.

**Consequence for the what-if tool:** The tool produces associative estimates, not causal ones. P(top10 | 2-stop) in our model means "among historical drivers with this context who ran a 2-stop, the fraction who finished top-10." It does NOT mean "if you force this driver to run a 2-stop, they will finish top-10 with this probability." Engineers must interpret the output as a benchmark comparison, not a causal guarantee.

This confound is not fixable with the current dataset without adding race-time position data or explicit causal modelling (e.g., propensity-weighted regression). It is acknowledged and documented here.

---

## 3. Other leakage checks

| Check | Status | Notes |
|-------|--------|-------|
| Calibration set not used in training | ✅ Pass | 2022 block used only for Platt fitting |
| Test set not touched before evaluation | ✅ Pass | No hyperparameter tuning on test |
| `qualifying_time_s` excluded | ✅ Pass | Column is all-null; not used |
| `safety_car_periods` not a predictor | ✅ Pass | All-zero column; classified as `audit` |
| `weather_actual` not a predictor | ✅ Pass | Post-race observation; used only in error slices |
| `positions_gained` not used | ✅ Pass | Direct outcome; classified as `outcome` |
| `dnf` / `status` not used | ✅ Pass | Post-race outcomes only |
| `finish_position` not used | ✅ Pass | Target for regression experiment; not in binary models |

---

## 4. Confounding audit for strategy features

The strategy features (`n_stops`, `compound_sequence`, `strategy_type`) are declared `scenario_input` — meaning the engineer sets them as hypothetical values, not as observed post-race values. This is consistent with the tool's design: "what would happen if I chose strategy X?" The leakage is acknowledged rather than eliminated:

| Strategy feature | Why it's allowed | Residual risk |
|-----------------|-----------------|---------------|
| n_stops | Engineer controls stop timing; declared what-if | Confounded with pace/degradation (see §2) |
| compound_sequence | Engineer controls compound choice; declared what-if | Weather-dependent; model ignores weather |
| strategy_type | Categorical encoding of n_stops pattern | Same confound as n_stops |
| stint lengths | Controls when the stops happen | Correlated with safety-car timing |

The residual risk is documented, not hidden. It must be included in any recommendation made with this tool.
