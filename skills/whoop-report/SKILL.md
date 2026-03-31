---
name: whoop-report
description: >
  Monthly Whoop performance analysis agent with subagent pipeline. Reads raw Whoop CSVs,
  dispatches parallel subagents for cleaning, analysis, PDF generation, and scorecard updates.
  Trigger phrases: "whoop report", "monthly whoop", "run whoop analysis", "whoop-report".
argument-hint: "[month folder name — e.g., 'Mar 26' or leave blank to auto-detect latest]"
---

You are an **orchestrator agent** for Whoop monthly performance analysis. You coordinate a
pipeline of subagents — you do NOT do the analysis yourself. Your job is to gather context,
locate data, dispatch subagents in the right order, verify their outputs, and present results.

Use `/opt/anaconda3/bin/python3` for all Python execution. Available packages: pandas, numpy, scipy, matplotlib.

---

## Phase 0 — Pre-Flight (Orchestrator)

**Subagent detection:** If you were dispatched by optimize-week or another orchestrator (the prompt
will include context like "This is the monthly Whoop analysis for..." or "DO NOT ASK QUESTIONS"),
skip the questions below — context is already in your prompt. Proceed directly to Phase 1.

**Standalone mode only** — ask these 3 questions and wait for answers:

1. Any injuries, illnesses, travel, or unusual events this month?
2. Specific focus areas? (sleep optimization, strain management, intervention testing, recovery trends)
3. Any new journal behaviors tracked this month?

Store the answers — you'll pass them as context to subagents that need them.

---

## Phase 1 — Locate Data (Orchestrator)

If `$ARGUMENTS` is provided, use it as the folder name:
```
/Users/loganmatson/Desktop/AI Work/Personal/Health/Whoop Performance/$ARGUMENTS/raw/
```

If no argument, find the most recent Whoop folder:
```
/Users/loganmatson/Desktop/AI Work/Personal/Health/Whoop Performance/
```
Pick the folder with the latest date in its name.

Verify these raw files exist:
- `physiological_cycles.csv` — daily recovery, HR, HRV, sleep, strain
- `sleeps.csv` — detailed sleep records including naps
- `workouts.csv` — individual workout records
- `journal_entries.csv` — self-reported behaviors

Note any missing files — pass this info to subagents so they skip dependent analyses.

Set these path variables for subagent prompts:
- `RAW_DIR`: path to raw/ folder
- `CLEAN_DIR`: path to cleaned/ folder (create it: `mkdir -p`)
- `OUTPUT_DIR`: `/Users/loganmatson/Desktop/AI Work/Health/Whoop Performance/[Month]/`
- `MONTH_LABEL`: e.g., "March 2026"
- `MONTH_SHORT`: e.g., "mar-26"

**Period comparison detection:**
If the data spans a range that includes a prior analysis cutoff (e.g., data from Dec 25 – Mar 28, with a previous analysis ending Feb 17), ask the user:

> "This data covers [start] to [end]. When was your last analysis cutoff? (e.g., Feb 17)"

Then set:
- `PRIOR_CUTOFF`: the date of the last analysis (e.g., "2026-02-17")
- `PERIOD_1_LABEL`: e.g., "Dec 25 – Feb 17 (baseline)"
- `PERIOD_2_LABEL`: e.g., "Feb 18 – Mar 28 (new)"

Pass these to ALL subagents. Every analysis should include a period comparison section:
- Compute the same metrics for Period 1 and Period 2 separately
- Report deltas, direction of change, and whether the change is meaningful
- Narrative framing: "What actions are helping? What's hurting? What changed?"

If there is no prior cutoff (first-ever analysis), skip period comparison.

---

## Phase 2 — Dispatch Subagent 1: Data Cleaner

Use the Agent tool to spawn a subagent with this prompt:

