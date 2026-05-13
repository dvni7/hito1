# PROMPTS.md — AI Interaction Log
**Capstone Hito 2 | F1 Race Strategy Advisor**

Extends Hito 1 PROMPTS.md. Documents AI-assisted reasoning specific to Hito 2 work.

---

## Interaction 3 — Expansion target selection and justification

**Context:**
Working on Hito 2. The assignment requires choosing an expansion target from: `is_top5`, `is_top3`, `finish_position` (regression), or `points` (regression). The Hito 1 decision context is a race engineer asking "which strategy maximises P(top10)?" — a points-scoring question. I need to choose a second target that adds decision value beyond `is_top10`, specifically one that can disagree with `is_top10` on at least one scenario and reveal a trade-off the single target cannot show.

**Prompts used:**
1. "In F1 race strategy, what is the decision context where is_top10 and is_top3 could give opposite recommendations? Give me a concrete example with grid position and strategy choice."
2. "For a binary classifier with 15.5% positive rate, is Platt scaling or isotonic regression more appropriate for calibration when calibration N ≈ 440? What are the known failure modes of Platt in low-positive-rate scenarios?"

**Output received:**
1. AI identified the saturation problem: when a driver starts P1–P3 with a front constructor, P(top10) approaches 0.95+ for all reasonable strategies — the target loses discrimination. P(top3) remains sensitive because podium probability depends on tyre phase and track position in a way that points-scoring does not. AI suggested: find a scenario where 1-stop preserves podium (track position) while 2-stop trades it for fresh tyres — the trade-off shows up in is_top3 but not is_top10. Suggested Qatar, Spain, or Austria 2024 as likely candidate races (high RBR/McLaren competitive context).
2. AI confirmed Platt scaling is appropriate for N ≈ 440 even with low positive rate. For very low positive rates (< 10%), Platt can over-compress probabilities near zero. At 15.5% positive rate, Platt should be reliable in the 0–0.4 range. Above 0.5, the calibration data is sparse (< 25 samples) — AI recommended reporting this as a documented limitation rather than switching to isotonic regression with insufficient data.

**Validation:**
- Scanned 889 test-set rows programmatically: 156 (17.5%) produced DISAGREE between is_top10 and is_top3 when comparing 1-stop vs 2-stop templates. This confirms the AI's prediction that disagreement cases exist and are concentrated at high-grid starters.
- Qatar 2024 was indeed the highest-magnitude DISAGREE case: is_top10 Δ = +0.002, is_top3 Δ = −0.286 — exactly matching the AI's predicted mechanism (is_top10 saturates, is_top3 remains sensitive).
- Calibration quality on test set: ECE ≈ 0.05 for is_top3, confirming Platt is adequate. Above-0.5 range flagged as limitation in `mitigations.md` risk 4.

**Adaptations made:**
- AI suggested Qatar or Austria — data confirmed Qatar 2024 has the strongest DISAGREE signal. Used Qatar rather than Austria.
- AI's Platt recommendation was for the generic case. I applied it as stated and documented the sparse-data limitation for predictions > 0.5 in `mitigations.md`.

**Final Decision:**
`is_top3` selected as expansion target. Justification: surfaces the podium trade-off that is_top10 cannot detect when P(top10) saturates near 1.0. Disagreement rate = 17.5% on 1-stop vs 2-stop comparisons. Qatar 2024 is the primary what-if scenario in `whatif_comparison.md`.

---

## Interaction 4 — Error analysis structure and failure mode hypotheses

**Context:**
Running the error analysis slice. I need to produce structured Brier scores by strategy_type, circuit_type, and weather_actual, and name concrete failure-mode hypotheses — not generic "the model has higher error in some cases." Assignment rubric says: "names specific contexts where the model fails (e.g., two-stop strategies at street circuits in wet races)."

**Prompts used:**
1. "For an F1 strategy model with no safety-car data, what are the specific race contexts where a strategy-based predictive model would fail most? Give me 3 concrete failure contexts with a mechanistic explanation."
2. "The cross-slice of one_stop × wet produces Brier 0.264, versus an overall Brier of 0.126. What is the most plausible F1-domain explanation for why 1-stop strategies in wet races are systematically harder to predict?"

