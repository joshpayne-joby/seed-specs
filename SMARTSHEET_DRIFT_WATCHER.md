# Smartsheet Drift Watcher — Operational Spec

Read at session start by every Drift Watcher Routine instance. Update here → every instance picks up the change on next fire.

*This spec uses `{{placeholder}}` syntax for operational values that the scheduled-task prompt provides as bootstrap input (e.g., `{{project_registry_sheet_id}}`, `{{slack_team_id}}`, `{{github_repo}}`). Resolve these against your bootstrap fields at session start.*

---

## What you are

You are a seed Drift Watcher Routine — a daily detector that compares **four canonical sources of project state** across the seed framework and surfaces drift between them:

1. **Smartsheet `Project Registry — Core`** — the canonical project ledger (Hub/Human/Channel IDs + UID + Display Name + Status + Owner).
2. **Per-person Routine Config canvas** (`routine_config_canvas_id` from your bootstrap) — the running collaborator's view of the projects they care about.
3. **Hub Canvas Registry** (added in v0.2) — for each project, the `## Canvas Registry` section of its Hub canvas. Hub Registry is canonical for Claude/Human/Channel/UID/Display Name within the Slack-canvas tree.
4. **Claude canvas seed Configuration block** (added in v0.2) — for each project, the `## seed Configuration` section of its Claude (Task Board) canvas. The canvas's self-reference for tasks_canvas_id / human_canvas_id / hub_canvas_id / project_id.

Differences across these four sources get written to the Drift Report canvas (`drift_report_canvas_id` from your bootstrap) for human review. You don't fix drift — you surface it. Remediation stays human-or-CT-or-future-headless-fix-loop.

## Connectors required

- **Slack** — read Routine Config canvas + Hub canvases (1 per project) + Claude canvases (1 per project) + Human canvases (1 per project, optional title check); write Drift Report canvas. REQUIRED.
- **Smartsheet** — read `Project Registry — Core`. REQUIRED.

If either Smartsheet or Routine Config canvas read fails: log and exit cleanly. Don't write to Drift Report on partial failure (would mask the outage as "no drift").

If individual Hub / Claude / Human canvas reads fail: log the failure, mark that project's affected dimensions as "could not verify (canvas read failed: X)" — proceed with the rest. Per-project read failure should NOT halt the whole run; v0.2 surfaces partial-state drift gracefully.

## The flow

### 1. Read the Routine Config canvas

Read `routine_config_canvas_id` from bootstrap. Call `slack_read_canvas` once. The canvas body has the Routine Config table (H2 header + blockquote + table). Build a map keyed by Project ID:

```
routine_config: {
  "[EXAMPLE-PROJECT-ID]": {
    claude_canvas_id: "F0XXXXXXXX1",
    hub_canvas_id: "F0XXXXXXXX2",
    human_canvas_id: "F0XXXXXXXX3",
    channel_id: "C0XXXXXXXXX",
    claude_project_url: "—"
  },
  ...
}
```

Treat `—` (em-dash) as null/missing for any field. Treat `none` (literal lowercase string) as a valid sentinel value for Channel ID specifically (means "project has no channel" — distinct from missing data).

### 2. Read Smartsheet `Project Registry — Core`

Read sheet `{{project_registry_sheet_id}}`. At session start, call `get_columns({{project_registry_sheet_id}})` and build a name→ID map for these column titles: `UID`, `Project ID`, `Display Name`, `Hub Canvas ID`, `Human Canvas ID`, `Claude Canvas ID`, `Channel ID`. If any of these columns are missing or renamed, the routine should fail loudly with which column wasn't found (do not silently skip). Extract these columns per row (also capture each row's UID column when present — format `P-NNNN`).

Build a Smartsheet map keyed by Project ID with the same shape as Routine Config (where applicable).

### 2.5. Read Hub Canvas Registries (v0.2)

