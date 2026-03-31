# Health Agent Ecosystem

A multi-agent AI system built to optimize personal health, training, and recovery — designed and built by Logan Matson using Claude Code.

## What This Is

Five interconnected AI pipelines that run weekly, share state via `pipeline-state.json`, and produce actionable outputs (meal plans, training schedules, risk forecasts, performance reports).

## Agents

| Agent | What It Does | Output |
|---|---|---|
| **Whoop Report** | 7-subagent orchestrator — analyzes monthly Whoop data | PDF analyst brief + interactive scorecard |
| **Neurologist Agent** | Weekly seizure risk forecaster based on strain, sleep, and HRV | GREEN/YELLOW/RED daily forecast |
| **Gordon Ramsay** | Autonomous meal planner — selects meals, calculates macros, pushes to Google Calendar | Interactive meal plan PWA |
| **Optimize Week** | Closed-loop weekly optimizer — runs all 5 pipelines in sequence | Full week plan: training + meals + risk |
| **Calendar Agent** | Training scheduler — reads Runna + GCal, places PPL lifts around the long run | GCal events |

## Architecture

All agents share state via [`pipeline-state.json`](pipeline-state.json). The Neurologist Agent reads training load from the Calendar Agent. Gordon Ramsay reads recovery context from the Whoop Report. Everything connects.

## Outputs

- [`outputs/whoop/`](outputs/whoop/) — Whoop scorecard (interactive HTML)
- [`outputs/meals/`](outputs/meals/) — Meal plan PWA
- [`outputs/screenshots/`](outputs/screenshots/) — GCal automation, grocery emails, meal prep views

## Skill Files

Each agent is defined as a Claude Code skill in [`skills/`](skills/). These are the actual prompt files that drive the agents.

## Portfolio Page

[View the full project showcase →](index.html)

---

Built with [Claude Code](https://claude.ai/code) · [@loganmatson](https://www.linkedin.com/in/loganmatson)