> You are a data cleaning agent. Use `/opt/anaconda3/bin/python3`.
>
> Read raw Whoop CSV files from: `{RAW_DIR}`
> Write cleaned output to: `{CLEAN_DIR}`
>
> Produce these files:
>
> | Output | Source | Key transformations |
> |--------|--------|-------------------|
> | `whoop-clean.csv` | physiological_cycles.csv | Normalize dates, coerce to float, flag anomaly rows |
> | `whoop-summary.csv` | whoop-clean.csv | Overview + descriptive stats (mean/min/max/SD) + weekly averages by ISO week |
> | `whoop-workouts-clean.csv` | workouts.csv | Simplify timestamps, drop GPS, reorder columns |
> | `whoop-workouts-summary.csv` | whoop-workouts-clean.csv | Aggregate by activity type: avg strain, calories, HR, duration, count |
> | `whoop-journal-clean.csv` | journal_entries.csv | Pivot to daily boolean matrix (one row per day, one column per behavior) |
>
> Anomaly criteria:
> - Missing recovery + missing HRV + missing sleep = anomaly
> - RHR < 30 or > 100, HRV < 10 or > 300, sleep < 30 min (non-nap) = flag
>
> Missing files: {list any missing raw files}
>
> After writing all files, confirm they exist and report row counts for each.

**Wait for Subagent 1 to complete. Verify cleaned files exist before proceeding.**

---

## Phase 3 — Dispatch Analysis Subagents (PARALLEL)

Launch ALL FOUR of these subagents simultaneously using parallel Agent tool calls.
Each writes its results to `{CLEAN_DIR}/analysis-results-{section}.json`.

### Subagent 2: Descriptive Stats + Weekly Averages

> You are a statistical analysis agent. Use `/opt/anaconda3/bin/python3`.
> Read cleaned data from: `{CLEAN_DIR}`
>
> **Analysis 3a — Descriptive Statistics:**
> Mean, min, max, SD for: Recovery %, RHR, HRV, Skin Temp, SpO2, Day Strain, Energy Burned, Max HR, Avg HR, Sleep Performance %, Sleep Duration, Deep Sleep, REM, Sleep Efficiency %, Sleep Consistency %
>
> **Analysis 3b — Weekly Averages:**
> Group by ISO week. Mean for all physiological metrics per week.
>
> If `PRIOR_CUTOFF` is set, also compute all descriptive stats separately for Period 1 and Period 2, and include deltas.
>
> Write results to: `{CLEAN_DIR}/analysis-results-descriptive.json`
> JSON structure: `{"descriptive": {...}, "weekly_averages": [...], "period_comparison": {...} (if applicable)}`

### Subagent 3: Regression + HRV Mediator

> You are a statistical analysis agent. Use `/opt/anaconda3/bin/python3`.
> Read cleaned data from: `{CLEAN_DIR}`
>
> **Analysis 3c — OLS Multivariate Regression:**
> Model: Recovery ~ Sleep Duration + Deep Sleep + REM + Sleep Efficiency + Sleep Onset Hour + Prior-Day Strain
> Report: n, R², adjusted R², F-statistic, and per-predictor: β, SE, t, p-value, significance (p<0.05).
> Requires n ≥ 30 complete rows. Skip if insufficient.
>
> **Analysis 3d — HRV Mediator Analysis:**
> 1. Pearson correlation: HRV vs Recovery (r, p)
> 2. Partial correlation: HRV vs Recovery controlling for sleep duration, deep sleep, REM, efficiency
> 3. If partial r is significant → HRV is independent recovery pathway
>
> If `PRIOR_CUTOFF` is set, run the regression and HRV analysis on each period separately and note if the model or relationships changed.
>
> Write results to: `{CLEAN_DIR}/analysis-results-regression.json`
> JSON structure: `{"regression": {...}, "hrv_mediator": {...}, "period_comparison": {...} (if applicable)}`

### Subagent 4: Dose-Response + Training Modality

> You are a statistical analysis agent. Use `/opt/anaconda3/bin/python3`.
> Read cleaned data from: `{CLEAN_DIR}`
> Read workout data from: `{CLEAN_DIR}/whoop-workouts-clean.csv`
>
> **Analysis 3e — Quadratic Dose-Response (Strain → Recovery):**
> Fit: Recovery = a + b·Strain + c·Strain²
> If c is significant (p<0.05): inverted-U confirmed, report optimal strain zone.
> If not: report as linear or null.
>
> **Analysis 3f — Natural Experiment (Training Modality):**
> Group days by primary workout type (running, strength, rest/none).
> Compare next-day: Recovery %, HRV, Deep Sleep.
> Report: group means, Cohen's d between groups, p-values (independent t-tests).
> Minimum ≥ 8 days per group.
>
> If `PRIOR_CUTOFF` is set, compare dose-response curves and modality effects between periods. Did the optimal strain zone shift? Did recovery response to training types change?
>
> Write results to: `{CLEAN_DIR}/analysis-results-dose-response.json`
> JSON structure: `{"dose_response": {...}, "training_modality": {...}, "period_comparison": {...} (if applicable)}`

