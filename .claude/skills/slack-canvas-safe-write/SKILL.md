---
name: slack-canvas-safe-write
description: Canonical safe-write protocol for Slack canvases — reject `section_id`, read-before-write, title/H1 collision check, H1-rename ghost detection, @mention read↔write translation, verify-after, and paste-fallback on failure. The mechanism is the precondition: invoking the skill IS reading current canvas state and shaping the call, so a model cannot shortcut to a bare `slack_update_canvas`. Required for any `slack_update_canvas` or `slack_create_canvas` call from any seed framework component.
when_to_use: Any time a seed component writes to a Slack canvas — Workshop canvas provisioning (Hub / Claude / Human / Routine Config / My Tasks / Prime / seed Changelog), Control Tower writes to Routine Config and My Tasks, the daily briefing Routine's end-of-run My Tasks full-replace, the provisioner Hub patch, Drift Watcher report writes, or one-off ad-hoc writes via the Slack MCP. Consumer-agnostic — same flow whether creating a canvas or full-replacing one. The consumer decides who is allowed to call it; this skill defines how the call must be shaped.
---

# Slack Canvas Safe Write

Canonical protocol for any write to a Slack canvas in the seed framework. This skill exists because the canvas-write rules in [`contracts/canvas-write-rules.md`](../../../contracts/canvas-write-rules.md) v1.1+ were documentation-only: every prompt that drives a canvas write inlined the six rules into its system prompt and trusted the model to follow them. Drift happened. Section-replace ghost blocks, title-vs-H1 double headers, H1-rename ghosts, write-from-memory clobber, and untranslated `<@USERID>` mentions have each been observed in production despite the rules being written down somewhere.

The flow below makes those patterns mechanically hard to repeat. Every call reads the canvas fresh, rejects `section_id`-style writes outright, translates `<@USERID>` / `<#CHANID>` to the `![](@USERID)` / `![](#CHANID)` write format, checks for an H1 rename, performs the full-canvas replace, verifies after, and falls back to a paste-in-chat block on API failure. No caching across invocations. No silent normalization of paired formats.

## When to use

Apply on every write — `slack_update_canvas` or `slack_create_canvas` — against any Slack canvas in the framework. There is no "small" or "trusted" canvas edit that skips this. The skill is the precondition; if you find yourself reaching for `slack_update_canvas` without having executed steps 0-8 below in this session, stop.

Consumers and what they pass:

