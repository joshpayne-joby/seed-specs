# contracts/canvas-write-rules.md
# Canvas Write Rules Contract
# Version 1.3 | June 2026 | Josh Payne

---

## What this is

Canonical rules for every write to a Slack Canvas in the seed framework. Extracted from `PROJECT_SETUP.md`'s inline Canvas Write Rules section so Workshop, Control Tower, Routines, and any future canvas-writing component can reference one source instead of duplicating the rules.

These rules exist because Slack's Canvas API has known behaviors (ghost-block bug on section-replace, title-vs-h1 duplication, etc.) that would corrupt canvases if ignored.

---

## Scope

These rules apply to **every write** to any Slack Canvas via `slack_update_canvas` or `slack_create_canvas`, including but not limited to:

- New canvas creation (provisioner: Hub, Claude, Human, Routine Config, My Tasks, etc.)
- Canvas content updates (Routine briefing writeback; CT writes to My Tasks; provisioner Hub patch)
- Mirror updates (CT canvas, Prime canvas — when paste-from-mirror is used)
- Manual ad-hoc writes via the Slack MCP

Rules apply equally to project canvases and personal/workspace canvases.

---

## The rules

**Rule 1 — Create once, patch only by full-canvas replace.**

`slack_create_canvas` writes the initial canvas content in one shot. Any later edit to that canvas — or to any pre-existing canvas — must use `slack_update_canvas` with `action=replace` and **no** `section_id` parameter. This replaces the entire canvas body at once.

**Rule 2 — Never use `section_id`. Section-level replace is universally unsafe.**

Do not call `slack_update_canvas` with a `section_id`. The Slack Canvas API has a confirmed bug where section-level replace appends a duplicate block instead of swapping the target. **The bug is not table-specific** — empirically observed on multiple block types: tables (the original 2026-04-XX finding), config paragraphs (AES-UNS pilot 2026-05-04 — Claude Canvas Canvas Registry block), and Hub Team-table rows (same pilot). Treat the bug as universal: any block type can ghost on section-replace. Full-canvas replace is the only write pattern that works for any edit, regardless of what block you think you're targeting.

**Rule 3 — Read before you patch.**

Before any full-canvas replace, fetch the full current canvas content. Modify it in memory. Write the whole canvas back in one call. Never write from memory of what you *think* is there.

**Rule 4 — If a write fails, paste in chat.**

If a canvas write fails, output the full updated canvas content as a fenced code block in chat and tell the user: *"Write failed. Copy the block above and paste into [canvas link] — replacing all content."* Never let work go silently wrong.

**Rule 5 — On `slack_create_canvas`, the title goes ONLY in the `title` parameter.**

Do not also place a `# [Canvas Name]` h1 at the top of the body content. Slack stores the title slot and body h1 independently and renders both, producing a visible 2× duplicate header. The body should start with the first paragraph or `##` sub-heading.

**Rule 6 — Full-canvas replace ghosts a CHANGED h1, not a removed one.**

Full-canvas replace cleans up cleanly when the new content has the same h1 as the old, or no h1 at all. It does NOT clean up when the new content has a *different* h1 — the old h1 persists as a ghost block above the new one, and only Slack UI deletion can remove it. If a canvas h1 rename is unavoidable, plan for a manual UI cleanup pass after the API write; otherwise keep the h1 stable across patches.

---

## Companion: @mention format translation (read↔write asymmetry)

This isn't a Canvas Write Rule per se but a related Slack Canvas API quirk worth knowing:

- `slack_read_canvas` returns user mentions as `<@USERID>`
- `slack_update_canvas` requires user mentions as `![](@USERID)` to render correctly

Translate before submitting. Angle-bracket format on write produces inconsistent rendering. Same pattern for channel mentions: read returns `<#CHANID>`, write needs `![](#CHANID)`.

---

## Companion notes — observed Slack canvas API behaviors

Reference for "what Slack actually does" under the hood. These are descriptive — what the API was empirically observed to do — and each behavior is what motivates the prescriptive rules above (or, where the behavior is benign once tolerated, motivates the verify-after tolerances in [`.claude/skills/slack-canvas-safe-write/SKILL.md`](../.claude/skills/slack-canvas-safe-write/SKILL.md) step 7). Listed for incident triage and for new framework consumers asking "why does the rule say what it says."

