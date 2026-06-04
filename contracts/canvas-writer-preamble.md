# Canvas Writer Preamble

The 6-line block below is the operational copy-paste for any prompt/system-message
that drives a Slack canvas write. Read-only Slack consumers don't need it.
Full contract with rationale, motivating cases, and version history:
`contracts/canvas-write-rules.md`.

---

**Canvas Write Rules** — applies to any `slack_update_canvas` / `slack_create_canvas` call.
Full contract: `contracts/canvas-write-rules.md`.

1. **Full-canvas replace only.** `action=replace`, no `section_id`. Section-level replace
   corrupts the canvas (ghost-block bug, universal — not table-specific).
2. **Read before write.** `slack_read_canvas` first → modify in memory → full canvas back
   in one call. Never write from memory.
3. **No body H1 matching title.** On create, title goes ONLY in the `title` parameter.
   Adding `# Name` to body too renders 2× (Slack stores title slot + body H1 separately).
4. **Renaming an H1 leaves a ghost.** Full-replace with a different h1 keeps the old h1
   above new content (only Slack UI delete clears it). Plan manual UI cleanup if needed.
5. **If a write fails**, paste the intended content as a fenced code block in chat for
   the user to paste manually.
6. **Translate @mentions for write.** `slack_read_canvas` returns `<@USERID>` and
   `<#CHANID>`. Translate to `![](@USERID)` and `![](#CHANID)` before writing —
   angle-bracket format produces broken renders in `slack_update_canvas`.

---

*v1.0 — derived from `contracts/canvas-write-rules.md` v1.1, Rules 1+2 → preamble 1,
Rule 3 → preamble 2, Rule 5 → preamble 3, Rule 6 → preamble 4, Rule 4 → preamble 5,
plus the @mention companion note → preamble 6. When you update either file, update
both — preamble must stay in sync with the contract's load-bearing imperatives.*
