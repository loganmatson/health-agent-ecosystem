# Health Agent Ecosystem

> A fully autonomous, multi-agent AI system that optimizes training, recovery, nutrition, and neurological health — built end-to-end in Claude Code by Logan Matson.

---

## The Idea

Most health apps give you data. This system acts on it.

Every week, five AI agents run in sequence. They read my Whoop biometrics, place my training sessions, forecast my seizure risk, plan my meals, and push everything to Google Calendar — without me touching a single input. The agents share state, inform each other's decisions, and produce outputs I actually use.

This isn't a demo. It runs weekly. The outputs in this repo are real.

---

## Why I Built It

I had brain surgery two years ago after eight years of epileptic seizures. I've been seizure-free since, but the risk doesn't disappear — it's managed. Stress, sleep deprivation, overtraining, and poor nutrition are all triggers. I'm also training for a marathon.

I built this system because I needed something that could hold all of those variables at once and surface a clear picture of what my week should look like. No app did that. So I built the agents myself.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    /optimize-week                        │
│           Closed-loop weekly orchestrator                │
└────────────────────────┬────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
  ┌─────────────┐ ┌─────────────┐ ┌──────────────────┐
  │Whoop Report │ │  Calendar   │ │  Gordon Ramsay   │
  │  (monthly)  │ │   Agent     │ │  Meal Planner    │
  └──────┬──────┘ └──────┬──────┘ └────────┬─────────┘
         │               │                 │
         └───────────────┼─────────────────┘
                         ▼
               ┌──────────────────┐
               │ pipeline-state   │
               │     .json        │
               └────────┬─────────┘
                        │
                        ▼
              ┌──────────────────┐
              │ Neurologist      │
              │ Agent            │
              │ (reads all feeds)│
              └──────────────────┘
