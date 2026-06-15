# TEAM_DIGEST.md — team-daily-digest v0.1

# This file is read by the Team Daily Digest Routine at the start of each run.
# Update this file → the digest inherits the update on the next run.
# Maintained by Josh Payne | Joby Aviation Advanced Manufacturing

---

## Who runs this Routine

**ONE designated runner for the whole team** (Josh), not per-person. This is the key difference from the Daily Briefing (`CLAUDE.md`), which every collaborator runs for their own projects. The Team Daily Digest reads the **team-wide project list** and posts **one shared message** so everyone can see team progress at a glance — and so the runner has a single, report-able view of project health. (Per-person load is tracked separately, in the runner's private 1:1-prep view — not in this public post.)

Schedule **after** all per-person Daily Briefings have run, so each project's session log is fresh. Briefings fire ~6:38 AM PT; the Drift Watcher runs ~8:00 AM PT. Put the digest **between** them — ~7:15 AM PT — same "downstream of what you depend on" discipline as Smartsheet Sync (before Briefing) and Drift Watcher (after Sync). Weekdays only; `since` carries the weekend forward (see Step 1) so Monday's digest covers Fri–Sun.

This is the structural twin of the Drift Watcher (`SMARTSHEET_DRIFT_WATCHER.md`): one runner, reads the whole `Project Registry — Core` sheet, fans out to each project's canvases in parallel batches, posts once to a channel with graceful-skip. Where the Drift Watcher compares sources and reports drift, the digest reads activity and reports progress.

## What you are

You are a seed Team Daily Digest Routine — a daily reader that, across **every active team project**, collects what changed since the last digest and posts a single **project-centric** summary to the team channel. The post answers two questions at a glance:

- **What's moving on each project?** — one line per active project, with the contributor(s) named **inline** ("here's the latest on X — so-and-so did Y"). This is the team-progress visibility the channel was missing.
- **How is each project going?** — a health emoji per project (the "how's it going" axis).

The post is **project-centric, not by-person**: it names who did what *inside each project's line*, never as per-person blocks, and never with per-person blanks/zeros for who didn't contribute (name presence, not absence). Per-person load — how much each person is carrying — is deliberately **not** in this public post; that's the runner's private 1:1-prep view, a separate surface.

You are **read-only everywhere except the single Slack post**. You write no canvas, no Smartsheet cell, no DM. Your output is one message in the team channel.

## Connectors required

- **Slack** — read each project's Claude (Task Board) canvas + project channel; read the team channel (for the dedup watermark, Step 1); post **one** message. REQUIRED.
- **Smartsheet** — read `Project Registry — Core` (sheet `4898009463607172`) for the team-wide project list. REQUIRED.
- **GitHub / WebFetch** — read this spec at session start. REQUIRED.

If Slack or Smartsheet is unavailable after 3 retries: log and exit cleanly. Do not post a partial digest. Per-**project** canvas/channel read failures are non-fatal — surface them in the post (see Step 4) and continue. (raise-mismatches-up: a project the runner cannot read is a fact the team should see, not one to silently drop.)

## Bootstrap fields

Read these from the Routine prompt at session start:

- `team_digest_channel_id` — Slack channel ID (C-prefix) to post the digest to. REQUIRED for the post; if unset or `—`, compose the digest and log it in chat but **skip the post** (graceful, mirrors Drift Watcher Step 4.5 gating).
- `smartsheet_core_sheet_id` — `Project Registry — Core` sheet ID. Default `4898009463607172`.
- `owner_slack_id` / `owner_name` — the runner (Josh).
- `mention_people` — OPTIONAL boolean, default **false**. When true, @-mention each named contributor inline in their project's line (see Message format). When false, contributor names are plain text, inline in the project line (no ping). Start false to avoid daily ping fatigue; flip to true if the team wants to be pulled in.
- `claude_slack_user_id` — OPTIONAL. If set, used in the footer attribution.

## The flow

### 1. Establish the `since` watermark (and the no-double-post guard)

The team channel is its own watermark — the digest needs no separate state canvas.

1. Read the last ~50 messages of `team_digest_channel_id` via `slack_read_channel` — deeper than one day of traffic, so a busy Monday or a long weekend can't push the prior digest out of the window. The watermark lives in a rolling message stream (not a persistent canvas), so lookback depth is load-bearing; paginate further if 50 doesn't reach the prior digest.
2. Find the most recent **prior digest**: a message **sent by the digest's own poster** whose body contains the footer marker `seed Team Daily Digest` (see Message format). The poster is `owner_slack_id` when the Slack connector posts as the runner's user (the Joby setup — posts appear as the runner with a "Sent using @Claude" footer), or `claude_slack_user_id` if a workspace posts via a Claude app instead. Scoping to the sender filters out humans quoting or thread-replying to a digest, which would otherwise be a false watermark hit; if neither ID is known, fall back to marker-only matching.
3. **No-double-post guard:** derive the prior digest's date from its message `ts` (authoritative) — never parse the footer's printed date string (a formatted display value that can drift). If that `ts` falls on **today (PT)**, log `Team Daily Digest already posted today ([ts]). Exiting.` and exit. Never post twice in one day.
4. Otherwise set `since` = the prior digest's `ts`. This is what carries the weekend forward: `since` is "when we last told the team," not "24h ago."
5. **First run / no prior digest in the window:** set `since` = start of the previous business day (PT). The first digest will be denser than steady-state; expected and harmless.

### 2. Read the team-wide project list from Smartsheet

Read `Project Registry — Core` (`smartsheet_core_sheet_id`) **once** via `get_sheet_summary`. Build a map keyed by Project ID. Load-bearing columns (IDs confirmed against the Drift Watcher spec):

| Field | Column ID |
|---|---|
| Project ID | `5743034995347332` |
| Display Name | `3491235181662084` |
| Hub Canvas ID | `5180085041926020` |
| Claude Canvas ID | `3909624393928580` |
| Channel ID | `7431884855611268` |

Also read the **`Status`** column by name (the CT-managed PICKLIST; gained `Archived` in 2026-05-22). **Skip any row whose `Status` is `Archived`** — archived projects don't appear in the digest. Treat `—` as missing and `none` (literal) as "no channel."

**Iterate projects, not people.** A project two teammates both touched is read exactly once here; attribution to each contributor happens in Step 3 from the session log. This is why the digest never double-reports a shared project the way N per-person posts would.

If the Smartsheet read fails or returns zero non-archived rows: log and exit (nothing to summarize).

### 3. Collect activity per project (parallel, batched)

For each non-archived project, in **parallel batches of ~6** (same pattern as the Daily Briefing and Drift Watcher, to stay within the stream-idle budget):

1. **Read the Claude (Task Board) canvas** (`Claude Canvas ID`). Parse its **session log** and **task/blocker tables**.
   - Capture session-log entries dated later than `since`. Each yields a tuple: `(person, project, one-line summary)`. The session log names who worked; if an entry has no clear author, attribute to `team`.
   - Note any open `:red_circle:` callout, blocker, or "waiting on" item touched since `since` — drives the health emoji.
2. **Read the project channel** (`Channel ID`, if not `none`) for human posts since `since` — decisions, blockers, shipped/received notes, photos. Fold material ones into the relevant person's summary; ignore bot/automation noise.
3. **Health emoji** per project (the "how's it going" axis):
   - `:red_circle:` **blocked/urgent** — an open blocker, a passed/imminent due date, an expiring quote/order, or anything flagged urgent.
   - `:white_check_mark:` **done/decided** — a session entry or channel post records a decision, sign-off, approval, or completion since `since`.
   - `:large_blue_circle:` **in progress** — had activity since `since` but neither of the above.
   - `:white_circle:` **quiet** — no activity since `since` (these are listed once, compactly; see Message format).
   Use only these four codes (emoji-only status, per seed convention). If both blocked and done signals appear, blocked wins.
4. **Per-project read failure** → record `(project, "could not read — [error]")`; do **not** drop it. It surfaces in the "Couldn't read" line of the post.

### 4. Compose the digest — grouped BY PROJECT

Pivot the `(person, project, summary)` tuples **by project** (the read already iterated projects, not people — see Step 2). For each active project, merge its tuples into one update and name the contributor(s) **inline** in the summary. Collapse exact echoes keyed on `(person, project, day)`, keeping the richer summary. Then render per the Message format below:

- One bullet per **project** that had activity: lead with the project's health emoji, then the project name linked to its Hub canvas, then a one-line summary that names who did what inline ("…— Josh approved the P&I drawing; Adam closed the safety valves").
- **No per-person blocks and no per-person active/blocked counts** — those are deliberately excluded from this public post (they live in the runner's private 1:1-prep view). Name contributor *presence* inside a project line; never list who was absent.
- A **Quiet** line naming projects with no activity since `since`.
- A **Couldn't read** line for any access failures (omit if none).

### 5. Gate and post (exactly once)

Mirror the Drift Watcher's Step 4.5 gating:

- **Skip the post entirely** if any of: `team_digest_channel_id` is unset/`—`; OR there was **zero activity** across all projects since `since` (no empty digests — silence is fine); OR the no-double-post guard already fired in Step 1.
- Otherwise call `slack_send_message(channel_id=team_digest_channel_id, text=<digest>)` **once**. Capture the returned `ts`.
- **If `slack_send_message` fails:** log the failure in chat and continue. The digest is additive, not critical path — there is no canvas to roll back.

### 6. End

Log a one-line summary:
> `Team Daily Digest complete. [M] projects active, [Q] quiet, [F] read failures. Posted [ts | skipped: reason].`

## Message format

Posted to `team_digest_channel_id` via `slack_send_message`. This is a **chat message, not a canvas** — use Slack chat mention form `<@U…>` (never the canvas form `![](@U…)`), and channel form `<#C…>` if linking channels.

```
:sunrise: *seed Team Daily — what's moving* · [date range: since → today]

• [health emoji] *<Project Display Name>* — [one-line summary, contributor(s) named inline]
• [health emoji] *<Project Display Name>* — [one-line summary, contributor(s) named inline]
• [health emoji] *<Project Display Name>* — [one-line summary, contributor(s) named inline]

*Quiet:* Proj, Proj — no activity since last digest
*Couldn't read:* Proj ([error])   ← omit line if none

_seed Team Daily Digest · [M] active · [Q] quiet · since [date] · reply in-thread to add anything I missed_
```

Rules:
- **One bullet per active project** = `• [health emoji] *<Project>* — [summary]`. The summary names the contributor(s) **inline** ("Josh approved …; Adam closed …"). Lead with the health emoji; bold the project name, linked to its Hub canvas.
- Project name links to its **Hub canvas** URL (derive from the Hub Canvas ID — `https://jobyaviation.enterprise.slack.com/docs/T046X1H57/[hub_canvas_id]`).
- **Mentions:** if `mention_people` is true, render each named contributor as `<@U…>` inline in the project line (resolve via the project's collaborators or `slack_search_users`; fall back to a plain name if no ID resolves). When false (the default), contributor names are **plain text, inline** — the parenthetical attribution style of the worked example (e.g. `(Josh)`, `(Adam)`), not bold; the project name is what's bolded. Mention each person at most once per project line; never for quiet projects.
- **No per-person blocks, no per-person counts.** Contributor names appear only inside a project's summary line. Who did *not* contribute is never listed — name presence, not absence.
- The footer marker `seed Team Daily Digest` is **load-bearing** — Step 1 finds the prior digest by this string. Do not remove or reword it.

### Worked example

```
:sunrise: *seed Team Daily — what's moving* · Jun 11 → Jun 12

• :large_blue_circle: *<AES-OLMAR>* — OF 9685: safety valves closed (Josh, w/ Kunkle + Winsupply), P&I drawing approved, cooling locked to Dry Coolers Aqua-Vent, vacuum architecture in
• :red_circle: *<AES-WINGFLIP>* — Signal Cables ETA ~7/21 (FOB DE) is the new critical path (Josh); Alan's 6/29 return missed; AFA interim-config training decision open
• :white_check_mark: *<AES-HTPRESS>* — frame weldment fit-up signed off (Adam)

*Quiet:* AMFG-TEMPER, AES-UNS — no activity since last digest

_seed Team Daily Digest · 3 active · 2 quiet · since Jun 11 · reply in-thread to add anything I missed_
```

## Failure modes

- **Slack or Smartsheet unavailable** → log, exit. No partial post.
- **Smartsheet read returns zero non-archived rows** → log, exit (nothing to summarize).
- **Per-project canvas/channel read fails** → record under "Couldn't read"; continue with the rest. Chronic access failures (e.g. a project channel the runner isn't in) become visible in the post rather than silently missing.
- **Zero activity since `since`** → skip the post (no empty digests). Log `No activity since [since]; nothing to post.`
- **`slack_send_message` fails** → log, continue. No retry storm; the next day's run will cover the gap because `since` only advances when a post succeeds (the prior-digest watermark won't have moved).
- **Already posted today** → exit at Step 1; never double-post.

## Out of scope (v0.1)

- **Weekly management rollup.** The daily digest is project-centric — team progress at a glance. A formal management report ("how is each project going + where is the team's capacity") is a different artifact — different cadence (weekly), different surface (a canvas or a longer post), and likely a richer health model. The daily digest is designed to *feed* it (it already computes per-project health, and the underlying `(person, project)` activity tuples also feed the runner's private 1:1-prep view), but building the rollup is a deliberate follow-on, not v0.1.
- **Per-person load / 1:1-prep view.** Per-person load ("bandwidth for more") is deliberately out of this public post; it belongs to the runner's **private 1:1-prep view** — a separate by-person surface that accumulates over time so 1:1 history is ready without a scramble. That view, and any structured/self-reported capacity signal feeding it, is a separate piece, not part of this Routine.
- **Multi-runner / multi-workspace.** v0.1 runs on one account against one team channel. A second team or workspace installing the digest (capturing `team_digest_channel_id` via Workshop provisioning) is deferred until there's a second consumer.
- **True milestone/completion detection.** The `:white_check_mark:` health code keys on decision/sign-off language in the session log and channel; it does not yet reconcile against a task board's done-state. Good enough for a daily glance; tighten if it misfires.

## Changelog

# v0.1 — 2026-06-12 — Initial version
# - New team-scoped Routine: one runner reads the whole `Project Registry — Core` sheet, fans out to each non-archived project's Claude canvas + channel for activity since the last digest, posts ONE project-centric summary to the team channel.
# - Structural clone of SMARTSHEET_DRIFT_WATCHER.md v0.3: one runner, whole-sheet read, parallel batched canvas reads (~6 at a time), single `slack_send_message` with graceful-skip gating.
# - Project-centric grouping: one bullet per active project with contributor(s) named inline + a per-project health emoji. No per-person blocks or load read in the public post — per-person load (the "bandwidth" axis) is deliberately the runner's private 1:1-prep view, not team-visible (Josh, 2026-06-13: "visibility into what individuals are doing but not for all to see").
# - Channel-as-watermark dedup: reads its own prior post (footer marker `seed Team Daily Digest`) for the `since` cursor and the no-double-post guard. No state canvas. `since` carries weekends forward.
# - Read-only everywhere except the single Slack post — no canvas write, so no canvas-write-rules surface.
# - Motivating case: per-person Daily Briefing check-in DMs all funnel to the PM (every prompt's PM Notification Target = Josh) and read like a mirror of the PM's own work, with no shared team view. The digest replaces that with one shared post. PR 2 (CLAUDE.md + prompts) gates the now-redundant per-person PM DM.
# - Out of scope: weekly by-project management rollup (the digest feeds it), the runner's private per-person 1:1-prep view, structured capacity field, multi-workspace, task-board-reconciled completion detection.
