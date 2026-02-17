# Fitness Decision Agent

A personal AI coach that looks at your body data every day and tells you exactly how hard to train.

---

## The Problem

Training consistently is easy. Training *intelligently* is hard.

Most people either:
- Train too hard when their body isn't recovered (leads to injury, stalled progress)
- Train too easy when they're actually ready to push (leaves gains on the table)

The issue is that knowing which situation you're in requires synthesizing a lot of noisy signals — your sleep last night, your heart rate variability, how much you've trained this week, what your strength trend looks like over the past month. Doing this correctly every day, by hand, is unrealistic.

That's what this project automates.

---

## What It Does

Every morning, the agent:

1. **Pulls your body data** from Whoop and Oura (HRV, sleep score, recovery score, resting heart rate)
2. **Computes your readiness** — a single score that combines all signals into an honest picture of how recovered you are
3. **Checks your training history** — how hard have you trained this week? This month? Are you trending toward injury?
4. **Prescribes today's workout** — specific sets, reps, weight, and how hard to push (expressed as "reps in reserve")
5. **Explains every decision** — you always know *why* it prescribed what it did
6. **Learns over time** — as you log sessions and outcomes, the system gets better at predicting what works for *your* body specifically

---

## Example Output

```
TODAY'S PRESCRIPTION — Lower (Squat focus)
───────────────────────────────────────────
  4 × 5 @ 102.5kg
  Stop 2 reps before failure (RIR 2)

  Why: Recovery is 74/100. HRV is slightly above your baseline (+0.4σ).
  Training load this week is healthy (AC ratio: 0.91). This is a solid
  moderate day — good time for consistent work, not a PR attempt.
───────────────────────────────────────────
```

If your recovery is poor:

```
TODAY'S PRESCRIPTION — Rest
───────────────────────────────────────────
  Take the day off.

  Why: Recovery 31/100. HRV is 2 standard deviations below your baseline.
  You've trained hard 5 days in a row (AC ratio: 1.7 — above injury
  threshold). Your body needs recovery, not more stress.
───────────────────────────────────────────
```

---

## The Technical Core

There are three interesting problems this project solves:

**1. Signal noise**
Raw HRV numbers are meaningless without context. A 55ms reading means nothing unless you know your baseline is 65ms — then it means you're 1.5 standard deviations below normal, which is significant. The system normalizes every signal to your personal history so comparisons are meaningful.

**2. Delayed feedback**
You don't know if today's workout was the right call until tomorrow (how did you recover?) or even next month (is your strength trending up?). This makes it a reinforcement learning problem — the system has to learn from outcomes that arrive late, not immediately.

**3. Safety constraints**
Unlike most ML systems, a bad output here has real physical consequences. Certain rules — don't train when your acute:chronic load ratio exceeds 1.5, don't train the same muscle group within 48 hours — are hard-coded and can never be overridden by the model, no matter what the data says.

---

## How It's Built

```
Your Wearables (Whoop, Oura)
        ↓
  Data Ingestion      pulls and normalizes your daily signals
        ↓
  State Model         computes readiness score + injury risk metrics
        ↓
  Policy Engine       decides today's training prescription
        ↓
  Guardrail Layer     safety checks that always run last
        ↓
  Prescription        what to do today, with explanation
        ↓
  [You train]
        ↓
  Feedback Logger     you log what actually happened
        ↓
  Reward Model        scores how good the prescription was (delayed)
        ↓
  RL Pipeline         uses that feedback to improve future prescriptions
```

The policy starts as a simple rule-based system (high recovery → train hard, low recovery → back off). Over time, as you accumulate logged sessions, it gets replaced by an ML model trained on your own data — one that learns your personal response patterns rather than relying on generic population averages.

---

## Key Concepts (Glossary)

**HRV (Heart Rate Variability)** — The variation in time between your heartbeats. Higher is generally better and indicates your nervous system is recovered. This is the single most informative recovery signal.

**HRV z-score** — How far your HRV today is from your personal average, measured in standard deviations. More useful than the raw number because it's relative to *your* baseline, not population norms.

**AC Ratio (Acute:Chronic Load Ratio)** — Your training load this week divided by your average training load over the last 4 weeks. Above 1.5 = elevated injury risk. Above 2.0 = high injury risk. This is a clinically validated metric used in sports science.

**RIR (Reps in Reserve)** — How many more reps you could have done before failure. RIR 0 = you hit failure. RIR 3 = you had 3 reps left. This is how the system expresses training intensity without needing to know your exact 1RM.

**e1RM (Estimated 1-Rep Max)** — Your estimated maximum for a single rep, calculated from your working sets using the Epley formula. Used to track strength trend over time.

**Offline RL** — A type of reinforcement learning where the model learns from historical data rather than live experimentation. We use this because "experimenting" with bad workout prescriptions has real physical consequences — we can't afford live exploration.