```

`pipeline-state.json` is the connective tissue. Every agent writes its output there. Every downstream agent reads from it. The Neurologist Agent is last in the chain because it needs everything — training load, recovery trends, meal timing — before it can forecast risk.

---

## The Agents

### Whoop Report Agent
**What it does:** Ingests raw Whoop CSVs (sleep, recovery, strain, journal entries) and dispatches 7 parallel subagents to clean, analyze, and visualize the data. Produces a PDF analyst brief and an interactive HTML scorecard.

**How it works:** Each subagent handles one slice — data cleaning, statistical modeling, visualization, PDF generation, scorecard rendering. They run in parallel and hand off outputs via file paths. The orchestrator assembles the final deliverables.

**Key outputs:**
- Recovery trend analysis with effect sizes (Cohen's d)
- HRV as independent predictor of recovery (partial r=0.80 controlling for sleep)
- Intervention ranking: which behaviors move the needle and by how much
- → [`outputs/whoop/scorecard-mar-26.html`](outputs/whoop/scorecard-mar-26.html)

---

### Neurologist Agent
**What it does:** Reads the previous week's Whoop journal, next week's training calendar, historical seizure-risk patterns, and the strain map from the Calendar Agent. Outputs a GREEN/YELLOW/RED daily risk forecast for the upcoming week.

**How it works:** Risk is modeled across four domains — physical strain, sleep debt, nutritional gaps, and schedule stress. Each domain is scored independently, then combined into a composite daily risk level. The agent flags specific days and recommends mitigations (e.g., swap a hard session, add a rest day).

**Why it matters:** This is the agent I built the whole system around. A bad week of training + poor sleep + travel is a real risk vector. Having a forecast surfaced on Sunday before the week starts changes how I plan.

---

### Gordon Ramsay — Meal Planner
**What it does:** Reads the pantry file, last week's meal log, recovery context from the Whoop pipeline, and the training calendar. Selects 4–5 entrees, builds a full week meal plan with per-day macro targets, generates an interactive PWA, and pushes cook nights to Google Calendar.

**How it works:** Meal selection is constraint-driven — protein targets shift based on training load (PEAK days get +1.5x carbs), carry-in entrees are tracked across weeks, and cook nights are placed around heavy training days. The PWA has a 3-tab interface: Plan, Cook, Groceries.

**Key outputs:**
- Interactive meal plan PWA with per-day macro bars
- GCal cook night events with color coding
- Grocery list emailed via GWS CLI
- → [`outputs/meals/meal-plan-mar-30.html`](outputs/meals/meal-plan-mar-30.html)

---

### Optimize Week — Orchestrator
**What it does:** Runs all five pipelines in a single command. Reads the current `pipeline-state.json`, determines what needs to run, dispatches agents in the correct order, and writes the final state when complete.

**How it works:** Each phase has a gate condition — if whoop data isn't fresh, the Whoop Agent runs first. If training isn't placed yet, Calendar Agent runs before Neurologist. The orchestrator enforces ordering and passes context between agents so nothing runs blind.

**The loop:** The system is genuinely closed. Whoop data informs training load. Training load informs seizure risk. Seizure risk informs meal timing. Meal timing feeds back into next week's recovery. One command, full week.

---

### Calendar Agent
**What it does:** Reads the Runna training calendar (source of truth for long runs), maps current GCal load, and places Push/Pull/Legs lifts around the long run without stacking hard sessions back-to-back.

**How it works:** Long run = Monday by default (the highest-strain session). PPL lifts fill the remaining days with built-in buffers. GCal MCP is called sequentially (parallel calls cause 500 errors) and events are written with correct color coding.

---

## Shared State — `pipeline-state.json`

Every agent reads and writes a single state file. This is what allows the system to be resumable, auditable, and composable.

```json
{
  "pipelines": {
    "optimize_week": { "risk_level": "GREEN", "training_sessions": 5 },
    "neurologist_agent": { "overall_risk": "GREEN", "forecast": { "2026-03-30": "GREEN" } },
    "gordon_ramsay": { "avg_protein_g": 169, "entrees": ["Chicken Breasts", ...] },
    "whoop_report": { "highlights": ["Recovery up +6 points to 57.6%", ...] }
  }
}
```

→ Full file: [`pipeline-state.json`](pipeline-state.json)

---

## Skill Files

Each agent is a Claude Code skill — a structured markdown prompt file that defines the agent's behavior, tool usage, output format, and handoff protocol.

| Skill File | Agent |
|---|---|
| [`skills/whoop-report/SKILL.md`](skills/whoop-report/SKILL.md) | Whoop Report |
| [`skills/neurologist-agent/SKILL.md`](skills/neurologist-agent/SKILL.md) | Neurologist Agent |
| [`skills/gordon-ramsay/SKILL.md`](skills/gordon-ramsay/SKILL.md) | Gordon Ramsay |
| [`skills/optimize-week/SKILL.md`](skills/optimize-week/SKILL.md) | Optimize Week |
| [`skills/calendar/SKILL.md`](skills/calendar/SKILL.md) | Calendar Agent |
| [`skills/gws-agent/SKILL.md`](skills/gws-agent/SKILL.md) | GWS Automation |

---

## Outputs

| Output | Description | Link |
|---|---|---|
| Whoop Scorecard | Interactive HTML — recovery trends, intervention rankings, HRV analysis | [View](outputs/whoop/scorecard-mar-26.html) |
| Meal Plan PWA | 3-tab interactive app — Plan / Cook / Groceries, per-day macro bars | [View](outputs/meals/meal-plan-mar-30.html) |
| GCal Automation | Screenshot — cook nights + training sessions placed automatically | [View](outputs/screenshots/GCal%20Screenshot.png) |
| Grocery Email | Auto-generated grocery list sent via GWS CLI | [View](outputs/screenshots/Groceries%20Email.png) |
| Meal Prep Views | Cooking tab, daily log, groceries tab from the PWA | [View](outputs/screenshots/) |

---

## Stack

- **Claude Code** — agent runtime, skill system, subagent orchestration
- **Whoop API** — raw biometric data (sleep, HRV, strain, recovery, journal)
- **Google Calendar MCP** — read/write training and cook night events
- **GWS CLI** — Gmail automation (grocery emails, digest delivery)
- **Python** — data cleaning, statistical analysis, PDF generation (pandas, scipy, matplotlib)
- **Netlify / GitHub Pages** — static hosting for HTML outputs

---

## Portfolio Page

[View the full interactive showcase →](index.html)

---

Built by [Logan Matson](https://www.linkedin.com/in/loganmatson) · Powered by [Claude Code](https://claude.ai/code)
