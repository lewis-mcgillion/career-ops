# career-ops — Skill Router

This instruction file defines how to route career-ops mode requests in GitHub Copilot CLI.

## Context Loading by Mode

After determining the mode from the user's natural language request, load the necessary files before executing.

### Modes that require `_shared.md` + their mode file

Read `modes/_shared.md` + `modes/_profile.md` (if exists) + `modes/{mode}.md`

Applies to: `auto-pipeline`, `oferta`, `ofertas`, `pdf`, `contacto`, `apply`, `pipeline`, `scan`, `batch`

### Standalone modes (only their mode file)

Read `modes/{mode}.md`

Applies to: `tracker`, `deep`, `training`, `project`

### Modes that benefit from subagents

For `scan`, `apply`, and `pipeline` (3+ URLs): launch as background subagent with the content of `_shared.md` + `modes/{mode}.md` injected into the subagent prompt:

```
task(
  agent_type="general-purpose",
  mode="background",
  name="career-ops-{mode}",
  description="career-ops {mode}",
  prompt="[content of modes/_shared.md]\n\n[content of modes/{mode}.md]\n\n[invocation-specific data]"
)
```

## File Reading Priority

Always read files in this order:
1. `modes/_shared.md` (system context, scoring, rules)
2. `modes/_profile.md` (user customizations — overrides _shared.md)
3. `modes/{mode}.md` (mode-specific instructions)
4. `cv.md` (candidate CV — required for evaluations)
5. `config/profile.yml` (candidate identity)
6. `article-digest.md` (proof points, if exists)

## Browser-Based Modes

These modes use browser interaction and should use Chrome DevTools tools:

- **scan**: Navigate career pages → `chrome-devtools-navigate_page` + `chrome-devtools-take_snapshot`
- **apply**: Read/fill application forms → `chrome-devtools-take_snapshot` + `chrome-devtools-fill` + `chrome-devtools-click`
- **auto-pipeline**: Verify offer URLs → `chrome-devtools-navigate_page` + `chrome-devtools-take_snapshot`
- **pipeline**: Extract JDs from URLs → `chrome-devtools-navigate_page` + `chrome-devtools-take_snapshot`

## Offer Verification

**NEVER trust web_search to verify if an offer is still active.** ALWAYS use Chrome DevTools:
1. `chrome-devtools-navigate_page` to the URL
2. `chrome-devtools-take_snapshot` to read content
3. Only footer/navbar without JD = closed. Title + description + Apply = active.
