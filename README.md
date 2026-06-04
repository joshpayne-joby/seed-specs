# seed-specs

Lean public mirror of seed framework specs that headless and cloud-Claude agents fetch at session start.

This repo exists because Claude.ai's scheduled-task and Claude-Project runtimes only permit outbound HTTP to a small allow-list of hosts — `raw.githubusercontent.com` is on it, secret gists are not. So the operational specs and templates these runtimes need at session start live here, in a deliberately bare-bones public repo.

## Contents

### Operational specs

- `SMARTSHEET_DRIFT_WATCHER.md` — operational spec for the daily Drift Watcher routine. Uses `{{placeholder}}` syntax for operational IDs (the scheduled-task prompt provides them via its Configuration block).

### Canvas body templates (`docs/templates/`)

WebFetched by the Workshop provisioner at canvas-creation time. Each file is just the canvas body content — no header, no fence. The consumer substitutes placeholders like `[project_id]` with real values before writing.

- `hub-canvas.md` — project Hub (`[Project] | Project Hub`)
- `claude-canvas.md` — project Claude Canvas (`[Project] | Task Board`)
- `human-canvas.md` — project Human Canvas / Field Reference
- `skills-file.md` — per-project SKILLS.md downloadable artifact
- `my-tasks-canvas.md` — per-person My Tasks canvas (`[Name] | My Tasks`)
- `routine-config-canvas.md` — per-person Routine Config canvas (`[Name] | Routine Config`)

### Safe-write skills (`.claude/skills/`)

WebFetched at runtime by any consumer that writes to Smartsheet rows or Slack canvases. The skills enforce write protocols mechanically — invocation IS the precondition.

- `smartsheet-safe-write/SKILL.md` — read-before-write, version-guarded, UID-auto-generating protocol for `update_rows` / `add_rows` against a Project Registry sheet.
- `slack-canvas-safe-write/SKILL.md` — read-before-write, full-replace-only, title-vs-H1 / ghost-H1 / @mention-translation protocol for `slack_update_canvas` / `slack_create_canvas`.

### Canvas write contracts (`contracts/`)

Reference docs cited by the skills and by inline-insurance preambles.

- `canvas-write-rules.md` — full rule set with rationale (expands the why behind the skill)
- `canvas-access.md` — channel-share vs. workspace-share access patterns
- `canvas-writer-preamble.md` — 6-rule inline-insurance subset for runtimes where the skill loader fails

### Prompt template (`prompts/`)

- `TEMPLATE.md` — the seed Routine prompt template, populated by the Workshop's COLLAB mode for each new collaborator.

## What this repo is NOT

- The canonical home for these specs — edits land here as mirrors, not as primary writes.
- A general-purpose seed framework reference.
- Auto-synced (yet). Mirror updates land via manual copy until a sync workflow is wired up.