**Output received:**
1. AI listed: (a) safety-car-induced pit stops at street circuits — the pit timing looks planned but isn't; (b) wet-to-dry transitions with 1-stop strategies — teams that gambled on dry conditions look like normal 1-stop races; (c) strategic 3-stop races driven by VSC — a 3-stop caused by VSC has different dynamics than a planned 3-stop. All three match the dataset limitations (no SC data, no VSC data).
2. AI explained: in wet races, a 1-stop strategy is typically a "dry-from-the-start" gamble. If the track dries on schedule, the driver keeps position; if it doesn't, they face a tire crisis. The binary outcome (gamble pays off vs doesn't) creates high variance that the pre-race features cannot capture — the model predicts a middle probability, is systematically wrong in both directions, and accumulates high Brier.

**Validation:**
- Failure mode 1 (one_stop × wet, Brier 0.264) confirmed in data. This is the exact "wet 1-stop gamble" mechanism AI described.
- Cross-slice two_stop × semi-street (Brier 0.159) matches failure mode (a) — SC-induced 2-stops at street circuits.
- AI's VSC 3-stop mechanism is plausible but not testable with current data (no VSC column). Noted as unverifiable hypothesis.

**Adaptations made:**
- AI mentioned "wet-to-dry transition" as a sub-type of the wet 1-stop failure. I did not have a column for transition races specifically — I used `weather_actual = wet` as a proxy and documented the limitation.
- AI's three failure modes are incorporated directly into `error_analysis.md` §6 priority table.

**Final Decision:**
Error analysis structure: three primary slices (strategy, circuit, weather) plus two cross-slices (strategy × weather, strategy × circuit). Four concrete failure modes documented in `error_analysis.md` with Brier values. The worst cell (one_stop × wet, Brier 0.264) is the headline finding.

---

## Interaction 5 — Confounding explanation and mitigation

**Context:**
The `leakage_audit.md` requires an explicit discussion of strategy confounding. I need to explain clearly why strategy_type is a `scenario_input` (not post-race leakage) while also acknowledging that it is confounded with car pace. This is a subtle distinction that the rubric specifically calls out.

**Prompts used:**
1. "In F1 strategy modeling, what is the difference between 'strategy as a user-controlled scenario input' and 'strategy as a confounded observational variable'? How do you handle this in a what-if comparison tool?"
2. "What are the 2-3 most important mitigations for strategy confounding in a race strategy advisory model that cannot afford full causal inference?"

**Output received:**
1. AI explained: the distinction is in the use, not the data. When you observe that team X ran a 2-stop and finished P4, you cannot know if the 2-stop caused P4 or if the team's underlying pace allowed both the 2-stop choice and the P4 finish. In a what-if tool, you declare the strategy as user-controlled and acknowledge that the model measures association, not causation. The tool remains useful as a benchmark comparison ("historically, drivers in this context who ran 2-stop got X% top-10 rate") but should not be stated as causal.
2. AI suggested: (a) propensity score weighting — estimate P(team chose 2-stop | context) and weight training samples; (b) domain restriction — only compare strategies that the team's pace profile historically supports; (c) explicit UI language that frames outputs as historical associations. All three are practical without full causal modelling.

**Validation:**
- Propensity weighting confirmed as standard approach in sports analytics literature for strategy confounding.
- AI's framing of "associative estimate vs causal prediction" is used verbatim in `whatif_comparison.md` interpretation and `leakage_audit.md` §2.

**Adaptations made:**
- AI recommended propensity score as mitigation 1 — included in `mitigations.md` risk 3 with implementation detail.
- AI's domain restriction idea is captured as mitigation 3.2 in `mitigations.md`.

**Final Decision:**
Confounding is documented as a structural limitation in `leakage_audit.md` §2. Three mitigations listed in `mitigations.md` risk 3. All AI recommendations incorporated. Language throughout uses "associative estimate" rather than "causal prediction."