### 1. CT corruption persistence (read-before-write inherits prior state)

**Behavior.** Once a canvas has an in-place Rule 5 or Rule 6 violation baked into its content — a duplicate body H1, a ghost H1 from a prior rename, or a body H1 matching the title slot — the read-before-write protocol PRESERVES that corruption on every subsequent skill-protected write. The next full-replace reads the corrupted body, modifies it in memory, writes it back, and the corruption rides forward unchanged.

**When it bites.** Canvases written by pre-skill producers (or by hand) that violated Rule 5 / Rule 6 carry their corruption forward indefinitely. New writes through the skill don't clean up the violation because the contract requires reading current state as the composition basis.

**Handled by.** Rules 5 and 6 prevent NEW corruption. Existing corruption requires manual Slack UI cleanup — the skill explicitly does not auto-strip body content it didn't put there, because doing so would mask intentional content the operator added. Rule 3's read-before-write is load-bearing for race protection; inheriting prior state is the deliberate trade-off.

**Empirical basis.** Live observed on `F0B19UB8H0S` 2026-06-01 — duplicate body-H1 (two `# Josh | Routine Config` lines back-to-back, one with trailing space, one without), and the title slot also `Josh | Routine Config`. Documented in `.claude/skills/slack-canvas-safe-write/SKILL.md` v0.1.1 changelog as "Rule 5/6 violations baked in before the skill existed."

### 2. FencedCodeBlock-in-list rejection

**Behavior.** Slack's canvas API hard-rejects writes containing a fenced code block nested inside a list item. Both paths reject atomically (no partial write), but the error shape differs:

- **Create path** (`slack_create_canvas`): specific parse-time error naming the offending line and block type: `canvas_creation_failed: line N: Unsupported block type (FencedCodeBlock) within list item`.
- **Update path** (`slack_update_canvas` with `action=replace`): generic `internal_error` — no line number, no block-type name. Operationally less informative; consumers diagnosing in production from an `internal_error` need to know to check for nested fences as a candidate cause.

Not a silent re-render; not a ghost block on either path. The whole write fails atomically and the existing canvas content is preserved on the update path.

**When it bites.** Markdown bodies that embed a code fence inside a `- ` or `1. ` list item — common in worked-example documentation prose, less common in machine-generated bodies. The Routine's My Tasks write and Workshop canvas creates do not naturally produce this shape; Drift Watcher report writes are the most likely consumer to trip it as those embed code fragments inside finding bullets. Note: most seed framework writes go through the update path, where the error is the less-informative `internal_error`.