### Subagent 5: Interventions + Correlations + Trajectory

> You are a statistical analysis agent. Use `/opt/anaconda3/bin/python3`.
> Read cleaned data from: `{CLEAN_DIR}`
> Read journal data from: `{CLEAN_DIR}/whoop-journal-clean.csv`
>
> Pre-flight context: {pass pre-flight answers here}
>
> **Analysis 3g — Behavioral Interventions:**
> For each journal behavior with ≥ 30 yes-days AND ≥ 30 no-days:
> (A behavior logged only 5 times with a huge effect size is bias, not signal. 30-day minimum eliminates this.)
> Compare on: Recovery %, HRV, Deep Sleep, Sleep Performance %.
> Report: group means, Cohen's d, p-value. Rank by |Cohen's d|.
> Flag paradoxical findings (e.g., sugar correlating with better outcomes).
>
> **Analysis 3h — Correlation Matrix:**
> 10 variables: Recovery, HRV, RHR, Strain, Sleep Duration, Deep Sleep, REM, Efficiency, SpO2, Skin Temp.
> Pearson r matrix. Flag |r| > 0.5.
>
> **Analysis 3i — Weekly Trajectory:**
> 4 series by week: Recovery, HRV, Strain, Deep Sleep.
> Narrative tags: fatigue accumulation, overreaching, recovery rebound.
>
> If `PRIOR_CUTOFF` is set, compare intervention effects between periods. Which behaviors became more/less impactful? Include a "what changed" narrative.
>
> Write results to: `{CLEAN_DIR}/analysis-results-interventions.json`
> JSON structure: `{"interventions": [...], "correlation_matrix": {...}, "weekly_trajectory": {...}, "period_comparison": {...} (if applicable)}`

**Wait for ALL FOUR subagents to complete. Verify all 4 JSON files exist.**

---

## Phase 4 — Merge Analysis Results (Orchestrator)

Read all 4 JSON files and merge them into a single `{CLEAN_DIR}/analysis-results.json`.
This combined file is the input for the output subagents.

---

## Phase 5 — Dispatch Output Subagents (PARALLEL)

Launch BOTH subagents simultaneously.

### Subagent 6: PDF Generator

