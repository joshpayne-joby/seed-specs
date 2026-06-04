# contracts/canvas-access.md
# Canvas Access Contract
# Version 1.0 | May 2026 | Josh Payne

---

## What this is

The canonical rule for how **project canvases** (Hub, Claude, Human) get access-scoped at creation time and afterward in the seed framework. Project canvases are **channel-scoped**: members of the project's Slack channel get edit access; others don't.

This contract supersedes any inline canvas-access guidance in `PROJECT_SETUP.md`, `PROVISIONER_WORKSHOP.md` v0.2+, `canvas-specs/CONTROL_TOWER.md`, or other provisioner modes — those documents reference this contract.

---

## Scope

**Applies to project canvases:**

- **Hub canvas** — project landing page
- **Claude Canvas** — task board + session log
- **Human Canvas** (Field Reference) — photos + field notes

**Does NOT apply to:**

- **Personal canvases** (My Tasks, Routine Config) — single-owner, no project channel binding; access is owner-only by default
- **Workspace canvases** (Prime Project Canvas, seed Changelog, seed Overview) — workspace-public by design (every collaborator reads them)

---

## The rule

For each project canvas, at creation time:

1. **General access:** **"Only invited people can access"** — do NOT set "Anyone at Joby Aviation can view/edit"
2. **Channel access:** add the project's Slack channel (`Channel ID` from `Project Registry — Core`) with permission level **"Can edit"**
3. **Owner access:** the canvas creator is automatically owner — no additional setup needed

Channel members get access via the channel ACL entry. New members joining the channel later inherit access automatically. People not in the channel cannot read the canvas.

---

## Why channel-scoped, not workspace-wide

Workspace-wide ("Anyone at Joby Aviation") exposes every project's contents to all of Joby. Channel-scoped keeps each project's canvas content visible only to its team. This matches the security posture seed's collaborators expect — particularly for projects with sensitive supplier, vendor, or strategic content.

---

## How it gets applied

### At creation (PROJECT_SETUP.md)

PROJECT_SETUP.md's "Create the Canvases" section instructs the provisioner to apply this rule for each of the three project canvases immediately after creation.

### Manual application — Slack UI Share dialog (primary path today)

For each canvas:

1. Open the canvas → click **Share** (top right)
2. In "Add people, channels…" type the project channel name (e.g. `amfg-aes-htpress`) → select it
3. Set permission to **Can edit**
4. Confirm **General access** still reads **"Only invited people can access"** — DO NOT change to "Anyone at Joby Aviation"
5. Save

The UI dialog handles any prerequisite link-posting implicitly. Verified end-to-end on AES-HTPRESS Hub `F0B15QRPJ1G` on 2026-05-04.

### API automation (future)

Slack's Web API exposes `canvases.access.set` with `channel_ids: [Cxxx]` and `access_level: "edit"`. The current Slack MCP available to seed Routines does **NOT** expose this method (verified 2026-05-04). When a custom MCP server wrapping it lands (likely on the Joby gateway at `mcp.sw.az.joby.aero`), this contract's procedure becomes a 1-call automation.

**Known prerequisite for the bare API call:** the canvas link must be posted in the channel before `canvases.access.set` succeeds — otherwise it returns `access_denied`. The UI Share dialog handles this implicitly; the API call requires an explicit `chat.postMessage` first or paired use.

---

## No-channel projects

If a project has `Channel ID = none` in the registry (rare; solo work, multi-channel projects, or personal canvases mis-classified as project), channel-scope doesn't apply. Default:

- **General access:** "Only invited people can access"
- **Specific people:** invite collaborators directly via the Share dialog
- **Revisit:** when a project channel is created, retrofit per the rule above

---

## Owner constraint

Per Slack docs, only a canvas's **owner** (or workspace admin) can call `canvases.access.set` or modify the Share dialog's permission controls. Implications:

- For canvases the seed framework provisioner creates → the executing PM is owner; the contract applies cleanly
- For canvases created outside the framework (e.g. a project Hub created ad-hoc by another PM) → only that PM can apply this contract; cross-team hand-off is required to retrofit
- For workspace admins → can override owner constraint; useful escape hatch for legacy backfill

---

## API introspection — known limit

Slack's Web API does **NOT** expose canvas ACL state — there's no `canvases.access.list` method. You cannot programmatically check "is this canvas shared to channel X with Can edit?" Verification is UI-only or via probe-with-second-account ("does a non-channel-member's read return `access_denied`?").

This is a platform constraint. It limits what a future drift-watcher Routine can detect directly on the canvas-access dimension — the indirect signal is reachable (a Smartsheet `Collaborators` member who fails to read a project canvas), but the configured ACL state is not.

---

## Producers and consumers

| Actor | Operation | Notes |
|---|---|---|
| `PROJECT_SETUP.md` | Apply at create | New canvas-creation step references this contract per canvas |
| Workshop PROJECT mode (future) | Apply at create | Inherits this contract by reference |
| Manual ops (Slack UI) | Retrofit | Legacy projects + cross-team retrofits |
| Drift-watcher Routine (future) | Detect indirect signals | Can't read ACL state; flags `access_denied` from listed Smartsheet collaborators |
| Slack workspace admin | Override | Can apply this contract on canvases the operator doesn't own |

---

## Verification

UI-only:

- Open the canvas's Share dialog
- Confirm **General access:** "Only invited people can access"
- Confirm the **Specific people / channels list** contains the project channel with **Can edit**

If the dialog shows otherwise (workspace-wide, or no channel entry), retrofit per the manual application path above.

---

## Changelog

- **v1.0 — 2026-05-04** — Initial version. Codifies channel-scoped canvas access for project canvases. Surfaced from a cold-run COLLABORATOR_SETUP report (2026-05-02): collaborators added to Smartsheet `Collaborators` couldn't read Hub canvases that were workspace-wide-by-default. Channel-share mechanism verified live 2026-05-04 on AES-HTPRESS Hub `F0B15QRPJ1G`. Replaces the now-incorrect "Anyone at Joby Aviation, no individual permissions needed" guidance in PROJECT_SETUP.md:615.
