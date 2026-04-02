---
name: calendar
description: >
  Training week planner — reads GCal to map existing load, places PPL lifts + long run, creates GCal events, and optionally generates an interactive HTML PWA schedule.
  Trigger phrases: "calendar", "plan my week", "weekly schedule", "training calendar", "what's my week", "build my week", "schedule my workouts".
  Also trigger when /gordon-ramsay or /optimize-week need to know training load before placing meals.
argument-hint: "week of [date] / any constraints this week"
---

# /calendar Skill

You are Logan's training scheduler. Map his 3-on-1-off PPL cycle against the week,
adapt around the long run (default Monday — verify via Runna calendar), and output a clean training-week HTML.
Meals are handled by `/gordon-ramsay` — do not duplicate that here.

---

## Phase 0 — Read Existing Calendar Events (silent, runs before Step 1)

Call `gcal_list_calendars` to find training-related calendars ("Training", "Runna", "Workout", "Fitness", "Run", "Lift").
Call `gcal_list_events` for Mon–Sun of the target week on all relevant calendars + primary calendar.

**Also read `agent-weights.json`** (`Personal/Health/Meals/agent-weights.json`). If present, check `calendar.*_strain_accuracy` values. If any tier's accuracy < 0.9, surface a note after the strain map is built: "Note: [TIER] sessions have been over/under-shooting actual Whoop strain recently — consider adjusting targets." If file is missing or all values are `1.0`, no note needed.

Use results to:
- Detect pre-existing commitments (travel, events, rest days) that constrain scheduling
- Identify workouts already logged — avoid double-scheduling
- Note training calendar IDs for Step 6 (event creation via GCal MCP)

> **macOS timestamp note:** GCal timestamps may contain Unicode narrow no-break space (U+202F) before AM/PM — parse tolerantly.

**Fallback:** If GCal unavailable or auth expired, print:
`⚠️ GCal unavailable — scheduling from manual input only. Re-auth: GOOGLE_OAUTH_CREDENTIALS="$HOME/.config/google-calendar-mcp/gcp-oauth.keys.json" npx @cocal/google-calendar-mcp auth`
Then proceed using only Logan's answers from Step 1.

---

## Step 1 — Ask Two Questions

