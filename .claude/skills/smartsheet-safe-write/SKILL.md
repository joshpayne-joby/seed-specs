---
name: smartsheet-safe-write
description: Canonical safe-write protocol for Smartsheet rows — read-before-write, mismatch check, optimistic-concurrency version guard, and verify-after. The mechanism is the precondition: invoking the skill IS reading current state, so a model cannot shortcut to a bare write. Required for any update_rows or add_rows call against `Project Registry — Core` (sheet `4898009463607172`) and applies as a default to any other Smartsheet sheet in the seed framework.
when_to_use: Any time a seed component writes to Smartsheet — CT v2.0+ per-person registry edits, Workshop project creation, PROJECT_SETUP.md Collaborator Sync, the daily SMARTSHEET_ROUTINE.md sync, or one-off ops. Consumer-agnostic — same flow whether adding a row or updating one. The consumer decides who is allowed to call it; this skill defines how the call must be shaped.
---

# Smartsheet Safe Write

Canonical protocol for any write to a Smartsheet row in the seed framework. This skill exists because of the AMFG-TEMPER incident (2026-05-01): a Claude instance wrote `Collaborators` and `Collaborator Names` without reading current state, overwriting a 7-name list with 2. Cell history attributed the corruption to the human who ran the instance. Both "always read before write" and "raise mismatches up" rules were violated.

The flow below makes that pattern structurally hard to repeat. Every call reads fresh, snapshots the version, checks for paired-column drift, re-checks version immediately before the write, then verifies after. No caching across invocations. No silent normalization.

## When to use

Apply on every write — `add_rows` or `update_rows` — against any Smartsheet sheet in the framework. There is no "small" or "trusted" write that skips this. The skill is the precondition; if you find yourself reaching for `update_rows` without having executed steps 1-6 below in this session, stop.

Consumers and what they pass:

