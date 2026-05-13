# whatif_comparison.md — Hito 2
**F1 Race Strategy Advisor | IIT414W Capstone**

---

## The core question

Can `is_top3` surface a strategy recommendation that `is_top10` alone cannot? Yes — and this document shows where and why.

---

## Scenario pair 1 — Qatar GP 2024, P2 start: **DISAGREE**

**Context (held fixed):**
- Race: Qatar Grand Prix 2024
- Grid position: P2
- Constructor tier: midfield (per dataset race-specific classification)
- Driver-race context fixed; only strategy inputs are varied

**Strategies compared:**

| Input | Scenario A (1-stop) | Scenario B (2-stop) |
|-------|--------------------|--------------------|
| n_stops | 1 | 2 |
| strategy_type | one_stop | two_stop |
| compound_sequence | M-H | S-M-H |
| stint1_length | 26 | 12 |
| stint2_length | 27 | 22 |
| stint3_length | 0 | 19 |
| total_pit_time_s | 23.0 | 46.0 |
| first_pit_lap | 26 | 12 |
| last_pit_lap | 26 | 34 |

**Model output:**

| | P(is_top10) | P(is_top3) |
|--|------------|-----------|
| Scenario A — 1-stop M-H | 0.904 | **0.834** |
| Scenario B — 2-stop S-M-H | 0.906 | 0.548 |
| Delta (B − A) | **+0.002** | **−0.286** |

**Verdict: DISAGREE**
- `is_top10` prefers B (2-stop) by 0.002 — effectively no signal, statistically indistinguishable.
- `is_top3` prefers A (1-stop) by 0.286 — decisive and actionable.

**What this means for the advisor:**

An engineer relying only on `is_top10` in this context has no basis to choose a strategy. The difference is within noise (0.002). They might default to the 2-stop as "marginally safer" — but the podium model says that 2-stop destroys podium probability by 28.6 percentage points.

The mechanism: a driver starting P2 on a 1-stop strategy maintains track position through the race, protecting a potential podium. A 2-stop trades that track position for fresh tyres at a cost — in the model's training distribution, aggressive 2-stop strategies from P2 tend to cluster around drivers who undercut their way to points but lose the podium fight in the process. `is_top10` cannot see this because P2 drivers almost always score points regardless of strategy.

**What we still don't know:** Strategy choice in the training data is not independent of car pace or race context. Teams that chose a 2-stop at Qatar 2024 may have done so because their degradation profile required it — not as a free choice. The model treats it as a free choice. This is the strategy-confounding limitation. See `leakage_audit.md`.

---

## Scenario pair 2 — Italian GP 2024, Sainz P5: **AGREE**

**Context (held fixed):**
- Race: Italian Grand Prix 2024
- Driver: SAI (Sainz, Ferrari)
- Grid position: P5
- Constructor tier: front

| | P(is_top10) | P(is_top3) |
|--|------------|-----------|
| Scenario A — 1-stop M-H | 0.958 | **0.252** |
| Scenario B — 2-stop S-M-H | 0.933 | 0.160 |
| Delta (B − A) | −0.025 | −0.092 |

**Verdict: AGREE** — both targets prefer 1-stop.

Here the two targets give the same recommendation, but `is_top3` adds important context: even under the preferred 1-stop strategy, P(podium) is only 0.25. A Ferrari P5 at Monza is fighting for points, not the podium — the engineer should optimise for reliability and track position (1-stop), not for the upside of a 2-stop gamble.

---

## Summary: when the targets disagree vs agree

From a systematic scan of 889 test-set rows using the same 1-stop vs 2-stop template:
- **156 rows (17.5%) produced a DISAGREE verdict** between is_top10 and is_top3.
- Disagree cases are concentrated among P1–P5 starters where is_top10 is near-certain for both strategies (> 0.85), but is_top3 strongly differentiates them.
- Agree cases dominate mid-grid starters (P8–P15) where both targets have meaningful spread across strategies.

**The engineering rule of thumb from this analysis:**

> When P(top10) > 0.85 under both strategies, stop using is_top10 as the decision criterion — it has saturated. Switch to is_top3 as the decisive target. This is exactly the situation where the expansion target pays off.