**Handled by.** No skill enforcement — the hard rejection surfaces the failure immediately (write error returned to the caller; the skill's step 8 paste-fallback engages). Producers should avoid embedding fenced blocks inside list items. The workaround is to lift the fenced block out of the list item — place the list item heading, then the fence at the same indent level as the surrounding prose.

**Empirical basis.** Reproduced 2026-06-01 via F3 sandbox probe on `F0B7ESV6QEP`. Create-path: `slack_create_canvas` with a body containing `- item.\n  \`\`\`\n  fenced\n  \`\`\`\n- item.` returned `canvas_creation_failed: line 14: Unsupported block type (FencedCodeBlock) within list item`. Update-path: `slack_update_canvas(action=replace)` with the same nested-fence pattern returned `internal_error` on two consecutive probe attempts; the canvas's prior content was preserved each time. Top-level fenced blocks (outside any list) round-trip cleanly in the same payload via both paths.

### 3. H1 auto-preservation (Rule 6 territory)

**Behavior.** Full-canvas replace does NOT delete the existing body H1 region when `intended_body` carries a different H1. The old H1 persists as a frozen block above the new content. Equivalent: the H1 region has independent persistence from the rest of the body content in Slack's canvas DOM. Only Slack UI deletion clears it.

**When it bites.** Renaming a canvas heading via API full-replace.

**Handled by.** Rule 6 above (ghost h1 on rename) + the skill's step 3 halt-for-confirmation. The skill surfaces the decision rather than silently writing through.

**Empirical basis.** Original 2026-04-XX observation that drove Rule 6 (multi-canvas write pattern); corroborated by `F0B19UB8H0S` 2026-06-01 where the trailing-whitespace H1 lineage demonstrated stuck H1 state.

### 4. NBSP conversion (leading ASCII space inside fenced code blocks)

**Behavior.** Slack's canvas API silently converts leading ASCII spaces (`U+0020`) inside fenced code blocks to non-breaking spaces (`U+00A0`) on storage. A writer emits two ASCII spaces of indentation before `meta:`; the read returns the equivalent line with two NBSP characters instead. The non-leading positions and non-code-block positions are not affected in the same way (full positional coverage not yet probed).

**When it bites.** Any parser that splits on ASCII space alone to detect YAML-style indentation — including the TOON nested-map parse on Routine Config canvases. A strict ASCII parser sees no indentation and halts.

**Handled by.** [`toon-canvas-block.md`](toon-canvas-block.md) v1.1 "Indentation whitespace" rule (parsers MUST treat ASCII space and NBSP as equivalent for structural indentation). The skill's step 7 verify-after tolerates NBSP-vs-space drift in code-block indentation so legitimate writes don't false-fire on the round-trip diff.

**Empirical basis.** Live observed on `F0B19UB8H0S` 2026-06-01 during the `slack-canvas-safe-write` skill live test.

### 5. Markdown table normalization

**Behavior.** Slack normalizes markdown tables on storage via three independent sub-rules applied simultaneously:

- **5a — Separator-row padding.** Separator-row cells (containing only dashes and optional alignment colons) get padded with 2 spaces around the dashes: `|---|` → `|  ---  |`.
- **5b — Header/data row cell-content trimming.** Header and data rows have optional inner-cell whitespace stripped: `| Col A |` → `|Col A|`.
- **5c — Post-table newline insertion.** One additional `\n` is inserted after the table closing row, so the gap between the table and the next block grows by one blank line.

The rendered table is unchanged across all three normalizations; the byte diff is non-empty for every one of them.

**When it bites.** Verify-after diffs on canvases containing markdown tables. Without tolerance for ALL three sub-rules, every table-bearing write fires false-positive mismatches even though the rendered content is identical.

**Handled by.** The skill's step 7 verify-after tolerates all three sub-rules in the normalized diff — separator rows normalized to canonical `|---|...|---|`, header/data rows normalized to remove inner-cell whitespace, post-table newline collapsed to a single `\n\n` (one blank line). Cell content itself (non-whitespace) is NOT normalized — real drift inside a data cell still surfaces as a verify failure.

**Empirical basis.** Reproduced 2026-06-01 on sandbox canvas `F0B7ESV6QEP` via `slack_create_canvas` + `slack_read_canvas` round-trip. Input `| Col A | Col B | Col C |\n|---|---|---|\n| Row 1 A | Row 1 B | Row 1 C |` read back as `|Col A|Col B|Col C|\n|  ---  |  ---  |  ---  |\n|Row 1 A|Row 1 B|Row 1 C|` followed by an extra blank line before the next block. Both #5a and #5b confirmed across two test tables (3-col minimal + 3-col with mixed cell widths).

---

## Producers and consumers

| Component | Operation | Notes |
|---|---|---|
| `PROJECT_SETUP.md` (provisioner) | Create + patch | Three project canvases at creation; Hub patch after Claude canvas exists |
| `canvas-specs/CONTROL_TOWER.md` (CT v2.0+) | Replace | Step 4 missing-project fallback writes My Tasks + Routine Config canvas |
| `CLAUDE.md` (briefing Routine) | Replace | End-of-run full replace of My Tasks |
| `PROVISIONER_WORKSHOP.md` (Workshop v0.6+, all three modes) | Create + full-replace | PROJECT mode Phase 1/2/3 + upgrade flows; COLLAB mode My Tasks + Routine Config canvases per onboard (replaces retired `COLLABORATOR_SETUP.md`); BOOTSTRAP mode Prime + seed Changelog canvases per fresh-workspace install (replaces retired `BOOTSTRAP.md`). Inherits these rules by reference. |
| Manual ops via Slack MCP | Replace | Ad-hoc canvas writes (e.g. Adam onboarding canvas refresh) |

---

## Failure modes captured by these rules

| Failure mode | Caught by |
|---|---|
| Ghost-block bug from section-replace | Rule 1, Rule 2 |
| Stale-content overwrite from memory-write | Rule 3 |
| Silent corruption from unreported write failure | Rule 4 |
| 2× duplicate header on canvas creation | Rule 5 |
| Ghost h1 persisting after rename | Rule 6 |

---

## Verification

For Rule 1-3: behavior is invisible from the API; only verify by reading the canvas after write and comparing to expected state.

For Rule 4: if you don't see a "Write failed. Copy the block above…" code block in chat after a failure, the rule was violated.

For Rule 5: visible after canvas creation — duplicate header in the rendered Slack UI.

For Rule 6: visible after a full-replace that changes h1 — old h1 appears as a frozen header above new content.

---

## Changelog

- **v1.3 — 2026-06-01** — Behaviors #2 + #5 reproduced; #5 expanded to 3 sub-rules
  - **What** — Companion note #2 gains reproduction for both create-path and update-path errors. Companion note #5 expanded to 3 sub-rules: separator padding, cell-content trimming, post-table newline insertion.
  - **Why** — F3 probe ran inline 2026-06-01 against sandbox `F0B7ESV6QEP`; #5 surfaced as broader than originally documented — three independent table normalizations, not one.
  - **Impact** — Drops "pending" hedge on #2 and #5. Pre-v1.3 verify-after false-fired on every table-bearing write (sub-rules 3b and 3c uncovered).
  - **Companions** — Same PR: `SKILL.md` v0.1.3 (step 7 expanded), `tests.md` v0.3 (Scenarios 11 + 13 reproduction details).

- **v1.2 — 2026-06-01** — Companion notes for 5 observed Slack canvas API behaviors
  - **What** — New "Companion notes — observed Slack canvas API behaviors" section documenting 5 behaviors: CT corruption persistence, FencedCodeBlock-in-list rejection (pending), H1 auto-preservation, NBSP conversion, table-separator padding (pending).
  - **Why** — Five behaviors surfaced during the `slack-canvas-safe-write` skill live test on `F0B19UB8H0S` 2026-06-01; the section is the "what Slack actually does" reference complementing prescriptive Rules 1-6.
  - **Impact** — Two behaviors flagged "reproduction notes pending" — empirical reproduction deferred to a future F3 Slack normalization probe. Descriptive only; no rule changes, no producer/consumer changes.
  - **Companions** — PR #107: [`toon-canvas-block.md`](toon-canvas-block.md) v1.1 (NBSP parser tolerance), `SKILL.md` v0.1.2 (step 7 verify tolerance), `tests.md` v0.2 (Scenarios 4 + 10-13 — 5 scenarios for 5 behaviors).
- **v1.1 — 2026-05-05** — Rule 2 strengthening from AES-UNS pilot (Workshop v0.1.2 → v0.1.3 cycle). Pilot Workshop hit the section-replace ghost-block bug on TWO non-table block types (Claude Canvas config paragraph, Hub Team table) and self-corrected to full-canvas replace both times. Prior wording suggested the bug was table-specific ("often a ghost empty table row"); reality is broader. Strengthened Rule 2 to call section-replace universally unsafe, regardless of block type. Producers/consumers table unchanged.
- **v1.0 — 2026-05-04** — Initial version. Rules 1-6 extracted verbatim from `PROJECT_SETUP.md` v1.16 (where they had lived since v1.10–v1.11). Added the @mention format translation note as companion knowledge. Producers/consumers table reflects current writers (`PROJECT_SETUP.md`, `mirrors/CONTROL_TOWER.md` CT v2.0+, briefing Routine, `COLLABORATOR_SETUP.md`, future Workshop). Lands alongside `contracts/canvas-access.md` v1.0 as the "two-contract ship" — both contracts are referenced from `PROJECT_SETUP.md` v1.17.