> You are a report generation agent. Use `/opt/anaconda3/bin/python3` with matplotlib.
>
> Read analysis results from: `{CLEAN_DIR}/analysis-results.json`
> Read cleaned data from: `{CLEAN_DIR}` (for charts that need raw series)
>
> Generate a multi-page PDF at:
> `{OUTPUT_DIR}/analyst-brief-{MONTH_SHORT}.pdf`
>
> Use `matplotlib.backends.backend_pdf.PdfPages` for multi-page output.
>
> Pre-flight context: {pass pre-flight answers}
> Month label: {MONTH_LABEL}
>
> **PDF Style — McKinsey/Bain Executive Report:**
> - Figure size: 8.5 x 11 inches (letter)
> - Primary palette: Navy `#1B2A4A`, Dark grey `#2D3436`, White `#FFFFFF`
> - Accent colors: Teal `#0097A7` (positive/good), Coral `#E74C3C` (negative/alert), Gold `#F39C12` (neutral)
> - Typography: sans-serif (Helvetica/Arial), bold hierarchy (20pt title, 16pt section, 10pt body)
> - Footer: navy branded bar spanning full width with report title + page number
>
> **Layout rules (CRITICAL — no white space waste):**
> - Every page has 2-3 visual elements (charts + insight boxes + KPI tiles). No single-chart pages.
> - Charts fill their space — axes expand to use available width/height.
> - When a chart takes ~60% width, the remaining 40% gets a stats panel (shaded box with key numbers).
> - Background fills everywhere: insight boxes = navy bg, KPI tiles = light grey with accent bar, alternating section shading.
> - Horizontal dividers between sections instead of vertical whitespace gaps.
> - No text-only pages — every page MUST have at least one chart or visual element.
> - Think of each page as a 2x3 or 3x2 grid. Every cell has something in it.
>
> **Insight boxes (on EVERY content page):**
> Each page gets a navy-background insight box at the bottom with 4 fields:
> - KEY FINDING: bold one-liner
> - IMPACT: why it matters
> - TREND: ↑/↓ delta vs prior period
> - ACTION: concrete next step
> Rendered as a shaded rectangle (navy bg, white text) via matplotlib FancyBboxPatch or fig.add_axes().
>
> **Narrative voice — executive presentation style:**
> - Lead every section with the "so what" — what does this mean for health, training, or sleep?
> - Put statistical backup in parentheses: "Sleep duration is the single biggest lever (β=0.13, p=0.005)."
> - Protocol recommendations sound like a coach, not a textbook.
>
> **Reference implementation:** See `generate_pdf.py` in the most recent month's `output/` folder for the
> full working template with all helper functions (insight_box, kpi_tile, section_header, add_footer).
> Use that as the starting point — adapt paths and month label, don't rewrite from scratch.
>
> **Pages (12 total, domain-sectioned):**
> 1. Cover + Executive Dashboard (navy banner, 4 KPI tiles with MoM arrows, exec summary, data overview)
> 2. Recovery Deep Dive (weekly trajectory + histogram + regression table + insight box)
> 3. Sleep Architecture & Quality (stacked bar + 3 sparklines: efficiency/consistency/debt + insight box)
> 4. Strain & Training Load (dose-response scatter + modality bars + weekly strain bars + insight box)
> 5. HRV Analysis (time series with 7-day rolling avg + scatter + stats callout + insight box)
> 6. Journal Behaviors: Positive Interventions (top 10 horizontal bar, significance color-coded + insight box)
> 7. Journal Behaviors: Risk Factors (top 10 negative behaviors, same format inverted + insight box)
> 8. Nutrition ↔ Performance (protein vs recovery scatter + tier box plot + daily timeline + insight box)
>    — reads `meal-annotations.json` from `Personal/Health/Meals/meal-annotations.json`
> 9. Correlation Matrix + Key Relationships (heatmap + top 5 callouts + insight box)
> 10. Weekly Trajectory Dashboard (2x2 grid: Recovery, HRV, Strain, Deep Sleep with trend lines)
> 11. Protocol Recommendations (5 recommendation cards with priority badges + impact bar chart)
> 12. Methodology & Data Notes (two-column layout + data quality table)
>
> Confirm the PDF was written and report page count.

### Subagent 7: Scorecard Builder

> You are a scorecard HTML agent.
>
> Read analysis results from: `{CLEAN_DIR}/analysis-results.json`
> Read the scorecard template from: `/Users/loganmatson/Desktop/AI Work/Personal/Health/Whoop Performance/Feb 26/output/scorecard-feb-26.html`
>
> Create a new scorecard at:
> `{OUTPUT_DIR}/scorecard-{MONTH_SHORT}.html`
>
> Update the `SCORECARD` DATA object:
> - `month`: "{MONTH_LABEL}"
> - `dateRange`: start — end dates from the data
> - `totalDays`, `totalWorkouts`, `anomalyRows`: from descriptive stats
> - `metrics[0]`: Avg Recovery % (include barPercent, barColor: "green")
> - `metrics[1]`: Avg HRV ms (include barPercent scaled to 200ms max, barColor: "blue")
> - `metrics[2]`: Avg Sleep in Xh Ym format (include barPercent scaled to 8h max, barColor: "amber")
> - `metrics[3]`: Avg Strain (include barPercent scaled to 21 max, barColor: "green")
> - `metrics[4]`: Top intervention name + Cohen's d + key effect (include badge: "Highest ROI")
>
> Month-over-month trends: Check if a prior month's scorecard exists in a sibling folder.
> If yes, read its DATA section and compute deltas. Set `trend` to "up"/"down"/"neutral" and `delta` to the difference string.
>
> Update `verdict` with this month's one-sentence key finding.
> Keep the existing visual design (dark theme, gradient accent, pulse dot, animated bars).
>
> Confirm the file was written.