| Consumer | Operation | Notes |
|---|---|---|
| Workshop PROJECT mode (Stages 4-6 — Hub, Claude, Human canvas creation) | `create` | One call per canvas. Title goes in `title` parameter only; body MUST NOT start with `# {title}`. |
| Workshop PROJECT mode (Stage 7 — Hub patch with Claude canvas ID) | `update` | Full-replace, no `section_id`. |
| Workshop PROJECT mode (Stage 8 — Routine Config row write at Phase 1+) | `update` | Routine Config canvas. Step 5 dual-write activates if the read shows a `` ```toon `` block (backtick-only per `contracts/toon-canvas-block.md` v1.1). |
| Workshop COLLAB mode (Stages C7-C8 — My Tasks + Routine Config create) | `create` | One call per canvas; same title/H1 rule as PROJECT mode. |
| Workshop BOOTSTRAP mode (Stages B6-B7 — Prime + seed Changelog create) | `create` | Same. |
| CT Step 4 missing-project fallback (Routine Config row append) | `update` | Read → in-memory append → full-replace. Dual-write rule applies if TOON present. |
| CT Step 4 missing-project fallback (My Tasks Project Registry row append) | `update` | Read → in-memory append → full-replace. |
| Briefing Routine end-of-run My Tasks write | `update` | Full-replace. The Routine composes its body fresh; the skill enforces the shape. |
| Drift Watcher Drift Report write | `update` | Full-replace. |
| Ad-hoc / manual writes via Slack MCP | either | Same flow. No exceptions for "just a small fix." |

## Inputs the consumer must provide

- `operation`: `create` or `update`
- For `create`: `title` (string — Slack title slot) and `intended_body` (markdown for the canvas body)
- For `update`: `canvas_id` (Slack F-prefix ID) and `intended_body` (full canvas markdown to replace with)
- `actor`: Slack user ID of the person whose Claude is writing (for the log line)
- `intent`: short string describing why (e.g. "Routine end-of-run My Tasks write", "CT Step 4 missing-project add")
- The consumer MUST NOT pass a `section_id` parameter — the skill rejects in step 0 if it is present in the call surface. Section-level replace is universally unsafe (canvas-write-rules.md Rule 2).

## The flow

Execute in order. Do not skip steps. Do not reorder steps 5 and 6.

### 0. Reject `section_id`; guard empty `intended_body`

**Reject `section_id`.** If the consumer's call shape includes a `section_id` parameter (anywhere — top-level, nested in a payload, anywhere), abort immediately:

> Section-level canvas writes are universally unsafe (canvas-write-rules.md Rule 2 — ghost-block bug, all block types). Drop `section_id` and pass `intended_body` as the full canvas markdown for full-canvas replace via this skill.

This is the structural guard. The skill's API never accepts `section_id`; if a caller is reaching for it, they are reading the wrong contract.

**Guard empty `intended_body`.** If `intended_body` is missing, empty, whitespace-only, or trivially short (< 1 non-blank line of content), abort:

> `intended_body` is empty or trivially short. The skill does not write blank canvases — the consumer is almost certainly invoking with a bug (compose failed silently, variable shadowed, or upstream read returned nothing). Compose the canvas body and re-invoke; if the intent is to clear a canvas's contents, do it manually in the Slack UI.

This catches the most common upstream-bug shape: a compose function returned `""` or `None` and the consumer didn't notice before reaching the skill. The skill is the last line of defense.

### 1. Read current canvas state (update only)

For `update`:

- Call `slack_read_canvas` with `canvas_id`. Capture as `current_canvas` with fields `title`, `body_markdown`.
- This read IS the precondition. A consumer cannot legitimately shortcut to a bare write — the skill's invocation flow forces the read to happen.
- Preserve `body_markdown` verbatim for steps 2-5 — the H1 check, the @mention scan, and the dual-write detection all consume from this single in-memory copy. Do not re-read.

For `create`:

- Skip this step. There is no current state.

### 2. Title / body H1 collision check (create only)

For `create`:

- Slack stores the title slot (set by the `title` parameter) and the body H1 (the first `# ` line in body markdown) independently and renders BOTH. If the body starts with `# {title}` matching the title parameter, the rendered canvas shows the title twice.
- Scan the first non-blank line of `intended_body`. If it matches `# <title>` (case-insensitive, whitespace-trimmed against the `title` parameter), abort:

> Body starts with `# {title}` matching the title parameter — Slack will render the title twice (title slot + body H1). Drop the leading `# {title}` line from `intended_body`; the body should start with the first paragraph or `##` sub-heading.

Strict reject, not silent strip. A silent strip would mask the consumer's mental model — better to surface the rule once and let the consumer fix it at the source.

For `update`: skipped. The body H1 on `update` is governed by step 3 (ghost-h1) instead.

### 3. H1-rename ghost detection (update only)

For `update`:

- Extract the first `# ` H1 line (if any) from `current_canvas.body_markdown` as `current_h1`. Strip leading and trailing whitespace (`.trim()` semantics).
- Extract the first `# ` H1 line (if any) from `intended_body` as `intended_h1`. Same strip.
- If both exist and `current_h1 != intended_h1` (after the strip), surface a halt-for-confirmation warning:

> H1 rename detected on canvas `<canvas_id>`:
>   existing: `<current_h1>`
>   intended: `<intended_h1>`
> Full-canvas replace does NOT remove a renamed H1 — the old H1 will persist as a ghost block above the new content, and only Slack UI deletion can clear it (canvas-write-rules.md Rule 6).
> Options:
>   1. Proceed and plan a manual UI cleanup pass to delete the ghost (acceptable for one-time renames).
>   2. Abort and revert `intended_body`'s H1 to match `<current_h1>` (no ghost, no cleanup).
> Confirm which.

Halt until the consumer (or surrounding agent) confirms. The skill does NOT silently proceed or silently abort — H1 rename is rare enough that surfacing the decision is the right cost.

If `current_h1 == intended_h1` (or both are absent, or one is absent and the other is present), no warning. The condition is gated on "both exist" — any case where either side lacks an H1 falls through to the no-warning path. Full-replace cleans up cleanly when the H1 is stable, removed entirely, or newly added.

