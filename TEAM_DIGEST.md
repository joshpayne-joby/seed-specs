# TEAM_DIGEST.md — team-daily-digest v0.2

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
- `person_digest_channel_id` — OPTIONAL. Slack channel ID (C-prefix) for the **private per-person 1:1 log** (Josh-only). When set, after the public post fires the digest does a second in-memory pivot **by person** and posts each active person's one-line update as a threaded reply under their own root message in this channel (see Step 6 and the Per-person reply format). **If unset or `—`, skip the per-person view entirely** — the digest behaves exactly as it does today (graceful, mirrors `team_digest_channel_id` Step 5 gating). This channel is also the accumulator the future management rollup reads, and it's where the per-person intelligence the now-off per-person PM check-in DM used to carry (CLAUDE.md v2.15) now lands — privately.
  - **PRIVACY INVARIANT (load-bearing):** this MUST be a private, Josh-only Slack channel. The Routine posts per-person 1:1 detail here and does **not** verify channel membership — the privacy boundary is exactly whatever this channel's access is. Do not point it at any channel another person can read, and in particular it must never equal `team_digest_channel_id` (Step 6 refuses to post if they collide — see Step 6's skip conditions).

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

- One bullet per **project** that had activity: lead with the project's health emoji, then the **bold project display name** followed by its **project-channel pill** `<#channel_id>` (or just the bold display name if the project has no channel), then a one-line summary that names who did what inline ("…— Josh approved the P&I drawing; Adam closed the safety valves").
- **No per-person blocks and no per-person active/blocked counts** — those are deliberately excluded from this public post (they live in the runner's private 1:1-prep view). Name contributor *presence* inside a project line; never list who was absent.
- A **Quiet** line naming projects with no activity since `since`.
- A **Couldn't read** line for any access failures (omit if none).

### 5. Gate and post (exactly once)

Mirror the Drift Watcher's Step 4.5 gating:

- **Skip the post entirely** if any of: `team_digest_channel_id` is unset/`—`; OR there was **zero activity** across all projects since `since` (no empty digests — silence is fine); OR the no-double-post guard already fired in Step 1.
- Otherwise call `slack_send_message(channel_id=team_digest_channel_id, text=<digest>)` **once**. Capture the returned `ts`.
- **If `slack_send_message` fails:** log the failure in chat and continue. The digest is additive, not critical path — there is no canvas to roll back.
- **Per-person view (Step 6) gating** mirrors this step, scoped to `person_digest_channel_id`: skip the whole step if that field is unset/`—` (digest behaves exactly as today) or if it equals `team_digest_channel_id` (refuse to post per-person detail to the public channel); skip a given person if their thread already has a reply dated today (PT); a per-person post failure is logged and non-fatal. The per-person work runs **after** the public post above has fully returned (whether it succeeded, failed, or was skipped), so nothing in Step 6 can jeopardize the live public post.

### 6. Per-person 1:1 log (private)

This step runs **only after Step 5 has fully returned** — the public digest is composed and posted (its `ts` captured **on success**) before any per-person compose or post begins. It runs whether the public post **succeeded, failed, or was skipped for an unset `team_digest_channel_id`**: the per-person view is **additive, not critical path** — there is no canvas to roll back, and a per-person failure (or a stream-idle timeout *during* this step) can never touch the already-committed public post; the next run's per-person watermark backfills any person whose reply didn't land. **Never interleave Step 6 work into the public compose/post.** Everything below is a **second pivot of the Step-3 `(person, project, summary)` tuples — by person** — never a second Smartsheet or canvas read. (Read-budget: the only NEW reads this step adds are ONE `slack_read_channel` of `person_digest_channel_id`, plus at most batched thread-reads for the few people whose latest reply falls outside that read window (see sub-step 3), plus up to ~N `slack_search_users` person→ID lookups — only for people not already resolved from the project collaborators already in hand. When `mention_people=true` those lookups already happened in Step 4; when `false` (the default) they are additive here. At a handful of people this is negligible.)

**Skip the entire step** if any of:
- `person_digest_channel_id` is unset/`—` (the digest then behaves exactly as today); OR
- `person_digest_channel_id` equals `team_digest_channel_id` (or is otherwise the public team channel) — per-person detail must NEVER post to the public channel. Log `Per-person channel == team channel; refusing to post per-person detail publicly. Skipping.` and exit the step; OR
- Step 5 skipped on **zero activity** (no people to surface).

(The Step 1 no-double-post guard is **not** a skip condition here: if that guard fires it has already exited the *whole Routine* before this step — so a same-day re-run never reaches Step 6.)

Otherwise:

**Resolve a stable per-person key (do this BEFORE pivoting).** The public post never needed a stable person identity — it renders each contributor inline once and never re-finds them. The per-person view is the first consumer that must re-find the *same* person across runs (to locate their root) and render a consistent thread, so an unstable key would silently fork a person's 1:1 history across two roots. For each distinct `person` in the Step-3 tuples, resolve a canonical key: prefer the person's **Slack user ID** (resolve via the project's collaborators or `slack_search_users` — the same mechanism `mention_people` uses in the Message format). Key the pivot, the root-find, and the no-double-post guard on that resolved ID, and embed it as a stable anchor in the root body (`<!-- person:UXXXX -->`, mirroring the `<!-- PROJECT_ID -->` anchor CLAUDE.md uses for `existing_sections`). A `person` that does **not** resolve to an ID falls back to a single normalized display string used **identically** in the anchor, the root-find match, and the reply line — never a short name in one place and a full name in another. **Known limitation (display-name fallback only):** if a person never resolves to a UID *and* the session log names them differently across runs ("Adam" one day, "Adam Skinner" the next), the two normalized strings yield two anchors and fork that person's thread. The "prefer UID" rule above closes this on the primary path; the fallback carries the residual risk any name-keyed system has. Acceptable at small-team scale.

**Drop the `team` sentinel.** When pivoting by person, drop tuples whose person is the `team` sentinel (unattributed activity from Step 3.1) — the per-person 1:1 log is for named individuals only. Unattributed activity still appears in the public project line; it simply gets no per-person thread.

**Pivot by person** (the public post already pivoted these same in-memory tuples by project in Step 4 — second view, no re-read). Collapse a person's tuples for the day into one condensed, **project-attributed** one-line summary. **Activity-only:** a person is active iff they appear in ≥1 of the Step-3 tuples (already filtered to entries dated after `since`) — do **not** recompute activity, and do **not** post "no activity" lines. A quiet stretch in a person's thread is itself the signal. The `since` cursor for *which* activity to include is the **same single watermark** established in Step 1 — the per-person pivot does not compute a second `since`.

**Read the private channel ONCE.** Read `person_digest_channel_id` newest-first via `slack_read_channel` and hold it in memory — same read-once discipline as the public Step 1, and the same load-bearing reason (a redundant read costs the stream-idle budget). **Paginate to channel start the first time** so EVERY person's root is located: a root is created once and thereafter only replied-to, so its position is fixed at its creation `ts` and **sinks** over time as newer roots accumulate above it; a shallow read that doesn't reach an older root would create a duplicate and fork that person's history. From this single read build a `roots[person_key] → root_ts` map (sender-scoped + marker, per sub-step 1). **Do NOT re-read the channel per person.**

Then, for each active person:

1. **Find-or-create that person's root message.** From the single in-memory channel read, find the message **sent by the digest's own poster** — `owner_slack_id` when the connector posts as the runner (the Joby setup), or `claude_slack_user_id` if a workspace posts via a Claude app, marker-only fallback if neither ID is known; the **same sender-scope as Step 1** — whose body contains the per-person root marker `seed Per-Person Log` **and** carries this person's identity anchor (`<!-- person:UXXXX -->`, or the exact normalized display-name string for an unresolved person — match the whole anchor/name token, not a loose substring, so a shorter name can't anchor a longer-named person's thread). Sender-scoping rejects a human reply in the private channel that quotes the marker as a false root hit. If no root exists for this person, create it once via `slack_send_message` (root template below) and use its `ts` as the threading anchor; reuse that anchor on every subsequent run.
2. **Per-person no-double-post guard** (mirror of Step 1, scoped to the thread): derive the person's **latest reply date from the root message's `latest_reply` `ts`** — as already returned by the single channel read above (`slack_read_channel` carries each root's `latest_reply` / `reply_count` metadata, so the guard needs **no extra thread read**), authoritative — never parse a printed `*Jun 15*` string. If a reply dated **today (PT)** already exists, **skip this person** — never post twice in one PT day for one person. In the public-post-failure case this per-person `ts`-derived guard is the *only* thing preventing a duplicate reply on the following day, so it must be evaluated independently per person every run, never short-circuited by the public path.
3. **Otherwise post one threaded reply** under the root (`thread_ts` = the root's `ts`) using the daily reply template below. To stay in the stream-idle budget, derive each person's latest-reply `ts` from the single channel read above where the reply is already in the window; fall back to a `slack_read_thread` **only** for a person whose latest reply isn't in that window, and run any such fallback reads in **parallel batches of ~6** (same discipline as Step 3) — never a serial one-at-a-time loop.

**Per-person post failures are non-fatal and isolated.** A failure to find/create a root or post a reply for one person is logged and the step continues to the next person; it can never affect the public post (already sent in Step 5) or any other person's thread. This is still **read-only except Slack posts** — after this step the Routine's only writes are the one public message (Step 5) plus per-person thread replies here; no canvas, no Smartsheet cell, no DM.

### 7. End

Log a one-line summary:
> `Team Daily Digest complete. [M] projects active, [Q] quiet, [F] read failures. Posted [ts | skipped: reason].`

## Message format

Posted to `team_digest_channel_id` via `slack_send_message`. This is a **chat message, not a canvas** — use Slack chat mention form `<@U…>` (never the canvas form `![](@U…)`), and channel form `<#C…>` if linking channels.

```
:sunrise: *seed Team Daily — what's moving* · [date range: since → today]

• [health emoji] *Project Display Name* <#channel_id> — [one-line summary, contributor(s) named inline]
• [health emoji] *Project Display Name* <#channel_id> — [one-line summary, contributor(s) named inline]
• [health emoji] *Project Display Name* <#channel_id> — [one-line summary, contributor(s) named inline]

*Quiet:* Proj, Proj — no activity since last digest
*Couldn't read:* Proj ([error])   ← omit line if none

_seed Team Daily Digest · [M] active · [Q] quiet · since [date] · reply in-thread to add anything I missed_
```

Rules:
- **One bullet per active project** = `• [health emoji] *Display Name* <#channel_id> — [summary]`. The summary names the contributor(s) **inline** ("Josh approved …; Adam closed …"). Lead with the health emoji; **bold the project display name**, then its project-channel pill.
- **Link via the project channel, NOT the Hub canvas.** Put the project's channel as a chat pill `<#channel_id>` right after the bold display name (use the `Channel ID` from Step 2). ⚠️ Do **not** link the Hub canvas (`/docs/…` URL): Slack attaches any linked canvas as a *file* into the team channel — cluttering the channel's Files and broadening that canvas's access beyond its own project channel. A channel pill is clickable, attaches nothing, and the channel is the natural place to dig into a project. If the project's `Channel ID` is `none`, show just the bold display name (no pill, no link).
- **Mentions:** if `mention_people` is true, render each named contributor as `<@U…>` inline in the project line (resolve via the project's collaborators or `slack_search_users`; fall back to a plain name if no ID resolves). When false (the default), contributor names are **plain text, inline** — the parenthetical attribution style of the worked example (e.g. `(Josh)`, `(Adam)`), not bold; the project name is what's bolded. Mention each person at most once per project line; never for quiet projects.
- **No per-person blocks, no per-person counts.** Contributor names appear only inside a project's summary line. Who did *not* contribute is never listed — name presence, not absence.
- The footer marker `seed Team Daily Digest` is **load-bearing** — Step 1 finds the prior digest by this string. Do not remove or reword it.

### Worked example

```
:sunrise: *seed Team Daily — what's moving* · Jun 11 → Jun 12

• :large_blue_circle: *Olmar Autoclaves and Ovens* <#C0B1V3B5HSL> — OF 9685: safety valves closed (Josh, w/ Kunkle + Winsupply), P&I drawing approved, cooling locked to Dry Coolers Aqua-Vent, vacuum architecture in
• :red_circle: *Wing-Flip Gantry* <#C0ATZAHBLH> — Signal Cables ETA ~7/21 (FOB DE) is the new critical path (Josh); Alan's 6/29 return missed; AFA interim-config training decision open
• :white_check_mark: *HI Temp Thermoplastics Press* <#C0B1JRLKAAD> — frame weldment fit-up signed off (Adam)

*Quiet:* AMFG-TEMPER, AES-UNS — no activity since last digest

_seed Team Daily Digest · 3 active · 2 quiet · since Jun 11 · reply in-thread to add anything I missed_
```

### Per-person reply format

Posted only to `person_digest_channel_id` (Step 6) — the **private** 1:1 log. Per-person detail (the daily lines, any load / "active on N" signal) lives **only** here and **never** leaks into the public #advanced-equipment post, which stays project-centric with no per-person blanks. Same chat-message rules as the public post: Slack chat mention form `<@U…>` / channel pill `<#C…>` only, never the canvas form `![](@U…)`.

One **root** message per person (find-or-create, Step 6), with each day's update as a **thread reply** under it — opening a person's thread shows their full accumulated 1:1 history in one place.

**Root** (created once per person; the `<!-- person:U… -->` anchor is the stable identity key Step 6 re-finds the root by — see Step 6's "Resolve a stable per-person key"):

```
:bust_in_silhouette: *Adam Skinner* — 1:1 log
_seed Per-Person Log · private · replies accumulate daily_ <!-- person:U086… -->
```

**Daily reply** (one condensed, project-attributed line per active person per day):

```
*Jun 15* · Adam Skinner: HTPRESS weldment signed off, Olmar valves
```

Rules:
- **One line per person per day**, condensed and project-attributed (Josh's settled "one-line summary" choice). Use the person's canonical display name (the same string anchored in their root), not a short form — so the root and the reply name the person identically. Do not reintroduce a richer per-person format or any new status vocabulary — no per-person health emoji, no "active on N" count.
- The reply leads with the **date** (`*Jun 15*`) so the thread reads as a dated log, but the per-person no-double-post guard derives "already posted today" from the reply **`ts`**, never from this printed date string.
- The root marker `seed Per-Person Log` is **load-bearing** — Step 6 re-finds each person's root by this string (sender-scoped to the digest's own poster) plus the `<!-- person:U… -->` identity anchor. It must stay distinct from the public footer marker `seed Team Daily Digest` so the two watermarks remain independently greppable. Do not remove or reword either.

## Failure modes

- **Slack or Smartsheet unavailable** → log, exit. No partial post.
- **Smartsheet read returns zero non-archived rows** → log, exit (nothing to summarize).
- **Per-project canvas/channel read fails** → record under "Couldn't read"; continue with the rest. Chronic access failures (e.g. a project channel the runner isn't in) become visible in the post rather than silently missing.
- **Zero activity since `since`** → skip the post (no empty digests). Log `No activity since [since]; nothing to post.`
- **`slack_send_message` fails** → log, continue. No retry storm; the next day's run will cover the gap because `since` only advances when a post succeeds (the prior-digest watermark won't have moved).
- **Already posted today** → exit at Step 1; never double-post.
- **Per-person channel unset (`person_digest_channel_id` unset/`—`)** → skip Step 6 entirely; the digest behaves exactly as it does today (backwards-compatible, mirrors the `team_digest_channel_id` graceful-skip).
- **Per-person channel == team channel** → refuse to run Step 6; log `Per-person channel == team channel; refusing to post per-person detail publicly. Skipping.` This turns a copy-paste mis-config (which would otherwise post per-person 1:1 lines into the public channel) into a logged no-op.
- **Per-person root find/create or reply post fails (Step 6)** → log that person, continue to the next person. Non-fatal and isolated: it runs after the public post has fully returned, so it can never touch the already-sent public digest, and one person's failure never affects another's thread. No retry storm; the next run covers the gap because a person's per-thread watermark only advances when their reply posts.
- **Already posted today for a person (Step 6)** → that person's thread already has a reply dated today (PT, derived from the reply `ts`); skip that person, never double-post per person per day.
- **First run after `person_digest_channel_id` is set is denser** → it creates a root AND posts a reply for every active person (≈2× the steady-state per-person post count, since roots don't exist yet). Expected and harmless; normalizes once each person's root exists and subsequent runs post one reply per active person.

## Out of scope

- **Weekly management rollup.** The daily digest is project-centric — team progress at a glance. A formal management report ("how is each project going + where is the team's capacity") is a different artifact — different cadence (weekly), different surface (a canvas or a longer post), and likely a richer health model. The daily digest is designed to *feed* it (it already computes per-project health, and the underlying `(person, project)` activity tuples also feed the runner's private 1:1-prep view), but building the rollup is a deliberate follow-on, not v0.1.
- **Per-person load / 1:1-prep view — now folded in (v0.2).** This used to be out of scope ("a separate piece, not part of this Routine"). As of v0.2 it ships **inside this Routine** as the private per-person 1:1 log: when `person_digest_channel_id` is set, Step 6 posts each active person's one-line, project-attributed daily update as a threaded reply under their own root message in that private channel (see Step 6 and the Per-person reply format). It stays out of the public #advanced-equipment post (privacy hard rule). The future weekly management rollup will read these accumulated per-person threads. Still out of scope: any structured/self-reported capacity ("bandwidth for more") signal feeding it.
- **Multi-runner / multi-workspace.** v0.1 runs on one account against one team channel. A second team or workspace installing the digest (capturing `team_digest_channel_id` via Workshop provisioning) is deferred until there's a second consumer.
- **True milestone/completion detection.** The `:white_check_mark:` health code keys on decision/sign-off language in the session log and channel; it does not yet reconcile against a task board's done-state. Good enough for a daily glance; tighten if it misfires.

## Changelog

# v0.2 — 2026-06-15 — Folded-in private per-person 1:1 log
# - **What** — New optional bootstrap field `person_digest_channel_id` (Slack C-id) and a new **Step 6 — Per-person 1:1 log (private)** (existing End step renumbered 6 → 7). When the field is set, after the public post fires the Routine does a **second in-memory pivot by person** of the same Step-3 tuples (never a second read) and posts each active person's **one-line, project-attributed** update as a **thread reply** under a per-person **root** message (find-or-create; marker `seed Per-Person Log` + a stable `<!-- person:U… -->` identity anchor) in the private channel. Opening a person's thread = their full accumulated 1:1 history.
# - **Why** — Gives the runner a private, accumulating by-person view for 1:1 prep without a scramble, and redirects the per-person intelligence the now-off per-person PM check-in DM used to carry (CLAUDE.md v2.15) into a private surface — complementary to that change, not undoing it. Also becomes the accumulator the future weekly management rollup will read.
# - **Privacy hard rule** — Per-person detail (daily lines, any load signal) lives ONLY in `person_digest_channel_id`, which MUST be a private Josh-only channel; Step 6 refuses to run if it equals `team_digest_channel_id`. The public #advanced-equipment post is byte-unchanged: still project-centric, contributors named inline, no per-person blocks, no per-person blanks.
# - **Safe ordering (load-bearing)** — read → compose public → POST public (Step 5, unchanged, fires first and fully) → compose per-person → post per-person replies (Step 6). Step 6 runs whether the public post succeeded, failed, or was skipped, and is additive: a per-person failure (or a stream-idle timeout during Step 6) can never touch the already-sent public post; the next run backfills. Per-person no-double-post mirrors Step 1's guard, scoped per person/thread (today-check derived from the reply `ts`, not the printed date).
# - **Read-once / budget** — Step 6 adds exactly ONE `slack_read_channel` of `person_digest_channel_id` (paginated to channel start so no person's root is missed → no duplicate-root history fork), held in memory and never re-read per person; per-person thread reads are batched ~6 and only for people whose latest reply is outside that window. No second Smartsheet or project-canvas read.
# - **Backwards-compatible / graceful-skip** — `person_digest_channel_id` unset/`—` ⇒ Step 6 skipped entirely ⇒ the digest behaves EXACTLY as today (mirrors the `team_digest_channel_id` gating).
# - **Impact** — Spec-only, no re-paste for the behavior (the Routine WebFetches this spec live + auto-mirror to seed-specs). Activating the per-person view needs a ONE-TIME prompt re-paste on Josh's account to add the `person_digest_channel_id` C-id to the prompt's Configuration block; until then the field is absent ⇒ treated as unset/`—` ⇒ per-person view gracefully skipped. Same spec-auto / prompt-manual split as `team_digest_channel_id`.

# v0.1.1 — 2026-06-15 — Link project channels, not Hub canvases (stops canvas attachments)
# - **What** — Each active project's bold display name is now followed by its **project-channel pill** (`<#channel_id>`) instead of being linked to its Hub canvas URL. Channel-less projects show just the bold display name.
# - **Why** — Linking a Hub canvas (`/docs/…` URL) in a Slack message makes Slack **attach that canvas as a file** to the team channel — observed on the first live run (3 Hub canvases attached to #advanced-equipment), which clutters the channel's Files and broadens each canvas's access beyond its own project channel. A channel pill is clickable, attaches nothing, and the channel is the natural dig-in point.
# - **Impact** — Spec-only; no re-paste (the Routine WebFetches this spec live), and auto-mirror to seed-specs is restored, so the next scheduled run posts clean.
# - **Observed** — 2026-06-15 first live run (digest ts 1781555175.437389).

# v0.1 — 2026-06-12 — Initial version
# - New team-scoped Routine: one runner reads the whole `Project Registry — Core` sheet, fans out to each non-archived project's Claude canvas + channel for activity since the last digest, posts ONE project-centric summary to the team channel.
# - Structural clone of SMARTSHEET_DRIFT_WATCHER.md v0.3: one runner, whole-sheet read, parallel batched canvas reads (~6 at a time), single `slack_send_message` with graceful-skip gating.
# - Project-centric grouping: one bullet per active project with contributor(s) named inline + a per-project health emoji. No per-person blocks or load read in the public post — per-person load (the "bandwidth" axis) is deliberately the runner's private 1:1-prep view, not team-visible (Josh, 2026-06-13: "visibility into what individuals are doing but not for all to see").
# - Channel-as-watermark dedup: reads its own prior post (footer marker `seed Team Daily Digest`) for the `since` cursor and the no-double-post guard. No state canvas. `since` carries weekends forward.
# - Read-only everywhere except the single Slack post — no canvas write, so no canvas-write-rules surface.
# - Motivating case: per-person Daily Briefing check-in DMs all funnel to the PM (every prompt's PM Notification Target = Josh) and read like a mirror of the PM's own work, with no shared team view. The digest replaces that with one shared post. PR 2 (CLAUDE.md + prompts) gates the now-redundant per-person PM DM.
# - Out of scope: weekly by-project management rollup (the digest feeds it), the runner's private per-person 1:1-prep view, structured capacity field, multi-workspace, task-board-reconciled completion detection.