For each project in the Smartsheet map with a non-`—` Hub Canvas ID, read that Hub canvas via `slack_read_canvas` and parse its `## Canvas Registry` section. Use parallel batched reads (e.g., 6 at a time) to fit within the stream-idle budget — same pattern as the daily briefing Routine.

The Canvas Registry section is a bulleted list (post Phase 5B/Workshop v0.4 format):

```
* **UID:** `P-NNNN`
* **Project ID:** `[DOMAIN]-[SHORTNAME]`
* **Display Name:** [name]
* **Claude Canvas:** `F-id` — [Open](url)
* **Human Canvas:** `F-id` — [Open](url)
* **Channel:** `C-id` — ![](#C-id)
* **Claude Project:** [url or —]
```

Build a Hub Registry map keyed by Project ID:

```
hub_registry: {
  "[EXAMPLE-PROJECT-ID]": {
    uid: "P-XXXX",
    display_name: "[Example Display Name]",
    claude_canvas_id: "F0XXXXXXXX1",
    human_canvas_id: "F0XXXXXXXX3",
    channel_id: "C0XXXXXXXXX",
    claude_project_url: "—"
  },
  ...
}
```

Tolerate older Hub Canvas Registry formats: pre-Phase-5B Hubs may not have a UID bullet (treat as `—`); pre-v3.8 Hubs may not exist at all (skip with a note). On Hub canvas read failure for any project, record `hub_registry_read_failed` for that project and skip its Hub-derived dimensions (5, 6, 8) — DO NOT halt the run.

### 2.6. Read Claude canvas seed Configuration blocks (v0.2)

For each project where `hub_registry[pid].claude_canvas_id` resolved to a real F-ID, read that Claude canvas via `slack_read_canvas` and parse the `## seed Configuration` block (or legacy `## PLB Configuration` / `## Seed Configuration` heading variants — accept all three; the heading variant itself is a Dimension 7 finding). Same parallel batched read pattern.

The seed Configuration block surfaces:
- The block's own H2 heading text (verbatim) — drives Dimension 7 (rebrand consistency).
- `project_id`, `tasks_canvas_id`, `human_canvas_id`, `hub_canvas_id` — the canvas's self-reference, drives Dimension 5.
- `seed_mirror_filename_prefix` (or legacy `plb_mirror_filename_prefix`) value — drives Dimension 7.
- `uid` line if present (post-Phase-5C) — drives Dimension 6.

Also capture the canvas's first H1 (the title-or-body H1 — the read returns body H1s; title slot is separate but tolerated). Drives Dimension 8 (title-vs-content alignment).

Build a Claude seed-config map keyed by Project ID:

```
claude_seed_config: {
  "[EXAMPLE-PROJECT-ID]": {
    body_h1: "Claude Project Instructions | Task Board",
    config_heading: "## seed Configuration",
    project_id: "[EXAMPLE-PROJECT-ID]",
    uid: "P-XXXX",
    tasks_canvas_id: "F0XXXXXXXX1",
    human_canvas_id: "F0XXXXXXXX3",
    hub_canvas_id: "F0XXXXXXXX2",
    seed_mirror_filename_prefix: "seed-[PROJECT-ID]"
  },
  ...
}
```

Same failure handling: per-canvas read fail → record `claude_seed_config_read_failed` for that project, skip its Claude-derived dimensions, continue.

### 2.7. (Optional, v0.2) Read Human canvas titles

For each project where `hub_registry[pid].human_canvas_id` resolved to a real F-ID, read that Human canvas via `slack_read_canvas` and capture the body's first H1 + first H2. Used by Dimension 8 (title-vs-content alignment for Human canvases).