The trailing-whitespace strip on both sides is load-bearing: real canvases in the wild carry incidental whitespace drift on H1 lines (e.g., `# Josh | Routine Config ` with a trailing space vs the conventional `# Josh | Routine Config` with no trailing space — both observed on `F0B19UB8H0S` 2026-06-01). Without the strip, an `update` that wrote the conventional form against a canvas with the trailing-space form would fire a false-positive rename warning every single time, requiring an operator confirm-loop for what's mechanically a no-op rename. Strip first, compare after.

### 4. `@mention` and `#channel` translation

`slack_read_canvas` returns mentions in angle-bracket format; `slack_update_canvas` and `slack_create_canvas` require image-syntax format. Translate every occurrence in `intended_body` before write:

- `<@USERID>` → `![](@USERID)` for any Slack user ID (`U` or `W` + 8-12 alphanumeric).
- `<#CHANID>` → `![](#CHANID)` for any Slack channel ID (`C` + 8-12 alphanumeric).
- `<#CHANID|name>` (the labeled channel-mention form) → `![](#CHANID)` (drop the label, the image syntax doesn't carry it).

The translation is idempotent: if `intended_body` already uses `![](@USERID)`, the regex doesn't match and nothing changes. The consumer does not have to remember to do this — the skill always does it.

Record the count of translations made for the log line in step 8.

### 5. Routine Config dual-write detection (conditional)

Detect whether the target canvas is the Routine Config canvas. Use the canvas title — already in memory from step 1 — as the canonical signal. Title-based detection is immune to body-content false fires (a Hub canvas referencing a Routine Config table, a how-to doc embedding the column header, a pre-v2.8 My Tasks canvas with a legacy `## Routine Config` section all match body-content checks but are not Routine Config canvases).

- For `update`: `current_canvas.title` contains `Routine Config` (Slack title-slot string, matched case-insensitively).
- For `create`: `title` parameter contains `Routine Config` (same match, just sourced from the call surface).

If yes, AND (for `update`) the existing `body_markdown` contains a `` ```toon `` fenced block:

- This is the Phase 2 dual-write state per [`contracts/routine-config-canvas.md`](../../../contracts/routine-config-canvas.md) v2.4 (TOON dual-write) and [`canvas-specs/CONTROL_TOWER.md`](../../../canvas-specs/CONTROL_TOWER.md) v2.0.6 (markdown-as-composition-source rule). Both contracts live on `main` as of 2026-05-29 — the rule below is the current state, not forward-compat speculation.
- `intended_body` MUST contain BOTH:
  1. The markdown table with header `|UID|Project ID|Claude Canvas ID|...|`
  2. A `` ```toon `` fenced block with `meta.id: routine-config`, `meta.version: 1`, and a `projects[N]{uid,project_id,claude_canvas_id,hub_canvas_id,human_canvas_id,channel_id,claude_project_url}` array per `contracts/toon-canvas-block.md` v1.1
- If either is missing, abort:

  > Routine Config canvas is in Phase 2 dual-write state (existing canvas has a `` ```toon `` block) but `intended_body` is missing <markdown table | TOON block | both>. Phase 2 requires BOTH formats in a single full-replace per `contracts/routine-config-canvas.md` v2.4 and `canvas-specs/CONTROL_TOWER.md` v2.0.6. Compose both formats from the in-memory markdown-table read (markdown-as-composition-source rule) and re-invoke.

- Row-count cross-check: parse the row count from the markdown table (excluding header + separator) and from the TOON `projects[N]` header. If they don't match, abort with the count mismatch surfaced verbatim. The Phase 2 invariant is that TOON is derived from markdown — if counts diverge, one of the two was composed from a different source, violating markdown-as-composition-source.
- The skill cannot mechanically verify "where did the in-memory list actually come from" (Smartsheet enumeration vs. markdown read), but row-count match + presence-of-both-formats is the strongest mechanical signal short of executing the composer.

If the read canvas did NOT contain a `` ```toon `` block, this rule is a no-op for this invocation (the canvas hasn't been touched by a TOON-aware writer yet — pre-cutover state under `CLAUDE.md` v2.12 Branch C narrow fallback). The next CT or Workshop write that emits the TOON block transitions the canvas into Phase 2 dual-write; subsequent invocations against that canvas enforce both formats.

TOON fence syntax: backtick form only (`` ```toon ``), per `contracts/toon-canvas-block.md` v1.1 — there is no tilde form on the contract.

For `create` of a Routine Config canvas: the skill emits a non-blocking notice if `intended_body` lacks a TOON block. Workshop's COLLAB-mode create at v0.6+ should emit the empty-projects TOON form (`projects[0]{...}:` with no rows) alongside the markdown placeholder row per `contracts/routine-config-canvas.md` v2.4 empty-projects case. The skill does not abort `create` — the notice is informational so an operator running an out-of-date Workshop hears the gap.

### 6. Write

For `create`:

- Call `slack_create_canvas` with `title=<title parameter>` and the canvas body set to `intended_body` (after the step 4 @mention translation). No leading `# {title}` in the body — step 2 already enforced this.

For `update`:

- Call `slack_update_canvas` with `canvas_id=<canvas_id>`, `action=replace`, and the canvas body set to `intended_body` (post-translation). NO `section_id` parameter — the skill never sets one.

If the API call raises an error (HTTP non-2xx, connector error, auth failure, rate limit, payload size, etc.), proceed to step 8 (paste fallback). Do NOT auto-retry.

### 7. Verify

Re-read the canvas via `slack_read_canvas`:

- For `create`: confirm `title` matches the parameter passed; confirm `body_markdown` begins with the same first non-blank line as `intended_body` (modulo the expected normalizations enumerated below).
- For `update`: confirm `body_markdown` matches `intended_body` modulo the expected normalizations.

Expected normalizations to apply before diffing — a difference matching one of these is NOT a verify failure:

1. **@mention round-trip.** `![](@USERID)` written → `<@USERID>` read back; `![](#CHANID)` written → `<#CHANID>` read back. Normalize both sides to the read form before comparing.
2. **NBSP-vs-ASCII-space drift in fenced code-block indentation.** Slack converts leading ASCII spaces inside fenced code blocks to non-breaking spaces (`U+00A0`) on storage — see [`contracts/canvas-write-rules.md`](../../../contracts/canvas-write-rules.md) v1.3 companion notes #4 and [`contracts/toon-canvas-block.md`](../../../contracts/toon-canvas-block.md) v1.1 "Indentation whitespace." Normalize both sides by collapsing leading NBSP sequences to the equivalent ASCII space count on lines inside fenced code blocks before comparing. This normalization does NOT apply to NBSP elsewhere in the body (non-leading positions, or outside fenced blocks — there NBSP is a value character).
3. **Markdown table normalization.** Slack applies three independent normalizations to markdown tables on storage — see [`contracts/canvas-write-rules.md`](../../../contracts/canvas-write-rules.md) v1.3 companion notes #5. Normalize both sides for ALL three before comparing:
   - **3a — Separator-row padding.** Slack pads separator-row cells with 2 spaces around the dashes (`|---|` → `|  ---  |`). A separator row is a row whose cells each contain only ASCII dashes (`-`), optional leading/trailing alignment colons (`:`), and whitespace — pattern `| *:?-+:? *|...| *:?-+:? *|`. Normalize by collapsing whitespace inside separator-row cells while **preserving dashes and alignment colons**: `|  ---  |` → `|---|` (no alignment); `|  :---  |` → `|:---|` (left); `|  ---:  |` → `|---:|` (right); `|  :---:  |` → `|:---:|` (center). The empirical probe covered only plain-dash separators; alignment-colon separators are theoretical for current seed canvases but the rule must not strip the colons.
   - **3b — Header/data row cell-content trimming.** Slack strips optional inner-cell whitespace on header and data rows (`| Col A |` → `|Col A|`). Normalize by stripping leading/trailing spaces inside each cell of non-separator rows.
   - **3c — Post-table newline insertion.** Slack inserts one additional `\n` after the table closing row. Normalize by collapsing the run of consecutive newlines immediately after a table to a single `\n\n` (one blank line) on both sides before comparing.
   Cell content itself (non-whitespace) is NOT normalized — real drift inside a data cell still surfaces as a verify failure.

Any byte difference remaining after these three normalizations — including trailing whitespace on body lines outside table separator cells, a single character of TOON-block content, or any drift outside the three expected categories — is a verify failure.

If the normalized diff is non-empty, surface:

> Canvas write verify mismatch on `<canvas_id>`:
>   first divergence at line `<N>`:
>     expected: `<line from intended_body>`
>     observed: `<line from re-read>`
> The write may have partially failed or been clobbered by a concurrent writer. Do NOT auto-retry — investigate (cf. canvas-write-rules.md Rule 3 read-before-write reasoning).

Do NOT auto-retry. Concurrent-writer clobber is the most likely cause; retrying compounds the race. The consumer decides whether to retry.

### 8. Paste fallback on write failure

If step 6 raised an error, do NOT raise the error up the stack silently. Emit a fenced code block containing the full `intended_body` with paste instructions, per canvas-write-rules.md Rule 4:

> Canvas write failed (`<error>`). Copy the block below and paste into <canvas link or canvas_id> via the Slack UI — replacing all content (Cmd-A, paste).
> ```
> <intended_body>
> ```

This is the structural enforcement of Rule 4 — "never let work go silently wrong." The consumer's user sees the work product even when the API path fails.

If a canvas link is constructable from `canvas_id` (workspace URL + canvas ID), include it for one-click paste. Otherwise, fall back to the bare `canvas_id` and the instruction "open this canvas in Slack."

### 9. Log

Emit a structured log line:

```
[slack-canvas-safe-write] canvas=<canvas_id> op=<create|update> actor=<slack_id> intent="<intent>" h1_changed=<true|false> mentions_translated=<count> dual_write=<active|inactive|n/a> verify=<ok|mismatch|fallback>
```

For `create`: `canvas_id` is the F-prefix ID **returned by `slack_create_canvas`** in step 6 — log it after the create call resolves, not before. If the create failed before returning an ID (paste-fallback path from step 8), log `canvas=create-failed` so the line still emits.

For `update`: `canvas_id` is the ID passed in by the consumer (step 1's read used the same value).

Until the seed Activity Feed yearly-rolling sheets land, the chat log is the trail.

## Bug-pattern knowledge baked in

Each enforcement step traces back to a specific Slack Canvas API behavior that has corrupted canvases in production. Cross-reference for incident review:

| Rule | Step | Production failure mode |
|---|---|---|
| canvas-write-rules.md Rule 1 (create once, patch by full-replace) | 0, 6 | (implicit — the skill's only operations are `create` and full-replace `update`). |
| canvas-write-rules.md Rule 2 (no `section_id`, all blocks) | 0 | Ghost-block bug on section-replace observed on table blocks, config paragraphs, Hub Team-table rows. Skill rejects `section_id` outright. |
| canvas-write-rules.md Rule 3 (read before patch) | 1 | Write-from-memory clobber. Skill performs the read as part of its own flow; the consumer cannot legitimately skip it. |
| canvas-write-rules.md Rule 4 (paste in chat on failure) | 8 | Silent write failure — work disappears into a stack trace with no recovery path. Skill emits the fenced block automatically. |
| canvas-write-rules.md Rule 5 (no body H1 on create) | 2 | 2× duplicate header on canvas creation when the body starts with `# {title}`. Skill aborts the create. |
| canvas-write-rules.md Rule 6 (ghost h1 on rename) | 3 | Old H1 persists as a frozen block when the new H1 differs. Skill surfaces the decision (halt-for-confirmation) rather than silently writing through. |
| `@mention` read↔write asymmetry (canvas-write-rules.md companion note) | 4 | Untranslated `<@USERID>` in a write produces broken mention renders. Skill translates automatically and idempotently. |
| routine-config-canvas.md v2.4 Phase 2 dual-write + CONTROL_TOWER.md v2.0.6 markdown-as-composition-source | 5 | TOON-only or markdown-only writes would leave the canvas in a single-format state, breaking the Routine's TOON-first reader (CLAUDE.md v2.12 Branch A / Branch B / Branch C decision tree). Skill detects existing TOON block, requires both formats in the same full-replace, cross-checks row counts. No-op only on canvases that haven't yet been touched by a TOON-aware writer; transitions automatically the first time a Phase 2 producer writes the canvas. |

## Universal availability — headless loader

This skill is invoked from three contexts. The mechanism for "load the skill" differs by context; the protocol does not.

**Local Claude Code session (interactive).** The skill auto-loads when the working directory is the `project-routines/` repo (or a sibling repo that contains a `.claude/skills/` directory pointing at this file). The user invokes it implicitly — any canvas-write request triggers the skill's flow.

**Cloud Claude Code session.** Same as local, provided the session opens the `project-routines/` repo as step zero. Convention: cloud Routine prompts include a "cd into `project-routines/` before any canvas write" line in their bootstrap.

**Headless Claude API (cron-scheduled Routines, scheduled tasks, Workshop running from a remote trigger).** There is no native skill loader in headless mode. The Routine's bootstrap inlines the SKILL.md content into the system prompt at agent init by `gh api`-fetching the file from `joshpayne-joby/project-routines` `main` at session start. The headless agent then treats the skill's flow as binding instructions for any canvas write in that session. Pattern matches CLAUDE.md v2.10's bootstrap-field approach.

Mirror of smartsheet-safe-write's loader pattern — the skill is just markdown; whichever consumer needs it pulls it in via the loader appropriate to its execution context.

## Acceptance criteria (from issue #63)

- [x] Skill lives at `.claude/skills/slack-canvas-safe-write/SKILL.md`.
- [x] Inputs the consumer must provide are explicitly listed; the skill rejects calls that include `section_id`.
- [x] Read-before-write enforced in step 1.
- [x] Title/body-H1 collision check enforced in step 2 (create).
- [x] H1-rename ghost detection enforced in step 3 (update).
- [x] `@mention` translation enforced in step 4.
- [x] Routine Config dual-write detection enforced in step 5 (TOON contracts live on main as of 2026-05-29; rule is current state, not forward-compat).
- [x] Write enforced as full-canvas replace only in step 6.
- [x] Verify-after enforced in step 7.
- [x] Paste fallback enforced in step 8.
- [x] At least one test — see `tests.md` in this directory.
- [x] Loader pattern for headless usage — see "Universal availability" above.

## Failure modes

| Mode | Response |
|---|---|
| Consumer passes `section_id` | Abort with the step 0 message. Universal — every block type is unsafe under section-replace. |
| `slack_read_canvas` fails (canvas missing, connector down) on `update` | Surface the raw error. The skill cannot proceed without a read. Do NOT silently fall back to a bare write. |
| Body starts with `# {title}` on `create` | Abort with the step 2 message. Strict reject, not silent strip. |
| H1 rename detected on `update` | Halt-for-confirmation per step 3. Do not silently proceed; the ghost-cleanup cost is the consumer's call. |
| `intended_body` is missing required dual-write formats on a Phase 2 Routine Config canvas | Abort with the step 5 message. |
| TOON row count ≠ markdown row count on a Phase 2 Routine Config write | Abort with the step 5 count-mismatch message. Indicates composition-source violation. |
| `slack_update_canvas` / `slack_create_canvas` API error | Step 8 paste fallback. Emit the fenced block in chat with paste instructions. Do NOT auto-retry. |
| Verify (step 7) shows a write didn't apply | Surface the mismatch. Do NOT auto-retry — concurrent-writer clobber is the most likely cause and retrying compounds the race. |
| `intended_body` empty or trivially short (< 1 non-blank line) | Step 0 guard. Abort with the empty-body message. The skill does not write blank canvases; the consumer is almost certainly invoking with a bug (silent compose failure, variable shadowing, empty upstream read). |

## What this skill does NOT do

- **Read or write Smartsheet.** Smartsheet row writes go through [`.claude/skills/smartsheet-safe-write/SKILL.md`](../smartsheet-safe-write/SKILL.md). Different concern, different bug surface.
- **Decide who is allowed to write the canvas.** The consumer's logic owns access control (CT decides "is this user allowed to add a project"; Workshop decides "is this person authorized to onboard a collaborator"). This skill enforces the shape of the write, not the authorization to make it.
- **Compose `intended_body`.** The consumer assembles the markdown body; the skill enforces it on the way to Slack. Composing the Routine Config TOON block from the markdown table read is the CT/Workshop writer's job — the skill cross-checks both formats are present and row-count-consistent but does not synthesize either.
- **Cache state across invocations.** Every call reads the canvas fresh. No "last-known body" memo, no cross-invocation `current_canvas`.
- **Auto-retry.** API errors, verify mismatches, and H1-rename halts all surface to the consumer. Retry is the consumer's call.
- **Provide a diff-style helper API (replace_field, replace_table_row, append_to_section).** The original issue #63 sketch proposed a typed-diff API; for v0.1 the skill enforces the rules and lets the consumer compose. Diff helpers can land in a follow-up once the enforcement surface stabilizes.

## Reference

- [`contracts/canvas-write-rules.md`](../../../contracts/canvas-write-rules.md) v1.3 — full canvas-write rules contract. Each rule maps to a step in the flow above (see "Bug-pattern knowledge baked in"); v1.2 added the "Companion notes — observed Slack canvas API behaviors" reference section; v1.3 captures reproduction notes for behaviors #2 and #5 and expands #5 to 3 sub-rules.
- [`contracts/canvas-writer-preamble.md`](../../../contracts/canvas-writer-preamble.md) v1.0 — 6-line operational preamble derived from the contract; prompts that drive canvas writes can paste from it instead of inlining the rules. Companion to this skill.
- [`contracts/routine-config-canvas.md`](../../../contracts/routine-config-canvas.md) v2.4 — Phase 2 TOON dual-write rules. Drives step 5.
- [`canvas-specs/CONTROL_TOWER.md`](../../../canvas-specs/CONTROL_TOWER.md) v2.0.6 — markdown-as-composition-source rule. Drives step 5's row-count cross-check.
- [`contracts/toon-canvas-block.md`](../../../contracts/toon-canvas-block.md) v1.1 — TOON block shape and fence syntax (backtick-only); v1.1 adds the "Indentation whitespace" tolerance rule that pairs with step 7's NBSP normalization. Drives step 5's TOON fence detection.
- [`CLAUDE.md`](../../../CLAUDE.md) v2.12 — Routine's TOON-first reader (Branch A / B / C decision tree). The skill's step 5 enforcement keeps writes consistent with what the reader expects.
- [`.claude/skills/smartsheet-safe-write/SKILL.md`](../smartsheet-safe-write/SKILL.md) — sibling skill, same structural pattern applied to Smartsheet rows. Reference for the read-snapshot-mismatch-write-verify shape.

## Changelog

- **v0.1.3 — 2026-06-01** — Step 7 markdown-table normalization expanded to 3 sub-rules
  - **What** — Normalization #3 expanded from single-rule (separator-row padding) to three sub-rules: separator-row padding (3a), header/data row cell-content trimming (3b), post-table newline insertion (3c).
  - **Why** — F3 probe 2026-06-01 reproduced behavior #5 in full on sandbox `F0B7ESV6QEP`; pre-v0.1.3 tolerance covered only 3a, leaving 3b and 3c as latent false-positive sources on every table-bearing write.
  - **Impact** — Verify-after no longer false-fires on standard markdown tables (every table write was previously affected). Cell content (non-whitespace) still NOT normalized — real drift inside a data cell still surfaces.
  - **Companions** — Same PR: [`canvas-write-rules.md`](../../../contracts/canvas-write-rules.md) v1.3 (behavior #5 expanded), `tests.md` v0.3 (Scenario 13 sub-rules).

- **v0.1.2 — 2026-06-01** — Step 7 verify-after tolerates 2 new expected normalizations
  - **What** — Step 7 now tolerates (a) NBSP-vs-ASCII-space drift in fenced code-block indentation, (b) table-separator padding inside markdown table separator rows.
  - **Why** — Both behaviors observed on `F0B19UB8H0S` 2026-06-01; without normalization, verify-after fires false-positive mismatches on every legitimate write touching indented TOON or markdown tables.
  - **Impact** — Normalizations are narrowly scoped (leading positions in fenced blocks only; separator rows only); real drift still surfaces — mid-value NBSP, content drift, trailing whitespace outside separator cells.
  - **Companions** — PR #107: `canvas-write-rules.md` v1.2 companion notes #4-#5, `toon-canvas-block.md` v1.1 "Indentation whitespace," `tests.md` v0.2 Scenarios 12-13.
- **v0.1.1 — 2026-06-01** — Step 3 H1 comparison strips trailing whitespace on both sides before equality check. Surfaced by live test on `F0B19UB8H0S` (Josh's Routine Config canvas) 2026-06-01: the canvas's body H1 carried a trailing space (`# Josh | Routine Config `, likely from an early hand-edit or a write that didn't normalize) while any subsequent producer composing the conventional form (`# Josh | Routine Config`, no trailing space) would have fired step 3's rename-halt warning on every invocation — a false-positive blocking pattern. The strip eliminates the false-positive without softening the real H1-rename detection (`# Josh | Routine Config` vs `# Josh | Daily Briefing` still triggers, as it should). Pure whitespace tightening; no behavior change on genuine renames. Companion observation: the same canvas has a duplicate body-H1 (two `# Josh | Routine Config` lines back-to-back, one with trailing space, one without), AND the title slot is also `Josh | Routine Config` — Rule 5/6 violations baked in before the skill existed; these need Slack UI cleanup (tracked separately, not in this patch).
- **v0.1 — 2026-05-29** — Initial implementation. Closes #63 (migrated from `joby/project-routines#26`). Replaces the scaffold-only SKILL.md that landed in `505a263` (PR #92 history) and was dropped in `309dcb1` per "no empty promises" — every step in the flow below is implementable against the Slack MCP today. The TOON dual-write rule in step 5 enforces `contracts/routine-config-canvas.md` v2.4 (Phase 2 dual-write) and `canvas-specs/CONTROL_TOWER.md` v2.0.6 (markdown-as-composition-source) — both live on `main` as of 2026-05-29 via PRs #94–#100. The PR for this skill (PR #102) sat behind that TOON contract wave; the rebase onto current main + reference refresh happened in commit `baf178e` before merge. Diff-style helper API (replace_field, etc., from the original issue sketch) deferred to a future v0.2 — v0.1 enforces the rules and lets consumers compose intended_body. Companions: `contracts/canvas-writer-preamble.md` v1.0 (operational copy-paste); `contracts/canvas-write-rules.md` v1.1 (full contract); `contracts/toon-canvas-block.md` v1.0 (TOON block shape).
  - **Round 1 bot-review fold-in:** (a) Step 5 `update`-path detection uses `current_canvas.title` (already in memory from step 1) instead of body-content checks — immune to false fires on canvases that reference Routine Config tables without being one. (b) Step 5 TOON fence syntax tightened to backtick-only per `contracts/toon-canvas-block.md` v1.0 (dropped dead tilde-form arm). (c) Step 7 verify diff is concretely defined: only @mention round-trip diffs are expected; anything else is a mismatch. (d) Step 0 gains an empty-`intended_body` guard so the named failure-mode case is structurally enforced, not just listed. (e) Step 9 log line clarifies `canvas=` carries the F-prefix ID returned by `slack_create_canvas` on `create`, or `create-failed` if the call errored before returning. (f) `tests.md` Scenario 5 promotes the programmatic pass criterion above the visual one; new Scenarios 8 + 9 cover read-failure and verify-mismatch failure modes; sandbox canvas/user/channel IDs marked as `<FILL_IN_*>` placeholders so first-run errors surface loudly. (g) `contracts/canvas-writer-preamble.md` footnote re-orders the contract-rule mapping into preamble order.
  - **Round 2 bot-review fold-in + main-rebase:** Round 2 caught one regression from round 1 — the "When to use" consumer table's Workshop Stage 8 row still carried the dropped `~~~toon` tilde-form AND a "Rule 7" misreference (`canvas-write-rules.md` only has Rules 1-6; the dual-write is from `routine-config-canvas.md`, not the write-rules contract, and the skill's step number is 5). Both fixed in the same cell: `` ```toon `` syntax + "Step 5" not "Rule 7." Same fold-in also handled the long-tail issue that the PR's TOON references were stale: `routine-config-canvas.md` v2.2+ "pending merge on `feat/routine-config-canvas-v2.2-toon`" and `CONTROL_TOWER.md` v2.0.4 "pending merge on `feat/ct-spec-v2.0.4-toon-dual-write`" both landed on main via #94–#100 while this PR sat. References updated to current main versions (v2.4 / v2.0.6); step 5 prose, bug-pattern table, acceptance criteria checklist, and reference section all reflect "current state, not forward-compat speculation." Companion citations added for `toon-canvas-block.md` v1.0 (fence syntax + block shape) and `CLAUDE.md` v2.12 (TOON-first reader Branch A/B/C — the skill's writer must keep canvases consistent with what the reader expects). `tests.md` Scenario 9 gains a mock-injection note for sandbox execution per round 2 bot recommendation.
