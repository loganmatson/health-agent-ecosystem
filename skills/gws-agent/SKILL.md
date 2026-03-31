---
name: gws-agent
description: >
  Google Workspace automation agent — runs workflow recipes across Gmail, Drive, Docs, Sheets,
  Calendar, and Slides via the gws CLI. Use when Logan says "run my morning digest", "prep my week",
  "sync my workspace", "check my emails", "summarize my calendar", "Google Workspace agent", or
  "gws-agent". Also triggers on multi-service Workspace requests like "pull my emails and update the sheet."
---

# gws-agent — Google Workspace Automation

Orchestrates multi-service Google Workspace workflows using the `gws` CLI + GCal MCP.
One command → autonomous multi-step execution across Gmail, Drive, Docs, Sheets, Calendar.

## Phase 0 — Auth Check (always run first, before any recipe)

```bash
gws auth status 2>&1
```

If `token_valid` is `false` or the command errors, re-authenticate immediately:

```bash
gws auth login -s drive,gmail,calendar,sheets,docs,slides
```

Token expires on a 7-day rolling window (GCP OAuth test mode). Do not skip this check — gws CLI calls will fail silently mid-pipeline if the token is expired.

---

## Prerequisites

**GWS CLI auth** — check with: `gws auth status`
If token invalid: `gws auth login -s drive,gmail,calendar,sheets,docs,slides`
Creds: `~/.config/gws/client_secret.json` (copied from google-calendar-mcp keys)
GCP project: `hale-cocoa-489901-m9`

**GCal MCP auth** — test mode tokens expire every 7 days. Check token age:
```bash
stat -f "%Sm" ~/.config/google-calendar-mcp/tokens.json
```
If >6 days old, re-auth before any calendar operations:
1. Open GCP Console → APIs & Services → OAuth consent screen → add self as test user if removed
2. Run a GCal MCP call (e.g. `gcal_list_calendars`) to trigger OAuth refresh
3. If refresh fails, delete `~/.config/google-calendar-mcp/tokens.json` and re-authenticate

For params with special characters or emojis, always use a temp file:
```bash
cat > /tmp/gws_params.json << 'EOF'
{"key": "value"}
EOF
gws service resource method --params "$(cat /tmp/gws_params.json)"
```

## Available Recipes

### /gws-agent morning-digest
Optimized 3-agent pipeline. Max 3 sentences per section in final output.
Memory file: `~/Desktop/AI Work/Personal/Health/digest-memory.md` (create if not exists)
Target: ~35-45K tokens per run.

---

#### PHASE 0 — Orchestrator reads digest-memory.md (no subagent)

Read `digest-memory.md` once. Extract the last-30-days context:
- Covered career topics, terms, quote authors, news sources, section types
- Count entries to determine Claude-chosen section rotation (count mod 4)
- Pass this context string to both subagents below (avoids duplicate reads)

If file doesn't exist, create it with a header and proceed with empty history.

---

#### PHASE 1 — Dispatch Agents A + B in parallel (use model: "sonnet")

**Agent A — Research Agent** (Sonnet)
Search for 4 real, recent AI news stories across two time windows:
- 2 stories from last 14 days: new models, Claude Code capabilities, job market/cuts
- 2 stories from last 4 days: government/policy, new tools, industry shifts
Return for each: exact headline | publication name | direct clickable URL | 2-sentence summary.
Hard rules: URLs must be real. No two stories from the same publication. No stories covering topics in the provided digest-memory context.
**Fallback if web search unavailable:** Fetch and extract from:
- https://techcrunch.com/category/artificial-intelligence/
- https://www.theverge.com/ai-artificial-intelligence
- https://www.politico.com/technology
- https://www.wired.com/tag/artificial-intelligence/

**Agent B — Content Writer** (Sonnet)
Receives the last-30-days context from Phase 0 (do NOT re-read digest-memory.md). Generate:
1. **Career Story** — Quick real-world story pivotal to Logan's career (advisory, cyber/compliance, AI, insurance/nonprofit). End with: "Term to remember: [X] — [one-sentence definition]"
2. **Weekly Market Pulse** — 1 data point + brief implication for AI/tech job market or advisory/consulting demand.
3. **AI Practice Takeaways** — 2 specific takeaways for Logan's Claude usage + 1 concrete thing to try today. Think: skills, agents, subagents, workflows, API keys, token efficiency. Not generic.
4. **[Claude-chosen section]** — Rotation provided in context: 0=mental model, 1=prompt pattern, 2=industry insight, 3=tool comparison. Label it clearly.
5. **Motivational Quotes** — 2 quotes fitting Logan's situation (early career, AI-forward, ambitious). Format: "Quote" — Source.

Hard rule: Do NOT repeat any career story topic, term, or quote author from the provided context.

---

#### PHASE 2 — Orchestrator curates + composes + sends (no subagent)

1. **Curate**: Pick best 3 of 4 stories from Agent A. Recency wins ties. No two from same publication.
2. **Compose**: Assemble all content into HTML email (cards with colored left borders, clean serif font).
   Subject: `Cyber & AI Morning Brief — [Month DD, YYYY]`
3. **Send** to lpmats09@gmail.com:

```bash
python3 << 'PYEOF'
import base64
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
msg = MIMEMultipart('alternative')
msg['To'] = 'lpmats09@gmail.com'
msg['From'] = 'lpmats09@gmail.com'
msg['Subject'] = 'Cyber & AI Morning Brief — [DATE]'
html = """[FULL HTML CONTENT]"""
msg.attach(MIMEText(html, 'html'))
raw = base64.urlsafe_b64encode(msg.as_bytes()).decode()
with open('/tmp/digest_raw.txt', 'w') as f:
    f.write(raw)
PYEOF
gws gmail users messages send --params '{"userId":"me"}' --json "{\"raw\":\"$(cat /tmp/digest_raw.txt)\"}"
```

4. **Update memory**: Append today's entry to `digest-memory.md`:
```
## [YYYY-MM-DD]
- Career story topic: [X]
- Term: [X]
- News sources used: [pub1, pub2, pub3]
- Quotes: [author1, author2]
- Claude-chosen section type: [mental model / prompt pattern / insight / comparison]
```

5. **State write — pipeline-state.json**: Read `Personal/Health/pipeline-state.json` (create from empty schema if missing). Update the `morning_digest` block:
   - `run_date`: today's ISO date (e.g. `"2026-03-21"`)
   - `career_topic`: the career/AI topic covered by Agent B's Career Story section
   - `cyber_topic`: the cyber/insurance topic if one was present; otherwise `""`
   - `digest_path`: `""` (email sent — no local file)

   Set top-level `last_updated` to today's ISO timestamp. Write the full file back using the Write tool.

### /gws-agent weekly-prep
1. Calendar: read next 7 days across all calendars
2. Gmail: scan for unread threads older than 2 days (likely need action)
3. Sheets: check for any sheets modified this week (surface names + last editor)
4. Output: weekly summary with action items flagged

### /gws-agent study-mode [session-number]
1. Drive: find IT Infrastructure Notes + any Classwork docs for the session
2. Read relevant sections
3. Create a focused GCal event "Study Block" for today at next available 2hr window
4. Output: doc links + key topics to review

### /gws-agent travel-sync
1. Gmail: search for confirmation emails (flights, hotels, bookings) for Scotland trip
2. Sheets: read Scotland London Dublin 2026 itinerary (ID: 1N12pyPrY-8uYVBriithNzAdmInSW2z8NC6vmU4QFiMQ)
3. Cross-reference: flag any itinerary rows with no confirmed booking
4. Output: booking status report — what's confirmed vs. still needed

### /gws-agent update-doc [doc-name] [content]
1. Drive: search for doc by name
2. Read current end index
3. Append content at end
4. Confirm write

### /gws-agent update-sheet [sheet-name] [range] [data]
1. Drive: search for sheet by name
2. Write data to specified range using USER_ENTERED input option
3. Confirm write

## Key CLI Syntax
```bash
# Search Drive
gws drive files list --params '{"q":"name contains \"X\"","fields":"files(id,name)"}'

# Read sheet range
gws sheets spreadsheets values get --params '{"spreadsheetId":"ID","range":"A1:K50"}'

# Batch write to sheet
gws sheets spreadsheets values batchUpdate \
  --params '{"spreadsheetId":"ID"}' \
  --json '{"valueInputOption":"USER_ENTERED","data":[{"range":"A1","values":[["val"]]}]}'

# Clear a cell/range
gws sheets spreadsheets values clear --params '{"spreadsheetId":"ID","range":"K4"}'

# Read doc
gws docs documents get --params '{"documentId":"ID","fields":"body(content(endIndex))"}'

# Append to doc
gws docs documents batchUpdate \
  --params '{"documentId":"ID"}' \
  --json '{"requests":[{"insertText":{"location":{"index":INDEX},"text":"CONTENT"}}]}'

# List Gmail (unread, last 24hrs)
gws gmail users messages list --params '{"userId":"me","q":"is:unread newer_than:1d","maxResults":20}'

# Get Gmail message
gws gmail users messages get --params '{"userId":"me","id":"MESSAGE_ID","format":"metadata"}'
```

## Known Document IDs
- Scotland London Dublin 2026: `1N12pyPrY-8uYVBriithNzAdmInSW2z8NC6vmU4QFiMQ`
- IT Infrastructure Notes: `1I5UjiksJoVCQFumkkGoPnr40pQJt7OiyIis4NUPxQ6I`
- AI Agents Blog Post Draft: `13W3wiOXGvoxxpGPUSWdfPArJGumjVMZbO4AGyTTVnr8`

## Parallelization Rules
- Always dispatch independent service calls in parallel (Gmail + Calendar + Drive can all run at once)
- Use background subagents for long-running reads when main agent needs to proceed
- Batch all writes into a single batchUpdate call — never write cells one at a time

## Guardrails

- **Subagent handoff context**: when dispatching Agents A and B in the morning-digest pipeline, prepend each prompt with a one-sentence context summary from Phase 0 before passing any raw data. Example: "The orchestrator read 30 days of digest history; your job is to find 4 new AI stories NOT covering: [topics]." Cuts hallucination and token load in downstream agents.
- **Auth before CLI**: always verify `gws auth status` returns `token_valid: true` before the first GWS CLI call. If expired, re-auth with `gws auth login -s drive,gmail,calendar,sheets,docs,slides` (7-day rolling token in test mode).

## Token Efficiency
- Never read A1:Z1000 — always specify the tightest range needed
- For large sheets, run a targeted read of just the columns you need
- For Gmail, use metadata format (not full) unless body content is required