**MCP (Model Context Protocol)** — The standard we use to connect the agent to external data sources (Whoop, Oura, your training log). Each data source is wrapped in an MCP server. The agent calls tools through MCP rather than hitting APIs directly, which keeps things modular.

---

## Glycogen Proxy Algorithm

The system computes a glycogen status score from 0 (depleted) to 1 (fully loaded) using two different paths:

**Path 1 (tracked data):** If you log carb intake, the system uses actual grams consumed in the last 24 hours relative to a personal threshold (calibrated to your bodyweight and training volume). Higher intake = higher glycogen status.

**Path 2 (inferred from proxies):** If carbs aren't tracked, the system estimates depletion by accumulating evidence:

- Recent training volume significantly above baseline adds depletion
- Multiple consecutive hard training days adds depletion
- Extended training block without a deload adds depletion
- Sustained caloric deficit (especially over 300kcal) adds significant depletion

The proxy approach starts at 100% assumed glycogen and subtracts based on how many depletion signals are present. Each signal contributes a fractional penalty.

**Why this works for RL:**

When you train the model on your tracked data, it sees both the direct carb numbers AND all the proxy features in the same trace. It learns: "when carbs are low, performance drops" but also "when carbs are low, these other patterns are usually present too."

Later, when someone doesn't track carbs, the model sees missing carb data but recognizes the proxy pattern it learned from your traces. It's learned the shape of glycogen's effect and can apply it indirectly.

**Integration into readiness:**

Glycogen status gets a 10% weight in the overall readiness score calculation. Recovery signals get 30%, HRV gets 25%, sleep gets 20%, load management gets 15%, glycogen gets 10%. This makes it meaningful but not dominant — a fully depleted glycogen score (0) drops readiness by 10 points, which is enough to shift prescription from heavy to moderate but not enough to force rest on its own.

---

## Project Structure

```
fitness-agent/
├── data/
│   ├── ingestion/        # Whoop + Oura API connectors
│   ├── state/            # Converts raw data into AthleteState
│   └── store/            # Persistent storage (SQLite)
│
├── mcp_servers/          # One MCP server per data source
│   ├── wearable_server.py
│   ├── nutrition_server.py
│   ├── calendar_server.py
│   └── performance_server.py
│
├── policy/               # The brain — maps state → prescription
│   ├── rule_based.py     # v0 baseline (start here)
│   ├── policy_network.py # v1 ML policy (after enough data)
│   └── guardrails.py     # Hard safety constraints
│
├── reward/               # Scores past prescriptions, trains ML policy
├── agent/                # Orchestrates everything end-to-end
├── evals/                # Test suite — 15 scenarios covering edge cases
└── notebooks/            # Training runs, analysis, backtest results
```
## Tech Stack

This project is built primarily in Python, with an emphasis on clean data modeling, reproducible evaluation, and a safe path from a rule-based policy to a learned policy.

## Core language
  - Python
  
## Logging
  - Python logging - used throughout the system, so every decision is traceable (error-based logging)
  Supabase event records — used to store structured history (state snapshots, prescriptions, guardrail triggers, and outcomes)
  
## Data modeling
  Pydantic
    - wearable signals
    - training sessions
    - computed athlete state
    - prescriptions and explanations
     
## Database
  Supabase
    - daily wearable metrics (Whoop + Oura)
    - training history and session logs
    - computed state snapshots
    
## Numerical computing
  - NumPy (rolling baselines, normalization, readiness scoring, trend features (fatigue + performance))
    
## Machine learning
  - PyTorch (used to train the model once enough logged sessions exist)
  - scikit-learn (Used for quick baselines, evaluation metrics, and early experiments before moving fully into PyTorch)

## Testing
  - Pytest
## Agents
  - Claude — used for:
      - generating clear natural-language explanations
      - summarizing trends over time
      - self-reflection / improvement loops

guardrail violations
---

## Current Status

| Milestone | Status |
|---|---|
| M1: Data ingestion + state model | 🔲 Not started |
| M2: Rule-based policy + guardrails | 🔲 Not started |
| M3: MCP servers | 🔲 Not started |
| M4: Reward model + offline RL training | 🔲 Not started |
| M5: Self-reflection loop | 🔲 Not started |
| M6: Eval dashboard | 🔲 Not started |
| M7: Live Whoop/Oura connectors | 🔲 Not started |

We build in this order intentionally. The rule-based policy (M2) must exist before the ML policy (M4) — it's the baseline we're trying to beat, and it's the safe fallback if the ML model regresses.

---

## Getting Started

```bash
git clone https://github.com/your-username/fitness-agent
cd fitness-agent
pip install -r requirements.txt
cp .env.example .env   # fill in your API keys
python -m data.store.migrations.init
```

You'll need:
- A Whoop account with Developer API access (free at developer.whoop.com)
- An Oura account with a Personal Access Token (free at cloud.ouraring.com)
- An Anthropic API key

---

## Questions

For the full technical spec — architecture diagrams, implementation details, code samples, and design decision rationale — see [`SPEC.md`](./SPEC.md).

