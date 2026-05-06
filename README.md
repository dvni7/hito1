# F1 Race Strategy Advisor — Hito 1

**Module 5 — Unit IV Capstone | Team deliverable**

## What this is

Hito 1 establishes the problem framing and baseline model for a scenario-based F1 race strategy tool. The model outputs P(top10 | strategy_scenario) — the calibrated probability that a driver finishes in the top 10 given a specified pit-stop strategy.

## Repository layout

```
Hito_1/
├── data/
│   └── f1_strategy_race_level.csv   # race-level dataset (2019–2024)
├── framing.md                        # 7-section problem framing document
├── hito1_baseline.ipynb              # executable baseline notebook
├── PROMPTS.md                        # AI interaction log (6-field standard)
└── README.md                         # this file
```

## Install

Python 3.9+ required.

```bash
pip install pandas numpy scikit-learn matplotlib
```

No other dependencies.

## Run

```bash
# From the Hito_1/ directory:
jupyter notebook hito1_baseline.ipynb
# Then: Kernel → Restart & Run All
```

The notebook runs end-to-end from a clean clone with no manual steps.

## Where to find what

| Artefact | Location |
|----------|----------|
| Decision context + target justification | `framing.md` § 1–2 |
| Heuristic baseline rationale | `framing.md` § 3 |
| What-if scenarios (Monaco 2024, Monza 2023) | `framing.md` § 4 |
| Dataset limitations (2 of 5 acknowledged) | `framing.md` § 5 |
| Experiment plan for Hito 2 | `framing.md` § 6 |
| Leakage audit (pre-race vs scenario vs audit) | `hito1_baseline.ipynb` — Cell "Leakage Audit" |
| Brier score, log loss, calibration curve | `hito1_baseline.ipynb` — Cell "Evaluation" |
| AI interaction documentation | `PROMPTS.md` |

## Locked design decisions

| Decision | Value |
|----------|-------|
| Target | `is_top10` |
| Train | 2019, 2020, 2021 |
| Calibration | 2022 |
| Test | 2023, 2024 |
| Reference Brier (floor) | 0.208 (grid-rule) |
| Reference Brier (target) | 0.132 (docent model) |

## Baseline results (test set 2023–2024)

| Model | Brier | Log Loss | ROC-AUC |
|-------|-------|----------|---------|
| Grid-rule reference (bare minimum) | 0.2080 | — | — |
| **Heuristic (qpos buckets)** | **0.1699** | 0.5233 | 0.8227 |
| **LR + Platt Calibration** | **0.1735** | 0.5228 | 0.8680 |
| Docent model (target to beat) | 0.1320 | — | 0.8920 |

Both baselines beat the grid-rule floor (Brier 0.208). Gap to docent model (0.132) is the target for Hito 2 experiments.