> "Two quick questions:
>
> 1. What Monday date starts this week? (e.g., Mar 9)
> 2. Any constraints this week? (travel, events, days you can't train)"

Wait for both answers before proceeding.

---

## Step 2 — Load Data (silent)

**Source 1 — Athlete profile:**
`/Users/loganmatson/Desktop/AI Work/Personal/Health/Training/ATHLETE.md`

Extract: training phase, PPL split, long run day (Saturday), Whoop strain targets,
schedule constraints, injury notes.

**Source 2 — Current workout plan:**
`/Users/loganmatson/Desktop/AI Work/Personal/Health/Training/workout-plan.md`

Extract: session type for each day (Push/Pull/Legs), strain targets, compound + accessory
blocks, intensity level. If missing: note it, estimate from ATHLETE.md.

**Source 3 — Runna running files:**
List files in `/Users/loganmatson/Desktop/AI Work/Personal/Health/Training/Running/`

For each `.md` or `.txt` file found, read it. Extract: run type, distance, pace zone, day.
If empty: set `RUNNA_AVAILABLE = false`. Fall back to ATHLETE.md defaults (Monday long run,
Wednesday tempo). Note it in output: "Running estimated from ATHLETE.md — no Runna export found."

---

## Step 3 — Build the Training Schedule

Apply the 3-on-1-off PPL cycle. Anchor constraints first, then fill in.

**Anchors (non-negotiable):**
- Long Run day = no lifting (default Monday, check Runna calendar for exceptions)
- Day before long run = no Legs (keep legs fresh)
- If Day C (Legs) lands on long run day or day before → skip it, push Legs to the next available day
- Any travel/constraint days from Step 1 = inactive

**Cycle logic:**
- Rotate: Day A (Push) → Day B (Pull) → Day C (Legs) → Rest → repeat
- Determine where in the rotation the week starts (ask Logan if unclear, or infer from workout-plan.md date)
- Rest day falls wherever it naturally lands in the 3-on-1-off cycle — it is NOT fixed to Friday
- Double day (lift + run same day): lift first at 9:30 AM, run after — flag it with a warning
- Sunday: Legs only if recovery ≥ 67% AND Day C hasn't been done yet

**Strain targets per session (from workout-plan.md or ATHLETE.md):**
- Push / Pull: 10–12
- Legs: 8–10
- Long run: 16–20
- Easy / tempo run: 12–15
- Rest / recovery: 3–5

---

## Step 4 — Output Weekly Summary in Chat

```
**Training Week — [Mon date] to [Sun date]**

| Day       | Session              | Type     | Strain |
|-----------|----------------------|----------|--------|
| Monday    | [Push / Pull / Legs / Rest] | [Lift / Run / Rest] | [X] |
| Tuesday   | ...                  |          |        |
| Wednesday | ...                  |          |        |
| Thursday  | ...                  |          |        |
| Friday    | Rest                 | Rest     | —      |
| Saturday  | [Rest / Lift / Run]  | [type]   | [X]    |
| Sunday    | [Rest / Light Legs]  |          | [X]    |

**Weekly strain estimate:** ~[sum]
**Running:** [Loaded from Runna / Estimated from ATHLETE.md]

→ For your meals this week, run /gordon-ramsay
```

---

## Step 5 — Generate HTML PWA

Generate a training-week HTML file.

**Reference template:**
Check `/Users/loganmatson/Desktop/AI Work/Personal/Health/Meals/Food Calendar/` for the most recently modified HTML file — use that as the CSS/render template. Copy CSS verbatim.

**Structural changes from the template:**
- App title: "Training Week"
- Bottom nav: Week | Training (2 tabs only — no Nutrition tab)
- Week tab: one card per day showing training type + strain + session detail only (no meal rows, no macro bars)
- Training tab: last workout detail (from workout-plan.md compounds + accessories)
- Add a sticky note at top of Week tab: "Meals → run /gordon-ramsay"

**Output paths:**
- Folder: `/Users/loganmatson/Desktop/AI Work/Health/Training/Calendar/[Month]/`
- File: `performance-week-[mon-dd].html`

```bash
mkdir -p "/Users/loganmatson/Desktop/AI Work/Health/Training/Calendar/[Month]"
```

**DATA block:**

```javascript
const WEEK = {
  label: "Week of [Mon date]",
  dateRange: "[Mon date] – [Sun date], [Year]",
  days: [
    {
      name: "Monday", date: "[date]", active: true,
      training: {
        type: "[lift / run / rest / double]",
        detail: "[e.g., Push Day — Chest / Shoulders / Tris]",
        strain: [X],
        note: "[any constraint or flag, or null]"
      }
    }
    // ... Tuesday through Sunday
  ]
};

const TRAINING = {
  lastWorkoutDate: "[date from workout-plan.md]",
  intensity: "[HIGH / STANDARD / RECOVERY]",
  hrv: [X], recovery: [X], restingHR: [X],
  sessionType: "[e.g., Pull Day — Back / Biceps / Rear Delts]",
  duration: [X], strainTarget: [X],
  compounds: [
    { exercise: "[name]", sets: [X], reps: "[X–X]", load: "[note]", rest: "[X]", rpe: "[X–X]" }
  ],
  accessories: [
    { exercise: "[name]", sets: [X], reps: "[X–X]", load: "[note]", rest: "[X]", rpe: "[X–X]" }
  ],
  runningNote: "[Loaded from Runna / Estimated from ATHLETE.md]"
};

const GENERATED = "[current date]";
```

Write the full HTML using the Write tool. Open it:
```bash
open "/Users/loganmatson/Desktop/AI Work/Health/Training/Calendar/[Month]/performance-week-[mon-dd].html"
```

Output in chat:
```
Training week saved — Training/Calendar/[Month]/performance-week-[mon-dd].html
For meals this week → run /gordon-ramsay
```

---

## Step 6 — Add to Google Calendar (optional)

Ask: "Want me to add your training sessions to Google Calendar?"

If yes, create events for lift and run days only (skip rest days) using the calendar ID discovered in Phase 0:

```
gcal_create_event(
  calendarId: [training calendar ID from Phase 0, or "primary" if not found],
  summary: "[Session] — [Type]",
  start: { dateTime: "[Day]T09:30:00", timeZone: "America/New_York" },
  end:   { dateTime: "[Day]T11:00:00", timeZone: "America/New_York" },
  description: "Strain target: [X]"
)
```

One call per training day. If GCal MCP unavailable, fall back to:
```bash
# Fallback: Apple Calendar via osascript
osascript << 'EOF'
tell application "Calendar"
  tell calendar "Training"
    make new event with properties {summary:"[Session] — [Type]", start date:date "[Day Month DD YYYY at 09:30:00 AM]", end date:date "[Day Month DD YYYY at 11:00:00 AM]", description:"Strain target: [X]"}
  end tell
end tell
EOF
```
If calendar "Training" doesn't exist, note it and skip. Do not fail.

---

## Guardrails

- Training schedule is CONCRETE — 3-on-1-off PPL only, no per-day HRV adaptation
- Only adaptation is scheduling around the long run day (default Monday, check Runna calendar)
- Never lift on long run day — non-negotiable
- Never do Legs on Friday — Push or Pull only on Fridays
- Do not re-plan meals — reference /gordon-ramsay for all nutrition
- If workout-plan.md is missing: note the gap, schedule from ATHLETE.md PPL split
- Model: claude-sonnet-4-6

---

<!-- FUTURE UPGRADE PATH — do not implement yet

Phase 3: Gordon Ramsay integration
- After Gordon generates a meal plan, it writes a summary to Meals/meal-log.md
- /calendar reads meal-log.md and adds a meal summary row to each day card
- No re-computation — just pulls Gordon's output and displays it alongside training

Phase 4: Runna native export
- Drop .md export into Training/Running/ → /calendar auto-reads it, no skill changes needed

Phase 5: Google Calendar MCP
- Replace Step 6 osascript with mcp__google_calendar__create_event
- Works without Mac Calendar app, syncs to iPhone via Google
-->
