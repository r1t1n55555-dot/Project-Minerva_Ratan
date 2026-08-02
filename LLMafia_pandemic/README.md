# MINERVA — Pandemic & Vaccine Diplomacy Simulator

A LangGraph multi-agent implementation of *The Refined Environmental Engine*.
Five nations, ten rounds, one round = ten days. Each nation is an autonomous LLM agent
that makes a public promise and then privately decides what to actually do.

## Contents

| File | What it is |
|---|---|
| `MINERVA_pandemic_simulation.ipynb` | **The deliverable.** Full engine, 43 cells, runs top to bottom in Colab. |
| `MINERVA_design_report.docx` | Design report — mechanics, scoring, verification results, reference run. |
| `minerva_source.py` | The same engine as a flat Python script, for running outside a notebook. |
| `minerva_results.png` | Six-panel chart: population, epidemic load, vaccinations, capital, trust, deaths. |
| `minerva_scores.png` | Final composite score bar chart. |
| `minerva_transcript.txt` | Readable transcript of a full ten-round game — every broadcast, promise, betrayal. |
| `minerva_transcript.json` | Same transcript as structured data, plus the config used. |
| `minerva_history.csv` | One row per country per round: population, capital, stockpile, sick, cured, deaths, trust. |
| `reference_run.log` | Raw console output of the reference run. |

## Running it

1. Open the notebook in Colab and run the first cell (installs `langgraph`, `requests`, `pandas`, `matplotlib`).
2. Paste a Groq key into `GROQ_API_KEY` in the configuration cell.
3. Run all.

With no key the notebook still runs end to end on deterministic heuristic agents, so it
never fails to produce output. Every tunable constant — round count, wave fraction, surge
multiplier, `D`, `K`, vaccine cost, tax rate, trust penalties, score weights, and the
nations themselves — lives in that single configuration cell.

**API cost:** 2 calls per living nation per round = 100 calls for a five-nation, ten-round
game, spaced by a 2-second rate limiter with automatic retry on HTTP 429.

## The core mechanic

```
wave = 0.10 x P_initial x m  +  K x currently_sick

m = 1.0  if roll <= alpha   (containment held, baseline wave)
m = 1.6  if roll >  alpha   (containment failed, surge)
```

Infrastructure protects against the **surge**, never against the baseline wave.
`D = 10 days = exactly one round`, so a citizen infected in round *r* dies at the end of
round *r* unless a vaccine reaches them. Ten rounds at ten percent is the entire nation —
hoard vaccines for a decisive final round and you arrive with no population left.

Two feedback loops make neglect compound: production is `R = R_max x (P/P_initial)`, and
tax income scales with the surviving population. Dead citizens are dead factory workers
*and* dead taxpayers.

## Scoring

```
score = 100 x (0.70 x survival + 0.15 x trust + 0.10 x capital + 0.05 x stockpile)
```

Stockpile carries the smallest weight on purpose: ending a pandemic with a full warehouse
means citizens died holding vials, so the scoring must not reward the hoarding the
environment exists to punish.

## Verification

Run offline with deterministic agents so the mechanics can be tested independently of any LLM:

| Scenario | Behaviour | Outcome |
|---|---|---|
| Neglect | No vaccination at all | All five nations annihilated |
| Hoarding | Cure 30%, save the rest | 9–17% of population survives |
| Selfish | Perfect self-care, zero aid | 55–91% survive; the weak collapse alone |
| Cooperative | Triage, partial aid, trade | 64–91% survive, no annihilations |

Also checked: malformed / single-quoted / prose-wrapped / unparseable LLM JSON all recover
or fall back without stopping a run; no negative population, capital, stockpile or sick
counts in any round; trust stays within 0–100; deaths never exceed initial population; and
both the last-nation-standing and total-annihilation termination branches fire correctly.

> **Note:** the reference run in the report and charts used heuristic agents (no API key
> was available at build time). Rerun with a Groq key and the leaderboard will differ —
> that is the point, since the agents are the only nondeterministic part of the system.
