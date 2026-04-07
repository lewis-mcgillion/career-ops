# Career-Ops — Copilot CLI Instructions

> This file is the primary entry point for GitHub Copilot CLI. It supplements `CLAUDE.md` (which Copilot CLI also reads) with Copilot-specific tool mappings and invocation patterns.

## How Users Invoke Modes

Copilot CLI does not have slash commands. Users invoke career-ops modes via **natural language**. Detect the user's intent and route to the correct mode:

| User says... | Mode | Read these files |
|---|---|---|
| Pastes a JD (text or URL) | `auto-pipeline` | `modes/_shared.md` + `modes/auto-pipeline.md` |
| "evaluate this offer" / "score this JD" | `oferta` | `modes/_shared.md` + `modes/oferta.md` |
| "compare these offers" / "rank offers" | `ofertas` | `modes/_shared.md` + `modes/ofertas.md` |
| "LinkedIn message" / "outreach" / "contact" | `contacto` | `modes/_shared.md` + `modes/contacto.md` |
| "research this company" / "deep dive" | `deep` | `modes/deep.md` |
| "generate PDF" / "create CV" | `pdf` | `modes/_shared.md` + `modes/pdf.md` |
| "evaluate this course" / "should I take..." | `training` | `modes/training.md` |
| "evaluate this project idea" | `project` | `modes/project.md` |
| "show tracker" / "application status" | `tracker` | `modes/tracker.md` |
| "help me apply" / "fill this form" | `apply` | `modes/_shared.md` + `modes/apply.md` |
| "scan portals" / "find new offers" | `scan` | `modes/_shared.md` + `modes/scan.md` |
| "process pipeline" / "process pending URLs" | `pipeline` | `modes/_shared.md` + `modes/pipeline.md` |
| "batch evaluate" / "process all" | `batch` | `modes/_shared.md` + `modes/batch.md` |
| "career-ops" / "what can you do" / "show menu" | `discovery` | Show the command menu below |

### Auto-Pipeline Detection

If the user's message is NOT a known sub-command AND contains JD text (keywords: "responsibilities", "requirements", "qualifications", "about the role", "we're looking for", company name + role) or a URL to a JD, execute `auto-pipeline` mode.

### Discovery Menu

When the user asks what career-ops can do or says "career-ops", show:

```
career-ops — Command Center

Available modes (just ask in natural language):

  "Evaluate this JD"      → AUTO-PIPELINE: evaluate + report + PDF + tracker (paste text or URL)
  "Process pipeline"      → Process pending URLs from inbox (data/pipeline.md)
  "Evaluate this offer"   → Evaluation only A-F (no auto PDF)
  "Compare these offers"  → Compare and rank multiple offers
  "LinkedIn outreach"     → Find contacts + draft message
  "Research [company]"    → Deep research prompt about company
  "Generate PDF"          → PDF only, ATS-optimized CV
  "Evaluate this course"  → Evaluate course/cert against career goals
  "Evaluate project idea" → Evaluate portfolio project idea
  "Show tracker"          → Application status overview
  "Help me apply"         → Live application assistant (reads form + generates answers)
  "Scan portals"          → Scan portals and discover new offers
  "Batch evaluate"        → Batch processing with parallel workers

Inbox: add URLs to data/pipeline.md → ask "process pipeline"
Or paste a JD directly to run the full pipeline.
```

## Tool Mapping (Copilot CLI ↔ Mode Files)

Mode files reference tools generically. Here is how they map to Copilot CLI tools:

| Generic reference | Copilot CLI tool |
|---|---|
| **WebSearch** | `web_search` tool |
| **WebFetch** | `web_fetch` tool |
| **Playwright / browser_navigate** | `chrome-devtools-navigate_page` |
| **Playwright / browser_snapshot** | `chrome-devtools-take_snapshot` |
| **Playwright / browser_click** | `chrome-devtools-click` |
| **Playwright / browser_fill** | `chrome-devtools-fill` |
| **Read file** | `view` tool |
| **Write file** | `create` tool (new files) or `edit` tool (existing files) |
| **Edit file** | `edit` tool |
| **Grep** | `grep` tool |
| **Bash** | `bash` tool |
| **Agent() / subagent** | `task()` tool with `agent_type` parameter |
| **`claude -p` worker** | `task(agent_type="general-purpose", mode="background")` |

### Browser Interaction (replacing Playwright)

Wherever modes reference Playwright for navigating career pages, verifying offers, or reading JDs:

```
# Instead of: browser_navigate(url) + browser_snapshot()
# Use:
chrome-devtools-navigate_page(url="{url}")
chrome-devtools-take_snapshot()
```

For form filling (apply mode):
```
chrome-devtools-take_snapshot()   # Read form fields
chrome-devtools-fill(uid="{field_uid}", value="{value}")
chrome-devtools-click(uid="{button_uid}")
```

### Subagents (replacing Agent() and claude -p)

For parallel processing (scan, pipeline with 3+ URLs, batch):

```
task(
  agent_type="general-purpose",
  mode="background",
  prompt="[content of modes/_shared.md]\n\n[content of modes/{mode}.md]\n\n[specific data]"
)
```

For exploration/research tasks:
```
task(
  agent_type="explore",
  mode="background",
  prompt="[research question with full context]"
)
```

### Batch Processing

`claude -p` headless workers are NOT available in Copilot CLI. Instead, use `task()` subagents:

1. For each offer to process, launch a `task(agent_type="general-purpose", mode="background")` with the full evaluation prompt
2. Include `modes/_shared.md` + `modes/oferta.md` content in the subagent prompt
3. Each subagent produces: report .md + PDF + tracker TSV line
4. After all subagents complete, run `node merge-tracker.mjs`

## Session Start Checklist

On the first message of each session, silently:

1. Run `node update-system.mjs check` and handle the result per CLAUDE.md
2. Check onboarding status (cv.md, config/profile.yml, modes/_profile.md, portals.yml)
3. If any are missing, enter onboarding mode (see CLAUDE.md)
4. Run `node cv-sync-check.mjs` before the first evaluation

## Language Modes

If the user's JD or request is in German → read from `modes/de/` instead of `modes/`.
If in French → read from `modes/fr/` instead of `modes/`.
See CLAUDE.md for full language mode details.
