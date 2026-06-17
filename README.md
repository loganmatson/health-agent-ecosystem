# Health Agent Ecosystem

> A production multi-agent system that plans my training, meals, and seizure-risk forecast every week — nine Claude Code agents coordinated through shared state, a two-tier memory architecture, and a three-tier autonomy framework. Built end-to-end by Logan Matson.

**See it live:**
[**Nine-agent deep-dive →**](https://loganmatson.github.io/health-agent-ecosystem/health-ecosystem-showcase.html) · [Interactive showcase →](https://loganmatson.github.io/health-ecosystem-index/)

> The deep-dive page is the system's own overview — verbatim live state from `pipeline-state.json`, the `/optimize-week` walkthrough, the two-tier memory model, the silent-fallback incident report, and the three-tier autonomy framework. Every number on the page traces back to a real file in this repo.

---

## The Idea

Most health apps give you data. This system acts on it.

Every week, one command runs a chain of AI agents. They read my Whoop biometrics, place my training sessions, forecast my seizure risk, plan my meals, push everything to Google Calendar, and check their own work for drift — without me touching a single input. The agents share state, inform each other's decisions, and produce outputs I actually use.

This isn't a demo. It runs weekly, in production, on my own health. The state in this repo is the real state — including the days it forecasts RED.

---

## Why I Built It

I had brain surgery two years ago after eight years of epileptic seizures. I've been seizure-free since, but the risk doesn't disappear — it's managed. Stress, sleep deprivation, overtraining, alcohol, and travel are all triggers. I've also run three marathons in the last six months.

I built this system because I needed something that could hold all of those variables at once and surface a clear picture of what my week should look like. No app did that. So I built the agents myself — and then I had to learn what running them in production actually demands: observability, rollback paths, and the discipline to assume the failure you can't see is the one happening right now.

---

## System Architecture

```
                        ┌──────────────────────────────────┐
                        │          /optimize-week          │
                        │   master orchestrator (1 command)│
                        └──────────────────┬───────────────┘
                                           │
   ┌───────────┬───────────┬──────────────┼───────────────┬───────────────┐
   ▼           ▼           ▼              ▼                ▼               ▼
┌────────┐ ┌────────┐ ┌──────────┐ ┌─────────────┐ ┌─────────────┐ ┌──────────┐
│calendar│ │ whoop- │ │neurologist│ │  gordon-    │ │ meal-log-   │ │ fridge-  │
│        │ │ report │ │  -agent   │ │  ramsay     │ │  update     │ │  raid    │
│training│ │+7 sub- │ │ risk      │ │ meal planner│ │ mid-week    │ │ pantry   │
│ placer │ │ agents │ │ forecast  │ │  9 outputs  │ │ actuals     │ │ editor   │
└───┬────┘ └───┬────┘ └─────┬─────┘ └──────┬──────┘ └──────┬──────┘ └────┬─────┘
    │          │            │              │               │             │
    └──────────┴────────────┴──────┬───────┴───────────────┴─────────────┘
                                   ▼
                        ┌──────────────────────┐
                        │   shared state        │
                        │  pipeline-state.json  │  ◄── two-tier memory:
                        │  (atomic write)       │      managed store + local
                        └──────────┬────────────┘
                                   ▼
                        ┌──────────────────────┐         ┌──────────────────┐
                        │       alfred          │◄────────┤ managed-memory-  │
                        │ governance / drift    │         │     agent        │
                        │ detection / backup    │         │ safety & rollback│
                        └──────────────────────┘         └──────────────────┘
```

`pipeline-state.json` is the connective tissue. Every agent writes its block; every downstream agent reads from it. The neurologist runs late in the chain because it needs everything — training load, recovery trends, meal timing — before it can forecast risk. **Alfred** runs last as a watchdog: it never plans, only verifies. The whole thing is a closed loop — Whoop data informs training load, training load informs seizure risk, seizure risk informs meal timing, meal timing feeds back into next week's recovery.

---

## The Nine Agents

Eight live in the project; the orchestrator lives globally. Each is a `SKILL.md` — versioned instructions, guardrails, and a documented *Gotchas* section recording real production failures so no future agent session repeats them.

### optimize-week — master orchestrator
One command plans the entire week. Runs **Week Brief → Pre-flight → Phases 0, 0.5, 1, 2, 2.5, 3, 4, 5, 5.5** — training, risk forecast, meals, groceries, the atomic state write, and a governance check. The only skill permitted multi-level spawn chains, capped and enforced by an explicit spawning-rules document. It *inlines* the planning agents (calendar, neurologist, gordon-ramsay) into its own context to reuse a single round of file reads, and *spawns* only the heavyweight Whoop pipeline — spawn when isolation pays for itself, inline when shared context does.

### calendar — training scheduler
Reads existing calendar events first: if I've already built the week, those events **are** the strain map — no regeneration (source-of-truth check). Otherwise it places a 3-on / 1-off lift cycle around hard anchors — never lift on long-run day, never legs the day before. Writes a schema-locked state block where `session_map` keys must be ISO dates; day names break the validator.

### whoop-report — monthly biometric analysis (orchestrator of 7 subagents)
Explicitly an orchestrator: *"you do NOT do the analysis yourself."* Dispatches a Haiku data cleaner, then four parallel Sonnet analysts (descriptive stats, OLS regression + HRV mediator, dose-response, behavioral interventions), merges their JSON, then two parallel Sonnet renderers (interactive report + scorecard). Freezes a monthly **baseline snapshot** into the managed memory store for month-over-month diffing. Key findings from the live window: HRV is an independent predictor of recovery (partial correlation holds controlling for sleep), with intervention effects ranked by size.

### neurologist-agent — self-calibrating seizure-risk forecaster
The agent I built the whole system around. Scores **7 risk factors** against a **30-day baseline** of journaled signals, applies protective credits (3+ day meditation / outdoors streaks) and a compounding rule, and outputs a **GREEN / YELLOW / RED** forecast per day plus proposed overrides for the other agents. It is a *consumer* of the calibration weights — it can read them but is structurally forbidden from writing them. It always surfaces overrides and waits; it never silently modifies the training or meal plan.

### gordon-ramsay — autonomous meal planner
Strain-aware meal placement: the night before a peak day carb-loads, the peak-day dinner gets the highest protein tier, rest days eat leftovers. Nine outputs per run — plan table, grocery list, macro snapshot, notes, reminders, meal log, annotations bridge, a 3-tab interactive meal-plan app, and calendar cook nights. Carries a brand style lock — design agents may polish but never replace its palette. Pulls entrees *only* from the curated pantry file; it is forbidden from inventing meals.

### meal-log-update — mid-week actuals corrector
Reconciles plan vs. reality. Validates last week's carry-in against the annotations bridge, interviews me on deviations, rebuilds the actuals table, replans forward, flags grocery gaps and low-protein days. Sets an `actuals_updated` flag in state so downstream validators know the week reflects what was actually eaten, not what was planned.

### fridge-raid — guided pantry editor
The smallest agent, deliberately: a confirm-at-every-step editor for the pantry file — the single source of truth every meal decision flows from. It's the one place human curation enters the system, so it gets its own guarded skill rather than ad-hoc edits.

### alfred — governance agent
The watchdog. Flags pipelines stale beyond a threshold, validates every state block against required-field schemas, cross-references that the meal and orchestrator blocks agree on the same week, and runs a **silent-fallback detector** — comparing the managed store's timestamp against the local snapshot to catch a memory layer that has quietly stopped persisting. Hard rules: never writes shared state, never modifies a skill file, never deletes, never spawns. A watchdog, not a gate — if Alfred itself errors, the pipeline still completes.

### managed-memory-agent — safety & rollback for the memory tier
Owns STATUS / UPDATE / INSPECT / ROLLBACK / SNAPSHOT against the managed store. Rollback is fully reversible by design: the pre-migration text of every modified skill file is preserved in the registry, and a remote deletion must confirm success *before* any local file is touched. Baseline snapshots are never deleted — even by rollback.

---

## Memory Architecture — Two Tiers, One Interface

Every pipeline writes through a single module that writes the **Anthropic managed memory store first** and *always* writes the **local JSON regardless**.

| Tier | Role |
|---|---|
| **Managed memory store** | Survives machine loss, readable by any future agent session. One record per pipeline block plus frozen monthly baseline snapshots. Writes use a content-hash precondition for optimistic concurrency. |
| **Local fallback** (`pipeline-state.json`) | The original source of truth — the migration never touched it, so rollback is always possible. The write call returns `True` only if the managed write succeeded; `False` means local-only. An atomic batch write detects partial failures and names exactly which blocks didn't persist. |

### Incident report: the silent fallback

For three weeks, every managed-memory write returned `False`. Local state kept updating perfectly — pipelines ran, plans shipped, nothing looked wrong. The managed store sat frozen.

**Root cause:** Claude Code launched from the IDE doesn't inherit the terminal shell environment, so the API key — which lived in a file only interactive shells read — never reached the process. The memory module did exactly what it was designed to do: fell back gracefully, silently.

**The real bug wasn't the missing key. It was that the fallback worked too well.** Graceful degradation without observability isn't resilience — it's silent data loss with good manners.

**The fix had four independent layers**, shipped the same day it was caught:
1. The key moved to a file read by *all* shell invocations, including IDE launches.
2. The memory module gained its own key resolver — environment first, then parse the file directly — bulletproof regardless of launch path.
3. Every pipeline skill now *mandates* capturing the write's return value and surfacing a `False`.
4. Alfred gained the silent-fallback detector as a post-run check.

The same failure now has four chances to be caught — at the environment, in the library, in every calling agent, and by the watchdog. The postmortem lives *inside* the skill file, so every future agent that touches the module reads the failure mode before writing a line.

---

## Governance — The Three-Tier Autonomy Framework

Every action any agent takes is classified before it runs.

| Tier | Meaning | Examples |
|---|---|---|
| **Tier 1 — Autonomous** | Reversible, low blast radius, inside the agent's job. Runs without asking. | Computing the risk forecast; generating the meal-plan app; an agent writing its *own* state block. |
| **Tier 2 — Surface First** | Outward-facing or hard to reverse. Show the action, wait for explicit "go." | Every calendar write; **neurologist overrides** (the pipeline halts mid-run until I confirm); pushing biometrics to a public repo; large calibration-weight changes; all Alfred remediation. |
| **Tier 3 — Halt** | Stop fully, report, never continue on partial data. | API key unresolvable; managed store unreachable; corrupt shared state; a required input file missing. |

Two principles the calibration loop **cannot** override:
- **Clinical rules beat learned weights.** Alcohol is never downgraded by protective credits even if its learned weight drops; a specific dizziness signal forces a yellow baseline no multiplier can cancel. The system may learn *how sensitive to be* — not learn its way out of a safety rule.
- **Subagents inherit, never expand.** A subagent's autonomy ceiling is its parent's. A newly created agent runs Tier-2-everything until specific actions are individually promoted. No agent earns trust by default — including the governance agent itself.

---

## Cost Engineering — Model Tiering

The cheapest model that's correct, never cheaper — documented in the skill files, not folklore.

| Subagent | Function | Model tier |
|---|---|---|
| Data Cleaner | CSV parsing, normalization, anomaly flagging — no reasoning required | Haiku |
| Descriptive Stats | Means/SDs, weekly averages, period deltas | Sonnet |
| Regression + HRV Mediator | OLS model, partial correlations | Sonnet |
| Dose-Response + Modality | Strain→recovery fit, effect sizes between training types | Sonnet |
| Interventions + Correlations | Behavioral effects with a minimum-sample gate (small-n effect sizes are bias, not signal) | Sonnet |
| Report + Scorecard Renderers | Token-fill templates, compute month-over-month trends | Sonnet |

**Opus is banned from this pipeline** by a written rule: the run is large enough that Opus would multiply cost with no quality benefit. The one rule that cuts the other way: any subagent writing to shared state is **Sonnet minimum** — smaller-model truncation is a known corruption vector.

---

## Shared State — `pipeline-state.json`

Every agent reads and writes a single state file. This is what makes the system resumable, auditable, and composable. A dead pipeline (a disabled morning digest) is left in state as a documented tombstone rather than silently deleted — so no agent ever resurrects it by accident.

→ Full file: [`pipeline-state.json`](pipeline-state.json)

---

## Skill Files

Each agent is a Claude Code skill — a structured prompt file defining the agent's behavior, tool usage, output format, and handoff protocol.

| Skill File | Agent |
|---|---|
| [`skills/optimize-week/SKILL.md`](skills/optimize-week/SKILL.md) | Master orchestrator |
| [`skills/calendar/SKILL.md`](skills/calendar/SKILL.md) | Training scheduler |
| [`skills/whoop-report/SKILL.md`](skills/whoop-report/SKILL.md) | Biometric analysis (7 subagents) |
| [`skills/neurologist-agent/SKILL.md`](skills/neurologist-agent/SKILL.md) | Seizure-risk forecaster |
| [`skills/gordon-ramsay/SKILL.md`](skills/gordon-ramsay/SKILL.md) | Meal planner |
| [`skills/gws-agent/SKILL.md`](skills/gws-agent/SKILL.md) | Workspace automation |

---

## Outputs

| Output | Description | Link |
|---|---|---|
| Whoop Report — June | Latest interactive analyst report | [Source](outputs/whoop/whoop-report-jun-26.html) |
| Whoop Scorecard — March | Interactive — recovery trends, intervention rankings, HRV analysis | [Live](https://nuero-agent-logan.netlify.app/) · [Source](outputs/whoop/scorecard-mar-26.html) |
| Whoop Scorecard — February | Prior-month comparison | [Source](outputs/whoop/scorecard-feb-26.html) |
| Analyst Brief — March | Full statistical report: regression, effect sizes, protocol recommendations | [PDF](outputs/whoop/analyst-brief-mar-26.pdf) |
| Analyst Brief — February | February baseline | [PDF](outputs/whoop/analyst-brief-feb-26.pdf) |
| Validator Dashboard | Retrospective grading of past risk forecasts against actual biometrics | [Source](outputs/validator-dashboard/index.html) |
| Meal Plan App | 3-tab interactive — Plan / Cook / Groceries, per-day macro bars | [Live](https://meal-plan-mar30-logan.netlify.app/) · [Source](outputs/meals/meal-plan-mar-30.html) |
| Calendar Automation | Cook nights + training sessions placed automatically | [View](outputs/screenshots/GCal%20Screenshot.png) |
| Grocery Email | Auto-generated grocery list | [View](outputs/screenshots/Groceries%20Email.png) |
| Meal Prep Views | Cooking tab, daily log, groceries tab from the app | [View](outputs/screenshots/) |

---

## Stack

- **Claude Code** — agent runtime, skill system, subagent orchestration
- **Anthropic Managed Memory** — durable cross-session state, the system's nervous system
- **Whoop API** — raw biometric data (sleep, HRV, strain, recovery, journal)
- **Google Calendar MCP** — read/write training and cook-night events (sequential writes — a production guardrail against concurrency errors)
- **Workspace CLI** — Gmail automation (grocery emails, digest delivery)
- **Python** — data cleaning and statistical analysis (pandas, scipy, matplotlib)
- **GitHub Pages / Netlify** — static hosting for HTML outputs

---

## Live Showcases

- **[Nine-agent deep-dive →](https://loganmatson.github.io/health-agent-ecosystem/health-ecosystem-showcase.html)** — verbatim live state, the `/optimize-week` walkthrough, the memory incident, and the governance model. Served straight from this repo ([`health-ecosystem-showcase.html`](health-ecosystem-showcase.html)).
- **[Interactive showcase →](https://loganmatson.github.io/health-ecosystem-index/)** — the full system, visualized

---

Built by [Logan Matson](https://www.linkedin.com/in/loganmatson) · Powered by [Claude Code](https://claude.com/claude-code)
