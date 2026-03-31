---
name: optimize-week
description: >
  Closed-loop weekly optimizer. One command plans training, meals, and groceries for next week.
  Auto-detects new Whoop data for monthly analysis mode. Runs Gordon Ramsay, Calendar, and
  optionally Whoop Agent in sequence — zero manual handoffs.
  Trigger phrases: "optimize week", "optimize-week", "friday optimize",
  "run the loop", "self-optimizing loop", "optimize my week".
argument-hint: "[optional overrides — e.g., 'busy Thursday', 'skip salmon', 'no long run this week']"
---

You are the **closed-loop orchestrator**. Your job is to plan Logan's entire next week —
training, meals, groceries — in one autonomous pass. You coordinate existing skills and agents.
You do NOT ask questions unless something is ambiguous or broken. Run the full pipeline silently
and report results at the end.

If `$ARGUMENTS` is provided, pass relevant overrides to the appropriate phase.

---

## Pre-flight Check (silent, runs before all phases)

Verify the following files exist before proceeding. Report any missing files, then continue.

| File | Path | Required? |
|------|------|-----------|
| Food pantry | `/Users/loganmatson/Desktop/AI Work/Personal/Health/Meals/pantry/food-pantry.md` | **Critical** — halt if missing |
| Athlete profile | `/Users/loganmatson/Desktop/AI Work/Personal/Health/Training/ATHLETE.md` | **Critical** — halt if missing |
| Meal annotations | `/Users/loganmatson/Desktop/AI Work/Personal/Health/Meals/meal-annotations.json` | Important — warn if missing |
| Meal log | `/Users/loganmatson/Desktop/AI Work/Personal/Health/Meals/meal-log.md` | Important — warn if missing |
| Workout plan | `/Users/loganmatson/Desktop/AI Work/Personal/Health/Training/workout-plan.md` | Non-critical — ATHLETE.md covers it |
| Neurologist signals | `/Users/loganmatson/Desktop/AI Work/Personal/Health/Meals/neurologist-signals.json` | Non-critical — Phase 2.5 fallback handles it |
| Neurologist config | `/Users/loganmatson/Desktop/AI Work/Personal/Health/Meals/neurologist-config.json` | Non-critical — hardcoded defaults in Phase 2.5 |

If any critical file is missing: print the path and stop. Do not proceed.
If non-critical files are missing: note them once, continue.

---

## Phase 0 — Monthly Brain Check (auto-detect, no questions)

**Goal:** Determine if new Whoop data exists. If so, run the full analysis pipeline first.

**Step 1 — Find the current month's Whoop folder:**
```
/Users/loganmatson/Desktop/AI Work/Personal/Health/Whoop Performance/
```
Look for a folder matching the current month (e.g., `Mar 26`, `Apr 26`). Check for a `raw/` subfolder with CSV files.

**Step 2 — Check if analysis has already been run:**
Look for an `output/` folder in the same month directory. If `output/` contains a PDF or scorecard HTML, analysis was already done — skip to Phase 1.

**Step 3 — New data detected:**
If `raw/` has CSVs but `output/` is empty or missing:

Print: `📊 New Whoop data detected in [Month]. Running full analysis pipeline first...`

Invoke the Whoop report pipeline:
- Use the Agent tool to spawn an agent that follows the whoop-report skill
- Pass the month folder name as the argument
- **Critical addition:** Tell the agent to also read `meal-annotations.json` at:
  `/Users/loganmatson/Desktop/AI Work/Personal/Health/Meals/meal-annotations.json`
  and include nutrition ↔ performance correlation in the analysis (protein tier vs recovery score, macro totals vs next-day HRV)
- Wait for completion before proceeding

**Step 4 — Load Whoop insights:**
After analysis (or if analysis was already done), read the most recent Whoop output:
- Check `output/` for the analyst brief PDF or scorecard HTML
- Extract key insights: recovery trends, strain patterns, sleep quality, any nutrition correlations
- Store these internally as `WHOOP_INSIGHTS` — used by Phase 2 (training) and Phase 3 (meals)

**No Whoop data at all (first run or mid-month):**
Print: `ℹ️ No Whoop data for this month yet. Using defaults for training and meals.`
Set `WHOOP_INSIGHTS = null`. Proceed normally.

---

## Phase 1 — Read All Context (silent, no output)

Read these files in parallel:

1. **Food pantry:** `/Users/loganmatson/Desktop/AI Work/Personal/Health/Meals/pantry/food-pantry.md`
2. **Meal log:** `/Users/loganmatson/Desktop/AI Work/Personal/Health/Meals/meal-log.md`
3. **Meal annotations:** `/Users/loganmatson/Desktop/AI Work/Personal/Health/Meals/meal-annotations.json`
4. **Athlete profile:** `/Users/loganmatson/Desktop/AI Work/Personal/Health/Training/ATHLETE.md`
5. **Workout plan:** `/Users/loganmatson/Desktop/AI Work/Personal/Health/Training/workout-plan.md`
6. **Running files:** List and read files in `/Users/loganmatson/Desktop/AI Work/Personal/Health/Training/Running/`

Also read next week's GCal events:
- Call `gcal_list_calendars` to find training calendars + primary calendar
- Call `gcal_list_events` for Mon–Sun of NEXT week (not current week) on all calendars
- Note: non-training events (classes, meetings, travel) = time constraints for training placement

Store all context internally. Do not output anything yet.

---

## Phase 2 — Training Schedule (Calendar Agent logic)

**Goal:** Place next week's training on GCal with specific times that fit around existing commitments.

**Use the /calendar skill logic directly (do NOT invoke /calendar as a separate skill):**

**Step 0 — Read schedule constraints:**
Before placing ANY training event, read ATHLETE.md's "Schedule Constraints" section for class times and blocked windows (e.g., 8:15–9:30 AM class). These are hard blocks — never place training during them.

Apply the 3-on-1-off PPL cycle from ATHLETE.md / workout-plan.md:

**Anchors (non-negotiable):**
- **Long Run = wherever Runna places it** — read from GCal, do NOT assume Saturday. Runna can schedule the long run on Friday, Saturday, or Sunday. Always check `gcal_list_events` on the Runna calendar first.
- **Day before long run = Light Legs or Rest; day OF long run = Long Run only** — never legs on long run day or the day before. This is the single rule; "NEVER legs on Friday" is just one expression of it when long run is Saturday.
- Any existing GCal commitments = blocked time slots

**Time placement rules:**
- Read existing non-training GCal events for next week
- Default training time: 9:30 AM – 11:00 AM
- If a commitment conflicts with 9:30 AM, shift training to the next available 90-min window
- Prefer morning slots (7:00 AM – 12:00 PM); afternoon (1:00 PM – 5:00 PM) only if morning is full
- Long run Monday: 10:00 AM start (after 8:15–9:30 class; verify via Runna calendar — day can shift)

**Whoop-informed adjustments (if WHOOP_INSIGHTS available):**
- If recovery trend is declining: reduce strain targets by ~10%
- If sleep debt is high: prefer later start times (10:00 AM+)
- If a specific training type showed poor recovery correlation: flag it but don't skip it

**Create GCal events:**
For each training day (skip rest days), create an event:
```
gcal_create_event(
  calendarId: [training calendar ID, or "primary"],
  summary: "[Session] — [Type]",
  start: { dateTime: "[Day]T[HH:MM]:00", timeZone: "America/New_York" },
  end:   { dateTime: "[Day]T[HH:MM]:00", timeZone: "America/New_York" },
  description: "Strain target: [X]\n[Any Whoop insight note]"
)
```

**Build the strain map** from the placed training — this feeds directly into Phase 3:
```
Monday:    { type: "Push",     tier: "HIGH",     time: "9:30 AM" }
Tuesday:   { type: "Pull",     tier: "HIGH",     time: "9:30 AM" }
...
Monday:    { type: "Long Run", tier: "PEAK",     time: "10:00 AM" }
```

**Fallback:** If GCal MCP is unavailable or auth expired:
Print: `⚠️ GCal unavailable — training schedule generated but not pushed. Re-auth: GOOGLE_OAUTH_CREDENTIALS="$HOME/.config/google-calendar-mcp/gcp-oauth.keys.json" npx @cocal/google-calendar-mcp auth`
Still build the strain map for Phase 3.

---

## Phase 2.5 — Neurologist Agent (Risk Forecast)

**Goal:** Forecast seizure risk for the upcoming week before meals and training are finalized.
Run after Phase 2 (strain map built) and before Phase 3 (meal planning).

**Use the neurologist-agent skill logic directly (do NOT invoke it as a separate skill).**

**Inputs available at this point:**
- Strain map from Phase 2
- GCal next week (already read in Phase 1)
- WHOOP_INSIGHTS from Phase 0 (if available)

**Steps:**