| Consumer | Operation | Notes |
|---|---|---|
| CT v2.0 — PM editing own row | `update_rows` | Whitelist applies (see below). Read row by `Project ID` lookup. |
| Workshop — new project creation | `add_rows` | No `before_state` (row doesn't exist yet). Steps 4 and 5 collapse to "is this Project ID already in the sheet?" pre-check. |
| PROJECT_SETUP.md — Collaborator Sync | `update_rows` | Paired-column write (`Collaborators` + `Collaborator Names`). Mismatch check is load-bearing here. |
| SMARTSHEET_ROUTINE.md — daily sync | `add_rows` | Same as Workshop. Currently scoped to add-only; if the routine ever updates, it routes through this skill. |
| One-off / ops | either | Same flow. No exceptions for "just a small fix." |
| Drive Activity Crawler — Apps Script | `update_rows` | EXEMPTED from full flow under the "Machine-managed columns under single-writer lock" carve-out (see Failure modes). Writes only `Drive Activity Last Checked` + `Drive Activity Last Result` — both machine-only, no human edits. LockService prevents concurrent invocations. Verify-after still required. |

## Inputs the consumer must provide

- `sheet_id` (default: `4898009463607172` for Project Registry — Core)
- `operation`: `add` or `update`
- `target_row_key`: for updates, the `Project ID` to look up (or row ID if already known); for adds, the new `Project ID` to assert is unique
- `intended_cells`: map of `column_id` → new value. Keys are `column_id` (not column title) because the Smartsheet API writes by `column_id`. Callers resolve titles to IDs via `live_columns` (populated in Step 0) or, as a last resort, the quick-reference schema table at the bottom of this skill — which may lag the live sheet.
- `actor`: Slack user ID of the person whose Claude is writing (for the log line)
- `intent`: short string describing why (e.g. "PM self-edit Status to On Hold", "Collaborator Sync from Hub")

## The flow

Execute in order. Do not skip steps. Do not reorder steps 6 and 7.

### 0. Read live schema

Call `get_columns` on `sheet_id`. Capture as `live_columns` — a map of `column_title` → `column_id`. Every column reference in steps below resolves through `live_columns[title]` rather than a hardcoded ID — this is what lets the protocol stay correct when a new column is added to the sheet without a lockstep update to this skill.

If a column referenced in `intended_cells` (or in the paired-column mismatch check) is NOT present in `live_columns`, abort with:

> Column `<column_id>` from `intended_cells` not found in live schema for sheet `<sheet_id>`.
> Live columns: `<title>` → `<column_id>`, `<title>` → `<column_id>`, … (full mapping).
> Either the column was removed (drop the key from `intended_cells`) or the caller's `column_id` is wrong (re-resolve from `live_columns` or the quick-reference table at the bottom of this skill).

This step is cacheable for headless runs that do multiple writes in one session: call `get_columns` once per session, reuse `live_columns` across writes. Per slim-routines#5 (token budget), the one extra `get_columns` call per session is a meaningful cost amortization vs. calling per-write.

### 1. Snapshot sheet version

Call `get_sheet_version` on `sheet_id`. Capture as `version_at_start`. This is the optimistic-concurrency anchor.

### 2. Read the target row's current state

For `update`:

- Call `get_sheet_summary` on the sheet, locate the row whose `Project ID` (`live_columns["Project ID"]`) matches `target_row_key`.
- Capture the current values of every column listed in `intended_cells` AND any paired column (Collaborators ↔ Collaborator Names move together — read both even if only one is being written).
- Store as `before_state` keyed by `column_id`.
- For forensic-grade writes (corrections to a corrupted cell, etc.), additionally call `get_cell_history` on each affected cell to confirm prior author and value lineage.

For `add`:

- Call `get_sheet_summary` to confirm `target_row_key` is NOT already present in column `live_columns["Project ID"]`. If it is, abort with `Project ID already exists in sheet — switch to update operation or escalate to user`.
- **UID auto-generation (load-bearing — runs inside the version-snapshot window from step 1, not split across calls).** First, confirm the UID column exists; then scan it:
  - **UID column existence check (first).** If `live_columns["UID"]` is not present, abort with `UID column not present on sheet — schema does not support auto-gen. Add a "UID" column or call add_rows directly without UID auto-gen.` This is a schema-missing failure, distinct from the data-missing case below.
  - From the same `get_sheet_summary` result, scan the UID column (`live_columns["UID"]`).
  - If `intended_cells` already contains key `live_columns["UID"]`, abort with `UID is auto-generated; do not pass UID in intended_cells`.
  - If the sheet has zero rows, set `next_uid = "P-0001"`.
  - If the sheet has rows but the UID column is empty across all of them, abort with `UID auto-generation failed: column exists but no UID data on sheet. Backfill required before next add.` Never default to `P-0001` in this state — would collide post-backfill. Distinct from the schema-missing case above: column is present, values are missing.
  - Otherwise, parse the trailing integer of every present UID (`P-NNNN+`), find the max, increment by 1, render as `P-` + zero-padded value (≥4 digits). Set `next_uid` to the result.
  - Collision guard: confirm `next_uid` does NOT already appear in the scanned UID set. If it does, abort with `UID collision detected — re-run the operation; the auto-gen lost a race.` (Should be impossible with auto-gen + version-snapshot, but guard anyway.)
  - Inject `intended_cells[live_columns["UID"]] = next_uid`.
  - **Return contract.** The computed `next_uid` is available to the caller as `assigned_uid` for downstream writes (e.g., Routine Config canvas, Hub Canvas Registry KV list). Consumers reference this name — `assigned_uid` — by convention; don't rename without updating CT (Step 4 missing-project flow), PROVISIONER_WORKSHOP (Stage 10), and PROJECT_SETUP in lockstep.

**UID gap scan (runs on both `add` and `update`, after the per-operation checks above).** During the same `get_sheet_summary` read, collect every row where the UID cell is blank, null, or the literal string `—`. If any such rows are found, surface a non-blocking warning before proceeding — do NOT abort:

> UID gap detected: the following rows are missing a UID and should be assigned one before the next add operation produces an incorrect sequence.
> | Project ID | Display Name |
> |---|---|
> | `<project_id>` | `<display_name>` |
> | … | … |
> These rows need UIDs backfilled in Smartsheet. Use `update_rows` via this skill for each row — the UID must be supplied manually (not auto-generated) since auto-gen is reserved for `add_rows`. After backfill, confirm the assigned values are unique and sequential with the existing set.

This is the standing catch for projects that were created before the UID column existed (e.g., `AES-DBLCONEX`, which predates the UID sprint and was never backfilled). The warning fires every run until the gap is closed — it is intentionally noisy so it gets fixed.

### 3. Compute the diff explicitly

Before writing, render the diff in a form the model has to look at:

- **Single-value cells** (Status, Notes, Display Name, etc.): `before → after`.
- **CSV-list cells** (Collaborators, Collaborator Names): show the list before, the additions, the removals, and the resulting list. Preserve order of the existing entries; new entries append.
- **No-op cells**: if `before == after`, drop from the write entirely. Don't send unchanged values.

If the diff is empty after no-op pruning, log `idempotent — no write needed` and exit without calling `update_rows`.

### 4. Mismatch check on paired columns

This is the AMFG-TEMPER guard rail. For paired columns, verify `before_state` is internally consistent:

- `Collaborators` (`live_columns["Collaborators"]`) and `Collaborator Names` (`live_columns["Collaborator Names"]`) must have the same count of CSV entries and align positionally.
- If they don't (e.g., 1 Slack ID vs. 5 names, or 7 names vs. 2 IDs): **STOP**. Do not write. Surface the discrepancy to the user verbatim:

  > Paired-column mismatch on row `[Project ID]`:
  > Collaborators (N entries): `<csv>`
  > Collaborator Names (M entries): `<csv>`
  > Per the raise-mismatches-up rule, this needs you to resolve before I write. How should I reconcile these?

- Do NOT silently normalize, dedup, or pad. Do NOT "fix" by overwriting one side from the other. The sheet is in an inconsistent state and the user has to choose the resolution.

This rule applies even when the intended write only touches one side of the pair — because the write will compound the mismatch.

### 5. Self-write whitelist (CT consumer only)

When invoked by a Control Tower acting as the row's PM editing their own row, only these columns are writable:

- `Status` (`live_columns["Status"]`)
- `Notes` (`live_columns["Notes"]`)
- `Last Session` (`live_columns["Last Session"]`) — only if the actor IS the row's `Owner`
- `Collaborators` (`live_columns["Collaborators"]`) and `Collaborator Names` (`live_columns["Collaborator Names"]`) — only if the change adds/removes the actor's own entry; refuse edits that touch other people's entries

Refuse writes (return to user, don't escalate to a different code path) on:

- `Owner` (`live_columns["Owner"]`) — CONTACT_LIST type, no canonical email source today; document the gap and stop
- `Project ID` (`live_columns["Project ID"]`) — primary key, never edited post-creation
- `Domain` (`live_columns["Domain"]`) — derived from Project ID prefix, not editable
- `Hub Canvas ID` (`live_columns["Hub Canvas ID"]`), `Human Canvas ID` (`live_columns["Human Canvas ID"]`), `Claude Canvas ID` (`live_columns["Claude Canvas ID"]`), `Channel ID` (`live_columns["Channel ID"]`) — set at creation, structural
- `Added` (`live_columns["Added"]`) — historical, immutable

This whitelist is CT-specific. Workshop, PROJECT_SETUP, and Routine consumers operate under their own scoped permissions (set at creation, sync from Hub, etc.) — they don't go through this whitelist but are still bound to steps 1-4 and 6-9.

### 6. Re-check sheet version

Call `get_sheet_version` again. If the returned version does not equal `version_at_start`, abort:

> Sheet drifted under us — version was `<v0>` at read, `<v1>` now. Aborting write to avoid clobbering someone else's edit. Re-run the operation; the next call will pick up the current state.

This is the optimistic-concurrency check. The window between read and write is small but non-zero, and another writer (different CT instance, the daily Routine, a human in the Smartsheet UI) can land in it. Better to abort and retry than to overwrite.

### 7. Write

Call `update_rows` (or `add_rows`) with the diff from step 3. Send only the cells that changed.

For `update_rows`: include the row ID resolved in step 2 and the changed cells keyed by `column_id`. The Smartsheet API writes by `column_id`; the IDs in the write payload come from `live_columns` (resolved in Step 0), not from the quick-reference table at the bottom of this skill.

For `add_rows`: include the full new row.

### 8. Verify

Re-read the affected cells via `get_sheet_summary` (or `get_cell_history` for forensic writes). Confirm each `intended_cells[column_id]` now matches the post-write value. If any cell did not apply as expected, surface the discrepancy:

> Write verify mismatch on row `[Project ID]`, column `<column_id>`: expected `<after>`, sheet now shows `<actual>`. The write may have partially failed or been clobbered immediately.

### 9. Log the event

Emit a structured log line in chat:

```
[smartsheet-safe-write] sheet=<sheet_id> row=<Project ID> op=<add|update> actor=<slack_id> intent="<intent>" version_before=<v0> version_after=<v1> diff:
  <column_id> (<column_name>): <before> → <after>
  ...
```

Once the seed Activity Feed yearly-rolling sheets land, this becomes a row write to that sheet keyed on timestamp + sheet_id + row + actor. Until then, the chat log is the trail — the user can grab it for incident review or Cell-history correlation.

## Failure modes

| Mode | Response |
|---|---|
| Sheet version drift between step 1 and step 6 | Abort with the version-drift message. Instruct the user to retry; do NOT silently re-read and re-attempt. |
| Paired-column mismatch in `before_state` | Halt. Surface the discrepancy verbatim. Do not proceed without explicit user resolution. |
| `target_row_key` not found on update | Abort with `Project ID <id> not found in sheet — verify the ID or use add operation.` |
| `target_row_key` already present on add | Abort with `Project ID <id> already exists — use update operation.` |
| Smartsheet API error (rate limit, network, auth) | Surface the raw error to the user. Do NOT silently retry. The user may be on the wrong account, the sheet may be locked, or the connector may be down. |
| Owner column write requested | Refuse. Return: "Owner is CONTACT_LIST and has no canonical email source today. Set this manually in Smartsheet UI; documenting the gap." |
| Verify (step 8) shows the write didn't apply | Surface the mismatch. Do NOT auto-retry — the most likely cause is a concurrent writer, and retrying compounds the race. |
| `intended_cells` empty after no-op pruning | Log `idempotent — no write needed` and exit. Don't call `update_rows` with zero cells. |
| UID column missing from sheet schema (add op) | Abort with the schema-missing message from step 2. Distinct from "column present, data empty" — different fix path. |
| Column in `intended_cells` not in `live_columns` | Abort with the step 0 message. Caller passed a stale or wrong `column_id`. |
| Rows with missing UIDs detected during gap scan | Non-blocking warning. Surface the Project IDs and Display Names. Do NOT abort the current write. Continue through steps 3-9 normally. The warning is the signal — closing it is the operator's responsibility. |

## Carve-out — Machine-managed columns under single-writer lock (v0.5+)

A narrow exemption for writers whose target columns satisfy ALL of:

1. **Machine-managed only — target columns only.** No human edits the *written* columns. No other automation writes them. The exempted writer is the sole producer. *Source columns* read by the writer may be human-editable (e.g., the Drive Activity Crawler reads `Drive Folder URL`, which is PM-editable — this is fine; the constraint applies to the write set, not the read set).
2. **Single-writer lock enforced.** The writer prevents concurrent invocations of itself via a runtime lock primitive (Apps Script `LockService`, file lock, distributed lock, etc.). Two instances of the writer cannot race against each other.
3. **No paired-column semantics.** The exempted columns are not part of a `(name, id)` paired pair (like `Collaborators` + `Collaborator Names`) where mismatch detection is load-bearing.
4. **Verify-after still required.** The writer re-reads the affected rows after the PUT and confirms intended values landed. Mismatch throws.

Under these conditions, the writer may skip:
- Step 1 — sheet version snapshot (no concurrent writers means no race to lose)
- Step 2's UID auto-gen and gap scan (the exempted columns don't include UID)
- Step 4 — paired-column mismatch (no pairs)
- Step 5 — whitelist (no human consumer to gate)
- Step 6 — version recheck (already handled by the writer's own lock)

Steps 0 (schema read), 3 (compute diff), 7 (write), 8 (verify), and 9 (log) still apply. (Note: Step 3's idempotent no-op short-circuit will not fire for writers whose target columns include a timestamp that changes every run — e.g., `Drive Activity Last Checked`. The step still applies in principle; it's just never the optimization path for those writers.)

**Recorded exemptions:**

| Consumer | Target columns | Lock | Rationale |
|---|---|---|---|
| Drive Activity Crawler Apps Script (`routines/drive-activity-crawler/`) | `Drive Activity Last Checked`, `Drive Activity Last Result` | `LockService.getScriptLock()` | Apps Script can't invoke a Claude skill; columns are machine-only manifests; verify-after enforced in the script. Documented since v0.5 (2026-06-04). |

If you add a human-editable column to an exempted writer's write set, the carve-out no longer holds. Either (a) port the full safe-write protocol into the writer's runtime or (b) split the human-editable column off into a separate writer that uses the full protocol.

## What this skill does NOT do

- **Read or write Slack canvases.** Canvas write rules live in [`contracts/canvas-write-rules.md`](../../../contracts/canvas-write-rules.md) v1.0+ (canonical home; CT mirror + PROJECT_SETUP keep references). Different concern.
- **Decide who is allowed to call it.** The consumer's logic owns access control (CT decides "is this PM allowed to edit this row"; Workshop decides "is this person authorized to create projects"). This skill enforces the shape of the write, not the authorization to make it.
- **Cache state across invocations.** Every call reads fresh. There is no "last-known version" memo — the version snapshot is per-call. (Within a single session, `live_columns` from Step 0 may be cached across multiple writes — but that's session-scoped, not cross-invocation.)
- **Auto-retry.** Drift, API errors, and verify mismatches all surface to the user. Retry is the user's call.

## Reference — Project Registry — Core schema

Sheet ID: `4898009463607172` | Workspace ID: `7647124075636612`

| Column | Column ID | Type | Notes |
|---|---|---|---|
| UID | `2844130685521796` | TEXT_NUMBER | Surrogate key. Format `P-NNNN`, variable-width zero-pad ≥4 digits (at `P-9999` next is `P-10000` — no future format-rev migration). Auto-assigned by this skill at `add` operation; never manually set. Stable across Project ID renames. UIDs are sequential at issuance, sparse over time — never reuse, even if the original row is deleted. |
| Project ID | `5743034995347332` | TEXT | Primary key. Never edited post-creation. |
| Display Name | `3491235181662084` | TEXT | Sourced from Hub canvas `## Canvas Registry`. |
| Domain | `7994834809032580` | PICKLIST | Derived from Project ID prefix (AES, AMFG, TFAB, RSHOP, PSPEC, TEAM-AI). |
| Status | `676485414555524` | PICKLIST | PM-editable. |
| Hub Canvas ID | `5180085041926020` | TEXT | Set at creation. |
| Human Canvas ID | `2928285228240772` | TEXT | Set at creation. `—` if missing. |
| Claude Canvas ID | `3909624393928580` | TEXT | Slack canvas ID (F0XXX) of the project's Claude task board canvas. Paired with Hub Canvas ID and Human Canvas ID for the three-canvas pattern. Added 2026-05-11 per joby/project-routines#6 (Path A). |
| Channel ID | `7431884855611268` | TEXT | Set at creation. `none` if missing. |
| Owner | `1802385321398148` | CONTACT_LIST | NOT writable today (no email source). |
| Collaborators | `3407511236677508` | TEXT | CSV of Slack IDs. Paired with Collaborator Names. |
| Collaborator Names | `6583842740932484` | TEXT | CSV of names. Paired with Collaborators. |
| Last Session | `6305984948768644` | DATE | Owner-writable. |
| Added | `4054185135083396` | DATE | Set at creation. Immutable. |
| Notes | `8557784762453892` | TEXT | PM-editable. |
| Drive Folder URL | `6333790778855300` | TEXT_NUMBER | PM-editable. URL of the project's Drive folder. Consumed by the Drive Activity Crawler Apps Script to seed its recursive crawl. Blank → script skips the row. Added 2026-06-07 (v0.5). |
| Drive Activity Last Checked | `4081990965170052` | TEXT_NUMBER | Machine-managed by Drive Activity Crawler. ISO 8601 timestamp of the most recent crawl. TEXT_NUMBER (not ABSTRACT_DATETIME) for clean verify-after round-trip — Smartsheet may normalize ABSTRACT_DATETIME values on read-back, which would cause spurious verify failures. Do NOT human-edit. Added 2026-06-07 (v0.5). |
| Drive Activity Last Result | `8585590592540548` | TEXT_NUMBER | Machine-managed by Drive Activity Crawler. JSON manifest `{ new_files[], total_count, truncated, scanned_at, since }` of files added/modified since `Drive Activity Last Checked`. Do NOT human-edit. Added 2026-06-07 (v0.5). |

This table is a **quick reference for humans reading the doc**, not the canonical column-ID map for the protocol. As of v0.3, the protocol reads the live schema via `get_columns` as Step 0 and resolves every column reference through `live_columns`. IDs in the table above are accurate as of the date in the most recent changelog entry, but the protocol works correctly even when the table lags behind a schema change.

When IDs change (a column is added, renamed, or removed), update this table — but the protocol stays correct in the meantime because Step 0 reads live. Other docs (`SMARTSHEET_ROUTINE.md`, `canvas-specs/CONTROL_TOWER.md` Step 4, future Workshop and PROJECT_SETUP additions) should reference this skill rather than duplicate the column-ID table.

## Changelog

- **v0.5 — 2026-06-04** — Machine-managed columns under single-writer lock carve-out + 3-column schema migration for the Drive Activity Crawler. New section in Failure modes documents the carve-out conditions (machine-only target, runtime lock primitive, no paired-column semantics, verify-after retained) and the steps that may be skipped under them (1, 2's UID work, 4, 5, 6). Schema table gains `Drive Folder URL` (PM-editable, seeds the crawler), `Drive Activity Last Checked` (machine-managed timestamp), and `Drive Activity Last Result` (machine-managed JSON manifest). Column IDs marked TBD pending the actual Smartsheet migration — script aborts loudly if columns missing. *(IDs backfilled 2026-06-07 in PR #117 — see schema table for resolved values.)* Consumer table row added for the Drive Activity Crawler with the exemption rationale. The exemption exists because Apps Script can't invoke a Claude skill and porting the full protocol into the script is overkill for write sets that satisfy the four conditions; the carve-out is deliberately narrow and other exempted writers must be added to the exemption table here to be legitimized.
- **v0.4 — 2026-05-17** — Added UID gap scan to Step 2. On every `add` or `update` operation, after the per-operation checks, scan the UID column for rows where the value is blank, null, or `—`. Surface a non-blocking warning listing affected Project IDs and Display Names. Does not abort the current write. Fires on every run until the gap is closed. Added corresponding failure-mode row. Motivated by `AES-DBLCONEX`, which predates the UID sprint and has never been backfilled.
- **v0.3 — 2026-05-11** — Added Step 0 (read live schema via `get_columns`) and rewrote steps 2 / 4 / 5 / 7 to resolve every column reference through `live_columns[title]` rather than hardcoded IDs. The protocol now works correctly when a new column is added to a sheet without a lockstep update to this skill. Triggered by today's column add: `Claude Canvas ID` (id `3909624393928580`, index 7) was added to `Project Registry — Core` for Path A (joby/project-routines#6) without updating this skill in lockstep — Step 0 + live-resolved references mean that gap is non-fatal. Step 2's UID auto-gen branch begins with an explicit `live_columns["UID"]` existence check, separating the "column missing" failure mode (schema doesn't support auto-gen) from the existing "column present but empty" backfill abort. Schema reference table demoted to quick-reference status (still updated to include `Claude Canvas ID`). Per slim-routines#5 (token budget): the extra `get_columns` call is cacheable across multiple writes in one session.
- **v0.2 — 2026-05-08** — UID surrogate key added to `Project Registry — Core` schema. New TEXT_NUMBER column `UID` (column_id `2844130685521796`) at index 0, format `P-NNNN` variable-width zero-pad. Auto-generated by step 2 of the flow on every `add` operation; reads max-UID inside the version-snapshot window for race-safety. Three guard rails baked in: reject manual UID input, abort on missing UID column data (no `P-0001` default that would collide post-backfill), collision check on the computed value. Live data backfill of existing rows handled separately in numeric UID sprint Session 5 — see `notes/launch-checklist.md`.
- **v0.1 — 2026-04-30** — Initial version. Closes the gap exposed by the AMFG-TEMPER incident (2026-05-01). Defines the read-snapshot-mismatch-version-write-verify-log flow as the canonical safe-write protocol for any Smartsheet row in the seed framework. Bakes in the Project Registry — Core schema. Whitelist scoped to CT self-edit consumer; other consumers add their own scoped permissions on top of the same flow.