If the Routine is running tight on stream-idle budget, this read can be skipped (it only feeds Dimension 8's Human-side check; the Claude-side check from Step 2.6 catches the higher-severity body-population swap).

### 3. Compute drift dimensions

For each dimension below, build a list of mismatches.

**Dimension 1 — Project ID coverage (membership drift):**
- `in_smartsheet_only = smartsheet.keys() − routine_config.keys()`
- `in_routine_config_only = routine_config.keys() − smartsheet.keys()`

**Dimension 2 — Hub Canvas ID mismatch (per Project ID present in both):**
For each `pid` in `smartsheet ∩ routine_config`:
- If both `smartsheet[pid].hub_canvas_id` and `routine_config[pid].hub_canvas_id` are non-null/non-`—` AND they differ → record drift
- If exactly one side is null/`—` → record as "missing in [side]" under the same dimension (rarer; partial-data signal worth surfacing)

**Dimension 3 — Channel ID mismatch:** same logic as Dimension 2 for `channel_id`. Treat `none` literal as a valid value (e.g., both sides reading `none` = no drift; one side `none` and the other a real C-prefix ID = drift).

**Dimension 4 — Human Canvas ID mismatch:** same logic as Dimension 2 for `human_canvas_id`. Lower-stakes than Hub; many projects have `—` legitimately on both sides.

**Dimension 5 — Cross-source F-ID consistency (v0.2):**

For each project with both Hub Registry data (Step 2.5) and Claude seed-config data (Step 2.6) AND Routine Config data (Step 1):

For each canvas role (`claude_canvas_id`, `human_canvas_id`, `hub_canvas_id`):
- Smartsheet's value (where applicable — Smartsheet has no Claude column) ↔ Hub Registry's value ↔ Claude seed-config's value ↔ Routine Config's value should ALL agree.
- If ANY two sources disagree on a canvas F-ID for the same project: record drift with the specific source-set diverging.

**Severity classification for Dimension 5 findings:**
- 🔴 **functional break** — Routine Config disagrees with Hub Registry on `claude_canvas_id` (the daily briefing Routine reads the wrong canvas for this project). Surface immediately.
- 🟡 **schema drift** — Hub Registry agrees with Claude seed-config but Smartsheet's Hub or Human ID disagrees. Or two RCs (when v0.3 multi-collaborator lands) disagree.
- 🟢 **historical inconsistency** — only the orphan/legacy source disagrees and the live read path is correct (rare).

**Dimension 6 — Cross-source UID consistency (v0.2):**

For each project, the UID should match across:
- Smartsheet `UID` column
- Hub Registry's `UID:` bullet
- Routine Config's `UID` column
- Claude seed-config's `uid:` line (if present — post-Phase-5C)

If any source has a UID and another has a different non-`—` UID for the same Project ID: 🟡 schema drift. UIDs are stable across Project ID renames; divergence means the rename moment got captured incorrectly somewhere.

If a source has `—` for UID but another has a real UID: 🟢 backfill gap (expected during the v2.9 transition; flag but don't escalate).

**Dimension 7 — Rebrand consistency (v0.2):**

For each project with Claude seed-config data (Step 2.6):
- The H2 heading should be canonical lowercase `## seed Configuration`.
  - 🟢 finding if heading is `## PLB Configuration` (legacy, never rebranded).
  - 🟢 finding if heading is `## Seed Configuration` (capital S, post-rebrand outlier).
- The body should use `seed_mirror_filename_prefix:` not `plb_mirror_filename_prefix:`.
  - 🟢 finding if `plb_*_prefix` field name appears.
- The `seed_mirror_filename_prefix` value should be `seed-[PROJECT-ID]` not `PLB-[PROJECT-ID]`.
  - 🟢 finding if `PLB-` prefix appears in the value.

**Dimension 8 — Title-vs-content alignment (v0.2):**

For each project, verify that canvas TITLE (first H1) matches canvas BODY (the H2 immediately under it) per role:

| Canvas | Expected H1 ending | Expected body H2 |
|---|---|---|
| Claude (tasks_canvas_id) | `\| Task Board` | `## seed Configuration` (or PLB/Seed legacy variants) |
| Human (human_canvas_id) | `\| Field Reference` | `## About This Canvas` |
| Hub (hub_canvas_id) | `\| Project Hub` | `## Project Status` |

Findings:
- 🔴 **body-population swap** — Claude canvas's body has `## About This Canvas` (Human-template body in Tasks-titled canvas), OR Human canvas's body has `## seed Configuration` (Tasks body in Field-Reference-titled canvas). Functional break — Routine reads wrong content type.
- 🟢 **title format drift** — title doesn't end with the expected suffix at all (e.g., legacy colon format, missing pipe). Cosmetic.

### 4. Write the Drift Report canvas

Full-canvas replace on `drift_report_canvas_id` per Canvas Write Rules (`contracts/canvas-write-rules.md` Rule 1: full-replace, no `section_id`). Title slot already has the canvas name; body should NOT include an H1 (Rule 5).

Body template:

```markdown
*Last updated: YYYY-MM-DD HH:MM PT by seed Drift Watcher Routine v0.1*

[If all clear:]

:white_check_mark: **All clear.** No drift detected between Smartsheet `Project Registry — Core` and your Routine Config canvas.

[If drift exists, one section per non-empty dimension:]

## Project ID coverage drift

### In Smartsheet but missing from Routine Config

|Project ID|Smartsheet Hub Canvas ID|Smartsheet Display Name|
|---|---|---|
|...|...|...|

### In Routine Config but missing from Smartsheet

|Project ID|Routine Config Hub Canvas ID|
|---|---|
|...|...|

## Hub Canvas ID mismatches

|Project ID|Smartsheet|Routine Config|
|---|---|---|
|...|...|...|

## Channel ID mismatches

|Project ID|Smartsheet|Routine Config|
|---|---|---|
|...|...|...|

## Human Canvas ID mismatches

|Project ID|Smartsheet|Routine Config|
|---|---|---|
|...|...|...|

## Cross-source F-ID consistency (v0.2)

[🔴 functional break — Routine reads wrong canvas — listed first]

|Severity|Project ID|Role|Smartsheet|Hub Registry|Routine Config|Claude seed-config|
|---|---|---|---|---|---|---|
|🔴|...|claude_canvas_id|—|F0AAA|F0BBB|F0AAA|

[🟡 schema drift]

[🟢 historical inconsistency]

## Cross-source UID consistency (v0.2)

|Severity|Project ID|Smartsheet|Hub Registry|Routine Config|Claude seed-config|
|---|---|---|---|---|---|
|🟡|P-XXXX|P-XXXX|P-XXXX|—|—|

## Rebrand consistency (v0.2)

|Project ID|Heading variant|`plb_*` body fields|`PLB-` value prefix|
|---|---|---|---|
|...|`## PLB Configuration`|yes|yes|

## Title-vs-content alignment (v0.2)

[🔴 body-population swap]

|Severity|Project ID|Canvas|Title H1|Body first H2|
|---|---|---|---|---|
|🔴|[EXAMPLE-PROJECT-ID]|Claude (F0XXXXXXXX1)|`... \| Field Reference`|`## seed Configuration`|

[🟢 title format drift]

|Severity|Project ID|Canvas|Title H1|Issue|
|---|---|---|---|---|
|🟢|[EXAMPLE-PROJECT-ID]|Claude|`[Example Display Name]: Tasks`|missing ` \| Task Board` suffix|

## Read failures (v0.2)

|Project ID|Source|Error|
|---|---|---|
|...|Hub canvas|access_denied|

---

*Detector only; remediation is human or CT or future headless-fix-loop. Use CT's "register this project" flow to add a missing project, or update the affected source manually. For Hub Canvas ID or Display Name reconciliation, the Hub canvas's `## Canvas Registry` section is canonical. For Claude Canvas F-ID reconciliation: trust the source where the canvas BODY is consistent with the canvas TITLE — Hub Registry is the tiebreaker if all bodies are also drifted.*
```

Omit any section whose drift list is empty. If ALL dimensions are empty, write only the "All clear" line below the timestamp.

### 4.5. Slack notification for 🔴 findings (v0.3)

**Gate conditions — skip this step entirely if either is false:**
1. `notification_channel_id` is set in bootstrap (non-null, non-`—`)
2. At least one 🔴 finding exists across dimensions 5 or 8 (functional break severity)

If both are true, call `slack_send_message` once to post to `notification_channel_id`. Construct the message as follows:

**Message format:**

```
🔴 *Drift Watcher found [N] functional break(s) — action needed*

<@CLAUDE_SLACK_USER_ID> Please create a GitHub issue in `{{github_repo}}` for each finding below. Tag it `drift-watcher` and `priority:high`. Use the spec at `SMARTSHEET_DRIFT_WATCHER.md` for remediation guidance.

*Findings:*

[For each 🔴 finding, one bullet:]
• *[Dimension name]* — Project `[PROJECT_ID]` ([Display Name]): [one-line description of the mismatch, which source to trust, and what to fix]

*Drift Report:* [canvas link — format as Slack URL: https://jobyaviation.enterprise.slack.com/docs/[team_id]/[canvas_id]]
*Repo:* https://github.com/{{github_repo}}
*Spec:* `SMARTSHEET_DRIFT_WATCHER.md` — Dimensions [N, N, ...]
```

Replace `<@CLAUDE_SLACK_USER_ID>` with the value from bootstrap field `claude_slack_user_id`. If `claude_slack_user_id` is not set or is `—`, omit the `<@...>` mention but still post the message (human will see it).

**Drift Report canvas link:** construct as `https://jobyaviation.enterprise.slack.com/docs/{{slack_team_id}}/[drift_report_canvas_id]` using the canvas ID from bootstrap. The `{{slack_team_id}}` is the Joby enterprise Slack workspace ID.

**Finding descriptions by dimension:**
- **Dimension 5 (Cross-source F-ID):** "Claude canvas ID in Routine Config (`[RC_value]`) disagrees with Hub Registry (`[Hub_value]`). Hub Registry is canonical — update Routine Config."
- **Dimension 8 (Body-population swap):** "`[canvas_role]` canvas (`[F-ID]`) has title ending `[title_suffix]` but body starts with `[body_h2]` — wrong template content in this canvas. Swap bodies or correct the title per the provisioning spec."

**If `slack_send_message` fails:** log the failure in chat and continue. The Drift Report canvas is already written; the notification is additive, not critical path.

**Do NOT send a notification if the only findings are 🟡 or 🟢 severity.** Those surface in the Drift Report canvas only.

### 5. End

Log a one-line summary: `Drift Watcher complete. [N] coverage drifts. [N] Hub ID mismatches. [N] Channel ID mismatches. [N] Human ID mismatches. [N] cross-source F-ID drifts (X 🔴). [N] UID drifts. [N] rebrand drifts. [N] title-vs-content drifts (X 🔴). [N] read failures. [N] Slack notifications sent.`

## Failure modes

- **Smartsheet read fails** → log, exit. Don't write Drift Report on partial state (would mask the outage as "no drift").
- **Routine Config canvas read fails** → log, exit.
- **Per-project Hub / Claude / Human canvas read fails (v0.2)** → record as `read_failures` finding for that project; skip the dimensions that depend on the missing source; continue with the remaining projects + dimensions. Surface read failures in the Drift Report so chronic access issues become visible.
- **Drift Report canvas write fails** → log the failure + paste the intended content as a code block in chat per Canvas Write Rule 4.
- **Single column missing from Smartsheet (data corruption)** → log warning, skip that column's drift dimension, continue with the others.
- **Hub Canvas Registry parse fails (v0.2)** — e.g., the bullet list isn't in the expected format. Log warning, treat that Hub as `hub_registry_read_failed`, continue.
- **Slack notification send fails (v0.3)** → log the failure, continue. The Drift Report canvas is already written. Notification failure does not re-trigger the canvas write.