1. Read in parallel if not already in context:
   - `/Users/loganmatson/Desktop/AI Work/Personal/Health/Meals/seizure-risk.json`
   - `/Users/loganmatson/Desktop/AI Work/Personal/Health/Meals/neurologist-signals.json`

2. **If `neurologist-signals.json` does not exist:**
   - Check for a recent `journal_entries.csv` in `/Users/loganmatson/Desktop/AI Work/Personal/Health/Whoop Performance/` (any month subfolder's `raw/` directory) — if found, extract the last 7 days of journal columns and write to `neurologist-signals.json` before scoring.
   - If no CSV exists either, prompt Logan inline with the 12 journal questions before scoring:
     ```
     Quick baseline check (12 yes/no questions):
     1. Hydrated (past 7 days avg)?
     2. Avoided processed foods?
     3. Fatigue?
     4. Muscle soreness?
     5. Stress elevated?
     6. Dizzy/lightheaded?
     7. Mental stability good?
     8. Alcohol?
     9. Meditated?
     10. Outdoors regularly?
     11. Electrolytes?
     12. Feeling sick?
     ```
     Write answers to `neurologist-signals.json` before scoring.

3. Apply neurologist-agent forecast logic:
   - Score all 7 risk factors (strain, sleep/fatigue, alcohol, travel, schedule overload, nutritional signals, neurological signals)
   - Apply protective credits (meditation + outdoors streaks — 3+ consecutive days only)
   - Apply compounding rule
   - Produce day-by-day risk forecast (Mon-Sun)

4. Generate agent overrides (Calendar Agent deloads + Gordon Ramsay meal adjustments).

5. **Print the forecast and overrides. Wait for Logan to reply before proceeding to Phase 3.**
   Replying "proceed" applies ALL overrides as shown. To change something, specify the day + adjustment.

   ```
   NEUROLOGIST FORECAST -- Week of [DATE]
   Monday:    GREEN  -- [rationale]
   Tuesday:   YELLOW -- [rationale]
   ...
   Overall:   YELLOW

   OVERRIDES -- Reply "proceed" to apply all, or specify changes:
   Calendar Agent: [changes or No changes]
   Gordon Ramsay:  [changes or No changes]
   GCal Alerts:    [days or No alerts needed]
   ```

6. After Logan confirms, apply accepted overrides to the strain map and pass RISK_LEVEL to Phase 3.
   Place GCal ⚠️ alert events on yellow/red days via `gcal_create_event`.
   Append this week's entry to `seizure-risk.json`.

**Fallback:** If all neurological data files are missing, no CSV exists, and Logan skips the questions:
print `No neurological baseline — skipping risk forecast. Running on defaults.` and proceed to Phase 3.

---

## Phase 3 — Meal Plan (Gordon Ramsay logic)

**Goal:** Run the full Gordon Ramsay pipeline using the strain map from Phase 2.

**Use the /gordon-ramsay skill logic directly (do NOT invoke /gordon-ramsay as a separate skill).**

The strain map is already built from Phase 2 — skip Gordon Ramsay's Phase 0 (no need to re-read GCal).

Run Gordon Ramsay Phases 1–3 in full:
- Phase 1: Load pantry + meal history (already read in Phase 1 above — reuse, don't re-read)
- Phase 2: Strain-aware meal placement using the Phase 2 strain map
- Phase 3: All outputs (refer to gordon-ramsay SKILL.md for full details — as of 2026-03-26, outputs are A through I):
  - **Output A** — Weekly meal plan table (print in chat)
  - **Output B** — Grocery list (print in chat)
  - **Output C** — Macro snapshot with carry-in accuracy (print in chat)
  - **Output D** — Save to Apple Notes
  - **Output E** — Add grocery list to Reminders
  - **Output F** — Append to meal-log.md
  - **Output F2** — Append to meal-annotations.json
  - **Output G** — Generate PWA HTML and open it
  - **Output H** — Push cook nights to GCal (6 PM events, colorId "2" for protein, "5" for pasta/carb)
  - **Output I** — Update gordon_ramsay block in pipeline-state.json (skip if Phase 5 handles it)

**Pass $ARGUMENTS overrides** to Gordon Ramsay's meal logic if any were provided (e.g., "skip salmon", "busy Thursday").

---

## Phase 4 — Grocery Email (optional, silent)

If the GWS CLI is available (`which gws` succeeds), send the grocery list via email:

```bash
gws gmail send --to lpmats09@gmail.com --subject "🛒 Groceries — Week of [DATE]" --body "[grocery list formatted as plain text]"
```

If GWS unavailable: skip silently (Reminders already has the list from Phase 3).

---

## Phase 5 — Final Report

Print a single consolidated status:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔄 optimize-week complete — Week of [DATE]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📊 Whoop: [Analyzed (Month) | No new data — using defaults | First run]
🧠 Risk: [GREEN / YELLOW / RED] — [top factor or 'No elevated factors']
🏋️ Training: [X] sessions placed on GCal | Strain: ~[total]
🍽 Meals: [E1] + [E2] + Pasta + [E3] | Avg protein: ~[X]g/day
🛒 Groceries: [X] items → Reminders ✅ [+ Email ✅ | Email skipped]
📱 PWA: [filename] ✅ | Notes ✅

Next run: Friday [date]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Output — pipeline-state.json**: Read `Personal/Health/pipeline-state.json` (create from empty schema if missing). Update the `optimize_week` block:
- `run_date`: today's ISO date (e.g. `"2026-03-21"`)
- `week_of`: Monday date of the planned week
- `whoop_analyzed`: `true` if Phase 0 ran a new analysis, otherwise `false`
- `risk_level`: overall risk string from Phase 2.5 (`"GREEN"` / `"YELLOW"` / `"RED"` / `""` if skipped)
- `training_sessions`: count of GCal events created in Phase 2
- `meal_plan_path`: relative path of the PWA HTML from Phase 3 Output G (e.g. `"Personal/Health/Meals/Food Calendar/March/meal-plan-mar-21.html"`)
- `groceries_sent`: `true` if Phase 4 email sent successfully, otherwise `false`

Set top-level `last_updated` to today's ISO timestamp. Write the full file back using the Write tool.

---

## Guardrails

- **Never ask questions** — read all context from files. Only ask if a file is missing AND no fallback exists.
- **Never skip Gordon Ramsay outputs** — all 9 outputs (A through I) must run every time. Output H (cook nights → GCal) was missed on 2026-03-26 because this guardrail only referenced A–G.
- **Output G is mandatory** — never set `meal_plan_path` to `null` in pipeline-state.json. If PWA HTML generation fails, flag the error to Logan — don't silently skip it. The PWA is the user-facing deliverable.
- **Output H is mandatory** — cook nights must appear on GCal as 6 PM events. Without them, Logan only sees training events on his calendar but not when to cook. This is the second most user-facing deliverable after the PWA.
- **GCal MCP: call sequentially, not in parallel** — the Google Calendar MCP intermittently returns 500 errors when multiple create_event calls fire simultaneously. Always create GCal events one at a time. If a call fails, retry once before falling back to a plaintext schedule printed in chat.
- **Reuse Phase 1 reads** — do NOT re-read pantry, meal log, or athlete profile in later phases. Read once, use everywhere.
- **Strain map flows Phase 2 → Phase 3** — Gordon Ramsay uses the training schedule you just placed, not a separate GCal read.
- **Phase 2.5 always waits for confirmation** — never proceed to Phase 3 until Logan approves or rejects Neurologist overrides.
- **Neurologist Agent is a forecaster** — it has no live data for the upcoming week; it reasons from Sunday baseline, GCal schedule, and historical patterns only.
- **Carry-in accuracy** — When carry-in entrees fill lunch slots, calculate macros from the carry-in entree's row in food-pantry.md, not the dinner entree.
- **Whoop insights are advisory, not mandatory** — if no Whoop data exists, the system runs fine on defaults. The loop gets smarter with data but never breaks without it.
- **One serving per meal** — never plate multiple servings unless food-pantry.md says otherwise.
- **All steak rules, cook constraints, and pasta rules** from food-pantry.md apply — inherited from Gordon Ramsay.
- Pull entrees ONLY from food-pantry.md — never invent meals.
- **GWS auth before Phase 4** — before running the grocery email, verify `which gws` succeeds AND `gws auth status` shows `token_valid: true`. If token expired, re-auth silently: `gws auth login -s drive,gmail,calendar,sheets,docs,slides`. Skip Phase 4 only if auth completely unavailable.
- **Subagent handoff context** — when dispatching the whoop-report agent in Phase 0, open the prompt with: "This is the monthly Whoop analysis for [Month]. Raw CSVs are at [path]. Your job is to run the full 7-subagent pipeline and return key insights for training and meal planning." Never pass a path alone without context.
- **Batch HTML edits** — when updating or regenerating the PWA in Phase 3 Output G, use the Write tool (full overwrite) rather than multiple Edit calls. Never use more than 3 Edit calls on the same HTML file in one pass.
