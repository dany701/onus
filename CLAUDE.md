# CLAUDE.md — Fitness Decision Agent (onus)

## Project Overview

An AI-powered personal fitness coach that ingests daily wearable data (Whoop, Oura), computes an athlete readiness score, and prescribes today's training with full reasoning. The system starts as a rule-based policy and progressively transitions to an offline-RL model trained on the user's own logged outcomes.

---

## Architecture

```
Wearables (Whoop, Oura)
        ↓
  Data Ingestion      pulls and normalizes daily signals
        ↓
  State Model         computes readiness score + injury risk metrics
        ↓
  Policy Engine       maps state → training prescription
        ↓
  Guardrail Layer     hard safety constraints (always runs last)
        ↓
  Prescription        what to do today, with explanation
        ↓
  [User trains]
        ↓
  Feedback Logger     logs actual session outcomes
        ↓
  Reward Model        scores how good the prescription was (delayed)
        ↓
  RL Pipeline         uses feedback to improve future prescriptions
```

---

## Project Structure

```
fitness-agent/
├── data/
│   ├── ingestion/        # Whoop + Oura API connectors
│   ├── state/            # Converts raw data into AthleteState
│   └── store/            # Persistent storage (Supabase/SQLite)
│
├── mcp_servers/          # One MCP server per data source
│   ├── wearable_server.py
│   ├── nutrition_server.py
│   ├── calendar_server.py
│   └── performance_server.py
│
├── policy/               # Maps state → prescription
│   ├── rule_based.py     # v0 baseline
│   ├── policy_network.py # v1 ML policy (after enough data)
│   └── guardrails.py     # Hard safety constraints
│
├── reward/               # Scores past prescriptions, trains ML policy
├── agent/                # Orchestrates everything end-to-end
├── evals/                # 15-scenario test suite covering edge cases
└── notebooks/            # Training runs, analysis, backtest results
```

---

## Tech Stack

- **Language:** Python
- **Data modeling:** Pydantic (wearable signals, AthleteState, prescriptions)
- **Database:** Supabase (daily metrics, training history, state snapshots)
- **Numerical computing:** NumPy (rolling baselines, normalization, AC ratio, trend features)
- **ML:** PyTorch (policy network), scikit-learn (baselines and evaluation metrics)
- **Testing:** Pytest
- **Logging:** Python `logging` (structured, error-based) + Supabase event records
- **Agent/LLM:** Claude API (natural-language explanations, trend summaries, self-reflection loops)
- **Integration:** MCP servers — one per data source

---

## Key Concepts

- **HRV z-score:** How far today's HRV is from the user's personal rolling average (in standard deviations). Always normalized to personal baseline, never population norms.
- **AC Ratio (Acute:Chronic Load Ratio):** Training load this week / average load over last 4 weeks. Hard-coded thresholds: >1.5 = elevated injury risk, >2.0 = high risk.
- **RIR (Reps in Reserve):** How many reps short of failure to stop. This is how training intensity is expressed without requiring a known 1RM.
- **e1RM:** Estimated 1-rep max calculated via the Epley formula from working sets.
- **Glycogen proxy:** Score 0–1 computed from carb tracking (direct) or proxy signals (consecutive hard days, caloric deficit, high training volume). Weighted 10% in readiness.
- **Readiness score weights:** Recovery 30%, HRV 25%, sleep 20%, load management 15%, glycogen 10%.
- **Offline RL:** Policy learns from historical logged sessions, not live experimentation. Required because bad prescriptions have real physical consequences.

---

## Hard Constraints (Never Override)

These rules in `guardrails.py` are **not suggestions** — they are safety constraints that always run after the policy and cannot be overridden by the model under any circumstances:

1. Never prescribe training when AC ratio > 1.5 (elevated injury risk)
2. Never train the same muscle group within 48 hours
3. If readiness < 30, prescription must be rest
4. Guardrail violations must always be logged to Supabase

When modifying the policy engine, **do not route around guardrails**. If a guardrail seems overly restrictive, discuss with the user before changing thresholds.

---

## Build Order (Milestones)

| Milestone | Description |
|---|---|
| M1 | Data ingestion + state model |
| M2 | Rule-based policy + guardrails |
| M3 | MCP servers |
| M4 | Reward model + offline RL training |
| M5 | Self-reflection loop |
| M6 | Eval dashboard |
| M7 | Live Whoop/Oura connectors |

**Always build in this order.** The rule-based policy (M2) must exist before the ML policy (M4) — it is the baseline we're comparing against and the safe fallback if the ML model regresses.

---

## Development Conventions

- Use Pydantic models for all data structures crossing module boundaries — no raw dicts
- Every decision must be logged (what state was seen, what was prescribed, why)
- All Supabase writes should be wrapped and not silently swallowed on failure
- Tests live in `evals/` and should cover the 15 core edge-case scenarios
- The Claude API is used only for natural-language output — it does not make policy decisions
- Do not use Claude to make or override training prescriptions; the policy engine owns that

---

## Environment Variables

Required in `.env` (never commit):
- `ANTHROPIC_API_KEY`
- Whoop API credentials
- Oura Personal Access Token
- Supabase URL + service key