**Wait for BOTH subagents to complete.**

---

## Phase 6 — Open + Confirm (Orchestrator)

1. Open the PDF: `open "{OUTPUT_DIR}/analyst-brief-{MONTH_SHORT}.pdf"`
2. Open the scorecard: `open "{OUTPUT_DIR}/scorecard-{MONTH_SHORT}.html"`
3. Print a summary:
```
Analysis complete — {MONTH_LABEL}
  PDF: Health/Whoop Performance/[Month]/analyst-brief-{MONTH_SHORT}.pdf
  Scorecard: Health/Whoop Performance/[Month]/scorecard-{MONTH_SHORT}.html
  Cleaned data: Health/Whoop Performance/[folder]/cleaned/

  Pipeline:
  - Subagent 1 (Data Cleaner): done
  - Subagents 2-5 (Analysis — parallel): done
  - Subagent 6 (PDF Generator): done
  - Subagent 7 (Scorecard Builder): done

  Key findings:
  1. [finding 1]
  2. [finding 2]
  3. [finding 3]

  Anything to adjust? (technical depth, focus areas, missing context)
```

Wait for feedback. Iterate if needed.

**Output — pipeline-state.json**: Read `Personal/Health/pipeline-state.json` (create from empty schema if missing). Update the `whoop_report` block:
- `run_date`: today's ISO date (e.g. `"2026-03-19"`)
- `month`: folder name used for this run (e.g. `"Feb 26"`)
- `output_path`: relative path of the analyst brief PDF (e.g. `"Personal/Health/Whoop Performance/Feb 26/output/analyst-brief-feb-26.pdf"`)
- `highlights`: list of the 3 key findings printed in the Phase 6 summary (strings)

Set top-level `last_updated` to today's ISO timestamp. Write the full file back using the Write tool.

---

## Guardrails

- **Subagent handoff context**: every Agent dispatch prompt must open with a one-sentence summary of what prior agents produced and what this agent's specific job is — never pass raw data without a context sentence. Example: "The data cleaner produced 5 CSVs with 45 days of cleaned Whoop data; your job is to run regression analysis." This prevents hallucination and cuts downstream token load.
- **Output schemas are mandatory**: each subagent must write results to the exact JSON file specified. Orchestrator verifies file existence before proceeding to the next phase.

## Error Handling

- If a subagent fails, report which one failed and why
- Do NOT re-run failed subagents automatically — ask the user first
- If an analysis is skipped due to insufficient data, note it in the JSON with `"skipped": true, "reason": "..."`
- The PDF generator and scorecard builder should gracefully handle missing analysis sections

---

## Gotchas

Real failures observed in live runs — distinct from Guardrails (which are rules). These are things that *actually went wrong*:

- **Subagent hallucinated metric name when CSV column was missing** — If a Whoop CSV export is missing a column (e.g., `respiratory_rate` not present in older exports), downstream subagents may invent a value or use a wrong column name. Always have the data-cleaner subagent output a column manifest before analysis agents run.
- **pipeline-state.json not written on partial run** — If any subagent fails mid-pipeline, the orchestrator must still write pipeline-state.json with whatever was completed + the failure details. A missing pipeline-state.json breaks the Loop Status Dashboard and optimize-week cross-checks.
- **PDF generation failed silently (reportlab not available)** — System Python doesn't have reportlab. Always use `/opt/anaconda3/bin/python3` with `matplotlib PdfPages` for PDF generation. Symptom: no PDF output, no error shown to user.
- **Wrong month folder path** — The correct path is `Personal/Health/Whoop Performance/[Mon YY]/` (e.g., `Mar 26/`). Stale paths like `DataSets/Whoop/` or `Health/Whoop Performance/` no longer exist — don't use them.
- **meal-annotations.json not found** — Whoop agent reads meal-annotations.json for nutrition↔recovery correlation. If Gordon Ramsay hasn't run for the analysis period, this file may be empty or missing. Note the gap in the report rather than skipping the section silently.
